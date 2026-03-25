# Prompt Frameworks Analysis: Roachie vs 9 Structured Prompt Paradigms

**Date:** 2026-03-25
**Subject:** Evaluating roachie's prompt engineering against RACE, RISE, STAR, SOAP, CLEAR, PASTOR, FAB, 5W1H, and GROW frameworks
**Core thesis:** "AI output quality = prompt clarity + structured thinking"

---

## Framework Evaluation

### 1. RACE — Role, Action, Context, Expectation

**What it prescribes:** Define who the AI is, what it should do, the surrounding context, and the expected output format.

**Pros — What roachie does well:**
- **Role** is explicit and domain-specific: `"You are a CockroachDB database administrator assistant"` with awareness of PCR (Physical Cluster Replication), multi-tenancy, and roachman topology — not a generic "helpful assistant"
- **Action** is constrained to tool execution: `"ALWAYS use actual cr_* tools with FULL PATH, NOT manual SQL"` — the LLM knows exactly what actions are permitted
- **Context** is the richest dimension — live cluster topology (nodes, ports, LB status), database schema (tables per tenant), tools version, connection parameters, persistent failure/success history, and RBAC role — all injected dynamically per session
- **Expectation** is a strict JSON contract: `{reasoning, commands[], user_message, needs_followup}` defined in `response_format.txt` with explicit field semantics

**Cons — What's missing:**
- Role doesn't vary by task type — the same "DBA assistant" role handles schema exploration, performance tuning, migration planning, and health checks, where different expertise framings could improve response quality
- Expectation doesn't define quality criteria — JSON structure is specified but there's no guidance on what makes a "good" reasoning explanation vs a superficial one

**Room to improve:**
- Add task-specific role modifiers: "Acting as a performance tuning specialist" when query matches monitoring/performance tools, "Acting as a migration architect" for DDL diff/migrate queries
- Add quality expectations to `response_format.txt`: "reasoning should explain WHY this tool was chosen over alternatives"

**RACE Score: 9/10** — All four elements are present and well-implemented. The structured JSON expectation is particularly strong.

---

### 2. RISE — Role, Identify (problem), Steps, Expectation

**What it prescribes:** Define the role, identify the specific problem, outline solution steps, and set expectations for the result.

**Pros — What roachie does well:**
- **Role** is established (same as RACE)
- **Identify** happens implicitly through regex + semantic matching — the system identifies which of 77 tools are relevant to the user's problem before the LLM even sees the query
- **Steps** are encoded in `tool_specific_rules.txt` as selection guides: "VIEW TOOL SELECTION GUIDE: Find by name → cr_find_views, Get DDL → cr_ddl_view, Search definitions → cr_grep_views, Validate → cr_check_views"
- **Expectation** for multi-step problems uses `needs_followup: true` to chain steps naturally

**Cons — What's missing:**
- Problem identification is done FOR the LLM (via regex/semantic matching), not BY the LLM — the model doesn't first analyze "what kind of problem is this?" before selecting tools
- Steps are static recipes in template files, not dynamically generated problem-solving plans
- No explicit "decompose this complex query into sub-problems" instruction in the prompt

**Room to improve:**
- Add a planning phase for complex queries: before jumping to tool execution, have the LLM output a brief problem decomposition in the `reasoning` field
- The agent loop's 3-iteration limit could support a plan-execute-refine pattern more explicitly

**RISE Score: 6.5/10** — Role and Expectation are strong, but problem identification is pre-computed rather than reasoned, and step planning is template-driven rather than dynamic.

---

### 3. STAR — Situation, Task, Action, Result

**What it prescribes:** Describe the current situation, define the task, specify what action to take, and describe the expected result.

**Pros — What roachie does well:**
- **Situation** is comprehensively defined: cluster topology (which clusters exist, their status, node counts), schema context (which databases/tables exist), and historical context (what failed before) — the LLM always knows the current state of the environment
- **Task** comes directly from the user's natural language query
- **Action** is constrained to the cr_* tool vocabulary with exact syntax patterns from `execution_pattern.txt`
- **Result** handling through the agent loop: command output is captured, truncated to 4000 chars, and fed back for interpretation

**Cons — What's missing:**
- **Result** expectations aren't pre-defined — the prompt says "respond in JSON" but doesn't describe what a successful result looks like for different query types (e.g., "for health checks, summarize with severity levels")
- Situation doesn't include temporal context beyond current date — no "this cluster was upgraded 2 days ago" or "last backup was 6 hours ago" awareness

**Room to improve:**
- Add result templates per tool category: "For monitoring queries, classify findings as Critical/Warning/OK"
- Include recent operational context (last backup time, last upgrade, recent alerts) in the situation section

**STAR Score: 7/10** — Situation is the standout strength. Task and Action are solid. Result interpretation could be more structured.

---

### 4. SOAP — Subject, Objective, Action, Plan

**What it prescribes:** Define the subject matter, clarify objectives, specify actions, and outline a plan.

**Pros — What roachie does well:**
- **Subject** is narrowly scoped: CockroachDB administration only, with explicit boundaries ("NEVER write SQL queries", "NEVER use shell metacharacters")
- **Objective** is implicitly "execute the right tool with the right flags" — every query resolves to a tool invocation
- **Action** is the best-defined element: 8 template files totaling ~500 lines of execution rules, flag guidance, hallucination warnings, and examples
- **Plan** emerges through multi-step execution: `needs_followup: true` enables sequential plan execution across up to 3 iterations

**Cons — What's missing:**
- No explicit planning step — the LLM jumps from query to command without articulating a plan first
- Objectives are always implicit ("help the user") rather than explicitly stated per-query
- No rollback planning — if a tool execution fails, reflexion handles retry but there's no "what should be undone" awareness

**Room to improve:**
- For complex queries (migration, upgrade, replication setup), require the LLM to output a numbered plan in the `reasoning` field before executing step 1
- Add explicit objective classification: "Is this a read-only information request or a state-changing operation?"

**SOAP Score: 6/10** — Subject and Action are strong. Objective is implicit. Planning is the weakest element — the system is reactive (execute → check → retry) rather than proactive (plan → execute → verify).

---

### 5. CLEAR — Context, Learn, Evaluate, Action, Review

**What it prescribes:** Provide context, learn from available information, evaluate options, take action, and review outcomes.

**Pros — What roachie does well:**
- **Context** is the system's strongest dimension — topology, schema, tools catalog, RBAC role, connection parameters, learning history
- **Learn** is partially implemented through persistent learning: JSONL failure/success databases inject "MISTAKES TO AVOID" and "SUCCESSFUL PATTERNS" into prompts — the system genuinely learns from past sessions
- **Evaluate** happens through tool matching: regex + semantic matching evaluates which of 77 tools best fit the query, deduplicating and ranking results
- **Action** is tool execution with full flag validation
- **Review** exists through the agent loop: command output is fed back, and reflexion triggers error analysis on failure

**Cons — What's missing:**
- **Learn** is limited to query→command mappings — the system doesn't learn user preferences, common workflows, or domain patterns beyond "this command worked/failed"
- **Evaluate** doesn't compare alternative approaches — the LLM is given matched tools but isn't asked "which of these 3 tools is best for this specific scenario and why?"
- **Review** is only triggered by `needs_followup: true` or failure — successful single-step commands get no review phase

**Room to improve:**
- Add explicit evaluation instruction: "If multiple tools match, explain in reasoning why you chose this one over alternatives"
- Add post-execution review for critical operations: after state-changing commands, verify the result with a read-only check
- Expand learning to capture user correction patterns, not just command success/failure

**CLEAR Score: 7/10** — The iterative improvement loop (Context → Learn → Action → Review) exists but isn't fully closed. The "Evaluate" step is the main gap — the system pre-filters tools but doesn't ask the LLM to reason about alternatives.

---

### 6. PASTOR — Problem, Amplify, Story, Transformation, Offer, Response

**What it prescribes:** A persuasion/marketing framework — identify a problem, amplify its impact, tell a story, show transformation, make an offer, and get a response.

**Pros — What roachie does well:**
- **Problem** identification works through natural language queries — users describe their DBA problems in plain English
- **Offer** is the tool execution — the system offers a concrete solution (the right cr_* command) rather than abstract advice

**Cons — What's missing:**
- This framework is designed for marketing and persuasion, which is fundamentally misaligned with roachie's purpose as a technical CLI tool
- No Amplify, Story, or Transformation elements exist or should exist — these are inappropriate for DBA tooling
- The system correctly avoids persuasive language in favor of precise technical output

**Room to improve:**
- Not applicable — PASTOR is the wrong framework for this domain. The system's direct, technical approach is correct for DBA operations.

**PASTOR Score: N/A** — Framework is designed for persuasion/copywriting. Roachie correctly ignores these patterns in favor of technical precision. Applying PASTOR would degrade output quality.

---

### 7. FAB — Features, Advantages, Benefits

**What it prescribes:** A product positioning framework — describe features, explain advantages over alternatives, and articulate user benefits.

**Pros — What roachie does well:**
- **Features** are well-documented: `tool_descriptions.txt` provides 77 one-line descriptions, `tool_notes.txt` details per-tool flags, and `--help` output is dynamically injected for matched tools
- The `user_message` field in responses sometimes explains benefits: "This will show you all tables with row counts so you can identify the largest tables"

**Cons — What's missing:**
- Like PASTOR, this is a product/sales framework, not a prompt engineering pattern for technical assistants
- **Advantages** over alternatives aren't articulated — the system doesn't explain "cr_query_stats is better than raw SQL because it normalizes fingerprints"
- **Benefits** are action-oriented ("here's the command") rather than outcome-oriented ("this will help you find the root cause of your latency issue")

**Room to improve:**
- While FAB as a complete framework doesn't apply, the "Benefits" element could enhance `user_message` responses — instead of just "Running cr_health" say "This checks node liveness, disk usage, and replication status so you can quickly identify unhealthy nodes"
- Tool descriptions in the catalog could include a "why use this" element alongside "what it does"

**FAB Score: 4/10** — Features are well-cataloged. Advantages and Benefits aren't part of the prompt design, though adding benefit context to user_message would improve the user experience.

---

### 8. 5W1H — Who, What, When, Where, Why, How

**What it prescribes:** Answer the fundamental questions — who is involved, what needs to be done, when and where it applies, why it matters, and how to do it.

**Pros — What roachie does well:**
- **Who** — RBAC role system (`admin`, `dba`, `analyst`, `monitor`) defines who is asking and restricts available commands accordingly
- **What** — Tool catalog with 77 tools across 11 categories, plus per-tool flag documentation
- **When** — Current date/time injected into system prompt; some tools are time-scoped (cr_query_stats `--since`, cr_changefeed_status)
- **Where** — Cluster topology defines where: Cluster A vs B, which tenant, which database, which port
- **How** — Execution patterns, flag syntax, few-shot examples — the "how" is the most thoroughly documented element
- **Why** — The `reasoning` field in JSON responses captures why a specific tool was chosen

**Cons — What's missing:**
- **Why** is requested but not enforced — the LLM can write minimal reasoning ("Using cr_tables to list tables") without explaining why this tool over alternatives
- **When** lacks operational context — no awareness of maintenance windows, backup schedules, or peak usage periods
- **Who** affects tool availability but not prompt tone or detail level — a `monitor` role user might need more explanation than an `admin`

**Room to improve:**
- Enforce meaningful `reasoning`: "Explain your tool selection rationale, including why alternatives were rejected"
- Add role-appropriate response depth: analysts get more explanation, admins get terse output
- Include operational timing context where available (last backup, cluster age, uptime)

**5W1H Score: 8/10** — Five of six elements are present and functional. "Why" (reasoning depth) and "When" (operational timing) are the gaps. The system naturally aligns with this framework because DBA queries are inherently "who needs what done where."

---

### 9. GROW — Goal, Reality, Options, Will

**What it prescribes:** A coaching/decision-making framework — define the goal, assess current reality, explore options, and commit to action.

**Pros — What roachie does well:**
- **Goal** is the user's natural language query — "show me slow queries" or "check replication lag"
- **Reality** is comprehensively established through topology, schema, and cluster status — the system knows what's actually running
- **Options** are pre-filtered: regex + semantic matching surfaces relevant tools, and `tool_specific_rules.txt` provides selection guides ("VIEW TOOL SELECTION GUIDE: 4 options with different purposes")
- **Will** (action commitment) is the command execution — the system commits to running the selected tool

**Cons — What's missing:**
- **Options** aren't presented to the user — the system makes the choice internally. For complex scenarios (migration, upgrade), showing "here are 3 approaches" would be valuable
- **Goal** clarification doesn't happen — if the user's query is ambiguous ("check the database"), the system guesses rather than asking for clarification
- No explicit trade-off analysis: "Option A is faster but Option B gives more detail"

**Room to improve:**
- For ambiguous queries, add a clarification step: "I found 3 possible interpretations — did you mean X, Y, or Z?"
- For complex operations (migration, upgrade, backup), present options with trade-offs before executing
- The `reasoning` field could explicitly follow GROW: "Goal: find slow queries. Reality: cluster has 3 nodes, movr database. Options: cr_query_stats (normalized), cr_slow_queries (active). Choice: cr_query_stats --top 10 because it covers historical patterns."

**GROW Score: 5.5/10** — Goal and Reality are strong. Options exist but aren't surfaced to the user. The coaching/decision-making pattern would significantly improve the system for complex, multi-option scenarios.

---

## Summary Scorecard

| # | Framework | Score | Roachie Alignment | Key Strength | Key Gap |
|---|-----------|-------|-------------------|--------------|---------|
| 1 | RACE | 9/10 | Naturally aligned | Strict JSON expectation + rich context | Role doesn't adapt per task type |
| 2 | RISE | 6.5/10 | Partially aligned | Tool selection guides as step recipes | Problem ID is pre-computed, not reasoned |
| 3 | STAR | 7/10 | Well aligned | Situation awareness (topology + schema) | Result expectations not pre-defined |
| 4 | SOAP | 6/10 | Partially aligned | Action rules are comprehensive | No explicit planning phase |
| 5 | CLEAR | 7/10 | Iterative loop exists | Persistent learning from failures | Evaluate step (compare alternatives) missing |
| 6 | PASTOR | N/A | Not applicable | Correctly avoids persuasion patterns | N/A — wrong domain |
| 7 | FAB | 4/10 | Minimally aligned | Tool features well-documented | Benefits/advantages not articulated |
| 8 | 5W1H | 8/10 | Naturally aligned | Who/What/Where/How all strong | Why (reasoning depth) underenforced |
| 9 | GROW | 5.5/10 | Partially aligned | Reality assessment is excellent | Options not surfaced to user |

**Overall Prompt Engineering Score: 7.5/10** — "Structured Thinking Without the Labels"

---

## Key Insight

> Roachie accidentally implements the best elements of RACE, 5W1H, and CLEAR without formally following any framework. Its prompt architecture — role definition, rich context injection, strict output format, iterative review loop, and persistent learning — covers ~80% of what these frameworks prescribe. The 20% gap is consistently the same pattern: **the system makes decisions internally rather than reasoning transparently about alternatives.** The LLM is told what tools exist and what format to respond in, but isn't asked to explain WHY it chose one approach over another, EVALUATE trade-offs, or PRESENT options to the user. Adding explicit reasoning depth ("explain your choice") and option surfacing ("for complex queries, list alternatives") would close the gap without restructuring the prompt architecture.

---

## Top 5 Improvements by Impact

| # | Improvement | Frameworks Addressed | Effort |
|---|-------------|---------------------|--------|
| 1 | **Enforce reasoning depth** — require "why this tool, not alternatives" in the reasoning field | RISE, CLEAR, 5W1H, GROW | Low — add 2 sentences to `response_format.txt` |
| 2 | **Add planning phase for complex queries** — multi-step operations should output a plan before executing step 1 | SOAP, RISE, GROW | Medium — modify agent loop to detect complex queries |
| 3 | **Surface options for ambiguous queries** — present 2-3 interpretations instead of guessing | GROW, CLEAR | Medium — add ambiguity detection heuristic |
| 4 | **Task-specific role modifiers** — "Acting as performance specialist" vs "Acting as migration architect" based on matched tool category | RACE, 5W1H | Low — map tool categories to role modifiers |
| 5 | **Result quality templates** — define what good output looks like per tool category (health: severity levels, performance: ranked by impact) | STAR, RACE | Medium — add per-category result expectations to prompts |

---

## What NOT to Do

| Anti-Pattern | Why |
|-------------|-----|
| Apply PASTOR/FAB to DBA prompts | Persuasion frameworks degrade technical precision |
| Add all 9 frameworks simultaneously | Prompt bloat would exceed token budgets, especially for Ollama (16K) |
| Make the LLM "coach" the user (full GROW) | Users want answers, not coaching questions — keep it directive |
| Require verbose reasoning for simple queries | "Show tables" shouldn't need a paragraph of justification |

---

## CLI Summary

**Prompt Framework Alignment Map**

```
Table 1: Framework Coverage in Current Prompt Architecture

Framework  Element        Implementation                              Status
─────────  ─────────────  ──────────────────────────────────────────  ──────────
RACE       Role           "CockroachDB DBA assistant"                 PRESENT
RACE       Action         "Use cr_* tools with FULL PATH"             PRESENT
RACE       Context        Topology + schema + history + RBAC          STRONG
RACE       Expectation    JSON {reasoning, commands[], ...}           STRONG
RISE       Identify       Regex + semantic pre-matching               PRE-COMPUTED
RISE       Steps          tool_specific_rules.txt selection guides    STATIC
STAR       Situation      Cluster topology + schema + date            STRONG
STAR       Result         Agent loop captures output                  NO TEMPLATES
SOAP       Plan           needs_followup chains steps                 IMPLICIT
CLEAR      Learn          JSONL failure/success injection             PRESENT
CLEAR      Evaluate       Tool matching ranks, no LLM comparison     GAP
CLEAR      Review         Reflexion on failure, agent loop            PRESENT
5W1H       Who            RBAC roles (admin/dba/analyst/monitor)      PRESENT
5W1H       Where          Cluster A/B, tenant, database, port         STRONG
5W1H       Why            reasoning field exists but unenforced       SHALLOW
GROW       Options        Tools pre-filtered, not shown to user       HIDDEN
GROW       Reality        Topology + schema + cluster status          STRONG
```

```
Table 2: Scorecard Summary

Framework   Score    Verdict
─────────   ─────    ─────────────────────────────────
RACE        9/10     Naturally aligned — best fit
5W1H        8/10     DBA queries naturally map to 5W1H
STAR        7/10     Strong situation, weak result templates
CLEAR       7/10     Iterative loop works, evaluate missing
RISE        6.5/10   Steps exist but aren't dynamic
SOAP        6/10     No explicit planning phase
GROW        5.5/10   Options hidden from user
FAB         4/10     Features documented, benefits absent
PASTOR      N/A      Wrong domain entirely
─────────   ─────    ─────────────────────────────────
OVERALL     7.5/10   "Structured Thinking Without Labels"
```

**Key Insight:** Roachie implements ~80% of the best prompt framework elements (role, context, format, iteration) without formally following any framework. The consistent 20% gap: the system decides internally rather than reasoning transparently about alternatives. Fix: enforce reasoning depth + surface options for complex queries.

**Top 3 Quick Wins:**
1. Add "explain why this tool, not alternatives" to response_format.txt (RACE/5W1H/GROW)
2. Map tool categories to role modifiers — "as performance specialist" (RACE)
3. Add result quality templates per tool category — severity levels, ranked lists (STAR)

**What the frameworks reveal about roachie's prompt design:**
- RACE is the natural fit — roachie already does Role+Action+Context+Expectation
- 5W1H maps perfectly to DBA queries — who/what/where/how are inherent to database ops
- CLEAR's iterative loop (learn→evaluate→act→review) matches the agent loop architecture
- GROW and SOAP expose the planning gap — complex queries jump to execution without articulating a plan
- PASTOR and FAB confirm the system correctly avoids persuasion in favor of precision
