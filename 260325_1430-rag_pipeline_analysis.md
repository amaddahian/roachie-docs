# Roachie RAG Pipeline Analysis -- Evaluation Against Standard RAG Framework

**Date:** 2026-03-25
**Scope:** Evaluation of roachie's AI/NL subsystem against the 6-stage RAG pipeline framework (Document Processing, Embedding Generation, Vector Storage, Retrieval, Response Generation, Evaluation)

---

## Context

Roachie is a **domain-specific RAG system** for CockroachDB administration. Unlike general-purpose RAG pipelines that retrieve from arbitrary document corpora, roachie retrieves from three structured knowledge sources:

1. **Tool catalog** -- 77 `cr_*` CLI tools with curated descriptions and `--help` docs
2. **CockroachDB knowledge base** -- 14 curated doc chunks on operational topics (151 lines)
3. **Live cluster state** -- Database schema, topology, and connection parameters fetched at runtime

This domain specificity shapes every stage of the pipeline.

---

## 1. Document Processing

### What Roachie Does

| Aspect | Implementation | File |
|--------|---------------|------|
| **Source documents** | 77 tool descriptions (1-2 lines each) + 14 knowledge base chunks (100-300 words each) | `tool_descriptions.txt`, `crdb_knowledge_base.txt` |
| **Chunking strategy** | Manual, hand-curated chunks delimited by `---` markers | `crdb_knowledge_base.txt` |
| **Text cleaning** | Tool descriptions augmented with real-world query phrases for 16 tools | `tool_descriptions.txt` |
| **Metadata extraction** | Each chunk has `CHUNK_ID` and `TITLE` fields parsed at embedding time | `generate_doc_embeddings.sh:69-95` |
| **Structure preservation** | Tools: flat name/description pairs. Docs: title/content hierarchy | Both generation scripts |
| **Live schema** | `_nl_fetch_schema_context()` queries cluster for databases, tables, columns -- sanitized via `tr -cd` | `llm_prompt.sh:1059-1137` |

### Pros

- **Hand-curated quality** -- Every tool description and doc chunk is written by the project author, not auto-extracted. This eliminates the noise and fragmentation problems that plague automated chunking of PDFs/HTML.
- **Query phrase augmentation** -- 16 high-traffic tools have real-world user phrases baked into descriptions (e.g., cr_columns includes "show me the list of columns for a table, what columns does a table have"). This closes the vocabulary gap between user intent and tool metadata.
- **Chunk size consistency** -- Doc chunks are uniformly 100-300 words, fitting well within embedding model context windows. No "too short" (under-specified) or "too long" (diluted) chunks.
- **Live context injection** -- Schema context fetched from the actual cluster is a form of real-time document processing that most RAG systems lack.
- **Metadata is structured** -- `CHUNK_ID` and `TITLE` enable direct referencing in retrieval results.

### Cons

- **Tiny corpus** -- Only 14 doc chunks and 77 tool descriptions. This is not a scalability challenge but a coverage gap. Many CockroachDB operational topics are not represented (e.g., multi-region setup, import/export, workload tuning, authentication methods).
- **No automated ingestion pipeline** -- Adding new docs requires manually editing `crdb_knowledge_base.txt` and re-running `generate_doc_embeddings.sh`. There is no web scraper, PDF parser, or automated chunker.
- **No overlap/sliding window** -- Chunks are strictly delimited with no content overlap between adjacent chunks. If a user query spans two topics (e.g., "how does GC TTL affect changefeeds?"), the retrieval must match both chunks independently.
- **Schema context is ephemeral** -- Cached per session only. No persistent schema history for tracking drift or changes over time.
- **No document versioning** -- Knowledge base content has no version tracking. If CockroachDB behavior changes between versions, stale chunks could provide incorrect guidance.

### Room to Improve

1. **Expand knowledge base to 50-100 chunks** covering all major CockroachDB operational topics. Priority: multi-region, import/export, workload tuning, security/auth, upgrade procedures, SQL tuning patterns.
2. **Add overlapping context** -- Include 1-2 sentences of related topic context at the end of each chunk (e.g., "See also: changefeed_troubleshooting" in the gc_ttl_tuning chunk) so the embedding captures cross-topic relationships.
3. **Automate doc ingestion** from CockroachDB's official docs (markdown source available on GitHub). Use a chunker that respects heading structure and produces 200-400 word chunks with 50-word overlap.
4. **Version-tag chunks** with the CockroachDB version they apply to (e.g., `VERSION: v23.2+` for READ COMMITTED content), enabling version-aware retrieval.

---

## 2. Embedding Generation

### What Roachie Does

| Aspect | Implementation | File |
|--------|---------------|------|
| **Models** | 3 providers: Ollama/nomic-embed-text (768-dim), OpenAI/text-embedding-3-small (1536-dim), Gemini/gemini-embedding-001 (3072-dim) | `generate_embeddings.sh` |
| **Batch processing** | Sequential per-tool embedding via curl (77 API calls per provider) | `generate_embeddings.sh:236-278` |
| **Pre-computation** | Embeddings generated offline, checked into git, shipped with releases | All 3 JSON files in `tools/embeddings/` |
| **Quality evaluation** | 135-query test suite (71 multi-tenant + 64 single-tenant) comparing regex vs semantic accuracy | `test_embeddings.sh` |
| **Dimensionality** | 768 (Ollama), 1536 (OpenAI), 3072 (Gemini) -- no dimensionality reduction | Embedding JSON files |
| **Magnitude pre-computation** | Each tool embedding includes pre-computed L2 magnitude for faster cosine similarity | `generate_embeddings.sh:275` |
| **Embedding reuse** | Query embedding computed once per query, cached and reused for both tool matching and Doc RAG | `llm_prompt.sh:194-202, 520-523` |

### Pros

- **Multi-provider support** -- Three embedding providers with automatic detection and graceful fallback. Users aren't locked into a single provider or API key requirement.
- **Pre-computed and shipped** -- Embeddings are generated offline and checked into git. Zero embedding cost at runtime for tool matching (only query embedding costs). No cold-start latency for new users.
- **Magnitude pre-computation** -- The `"magnitude"` field in each tool's embedding eliminates redundant sqrt computation during cosine similarity (77 sqrt operations saved per query).
- **Embedding reuse across retrieval stages** -- Query embedding is computed once and reused for both tool matching and Doc RAG. This saves 100-500ms and one API call per query.
- **Strong evaluation discipline** -- The 135-query test suite with per-provider accuracy breakdowns is more rigorous than most production RAG systems. The combined accuracy (99-100%) validates embedding quality.
- **Purpose-focused descriptions** -- Embeddings are generated from curated descriptions, not raw `--help` text. This focuses the embedding on semantic intent rather than CLI syntax noise.
- **Local option** -- Ollama/nomic-embed-text provides a free, private, zero-latency embedding path. Most RAG systems require cloud API access.

### Cons

- **No batch API usage** -- Embeddings are generated one-at-a-time via sequential curl calls. OpenAI and Gemini both support batch embedding APIs that process multiple inputs in a single request, which would be 10-20x faster for regeneration.
- **No embedding quality metrics beyond accuracy** -- The test suite measures "correct tool in top-7" but doesn't evaluate embedding space properties (cluster separation, intra-class vs inter-class distance, nearest-neighbor consistency). Poor embedding quality might be masked by the regex fallback.
- **No dimensionality reduction** -- Gemini embeddings are 3072-dim (5MB file for 77 tools). While this is manageable at 77 tools, it would become unwieldy at 500+ tools. PCA or Matryoshka truncation could reduce to 256-512 dims with minimal accuracy loss.
- **Single embedding per tool** -- Each tool has one embedding from its description. Tools with multiple use cases (e.g., `cr_backup` handles backup, restore, schedule, and list) get a single averaged semantic vector. Multi-vector representations could improve matching for multi-function tools.
- **No embedding drift detection** -- If the embedding model is updated (e.g., nomic-embed-text v1 to v2), pre-computed embeddings become stale. No version tracking or staleness warning.
- **Query embedding not cached across queries** -- While the embedding is reused within a single query (tool match + Doc RAG), it is not cached across session queries. Repeated similar queries re-compute the same embedding.

### Room to Improve

1. **Use batch embedding APIs** -- OpenAI supports up to 2048 inputs per request. Send all 77 descriptions in one call instead of 77 sequential calls. This reduces regeneration time from ~30s to ~2s.
2. **Add embedding space evaluation** -- Compute inter-cluster distance metrics: are similar tools (cr_ddl_table, cr_ddl_view, cr_ddl_database) closer to each other than to unrelated tools (cr_backup, cr_health)? Visualize with t-SNE or UMAP to identify clustering issues.
3. **Consider multi-vector representations** -- For multi-function tools (cr_backup, cr_ceph_storage), generate embeddings from multiple description variants and store the highest-similarity match. ColBERT-style late interaction could also apply here.
4. **Add model version tracking** -- Include the embedding model version hash in the JSON metadata. Warn at startup if the local model version differs from the version used to generate embeddings.
5. **Implement a session-level embedding cache** -- Cache query embeddings in an associative array keyed by normalized query text. If the same (or very similar) query is repeated, skip the API call.

---

## 3. Vector Storage

### What Roachie Does

| Aspect | Implementation | File |
|--------|---------------|------|
| **Storage format** | Flat JSON files (one per provider) | `tools/embeddings/*.json` |
| **Indexing** | None -- linear scan via jq over all entries | `llm_prompt.sh:210-220` |
| **Scale** | 77 tool vectors + 14 doc chunk vectors | 6 JSON files total (~11MB) |
| **Update strategy** | Full regeneration via `generate_embeddings.sh` | Manual script execution |
| **Persistence** | Checked into git, shipped with releases | `.json` files in repo |

### Pros

- **Zero dependencies** -- No vector database required. No pgvector, no Pinecone, no FAISS, no ChromaDB. The entire vector store is a JSON file parsed by jq. This is a massive operational simplification for a CLI tool.
- **Atomic updates** -- Regeneration produces a complete new file via `mv` (atomic on POSIX). No partial index corruption possible.
- **Version-specific overrides** -- The `_nl_resolve_resource()` function supports `tools/embeddings/25.4/` version-specific embedding files with fallback to shared ones. This enables version-aware tool matching without duplicating the full set.
- **Portable** -- Embeddings ship with the codebase. No external database connection, no cloud service, no running daemon. Works offline, works on air-gapped systems.
- **Size is manageable** -- 11MB total for all 6 files (3 tool + 3 doc). Fits comfortably in git, loads fast.

### Cons

- **Linear scan (O(n))** -- Every query scans all 77 tools (or 14 doc chunks) via jq. While fast at current scale (~100-200ms), this becomes O(n) with corpus growth. At 500+ tools, jq parsing alone could take 1-2 seconds.
- **No approximate nearest neighbor (ANN)** -- No HNSW, IVF, or LSH indexing. Every similarity computation is exact (brute-force cosine). This is fine at n=77 but doesn't scale.
- **jq as vector math engine** -- jq is a JSON processor, not a numerical computation library. Dot products are computed via `[range(0;n)] | map(a[.] * b[.]) | add` which is interpreted, not SIMD-optimized. Python/numpy or even awk would be 10-100x faster for the same operation.
- **No incremental updates** -- Adding one new tool requires regenerating all 77 embeddings (or at minimum, rebuilding the JSON file). No append-only or delta update mechanism.
- **No metadata filtering** -- Cannot filter vectors by metadata before similarity search (e.g., "only match monitoring tools" or "only match tools available in v25.4"). All 77 vectors are compared every time.
- **No persistence of query embeddings** -- Computed query embeddings are stored in temp files and discarded at session end. A persistent cache could avoid re-computation for common queries.

### Room to Improve

1. **Short-term: Pre-filter by regex before vector scan** -- This is already implemented (regex acts as intent gate). The semantic scan is skipped entirely when regex returns 5+ results or 0 results. This is the right optimization for the current scale.
2. **Medium-term: SQLite + vector extension** -- Replace JSON files with SQLite using the `sqlite-vec` extension (or CockroachDB itself via pgvector). This provides indexed ANN search, metadata filtering, and incremental updates while remaining a single-file database.
3. **Long-term: Consider FAISS or Hnswlib** -- If the corpus grows to 500+ entries, a purpose-built ANN index would provide sub-millisecond retrieval. Python bindings are lightweight and could be called from bash via a small Python helper.
4. **Add metadata filtering** -- Include tool category metadata (monitoring, schema, security, performance) in the embedding file. Allow pre-filtering by category before similarity search to reduce false positives.
5. **Support incremental updates** -- Allow `generate_embeddings.sh --add cr_new_tool` to append a single tool's embedding without regenerating all 77. Requires JSON append logic (jq can handle this).

---

## 4. Retrieval Process

### What Roachie Does

| Aspect | Implementation | File |
|--------|---------------|------|
| **Primary retrieval** | Regex keyword matching (TSV-based, bash `[[ =~ ]]`) | `llm_prompt.sh:309-403` |
| **Secondary retrieval** | Cosine similarity against pre-computed embeddings via jq | `llm_prompt.sh:170-221` |
| **Combining strategy** | Regex as intent gate, semantic as priority ranker, union + dedup + top-7 | `llm_prompt.sh:405-460` |
| **K selection** | Top 7 tools (configurable via `NL_MAX_ENRICHED_TOOLS`) | `llm_config.sh:35` |
| **Minimum threshold** | Cosine similarity >= 0.35 (`NL_MIN_SIMILARITY`) | `llm_config.sh:37` |
| **Doc retrieval** | Cosine similarity against 14 doc chunks, top 3, min similarity 0.45 | `llm_prompt.sh:231-294` |
| **Re-ranking** | Regex match count (multi-group bonus), semantic score, combined priority | `llm_prompt.sh:389-403, 440-460` |
| **Short-circuit** | If regex finds 5+ matches, skip semantic entirely (high-confidence gate) | `llm_prompt.sh:409-412` |
| **Special patterns** | `&ceph_gate`, `&!ceph_gate`, `&dual:PATTERN` for conditional matching | `llm_prompt.sh:340-374` |

### Pros

- **Hybrid retrieval is the standout strength** -- The regex-first, semantic-second architecture is a sophisticated retrieval strategy that outperforms either method alone:
  - Regex provides **high precision** (exact keyword matches have near-zero false positives)
  - Semantic provides **high recall** (catches synonyms, paraphrases, and intent variations that regex misses)
  - The union + dedup + top-7 fusion achieves 99-100% combined accuracy across 135 test queries
- **Intent gating prevents hallucinated retrieval** -- If regex finds nothing, semantic is skipped entirely. This prevents the "semantic matching always returns something" problem where cosine similarity pulls in tangentially related results for off-topic queries.
- **High-confidence short-circuit** -- When regex matches 5+ tools, the query intent is unambiguous. Skipping semantic saves 100-200ms without accuracy loss.
- **Multi-group scoring** -- Tools that match multiple keyword groups get higher scores. This is a simple but effective relevance ranking that approximates BM25-style term frequency.
- **Conditional matching** -- The `&ceph_gate`, `&!ceph_gate`, and `&dual:` prefixes enable context-sensitive retrieval that pure vector similarity cannot achieve (e.g., "storage" means Ceph tools only when "ceph" or "s3" is also mentioned).
- **Tool help enrichment** -- Retrieved tools get their full `--help` documentation injected into the system prompt. This is a form of chunk expansion that gives the LLM complete context, not just the matched description.
- **Parallel help fetching** -- Tool `--help` docs are fetched in parallel background jobs and cached per-session.

### Cons

- **No BM25 or TF-IDF** -- The regex matcher is binary (match/no match per keyword group). It doesn't weight by term frequency, inverse document frequency, or field length. A tool description that mentions "query" once scores the same as one that is entirely about queries.
- **No learned re-ranking** -- The combination of regex + semantic uses a simple priority rule (semantic first, regex supplement). A learned re-ranker (cross-encoder or reciprocal rank fusion with learned weights) could improve precision in ambiguous cases.
- **No query expansion** -- User queries are matched as-is. No synonym expansion, no query rewriting, no hypothetical document embedding (HyDE). The query phrase augmentation in descriptions partially compensates, but query-side expansion would help more.
- **Similarity threshold (0.35) is very low** -- At 0.35, nearly half of all tools will exceed the threshold for any query. The top-7 cutoff is doing most of the filtering work, not the similarity threshold. This means the threshold provides almost no noise reduction.
- **No negative feedback loop** -- When the user provides negative feedback ("wrong tool"), this information is stored in persistent learning but is not used to adjust retrieval scores. The same wrong tool will be retrieved again for the same query.
- **Doc RAG competes with tool matching for prompt space** -- Both tool docs and knowledge base chunks are injected into the system prompt. With 7 tools + 3 doc chunks, the prompt can grow significantly, especially for Ollama's 16K context window.
- **No cross-encoder re-ranking** -- After initial retrieval, there is no second-stage re-ranking using a more expensive model (cross-encoder). The initial cosine similarity score is the final score.

### Room to Improve

1. **Implement reciprocal rank fusion (RRF)** -- Instead of "semantic first, regex supplement," use RRF to combine the two ranked lists. RRF assigns scores as `1/(k + rank)` for each list and sums them. This would give tools that appear in both lists a significant boost over tools in only one.
2. **Raise similarity threshold to 0.50** -- The current 0.35 is effectively no filter. At 0.50, only genuinely similar tools pass, and the top-7 cutoff becomes a tiebreaker rather than the primary filter.
3. **Add query expansion for Ollama** -- For local models without strong paraphrase understanding, expand the query with common synonyms before matching (e.g., "table" -> "table schema columns," "slow" -> "slow performance latency query").
4. **Implement negative feedback integration** -- When a user marks a result as incorrect, reduce the retrieval score for that tool/query pair. Store as a negative bias in the persistent learning DB and subtract from cosine similarity during retrieval.
5. **Add retrieval-aware token budgeting** -- Track the total tokens consumed by retrieved content (tool docs + knowledge base chunks) and cap at a percentage of the context window (e.g., 40%). If the budget is exceeded, truncate the least-relevant chunk.

---

## 5. Response Generation

### What Roachie Does

| Aspect | Implementation | File |
|--------|---------------|------|
| **LLM providers** | 5 providers: Anthropic Claude, OpenAI GPT-4/5, Gemini 2.5, Vertex AI, Ollama (local LoRA) | `src/lib/providers/*.sh` |
| **Prompt engineering** | Role-based system prompt with 7 template files (parameter rules, execution rules, few-shot examples, tool catalog, connection rules) | `llm_prompt.sh`, `tools/utils/prompts/` |
| **Context window** | Per-provider limits: 200K (Anthropic), 128K (OpenAI), 1M (Gemini), 16K (Ollama) | `llm_config.sh:24-32` |
| **History management** | Drop oldest 2 messages at 80% context, preserve sticky context (cluster/db/tenant/host) | `llm_assistant.sh:556-608` |
| **Response format** | Structured JSON: `{reasoning, user_message, commands[], needs_followup}` | System prompt template |
| **Agent loop** | Max 3 iterations with reflexion (error-driven self-correction) | `llm_assistant.sh:1381-1533` |
| **Command validation** | 6-layer: whitelist, normalize, metachar, SQL mutation, tool existence, RBAC | `llm_exec.sh` |
| **Streaming** | SSE streaming for cloud providers with progress dots | All cloud provider plugins |
| **Personality** | 3 modes: roachie (casual), roachman (professional), default (balanced) | `llm_prompt.sh` |
| **Temperature** | 0.2 default (low creativity, high consistency); configurable via `/temp` | `llm_assistant.sh:1260` |

### Pros

- **Exceptional prompt engineering** -- The system prompt is a masterclass in domain-specific prompt design:
  - Structured JSON response format eliminates parsing ambiguity
  - Few-shot examples with explicit "WRONG" counter-examples prevent common mistakes
  - Parameter rules with positive AND negative examples ("use --insecure" AND "NEVER use just --insecure for diff tools")
  - Personality modes add user engagement without compromising accuracy
- **Agent loop with reflexion** -- The 3-iteration agent loop with error-driven self-correction is a production-grade agentic pattern. When a command fails, the system automatically analyzes the error, identifies what went wrong, and tries a different approach.
- **Defense-in-depth validation** -- Even if the LLM generates a dangerous command, it is blocked by the 6-layer validation pipeline before execution. This is far more robust than relying on prompt instructions alone.
- **Context window optimization is sophisticated** -- The history trimming with sticky context preservation is clever. When old messages are evicted, connection parameters (cluster, database, tenant, host) are extracted and persisted in a summary. This prevents the LLM from "forgetting" which cluster it's working with.
- **Multi-provider seamless fallback** -- If the primary LLM fails, the system automatically offers fallback providers. In batch mode, it auto-selects the next available provider without user intervention.
- **Structured output enforced by system prompt** -- The JSON response format is specified in the system prompt and validated by the parser. Malformed responses trigger fallback parsing (regex extraction, brute-force JSON substring matching for Gemini).
- **Low temperature (0.2)** -- Correct choice for a tool-calling system where consistency matters more than creativity.

### Cons

- **No response verification against source** -- The generated response is not checked against the retrieved tool documentation for factual accuracy. If the LLM hallucates a flag name (e.g., `--format=table` when the tool only supports `--format json`), the system relies on command execution failure + reflexion to self-correct. A pre-execution verification step could catch this.
- **No confidence scoring** -- The LLM's response doesn't include a confidence estimate. The system treats all commands equally regardless of whether the LLM is certain or guessing. A confidence threshold could trigger "are you sure?" prompts for low-confidence commands.
- **No chain-of-thought verification** -- The `reasoning` field is displayed to the user but not verified against the actual command. If the reasoning says "I'll check table sizes" but the command is `cr_query_stats`, this mismatch is not detected.
- **Compact prompt for Ollama is too sparse** -- The Ollama compact prompt omits parameter rules, few-shot examples, and detailed execution rules. This explains a significant portion of Ollama's accuracy gap (90% vs 95%+ for cloud providers). The 16K context limit forces this trade-off.
- **No response caching** -- If the user asks the same question twice in a session, the full LLM call is repeated. Common informational queries (e.g., "what tools are available?") could be cached.
- **Reflexion can loop on the same error** -- While the reflexion prompt says "do NOT repeat the same command that failed," there is no programmatic enforcement. The LLM might try the same tool with slightly different flags, hitting the same error 3 times.

### Room to Improve

1. **Add pre-execution flag verification** -- Before executing a command, parse the generated flags and compare against the tool's `--help` output (already cached). If a flag doesn't exist, inject a correction before execution rather than waiting for the command to fail.
2. **Add command deduplication in reflexion** -- Track commands attempted across iterations. If the reflexion generates the same base command (`cr_tables -d movr`), force a different approach or exit the loop.
3. **Consider response caching for informational queries** -- Cache `{query_hash: response}` for queries where `commands: []`. Informational responses are deterministic and don't depend on cluster state.
4. **Add a confidence-based execution gate** -- Extract implicit confidence from the LLM's reasoning (e.g., "I think...", "this might...", "I'm not sure...") and prompt for confirmation on low-confidence commands.
5. **Expand the Ollama compact prompt** -- With 16K context and typical prompt sizes of ~2K tokens, there is room for a "medium" prompt that includes parameter rules and a few critical few-shot examples without the full tool catalog. This could bridge the 5% accuracy gap.

---

## 6. Evaluation

### What Roachie Does

| Aspect | Implementation | File |
|--------|---------------|------|
| **Accuracy testing** | 135-query test suite (multi-tenant + single-tenant) with expected tool mapping | `test_embeddings.sh` |
| **Batch testing** | `roachie-batch` runs prompts through the full pipeline with JSONL output | `bin/roachie-batch` |
| **Assertion framework** | Regex-based assertions for validating expected output patterns | `roachie-batch --assertions` |
| **Metrics logging** | 18+ field CSV per query: provider, tokens, cost, timing, complexity, tools, match method | `llm_metrics.sh` |
| **Cost tracking** | Per-query and session cost with flock-serialized accumulator and budget alerts | `llm_metrics.sh` |
| **User feedback** | Optional feedback collection (y/n/skip) stored to JSONL and CSV | `llm_assistant.sh:1028-1093` |
| **Persistent learning** | Failure/success patterns stored across sessions, injected into future prompts | `llm_metrics.sh:456-700` |
| **Debug logging** | 3 verbosity levels: basic, previews, full prompt+response | `llm_metrics.sh:420-450` |
| **Provider comparison** | Batch testing across multiple providers with accuracy comparison | `roachie-batch` with `--provider` flag |
| **Timing breakdowns** | Per-query: enrichment_ms, api_ms, execution_ms, total_ms | `llm_assistant.sh:1565-1566` |

### Pros

- **End-to-end batch testing** -- `roachie-batch` is a full-pipeline evaluation tool that runs real prompts through real LLM providers with real command execution. This is ground truth evaluation, not synthetic benchmarks.
- **Multi-provider comparison** -- The same prompt set can be run against every provider to produce comparative accuracy tables. The accuracy matrix (Gemini 100%, OpenAI 100%, Ollama 99.3%) quantifies provider quality differences.
- **Rich metrics pipeline** -- 18+ fields per query provide deep observability: which tools were matched, what method was used (regex/semantic/combined), how long each phase took, what the cost was, whether past failures were matched. This data enables data-driven optimization.
- **Persistent learning creates a feedback loop** -- Failures are stored and injected into future system prompts. This is a simple but effective form of online learning that improves accuracy over time without retraining.
- **Cost tracking with budget alerts** -- Session-level cost monitoring with configurable budget limits prevents runaway API costs. This is a practical feature most RAG systems lack.
- **User feedback collection** -- The opt-in feedback mechanism (y/n/skip after each response) feeds into both persistent learning and feedback logs, creating labeled training data for future analysis.

### Cons

- **No automated quality metrics (DeepEval, Ragas)** -- There is no automated evaluation of response quality beyond "did the command succeed." Standard RAG metrics are absent:
  - **Faithfulness** -- Does the response only use information from retrieved context? (Not measured)
  - **Answer relevance** -- Does the response actually answer the user's question? (Not measured)
  - **Context precision** -- Were the retrieved tools/docs actually relevant? (Measured only by top-7 accuracy, not per-query precision)
  - **Context recall** -- Were all relevant tools retrieved? (Not measured -- only "was the correct tool in the top 7")
- **No regression testing in CI** -- The accuracy test suite (`test_embeddings.sh`) is run manually. A code change that breaks regex keywords or embedding loading would not be caught until manual testing.
- **No A/B testing framework** -- Cannot compare two prompt versions, two enrichment strategies, or two similarity thresholds on the same query set with statistical significance.
- **Feedback collection is sparse** -- Feedback is opt-in and most users skip it. The persistent learning database has at most 15 failures and 10 successes -- too small for meaningful analysis.
- **No latency SLAs** -- While timing is tracked, there are no alerts or thresholds for degraded performance (e.g., "enrichment took >500ms, investigate").
- **No hallucination detection** -- If the LLM generates a plausible but incorrect command (e.g., correct tool with wrong flags), this is not detected unless the command fails at execution. Silent wrong answers are the hardest class of errors.
- **Batch evaluation doesn't test multi-turn** -- `roachie-batch` tests single-turn queries only. Multi-turn agent loop behavior (reflexion, follow-up, needs_followup) is untested in automated evaluation.

### Room to Improve

1. **Implement lightweight quality metrics** -- Without external frameworks, measure:
   - **Command validity rate** -- % of generated commands that pass `_validate_command()` on the first try (no reflexion needed)
   - **First-attempt success rate** -- % of commands that succeed without reflexion
   - **Tool precision@7** -- Of the 7 enriched tools, how many were actually used in the final command?
   - **Context utilization** -- Was the knowledge base doc chunk actually referenced in the LLM's reasoning?
2. **Add accuracy regression to CI** -- Run `test_embeddings.sh --mode all --regex-only` in CI. This validates regex keywords without API keys and catches regressions in the matching pipeline.
3. **Implement multi-turn batch testing** -- Extend `roachie-batch` to support conversation files with multiple exchanges per "session," testing agent loop behavior and context persistence.
4. **Add latency percentile tracking** -- Track p50, p90, p99 for enrichment, API call, and execution phases. Alert when p90 exceeds a threshold.
5. **Build a hallucination detection layer** -- After the LLM generates a command, parse the flags and compare against the tool's `--help` output. Flag any unrecognized flags or invalid flag values before execution.
6. **Increase feedback collection** -- Make feedback non-optional for the first 50 queries of each session (with easy skip via Enter). Build a labeled dataset for training data generation and quality analysis.

---

## Summary Scorecard

| RAG Stage | Roachie Score | Industry Typical | Key Strength | Key Gap |
|-----------|--------------|-----------------|--------------|---------|
| 1. Document Processing | 7.5/10 | 6/10 | Hand-curated quality; live schema injection | Tiny corpus (14 chunks); no automated ingestion |
| 2. Embedding Generation | 8.5/10 | 7/10 | 3 providers; pre-computed + shipped; 135-query evaluation | No batch APIs; no embedding space analysis |
| 3. Vector Storage | 6/10 | 7/10 | Zero dependencies; portable; git-shipped | Flat JSON + jq linear scan; no ANN indexing |
| 4. Retrieval Process | 9/10 | 7/10 | Hybrid regex+semantic with intent gating; 99-100% accuracy | No BM25/TF-IDF; no learned re-ranking; no query expansion |
| 5. Response Generation | 9.5/10 | 7/10 | Exceptional prompt engineering; agent loop with reflexion; 6-layer validation | No response verification; no confidence scoring |
| 6. Evaluation | 7/10 | 6/10 | End-to-end batch testing; persistent learning; cost tracking | No automated quality metrics; no CI regression; no multi-turn evaluation |

### Overall Assessment

Roachie is a **domain-optimized RAG system** that makes deliberate trade-offs appropriate for its scale and use case:

- **Where it excels:** Response generation (prompt engineering + agent loop + safety validation) and retrieval (hybrid matching) are production-grade, exceeding typical RAG implementations. The defense-in-depth command validation is exceptional.

- **Where it's adequate:** Embedding generation and evaluation are solid with room for growth. The multi-provider embedding support and 135-query test suite are above average.

- **Where it has room to grow:** Document processing (tiny corpus, no automated ingestion), vector storage (flat JSON, no indexing), and evaluation (no automated quality metrics, no CI regression). These are areas where the system is optimized for simplicity over scalability -- appropriate today, but would need investment if the corpus grows to 500+ entries.

The key insight is that roachie compensates for weaker document processing and vector storage with **exceptional retrieval and generation**. The hybrid regex+semantic matching, intent gating, and multi-iteration reflexion loop create a system that achieves 99%+ tool matching accuracy despite using flat JSON files and manual document curation. This is a pragmatic architecture that prioritizes reliability over generality.

---

## CLI Summary (Quick Reference)

### Scorecard

| RAG Stage | Score | Key Strength | Key Gap |
|-----------|-------|-------------|---------|
| 1. Document Processing | **7.5/10** | Hand-curated quality; live schema injection | Tiny corpus (14 doc chunks); no automated ingestion |
| 2. Embedding Generation | **8.5/10** | 3 providers; pre-computed + shipped; 135-query eval | No batch APIs; no embedding space analysis |
| 3. Vector Storage | **6/10** | Zero dependencies; portable; git-shipped | Flat JSON + jq linear scan; no ANN indexing |
| 4. Retrieval Process | **9/10** | Hybrid regex+semantic with intent gating; 99-100% | No BM25; no learned re-ranking; no query expansion |
| 5. Response Generation | **9.5/10** | Exceptional prompt engineering; agent loop; 6-layer validation | No response verification; no confidence scoring |
| 6. Evaluation | **7/10** | End-to-end batch testing; persistent learning; cost tracking | No automated quality metrics (DeepEval/Ragas); no CI regression |

### Where Roachie Excels

- **Retrieval** -- The hybrid regex+semantic matching with intent gating is the standout. Regex provides precision, semantic provides recall, and the combined approach hits 99-100% accuracy. The conditional patterns (`&ceph_gate`, `&dual:`) enable context-sensitive matching that pure vector similarity can't achieve.

- **Response Generation** -- The system prompt is a masterclass in domain-specific prompt design with few-shot examples, explicit wrong-answer counter-examples, and structured JSON output. The 3-iteration agent loop with reflexion is production-grade agentic reasoning. The 6-layer command validation is far more robust than relying on prompt instructions alone.

### Where It's Adequate but Could Grow

- **Embedding Generation** -- Multi-provider with pre-computation is strong, but sequential API calls (77 per provider) should use batch APIs. Embedding space quality isn't measured beyond top-7 accuracy.

- **Evaluation** -- Batch testing and metrics logging are above average, but lacks automated quality metrics (faithfulness, relevance, hallucination detection) and CI regression testing for the accuracy suite.

### Where the Biggest Gaps Are

- **Document Processing** -- Only 14 knowledge base chunks covering 7 topics. No automated doc ingestion, no overlap between chunks, no version tagging. Expanding to 50-100 chunks on all major CockroachDB topics would significantly improve Doc RAG quality.

- **Vector Storage** -- Flat JSON with jq linear scan works at 77 tools but won't scale. No ANN indexing, no metadata filtering, no incremental updates. SQLite with `sqlite-vec` would be a natural upgrade path that keeps the zero-external-dependency philosophy.

### Top 5 Improvements by Impact

1. **Expand knowledge base** to 50-100 curated chunks (biggest Doc RAG quality gain)
2. **Add pre-execution flag verification** against tool `--help` (catch hallucinated flags before execution fails)
3. **Implement reciprocal rank fusion** for combining regex + semantic scores (better than simple priority ordering)
4. **Add accuracy regression to CI** with `test_embeddings.sh --regex-only` (prevent silent drift)
5. **Use batch embedding APIs** for 10-20x faster regeneration (operational improvement)

The key insight: roachie compensates for weaker document processing and vector storage with **exceptional retrieval and generation**. This is a pragmatic architecture that prioritizes reliability over generality -- appropriate for a 77-tool domain-specific system.
