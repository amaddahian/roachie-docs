# Medium-Priority Recommendations — Implementation Summary

**Date:** 2026-03-25
**Scope:** All 17 medium-effort items from Tier 1 and Tier 2
**Status:** All 17 items FIXED

---

## Implementation Summary — All 17 Medium-Priority Items

### Tier 1 (Highest Impact)
| # | ID | What was done |
|---|-----|--------------|
| 1 | T1-1 | `plan` field in response_format.txt + display in agent loop |
| 2 | T1-4 | Enriched-vs-used tool correlation logged via `_nl_debug "RETRIEVAL_QUALITY"` |
| 3 | T1-7 | `_nl_rerank_tools()` — LLM-based cross-encoder re-ranking, opt-in `NL_RERANK=1` |
| 4 | T1-8 | Session summary saved to `~/.roachie/sessions/last_session.json`, loaded at next start |
| 5 | T1-9 | Reflexion corrections saved to `nl_corrections_persistent.jsonl`, consumed by `--dpo` flag |
| 6 | T1-10 | 53 new doc chunks in `tools/utils/doc_chunks.txt` (67 total, target was 50+) |

### Tier 2 (Medium-High Impact)
| # | ID | What was done |
|---|-----|--------------|
| 7 | T2-1 | `--dpo` flag in `prepare_training_data.py` extracts DPO preference pairs |
| 8 | T2-2 | `--judge` / `--judge-provider` in roachie-batch, 1-5 quality scoring |
| 9 | T2-3 | `bin/roachie-watch` — daemon with YAML config, webhook alerts, `--once` mode |
| 10 | T2-4 | `_nl_audit_command()` logs to `nl_audit.csv` with 11 columns |
| 11 | T2-5 | `confidence` field in response_format.txt + debug logging |
| 12 | T2-10 | RRF scoring `1/(k+rank)` replaces simple semantic-first dedup |
| 13 | T2-11 | Low confidence (<0.5) triggers automatic self-evaluation follow-up |
| 14 | T2-12 | `_nl_tiered_model_override()` routes simple queries to cheaper models (opt-in) |
| 15 | T2-13 | `tool_chains.txt` with 6 common diagnostic workflows |
| 16 | T2-14 | `result_templates.txt` with severity/ranking/structured output templates |
| 17 | T2-15 | 25 supplemental examples (17 informational + 8 multi-turn), `--supplemental` flag |

### Test Results
- **38/38 test suites pass** (100%)
- **ShellCheck clean** on all modified files (no new warnings)

---

## Files Changed

### Modified (8 files)

| File | Changes |
|------|---------|
| `bin/roachie-batch` | T2-2: `--judge` / `--judge-provider` flags, `_batch_judge_response()` function |
| `src/lib/llm_assistant.sh` | T1-8: session persistence, T2-4: audit hooks, T2-11: self-eval, T2-12: tiered model, T1-9: correction pairs |
| `src/lib/llm_config.sh` | T2-12: `_NL_TIERED_MODELS`, T1-7: `_NL_RERANK` constants |
| `src/lib/llm_metrics.sh` | T2-4: `_nl_audit_command()` + `_nl_init_audit_log()`, T1-9: `_save_correction_pair()` |
| `src/lib/llm_prompt.sh` | T2-10: RRF scoring, T1-7: `_nl_rerank_tools()`, T2-14: result_templates.txt injection |
| `src/lib/llm_providers.sh` | T2-12: `_nl_tiered_model_override()` |
| `tools/utils/finetune/prepare_training_data.py` | T2-1: `--dpo` flag + `extract_dpo_pairs()`, T2-15: `--supplemental` flag |
| `tools/utils/prompts/response_format.txt` | T1-1: `plan` field, T2-5: `confidence` field |

### Created (5 files)

| File | Purpose |
|------|---------|
| `bin/roachie-watch` | T2-3: Daemon/watch mode with YAML config, webhook alerts, `--once` mode |
| `tools/utils/doc_chunks.txt` | T1-10: 53 new doc RAG chunks (security, multi-region, changefeeds, etc.) |
| `tools/utils/prompts/tool_chains.txt` | T2-13: 6 common multi-step diagnostic workflows |
| `tools/utils/prompts/result_templates.txt` | T2-14: Output quality templates per tool category |
| `tools/utils/finetune/supplemental_examples.jsonl` | T2-15: 25 training examples (17 informational + 8 multi-turn) |

---

## New Environment Variables (all opt-in)

| Variable | Default | Description |
|----------|---------|-------------|
| `NL_TIERED_MODELS` | `0` | Route simple queries to cheaper models (T2-12) |
| `NL_RERANK` | `0` | Enable cross-encoder re-ranking via LLM (T1-7) |
| `NL_RERANK_PROVIDER` | `gemini` | Provider for re-ranking LLM calls |

---

```
Table 1: Tier 1 Implementation Details

#   ID     Area         Implementation
──  ─────  ───────────  ──────────────────────────────────────────────────────────────────────────────
1   T1-1   Agent Loop   plan field in response_format.txt + display in agent loop
2   T1-4   Retrieval    enriched-vs-used tool correlation in debug log after execution
3   T1-7   Retrieval    _nl_rerank_tools() LLM re-ranking, opt-in NL_RERANK=1
4   T1-8   Memory       ~/.roachie/sessions/last_session.json saved on exit, loaded on start
5   T1-9   Learning     nl_corrections_persistent.jsonl + --dpo flag in training pipeline
6   T1-10  RAG          53 new doc chunks in doc_chunks.txt (67 total)
```

```
Table 2: Tier 2 Implementation Details

#    ID     Area          Implementation
──   ─────  ────────────  ──────────────────────────────────────────────────────────────────────────────
7    T2-1   Fine-Tuning   --dpo flag extracts DPO pairs from correction logs
8    T2-2   Evaluation    --judge / --judge-provider flags in roachie-batch, 1-5 scoring
9    T2-3   Deployment    bin/roachie-watch daemon with YAML config + webhook alerts
10   T2-4   Governance    nl_audit.csv logging with _nl_audit_command() (11 columns)
11   T2-5   Decisions     confidence field in response_format.txt + debug logging
12   T2-10  Retrieval     RRF scoring 1/(k+rank) replaces simple dedup
13   T2-11  Agent Loop    Low confidence (<0.5) triggers automatic follow-up
14   T2-12  LLM / Cost    _nl_tiered_model_override(), opt-in NL_TIERED_MODELS=1
15   T2-13  Prompts       tool_chains.txt with 6 common sequences
16   T2-14  Output        result_templates.txt with severity/ranking templates per category
17   T2-15  Training      25 supplemental examples + --supplemental flag in pipeline
```

```
Table 3: New Files Created

File                                                  Purpose                                    Size
────────────────────────────────────────────────────  ─────────────────────────────────────────   ──────
bin/roachie-watch                                     Daemon/watch mode binary                   ~220 lines
tools/utils/doc_chunks.txt                            53 new doc RAG chunks                      ~109 lines
tools/utils/prompts/tool_chains.txt                   6 multi-step workflow chains                ~37 lines
tools/utils/prompts/result_templates.txt              Output quality templates per category        ~30 lines
tools/utils/finetune/supplemental_examples.jsonl      25 training examples (info + multi-turn)    25 lines
```

```
Table 4: Summary Statistics

Category                          Count    Status
──────────────────────────────    ─────    ──────────
Tier 1 items (4+ reports)          6       ALL FIXED
Tier 2 items (2-3 reports)         11      ALL FIXED
──────────────────────────────    ─────    ──────────
Total implemented                  17      100%
Previously completed (low-effort)  4       FIXED
Grand total recommendations        21      ALL FIXED

Files modified:    8
Files created:     5
Total lines added: ~590
Test suites:       38/38 pass (100%)
ShellCheck:        Clean (0 new warnings)
```
