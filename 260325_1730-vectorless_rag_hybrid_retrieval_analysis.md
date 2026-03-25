# Roachie: Vectorless RAG and Hybrid Retrieval Analysis

**Date:** 2026-03-25
**Scope:** Evaluate roachie's retrieval architecture against the "Vectorless RAG" and hybrid retrieval framework -- vectors vs. BM25/keyword vs. SQL vs. knowledge graphs vs. LLM-native context

---

## Framework Recap

The conventional RAG stack (embed -> vector DB -> similarity search) works but carries baggage: embedding drift, chunk size tuning, missed keyword matches, vector store infrastructure cost, and re-indexing nightmares.

**Vectorless RAG** uses alternative retrieval methods:

| Method | Strength | Best For |
|--------|----------|----------|
| BM25 / TF-IDF | Exact keyword precision | Structured queries, known terminology |
| Knowledge graphs | Relationship reasoning | Entity connections, workflow chains |
| SQL-based retrieval | Structured data lookups | When data already lives in databases |
| LLM-native context | Zero retrieval overhead | Small document sets that fit in context |
| ColBERT late interaction | Token-level matching without dense vectors | Nuanced semantic matching |

**The real unlock: hybrid systems** that route queries to the right retrieval method at runtime.

---

## Roachie's Retrieval Architecture: Already Hybrid

This framework is the most aligned with what roachie already does. Roachie's retrieval system is a **multi-method hybrid** that uses four of the five vectorless methods alongside traditional vector similarity:

| Retrieval Method | Used in Roachie? | Implementation | Role |
|-----------------|-----------------|----------------|------|
| BM25 / TF-IDF (keyword) | **Yes (primary gate)** | Regex keyword matching via `regex_keywords.tsv` | Intent detection, high-precision filtering |
| Knowledge graphs | **No** | -- | Not implemented |
| SQL-based retrieval | **Yes** | Live schema context via `cockroach sql` queries | Real-time environmental awareness |
| LLM-native context | **Yes** | Tool catalog + few-shot examples in system prompt | Static domain knowledge |
| ColBERT late interaction | **No** | -- | Not implemented |
| Dense vector similarity | **Yes** | Pre-computed embeddings + cosine similarity | Semantic tool/doc matching |

**Roachie's hybrid retrieval flow:**

```
User query arrives
  │
  ├──→ [1] Regex keywords (BM25-like)        ~1ms, 73 patterns
  │       Intent gate: high precision, zero cost
  │       Result: scored tool list (higher score = more keyword groups matched)
  │
  ├──→ [2] Semantic vectors (if regex < 5)    ~10-200ms, 3 providers
  │       Cosine similarity against pre-computed embeddings
  │       Result: ranked tool list (by similarity score)
  │
  ├──→ [3] Combine: semantic first, regex supplements, deduplicate, top 7
  │
  ├──→ [4] Tool --help injection              ~500ms, parallel fetch
  │       LLM-native: inject matched tool docs into context window
  │
  ├──→ [5] Doc RAG (vector similarity)        ~10ms, reuses query embedding
  │       Top 3 CockroachDB knowledge chunks injected
  │
  └──→ [6] Live schema (SQL retrieval)        Once per session, cached
          SHOW DATABASES + information_schema queries
          Real-time tables, columns, data types injected
```

---

## Method-by-Method Evaluation

### 1. BM25 / TF-IDF Scoring (Keyword Matching)

**Framework says:** "Yes, the 'old' stuff still hits."
**Roachie's implementation:** Regex keyword matching via `regex_keywords.tsv`

#### How It Works

73 keyword patterns mapped to tool names in a tab-separated file:

```
# regex_keywords.tsv examples:
slow|query|queries|performance|latency     cr_query_stats cr_query_history cr_plan cr_workload cr_wait_events
table|tables|size|storage|space|large      cr_tables cr_size cr_db_size cr_db_tables_rowcount
lock|locks|blocking|blocked|contention     cr_show_locks cr_transactions
```

**Execution:** Pure bash `[[ =~ ]]` regex matching -- no external process, no index, no embedding. ~1ms per query. Patterns are cached in memory after first load (`_NL_CACHED_KEYWORDS`).

**Scoring:** Tools are scored by how many keyword groups they match. A query containing both "slow" and "query" gives `cr_query_stats` a score of 2 (matches two keyword groups), while `cr_tables` scores 0.

**Special patterns:**
- `&ceph_gate` -- Ceph-context flag (activates storage tools)
- `&!ceph_gate` -- Inverse gate (exclude ceph tools when not in ceph context)
- `&dual:PATTERN2` -- Requires both patterns (e.g., "show" AND "DDL" together)

#### Pros

- **Zero infrastructure cost** -- No vector store, no embedding model, no API call. A 73-line TSV file loaded into a bash variable. This is the cheapest retrieval method possible.

- **Perfect precision for known terminology** -- When a user says "show locks," the regex match is 100% precise: `cr_show_locks` and `cr_transactions`. No embedding model can be more precise for exact keyword matches.

- **Acts as intent gate for semantic search** -- This is the key architectural insight. Regex doesn't replace semantic search; it gates it. If regex finds 5+ confident matches, semantic search is skipped entirely (short-circuit at `llm_prompt.sh:408-411`). This saves 10-200ms per query when keyword matching is sufficient.

- **No embedding drift** -- Keyword patterns don't change when embedding models are updated. "lock" always matches "lock." No re-indexing, no drift, no version management.

- **Supports complex query patterns** -- Dual patterns (`&dual:`) require two concepts to co-occur, reducing false positives. "show" alone matches too many tools, but "show" AND "DDL" together precisely matches `cr_ddl_table`. This is richer than simple BM25 term matching.

#### Cons

- **Manual maintenance** -- Every new tool or new query pattern requires a human to update `regex_keywords.tsv`. There's no automated pattern discovery from usage data.

- **No term frequency weighting** -- Unlike BM25, the regex system doesn't weight by how rare a term is. "table" and "contention" are weighted equally, even though "contention" is far more discriminating (maps to only 2 tools vs 7).

- **No stemming or lemmatization** -- "querying" doesn't match the pattern `query|queries`. The pattern must explicitly list each word form. Misspellings are completely missed.

- **Binary match only** -- A pattern either matches or doesn't. There's no partial match score. BM25 would give a partial score for "table lock" even if neither keyword matches perfectly.

#### Room to Improve

1. **Add TF-IDF weighting to keyword scores** -- Weight each keyword group by its inverse document frequency (how many tools it maps to). "contention" maps to 2 tools and should score higher than "table" which maps to 7. This is trivial to compute from `regex_keywords.tsv` at load time.

2. **Add stemming** -- Normalize query terms before matching: "querying" -> "query", "locked" -> "lock", "tables" -> "table". A simple suffix-stripping function in bash (remove -ing, -ed, -s, -es) would catch most cases without external dependencies.

3. **Auto-generate keyword patterns from usage data** -- Analyze the metrics CSV to find queries that succeeded but weren't matched by regex. Extract new keyword patterns from these queries and suggest additions to `regex_keywords.tsv`.

---

### 2. Knowledge Graphs

**Framework says:** "Preserve entity relationships."
**Roachie's implementation:** Not implemented.

#### What's Missing

Roachie has no graph structure modeling relationships between:
- **Tools** -- `cr_query_stats` often leads to `cr_plan`, which often leads to `cr_indexes`. These workflow chains exist only in the LLM's training data.
- **Database objects** -- Table A references Table B via foreign key. No graph models this relationship for retrieval.
- **Troubleshooting paths** -- "high latency" -> check `cr_query_stats` -> if full scan, check `cr_indexes` -> if missing, use `cr_index_advisor`. This decision tree is implicit, not modeled.

#### Room to Improve

1. **Add a tool relationship graph** -- A simple JSON adjacency list:
   ```json
   {
     "cr_query_stats": {"leads_to": ["cr_plan", "cr_indexes"], "context": "performance"},
     "cr_plan": {"leads_to": ["cr_index_advisor", "cr_genstats"], "context": "optimization"},
     "cr_show_locks": {"leads_to": ["cr_transactions", "cr_wait_events"], "context": "contention"}
   }
   ```
   When a tool is selected, also inject its neighbors as "related tools." This adds graph-based retrieval without a graph database.

2. **Build schema relationship context** -- When the live schema fetch detects foreign keys (via `information_schema.table_constraints`), include them in the schema context: "rides.vehicle_city REFERENCES vehicles.city." This gives the LLM relationship awareness.

---

### 3. SQL-Based Retrieval

**Framework says:** "When structured data already lives in databases."
**Roachie's implementation:** Live schema context via `_nl_fetch_schema_context()`

#### How It Works

At session start, roachie queries the actual CockroachDB cluster for its schema:

```sql
-- Step 1: Discover tenants
SELECT name FROM [SHOW VIRTUAL CLUSTERS] WHERE data_state = 'ready';

-- Step 2: Per tenant, discover databases
SELECT database_name FROM information_schema.databases
  WHERE database_name NOT IN ('system', 'crdb_internal', ...);

-- Step 3: Per database, discover tables + columns
SELECT table_schema, table_name FROM information_schema.tables;
SELECT column_name, data_type FROM information_schema.columns WHERE table_name = '...';
```

Results are formatted into a compact schema representation and injected into the system prompt:

```
DATABASE SCHEMA (2 tenants queried):
TENANT: va
  Databases: movr, test_lab
  movr: rides(id UUID, city VARCHAR, vehicle_city VARCHAR, ...), vehicles(id UUID, ...)
  test_lab: products(id INT, name STRING, price DECIMAL)
TENANT: vb
  Databases: db1, db2
```

#### Pros

- **Real-time accuracy that no static corpus can match** -- The schema context reflects the actual current state of the database. When a table is added or dropped, the next session automatically picks it up. No re-indexing, no embedding update, no corpus refresh.

- **Zero hallucination on schema facts** -- When the LLM says "I'll query the rides table in the movr database," it's because it can see the table in the injected schema. Without this, the LLM would guess table names based on its training data.

- **Multi-tenant awareness** -- The SQL retrieval queries up to 3 tenants (system + 2 virtual clusters), giving the LLM knowledge of the entire multi-tenant topology. This is structured data retrieval that no embedding model could provide.

- **Bounded and token-efficient** -- Max 8 databases, 50 tables per database, compact format truncated to 15 lines for Ollama. The SQL retrieval is scoped to prevent token budget overflow.

- **Graceful degradation** -- If the cockroach CLI is missing, the connection fails, or the query times out (5-second timeout), the system continues without schema context. No crash, no error -- just reduced awareness.

#### Cons

- **Session-cached only** -- Schema is fetched once at session start. Mid-session DDL changes (CREATE TABLE, DROP TABLE) are invisible until the next session.

- **No row-level data retrieval** -- The system fetches schema metadata (table/column names) but not data values. It can't answer "what's the most common city in the rides table?" from retrieval alone -- it must execute `cr_query_stats` or raw SQL.

- **No index retrieval** -- The schema context includes tables and columns but not indexes. The LLM doesn't know which indexes exist, which affects its ability to diagnose performance issues without running a tool first.

- **No query statistics retrieval** -- The system doesn't pre-fetch query statistics (slow queries, hot tables) at session start. This structured data is already in the database and could inform the LLM's initial context.

#### Room to Improve

1. **Add `/refresh-schema` command** -- Allow mid-session schema refresh without restarting.

2. **Add index context to schema retrieval** -- Query `information_schema.statistics` or `SHOW INDEXES` to include indexes in the schema context. "rides has indexes: rides_pkey, rides_city_idx" helps the LLM diagnose performance issues.

3. **Add lightweight query statistics** -- Pre-fetch top-5 slowest queries from `crdb_internal.node_statement_statistics` at session start. This structured data retrieval would let the LLM proactively suggest performance improvements without the user asking.

---

### 4. LLM-Native Context (Feeding Docs Directly into Context Windows)

**Framework says:** "For smaller document sets that fit in context."
**Roachie's implementation:** Tool catalog, few-shot examples, and persistent learning injected directly into the system prompt.

#### How It Works

Roachie injects three types of knowledge directly into the LLM's context window:

| Content | Size | Source | Updated |
|---------|------|--------|---------|
| Tool catalog (77 tools categorized) | ~1,200 tokens | `tool_descriptions.txt` | Manual |
| Matched tool `--help` docs (top 7) | ~1,400 tokens | Tool binary `--help` output | Dynamic per query |
| Few-shot examples | ~300 tokens | `few_shot_examples.txt` | Manual |
| Parameter rules | ~200 tokens | `parameter_rules.txt` | Manual |
| Execution rules | ~200 tokens | `execution_rules.txt` | Manual |
| Persistent learning (failures/successes) | ~200-500 tokens | JSONL databases | Cross-session |
| Doc RAG chunks (top 3) | ~300-600 tokens | `crdb_knowledge_base.txt` | Manual |
| Schema context | ~200-400 tokens | Live SQL queries | Per session |

**Total prompt size:** ~4,000-5,000 tokens (cloud) or ~1,000-1,500 tokens (Ollama compact)

#### Pros

- **No retrieval pipeline for static knowledge** -- The tool catalog (77 descriptions), few-shot examples, and execution rules fit easily in the context window. There's no reason to embed and retrieve them -- they're small enough to include directly. This eliminates an entire retrieval pipeline.

- **Tool documentation set is small enough** -- Even including all 77 tool descriptions (~1,200 tokens) plus 7 matched tool `--help` docs (~1,400 tokens), the total is ~2,600 tokens. Modern context windows (128K-1M) can absorb this easily.

- **Persistent learning is genuinely LLM-native** -- Past failures and successes are injected directly as context, not retrieved via similarity search. This is the purest form of LLM-native retrieval: "here's what went wrong before, don't repeat it."

- **Compact prompt variant for small contexts** -- The Ollama compact prompt (~1K tokens) is designed for 16K context windows. The full prompt (~4K tokens) is for 128K+ windows. This shows awareness of context budget constraints.

#### Cons

- **Base prompt includes irrelevant content** -- The tool catalog lists all 77 tools even when only 7 are relevant. The execution rules are included even for informational queries. This wastes ~800-1,200 tokens per query on content the LLM won't use.

- **Few-shot examples are not query-matched** -- The same examples are included for every query type. A schema query gets performance examples; a performance query gets schema examples. Query-matched few-shot selection would be more token-efficient and more helpful.

- **No progressive context loading** -- Everything is loaded upfront. A smarter approach would start with minimal context and add more only if the LLM requests it (the Skills model from the previous analysis).

#### Room to Improve

1. **Remove tool catalog from base prompt** -- The 77-tool catalog is redundant when 7 matched tools with full `--help` docs are already injected. Remove the catalog and save ~1,200 tokens. The matched tools already cover what the LLM needs.

2. **Add query-matched few-shot examples** -- Select 2-3 examples most similar to the current query (by tool match overlap). If the query matches `cr_query_stats`, include the performance analysis example, not the DDL example.

---

### 5. Dense Vector Similarity (Traditional RAG)

**Framework says:** "Works, but comes with baggage."
**Roachie's implementation:** Pre-computed embeddings with jq-based cosine similarity.

#### How It Works

**Offline (one-time):**
1. 77 tool descriptions embedded via 3 providers (Ollama 768d, OpenAI 1536d, Gemini 3072d)
2. 14 CockroachDB doc chunks embedded via same 3 providers
3. Stored as JSON files in `tools/embeddings/`

**Runtime (per-query):**
1. User query embedded via one provider (~1ms local, ~100ms API)
2. Query embedding cached in temp file for reuse
3. Cosine similarity computed in jq against all tool embeddings
4. Top-7 tools above 0.35 similarity threshold returned
5. Same embedding reused for Doc RAG (top-3 chunks above 0.45 threshold)

**Implementation:** `_semantic_match_tools()` at `llm_prompt.sh:173-221` and `_doc_rag_retrieve()` at `llm_prompt.sh:231-294`.

#### How Roachie Avoids the "Vector RAG Baggage"

The framework identifies 5 pain points of traditional vector RAG. Here's how roachie handles each:

| Pain Point | Traditional RAG | Roachie's Approach |
|-----------|----------------|-------------------|
| **Embedding drift** | Model updates change vector spaces; old indexes become stale | Pre-computed offline; re-run `generate_embeddings.sh` when model changes. 77 tools + 14 docs takes seconds, not hours |
| **Chunk size tuning** | Never feels right; too small = lost context, too big = noise | 77 tool descriptions are naturally one-chunk-per-tool (~20-50 words each). 14 doc chunks are hand-curated at ~100-300 words. No automatic chunking needed |
| **Missed keyword matches** | Semantic search misses exact terms | Regex acts as primary gate; semantic only supplements. Keywords are never missed because regex runs first |
| **Infrastructure cost** | Pinecone/Weaviate/Qdrant hosting, scaling, maintenance | No vector database. JSON files + jq cosine similarity. Zero infrastructure cost. Works offline |
| **Re-indexing nightmares** | Changing embedding model = re-index everything | `generate_embeddings.sh --provider all` re-embeds all 77+14 items in ~30 seconds. No index, no migration |

#### Pros

- **Avoids all five pain points** -- No vector database (file-based), no re-indexing (regenerate in seconds), no chunk size problems (natural one-per-tool chunks), no missed keywords (regex gate), no embedding drift (re-generate is trivial).

- **Three-provider redundancy** -- If Ollama is unavailable, fall back to OpenAI. If OpenAI is unavailable, fall back to Gemini. If no embedding model is available, fall back to regex-only. Four fallback levels prevent retrieval failure.

- **Embedding reuse eliminates redundant API calls** -- A single query embedding is computed once and reused for both tool matching and Doc RAG retrieval. This saves 50-500ms per query compared to computing two separate embeddings.

- **Pre-computed tool magnitudes** -- The embedding files can include pre-computed magnitude values (`.value.magnitude`), avoiding the per-tool magnitude calculation during cosine similarity. This optimization is supported at `llm_prompt.sh:215`.

#### Cons

- **Flat file doesn't scale** -- With 77 tools and 14 doc chunks (~91 items), jq-based cosine similarity takes ~5ms. At 1,000 items, it would take ~50ms. At 10,000 items, it would be impractical. The current approach has a scaling ceiling.

- **No approximate nearest neighbor** -- Every query computes exact cosine similarity against every embedding. ANN indexes (HNSW, IVF) would provide sub-millisecond retrieval even at scale, but aren't needed at current corpus size.

- **No cross-encoder re-ranking** -- The bi-encoder cosine similarity is the only ranking signal. A cross-encoder evaluating (query, tool_description) pairs jointly would improve precision for ambiguous queries.

- **Three separate embedding files per provider** -- Tool embeddings and doc embeddings are stored separately per provider, totaling 6 files. Version-specific embeddings add more. This file proliferation is manageable but not elegant.

#### Room to Improve

1. **Add cross-encoder re-ranking** -- After cosine similarity selects top-7, run a cross-encoder on (query, description) pairs to re-rank. Most impactful for ambiguous queries where two tools have similar embeddings.

2. **Consolidate embedding files** -- Merge tool and doc embeddings into a single file per provider. Add a `type` field ("tool" or "doc") to each entry. Reduces file count from 6 to 3.

---

## The Hybrid Architecture Assessment

The framework argues: "Production RAG in 2026 isn't 'vectors vs no vectors.' It's routing queries to the right retrieval method at runtime."

### Roachie's Hybrid Routing

| Retrieval Method | When Roachie Uses It | Routing Logic |
|-----------------|---------------------|---------------|
| Regex/keyword | **Always first** | Primary gate, ~1ms |
| Dense vectors | **When regex finds < 5 matches** | Short-circuit optimization |
| Doc RAG vectors | **When enabled and embedding available** | Always runs (reuses query embedding) |
| SQL-based schema | **Once per session** | Cached, not per-query |
| LLM-native context | **Always** | Base prompt components |

**Routing decision tree:**
```
Query arrives
  → Regex: found 5+ confident matches?
      YES → Skip semantic, use regex results (saves 10-200ms)
      NO  → Run semantic cosine similarity
            → Combine: semantic first, regex supplements
  → Doc RAG: embedding provider available?
      YES → Cosine similarity on doc chunks (reuses query embedding)
      NO  → Skip doc RAG
  → Schema: cached from session start?
      YES → Already in prompt
      NO  → Query skipped (no cluster connection)
  → LLM-native: always included
      → Tool catalog, execution rules, few-shot examples
```

### Pros of Roachie's Hybrid

- **Four of five methods implemented** -- Regex (BM25-like), dense vectors, SQL-based, and LLM-native context are all active. Only knowledge graphs are missing. This is a genuine multi-method hybrid, not just "vectors + keyword search."

- **Intelligent routing, not brute-force fusion** -- The regex-first gate with semantic short-circuit is a cost-aware routing decision. When keywords are sufficient (most queries), the expensive embedding computation is skipped. This is exactly the "route to the right method" principle.

- **Each method covers a different weakness** -- Regex catches exact keywords (high precision). Vectors catch synonyms and paraphrases (high recall). SQL catches real-time schema facts (environmental accuracy). LLM-native context provides domain rules (persistent knowledge). No single method could cover all four.

- **Graceful degradation across all methods** -- If vectors fail (no embedding model), regex handles it. If SQL fails (no cluster connection), the prompt continues without schema. If Doc RAG fails, tool matching still works. The system never crashes due to retrieval failure.

- **Measured and proven** -- 135 test queries across 3 providers, 2 test modes, measuring accuracy per retrieval method:
  - Combined (regex + semantic): 99-100% across all providers
  - Regex alone: 96.8%
  - Semantic alone: 93-98% depending on provider
  - The hybrid measurably outperforms any single method

### Cons of Roachie's Hybrid

- **No knowledge graph** -- Tool relationships, troubleshooting workflows, and schema dependencies are not modeled as a graph. The LLM must infer relationships from its training data rather than receiving explicit relationship context.

- **No query routing based on query type** -- The routing is method-based (regex first, then semantic) not query-type-based (schema queries → SQL, performance queries → tool docs). A query classifier could route different query types to specialized retrieval strategies.

- **No ColBERT or late interaction** -- The system uses bi-encoder cosine similarity (fast but imprecise) or regex (precise but rigid). ColBERT-style token-level matching would offer a middle ground: semantic understanding with keyword precision.

- **Retrieval methods are hardcoded, not learned** -- The routing logic (regex threshold of 5, semantic threshold of 0.35, Doc RAG threshold of 0.45) uses fixed thresholds. These could be learned from usage data: "for schema queries, regex alone achieves 100% accuracy, so skip semantic entirely."

- **No retrieval fusion scoring** -- When regex and semantic results are combined, semantic tools are listed first (priority) and regex supplements. But there's no unified score that weights both signals. A Reciprocal Rank Fusion or weighted combination would produce a more principled ranking.

### Room to Improve

1. **Add Reciprocal Rank Fusion (RRF)** -- Instead of "semantic first, regex supplements," compute a fused score:
   ```
   RRF_score(tool) = 1/(k + regex_rank) + 1/(k + semantic_rank)
   ```
   This principled fusion weights both signals proportionally and handles cases where regex and semantic disagree.

2. **Add query-type routing** -- Classify queries before retrieval:
   - `schema_query` → prioritize SQL-based schema context, reduce tool matching
   - `performance_query` → prioritize tool RAG, include performance doc chunks
   - `informational_query` → prioritize Doc RAG, skip tool matching
   - `command_query` → prioritize regex (exact tool name), skip semantic

3. **Add a lightweight tool relationship graph** -- Model "leads_to" relationships between tools. When `cr_query_stats` is selected, also suggest `cr_plan` and `cr_indexes` as next steps. This adds graph-based retrieval with minimal infrastructure (a JSON adjacency list).

4. **Learn routing thresholds from usage data** -- Track per-query-type accuracy with different retrieval method combinations. If schema queries achieve 100% accuracy with regex alone, auto-skip semantic for schema queries. If performance queries need semantic, keep it enabled. Tune thresholds from the metrics CSV.

5. **Add ColBERT-style late interaction for ambiguous queries** -- When the top-2 tools have similar cosine similarity scores (within 0.05), use a token-level interaction model to disambiguate. This targeted use avoids the cost of running ColBERT on every query while catching the hardest cases.

---

## Does Roachie Need a Vector Database?

The framework asks when vectors make sense vs. when vectorless is better. Here's roachie's position:

### Arguments Against Adding a Vector Database

| Factor | Roachie's Situation | Verdict |
|--------|-------------------|---------|
| Corpus size | 77 tools + 14 doc chunks = 91 items | **Too small for a vector DB** |
| Re-indexing burden | `generate_embeddings.sh` re-embeds all 91 items in ~30 seconds | **Trivial without a vector DB** |
| Infrastructure cost | Zero (JSON files on disk) | **Would increase with a vector DB** |
| Deployment complexity | Zero-dependency local operation | **Vector DB adds a server dependency** |
| Query latency | ~5ms for jq cosine similarity on 91 items | **Fast enough without ANN indexes** |

### When Roachie Would Need a Vector Database

| Scenario | Trigger | Recommended DB |
|----------|---------|---------------|
| Knowledge base grows to 1,000+ docs | CockroachDB full docs ingestion | ChromaDB (embedded, no server) |
| Multi-cluster memory (1,000+ past queries) | Episodic retrieval across sessions | Qdrant (lightweight, single binary) |
| User-defined skills library (100+ skills) | Community skill sharing | SQLite with FTS5 (no separate service) |

**Current verdict: No vector database needed.** The corpus is small (91 items), re-indexing is trivial (30 seconds), and jq cosine similarity is fast enough (~5ms). Adding a vector database would increase complexity without improving retrieval quality.

**Future trigger:** If the knowledge base grows beyond ~500 items or if episodic memory (past query retrieval) is added, a lightweight embedded database (ChromaDB or SQLite with vector extensions) would be warranted.

---

## Summary Scorecard

| Retrieval Method | Roachie Score | Maturity | Key Strength | Key Gap |
|-----------------|---------------|----------|-------------|---------|
| BM25/Keyword (Regex) | **9/10** | Production | Intent gate, zero cost, no drift | No stemming, no TF-IDF weighting |
| Knowledge Graphs | **0/10** | Not started | -- | No tool relationships, no schema graph |
| SQL-Based Retrieval | **8/10** | Production | Live schema, multi-tenant, real-time | Session-cached only, no index metadata |
| LLM-Native Context | **7/10** | Production | Small corpus fits in context, persistent learning | Irrelevant content always loaded |
| Dense Vectors | **8/10** | Production | 3 providers, pre-computed, no vector DB | No re-ranking, no fusion scoring |
| **Hybrid Routing** | **8/10** | Production | 4-method hybrid, intelligent short-circuit | No query-type routing, fixed thresholds |

### Overall Hybrid Retrieval Score: 8/10

---

## The Key Insight

**Roachie accidentally built the architecture the framework recommends.** The hybrid retrieval system -- regex as intent gate, vectors for semantic understanding, SQL for structured lookups, LLM-native for small document sets -- is precisely the "production RAG in 2026" pattern the framework describes. And it did this without a vector database, without a chunking pipeline, without re-indexing infrastructure.

The reason this works is roachie's unique situation:
- **Small, structured corpus** (77 tools, 14 docs) -- fits in JSON files, no vector DB needed
- **Domain-specific terminology** -- "cr_query_stats" is an exact match, not a semantic concept
- **Structured data already in a database** -- Schema context is SQL, not embeddings
- **Each method covers a different weakness** -- regex for precision, vectors for recall, SQL for reality, LLM-native for rules

### What Makes Roachie's Hybrid Strong (Already)
- Regex-first routing with semantic short-circuit (cost-aware)
- 99-100% accuracy across 135 test queries (measured, not assumed)
- No vector database infrastructure (zero operational cost)
- Three-provider embedding fallback with graceful degradation to regex-only
- Live SQL retrieval provides knowledge no static corpus can match

### What Would Make It Stronger (Gaps)
- Knowledge graph for tool relationships and troubleshooting workflows
- Query-type routing (schema → SQL, performance → tools, informational → docs)
- Reciprocal Rank Fusion instead of priority-based combination
- Learned thresholds from usage data instead of fixed values
- Stemming/normalization in keyword matching

### Top 5 Improvements by Impact

1. **Add tool relationship graph** -- A JSON adjacency list modeling "leads_to" relationships between tools. When `cr_query_stats` is selected, also suggest `cr_plan` and `cr_indexes`. Highest impact because it enables multi-step workflow guidance -- the feature gap users would notice most.

2. **Add query-type routing** -- Classify queries and route to the optimal retrieval strategy per type. Schema queries get SQL-heavy retrieval; performance queries get tool-heavy retrieval; informational queries get doc-heavy retrieval. Reduces wasted retrieval and improves relevance.

3. **Add Reciprocal Rank Fusion** -- Replace "semantic first, regex supplements" with principled score fusion. This handles disagreements between regex and semantic more gracefully and produces a more reliable ranking.

4. **Add stemming to keyword matching** -- A simple suffix-stripper (remove -ing, -ed, -s, -es, -ly) would catch "querying" -> "query", "locked" -> "lock", "statistics" -> "statistic". Improves regex recall with zero cost.

5. **Add index metadata to schema retrieval** -- Include index names and columns in the schema context. This gives the LLM enough information to diagnose "missing index" performance issues without needing to run a tool first.

---

## CLI Summary (Quick Reference)

### Hybrid Retrieval Score: 8/10 -- "Accidentally Built the Right Architecture"

| Method | Score | Status | One-Line Assessment |
|--------|-------|--------|---------------------|
| BM25/Keyword (Regex) | **9/10** | Production | Intent gate, zero cost, no drift; needs stemming |
| Knowledge Graphs | **0/10** | Missing | No tool relationships or schema graph |
| SQL-Based Retrieval | **8/10** | Production | Live schema, multi-tenant; session-cached only |
| LLM-Native Context | **7/10** | Production | Small corpus fits in context; some waste |
| Dense Vectors | **8/10** | Production | 3 providers, pre-computed, no vector DB; no re-ranking |
| **Hybrid Routing** | **8/10** | Production | 4-method hybrid with short-circuit; no query-type routing |

### The Key Insight

Roachie accidentally built the "production RAG in 2026" architecture the framework recommends: multiple retrieval methods routed at runtime based on query characteristics. It did this without a vector database, without a chunking pipeline, and without re-indexing infrastructure -- because its corpus is small (91 items), its terminology is domain-specific, and its structured data already lives in a database.

### Does Roachie Need a Vector Database?

**No.** The corpus is too small (91 items), re-indexing is trivial (30 seconds), and jq cosine similarity is fast enough (~5ms). A vector DB would add infrastructure cost without improving quality. **Future trigger:** if the knowledge base grows beyond ~500 items or episodic memory is added.

### How Roachie Avoids the Five Vector RAG Pain Points

| Pain Point | Roachie's Avoidance |
|-----------|-------------------|
| Embedding drift | Pre-computed offline; re-generate in 30 seconds |
| Chunk size tuning | Natural one-per-tool chunks; hand-curated doc chunks |
| Missed keyword matches | Regex runs first as intent gate; keywords never missed |
| Infrastructure cost | JSON files + jq; zero infrastructure |
| Re-indexing nightmares | `generate_embeddings.sh --provider all` = 30 seconds |

### Top 5 Improvements by Impact

1. **Tool relationship graph** -- JSON adjacency list for workflow chains (cr_query_stats -> cr_plan -> cr_indexes)
2. **Query-type routing** -- Route schema/performance/informational queries to optimal retrieval strategy
3. **Reciprocal Rank Fusion** -- Principled score combination instead of priority-based ordering
4. **Keyword stemming** -- Simple suffix stripping for "querying" -> "query", "locked" -> "lock"
5. **Index metadata in schema context** -- Include index names/columns for performance diagnosis
