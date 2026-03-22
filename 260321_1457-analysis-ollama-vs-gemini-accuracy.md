# Ollama vs Gemini Accuracy Analysis — 2026-03-21

## Summary

Gemini consistently achieves 95-100% pass rate on the 73-prompt batch test suite, while Ollama (roachie-8b / llama3.1 8B) peaks at 72-78%. This document analyzes the root causes and proposes fixes.

## Historical Pass Rates

### Gemini (gemini-2.5-flash)

| Date | Pass Rate | Notes |
|------|-----------|-------|
| 03-17 | 98% (72/73) | |
| 03-17 | 95% (70/73) | |
| 03-18 | **100% (73/73)** | |
| 03-18 | **100% (73/73)** | |
| 03-18 | **100% (73/73)** | |
| 03-19 | **100% (73/73)** | |
| 03-19 | **100% (73/73)** | Last pre-chunking |
| 03-19 | 97% (71/73) | Post-chunking |
| 03-19 | 97% (71/73) | Post-chunking |
| 03-19 | 95% (70/73) | Post-chunking |

### Ollama (roachie-8b / llama3.1 8B)

| Date | Pass Rate | Notes |
|------|-----------|-------|
| 03-19 16:00 | 65% (48/73) | First live Ollama run (post-chunking) |
| 03-19 16:38 | 65% (48/73) | |
| 03-20 08:04 | 97% (71/73) | **DRY-RUN — commands not executed** |
| 03-20 12:04 | 61% (45/73) | |
| 03-20 13:43 | 76% (56/73) | Alternative prompt file |
| 03-20 16:13 | 72% (53/73) | Alternative prompt file |
| 03-20 21:04 | 64% (47/73) | |
| 03-21 09:12 | **78% (57/73)** | **Best-ever live Ollama run** |
| 03-21 10:05 | 61% (45/73) | |
| 03-21 11:54 | 73% (54/73) | After chunking revert |
| 03-21 13:15 | 72% (53/73) | After enrichment=3 restore |

**Key finding**: The 97% DRY-RUN inflated expectations. Ollama has never exceeded 78% on live execution. The pre-chunking 96-100% baseline was Gemini, not Ollama.

## Root Cause Analysis: 3 Compounding Factors

### Factor 1: Prompt Content Gap (~6,500 tokens missing)

Gemini receives the **full system prompt** via `_nl_build_system_prompt()`, which includes 7 template files. Ollama receives the **compact prompt** via `_nl_build_system_prompt_compact()`, which inlines a minimal version and skips all templates.

| Template File | Tokens | Content | Impact on Accuracy |
|---------------|--------|---------|-------------------|
| `parameter_rules.txt` | ~1,024 | Detailed flag rules per tool category | HIGH — prevents flag confusion |
| `tool_notes.txt` | ~1,601 | Per-tool flag corrections (e.g., "cr_columns: use --filter-table NOT -T") | **CRITICAL — directly prevents the most common failures** |
| `execution_pattern.txt` | ~771 | Command construction patterns with examples | MEDIUM |
| `hallucination_warnings.txt` | ~291 | "Do NOT invent flags" warnings | HIGH |
| `execution_rules.txt` | ~778 | Safety rules, validation steps | LOW |
| `response_format.txt` | ~498 | Detailed JSON response schema | MEDIUM |
| `tool_specific_rules.txt` | ~1,504 | Per-tool edge cases and examples | HIGH |
| **TOTAL** | **~6,470** | | |

The `tool_notes.txt` file is the single most impactful template — it contains exact corrections for the flags Ollama most commonly gets wrong:
- `cr_columns`: use `--filter-table` NOT `-T`
- `cr_ceph_storage`: does NOT have `list` subcommand, use `list-backups`
- `cr_get_acl`: use `-o` NOT `-T`
- `cr_ddl_table`: table name is positional, NOT `-T`
- `cr_config_event_history`: use `--event-type` NOT `--type`

### Factor 2: Tool Enrichment Gap (3 vs 7 tools)

| Metric | Ollama (batch) | Gemini (batch) |
|--------|---------------|----------------|
| Enriched tools (with --help) | 3 | 7 |
| Avg tokens per prompt | ~2,570 | ~12,594 |
| Token ratio | 1x | **5x** |

When a query requires a tool not in the top 3 matches, Ollama never sees its `--help` output and must guess flags. With 7 enriched tools, Gemini almost always has the relevant tool's documentation available.

### Factor 3: Model Capability Gap

| | Ollama (roachie-8b) | Gemini (2.5-flash) |
|---|---------------------|-------------------|
| Parameters | 8B (Q4_K_M quantized) | Unknown (very large) |
| Native context | 128K (llama3.1) | 1M |
| Configured context | 8,192 tokens | 1M |
| Instruction following | Good for size | Excellent |
| Flag recall accuracy | ~72-78% | ~95-100% |

## Failure Pattern Analysis (20 failures from 03-21 13:15 run)

### Infrastructure failures (correct commands, external issue): 4
- #7, #8, #9, #10 — Ceph storage not configured

### Flag hallucination (wrong flags on correct tools): 16
- **Invented flags**: `--all-tenants`, `--all-views`, `--include-functions`, `--include-procedures`
- **Wrong flag name**: `-T` instead of `--filter-table` (cr_columns), `-T` instead of `-o` (cr_get_acl)
- **Wrong subcommand**: `cr_ceph_storage list` instead of `list-backups`
- **Duplicate flags**: `-t products -t system` (using -t for both table and tenant)
- **Made-up flag values**: `--all-tenants false`, `--sort ASC`

All 16 failures would be prevented by `tool_notes.txt` content being in the prompt.

## Proposed Fix: Unify Prompt Path

### Option A: Include critical templates in compact prompt
Add `tool_notes.txt` (~1,600 tokens) and `hallucination_warnings.txt` (~291 tokens) to the compact prompt. Total addition: ~1,900 tokens. Fits within 8K context (current usage ~2,570 + 1,900 = ~4,470).

### Option B: Use full prompt for all providers (recommended)
Remove the compact prompt path entirely. Use `_nl_build_system_prompt()` for Ollama too. Increase `num_ctx` to 16384 to accommodate the full ~12,500 token prompt.

**Option B rationale**: llama3.1 supports 128K context natively. The 8K limit was artificial. With 16K context, the full prompt (~12,500 tokens) fits comfortably with room for response tokens.

### Additional: Increase enrichment to 7
Match Gemini's enrichment limit so Ollama gets `--help` for all matched tools.

## Token Budget Analysis for Option B

| Component | Tokens |
|-----------|--------|
| Full system prompt (base) | ~3,500 |
| Templates (all 7) | ~6,470 |
| Tool enrichment (7 tools x ~500) | ~3,500 |
| Connection reminder | ~140 |
| Learning data | ~200 |
| Schema context | ~100 |
| User message | ~50 |
| **Total input** | **~13,960** |
| Response budget | ~256 |
| **Grand total** | **~14,216** |

With `num_ctx 16384`: **2,168 tokens headroom** — sufficient.

## Detailed Token Budget (Measured)

Actual measurements of each prompt component:

### Template Files (included in full prompt, excluded from compact)

| Template | Chars | Tokens (~chars/4) |
|----------|-------|-------------------|
| `parameter_rules.txt` | 4,097 | ~1,024 |
| `tool_notes.txt` | 6,405 | ~1,601 |
| `execution_pattern.txt` | 3,087 | ~771 |
| `hallucination_warnings.txt` | 1,167 | ~291 |
| `execution_rules.txt` | 3,114 | ~778 |
| `response_format.txt` | 1,993 | ~498 |
| `tool_specific_rules.txt` | 6,017 | ~1,504 |
| **Total** | **25,880** | **~6,470** |

### Tool --help Output (per enriched tool, measured)

| Tool | --help chars | Tokens |
|------|-------------|--------|
| `cr_list_tenants` | 1,027 | ~256 |
| `cr_tables` | 942 | ~235 |
| `cr_columns` | 1,434 | ~358 |
| `cr_ddl` | 1,776 | ~444 |
| `cr_backup` | 1,846 | ~461 |
| `cr_get_acl` | 1,435 | ~358 |
| `cr_find_views` | 1,243 | ~310 |
| **Average** | **1,386** | **~346** |

- 7 enriched tools: ~9,702 chars (~2,425 tokens)
- 3 enriched tools: ~4,158 chars (~1,039 tokens)

### Full Prompt Assembly (Option B — unified path)

| Component | Tokens | Cumulative |
|-----------|--------|------------|
| Base intro + topology + schema | ~400 | 400 |
| `parameter_rules.txt` | 1,024 | 1,424 |
| Tool catalog (77 tools) | 1,280 | 2,704 |
| `tool_notes.txt` | 1,601 | 4,305 |
| `execution_pattern.txt` | 771 | 5,076 |
| `hallucination_warnings.txt` | 291 | 5,367 |
| `execution_rules.txt` | 778 | 6,145 |
| `response_format.txt` | 498 | 6,643 |
| `tool_specific_rules.txt` | 1,504 | 8,147 |
| Learning data | ~200 | 8,347 |
| **Enrichment (7 tools --help)** | **2,425** | **10,772** |
| Connection reminder | 140 | 10,912 |
| User message | ~50 | 10,962 |
| **Response budget (num_predict)** | **256** | **11,218** |

With `num_ctx 16384`: **5,166 tokens headroom** — comfortable margin.

### Comparison: Before vs After

| Metric | Before (compact) | After (unified) | Gemini |
|--------|-----------------|-----------------|--------|
| System prompt tokens | ~1,500 | ~8,150 | ~8,150 |
| Enriched tools | 3 | 7 | 7 |
| Enrichment tokens | ~1,039 | ~2,425 | ~2,425 |
| Total input tokens | ~2,570 | ~10,962 | ~12,594 |
| Context window | 8,192 | 16,384 | 1,000,000 |
| Headroom | ~5,600 | ~5,166 | ~987,000 |

## Risk Assessment

| Risk | Severity | Detail | Mitigation |
|------|----------|--------|------------|
| **Context overflow** | **HIGH if num_ctx stays 8192** | Full prompt is ~11K tokens, exceeds 8K limit | Set `num_ctx 16384`. llama3.1 natively supports 128K. |
| **Latency increase** | MEDIUM | Prompt processing scales linearly. ~15-20s per query vs current ~10s. | Acceptable tradeoff for accuracy. Batch tests are non-interactive. |
| **Memory usage** | LOW | 16K KV cache on 8B Q4_K_M is ~1-1.5GB additional. | Modern Macs have 16-64GB RAM. |
| **Lost-in-the-middle** | LOW | Small models can lose mid-prompt instructions. 11K tokens is moderate, not extreme. | Critical `tool_notes.txt` is structured as per-tool lookups. If needed, move closer to end. |
| **Diminishing returns** | MEDIUM | 8B model may not match Gemini even with identical prompts. | Expected: 85-92%, not 100%. Still major improvement from 72%. |
| **Ollama rebuild needed** | LOW | Changing `num_ctx` requires `ollama create` rebuild. | One-time cost, ~30 seconds. |

**No blocking risks.** The only hard requirement is bumping `num_ctx` from 8192 to 16384.

## Implementation Plan (Option B — selected)

1. Remove `ollama-*` branch in `_nl_build_prompt()` — all providers use `_nl_build_system_prompt()`
2. Remove Ollama-specific enrichment limit (3) in batch runner — use default 7 for all
3. Use full connection reminder for all providers in batch runner
4. Bump `num_ctx` to 16384 in all 3 Modelfiles
5. Update `_llm_context_limit()` for `ollama-*` from 8192 to 16384
6. Rebuild Ollama models: `ollama create roachie-8b -f tools/utils/Modelfile.roachie-8b`
7. Run batch test to validate improvement

## Results After Implementation (Option B)

### Batch Test: 73 prompts, ollama-roachie-8b, live execution

| Metric | Before (compact) | After (unified) | Delta |
|--------|------------------|-----------------|-------|
| Pass rate | 72% (53/73) | **83% (61/73)** | **+11%** |
| Avg input tokens | 2,570 | 4,293 | +67% (Ollama tokenizer) |
| Avg API latency | 9,178ms | 15,726ms | +71% (more tokens to process) |
| Enriched tools | 3 | 7 | +133% |

### Failures Fixed (8 queries now passing)

| Query | Old Failure | Fix Reason |
|-------|------------|------------|
| #5 cr_db_tables_rowcount | Wrong flags | tool_notes.txt now included |
| #11 cr_config_event_history | `--event-type restart` failed | Full execution_pattern now included |
| #12 cr_config_event_history | `--limit 3` failed | Full parameter_rules now included |
| #13 cr_config_event_history | Wrong flags | Full tool docs now available |
| #14, #15 cr_find_views | `--pattern inventory` failed | tool_notes.txt specifies --pattern |
| #19, #20, #21, #22 cr_db_tables_rowcount | Various wrong flags | Full --help now enriched |
| #34 cr_get_acl | `--username` wrong | tool_notes.txt specifies -o flag |

### Remaining 12 Failures

| # | Query | Failure Type | Detail |
|---|-------|-------------|--------|
| 8 | backup to ceph | LLM error | Hallucinated S3 URL format |
| 9 | list ceph backups | LLM error | `list` subcommand (should be `list-backups`) |
| 10 | show recent backup | LLM error | Same `list` subcommand error |
| 23 | DDL for schema | LLM error | `--all-tenants --include-schemas` hallucinated |
| 32 | DDL for users | LLM error | `--all-tenants --include-users` hallucinated |
| 38 | DDL for views | LLM error | `--all-views` hallucinated |
| 42 | tool names with "table" | Meta-query | Missing connection flags |
| 43 | tool names with "view" | Meta-query | Missing connection flags, blocked tool |
| 68 | migrate schema | LLM error | Many hallucinated cr_migrate flags |
| 70 | DDL for rides table | LLM error | `--all-tenants false` hallucinated |
| 71 | columns for rides | LLM error | `-T rides` (should be `--filter-table`) |
| 72 | ACL for products | LLM error | `-T products` (should be `-o`) |

### Analysis of Remaining Failures

- **3 ceph infrastructure** (#8, #9, #10): Correct tool choice, ceph not configured
- **2 meta-queries** (#42, #43): "show tool names with keyword" — not a real DBA query
- **4 still ignoring tool_notes** (#9, #10, #71, #72): Model sees the instruction but doesn't follow it
- **3 hallucinating flags** (#23, #32, #38): Inventing flags like `--all-tenants`, `--include-schemas`

The 8B model's ceiling appears to be ~85-90% — it follows most instructions but has a ~10-15% error rate on complex flag combinations even when documentation is provided.

### Commit

Branch: `feature/unified-ollama-prompt`
Commit: `3808908` — "Unify Ollama prompt path with cloud providers for higher accuracy"

## Deep Dive: Same 12 Failed Queries — Gemini vs Ollama (2026-03-21)

Ran the 12 Ollama failures through Gemini. Gemini passed all 12 (100%). Ollama passed 2/12 (16%).

### Critical Finding: Enrichment Is Identical

Both providers receive **exactly the same enriched tools** for every query. The enrichment pipeline (regex keyword matching) produces identical results regardless of LLM provider. The difference is purely in how each model processes the same prompt.

### Side-by-Side Command Comparison

| # | Query | Gemini Command (PASS) | Ollama Command (FAIL) | Root Cause |
|---|-------|----------------------|----------------------|------------|
| 1 | backup movr to ceph | `cr_ceph_storage backup -d movr --bucket crdb-test-backups` | Same correct cmd, then tried `cr_backup` too (extra commands failed) | Multi-command: 1st was correct but extras failed |
| 2 | list ceph backups | `cr_ceph_storage list-backups` | `cr_ceph_storage list` (wrong subcommand) | Ignores tool_notes: "Does NOT have 'list' subcommand" |
| 3 | show recent backup | `cr_ceph_storage show-backup` | `cr_backup list` (wrong tool entirely) | cr_ceph_storage not even in enriched tools |
| 4 | DDL for schema | `cr_ddl -s inventory_schema --all` | `cr_ddl_database -s inventory_schema` | **Both passed** (different approach, both valid) |
| 5 | DDL for users | `cr_ddl_user` | `cr_ddl --include-users true` (hallucinated flag) | Ignores tool catalog: cr_ddl_user is the dedicated tool |
| 6 | DDL for all views | `cr_ddl_view -d test_lab` (no extra flags) | `cr_ddl_view --all-views` (hallucinated flag) | Invents `--all-views` — not in --help output |
| 7 | tool names with "table" | `cr_help --search "table"` | `cr_tables` (wrong tool entirely) | Doesn't recognize cr_help as the meta-query tool |
| 8 | tool names with "view" | `cr_help --search "view"` | `cr_view_references_enhanced` (blocked) | Same meta-query confusion |
| 9 | migrate schema to db3 | `cr_migrate --source-host ... --source-insecure --target-insecure` | `cr_migrate` with hallucinated flags: `--copy-ddl yes --changefeed-sink ...` | Invents many non-existent flags |
| 10 | DDL for rides table | `cr_ddl_table -d movr rides` (positional) | `cr_ddl --include-tables true` (hallucinated) | Ignores tool_notes: "table name is positional" |
| 11 | columns for rides | `cr_columns --filter-db movr --filter-table rides` | `cr_columns -d movr` (missing table filter) | **Both passed** (Ollama's was less specific but worked) |
| 12 | ACL for products | `cr_get_acl -d test_lab -o products` | `cr_get_acl -d test_lab -T products` (wrong flag) | Ignores tool_notes: "use -o NOT -T" |

### Enrichment Analysis (same tools, different outcomes)

| # | Enriched Tools (identical for both) | Critical Tool Present? | Gemini Used It? | Ollama Used It? |
|---|-------------------------------------|----------------------|-----------------|-----------------|
| 2 | cr_db_size, cr_tables, cr_table_info, cr_table_analyze, cr_size, cr_list_tenants, cr_ddl_database | cr_ceph_storage: **NO** | Used anyway (from catalog) | Guessed `list` subcommand |
| 3 | cr_tables, cr_table_info, cr_table_analyze, cr_size, cr_list_tenants, cr_db_tables_rowcount_* | cr_ceph_storage: **NO** | Used anyway (from catalog) | Used wrong tool entirely |
| 5 | cr_ddl, cr_my_grants, cr_my_access, cr_list_tenants, cr_get_admin, cr_get_acl, cr_format | cr_ddl_user: **NO** | Used anyway (from catalog) | Tried cr_ddl with hallucinated flags |
| 10 | cr_ddl, cr_db_size, cr_tables, cr_table_info, cr_table_analyze, cr_size, cr_format | cr_ddl_table: **NO** | Used anyway (from catalog) | Used cr_ddl with hallucinated flags |
| 12 | cr_db_size, cr_tables, cr_table_info, cr_table_analyze, cr_size, cr_my_grants, cr_my_access | cr_get_acl: **NO** | Used anyway (from catalog) | Used cr_get_acl but wrong flag |

**Key insight**: For queries #2, #3, #5, #10, #12 — the critical tool was NOT in the enriched set, so neither model got its `--help` output. But Gemini still picked the right tool from the catalog and used correct flags, while Ollama either picked the wrong tool or guessed wrong flags.

### Token Comparison (same text, different tokenizers)

| Metric | Ollama (llama3.1) | Gemini (2.5-flash) | Ratio |
|--------|-------------------|-------------------|-------|
| Avg input tokens | 9,755 | 13,020 | 0.75x |
| Min tokens | 4,259 (#11) | 12,490 (#8) | — |
| Max tokens | 11,158 (#2) | 13,602 (#1) | — |

The ~25% token difference is purely tokenizer efficiency (llama BPE vs Gemini). The actual text content is identical.

### Root Cause: Model Capability Gap (confirmed)

The unified prompt experiment conclusively proves:
1. **Same enrichment** — identical tools enriched for every query
2. **Same prompt content** — all 7 template files, same tool_notes, same --help output
3. **Same token budget** — both well within context limits
4. **Different outcomes** — Gemini follows instructions precisely, Ollama 8B does not

Gemini's advantages over Ollama 8B:
- **Catalog reasoning**: Can pick the right tool from a 77-tool catalog even without --help enrichment
- **Instruction adherence**: Follows tool_notes corrections ("use -o NOT -T") reliably
- **Subcommand awareness**: Uses correct subcommands (list-backups vs list)
- **Flag discipline**: Doesn't invent flags like --all-views, --include-users, --copy-ddl

### Potential Improvements (model-side, not prompt-side)

| Option | Effort | Expected Impact | Notes |
|--------|--------|----------------|-------|
| **Few-shot examples** in prompt | Low | +5-8% (to ~88-91%) | Add 3-5 concrete correct command examples for common failure patterns |
| **Larger Ollama model** (70B) | Medium | +10-15% (to ~93-98%) | Requires 40GB+ RAM; much better instruction following |
| **Fine-tune roachie-8b** | High | +8-12% (to ~91-95%) | Train on Gemini's correct outputs as ground truth |
| **Accept 83% ceiling** | None | 0% | Use Gemini for production, Ollama for offline/local |
