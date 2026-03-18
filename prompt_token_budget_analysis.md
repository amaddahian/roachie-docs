# System Prompt Token Budget Analysis — 2026-03-18

Analysis of the roachie NL system prompt composition, measuring each component's contribution to total token usage. This informs R2 (Prompt Token Budget Management, P0).

## Table 1: Base Static Prompt Breakdown

The base prompt is built by `_nl_build_system_prompt()` and sent on **every query**.

| Component | Function | Chars | Est. Tokens | Notes |
|-----------|----------|------:|------------:|-------|
| Intro + topology | `_nl_build_system_prompt()` | ~500 | ~125 | Cluster description, timestamp, version |
| Parameter rules | `_nl_prompt_parameter_rules()` | ~4,170 | ~1,043 | Dual-param tools (ddl_diff, catalog_diff, ceph) |
| Tool catalog (77 tools) | `_nl_generate_tool_catalog()` | ~8,500 | ~2,125 | All tools with descriptions, grouped by category |
| Per-tool flag notes | `_nl_prompt_tool_notes()` | ~3,640 | ~910 | Expert knowledge: flag confusion, positional args |
| Execution rules + response format | `_nl_prompt_execution_rules()` | ~10,230 | ~2,558 | Rules, JSON format, guides (DDL, views, schema, etc.) |
| Hallucination warnings | (inline in `_nl_prompt_tools()`) | ~1,200 | ~300 | Skipped when native tool calling is active |
| **Total base static** | | **~27,700** | **~6,935** | |

## Table 2: Per-Query Enrichment Components

These are appended to the base prompt during `_enrich_system_prompt_with_tool_help()` and `_nl_build_prompt()`.

| # | Component | Chars | Est. Tokens | Source | Injected When |
|---|-----------|------:|------------:|--------|---------------|
| 1 | Tool --help output (up to 7 tools) | ~13,000 | ~3,265 | `_enrich_system_prompt_with_tool_help()` | Per query (matched tools) |
| 2 | Doc RAG chunks (up to 3) | ~4,200 | ~1,045 | `_doc_rag_retrieve()` | Per query (when embedding provider available) |
| 3 | Schema context | ~1,900-24,000 | ~475-6,000 | `_nl_fetch_schema_context()` | Cached at session start |
| 4 | Persistent learning (failures + successes) | ~3,000-9,000 | ~750-2,250 | `_nl_build_prompt()` | Per query |

### Component Details

**1. Tool --help output** — sampled from representative tools:

| Tool | Chars | Est. Tokens |
|------|------:|------------:|
| cr_tables | 942 | ~235 |
| cr_query_stats | 1,475 | ~368 |
| cr_ddl | 1,776 | ~444 |
| cr_columns | 1,434 | ~358 |
| cr_backup | 1,846 | ~461 |
| cr_ceph_storage | 4,312 | ~1,078 |
| cr_health | 1,275 | ~318 |
| **Total (7 tools)** | **13,060** | **~3,265** |

Range per tool: 942-4,312 chars. Average: ~1,866 chars (~466 tokens).

**2. Doc RAG chunks** — from pre-computed embeddings:

| Embedding File | Total Chunks | Avg Content/Chunk | Top 3 Chunks Total |
|----------------|:------------:|------------------:|-------------------:|
| openai_text-embedding-3-small.json | 14 | 1,246 chars (~311 tokens) | 4,181 chars (~1,045 tokens) |
| gemini_gemini-embedding-001.json | 14 | 1,246 chars (~311 tokens) | 4,181 chars (~1,045 tokens) |
| ollama (missing) | — | — | — |

**3. Schema context** — depends on cluster state:

| Cluster Size | Chars | Est. Tokens |
|--------------|------:|------------:|
| 3 databases x 10 tables | ~1,900 | ~475 |
| 5 databases x 25 tables | ~8,000 | ~2,000 |
| 8 databases x 50 tables (max) | ~24,000 | ~6,000 |

Per-database overhead: ~100 chars header + ~60 chars/table (schema.name: col1 type, col2 type, ...).
Limits: `_NL_MAX_SCHEMA_DATABASES=8`, `_NL_MAX_SCHEMA_TABLES=50`.

**4. Persistent learning** — loaded from JSONL databases:

| Sub-component | Max Entries | Est. Chars/Entry | Max Total |
|---------------|:-----------:|-----------------:|----------:|
| Persistent failures | 15 | ~300-400 | ~6,000 |
| Persistent successes | 10 | ~150-200 | ~2,000 |
| In-session failures | 3 | ~200 | ~600 |
| In-session successes | 3 | ~150 | ~450 |
| Personality instructions | 1 | ~300 | ~300 |
| **Max total** | | | **~9,350 (~2,338 tokens)** |

## Table 3: Total Prompt Size Scenarios

| Scenario | Base | Help | Doc RAG | Schema | Learning | **Total Chars** | **Total Tokens** |
|----------|-----:|-----:|--------:|-------:|---------:|----------------:|-----------------:|
| Minimum (no schema, no learning, 3 tools) | 27,700 | 5,600 | 0 | 0 | 0 | **33,300** | **~8,325** |
| Lean (no schema, light learning, 5 tools) | 27,700 | 9,300 | 3,100 | 0 | 1,500 | **41,600** | **~10,400** |
| Typical (3 DBs, some learning, 5 tools) | 27,700 | 9,300 | 4,200 | 1,900 | 3,000 | **46,100** | **~11,525** |
| Heavy (5 DBs, full learning, 7 tools) | 27,700 | 13,060 | 4,200 | 8,000 | 6,000 | **58,960** | **~14,740** |
| Maximum (8 DBs, full learning, 7 tools) | 27,700 | 13,060 | 4,200 | 24,000 | 9,000 | **77,960** | **~19,490** |

## Table 4: Provider Context Window Utilization

| Provider | Context Window | Typical Usage | Max Usage | Headroom |
|----------|---------------:|--------------:|----------:|---------:|
| Anthropic (Claude) | 200,000 | 11,525 (5.8%) | 19,490 (9.7%) | Abundant |
| Gemini | 1,000,000 | 11,525 (1.2%) | 19,490 (1.9%) | Abundant |
| OpenAI (GPT-4o) | 128,000 | 11,525 (9.0%) | 19,490 (15.2%) | Comfortable |
| Ollama (roachie-8b) | 4,096 | 11,525 (**281%**) | 19,490 (**476%**) | **OVERFLOWS** |
| Ollama compact prompt | 4,096 | ~3,500 (85%) | ~5,500 (**134%**) | **Tight/Overflow** |

## Key Findings

1. **Base static prompt is the largest single component at ~6,935 tokens** — sent on every query regardless of relevance. The 77-tool catalog with descriptions (~2,125 tokens) and per-tool flag notes (~910 tokens) are redundant with the enrichment phase that already appends full `--help` for matched tools.

2. **Tool --help output (~3,265 tokens for 7 tools) is the most valuable component** — it contains the exact flags the LLM needs to avoid hallucination. This should be preserved.

3. **Doc RAG is modest at ~1,045 tokens** for 3 chunks. Not a bloat culprit.

4. **Schema context is highly variable** — small clusters are fine (~475 tokens), but maxed-out config adds ~6,000 tokens. Consider dynamic truncation based on remaining token budget.

5. **Ollama overflows even with the compact prompt.** The compact prompt (~1,500 tokens base) plus enrichment (5 tools at ~2,330 tokens + learning) exceeds 4,096.

6. **Learning data can grow to ~2,338 tokens** with full failure/success history. No budget cap exists — it grows until max entries are reached.

## Recommended R2 Implementation

### Quick Wins (save ~2,750 tokens on every query)

1. **Replace tool catalog descriptions with names-only listing** (~2,125 -> ~289 tokens, saves ~1,836 tokens). Tool matching already selects relevant tools; the catalog serves only as a fallback reference.

2. **Move per-tool flag notes into enrichment** (~910 tokens saved from base). Only inject notes for tools that were matched, not all 77.

### Medium Effort

3. **Add per-provider token budget caps** — measure prompt size before API call, truncate learning data first, then schema context, to fit within budget.

4. **Dynamic schema truncation** — reduce `_NL_MAX_SCHEMA_TABLES` when remaining budget is tight (especially for Ollama).

### Ollama-Specific

5. **Cap Ollama enrichment to 3 tools** (not 5), skip Doc RAG, limit learning to 5 entries. Target: total prompt under 3,000 tokens to leave room for user query + response.

---

*Generated from analysis of `llm_prompt.sh` (1,807 lines), `llm_assistant.sh` (1,585 lines), `llm_config.sh` (47 lines), and `llm_metrics.sh` (692 lines).*
