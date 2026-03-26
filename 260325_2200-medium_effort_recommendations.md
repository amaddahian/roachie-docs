# Medium-Priority Recommendations — Implementation Tracker

**Date:** 2026-03-25
**Source:** Combined Recommendations (260325_2100)
**Scope:** All 17 medium-effort items from Tier 1 and Tier 2 (excluding 4 already completed)
**Status:** Implementation pending

---

## Tier 1 — Medium Effort, Highest Impact (6 items)

These were independently identified across 4+ analysis frameworks, making them the highest-confidence improvements.

| # | ID | Recommendation | Area | Effort | Reports | Status |
|---|-----|---------------|------|--------|---------|--------|
| 1 | T1-1 | **Goal decomposition / planning phase** — when LLM detects multi-step objective, output `plan` field with ordered steps before executing step 1. Show plan to user, execute step-by-step, track progress. | Agent Loop | Medium | 8 | PENDING |
| 2 | T1-4 | **Retrieval quality tracking / feedback loop** — log which of the 7 retrieved tools the LLM actually selected. Correlate retrieval accuracy with execution success. Tune descriptions for consistently-retrieved-but-never-used tools. | Retrieval | Medium | 6 | PENDING |
| 3 | T1-7 | **Cross-encoder re-ranking after retrieval** — after cosine similarity selects top-7 tools, run a cross-encoder on (query, description) pairs to re-rank. Most impactful for ambiguous queries where two tools have similar embeddings. | Retrieval | Medium | 5 | PENDING |
| 4 | T1-8 | **Session summary persistence / cross-session memory** — at session end, save a 3-sentence summary. At next session start, load as context. Optionally add user preference memory (frequently used databases, preferred output formats). | Memory | Medium | 5 | PENDING |
| 5 | T1-9 | **Capture user corrections as labeled training data** — when reflexion corrects a tool choice, save both failed and corrected commands as training pair. Connect existing feedback collection to training pipeline. | Learning | Medium | 5 | PENDING |
| 6 | T1-10 | **Expand doc knowledge base (14 to 50+ chunks)** — add chunks for security, multi-region, changefeeds, import/export, scheduled jobs, connection pooling, SQL tuning. Each ~130-200 words in existing CHUNK_ID|TITLE|CONTENT format. | RAG | Medium | 5 | PENDING |

## Tier 2 — Medium Effort, Medium-High Impact (11 items)

These were identified across 2-3 analysis frameworks.

| # | ID | Recommendation | Area | Effort | Reports | Status |
|---|-----|---------------|------|--------|---------|--------|
| 7 | T2-1 | **DPO (Direct Preference Optimization) training** — use positive feedback as preferred examples and negative/corrected examples as rejected. Data pipeline exists; needs pairing logic + DPO training support in `run_lora.sh`. | Fine-Tuning | Medium | 3 | PENDING |
| 8 | T2-2 | **LLM-as-a-Judge in batch pipeline** — after batch test execution, use a second LLM call to evaluate response quality: "Is this the right tool? Are the flags correct? Score 1-5." Automates quality assessment at scale. | Evaluation | Low | 3 | PENDING |
| 9 | T2-3 | **Daemon/Watch mode for persistent monitoring** — `roachie-watch --config watches.yaml` runs persistent health checks and sends alerts (Slack, email, webhook) when thresholds are breached. | Deployment | High | 3 | PENDING |
| 10 | T2-4 | **Audit logging for command executions** — log every command with timestamp, user, role, tool, database, authorization result. Creates audit trail for compliance and debugging. | Governance | Medium | 3 | PENDING |
| 11 | T2-5 | **Confidence scoring in tool selection** — LLM outputs `confidence` field (0-1) alongside each command. Low (<0.5): present alternatives. High (>0.9): execute without confirmation. | Decision Making | Medium | 3 | PENDING |
| 12 | T2-10 | **Reciprocal Rank Fusion (RRF)** — instead of "semantic first, regex supplements," compute fused score: `1/(k + rank_regex) + 1/(k + rank_semantic)`. Tools in both lists get significant boost. | Retrieval | Medium | 2 | PENDING |
| 13 | T2-11 | **Self-evaluation after command execution** — ask LLM: "Does this output answer the user's question? Score 1-5." If score < 3, trigger follow-up automatically. | Agent Loop | Medium | 3 | PENDING |
| 14 | T2-12 | **Tiered model selection by query complexity** — route simple queries ("show tables") to cheaper/faster models and complex queries to capable models. Use tool match count as complexity signal. | LLM / Cost | Medium | 3 | PENDING |
| 15 | T2-13 | **Tool chain documentation** — `tool_chains.txt` with common sequences (e.g., cr_query_stats -> cr_plan -> cr_indexes). Inject relevant chains into system prompt when first tool in chain is selected. | Prompts | Low | 2 | PENDING |
| 16 | T2-14 | **Result quality templates per tool category** — health checks get severity levels (Critical/Warning/OK), performance queries get ranked-by-impact lists, schema queries get structured summaries. | Output | Medium | 2 | PENDING |
| 17 | T2-15 | **Negative/multi-turn training examples** — add 15-20 informational examples with `"commands": []` and 5-10 multi-turn examples showing `needs_followup: true`. | Training | Medium | 2 | PENDING |

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
1   T1-1   Goal decomposition / planning phase             Agent Loop   Medium  8        PENDING
2   T1-4   Retrieval quality tracking / feedback loop       Retrieval    Medium  6        PENDING
3   T1-7   Cross-encoder re-ranking after retrieval         Retrieval    Medium  5        PENDING
4   T1-8   Session summary persistence / cross-session mem  Memory       Medium  5        PENDING
5   T1-9   Capture user corrections as training data        Learning     Medium  5        PENDING
6   T1-10  Expand doc knowledge base (14 → 50+ chunks)     RAG          Medium  5        PENDING
```

```
Table 2: Medium-Priority Recommendations — Tier 2 (2-3 reports)

#    ID     Recommendation                                  Area          Effort  Reports  Status
──   ─────  ──────────────────────────────────────────────  ────────────  ──────  ───────  ───────
7    T2-1   DPO (Direct Preference Optimization) training   Fine-Tuning   Medium  3        PENDING
8    T2-2   LLM-as-a-Judge in batch pipeline                Evaluation    Low     3        PENDING
9    T2-3   Daemon/Watch mode for persistent monitoring     Deployment    High    3        PENDING
10   T2-4   Audit logging for command executions            Governance    Medium  3        PENDING
11   T2-5   Confidence scoring in tool selection            Decisions     Medium  3        PENDING
12   T2-10  Reciprocal Rank Fusion (RRF)                    Retrieval     Medium  2        PENDING
13   T2-11  Self-evaluation after command execution         Agent Loop    Medium  3        PENDING
14   T2-12  Tiered model selection by query complexity      LLM / Cost    Medium  3        PENDING
15   T2-13  Tool chain documentation                        Prompts       Low     2        PENDING
16   T2-14  Result quality templates per tool category      Output        Medium  2        PENDING
17   T2-15  Negative/multi-turn training examples           Training      Medium  2        PENDING
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
Table 4: Suggested Implementation Order (impact-to-effort ratio)

Priority  ID     Recommendation                       Effort  Reports  Rationale
────────  ─────  ───────────────────────────────────  ──────  ───────  ──────────────────────────────
1         T2-2   LLM-as-a-Judge                       Low     3        Easiest remaining; high leverage
2         T2-13  Tool chain documentation              Low     2        Content only; no code changes
3         T1-4   Retrieval quality tracking             Medium  6        Correlation on existing CSV data
4         T1-1   Goal decomposition / planning         Medium  8        #1 most-recommended improvement
5         T1-9   Corrections as training data           Medium  5        Connects feedback to pipeline
6         T1-10  Expand doc chunks (14 → 50+)          Medium  5        Content authoring only
7         T1-8   Session summary persistence            Medium  5        File-based, straightforward
8         T2-15  Multi-turn training examples           Medium  2        Content authoring for training
9         T2-5   Confidence scoring                     Medium  3        JSON schema + prompt change
10        T2-11  Self-evaluation                        Medium  3        Extends agent loop
11        T2-4   Audit logging                          Medium  3        CSV append in exec path
12        T2-12  Tiered model selection                 Medium  3        Routing logic in dispatcher
13        T2-14  Result quality templates               Medium  2        Prompt templates per category
14        T2-10  Reciprocal Rank Fusion (RRF)           Medium  2        Scoring formula in enrichment
15        T2-1   DPO training                           Medium  3        Training pipeline extension
16        T1-7   Cross-encoder re-ranking               Medium  5        Requires additional model/API
17        T2-3   Daemon/Watch mode                      High    3        Largest scope; new binary
```

```
Table 5: Summary Statistics

Category                 Count    Percentage
───────────────────────  ─────    ──────────
Tier 1 (4+ reports)      6        35%
Tier 2 (2-3 reports)     11       65%
───────────────────────  ─────    ──────────
Total pending            17       100%
Already completed        4        (from low-effort batch)

Effort breakdown:
  Low effort             2        12%
  Medium effort          14       82%
  High effort            1        6%

Area breakdown:
  Retrieval              3        18%
  Agent Loop             2        12%
  Training / Learning    4        24%
  Prompts / Output       3        18%
  Memory / Governance    2        12%
  LLM / Evaluation       2        12%
  Deployment             1        6%
```
