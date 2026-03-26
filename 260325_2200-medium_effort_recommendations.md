# Medium-Priority Recommendations — Implementation Tracker

**Date:** 2026-03-25
**Source:** Combined Recommendations (260325_2100)
**Scope:** All 17 medium-effort items from Tier 1 and Tier 2 (excluding 4 already completed)
**Status:** All 17 items FIXED

---

## Tier 1 — Medium Effort, Highest Impact (6 items)

These were independently identified across 4+ analysis frameworks, making them the highest-confidence improvements.

| # | ID | Recommendation | Area | Effort | Reports | Status |
|---|-----|---------------|------|--------|---------|--------|
| 1 | T1-1 | **Goal decomposition / planning phase** — `plan` field in response_format.txt + display in agent loop | Agent Loop | Medium | 8 | **FIXED** |
| 2 | T1-4 | **Retrieval quality tracking / feedback loop** — enriched-vs-used correlation in debug log after execution | Retrieval | Medium | 6 | **FIXED** |
| 3 | T1-7 | **Cross-encoder re-ranking after retrieval** — `_nl_rerank_tools()` in llm_prompt.sh, opt-in via `NL_RERANK=1` | Retrieval | Medium | 5 | **FIXED** |
| 4 | T1-8 | **Session summary persistence / cross-session memory** — `~/.roachie/sessions/last_session.json` saved on exit, loaded on start | Memory | Medium | 5 | **FIXED** |
| 5 | T1-9 | **Capture user corrections as labeled training data** — `nl_corrections_persistent.jsonl` saved during reflexion, consumed by `--dpo` flag | Learning | Medium | 5 | **FIXED** |
| 6 | T1-10 | **Expand doc knowledge base (14 to 50+ chunks)** — 53 new doc chunks in `tools/utils/doc_chunks.txt` (67 total) | RAG | Medium | 5 | **FIXED** |

## Tier 2 — Medium Effort, Medium-High Impact (11 items)

These were identified across 2-3 analysis frameworks.

| # | ID | Recommendation | Area | Effort | Reports | Status |
|---|-----|---------------|------|--------|---------|--------|
| 7 | T2-1 | **DPO (Direct Preference Optimization) training** — `--dpo` flag in prepare_training_data.py, extracts pairs from corrections DB | Fine-Tuning | Medium | 3 | **FIXED** |
| 8 | T2-2 | **LLM-as-a-Judge in batch pipeline** — `--judge` / `--judge-provider` flags in roachie-batch, scores 1-5 per result | Evaluation | Low | 3 | **FIXED** |
| 9 | T2-3 | **Daemon/Watch mode for persistent monitoring** — `bin/roachie-watch` with YAML config, webhook alerts, daemon loop | Deployment | High | 3 | **FIXED** |
| 10 | T2-4 | **Audit logging for command executions** — `nl_audit.csv` with timestamp, user, role, tool, result, provider, trace_id | Governance | Medium | 3 | **FIXED** |
| 11 | T2-5 | **Confidence scoring in tool selection** — `confidence` field in response_format.txt + debug logging | Decision Making | Medium | 3 | **FIXED** |
| 12 | T2-10 | **Reciprocal Rank Fusion (RRF)** — RRF scoring `1/(k+rank)` replaces simple dedup in llm_prompt.sh enrichment | Retrieval | Medium | 2 | **FIXED** |
| 13 | T2-11 | **Self-evaluation after command execution** — low confidence (<0.5) triggers automatic follow-up in agent loop | Agent Loop | Medium | 3 | **FIXED** |
| 14 | T2-12 | **Tiered model selection by query complexity** — `_nl_tiered_model_override()` in llm_providers.sh, opt-in `NL_TIERED_MODELS=1` | LLM / Cost | Medium | 3 | **FIXED** |
| 15 | T2-13 | **Tool chain documentation** — `tool_chains.txt` with 6 common sequences, injected into system prompt | Prompts | Low | 2 | **FIXED** |
| 16 | T2-14 | **Result quality templates per tool category** — `result_templates.txt` with severity/ranking/structured templates per category | Output | Medium | 2 | **FIXED** |
| 17 | T2-15 | **Negative/multi-turn training examples** — 25 examples in `supplemental_examples.jsonl`, `--supplemental` flag in pipeline | Training | Medium | 2 | **FIXED** |

---

## Already Completed (from low-effort batch)

These Tier 2 items were implemented as part of the low-effort batch (260325_2130):

| ID | Recommendation | Status |
|-----|---------------|--------|
| T2-6 | Per-stage latency tracking | **FIXED** |
| T2-7 | Task-specific role modifiers | **FIXED** |
| T2-8 | Provenance metadata in embedding files | **FIXED** |
| T2-9 | Quality gate after embedding regen | **FIXED** |

---

## Implementation Notes

Each item will be updated to **FIXED** with a brief description of the change made.

### Suggested Implementation Order (by impact-to-effort ratio)

1. **T2-2** (LLM-as-a-Judge) — Low effort, high leverage for training quality
2. **T2-13** (Tool chain docs) — Low effort, improves multi-step reasoning
3. **T1-4** (Retrieval quality tracking) — correlation on existing CSV data
4. **T1-1** (Goal decomposition) — highest-frequency recommendation (8 reports)
5. **T1-9** (Corrections as training data) — connects existing feedback to pipeline
6. **T1-10** (Expand doc chunks) — content authoring, no code changes
7. **T1-8** (Session summary persistence) — file-based, straightforward
8. **T2-15** (Multi-turn training examples) — content authoring for training set
9. **T2-5** (Confidence scoring) — JSON schema + prompt change
10. **T2-11** (Self-evaluation) — extends agent loop with quality check
11. **T2-4** (Audit logging) — CSV append in execution path
12. **T2-12** (Tiered model selection) — routing logic in provider dispatcher
13. **T2-14** (Result quality templates) — prompt templates per category
14. **T2-10** (RRF) — scoring formula change in enrichment
15. **T2-1** (DPO training) — training pipeline extension
16. **T1-7** (Cross-encoder re-ranking) — requires additional model/API
17. **T2-3** (Daemon/Watch mode) — largest scope, new binary

---

## CLI Summary

```
Table 1: Medium-Priority Recommendations — Tier 1 (4+ reports)

#   ID     Recommendation                                  Area         Effort  Reports  Status
──  ─────  ──────────────────────────────────────────────  ───────────  ──────  ───────  ───────
1   T1-1   Goal decomposition / planning phase             Agent Loop   Medium  8        FIXED
2   T1-4   Retrieval quality tracking / feedback loop       Retrieval    Medium  6        FIXED
3   T1-7   Cross-encoder re-ranking after retrieval         Retrieval    Medium  5        FIXED
4   T1-8   Session summary persistence / cross-session mem  Memory       Medium  5        FIXED
5   T1-9   Capture user corrections as training data        Learning     Medium  5        FIXED
6   T1-10  Expand doc knowledge base (14 → 50+ chunks)     RAG          Medium  5        FIXED
```

```
Table 2: Medium-Priority Recommendations — Tier 2 (2-3 reports)

#    ID     Recommendation                                  Area          Effort  Reports  Status
──   ─────  ──────────────────────────────────────────────  ────────────  ──────  ───────  ───────
7    T2-1   DPO (Direct Preference Optimization) training   Fine-Tuning   Medium  3        FIXED
8    T2-2   LLM-as-a-Judge in batch pipeline                Evaluation    Low     3        FIXED
9    T2-3   Daemon/Watch mode for persistent monitoring     Deployment    High    3        FIXED
10   T2-4   Audit logging for command executions            Governance    Medium  3        FIXED
11   T2-5   Confidence scoring in tool selection            Decisions     Medium  3        FIXED
12   T2-10  Reciprocal Rank Fusion (RRF)                    Retrieval     Medium  2        FIXED
13   T2-11  Self-evaluation after command execution         Agent Loop    Medium  3        FIXED
14   T2-12  Tiered model selection by query complexity      LLM / Cost    Medium  3        FIXED
15   T2-13  Tool chain documentation                        Prompts       Low     2        FIXED
16   T2-14  Result quality templates per tool category      Output        Medium  2        FIXED
17   T2-15  Negative/multi-turn training examples           Training      Medium  2        FIXED
```

```
Table 3: Already Completed (from low-effort batch)

ID    Recommendation                           Status
────  ─────────────────────────────────────    ──────
T2-6  Per-stage latency tracking               FIXED
T2-7  Task-specific role modifiers             FIXED
T2-8  Provenance metadata in embedding files   FIXED
T2-9  Quality gate after embedding regen       FIXED
```

```
Table 4: Implementation Summary

#   ID     Implementation Detail                                                      Files Changed
──  ─────  ────────────────────────────────────────────────────────────────────────   ──────────────────────
1   T1-1   plan field in response_format.txt + display in agent loop                  llm_assistant.sh, response_format.txt
2   T1-4   enriched-vs-used tool correlation in debug log                             llm_assistant.sh
3   T1-7   _nl_rerank_tools() LLM re-ranking, opt-in NL_RERANK=1                     llm_prompt.sh, llm_config.sh
4   T1-8   ~/.roachie/sessions/last_session.json saved/loaded                         llm_assistant.sh
5   T1-9   nl_corrections_persistent.jsonl + --dpo in training pipeline               llm_metrics.sh, llm_assistant.sh, prepare_training_data.py
6   T1-10  53 new doc chunks in doc_chunks.txt (67 total)                             tools/utils/doc_chunks.txt
7   T2-1   --dpo flag extracts DPO pairs from correction logs                         prepare_training_data.py
8   T2-2   --judge / --judge-provider flags in roachie-batch                          bin/roachie-batch
9   T2-3   bin/roachie-watch daemon with YAML config + webhooks                       bin/roachie-watch (new)
10  T2-4   nl_audit.csv logging with _nl_audit_command()                              llm_metrics.sh, llm_assistant.sh
11  T2-5   confidence field in response_format.txt + debug logging                    response_format.txt, llm_assistant.sh
12  T2-10  RRF scoring 1/(k+rank) replaces simple dedup                               llm_prompt.sh
13  T2-11  Low confidence (<0.5) triggers automatic follow-up                          llm_assistant.sh
14  T2-12  _nl_tiered_model_override(), opt-in NL_TIERED_MODELS=1                     llm_providers.sh, llm_assistant.sh, llm_config.sh
15  T2-13  tool_chains.txt with 6 common sequences                                    tools/utils/prompts/tool_chains.txt
16  T2-14  result_templates.txt with severity/ranking templates                        tools/utils/prompts/result_templates.txt
17  T2-15  25 supplemental examples + --supplemental flag                              supplemental_examples.jsonl, prepare_training_data.py
```

```
Table 5: Summary Statistics

Category                 Count    Status
───────────────────────  ─────    ──────────
Tier 1 (4+ reports)      6        ALL FIXED
Tier 2 (2-3 reports)     11       ALL FIXED
───────────────────────  ─────    ──────────
Total implemented        17       100%
Previously completed     4        (from low-effort batch)
Grand total              21       ALL FIXED

New files created:
  bin/roachie-watch                              Daemon/watch mode binary
  tools/utils/doc_chunks.txt                     53 new doc RAG chunks
  tools/utils/prompts/tool_chains.txt            6 multi-step workflow chains
  tools/utils/prompts/result_templates.txt       Output quality templates per category
  tools/utils/finetune/supplemental_examples.jsonl  25 training examples (info + multi-turn)

New env vars (all opt-in):
  NL_TIERED_MODELS=1    Route simple queries to cheaper models
  NL_RERANK=1           Enable cross-encoder re-ranking via LLM
  NL_RERANK_PROVIDER    Provider for re-ranking (default: gemini)
```
