# LangChain-Style RAG Pipeline Analysis: Roachie vs 9-Module Architecture

**Date:** 2026-03-25
**Subject:** Evaluating roachie's RAG implementation against the canonical loaders → splitters → embeddings → vectorstore → retriever → prompts → chains → llms → memory pipeline
**Core thesis:** RAG is a pipeline, not a feature — each module must exist, connect cleanly, and be independently testable

---

## Module Evaluation

### 1. loaders/ — Ingest PDFs, CSVs, Web Pages, Databases

**What it prescribes:** A loader abstraction that normalizes data from heterogeneous sources (documents, structured data, web content, databases) into a common format for downstream processing.

**Pros — What roachie does well:**
- **Database loader** is live and production-grade: `_nl_fetch_schema_context()` queries CockroachDB via `SHOW DATABASES` + `information_schema`, extracting database names, table names, and tenant assignments — a genuine database loader that runs at session start
- **File loader** for tool descriptions: reads `tool_descriptions.txt` (77 pipe-delimited entries) and `crdb_knowledge_base.txt` (14 CHUNK_ID|TITLE|CONTENT entries) — structured flat-file ingestion
- **Shell loader** for tool help: `--help` output fetched live via shell execution for matched tools during prompt enrichment — dynamic content loading
- **Metrics loader**: reads JSONL failure/success files and CSV session metrics — structured data ingestion for learning

**Cons — What's missing:**
- No loader abstraction — each data source has its own bespoke loading code scattered across `llm_prompt.sh`, `llm_metrics.sh`, and `generate_embeddings.sh`
- No PDF, web page, or unstructured document loaders — the 14 doc chunks were manually extracted from CockroachDB documentation
- No incremental loading — every session re-queries the database schema from scratch, re-reads tool descriptions, and re-loads embeddings
- No error handling standardization — database loader fails silently on connection errors (returns empty), file loaders assume files exist

**Room to improve:**
- Add a web loader for CockroachDB release notes / changelog pages — automate knowledge base updates
- Cache schema context with a TTL rather than re-querying every session
- Create a lightweight loader interface: `load_source(type, path) → chunks[]` to standardize across sources

**Score: 5/10** — Four distinct loader types exist (database, file, shell, metrics) but there's no abstraction layer, no unstructured document support, and no loader registry.

---

### 2. splitters/ — Chunk Large Documents Intelligently

**What it prescribes:** Split large documents into semantically meaningful chunks optimized for embedding and retrieval — not arbitrary fixed-length cuts.

**Pros — What roachie does well:**
- **Tool descriptions are naturally atomic** — each of 77 tools is one chunk with a curated purpose description plus augmented query phrases for 16 high-traffic tools. No splitting needed because the source data is already at the right granularity
- **Doc knowledge base uses topic-based chunking** — 14 manually curated chunks organized by CockroachDB topic (gc_ttl, changefeeds, slow queries, node health, backup/restore, contention, indexes, PCR, schema design, crdb_internal, error codes, performance tuning, multi-tenant, storage/compaction). Each chunk is self-contained with title and content
- **Schema context uses structured chunking** — database/table listings chunked by tenant, truncated to 15 lines for compact prompts, with max 5 databases and 50 tables per database as chunk size limits
- **Prompt templates are file-based chunks** — 8 template files in `tools/utils/prompts/` each covering one topic (execution rules, response format, hallucination warnings, etc.), loaded conditionally

**Cons — What's missing:**
- No automated splitting — all chunking is manual/pre-defined, not algorithmic
- No splitting of `--help` output — tool help text (5-100 lines) is injected whole, which can be wasteful for tools with lengthy help
- No overlap between chunks — adjacent doc chunks share no context, so a query at the boundary between "indexes" and "performance tuning" might miss relevant content
- No chunk size optimization — some doc chunks are 5 lines, others are 30 lines, with no target token count

**Room to improve:**
- Add help text summarization: for verbose tools, extract only the relevant flags rather than injecting full `--help` output
- Add cross-references between related doc chunks: "See also: performance tuning" at the end of the indexes chunk
- The natural atomicity of tool descriptions is a genuine strength — don't over-engineer splitting for data that's already well-chunked

**Score: 7/10** — Domain-natural chunking (one tool = one chunk, one topic = one chunk) is a deliberate and effective design choice. The system doesn't need a text splitter because its source data is already at the right granularity. Points lost for no automated splitting capability and no help text optimization.

---

### 3. embeddings/ — Convert Text into Vectors

**What it prescribes:** An embedding layer that converts text chunks into dense vector representations for similarity search, supporting multiple models and providers.

**Pros — What roachie does well:**
- **Three-provider embedding redundancy**: Ollama/nomic-embed-text (768d), OpenAI/text-embedding-3-small (1536d), Gemini/gemini-embedding-001 (3072d) — each with dedicated generation scripts and pre-computed output files
- **Automatic provider detection**: `_detect_embedding_provider()` auto-selects available embedding provider with graceful fallback chain: Ollama → OpenAI → Gemini → regex-only (no vectors)
- **Two embedding pipelines**: `generate_embeddings.sh` for tool descriptions (77 vectors), `generate_doc_embeddings.sh` for knowledge base chunks (14 vectors) — separate but parallel
- **Pre-computation model**: embeddings are generated at build time and stored as JSON files in `tools/embeddings/`, not computed at query time — zero embedding latency for tool matching at inference
- **Query embedding at inference**: user queries are embedded on-the-fly (single API call, ~10-50ms) for cosine similarity against pre-computed vectors

**Cons — What's missing:**
- No embedding abstraction — each provider has its own API call pattern hardcoded in the generation scripts
- No embedding caching for queries — if the same query is asked twice, it's re-embedded twice
- No embedding quality validation — no check that generated vectors are the right dimensionality or non-zero
- No batch embedding optimization — generation scripts embed one description at a time rather than batching API calls

**Room to improve:**
- Add a simple query embedding cache: hash the query text, cache the vector for the session duration
- Add validation after embedding generation: verify dimensionality, check for zero/NaN vectors
- Batch API calls in generation scripts: most providers support batch embedding (10-50 texts per call)

**Score: 8/10** — Three-provider support with automatic fallback, pre-computation for zero-latency tool matching, and separate pipelines for tools vs docs. This is one of roachie's strongest RAG components.

---

### 4. vectorstore/ — Store and Search Embeddings (FAISS, Chroma, Pinecone)

**What it prescribes:** A persistent, queryable vector storage layer that supports efficient similarity search at scale (FAISS for local, Chroma for lightweight, Pinecone for cloud).

**Pros — What roachie does well:**
- **Flat JSON files as vector store**: pre-computed embeddings stored in `tools/embeddings/` as JSON arrays with tool name + vector pairs — loaded into memory for similarity search
- **In-memory cosine similarity via jq**: `_semantic_match_tools()` computes cosine similarity between query vector and all tool vectors using a single jq expression — elegant, zero-dependency implementation
- **Pre-computed magnitudes**: tool vector magnitudes can be pre-computed to avoid redundant calculation per query
- **Provider-specific stores**: separate embedding files per provider, selected automatically based on available embedding model

**Cons — What's missing:**
- No vector database — flat JSON files are loaded entirely into memory for every query, no indexing
- No approximate nearest neighbor (ANN) — brute-force cosine similarity over all 77 vectors
- No persistence beyond files — no CRUD operations on individual vectors, no upsert/delete
- No filtering — can't query "most similar tool in the monitoring category" without post-filtering
- No multi-vector queries — can't combine tool embeddings with doc embeddings in a single search

**Room to improve:**
- **At current scale (77 tools + 14 docs), flat files are correct** — brute-force over 91 vectors takes <1ms, a vector DB would add latency and operational complexity
- Add metadata filtering to similarity search: filter by tool category before computing similarity
- If scaling beyond 500 tools, consider FAISS (local, zero-infra) or SQLite with vector extension
- Do NOT add Pinecone/Chroma/Weaviate for 77 vectors — this would be pure over-engineering

**Score: 6/10** — The implementation is right-sized for the scale. Flat JSON + jq cosine similarity is elegant and fast for 77 vectors. Points lost for no indexing, filtering, or CRUD — but these aren't needed yet.

---

### 5. retriever/ — Fetch Top-K Relevant Chunks

**What it prescribes:** A retrieval layer that takes a query and returns the K most relevant chunks from the vector store, potentially combining multiple retrieval strategies.

**Pros — What roachie does well:**
- **Hybrid retrieval** is the system's crown jewel: regex keyword matching (intent gate, ~1ms) + semantic cosine similarity (supplemental, ~10-200ms) — two independent retrieval strategies combined
- **Intelligent fusion**: semantic results listed first (priority), regex supplements, deduplicated, truncated to top 7 — principled result merging
- **Intent gating**: regex acts as a fast filter — if regex finds nothing, semantic is skipped entirely (avoids false positive semantic matches on off-topic queries)
- **Configurable parameters**: `_NL_MAX_ENRICHED_TOOLS=7` (top-K), `_NL_MIN_SIMILARITY=0.35` (threshold), both overridable via environment variables
- **Doc RAG retriever**: separate `_doc_rag_retrieve()` for knowledge base chunks with its own top-K (`_NL_DOC_RAG_TOP_K=3`)
- **99-100% accuracy** across 135 test queries with 3 embedding providers — validated retrieval quality
- **Accuracy test suite**: `test_embeddings.sh` with two modes (multi-tenant 71 queries, single-tenant 64 queries) and per-provider reporting

**Cons — What's missing:**
- No re-ranking step — cosine similarity is the only semantic relevance signal, no cross-encoder or learned re-ranker
- No query expansion — the user's query is matched as-is, no synonym expansion or query rewriting
- No relevance feedback — retrieved results don't influence future retrieval (no learning-to-rank)
- Tool retrieval and doc retrieval are separate pipelines with no unified retriever interface

**Room to improve:**
- Add lightweight re-ranking: after top-K semantic retrieval, score results by keyword overlap with the query as a second signal
- Add query normalization: lowercase, remove stop words, expand common abbreviations ("perf" → "performance")
- Unify tool and doc retrieval behind a common interface: `retrieve(query, source_type, top_k) → chunks[]`

**Score: 9/10** — Hybrid retrieval with intent gating, configurable parameters, and 99%+ validated accuracy. This is production-quality retrieval. The only gaps are re-ranking and query expansion, which are optimizations rather than fundamentals.

---

### 6. prompts/ — Structured Prompt Templates

**What it prescribes:** A template system for constructing prompts with variable substitution, conditional sections, and reusable components.

**Pros — What roachie does well:**
- **8 modular template files** in `tools/utils/prompts/`: execution_pattern, execution_rules, few_shot_examples, hallucination_warnings, parameter_rules, response_format, tool_notes, tool_specific_rules — each covering one concern
- **Conditional loading**: `_nl_load_prompt_template()` loads templates by name, with templates included/excluded based on context (hallucination warnings only for cloud providers, compact format for Ollama)
- **Variable substitution**: `{{TOOLS_DIR}}` placeholder replaced at load time with actual tools path
- **Two prompt variants**: `_nl_build_system_prompt()` (full, ~2K tokens base) for cloud providers (128K-1M context) and `_nl_build_system_prompt_compact()` (~500 tokens base) for Ollama (16K context)
- **Dynamic enrichment**: per-query tool help injected into the prompt after retrieval, so the prompt contains only relevant tool documentation
- **Personality modes**: "Roachie" (friendly) and "Roachman" (professional) tone modifiers injected conditionally
- **Structured output spec**: `response_format.txt` defines exact JSON schema with field semantics and multi-step reasoning rules

**Cons — What's missing:**
- No prompt versioning — template changes overwrite previous versions with no A/B testing capability
- No token counting — prompt assembly doesn't track total token count, risking context overflow for Ollama
- Variable substitution is limited to `{{TOOLS_DIR}}` — no general-purpose template engine
- No prompt validation — malformed templates aren't detected at load time

**Room to improve:**
- Add token estimation after prompt assembly: warn if total system prompt exceeds 80% of model context window
- Add prompt version tracking: hash the assembled prompt, log which version produced which accuracy
- The template system is functional and maintainable — don't add Jinja2/Mustache complexity to a bash project

**Score: 8/10** — Modular templates, conditional loading, two prompt variants, and structured output specification. The system is well-designed for a bash-based project. Token counting is the main gap.

---

### 7. chains/ — Connect Retriever + LLM

**What it prescribes:** An orchestration layer that chains the retriever to the LLM — retrieval results are injected into the prompt, which is sent to the model, and the response is parsed and acted upon.

**Pros — What roachie does well:**
- **The enrichment chain** is well-defined: user query → regex matching → semantic matching → hybrid fusion → tool help fetching → prompt injection → LLM call → JSON parsing → command execution — a complete RAG chain
- **Agent loop chain**: the 3-iteration observe-reason-act-refine loop chains LLM responses to subsequent LLM calls with command output as context — multi-step chain execution
- **Reflexion chain**: on command failure, error output is injected with explicit analysis instructions, creating a self-correcting chain
- **Doc RAG chain**: separate retrieval → prompt injection path for knowledge base chunks, running in parallel with tool retrieval
- **Schema chain**: live SQL → schema context → prompt injection — a separate data chain feeding the same prompt

**Cons — What's missing:**
- No chain abstraction — the pipeline is implemented as sequential function calls in `llm_prompt.sh` and `llm_assistant.sh`, not as composable chain objects
- No branching chains — can't route to different chains based on query type (informational vs operational vs diagnostic)
- No chain caching — identical queries re-execute the full chain (retrieval → enrichment → LLM call) every time
- No chain monitoring — no per-stage latency tracking within the chain (only total query latency is logged)
- Tool retrieval and doc retrieval chains don't share results — a query about "slow queries" retrieves tools AND doc chunks independently with no cross-referencing

**Room to improve:**
- Add per-stage timing: log how long retrieval, enrichment, LLM call, and command execution each take
- Add query type routing: informational queries (no tool execution needed) could skip the command pipeline
- The sequential function call approach is appropriate for bash — don't add LangChain-style chain abstractions to shell scripts

**Score: 7/10** — The retrieval-to-execution pipeline is complete and includes multi-step chaining (agent loop) and self-correction (reflexion). Missing chain abstraction and per-stage monitoring, but the functional implementation works.

---

### 8. llms/ — Model Configuration

**What it prescribes:** A configurable LLM layer supporting multiple models, providers, parameters (temperature, tokens, context window), and fallback strategies.

**Pros — What roachie does well:**
- **5 providers** behind a unified dispatcher: Anthropic Claude (Sonnet/Haiku/Opus), OpenAI (GPT-4.1/5/o-series), Gemini (2.5 Flash/Pro), Vertex AI (Claude via Google Cloud), Ollama (5 local models) — all accessed through `_call_llm_api()`
- **Per-provider plugins**: `src/lib/providers/anthropic.sh`, `openai.sh`, `gemini.sh`, `vertex.sh`, `ollama.sh` — clean separation of API-specific logic
- **Shared request builders**: `_build_openai_request()` and `_build_gemini_request()` eliminate duplication across streaming/non-streaming and native/non-native tool calling paths
- **Model-specific parameter handling**: GPT-5 and o-series models skip temperature (unsupported); Ollama models use `num_predict` instead of `max_tokens`; Anthropic uses `max_tokens`, OpenAI uses `max_completion_tokens`
- **Automatic provider detection**: `_detect_llm_provider()` scans for available API keys and local models with configurable priority
- **Fallback on failure**: in batch mode (`ROACHIE_BATCH=1`), failed provider auto-selects next available provider without user intervention
- **Cost tracking**: `_llm_cost_per_1m_tokens()` tracks per-model pricing for 20+ models
- **Native tool calling**: MCP schema conversion for Anthropic, OpenAI, and Gemini — the LLM layer supports both text-based and structured tool invocation

**Cons — What's missing:**
- No model comparison framework — can't A/B test the same query across providers and compare quality
- No automatic model selection based on query complexity — simple queries use the same expensive model as complex ones
- No rate limiting within the LLM layer — rate limit handling is per-provider with no unified strategy
- No model warm-up or health checks — if Ollama is running but the model isn't loaded, the first query is slow

**Room to improve:**
- Add complexity-based model routing: simple queries ("show tables") → fast/cheap model, complex queries ("analyze performance across tenants") → powerful model
- Add model health checks at startup: verify the selected model responds before entering the REPL
- The multi-provider architecture with automatic fallback is already strong — focus on routing optimization, not more providers

**Score: 9/10** — Five providers, unified dispatcher, per-provider plugins, automatic detection, fallback strategies, cost tracking, and native tool calling. This is the most mature module in roachie's RAG pipeline.

---

### 9. memory/ — Optional Chat History

**What it prescribes:** A memory layer that maintains conversation context across turns, enabling multi-turn interactions, context-aware follow-ups, and session persistence.

**Pros — What roachie does well:**
- **In-session conversation history**: `_nl_messages_json` maintains a JSON array of `{role, content}` message pairs across turns within a session
- **History windowing**: capped at last 10 turns (20 messages) to prevent context overflow, with oldest messages dropped first
- **Agent loop context preservation**: during multi-step execution, keeps last 6 messages (3 turns) to maintain continuity across iterations
- **Persistent learning memory**: JSONL failure/success databases (30-day TTL) inject cross-session patterns into prompts — "MISTAKES TO AVOID" and "SUCCESSFUL PATTERNS"
- **Schema caching**: cluster topology and database schema cached per session, avoiding redundant SQL queries
- **History truncation with context summary**: when dropping old messages, preserves a summary of the dropped context

**Cons — What's missing:**
- **No cross-session conversation history** — each REPL session starts with empty `_nl_messages_json`. Previous conversations are lost
- **No long-term memory** beyond failure/success patterns — the system doesn't remember "user frequently asks about movr database" or "user prefers JSON format"
- **No memory retrieval** — conversation history is prepended wholesale, not selectively retrieved based on relevance to the current query
- **No memory summarization** — old messages are dropped, not compressed into summaries that preserve key facts
- **No episodic memory** — can't recall "last time you asked about replication lag, the issue was node 3 being down"

**Room to improve:**
- Add session summary persistence: at session end, save a 3-sentence summary of what was discussed, load it at next session start
- Add user preference memory: track frequently used databases, preferred output formats, common query patterns
- Add relevant history retrieval: instead of prepending all history, embed each turn and retrieve only turns relevant to the current query
- Keep cross-session memory lightweight — a SQLite table with (timestamp, summary, key_entities) would suffice

**Score: 5.5/10** — In-session history works with sensible windowing. Persistent learning (failure/success patterns) provides basic cross-session memory. Missing: cross-session conversation history, user preference memory, and selective memory retrieval.

---

## Summary Scorecard

| # | Module | Score | LangChain Equivalent | Roachie Implementation |
|---|--------|-------|---------------------|----------------------|
| 1 | loaders/ | 5/10 | DocumentLoader, CSVLoader, WebLoader | Live SQL + flat files + shell exec (no abstraction) |
| 2 | splitters/ | 7/10 | RecursiveCharacterTextSplitter | Domain-natural atomic chunks (no splitting needed) |
| 3 | embeddings/ | 8/10 | OpenAIEmbeddings, HuggingFace | 3 providers, pre-computed, auto-detection, fallback |
| 4 | vectorstore/ | 6/10 | FAISS, Chroma, Pinecone | Flat JSON + jq cosine similarity (right-sized) |
| 5 | retriever/ | 9/10 | EnsembleRetriever, BM25 + Dense | Hybrid regex+semantic, intent gating, 99%+ accuracy |
| 6 | prompts/ | 8/10 | PromptTemplate, ChatPromptTemplate | 8 modular templates, conditional loading, 2 variants |
| 7 | chains/ | 7/10 | RetrievalQA, ConversationalRetrievalChain | Enrichment → LLM → execution pipeline + agent loop |
| 8 | llms/ | 9/10 | ChatOpenAI, ChatAnthropic | 5 providers, unified dispatcher, native tool calling |
| 9 | memory/ | 5.5/10 | ConversationBufferWindowMemory | In-session history + persistent learning, no cross-session |

**Overall RAG Pipeline Score: 7.2/10** — "Production Pipeline in Shell Scripts"

---

## Key Insight

> Roachie implements all 9 modules of a LangChain-style RAG pipeline entirely in bash — no Python, no frameworks, no vector databases. The retriever (9/10) and LLM layer (9/10) are genuinely production-grade, achieving what typically requires LangChain + FAISS + multiple Python packages. The system succeeds because its domain is narrow (77 tools, 14 doc chunks) and its data is naturally structured (each tool is one chunk, each topic is one chunk), eliminating the need for sophisticated splitters or vector databases. The weakest modules — loaders (5/10) and memory (5.5/10) — are the expected gaps for a CLI tool that wasn't designed as a data platform. The honest question isn't "should roachie adopt LangChain?" (no — the bash implementation is simpler, faster, and has zero dependencies) but rather "which LangChain patterns are worth stealing?" Answer: query expansion for the retriever, token counting for prompts, and session summary persistence for memory.

---

## Architecture Comparison

```
LangChain RAG Pipeline          Roachie Equivalent
──────────────────────          ─────────────────────────────────────────────
loaders/                   →    _nl_fetch_schema_context() + flat file reads
  PDFLoader                     (not implemented)
  CSVLoader                     CSV metrics reader in llm_metrics.sh
  WebBaseLoader                 (not implemented)
  SQLDatabaseLoader             SHOW DATABASES + information_schema queries

splitters/                 →    Domain-natural atomicity (no splitting needed)
  RecursiveCharacterSplitter    (not needed — tools are already atomic)
  TokenTextSplitter             (not needed — doc chunks are manually sized)

embeddings/                →    3-provider embedding layer
  OpenAIEmbeddings              tools/embeddings/openai_*.json
  HuggingFaceEmbeddings         tools/embeddings/ollama_*.json (nomic-embed)
  GoogleEmbeddings              tools/embeddings/gemini_*.json

vectorstore/               →    Flat JSON + jq cosine similarity
  FAISS                         (not needed at 77 vectors)
  Chroma                        (not needed at 77 vectors)
  Pinecone                      (not needed at 77 vectors)

retriever/                 →    Hybrid regex + semantic matching
  BM25Retriever                 regex_keywords.tsv (keyword matching)
  EnsembleRetriever             _enrich_system_prompt_with_tool_help()
  SimilaritySearch              _semantic_match_tools() via jq

prompts/                   →    8 template files + 2 prompt builders
  PromptTemplate                tools/utils/prompts/*.txt
  ChatPromptTemplate            _nl_build_system_prompt[_compact]()

chains/                    →    Sequential function pipeline + agent loop
  RetrievalQA                   enrichment → LLM → JSON parse → execute
  ConversationalRetrieval       agent loop with needs_followup chaining

llms/                      →    5-provider unified dispatcher
  ChatOpenAI                    src/lib/providers/openai.sh
  ChatAnthropic                 src/lib/providers/anthropic.sh
  ChatGoogleGenerativeAI        src/lib/providers/gemini.sh
  Ollama                        src/lib/providers/ollama.sh

memory/                    →    In-session JSON array + JSONL learning files
  ConversationBufferWindow      _nl_messages_json (last 10 turns)
  ConversationSummary           (not implemented)
  EntityMemory                  (not implemented)
```

---

## Top 5 Improvements by Impact

| # | Improvement | Module | Effort |
|---|------------|--------|--------|
| 1 | **Add token counting to prompt assembly** — warn before exceeding model context window | prompts/ | Low — estimate tokens from character count |
| 2 | **Add session summary persistence** — save 3-sentence summary at session end, load at next start | memory/ | Medium — write summary to file, load at init |
| 3 | **Add per-stage latency tracking** — log retrieval, enrichment, LLM call, execution times separately | chains/ | Low — wrap each stage with timing |
| 4 | **Add query normalization** — lowercase, expand abbreviations, remove stop words before retrieval | retriever/ | Low — add preprocessing step |
| 5 | **Add web loader for CockroachDB docs** — automate knowledge base updates from release notes | loaders/ | Medium — curl + HTML parsing script |

---

## What NOT to Do

| Anti-Pattern | Why |
|-------------|-----|
| Replace bash with LangChain/Python | The bash implementation has zero dependencies, starts in <100ms, and achieves 99%+ retrieval accuracy — LangChain would add complexity without improving results |
| Add FAISS/Chroma for 77 vectors | Brute-force cosine similarity over 77 vectors takes <1ms — a vector DB adds 50-200ms startup latency |
| Add RecursiveCharacterTextSplitter | Tool descriptions are already atomic chunks — splitting would degrade retrieval quality |
| Add ConversationSummaryMemory | For a CLI tool, session-scoped memory with persistent learning is sufficient — LLM-based summarization per turn is expensive |
| Add a chain framework (LangGraph, CrewAI) | Sequential bash functions are simpler, faster, and more debuggable than a graph-based chain framework |

---

## CLI Summary

**RAG Pipeline Module Map**

```
Table 1: Module-by-Module Assessment

Module         Score    LangChain Analog              Roachie Implementation
───────────    ─────    ──────────────────────────    ──────────────────────────────────
loaders/       5/10     DocumentLoader family         Live SQL + flat files + shell exec
splitters/     7/10     RecursiveCharacterSplitter    Domain-natural atomic chunks
embeddings/    8/10     OpenAI/HF/Google Embeddings   3 providers, pre-computed, fallback
vectorstore/   6/10     FAISS / Chroma / Pinecone     Flat JSON + jq cosine similarity
retriever/     9/10     EnsembleRetriever             Hybrid regex+semantic, 99%+ accuracy
prompts/       8/10     PromptTemplate                8 modular files, 2 variants
chains/        7/10     RetrievalQA chain             Enrichment→LLM→execute + agent loop
llms/          9/10     ChatOpenAI / ChatAnthropic    5 providers, unified dispatcher
memory/        5.5/10   ConversationBufferWindow      In-session history + JSONL learning
───────────    ─────    ──────────────────────────    ──────────────────────────────────
OVERALL        7.2/10                                 "Production Pipeline in Shell Scripts"
```

```
Table 2: Module Tiers

Tier          Modules                          Avg Score
────────────  ───────────────────────────────  ─────────
Excellent     retriever/, llms/                9.0
Strong        embeddings/, prompts/            8.0
Good          splitters/, chains/              7.0
Adequate      vectorstore/                     6.0
Basic         loaders/, memory/                5.3
```

**Key Insight:** Roachie implements all 9 LangChain RAG modules in pure bash with zero Python dependencies. The retriever (9/10) and LLM layer (9/10) are genuinely production-grade. The system works because its domain is narrow and data is naturally structured — 77 atomic tool chunks don't need a text splitter, and 91 total vectors don't need a vector database. The right question isn't "adopt LangChain?" but "which patterns to steal?" — token counting, session summaries, and query normalization.

**Top 3 Quick Wins:**
1. Add token counting to prompt assembly — prevent context overflow (prompts/)
2. Add per-stage latency tracking — identify pipeline bottlenecks (chains/)
3. Add query normalization — expand abbreviations, remove stop words (retriever/)

**The Zero-Dependency Advantage:**
- LangChain RAG: Python + pip install langchain + FAISS/Chroma + embedding SDK = ~500MB, 5-10s startup
- Roachie RAG: bash + jq + curl = ~5MB, <100ms startup, 99%+ retrieval accuracy
- At 77 tools, the bash implementation wins on every dimension except extensibility
