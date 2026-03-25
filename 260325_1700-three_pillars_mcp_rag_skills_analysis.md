# Roachie: Three Pillars Analysis -- MCP, RAG, and Skills

**Date:** 2026-03-25
**Scope:** Evaluate roachie against the "3 Core Pillars of Every AI Agent's Context" framework -- MCP for tool integration, RAG for knowledge retrieval, and Skills for token-efficient reusable actions.

---

## Framework Recap

Every AI agent struggles with three core problems:

| Problem | Solution | Without It | With It |
|---------|----------|-----------|---------|
| Connecting to external tools requires custom API code every time | **MCP** | Every new tool = new custom code | One standardized protocol |
| Answering accurately from knowledge it was never trained on | **RAG** | Agents confidently hallucinate | Retrieve first, then reason |
| Repeating the same instructions in prompts wastes tokens | **Skills** | Bloated prompts, wasted tokens | Load only what's needed, when needed |

---

## Roachie's Coverage Map

| Pillar | Implemented? | Implementation | Maturity |
|--------|-------------|----------------|----------|
| MCP | **Yes** | 39 tools via Google GenAI Toolbox + native tool calling schema conversion | Production |
| RAG | **Yes** | Hybrid regex+semantic tool RAG + Doc RAG (14 chunks) + live schema injection | Production |
| Skills | **Partially** | Dynamic prompt enrichment + modular prompt templates, but no formal skill manager | Emerging |

**Key finding:** Roachie addresses all three problems, but solves them through different mechanisms than the framework prescribes. MCP is implemented as a standard protocol. RAG is a custom hybrid system (stronger than basic RAG). Skills are solved architecturally through dynamic prompt construction rather than a formal skill manager -- effective but not token-optimal.

---

## Pillar 1: MCP (Model Context Protocol)

**The problem:** Connecting to external tools requires writing custom API code every time.
**The solution:** A standardized protocol so the agent plugs into any server through one interface.

### How Roachie Implements MCP

Roachie has **two layers** of tool integration:

#### Layer 1: Native CLI Tool Execution (Primary)

The core system doesn't use MCP internally -- it uses a custom tool integration that predates MCP:

| Component | Implementation | File |
|-----------|---------------|------|
| Tool registry | 77 cr_* bash CLI tools with curated descriptions | `tools/utils/tool_descriptions.txt` |
| Tool discovery | Hybrid regex + semantic matching (99-100% accuracy) | `llm_prompt.sh:300-540` |
| Tool documentation | `--help` output fetched in parallel, injected per query | `llm_prompt.sh:496-516` |
| Tool validation | 6-layer pipeline (whitelist, normalize, metachar, SQL, existence, RBAC) | `llm_exec.sh:1-336` |
| Tool execution | Safe array-based invocation against live CockroachDB | `llm_assistant.sh:861-1021` |

#### Layer 2: MCP Standard Protocol (External Integration)

Roachie also exposes tools via the MCP standard for external agent consumption:

| Component | Implementation | File |
|-----------|---------------|------|
| MCP server | Google GenAI Toolbox configuration | `tools/mcp/tools.yaml` |
| Tool count | 39 of 77 tools (SQL-expressible subset) | `tools/mcp/tools.yaml` |
| Toolsets | 5 groups: schema (8), monitoring (11), health (13), security (5), statistics (3) | `tools/mcp/tools.yaml:48-90` |
| Data sources | 2: `crdb` (default) + `crdb-system` (system tenant) | `tools/mcp/tools.yaml:19-45` |
| Native tool calling | MCP schema -> Anthropic/OpenAI/Gemini format conversion | `llm_providers.sh:548-611` |

#### Layer 3: Native Tool Calling Bridge

When `NL_NATIVE_TOOLS=1`, roachie converts its tool schemas to provider-native formats:

```
tools/schemas/tools.json (MCP format)
  → _nl_mcp_to_anthropic()   → Anthropic tool_use format
  → _nl_mcp_to_openai()      → OpenAI function calling format
  → _nl_mcp_to_gemini()      → Gemini functionDeclarations format
```

This means the LLM can invoke tools through structured function calling rather than generating command strings in JSON.

### Pros

- **Dual integration model** -- Roachie has both an internal tool system (77 bash tools with custom matching) and an external MCP interface (39 SQL-based tools via GenAI Toolbox). Internal tools are richer (bash orchestration, multi-step logic); MCP tools are standardized and portable.

- **MCP makes roachie's tools available to any agent** -- Claude Desktop, custom LangChain agents, or any MCP-compatible client can use roachie's 39 SQL tools without knowing anything about roachie's internals. This decouples tool capability from the roachie orchestrator.

- **Schema conversion supports three providers** -- The native tool calling bridge converts MCP schemas to Anthropic, OpenAI, and Gemini formats via dedicated functions (`_nl_mcp_to_anthropic()`, `_nl_mcp_to_openai()`, `_nl_mcp_to_gemini()`). This avoids per-provider custom integration code -- exactly the problem MCP solves.

- **Toolset organization** -- The 39 MCP tools are organized into 5 logical toolsets (schema, monitoring, health, security, statistics). Clients can request specific toolsets rather than receiving all 39 tools, reducing context overhead.

- **Two data sources** -- The `crdb` and `crdb-system` sources separate default-tenant and system-tenant queries. System-only tools (cr-list-tenants, cr-health-nodes, cr-replication-lag) use `crdb-system` automatically.

### Cons

- **Only 39 of 77 tools are MCP-exposed** -- Complex tools requiring bash orchestration (cr_backup, cr_ddl_table, cr_create_cluster, cr_monitor, cr_watch, cr_upgrade_version, cr_migrate) are excluded because MCP Toolbox only supports SQL-expressible operations. This means external agents get a reduced capability set.

- **MCP is not the primary execution path** -- The internal pipeline (prompt -> LLM -> JSON -> validate -> execute bash tool) doesn't go through MCP. MCP is an external-facing interface, not the internal architecture. This means the internal system carries its own custom integration burden -- the very problem MCP is designed to solve.

- **No MCP client in roachie** -- Roachie is an MCP *server* (exposing tools) but not an MCP *client* (consuming other MCP servers). It can't connect to external MCP services (Slack, Brave Search, databases) through the standard protocol. All external integrations would require custom code.

- **No dynamic tool registration** -- The MCP tool set is statically defined in `tools.yaml`. New tools require manual YAML editing. There's no API to register/deregister tools at runtime.

- **Ollama doesn't support native tool calling** -- When using local models, the system falls back to prompt-based tool selection (generating tool names in JSON text) rather than structured function calling. The MCP schema conversion only benefits cloud providers.

### Room to Improve

1. **Make MCP the internal execution path** -- Instead of the custom bash tool execution pipeline, route all tool calls through MCP internally. This means:
   - Tool matching produces an MCP tool call (not a bash command string)
   - Tool execution goes through the MCP server (not direct bash invocation)
   - Tool results come back as structured MCP responses (not raw stdout)
   - Benefit: one tool integration path for both internal and external use; eliminates the dual-system complexity.

2. **Add MCP client capability** -- Allow roachie to consume external MCP servers. For example, connect to a Slack MCP server to send alerts, or a Grafana MCP server to read dashboards. This would extend roachie's reach beyond CockroachDB tools without custom integration code.

3. **Bridge the 38-tool gap** -- For the 38 tools excluded from MCP (bash-orchestrated), create MCP wrappers that invoke the bash tools via subprocess. The tools already have `--help` documentation and consistent flag interfaces. A thin wrapper could expose them as MCP tools while preserving the bash orchestration logic.

4. **Add dynamic tool registration** -- Allow users to register custom tools at runtime: `roachie-nl --register-tool my_tool.yaml`. This enables extending roachie's capabilities without modifying the codebase.

---

## Pillar 2: RAG (Retrieval-Augmented Generation)

**The problem:** Answering accurately from knowledge the LLM was never trained on.
**The solution:** Retrieve relevant information first, then reason over it.

### How Roachie Implements RAG

Roachie has **three RAG subsystems** serving different knowledge needs:

#### RAG Subsystem 1: Tool RAG (Primary)

The most sophisticated retrieval system -- finds the right tools for each user query:

| Stage | Implementation | Performance |
|-------|---------------|-------------|
| **Document processing** | 77 tool descriptions curated in `tool_descriptions.txt` (pipe-delimited, augmented with real-world query phrases) | Static, manual updates |
| **Embedding generation** | Pre-computed offline for 3 providers: Ollama/nomic-embed-text (768d), OpenAI/text-embedding-3-small (1536d), Gemini/gemini-embedding-001 (3072d) | Offline, one-time |
| **Storage** | JSON files in `tools/embeddings/` (one per provider) | File-based, version-aware |
| **Retrieval** | Hybrid: regex keyword matching (intent gate) + cosine similarity (semantic ranking) | 99-100% accuracy (135 queries) |
| **Injection** | Top-7 tool `--help` outputs injected into system prompt | Per-query, parallel fetch |

**Hybrid retrieval flow:**
```
User query
  → Regex keywords (73 patterns in regex_keywords.tsv)     ~1ms
  → If < 5 regex matches: semantic cosine similarity        ~10-200ms
  → Combine: semantic first (priority), regex supplements
  → Deduplicate, top 7
  → Fetch --help for each (parallel background jobs)
  → Inject into system prompt as "RELEVANT TOOL DOCUMENTATION"
```

#### RAG Subsystem 2: Doc RAG (Knowledge Base)

Retrieves CockroachDB-specific knowledge the LLM was never trained on:

| Stage | Implementation | Performance |
|-------|---------------|-------------|
| **Document processing** | 14 hand-curated CockroachDB knowledge chunks in `crdb_knowledge_base.txt` | Static, manual |
| **Embedding generation** | Pre-computed per provider, stored in `tools/embeddings/docs/` | Offline |
| **Retrieval** | Cosine similarity, top-3 above 0.45 threshold | Reuses query embedding from Tool RAG |
| **Injection** | Chunk content injected into system prompt after tool documentation | Per-query |

**Embedding reuse optimization:** The query embedding computed for Tool RAG is cached in `${_NL_METRICS_PREFIX}_query_embedding` and reused by Doc RAG, eliminating a redundant API call (`llm_prompt.sh:518-524`).

#### RAG Subsystem 3: Live Schema Context (Real-Time RAG)

Retrieves database structure directly from the cluster -- knowledge no static corpus could provide:

| Stage | Implementation | Performance |
|-------|---------------|-------------|
| **Data source** | Live CockroachDB cluster via `cockroach sql` CLI | Real-time |
| **Scope** | Databases (max 8), tables per database (max 50), column definitions | Bounded |
| **Multi-tenant** | Queries up to 3 virtual clusters (system + 2 tenants) | Parallel |
| **Caching** | Cached per session in `_cached_schema_context` | Once per session |
| **Injection** | Compact format injected between topology and tool catalog | Fixed position |

### Pros

- **Three complementary RAG subsystems** -- Tool RAG finds the right tools, Doc RAG provides domain knowledge, Schema RAG provides environmental awareness. Each addresses a different "knowledge the LLM was never trained on" problem. This is a richer RAG architecture than the framework's single-pipeline model.

- **Hybrid retrieval eliminates the single-strategy weakness** -- Pure embedding search misses keyword-specific queries; pure keyword search misses semantic intent. The hybrid approach (regex as intent gate, semantic as ranker) achieves 99-100% accuracy across 135 test queries with 3 embedding providers. This is measurably better than either strategy alone.

- **Live schema is genuinely novel** -- Most RAG systems retrieve from static corpora. Roachie queries the actual database at session start to learn what databases, tables, and columns exist. This means the system has real-time, accurate knowledge of the user's environment -- no static corpus can provide this.

- **Pre-computed embeddings eliminate runtime cost** -- Tool and doc embeddings are computed once offline and shipped with the code. Runtime cost is a single query embedding (~1ms local, ~100ms API). No vector database server needed, no embedding pipeline at startup.

- **Three-provider fallback with graceful degradation** -- Ollama (local, free) -> OpenAI (API, cheap) -> Gemini (API, free tier) -> regex-only (no embeddings). The system never fails due to missing embeddings; it progressively degrades from hybrid to keyword-only retrieval.

- **Token efficiency through selective retrieval** -- Only 7 tools (of 77) are injected per query. Only 3 doc chunks (of 14) are retrieved. Schema context is bounded to 15 compact lines. This keeps the prompt focused and token-efficient.

### Cons

- **No vector database** -- Embeddings are stored in flat JSON files and loaded entirely into memory for jq-based cosine similarity. With 77 tools and 14 doc chunks this is fast (~5ms), but it doesn't scale to thousands of documents. A vector database (Qdrant, ChromaDB) would enable larger knowledge bases with approximate nearest neighbor search.

- **Static document corpus** -- The 14 CockroachDB knowledge chunks and 77 tool descriptions are manually curated. When CockroachDB releases new features or deprecates old ones, the corpus is stale until a developer updates it. There's no automated ingestion pipeline.

- **No chunking strategy for large documents** -- Doc RAG uses 14 pre-chunked documents. There's no automated chunking pipeline for ingesting new documents. If someone wanted to add the entire CockroachDB documentation (thousands of pages), they'd need to manually chunk it.

- **No re-ranking** -- Retrieved results are ranked by bi-encoder cosine similarity only. A cross-encoder re-ranker could significantly improve precision for ambiguous queries by evaluating (query, document) pairs jointly rather than comparing independent embeddings.

- **No feedback loop from retrieval to accuracy** -- The system doesn't track which retrieved tools were actually used by the LLM vs. ignored. This data would identify false positive retrievals and enable retrieval quality optimization.

- **Schema context is session-cached only** -- If tables are added mid-session, the schema context is stale until the next session. There's no refresh mechanism.

### Room to Improve

1. **Add retrieval quality tracking** -- Log which of the 7 retrieved tools the LLM actually selected. If a tool is consistently retrieved but never used, it's a false positive. Use this data to tune descriptions, similarity thresholds, or regex patterns. The metrics CSV already logs `enriched_tools` and `commands_executed` -- this is a correlation analysis on existing data.

2. **Add automated doc ingestion** -- Create a pipeline that: takes a CockroachDB release notes URL -> scrapes the content -> chunks into knowledge-base-sized pieces -> computes embeddings -> adds to the doc RAG corpus. Run this on each new CockroachDB release.

3. **Add schema refresh command** -- `/refresh-schema` in the REPL that re-queries the cluster for database/table structure. This handles mid-session schema changes without restarting.

4. **Consider cross-encoder re-ranking** -- After retrieving the top-7 tools by cosine similarity, run a cross-encoder on (query, tool_description) pairs to re-rank. This adds ~100ms but catches cases where two tools have similar embeddings but different intent matches.

5. **Scale to vector database if corpus grows** -- If the knowledge base grows beyond ~100 documents, migrate from jq-based cosine similarity to a lightweight vector database (ChromaDB or Qdrant). Keep the current approach for tool embeddings (77 is small enough for in-memory search).

---

## Pillar 3: Agent Skills

**The problem:** Repeating the same instructions in prompts wastes tokens on every call.
**The solution:** A skill manager that stores reusable prompts and actions, loading only what's needed when it's needed.

### How Roachie Implements Skills

Roachie doesn't have a formal "Skill Manager" component, but it **solves the same problem through architectural design**:

#### Mechanism 1: Modular Prompt Templates (Static Skills)

The system prompt is assembled from 8 modular template files that are loaded conditionally:

| Template File | Purpose | Token Cost | Loaded When |
|--------------|---------|-----------|-------------|
| `parameter_rules.txt` | Dual-parameter tool rules (cr_ddl_diff, cr_migrate, cr_catalog_diff) | ~200 tokens | Always |
| `tool_notes.txt` | Critical per-tool flag notes | ~150 tokens | Always |
| `execution_pattern.txt` | Command template with `{{TOOLS_DIR}}` substitution | ~100 tokens | Always |
| `hallucination_warnings.txt` | Common LLM mistakes to avoid | ~200 tokens | Only when `NL_NATIVE_TOOLS != 1` |
| `response_format.txt` | JSON output schema specification | ~150 tokens | Always |
| `execution_rules.txt` | SQL/command validation rules | ~200 tokens | Always |
| `tool_specific_rules.txt` | Per-tool special handling | ~100 tokens | Always |
| `few_shot_examples.txt` | Real query -> command examples | ~300 tokens | Always |

**Implementation:** `_nl_load_prompt_template()` in `llm_prompt.sh:38-57` loads templates from `tools/utils/prompts/`, with `{{TOOLS_DIR}}` variable substitution and path traversal prevention.

#### Mechanism 2: Dynamic Tool Help Injection (Query-Specific Skills)

Instead of including all 77 tools' documentation in every prompt (massive token waste), roachie injects only the relevant tools:

```
Without dynamic injection:  77 tools x ~200 tokens/tool = ~15,400 tokens per query
With dynamic injection:      7 tools x ~200 tokens/tool =  ~1,400 tokens per query
Token savings: ~90% per query
```

**How it works:**
1. Query arrives -> regex + semantic matching identifies 7 relevant tools
2. Tool `--help` output fetched in parallel (background jobs)
3. Only matched tools' documentation injected under "RELEVANT TOOL DOCUMENTATION"
4. Duplicate avoidance: tools already documented in base prompt are skipped (`llm_prompt.sh:486-494`)

#### Mechanism 3: Compact Prompt for Local Models (Model-Specific Skills)

Ollama models get a different, token-optimized prompt variant:

| Aspect | Full Prompt (Cloud) | Compact Prompt (Ollama) |
|--------|-------------------|----------------------|
| Builder | `_nl_build_system_prompt()` | `_nl_build_system_prompt_compact()` |
| Tool listing | Full categorized catalog with descriptions | Condensed one-line-per-tool listing |
| Schema context | Full | Truncated to 15 lines |
| Response format | Detailed with examples | Single-line JSON template |
| Flag rules | Full documentation | 7 critical rules only |
| Estimated size | ~3,000-4,000 tokens | ~800-1,200 tokens |

#### Mechanism 4: Persistent Learning Injection (Experience-Based Skills)

Past failures and successes are injected as "learned skills":

```
Recent failure patterns (last 30 days, max 15):
  "Query 'show table sizes' failed with cr_tables -T: use positional argument instead"

Recent success patterns (last 30 days, max 10):
  "Query 'show slow queries' succeeded with cr_query_stats -d movr --top 5"
```

These act as experience-based skills: the agent learns to avoid known bad patterns and prefer known good patterns without being re-instructed each time.

#### Mechanism 5: Conditional Prompt Sections (Feature-Gated Skills)

Several prompt sections are conditionally included based on runtime configuration:

| Condition | Included Section | Excluded Section |
|-----------|-----------------|-----------------|
| `NL_NATIVE_TOOLS=1` | MCP tool schemas | Hallucination warnings (model constrained by schema) |
| Ceph context detected | Ceph storage tools | Non-ceph storage tools |
| Ollama provider | Compact prompt, plain text headers | Full prompt, unicode headers |
| No persistent failures loaded | (nothing) | Failure examples section |

### Pros

- **Dynamic tool injection saves ~90% of tokens** -- The single biggest skill-like optimization. Instead of dumping all 77 tool descriptions into every prompt (~15K tokens), only 7 relevant tools are injected (~1.4K tokens). This is exactly the "load only what you need, when you need it" principle that Skills are designed for.

- **Dual prompt variants save tokens for local models** -- The compact prompt for Ollama (~1K tokens) vs the full prompt for cloud providers (~4K tokens) is a model-aware skill optimization. Local models with 16K context windows get a prompt that leaves room for conversation; cloud models with 128K-1M windows get richer context.

- **Persistent learning acts as accumulated skill** -- When the system learns "cr_ddl_table fails with -T flag, use positional argument instead," this becomes a reusable skill injected into every future prompt. The agent doesn't need to re-learn this lesson each session.

- **Template modularity enables conditional loading** -- The 8 template files can be independently updated, versioned, and conditionally loaded. Hallucination warnings are skipped when native tool calling is active (the LLM is constrained by the tool schema, making warnings unnecessary). This is feature-gated skill management.

- **Duplicate avoidance prevents token waste** -- Tools already documented in the base system prompt (with flags and examples) are skipped during enrichment (`llm_prompt.sh:486-494`). This prevents the same information from appearing twice in the prompt.

### Cons

- **No formal skill manager** -- The framework describes a Skill Manager that receives skill requests from the LLM, retrieves the right skill, and returns results. Roachie has no such component. Skills (prompt templates, tool docs) are loaded by the orchestrator before the LLM call, not requested by the LLM during execution.

- **Base prompt templates are always loaded** -- While tool documentation is dynamic, the 6 always-loaded templates (parameter_rules, tool_notes, execution_pattern, response_format, execution_rules, tool_specific_rules) are injected into every query regardless of relevance. If the user asks "what is CockroachDB?", they still get SQL execution rules and flag documentation -- ~800 wasted tokens.

- **No skill caching across queries** -- If two consecutive queries need the same tool (e.g., both about cr_query_stats), the system re-fetches and re-injects the tool's `--help` output for each query. There's no per-session skill cache that says "we already injected cr_query_stats documentation in the last turn."

- **No user-defined skills** -- Users cannot create, save, or share reusable skill definitions. Common patterns like "run health check on all tenants" or "generate weekly performance report" must be specified from scratch each time. There's no skill library.

- **Prompt grows monotonically within a session** -- As the conversation progresses, the message history grows. Persistent learning adds tokens. Schema context adds tokens. There's no mechanism to drop irrelevant prompt sections as the session context becomes more specific.

- **Few-shot examples are static** -- The `few_shot_examples.txt` file provides fixed examples. There's no mechanism to select the most relevant examples for each query type (e.g., show schema examples for schema queries, performance examples for performance queries).

### Room to Improve

1. **Add a skill manager with LLM-requested skills** -- Instead of pre-loading all relevant tool documentation, let the LLM request specific tools:
   ```json
   {"skill_request": "need_tool_help", "tool": "cr_query_stats"}
   ```
   The skill manager retrieves the tool's `--help` output and returns it. This moves from "predict what the LLM needs" (current) to "give the LLM what it asks for" (skill pattern). Reduces wasted tokens when the predicted tools are wrong.

2. **Add conditional base prompt sections** -- Analyze the query type before building the base prompt. Informational queries ("what is CockroachDB?") don't need execution rules, flag documentation, or parameter rules. Skip these sections for zero-command queries. Estimated savings: ~800 tokens per informational query.

3. **Add user-defined skills** -- Allow users to save reusable query templates:
   ```
   /skill save "health-check" "Run cr_health, cr_health_liveness, cr_cluster_stats on all tenants"
   /skill run "health-check"
   ```
   Skills stored in `~/.roachie/skills/` as structured files with name, description, prompt template, and expected tools.

4. **Add dynamic few-shot example selection** -- Instead of including all few-shot examples in every prompt, select the 2-3 most similar examples based on the query's matched tools. If the query matches cr_query_stats, include the query performance few-shot example. Saves ~200 tokens per query and provides more relevant examples.

5. **Add per-session tool documentation cache** -- Once a tool's `--help` output is injected in turn 1, cache it for the session. If the same tool is needed in turn 3, reference the cached version: "See cr_query_stats documentation from earlier in this conversation." Saves ~200 tokens per repeated tool.

---

## Cross-Pillar Integration

The framework treats MCP, RAG, and Skills as independent pillars. In practice, roachie's implementations are **deeply integrated**:

### Integration 1: RAG Feeds Skills

Tool RAG retrieval (Pillar 2) determines which tool documentation skills (Pillar 3) are loaded. The retrieval result directly drives skill selection:

```
User query → RAG (regex + semantic) → top-7 tools → Skill (inject --help docs)
```

Without RAG, all 77 tools' documentation would be loaded (pure Skill problem). Without Skills, the retrieved tools' documentation couldn't be efficiently injected (pure RAG problem). The integration is synergistic.

### Integration 2: MCP and RAG Share Tool Metadata

The same `tool_descriptions.txt` file that drives RAG retrieval also informs MCP tool descriptions. Tool embeddings used for RAG semantic matching are computed from the same descriptions that external MCP clients see. This ensures consistency between internal retrieval and external discovery.

### Integration 3: Skills Reduce RAG's Token Burden

By injecting only 7 of 77 tools' documentation (Skill optimization), the system leaves room in the context window for Doc RAG chunks and schema context (RAG enrichment). Without the Skill-like dynamic injection, the base prompt would consume too many tokens for RAG results to fit.

### Integration Matrix

| Interaction | How It Works |
|-------------|-------------|
| RAG -> Skills | Retrieved tools determine which skills (docs) are loaded |
| Skills -> RAG | Token-efficient skills leave room for RAG context in the prompt |
| MCP -> RAG | MCP tool schemas share metadata with RAG tool descriptions |
| MCP -> Skills | Native tool calling (`NL_NATIVE_TOOLS=1`) eliminates hallucination warning skill |
| RAG -> MCP | Semantic matching helps select which MCP tools to include in native calls |

---

## Summary Scorecard

| Pillar | Score | What Roachie Gets Right | What's Missing |
|--------|-------|------------------------|----------------|
| **MCP** | **7/10** | 39 MCP-exposed tools, native tool calling for 3 providers, toolset organization | Only 39/77 tools exposed, MCP not the internal path, no MCP client capability |
| **RAG** | **8.5/10** | 3-subsystem hybrid (tool + doc + schema), 99-100% accuracy, live schema, 3-provider fallback | Static doc corpus, no vector DB, no re-ranking, no retrieval feedback loop |
| **Skills** | **6/10** | Dynamic injection saves ~90% tokens, dual prompt variants, persistent learning as accumulated skill | No formal skill manager, base templates always loaded, no user-defined skills |

### Overall: 7.2/10

---

## The Key Insight

Roachie solves all three pillar problems, but **the solutions vary in sophistication**:

- **RAG is the most mature pillar** (8.5/10) -- The hybrid regex + semantic retrieval with three knowledge sources (tools, docs, live schema) and three embedding providers is more sophisticated than what the framework describes. The live schema context is genuinely novel -- most RAG systems retrieve from static corpora.

- **MCP is well-implemented but underutilized** (7/10) -- The MCP server and native tool calling bridge exist and work, but MCP is an external-facing interface, not the internal execution path. The internal system still carries its own custom tool integration. Making MCP the unified internal/external tool protocol would simplify the architecture.

- **Skills is the biggest opportunity** (6/10) -- Roachie solves the token waste problem architecturally (dynamic injection, compact prompts) rather than through a formal skill system. This works but leaves value on the table: no user-defined skills, no LLM-requested documentation, no conditional base prompt sections, and static few-shot examples.

### Where Roachie Exceeds the Framework

1. **Three RAG subsystems** vs the framework's single RAG pipeline -- Tool RAG, Doc RAG, and live schema each address different knowledge gaps
2. **Hybrid retrieval** vs the framework's embedding-only search -- Regex + semantic achieves higher accuracy than either alone
3. **Pre-computed embeddings** vs the framework's runtime embedding pipeline -- Zero-startup-cost retrieval, no vector DB dependency
4. **Persistent learning as accumulated skill** -- The framework doesn't mention learning from experience; roachie's persistent failure/success injection acts as an evolving skill library

### Where Roachie Falls Short

1. **No MCP client** -- Can't consume external MCP servers (Slack, Grafana, etc.)
2. **No formal skill manager** -- Skills are loaded by the orchestrator, not requested by the LLM
3. **No user-defined skills** -- Users can't create reusable action templates
4. **Static doc corpus** -- No automated knowledge base updates
5. **No dynamic few-shot selection** -- Examples are fixed regardless of query type

### Top 5 Improvements by Impact

1. **Add user-defined skills** -- `/skill save` and `/skill run` for reusable action templates. Highest user-facing impact; enables workflow automation without re-instruction.

2. **Add conditional base prompt loading** -- Skip execution rules, flag docs, and parameter rules for informational queries. ~800 token savings per zero-command query.

3. **Add MCP client capability** -- Enable roachie to consume external MCP servers for alerts (Slack), dashboards (Grafana), and documentation (Confluence). Extends reach without custom integration code.

4. **Add automated doc corpus updates** -- Pipeline to ingest CockroachDB release notes and update Doc RAG embeddings. Keeps knowledge base current without manual curation.

5. **Add dynamic few-shot selection** -- Match few-shot examples to query type using the same semantic similarity used for tool matching. More relevant examples = better LLM output.

---

## CLI Summary (Quick Reference)

### Three Pillars Scorecard

| Pillar | Score | One-Line Assessment |
|--------|-------|---------------------|
| **MCP** | **7/10** | 39 tools exposed via GenAI Toolbox + native tool calling for 3 providers; not the internal path |
| **RAG** | **8.5/10** | 3-subsystem hybrid (tool + doc + schema), 99-100% accuracy, live schema context |
| **Skills** | **6/10** | Dynamic injection saves ~90% tokens; no formal skill manager or user-defined skills |
| **Overall** | **7.2/10** | All three problems solved, but with varying sophistication |

### The Key Insight

Roachie solves all three pillar problems but the solutions vary in maturity. RAG is the strongest pillar -- the hybrid regex + semantic retrieval with three knowledge sources exceeds what the framework describes. MCP exists but is external-facing only. Skills are solved architecturally (dynamic prompt construction) rather than through a formal skill system.

### Where Roachie Exceeds the Framework
- Three RAG subsystems vs one (tool + doc + live schema)
- Hybrid retrieval (regex + semantic) vs embedding-only search
- Pre-computed embeddings with zero startup cost
- Persistent learning as accumulated experience-based skills

### Where Roachie Falls Short
- No MCP client (can expose tools, can't consume external MCP servers)
- No formal skill manager (skills loaded by orchestrator, not requested by LLM)
- No user-defined skills (no `/skill save`, `/skill run`)
- Static doc corpus (no automated knowledge base updates)
- No dynamic few-shot example selection

### Top 5 Improvements by Impact
1. **User-defined skills** -- `/skill save` + `/skill run` for reusable action templates
2. **Conditional base prompt loading** -- skip irrelevant sections for informational queries (~800 token savings)
3. **MCP client capability** -- consume external MCP servers (Slack, Grafana, Confluence)
4. **Automated doc corpus updates** -- ingest CockroachDB release notes automatically
5. **Dynamic few-shot selection** -- match examples to query type via semantic similarity
