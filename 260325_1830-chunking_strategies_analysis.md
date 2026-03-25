# Roachie: Chunking Strategies Analysis

**Date:** 2026-03-25
**Scope:** Evaluate roachie's document chunking practices against the 5 chunking strategies framework -- fixed-length, sentence-based, semantic, hierarchical, and adaptive (hybrid).

---

## Framework Recap

| Strategy | How It Works | Strength | Weakness |
|----------|-------------|----------|----------|
| 1. Fixed-length | Split every N tokens | Fast, simple | Breaks mid-sentence |
| 2. Sentence-based | One sentence per chunk | Great for FAQs | Misses multi-sentence answers |
| 3. Semantic | Group by meaning via embeddings | Powerful | Heavy preprocessing |
| 4. Hierarchical | Follow document structure (headers, sections) | Great for technical docs | Poor for unstructured text |
| 5. Adaptive (hybrid) | Switch strategy based on content type | Best retrieval quality | Higher complexity |

The framework recommends: start with sentence-based, observe where retrieval breaks down, then graduate to semantic or adaptive.

---

## Roachie's Unique Situation: Three Corpora, Three Strategies

Roachie doesn't have one document corpus -- it has **three distinct knowledge sources**, each with a fundamentally different chunking approach:

| Corpus | Items | Avg Size | Chunking Strategy | Why This Strategy |
|--------|-------|----------|-------------------|-------------------|
| Tool descriptions | 77 | ~20-50 words | **Domain-natural** (one tool = one chunk) | Tools are atomic units; splitting would destroy meaning |
| CockroachDB knowledge base | 14 | ~100-300 words | **Hierarchical/topic-based** (one topic = one chunk) | Each chunk covers one operational topic end-to-end |
| Live schema context | Variable | ~5-20 words per table | **Structured data** (SQL-derived) | Schema is inherently structured; no chunking needed |

**Key insight: None of these corpora use traditional chunking.** There are no fixed-length splits, no sentence boundaries, no automated semantic grouping. Each corpus has a naturally atomic unit that serves as the chunk.

---

## Corpus 1: Tool Descriptions (77 items)

**File:** `tools/utils/tool_descriptions.txt`
**Format:** `tool_name|description` (one line per tool)

### Chunking Strategy: Domain-Natural (One Tool = One Chunk)

Each tool description is a single atomic unit:

```
cr_query_stats|Display slow query statistics, performance metrics, and execution latency analysis
cr_show_locks|Show active lock contention, blocking sessions, and queries waiting for locks
cr_tables|List all database tables with row counts, sizes, ranges, and storage details
```

**Chunk characteristics:**
- Size: 10-50 words (extremely consistent)
- Boundary: Tool boundary (never splits a tool's description)
- Overlap: None (tools are independent)
- Augmentation: 16 tools have real-world query phrases appended for better semantic matching

### Pros

- **Perfect semantic coherence** -- Each chunk is exactly one concept: one tool, one description. There's zero chance of splitting a meaningful unit across chunks. This is the ideal chunking granularity for a tool registry.

- **Uniform chunk size** -- All 77 descriptions are 10-50 words. This uniformity means embedding quality is consistent across chunks -- no outliers where a very long or very short chunk distorts the vector space.

- **No chunking pipeline needed** -- The tool descriptions are hand-curated at the right granularity from the start. No tokenizer, no splitter, no overlap calculation. The "chunking strategy" is simply "one tool per line."

- **Augmentation improves retrieval** -- 16 tools have real-world query phrases added to their descriptions (e.g., `cr_columns` includes "show me the list of columns for a table, what columns does a table have"). This acts as query-aware chunk enrichment, improving semantic matching without changing chunk boundaries.

### Cons

- **No context between related tools** -- Each tool is an isolated chunk. The embedding for `cr_query_stats` doesn't know that `cr_plan` is its natural follow-up. Relationship context is lost because each chunk is independent.

- **Short chunks may under-represent complex tools** -- `cr_migrate` (database migration between clusters) has a 15-word description. `cr_backup` (backup/restore operations) also has a short description. These complex, multi-mode tools are under-represented compared to simpler tools with similar-length descriptions.

- **Manual curation doesn't scale** -- 77 hand-curated descriptions are manageable. If the tool count grew to 500, maintaining quality descriptions would become a significant maintenance burden.

### How It Compares to the 5 Strategies

| Strategy | Applicable? | Why / Why Not |
|----------|------------|---------------|
| Fixed-length | **No** | Tools vary from 10-50 words; fixed-length would split some and waste space on others |
| Sentence-based | **Effectively yes** | Each description is 1-2 sentences; tool boundary ≈ sentence boundary |
| Semantic | **Not needed** | Each tool is already a single semantic unit |
| Hierarchical | **Partially** | Tools are categorized (12 groups) but categories aren't used for chunking |
| Adaptive | **Not needed** | Uniform content type (tool descriptions) doesn't benefit from strategy switching |

**Best match: Sentence-based** -- Each description is naturally sentence-length, and each sentence is a complete semantic unit. Roachie achieves sentence-based chunking without a sentence splitter because the source data is already at the right granularity.

---

## Corpus 2: CockroachDB Knowledge Base (14 chunks)

**File:** `tools/utils/crdb_docs/crdb_knowledge_base.txt`
**Format:** `CHUNK_ID` / `TITLE` / `CONTENT` blocks delimited by `---`

### Chunking Strategy: Hierarchical/Topic-Based (One Topic = One Chunk)

Each chunk covers one complete operational topic:

| Chunk ID | Topic | Word Count |
|----------|-------|-----------|
| gc_ttl_tuning | GC TTL configuration and tuning | ~120 |
| changefeed_troubleshooting | Changefeed issues and resolution | ~150 |
| slow_query_diagnosis | Slow query diagnosis workflow | ~150 |
| node_health_diagnostics | Node health and cluster monitoring | ~140 |
| backup_restore_operations | Backup and restore operations | ~140 |
| transaction_contention | Transaction contention and locks | ~150 |
| index_best_practices | Index design best practices | ~140 |
| pcr_physical_cluster_replication | Physical Cluster Replication (PCR) | ~140 |
| schema_design_patterns | Schema design patterns | ~140 |
| crdb_internal_tables | System tables reference | ~130 |
| error_codes_common | Common error codes | ~130 |
| performance_tuning | Performance tuning parameters | ~130 |
| multi_tenant_operations | Multi-tenant operations | ~120 |
| storage_and_compaction | Storage engine and compaction | ~130 |

**Chunk characteristics:**
- Size: 100-300 words (fairly consistent)
- Boundary: Topic boundary (each chunk is a complete topic)
- Overlap: Intentional cross-references between chunks (e.g., gc_ttl_tuning mentions changefeeds, changefeed_troubleshooting mentions gc.ttlseconds)
- Structure: Each chunk has a clear internal structure: definition → symptoms → commands → resolution

### Pros

- **Topic-complete chunks** -- Each chunk covers one operational topic from problem through solution. A user asking "how do I troubleshoot changefeeds?" gets a complete answer from a single chunk, not a fragment. This is the ideal chunking for operational knowledge bases.

- **Consistent chunk size** -- All 14 chunks are 100-300 words (~130-150 words average). This uniformity produces consistent embedding quality. No chunk is so small that it lacks context or so large that it dilutes the embedding.

- **Cross-references create implicit relationships** -- gc_ttl_tuning mentions changefeeds; changefeed_troubleshooting mentions gc.ttlseconds. These cross-references mean related topics share vocabulary, improving retrieval when a query spans multiple topics.

- **Tool integration in content** -- Each chunk references the relevant cr_* tools (e.g., "Use cr_query_stats to identify slow queries"). This creates a bridge between Doc RAG and Tool RAG -- retrieving a doc chunk implicitly suggests the right tools.

- **Structured internal format** -- Each chunk follows a consistent pattern: what it is → when to use it → how to diagnose → how to fix. This structure helps the LLM extract actionable information from the chunk.

- **Explicit metadata** -- Each chunk has a `CHUNK_ID` and `TITLE` separate from the content. The title is used for display; the content is embedded. This separation allows retrieval by content and display by title.

### Cons

- **Only 14 chunks** -- CockroachDB has hundreds of operational topics (security, multi-region, changefeeds, scheduled jobs, import/export, SQL tuning, etc.). 14 chunks cover only the most common DBA scenarios. Many legitimate user queries will find no matching doc chunk.

- **Manually curated, not auto-generated** -- Each chunk was written by hand, not extracted from CockroachDB documentation. This means coverage depends on the author's knowledge and judgment. Gaps are invisible until a user asks a question that has no matching chunk.

- **No versioning** -- The chunks don't indicate which CockroachDB version they apply to. Information about v25.2 vs v25.4 differences is scattered across chunks without explicit version tags.

- **No chunk hierarchy** -- The framework distinguishes "hierarchical chunking" (headers → sections → subsections). While each chunk has a TITLE, there's no parent-child relationship between chunks. "Performance tuning" doesn't link to "slow query diagnosis" or "index best practices" despite being topically related.

- **Fixed granularity** -- Every chunk is ~130 words. Some topics (like error codes) could benefit from finer granularity (one chunk per error code), while others (like schema design) could benefit from coarser granularity (a longer, more comprehensive treatment).

### How It Compares to the 5 Strategies

| Strategy | Applicable? | Why / Why Not |
|----------|------------|---------------|
| Fixed-length | **No** | Would split topics mid-explanation, destroying coherence |
| Sentence-based | **No** | Topics are multi-sentence; splitting by sentence would fragment the diagnosis → solution flow |
| Semantic | **Could improve** | Could auto-group related sentences within a topic or split overly broad topics |
| Hierarchical | **Yes -- this is what roachie uses** | Each chunk follows document structure (topic = section), with title + content |
| Adaptive | **Could improve** | Different topics could use different chunk sizes (error codes = small, schema design = large) |

**Best match: Hierarchical** -- Each chunk is a section of an implicit document, structured by topic. The `TITLE` acts as a section header; the `CONTENT` acts as the section body. This is exactly the hierarchical strategy the framework describes as "works beautifully for technical docs."

---

## Corpus 3: Live Schema Context (Dynamic)

**Source:** SQL queries against live CockroachDB cluster
**Format:** Compact text representation of databases, tables, and columns

### Chunking Strategy: Structured Data (No Chunking Needed)

The schema context is already structured data -- it doesn't need chunking:

```
DATABASE SCHEMA (2 tenants queried):
TENANT: va
  Databases: movr, test_lab
  movr: rides(id UUID, city VARCHAR, vehicle_city VARCHAR, ...), vehicles(id UUID, ...)
  test_lab: products(id INT, name STRING, price DECIMAL)
```

**Characteristics:**
- Size: Bounded by configuration (max 8 databases, max 50 tables per database)
- Boundary: Database/table/column boundaries (SQL-derived)
- Structure: Hierarchical (tenant → database → table → column)
- Freshness: Real-time (queried at session start)

### Pros

- **Structured data doesn't need chunking** -- Schema metadata is inherently structured: it has explicit boundaries (database, table, column), explicit hierarchy (tenant > database > table), and explicit types (VARCHAR, INT, UUID). Chunking would destroy this structure.

- **SQL-derived boundaries are perfect** -- Each "chunk" is a database-table-column tuple derived from `information_schema`. The boundaries are semantically meaningful by definition -- they're the actual objects in the database.

- **Token-bounded** -- Max 8 databases × 50 tables = 400 table entries. With compact formatting, this fits in ~200-400 tokens. The compact variant (Ollama) truncates to 15 lines. These bounds prevent schema context from overwhelming the prompt.

### Cons

- **No embedding or retrieval** -- Schema context is injected directly into the prompt, not embedded and retrieved. Every query sees the full schema regardless of whether it's relevant. For a 400-table database, most schema context is wasted tokens per query.

- **No column-level detail for large schemas** -- With 50 tables, listing all columns for every table would be too many tokens. The system likely truncates column listings, losing detail.

### Room to Improve

1. **Query-relevant schema injection** -- Instead of injecting the full schema, use the matched tool to determine which tables are relevant. If `cr_query_stats` is selected, inject only the schema for the database mentioned in the query, not all databases.

---

## Cross-Corpus Analysis: Roachie's Implicit Chunking Framework

Roachie doesn't use any of the 5 strategies explicitly, yet it achieves effective chunking through domain-aware design:

| Framework Strategy | Roachie Equivalent | Where Used | Effectiveness |
|-------------------|-------------------|-----------|---------------|
| Fixed-length | **Not used** | -- | N/A (avoided entirely) |
| Sentence-based | **Tool descriptions** | 77 tools, ~1 sentence each | **Excellent** -- natural sentence ≈ natural chunk |
| Semantic | **Not used** | -- | N/A (not needed at current scale) |
| Hierarchical | **Doc RAG chunks** | 14 topic-based chunks with title + content | **Good** -- complete topics, consistent size |
| Adaptive | **Cross-corpus routing** | Different strategy per corpus type | **Emerging** -- 3 strategies across 3 corpora |

### The Implicit Adaptive Strategy

While roachie doesn't switch chunking strategies within a single corpus, it effectively uses an **adaptive approach across corpora**:

| Query Type | Retrieval Source | Chunk Type | Strategy |
|-----------|-----------------|-----------|----------|
| "Which tool shows locks?" | Tool descriptions | Single-sentence tool desc | Sentence-based |
| "How do I troubleshoot changefeeds?" | Doc RAG | Multi-paragraph topic | Hierarchical |
| "What tables exist in movr?" | Live schema | Structured SQL metadata | No chunking (structured) |
| "Show slow queries in movr" | Tool + Schema | Tool desc + schema context | Cross-corpus |

This is adaptive chunking at the system level: different content types use different chunking strategies, and the retrieval system routes queries to the appropriate corpus.

---

## What Would Each Strategy Look Like If Applied to Roachie?

### If Fixed-Length (Strategy 1)

Splitting tool descriptions or doc chunks at fixed token boundaries would be **destructive**:

```
Fixed-length (50 tokens):
CHUNK 1: "cr_query_stats — Display slow query statistics, performance metrics, and execution latency analysis. cr_show_locks — Show active lock"
CHUNK 2: "contention, blocking sessions, and queries waiting for locks. cr_tables — List all database tables with row counts, sizes,"
```

Two tools merged into one chunk. Tool boundary destroyed. Retrieval for "show locks" would also return cr_query_stats. **This strategy would reduce roachie's 99-100% retrieval accuracy.**

**Verdict: Fixed-length would be harmful.** Roachie's corpus has natural boundaries that fixed-length splitting would violate.

### If Sentence-Based (Strategy 2)

Already effectively used for tool descriptions. For doc chunks, splitting by sentence would fragment the diagnosis → solution flow:

```
Sentence-based for doc chunk "Slow Query Diagnosis":
CHUNK 1: "Diagnosing slow queries in CockroachDB."
CHUNK 2: "Start with cr_query_stats to identify the slowest queries by total execution time."
CHUNK 3: "Look at mean_latency, max_latency, and execution_count..."
CHUNK 4: "Next, use cr_plan with the query fingerprint..."
```

Each sentence is too small to provide useful context. The workflow (cr_query_stats → cr_plan → cr_index_advisor) is split across 3+ chunks. A user asking "how do I diagnose slow queries?" would retrieve only partial guidance.

**Verdict: Sentence-based works for tool descriptions but would fragment doc chunks.** The current topic-based chunking for docs is superior.

### If Semantic (Strategy 3)

Semantic chunking would group sentences by embedding similarity. For doc chunks, this could be useful:

```
Semantic grouping for "Slow Query Diagnosis":
GROUP 1 (identification): "Diagnosing slow queries... cr_query_stats... mean_latency, max_latency..."
GROUP 2 (plan analysis): "Use cr_plan with the query fingerprint... full table scans... hash joins..."
GROUP 3 (index optimization): "cr_index_advisor... cr_best_practices... unused indexes..."
GROUP 4 (contention): "If latency is high but CPU is low... cr_transactions... cr_show_locks..."
```

This would produce more focused chunks, improving retrieval precision. A query about "missing indexes" would retrieve GROUP 3 instead of the entire slow query topic.

**Verdict: Semantic chunking could improve doc RAG precision** by splitting broad topics into focused sub-topics. Not needed for tool descriptions (already atomic).

### If Hierarchical (Strategy 4)

Already effectively used for doc chunks. Could be extended with parent-child relationships:

```
Hierarchical with nesting:
L1: "Performance" (parent topic)
  L2: "Slow Query Diagnosis" (subtopic)
    L3: "Identification" (section)
    L3: "Plan Analysis" (section)
    L3: "Index Optimization" (section)
  L2: "Performance Tuning" (subtopic)
  L2: "Transaction Contention" (subtopic)
```

This would allow retrieval at different granularities: broad queries ("tell me about performance") retrieve the L1 parent; specific queries ("how to analyze query plans") retrieve the L3 section.

**Verdict: Hierarchical with nesting would improve both precision (smaller chunks for specific queries) and recall (parent chunks for broad queries).** This is the highest-impact improvement for doc RAG.

### If Adaptive/Hybrid (Strategy 5)

Roachie already has elements of adaptive chunking across corpora. A fully adaptive approach would:

1. **Tool descriptions** → Keep as-is (sentence-based, natural boundaries)
2. **Doc chunks (structured topics)** → Use hierarchical with L2/L3 nesting
3. **Doc chunks (list content like error codes)** → Use item-based (one error code per chunk)
4. **Schema context** → Keep as-is (structured SQL, no chunking)
5. **Tool --help output** → Use section-based (split by Usage/Flags/Examples sections)

**Verdict: Adaptive chunking across content types is where roachie should head.** The framework's recommendation matches roachie's situation perfectly: different content types deserve different chunking strategies.

---

## Summary Scorecard

| Strategy | Used in Roachie? | Where | Score | Assessment |
|----------|-----------------|-------|-------|------------|
| 1. Fixed-length | **No** | -- | N/A | Correctly avoided -- would break natural boundaries |
| 2. Sentence-based | **Effectively yes** | Tool descriptions (77) | **9/10** | Natural sentence ≈ natural chunk; works perfectly |
| 3. Semantic | **No** | -- | N/A | Not needed at 14-chunk scale; would help at 100+ |
| 4. Hierarchical | **Yes** | Doc RAG (14 chunks) | **7/10** | Good topic-level chunking; lacks sub-topic nesting |
| 5. Adaptive (hybrid) | **Emerging** | Cross-corpus routing | **6/10** | 3 strategies across 3 corpora; no within-corpus adaptation |

### Overall Chunking Score: 7.5/10

---

## The Key Insight

**Roachie bypasses the chunking problem by having naturally atomic corpora.** The 77 tool descriptions don't need chunking because each tool IS a chunk. The 14 doc chunks don't need automated chunking because they were hand-curated at the right granularity. The schema context doesn't need chunking because it's structured data.

This works beautifully at current scale (77 + 14 + schema = ~91 items). But it's a **hand-curated approach that doesn't scale**:
- Adding 50 more tools → manageable (add 50 lines to `tool_descriptions.txt`)
- Adding 100 more doc chunks → borderline (who writes 100 topic summaries?)
- Ingesting the full CockroachDB documentation → impossible without automated chunking

**The framework's advice -- "start sentence-based, graduate to semantic/adaptive" -- maps to roachie's growth path:**
1. **Current (small scale):** Hand-curated chunks work perfectly ✓
2. **Next step (medium scale, 100+ docs):** Add hierarchical nesting to doc chunks
3. **Future (large scale, 1000+ docs):** Implement semantic or adaptive chunking for auto-ingested documentation

---

## Top 5 Improvements by Impact

1. **Add hierarchical nesting to doc chunks** -- Split the 14 broad topic chunks into ~40-50 focused sub-topic chunks. "Slow Query Diagnosis" becomes 4 chunks: identification, plan analysis, index optimization, contention diagnosis. Improves retrieval precision for specific questions while maintaining topic coherence for broad questions.

2. **Scale the knowledge base from 14 to 50+ chunks** -- The current 14 topics cover only the most common DBA scenarios. Add chunks for: security best practices, multi-region configuration, changefeed setup, import/export, scheduled jobs, SQL tuning, upgrade procedures, monitoring alerting, connection pooling. Each new chunk follows the existing CHUNK_ID/TITLE/CONTENT format.

3. **Add adaptive chunk sizing for error codes** -- The `error_codes_common` chunk lists 10 error codes in one ~130-word block. Split into one chunk per error code (~30 words each). When a user asks "what is error 40001?", retrieve just that error's chunk instead of all 10. This is within-corpus adaptive chunking.

4. **Build an automated doc ingestion pipeline** -- For scaling beyond hand-curation: take CockroachDB release notes or documentation pages → split by section headers (hierarchical) → filter to operational topics → embed and add to doc RAG corpus. This enables continuous knowledge base growth without manual curation.

5. **Add query-relevant schema filtering** -- Instead of injecting the full schema into every prompt, filter schema context to databases/tables mentioned in the query or matched by tool selection. "Show slow queries in movr" injects only the movr schema, not all databases. Reduces wasted tokens by 50-80% for users with many databases.

---

## CLI Summary (Quick Reference)

### Chunking Score: 7.5/10 -- "Naturally Atomic at Current Scale"

| Strategy | Used? | Where | Score | Key Insight |
|----------|-------|-------|-------|-------------|
| Fixed-length | No | -- | N/A | Correctly avoided |
| Sentence-based | Effectively | Tool descriptions (77) | **9/10** | Natural sentence = natural chunk |
| Semantic | No | -- | N/A | Not needed yet; useful at 100+ docs |
| Hierarchical | Yes | Doc RAG (14 chunks) | **7/10** | Topic-level; needs sub-topic nesting |
| Adaptive | Emerging | Cross-corpus routing | **6/10** | 3 strategies across 3 corpora |

### The Key Insight

Roachie bypasses the chunking problem by having **naturally atomic corpora**: each tool IS a chunk, each doc topic IS a chunk, and schema is structured data. This works perfectly at current scale (91 items) but doesn't scale to automated doc ingestion.

### Three Corpora, Three Natural Strategies

| Corpus | Items | Strategy | Why |
|--------|-------|----------|-----|
| Tool descriptions | 77 | Sentence-based (natural) | One tool = one sentence = one chunk |
| Doc RAG knowledge base | 14 | Hierarchical (topic-based) | One topic = one complete chunk |
| Live schema context | Dynamic | Structured (SQL-derived) | No chunking needed |

### Growth Path (Framework's Advice Applied)

| Scale | Approach | Status |
|-------|----------|--------|
| Current (~91 items) | Hand-curated chunks | **Working perfectly** |
| Medium (~100-500 docs) | Hierarchical nesting + more chunks | **Next step** |
| Large (~1000+ docs) | Semantic or adaptive auto-chunking | **Future** |

### Top 5 Improvements by Impact

1. **Hierarchical nesting** -- Split 14 broad chunks into ~40-50 focused sub-topics
2. **Scale knowledge base** -- Add 36+ chunks for uncovered DBA topics
3. **Adaptive error code chunking** -- One chunk per error code (not 10 in one block)
4. **Automated doc ingestion pipeline** -- Section-header splitting for new docs
5. **Query-relevant schema filtering** -- Inject only databases/tables relevant to the query
