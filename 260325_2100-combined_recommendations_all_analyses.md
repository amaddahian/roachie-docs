# Combined Recommendations: All Analysis Reports (2026-03-25)

**Date:** 2026-03-25
**Source:** 14 analysis reports produced during the afternoon session
**Total unique recommendations extracted:** 95+
**Purpose:** Unified, deduplicated, prioritized improvement roadmap across all frameworks

---

## Source Reports

| # | Report | Score | Framework |
|---|--------|-------|-----------|
| 1 | v19 Architecture Review | 8.8/10 | 12 actionable code-level recommendations |
| 2 | RAG Pipeline Analysis | 7.9/10 | 6-stage RAG evaluation |
| 3 | Model Architecture Analysis | 7.4/10 | 6 architecture paradigms |
| 4 | Fine-Tuning Techniques | 5.5/10 | 8 training techniques |
| 5 | AI Agent vs Agentic AI | 6.3/10 | 10 agentic dimensions |
| 6 | Real AI Stack | 6.5/10 | 13-layer production stack |
| 7 | Three Pillars (MCP/RAG/Skills) | 7.2/10 | 3 agent context pillars |
| 8 | Vectorless RAG / Hybrid Retrieval | 8.0/10 | 5 retrieval methods + hybrid routing |
| 9 | Agentic AI Litmus Test | 5.6/10 | 5 agentic capabilities |
| 10 | Chunking Strategies | 7.5/10 | 5 chunking methods |
| 11 | MCP + RAG + Agents Stack | 7.1/10 | 3-layer unified stack |
| 12 | Prompt Frameworks | 7.5/10 | 9 prompt frameworks (RACE, RISE, etc.) |
| 13 | Data Foundations | 4.6/10 | 12 production data pillars |
| 14 | LangChain RAG Pipeline | 7.2/10 | 9-module RAG pipeline |

---

## Methodology

Recommendations were extracted from every "Room to Improve" section and "Top N Improvements" table across all 14 reports. Duplicates were merged (many recommendations appear in 3-5 reports). Each recommendation is tagged with:
- **Area**: which system component it targets
- **Effort**: Low / Medium / High
- **Impact**: High / Medium / Low
- **Sources**: which reports recommend it (by number from table above)
- **Frequency**: how many reports independently recommend this improvement

Recommendations are grouped by system area and sorted by a combined score of (frequency x impact).

---

## Tier 1: Most-Recommended Improvements (4+ reports)

These improvements were independently identified across 4 or more analysis frameworks. They represent the highest-confidence opportunities.

### T1-1: Add Goal Decomposition / Planning Phase for Complex Queries
**Area:** Agent Loop / Orchestration
**Effort:** Medium
**Impact:** High
**Frequency:** 8 reports (#5, #6, #7, #8, #9, #11, #12, #13)

When the LLM detects a multi-step objective, output a `plan` field with ordered steps before executing step 1. Show the plan to the user for approval, then execute each step, feeding results forward. Track plan progress.

**Why it matters:** This is the single most-recommended improvement across all frameworks. Every analysis that evaluates orchestration, decision-making, or agentic capability identifies the lack of planning as the primary gap. The agent loop already supports multi-step execution — it just needs the LLM to articulate the plan first.

**Implementation sketch:**
- Add `plan` field to response JSON schema in `response_format.txt`
- When plan is present, display to user, then execute step-by-step
- Track completed/remaining steps across iterations
- Allow user to modify plan mid-execution

---

### T1-2: Enforce Reasoning Depth / Explain Tool Selection Rationale
**Area:** Prompts / Response Quality
**Effort:** Low
**Impact:** High
**Frequency:** 6 reports (#5, #7, #8, #9, #11, #12)

Require the LLM to explain WHY it chose a specific tool over alternatives in the `reasoning` field. Add to `response_format.txt`: "reasoning must explain why this tool was selected and why alternatives were rejected."

**Why it matters:** Multiple frameworks (RACE, 5W1H, GROW, CLEAR) identify that the system makes decisions internally without transparent reasoning. Adding 2 sentences to the response format template enforces this at near-zero cost.

---

### T1-3: Add Token Counting / System Prompt Size Guard
**Area:** Prompts / Context Management
**Effort:** Low
**Impact:** High
**Frequency:** 6 reports (#1, #6, #12, #13, #14, plus v19 R3)

After prompt assembly, estimate token count and warn if system prompt exceeds 60% of model context window. Critical for Ollama (16K context) where full enrichment can consume ~10K tokens.

**Why it matters:** v19 R3 identifies this as a concrete code-level issue. Multiple framework analyses flag it as a gap. The fix is straightforward: character count / chars-per-token ratio after enrichment.

---

### T1-4: Add Retrieval Quality Tracking / Feedback Loop
**Area:** Retrieval / Metrics
**Effort:** Medium
**Impact:** High
**Frequency:** 6 reports (#2, #6, #7, #8, #11, #14)

Log which of the 7 retrieved tools the LLM actually selected in its response. Correlate retrieval accuracy with execution success. If a tool is consistently retrieved but never used, it's a false positive — tune descriptions or similarity thresholds.

**Why it matters:** The metrics CSV already logs `enriched_tools` and `commands_executed`. This is a correlation analysis on existing data — no new infrastructure needed.

---

### T1-5: Add Query Normalization Before Retrieval
**Area:** Retrieval
**Effort:** Low
**Impact:** Medium
**Frequency:** 5 reports (#6, #8, #12, #13, #14)

Before embedding, normalize the query: lowercase, strip filler words, expand abbreviations ("db" → "database", "tbl" → "table", "perf" → "performance"). Add stemming for common suffixes (-ing, -ed, -s).

**Why it matters:** Multiple frameworks identify that the retriever processes raw user input without preprocessing. A simple normalization function would improve retrieval precision for informal queries.

---

### T1-6: Include System Prompt in LoRA Training Data
**Area:** Fine-Tuning Pipeline
**Effort:** Low (code change) / Medium (retrain)
**Impact:** High
**Frequency:** 4 reports (#1, #3, #4, #9)

Training examples contain only user/assistant pairs. At inference, the model receives a 3-4K token system prompt it never saw during training. This train/test distribution mismatch explains the 90% accuracy ceiling.

**Why it matters:** v19 R1 (Critical) — the single most impactful fix for local model accuracy. Generate a representative system prompt at training time.

---

### T1-7: Add Cross-Encoder Re-Ranking After Retrieval
**Area:** Retrieval
**Effort:** Medium
**Impact:** Medium
**Frequency:** 5 reports (#2, #7, #8, #13, #14)

After cosine similarity selects top-7 tools, run a cross-encoder on (query, description) pairs to re-rank. Most impactful for ambiguous queries where two tools have similar embeddings.

**Why it matters:** The current retriever achieves 99%+ accuracy, so this is an optimization — but multiple frameworks identify it as the natural next step for retrieval quality.

---

### T1-8: Add Session Summary Persistence / Cross-Session Memory
**Area:** Memory
**Effort:** Medium
**Impact:** Medium
**Frequency:** 5 reports (#5, #6, #9, #11, #14)

At session end, save a 3-sentence summary of what was discussed. At next session start, load the summary as context. Optionally add user preference memory (frequently used databases, preferred output formats).

**Why it matters:** Every framework that evaluates memory identifies cross-session conversation history as the main gap. The persistent learning DB (failure/success) provides pattern memory but not conversational memory.

---

### T1-9: Capture User Corrections as Labeled Training Data
**Area:** Learning / Fine-Tuning
**Effort:** Medium
**Impact:** High
**Frequency:** 5 reports (#4, #5, #9, #13, #14)

When reflexion corrects a tool choice, save both the failed and corrected commands as a training pair. When users provide feedback, flow it into LoRA training data. Connect the existing feedback collection to the training pipeline.

**Why it matters:** The data already exists (JSONL feedback, reflexion corrections) but never flows to model weights. This is the most frequently cited gap between "AI Agent" and "Agentic AI."

---

### T1-10: Expand Doc Knowledge Base (14 → 50+ Chunks)
**Area:** RAG / Knowledge
**Effort:** Medium
**Impact:** Medium
**Frequency:** 5 reports (#2, #7, #8, #10, #11)

Add chunks for uncovered topics: security, multi-region, changefeeds, import/export, scheduled jobs, connection pooling, SQL tuning. Each ~130-200 words following the existing CHUNK_ID|TITLE|CONTENT format.

**Why it matters:** 14 doc chunks cover the basics but miss important CockroachDB operational topics. The embedding and retrieval infrastructure is already built — it just needs more content.

---

## Tier 2: Frequently-Recommended Improvements (2-3 reports)

### T2-1: Add DPO (Direct Preference Optimization) Training
**Area:** Fine-Tuning
**Effort:** Medium
**Impact:** High
**Frequency:** 3 reports (#4, #5, #9)

Use positive feedback as preferred examples and negative/corrected examples as rejected examples. The data pipeline exists; needs pairing logic + DPO training support in `run_lora.sh`.

---

### T2-2: Add LLM-as-a-Judge to Batch Pipeline
**Area:** Evaluation
**Effort:** Low
**Impact:** High
**Frequency:** 3 reports (#2, #4, #9)

After batch test execution, use a second LLM call to evaluate response quality: "Is this the right tool? Are the flags correct? Score 1-5." Automates quality assessment at scale.

---

### T2-3: Add Daemon/Watch Mode for Persistent Monitoring
**Area:** Deployment
**Effort:** High
**Impact:** High
**Frequency:** 3 reports (#5, #6, #9)

`roachie-watch --config watches.yaml` that runs persistent health checks and sends alerts (Slack, email, webhook) when thresholds are breached. Transforms roachie from a copilot into an autonomous monitoring agent.

---

### T2-4: Add Audit Logging for Command Executions
**Area:** Governance / Observability
**Effort:** Medium
**Impact:** Medium
**Frequency:** 3 reports (#5, #6, #13)

Log every command execution with timestamp, user, role, tool, database, authorization result. Creates an audit trail for compliance and debugging.

---

### T2-5: Add Confidence Scoring to Tool Selection
**Area:** Decision Making
**Effort:** Medium
**Impact:** Medium
**Frequency:** 3 reports (#5, #8, #9)

Have the LLM output a `confidence` field (0-1) alongside each command. When confidence is low (<0.5), present alternatives to the user instead of executing. When high (>0.9), execute without confirmation.

---

### T2-6: Add Per-Stage Latency Tracking
**Area:** Observability / Chains
**Effort:** Low
**Impact:** Medium
**Frequency:** 3 reports (#6, #13, #14)

Log retrieval time, enrichment time, LLM call time, and execution time separately in the metrics CSV. Currently only total query latency is logged.

---

### T2-7: Add Task-Specific Role Modifiers to System Prompt
**Area:** Prompts
**Effort:** Low
**Impact:** Medium
**Frequency:** 3 reports (#6, #12, #14)

Map tool categories to role modifiers: "Acting as a performance tuning specialist" when monitoring tools are matched, "Acting as a migration architect" for DDL diff/migrate queries. Same LLM, different expertise framing.

---

### T2-8: Add Provenance Metadata to Embedding Files
**Area:** Data Versioning / Lineage
**Effort:** Low
**Impact:** Medium
**Frequency:** 3 reports (#3, #13, #14)

Include generation metadata in embedding files: `{source_hash: "abc123", model: "nomic-embed-text", generated: "2026-03-12", accuracy_at_generation: "99.3%"}`. Makes embeddings self-documenting and traceable.

---

### T2-9: Automated Quality Gate After Embedding Regeneration
**Area:** Data Quality / CI
**Effort:** Low
**Impact:** Medium
**Frequency:** 3 reports (#2, #13, #14)

After running `generate_embeddings.sh`, automatically run `test_embeddings.sh`. Fail if accuracy drops below threshold. Chain existing scripts — no new infrastructure.

---

### T2-10: Add Reciprocal Rank Fusion (RRF)
**Area:** Retrieval
**Effort:** Medium
**Impact:** Medium
**Frequency:** 2 reports (#8, #11)

Instead of "semantic first, regex supplements," compute a fused score: `1/(k + rank_regex) + 1/(k + rank_semantic)`. Tools appearing in both lists get a significant boost.

---

### T2-11: Add Self-Evaluation After Command Execution
**Area:** Agent Loop
**Effort:** Medium
**Impact:** Medium
**Frequency:** 3 reports (#5, #9, #12)

After command execution, ask the LLM: "Does this output answer the user's question? Score 1-5." If score < 3, trigger a follow-up automatically. Closes the quality evaluation gap.

---

### T2-12: Add Tiered Model Selection by Query Complexity
**Area:** LLM / Cost
**Effort:** Medium
**Impact:** Medium
**Frequency:** 3 reports (#3, #5, #6)

Route simple queries ("show tables") to cheaper/faster models (GPT-4.1-mini, Gemini Flash) and complex queries ("analyze performance across tenants") to more capable models. Use the query classifier or tool match count as complexity signal.

---

### T2-13: Add Tool Chain Documentation
**Area:** Prompts / Skills
**Effort:** Low
**Impact:** Medium
**Frequency:** 2 reports (#6, #7)

Create a `tool_chains.txt` file documenting common sequences: "For query optimization: cr_query_stats → cr_plan → cr_indexes." Inject relevant chains into the system prompt when the first tool in a chain is selected.

---

### T2-14: Add Result Quality Templates Per Tool Category
**Area:** Prompts / Output
**Effort:** Medium
**Impact:** Medium
**Frequency:** 2 reports (#8, #12)

Define what good output looks like per tool category: health checks get severity levels (Critical/Warning/OK), performance queries get ranked-by-impact lists, schema queries get structured summaries.

---

### T2-15: Add Negative/Multi-Turn Training Examples
**Area:** Fine-Tuning
**Effort:** Medium
**Impact:** High
**Frequency:** 2 reports (#1, #4)

Add 15-20 informational examples with `"commands": []` and 5-10 multi-turn examples showing `needs_followup: true`. Currently the model only sees single-turn command-generation examples.

---

## Tier 3: Single-Report Recommendations (grouped by area)

### Agent Loop / Orchestration
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-1 | Make iteration limit adaptive (LLM requests more iterations via response field, cap at 10) | #5 | Medium |
| T3-2 | Add workflow checkpointing (save agent loop state to temp file for resume on interrupt) | #5 | Medium |
| T3-3 | Add parallel command execution (background jobs for independent commands, join results) | #6 | Medium |
| T3-4 | Add pre-execution flag verification (parse flags against --help before running) | #2 | Medium |
| T3-5 | Add command deduplication in reflexion (prevent same command from being retried) | #2 | Low |

### Prompts / Context
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-6 | Surface options for ambiguous queries (present 2-3 interpretations instead of guessing) | #12 | Medium |
| T3-7 | Remove tool catalog from base prompt (redundant when enriched tools are injected, saves ~1,200 tokens) | #8 | Low |
| T3-8 | Add query-matched few-shot examples (select examples by tool match overlap) | #8 | Medium |
| T3-9 | Query-relevant schema injection (inject only schema for mentioned database, not all) | #10 | Medium |
| T3-10 | Add output formatting (detect tabular output, render as aligned tables) | #6 | Medium |

### Retrieval
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-11 | Add TF-IDF weighting to keyword scores (weight by inverse document frequency) | #8 | Low |
| T3-12 | Add tool relationship graph (JSON adjacency list for related tools) | #8 | Medium |
| T3-13 | Consolidate embedding files (merge tool + doc embeddings per provider, add type field) | #8 | Low |
| T3-14 | Raise similarity threshold from 0.35 to 0.50 | #8 | Low |

### Memory
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-15 | Add episodic retrieval (embed past queries, retrieve similar past interactions as few-shot) | #9 | Medium |
| T3-16 | Add user profiles (`~/.roachie/users/<username>.json` with preferences) | #9 | Medium |
| T3-17 | Add output persistence (store last N query results, `!!` to re-display) | #6 | Low |

### Fine-Tuning / Training
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-18 | Automate the fine-tuning pipeline (cron/hook: data accumulation → train → deploy → regression test) | #5 | High |
| T3-19 | Add A/B testing for models (run two Ollama models simultaneously, promote winner) | #5 | Medium |
| T3-20 | Extend training data sanitization (normalize --flag=value, ensure --insecure) | #1 R8 | Low |
| T3-21 | Add 8-bit QLoRA option alongside 4-bit | #4 | Low |

### LLM / Providers
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-22 | Implement Ollama streaming (same progress-dot pattern as cloud providers) | #1 R9 | Medium |
| T3-23 | Add response caching for deterministic queries ("what does cr_tables do?") | #6 | Medium |
| T3-24 | Enable Gemini thinking budget (default 2048 tokens) | #1 R11 | Low |
| T3-25 | Add metrics-to-routing feedback loop (auto-select optimal provider from historical data) | #5 | Medium |

### Governance / Security
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-26 | Validate preferences file syntax before sourcing (bash -n check) | #1 R5 | Low |
| T3-27 | Validate NL_BUDGET numeric input | #1 R4 | Low |
| T3-28 | Rename inverted check functions (_has_metachar instead of _check_metachar) | #1 R7 | Low |
| T3-29 | Add factual verification for key claims (verify node count, table existence against cache) | #6 | Medium |

### Data / Infrastructure
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-30 | Consolidate metrics CSVs into queryable storage (SQLite for trend analysis) | #13 | Medium |
| T3-31 | Add web loader for CockroachDB docs (automated knowledge base updates) | #14 | Medium |
| T3-32 | Add `/status` command (show connection, provider, model, session cost) | #6 | Low |
| T3-33 | Add `/refresh-schema` command (mid-session schema re-fetch) | #8 | Low |
| T3-34 | Add CI regression testing for semantic accuracy (regex-only mode) | #1 R12 | Low |
| T3-35 | Add `roachie-stats` command (metrics summary from CSV) | #5 | Medium |

### MCP / Integration
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-36 | Add MCP client capability (consume external MCP servers like Slack) | #11 | High |
| T3-37 | Bridge the 38-tool MCP gap (MCP wrappers for complex bash tools) | #11 | High |
| T3-38 | Add structured output mode (--json) to key tools for reliable chaining | #6 | Medium |

### Performance
| # | Recommendation | Source | Effort |
|---|---------------|--------|--------|
| T3-39 | Optimize Doc RAG query magnitude computation (compute once outside loop) | #1 R6 | Low |
| T3-40 | Add concurrency limit to parallel tool help fetch (cap at 8) | #1 R10 | Low |
| T3-41 | Use batch embedding APIs (one call per 77 descriptions vs 77 calls) | #2 | Low |

---

## Consolidated Priority Matrix

### Immediate — Low Effort, High Impact (do this week)

| # | Recommendation | Area | Sources |
|---|---------------|------|---------|
| T1-2 | Enforce reasoning depth in response_format.txt | Prompts | 6 reports |
| T1-3 | Add token counting / system prompt size guard | Prompts | 6 reports |
| T1-6 | Include system prompt in LoRA training data | Training | 4 reports |
| T3-27 | Validate NL_BUDGET numeric input | Governance | v19 R4 |
| T3-26 | Validate preferences file syntax | Governance | v19 R5 |
| T2-8 | Add provenance metadata to embedding files | Data | 3 reports |
| T2-9 | Automated quality gate after embedding regen | Quality | 3 reports |

### Short-Term — Low/Medium Effort, Medium-High Impact (next sprint)

| # | Recommendation | Area | Sources |
|---|---------------|------|---------|
| T1-1 | Add goal decomposition / planning phase | Agent Loop | 8 reports |
| T1-5 | Add query normalization before retrieval | Retrieval | 5 reports |
| T1-4 | Add retrieval quality tracking | Retrieval | 6 reports |
| T2-6 | Add per-stage latency tracking | Observability | 3 reports |
| T2-7 | Add task-specific role modifiers | Prompts | 3 reports |
| T2-2 | Add LLM-as-a-Judge to batch pipeline | Evaluation | 3 reports |
| T2-15 | Add negative/multi-turn training examples | Training | 2 reports |
| T3-20 | Extend training data sanitization | Training | v19 R8 |

### Medium-Term — Medium Effort, Medium Impact (next month)

| # | Recommendation | Area | Sources |
|---|---------------|------|---------|
| T1-9 | Capture user corrections as training data | Learning | 5 reports |
| T1-8 | Add session summary persistence | Memory | 5 reports |
| T1-10 | Expand doc knowledge base to 50+ chunks | RAG | 5 reports |
| T1-7 | Add cross-encoder re-ranking | Retrieval | 5 reports |
| T2-1 | Add DPO training | Training | 3 reports |
| T2-4 | Add audit logging | Governance | 3 reports |
| T2-5 | Add confidence scoring | Decision Making | 3 reports |
| T2-11 | Add self-evaluation | Agent Loop | 3 reports |
| T2-12 | Add tiered model selection | LLM | 3 reports |
| T3-22 | Implement Ollama streaming | LLM | v19 R9 |

### Long-Term — High Effort, Strategic Impact (future)

| # | Recommendation | Area | Sources |
|---|---------------|------|---------|
| T2-3 | Add daemon/watch mode | Deployment | 3 reports |
| T3-36 | Add MCP client capability | Integration | #11 |
| T3-37 | Bridge MCP 38-tool gap | Integration | #11 |
| T3-18 | Automate fine-tuning pipeline | Training | #5 |

---

## What NOT to Do (Consistently Identified Across Reports)

| Anti-Pattern | Reports | Why |
|-------------|---------|-----|
| Add a vector database (FAISS, Chroma, Pinecone) | #2, #8, #13, #14 | 77 tools + 14 doc chunks fit in memory; flat files are correct at this scale |
| Add LangChain/Python framework | #14 | Bash implementation has zero dependencies, <100ms startup, 99%+ accuracy |
| Add Kafka/streaming for events | #13 | CLI is request-response by design; streaming is architectural mismatch |
| Apply marketing/persuasion frameworks (PASTOR, FAB) | #12 | DBA tooling needs technical precision, not persuasion |
| Add full RLHF/PPO infrastructure | #4 | Too complex for current data volume; DPO achieves 80% of the benefit |
| Build a data lake or metadata catalog | #13 | JSON headers with provenance metadata solve the problem without infrastructure |
| Add multi-agent orchestration framework | #5, #6, #11 | Single-agent with prompt profiles achieves specialization without framework overhead |

---

## Statistics

```
Table 1: Recommendations by System Area

Area                    Tier 1    Tier 2    Tier 3    Total
──────────────────────  ──────    ──────    ──────    ─────
Prompts / Context       2         2         5         9
Retrieval               3         1         4         8
Fine-Tuning / Training  2         2         4         8
Agent Loop              1         1         5         7
Memory                  1         0         3         4
LLM / Providers         0         1         4         5
Data / Infrastructure   0         2         6         8
Governance / Security   0         1         4         5
MCP / Integration       0         0         3         3
Observability           0         1         0         1
Evaluation              0         1         0         1
Deployment              0         1         0         1
Performance             0         0         3         3
──────────────────────  ──────    ──────    ──────    ─────
Total                   10        15        41        66
```

```
Table 2: Recommendations by Effort Level

Effort    Count    Percentage
──────    ─────    ──────────
Low       22       33%
Medium    36       55%
High      5        8%
Done      3        5%
──────    ─────    ──────────
Total     66       100%
```

```
Table 3: Cross-Report Frequency Distribution

Frequency (reports)    Count    Tier
─────────────────────  ─────    ────
8 reports              1        Tier 1 (planning)
6 reports              4        Tier 1
5 reports              5        Tier 1
4 reports              1        Tier 1
3 reports              15       Tier 2
2 reports              5        Tier 2
1 report               35       Tier 3
```

---

## Key Insight

> Across 14 independent analysis frameworks, the same 10 improvements surface repeatedly as the highest-priority gaps. The most-recommended improvement (goal decomposition/planning, 8 reports) and the most impactful code fix (system prompt in training data, v19 R1 Critical) are both well-understood problems with clear implementation paths. The 7 "Immediate" items are all Low effort — they could be completed in a single day. The consistent "What NOT to Do" list across 4+ reports confirms that the architecture is right-sized: flat files beat vector databases, bash beats Python frameworks, and single-agent with prompt profiles beats multi-agent orchestration at this scale.

---

## CLI Summary

```
Table 4: Top 10 Recommendations by Cross-Report Frequency

#   Recommendation                                    Reports  Effort  Area
──  ────────────────────────────────────────────────  ───────  ──────  ──────────────
1   Goal decomposition / planning phase               8        Medium  Agent Loop
2   Enforce reasoning depth in response format         6        Low     Prompts
3   Token counting / system prompt size guard          6        Low     Prompts
4   Retrieval quality tracking / feedback loop         6        Medium  Retrieval
5   Query normalization before retrieval               5        Low     Retrieval
6   System prompt in LoRA training data                4        Low     Training
7   Cross-encoder re-ranking after retrieval           5        Medium  Retrieval
8   Session summary persistence                        5        Medium  Memory
9   Capture corrections as training data               5        Medium  Learning
10  Expand doc knowledge base (14 → 50+ chunks)       5        Medium  RAG
```

**The Pattern:** 33% of recommendations are Low effort. The top 3 quick wins (#2, #3, #5) could each be implemented in under an hour and together address gaps identified by 17 report-mentions.

**Total:** 66 unique recommendations from 14 analysis frameworks, deduplicated and prioritized. 10 in Tier 1 (4+ reports), 15 in Tier 2 (2-3 reports), 41 in Tier 3 (single report).
