# Roachie: MCP + RAG + Agents as a Unified Stack Analysis

**Date:** 2026-03-25
**Scope:** Evaluate roachie against the "they're layers of the same system" framework -- how well do MCP (connection), RAG (memory), and Agents (doers) work together as an integrated stack?

---

## Framework Recap

MCP, RAG, and Agents are not competitors -- they're complementary layers:

| Layer | Role | Analogy |
|-------|------|---------|
| **MCP** | The connection -- standardized plug for external tools/data | USB-C port |
| **RAG** | The memory -- fetch what the LLM doesn't know | Memory/knowledge |
| **Agent** | The doer -- goal-oriented observe-reason-act-refine loop | Independent worker |

**The power is in the stack:** An agent breaks down the task, RAG pulls the relevant knowledge, MCP connects to the tools that execute the action. No custom code per step.

---

## Roachie's Stack: How the Three Layers Work Together

### Layer Map

| Layer | Roachie Implementation | Maturity |
|-------|----------------------|----------|
| **MCP (Connection)** | 39 tools via GenAI Toolbox + native tool calling + 77 cr_* bash tools via custom pipeline | Production |
| **RAG (Memory)** | Hybrid regex+semantic tool matching + Doc RAG (14 chunks) + live schema SQL retrieval | Production |
| **Agent (Doer)** | Single agent with observe-reason-act-refine loop (reflexion, 3 iterations max) | Production |

### How They Run Together in Roachie

The framework's onboarding example: "agent breaks down task → RAG pulls knowledge → MCP executes actions." Here's roachie's equivalent flow for a real query:

**User asks:** "Show me the slow queries in the movr database on tenant va"

```
Step 1: AGENT observes
  → Receives user query
  → Classifies intent (performance analysis)

Step 2: RAG provides memory
  → Regex matching: "slow" + "query" → cr_query_stats, cr_query_history, cr_plan    (~1ms)
  → Semantic matching: cosine similarity confirms cr_query_stats as top match         (~10ms)
  → Doc RAG: retrieves "Slow Query Diagnosis" chunk with diagnosis workflow            (~10ms)
  → Schema context: knows movr database exists on tenant va with rides, vehicles tables (cached)
  → Tool --help: fetches cr_query_stats flags documentation                            (~500ms)
  → All injected into system prompt as LLM context

Step 3: AGENT reasons
  → LLM receives: tool docs + schema + doc chunk + conversation history
  → Generates JSON: {"reasoning": "...", "commands": [{"command": "cr_query_stats -d movr -h localhost -p 26257 -t va --insecure --top 5"}]}

Step 4: MCP/Tools execute
  → 6-layer validation: whitelist ✓, normalize ✓, metachar ✓, SQL guard ✓, exists ✓, RBAC ✓
  → cr_query_stats executes against live CockroachDB cluster
  → Returns actual slow query data

Step 5: AGENT refines (if needed)
  → If needs_followup=true: output fed back → LLM reasons about results → suggests cr_plan
  → If command failed: reflexion triggers → analyzes error → tries different approach
  → If success: logs metrics, updates persistent learning
```

**This is exactly the framework's model:** Agent orchestrates, RAG provides context, MCP/tools execute. No custom code per query -- the same pipeline handles all 77 tools across all query types.

---

## Layer 1: MCP (The Connection)

**Framework says:** "The USB-C port for AI. Before MCP, connecting an LLM to a database required custom glue code. MCP creates a standardized plug."

### How Roachie Connects

Roachie has **two connection systems** -- one custom, one MCP-standard:

| Connection System | Tools | Protocol | Used By |
|-------------------|-------|----------|---------|
| Custom bash pipeline | 77 cr_* tools | LLM → JSON → validate → bash execution | Internal (roachie-nl, roachman) |
| MCP via GenAI Toolbox | 39 SQL-based tools | MCP standard protocol | External (Claude Desktop, any MCP client) |

Additionally, native tool calling bridges MCP schemas to provider-specific formats:
- `_nl_mcp_to_anthropic()` → Anthropic tool_use format
- `_nl_mcp_to_openai()` → OpenAI function calling format
- `_nl_mcp_to_gemini()` → Gemini functionDeclarations format

### Pros

- **Two connection interfaces cover all use cases** -- The custom bash pipeline handles complex orchestration tools (backup, migrate, DDL generation) that need bash logic. The MCP interface exposes SQL-expressible tools to any MCP client. Together, they provide 100% tool coverage: 77 internally + 39 externally.

- **MCP removes integration friction for external consumers** -- Any MCP-compatible client (Claude Desktop, custom LangChain agents, Cursor, etc.) can use roachie's 39 database tools without knowing anything about bash, cr_* tools, or CockroachDB connection parameters. This is exactly the framework's "standardized plug" vision.

- **The custom pipeline is actually more capable than MCP** -- The 38 tools excluded from MCP (cr_backup, cr_migrate, cr_ddl_table, cr_create_cluster, etc.) require multi-step bash orchestration that pure SQL can't express. The custom connection handles these while MCP handles the SQL-expressible subset. This is a pragmatic split.

- **Native tool calling bridges MCP to all providers** -- When `NL_NATIVE_TOOLS=1`, roachie converts MCP tool schemas to provider-native formats. The LLM invokes tools through structured function calling rather than generating command strings. This eliminates the "glue code" problem for 3 cloud providers simultaneously.

### Cons

- **Two connection systems = two integration burdens** -- The custom bash pipeline requires its own validation, execution, and error handling code. The MCP interface requires its own YAML configuration and SQL translation. A unified approach would reduce maintenance.

- **MCP is external-facing only** -- The internal pipeline doesn't use MCP. When roachie-nl processes a query, it goes through the custom bash pipeline, not through MCP. This means roachie has the standard plug but doesn't use it internally -- it uses a custom plug instead.

- **No MCP client capability** -- Roachie can expose tools via MCP (server) but can't consume external MCP servers (client). It can't connect to Slack, Jira, Grafana, or other MCP-enabled services. The connection is one-directional.

- **38-tool gap between internal and MCP** -- External MCP clients only access 39 of 77 tools. Complex operations (backup, migration, DDL generation) are unavailable via MCP. An external agent using roachie via MCP gets a reduced capability set.

### Room to Improve

1. **Add MCP client capability** -- Enable roachie to consume external MCP servers. "Show me the slow queries and post a summary to Slack" would use roachie's tools (MCP server) for the query and Slack's MCP server for the posting. One query, two MCP connections, zero custom integration code.

2. **Bridge the 38-tool gap** -- Create MCP wrappers for complex bash tools. The wrapper invokes the bash tool via subprocess and returns structured output. External MCP clients would then access all 77 tools, not just 39.

---

## Layer 2: RAG (The Memory)

**Framework says:** "LLMs don't know what happened in your 9am meeting. RAG fixes this by fetching the specific document, email, or data row the model actually needs before it answers."

**Framework also warns:** "RAG is the most overhyped of the three. Most teams bolt it on and say it's finished."

### How Roachie Remembers

Roachie has **five memory sources**, not just one RAG pipeline:

| Memory Source | What It Knows | How It Retrieves | Freshness |
|--------------|--------------|-----------------|-----------|
| Tool RAG | Which of 77 tools match the query | Hybrid regex + cosine similarity | Static (updated on tool changes) |
| Doc RAG | CockroachDB operational knowledge | Cosine similarity, top-3 chunks | Static (14 hand-curated chunks) |
| Live schema | What databases, tables, columns exist | SQL queries to live cluster | Real-time (per session) |
| Persistent learning | What failed/succeeded in past sessions | JSONL lookup, injected into prompt | Cross-session (30-day TTL) |
| Conversation history | What was said in this session | JSON message array, last 10 turns | Session-scoped |

### Pros

- **RAG is not "bolted on" -- it's architecturally integrated** -- The framework warns about teams that bolt RAG on and call it done. Roachie's RAG is deeply woven into the pipeline: tool matching drives which documentation is injected, schema context drives which databases the LLM knows about, persistent learning drives which mistakes to avoid. RAG isn't a feature -- it's the retrieval foundation that makes everything else work.

- **Five memory sources cover five knowledge gaps** -- The LLM doesn't know which tools exist (Tool RAG), how CockroachDB works (Doc RAG), what the user's database looks like (schema), what failed before (persistent learning), or what was discussed earlier (history). Each memory source addresses one gap. This is comprehensive, not cosmetic.

- **Hybrid retrieval avoids RAG's biggest weakness** -- The framework mentions that RAG reduces hallucination. Roachie's hybrid regex + semantic approach goes further: regex catches exact keyword matches (100% precision), semantic catches paraphrases (broad recall), and the combination achieves 99-100% accuracy across 135 test queries. This is measurably better than embedding-only RAG.

- **Live schema is genuinely novel RAG** -- Most RAG systems retrieve from static document corpora. Roachie queries the actual database for its schema. This means the "memory" is always accurate -- no stale embeddings, no outdated documents, no re-indexing. The framework says "RAG fetches the specific data row the model needs" -- roachie fetches the actual database structure.

- **Persistent learning makes RAG adaptive** -- The failure/success databases mean RAG's context improves over time. A failure logged in session N appears as a warning in session N+1. This is RAG that learns from experience, not just retrieves from a fixed corpus.

### Cons

- **Doc RAG corpus is too small** -- 14 chunks covering only the most common DBA topics. Many legitimate CockroachDB questions (security hardening, multi-region setup, changefeed configuration, scheduled jobs) have no matching doc chunk. The retrieval is accurate but coverage is incomplete.

- **No retrieval feedback loop** -- The system doesn't track which retrieved documents were actually useful. If a doc chunk is consistently retrieved but never relevant, there's no mechanism to downweight it. Retrieval quality is never evaluated.

- **Schema RAG is session-cached** -- Schema context is fetched once at session start. Mid-session DDL changes are invisible until restart. For long-running DBA sessions where schema evolves, this is a gap.

- **Persistent memory is shallow** -- 15 failures and 10 successes, 30-day TTL. This is enough for "don't repeat yesterday's mistake" but not enough for "learn general patterns across hundreds of interactions."

### Room to Improve

1. **Scale doc RAG to 50+ chunks** -- Add chunks for uncovered topics: security, multi-region, changefeeds, import/export, scheduled jobs, connection pooling, SQL tuning. Each ~130 words following the existing format.

2. **Add retrieval quality tracking** -- Log which retrieved tools/docs the LLM actually used. Correlate retrieval accuracy with execution success. Use this data to tune descriptions and thresholds.

3. **Add schema refresh command** -- `/refresh-schema` to re-query the cluster mid-session without restarting.

---

## Layer 3: Agent (The Doer)

**Framework says:** "A goal-oriented worker in a continuous loop: Observe → Reason → Act → Refine."

### How Roachie "Does"

| Agent Phase | Framework Definition | Roachie Implementation |
|------------|---------------------|----------------------|
| **Observe** | Identifies current state | Receives user query + command output from previous iteration |
| **Reason** | Determines next step | LLM generates JSON with reasoning, tool selection, and flags |
| **Act** | Executes commands | Validates and executes cr_* tools against live CockroachDB |
| **Refine** | Checks result, adjusts | Reflexion on failure: analyzes error, tries different approach |

**Agent loop:** `llm_assistant.sh:1381-1533`, max 3 iterations with two continuation triggers:
- `needs_followup=true` -- LLM explicitly requests follow-up to analyze output
- Reflexion -- system detects command failure and forces self-correction

### Pros

- **The observe-reason-act-refine loop is implemented** -- This is not a theoretical architecture; it's running code. The agent observes (query + previous output), reasons (LLM generates structured JSON), acts (validated tool execution), and refines (reflexion on failure). Each phase is distinct, timed, and logged.

- **Refine is genuine self-correction, not retry** -- When a command fails, the reflexion prompt doesn't just retry. It constructs explicit analysis instructions: "Check the error message. Was it a wrong flag, missing parameter, or wrong tool? Review the tool docs. Try a DIFFERENT approach. Do NOT repeat the same command." This is intelligent refinement, not blind retry.

- **The agent acts on the real world** -- The framework distinguishes agents that "create files, send messages, update records." Roachie's agent executes commands against live CockroachDB clusters -- it reads real metrics, queries real data, and could modify real state (within guardrails). The actions have real consequences.

- **Multi-command execution within a single cycle** -- The LLM can generate multiple commands in one response (e.g., `cr_query_stats` + `cr_plan`), executed sequentially with independent validation and error tracking. This is an agent executing a multi-step action, not just one command.

- **The agent loop integrates all three layers** -- Each iteration re-enriches the system prompt (RAG layer) with new tool documentation based on the reasoning from the previous step. The execution goes through the validation pipeline (MCP/connection layer). This is the three layers working as one system.

### Cons

- **No task breakdown** -- The framework says agents "break down the task." Roachie's agent doesn't decompose goals. It goes directly from query to command. "Optimize the movr database" isn't decomposed into [analyze queries → identify missing indexes → generate recommendations → apply changes]. The user must drive each step.

- **Max 3 iterations limits complex tasks** -- The agent loop caps at 3 iterations. Some legitimate workflows (investigate performance across multiple tables, compare schemas across tenants) need 5-10 steps. The agent is cut short before completing the mission.

- **No persistent goals** -- The framework says agents work "until the mission is accomplished." Roachie's agent works until the query is answered -- then waits for the next query. There's no persistent goal like "keep monitoring cluster health and alert on anomalies."

- **Single agent, no delegation** -- The framework mentions agents that "create files, send messages, update records" across different systems. Roachie's agent only executes cr_* tools against CockroachDB. It can't delegate to a Slack agent, a Jira agent, or an email agent.

### Room to Improve

1. **Add goal decomposition** -- When the LLM detects a multi-step objective, output a `plan` field with ordered steps. Execute each step, track progress, allow user to see/modify the plan.

2. **Increase iteration limit for planned workflows** -- When a plan is present, allow up to 10 iterations. The plan provides bounded scope; the cap can be relaxed safely.

3. **Add delegation via MCP client** -- Enable the agent to delegate sub-tasks to external MCP servers: "analyze slow queries (roachie) → post summary to Slack (Slack MCP) → create Jira ticket (Jira MCP)."

---

## The Stack Integration Assessment

The framework's key point: "The power is in the stack." How well do roachie's three layers integrate?

### Integration Scenario: Full Stack in Action

**User:** "The movr database on tenant va is slow. Diagnose and recommend fixes."

**How the stack should work together:**

| Step | Layer | Action | Status in Roachie |
|------|-------|--------|-------------------|
| 1 | Agent | Break task into: [check slow queries, analyze plans, recommend indexes] | **Missing** (no decomposition) |
| 2 | RAG | Retrieve: cr_query_stats docs + "Slow Query Diagnosis" chunk + movr schema | **Working** (hybrid retrieval) |
| 3 | Agent | Reason: "Start with cr_query_stats to find the slowest queries" | **Working** (LLM reasoning) |
| 4 | MCP/Tools | Execute: `cr_query_stats -d movr -t va -h localhost -p 26257 --insecure --top 5` | **Working** (validated execution) |
| 5 | Agent | Observe: sees slow query on rides table, full table scan | **Working** (output fed back) |
| 6 | RAG | Re-enrich: retrieve cr_plan docs based on reasoning | **Working** (re-enrichment) |
| 7 | Agent | Reason: "Run cr_plan on the slowest query to see the execution plan" | **Working** (follow-up reasoning) |
| 8 | MCP/Tools | Execute: `cr_plan -d movr -t va ...` | **Working** (validated execution) |
| 9 | Agent | Observe: sees missing index on rides.city column | **Working** (output analysis) |
| 10 | Agent | Reason: "Recommend adding index, run cr_index_advisor" | **Partially working** (may hit 3-iteration cap) |
| 11 | MCP/Slack | Post summary to team Slack channel | **Missing** (no MCP client) |
| 12 | Agent | Refine: mark task complete, log results | **Partially working** (logs metrics, no task tracking) |

**Steps 1-9 work well.** The three layers cooperate: RAG provides the right tool docs and domain knowledge, the agent reasons and iterates, and the tools execute against the live database.

**Steps 10-12 have gaps.** The 3-iteration cap may cut short the workflow. There's no MCP client for cross-system actions. There's no task completion tracking.

### Integration Strengths

| Integration | How It Works | Quality |
|------------|-------------|---------|
| RAG → Agent | Retrieved tool docs inform agent reasoning | **Strong** -- per-query enrichment with re-enrichment on follow-up |
| Agent → MCP/Tools | Agent's JSON output drives tool execution | **Strong** -- structured handoff with 6-layer validation |
| MCP/Tools → Agent | Tool output fed back for next reasoning step | **Strong** -- truncated, sanitized, prompt-injection-safe |
| RAG → MCP/Tools | Schema context helps agent generate correct flags | **Strong** -- knows real database/table names |
| Agent → RAG | Agent's reasoning triggers re-enrichment with new tool docs | **Strong** -- adapts context per iteration |

### Integration Weaknesses

| Gap | Impact | Fix |
|-----|--------|-----|
| No MCP client (only server) | Agent can't delegate to external systems | Add MCP client for Slack, Jira, etc. |
| No task decomposition | Complex goals handled as single queries | Add plan field in LLM output |
| No cross-system workflow | "Diagnose + alert team" requires manual Slack step | MCP client enables this |
| Iteration cap (3) | Long workflows truncated | Increase cap when plan is present |
| No persistent goal | Agent forgets objective between queries | Add session-level goal context |

---

## Comparison: Roachie vs. The Framework's Onboarding Example

The framework gives a concrete example:

> "Here's what onboarding a client actually looks like:
> → The agent breaks down the task
> → pulls the contract details and your SOPs via RAG
> → creates the Slack channel, updates the CRM, and sends the welcome email via MCP.
> No custom code per step."

**Roachie's equivalent DBA scenario:** "A new database tenant needs to be set up and validated."

| Framework Step | Roachie Equivalent | Works? |
|---------------|-------------------|--------|
| Agent breaks down task | "Create tenant, load schema, validate health" | **No** -- user must issue 3 separate queries |
| RAG pulls knowledge | Tool docs for cr_create_cluster, cr_health, schema context | **Yes** -- hybrid retrieval finds the right tools |
| MCP creates Slack channel | Post setup completion to team chat | **No** -- no MCP client for Slack |
| MCP updates CRM | Log tenant creation in tracking system | **No** -- no MCP client for external systems |
| MCP sends welcome email | Notify team of new tenant availability | **No** -- no MCP client for email |
| No custom code per step | Same pipeline handles all tool types | **Yes** -- unified enrichment + validation + execution |

**Verdict:** Roachie nails the RAG + internal tool execution parts but **lacks the cross-system MCP client capability** that makes the framework's vision compelling. The "power of the stack" diminishes when the agent can only act within one system (CockroachDB) rather than across systems (CockroachDB + Slack + Jira + email).

---

## Summary Scorecard

| Layer | Score | What Works | What's Missing |
|-------|-------|-----------|----------------|
| **MCP (Connection)** | **7/10** | 77 internal tools + 39 MCP-exposed + native tool calling for 3 providers | No MCP client; 38-tool MCP gap; two parallel systems |
| **RAG (Memory)** | **8.5/10** | 5 memory sources, hybrid retrieval, live schema, persistent learning | Small doc corpus (14), no retrieval feedback, session-cached schema |
| **Agent (Doer)** | **6/10** | Observe-reason-act-refine loop, reflexion, multi-command execution | No task decomposition, 3-iteration cap, no cross-system delegation |
| **Stack Integration** | **7/10** | All three layers cooperate per-query; re-enrichment per iteration | No cross-system workflows; no task-level orchestration |

### Overall Stack Score: 7.1/10

---

## The Key Insight

Roachie has all three layers of the stack, and they work well together **within the CockroachDB domain**. The RAG → Agent → Tools flow is production-grade: hybrid retrieval provides accurate context, the agent reasons and self-corrects, and tools execute validated commands against live databases.

**What roachie gets right about the stack:**
- RAG is not "bolted on" -- it's architecturally integrated with five memory sources
- The agent genuinely observes, reasons, acts, and refines (not just prompt → response)
- Tool execution is real (live database operations, not text generation)
- The layers feed each other: RAG enriches the agent, the agent drives tools, tool output feeds back to the agent, and the agent triggers re-enrichment from RAG

**Where the stack breaks down:**
- The agent can't break down complex goals (no planning layer)
- The agent can only act within CockroachDB (no MCP client for external systems)
- The stack is query-scoped, not mission-scoped (no persistent goals across queries)

**The framework's "power is in the stack" vision requires one key upgrade:** MCP client capability. With it, roachie's agent could: diagnose a database issue (internal tools) → post findings to Slack (Slack MCP) → create a remediation ticket (Jira MCP) → schedule a follow-up check (calendar MCP). Without it, the agent is a powerful single-system worker that can't delegate or communicate.

### Top 5 Upgrades to Complete the Stack

1. **Add MCP client capability** (Connection) -- Enable roachie to consume external MCP servers. This is the single highest-impact upgrade: it transforms the agent from a CockroachDB specialist into a DBA workflow orchestrator that can diagnose, communicate, and coordinate across systems.

2. **Add task decomposition** (Agent) -- When the LLM detects a multi-step objective, output a plan with ordered steps. This lets the agent "break down the task" as the framework describes, rather than handling one query at a time.

3. **Scale Doc RAG to 50+ chunks** (Memory) -- Expand the knowledge base to cover the full range of DBA scenarios. The agent can only act on knowledge it has; more knowledge = better actions.

4. **Add cross-system workflow templates** (Stack integration) -- Pre-defined workflows like "diagnose + alert" that chain internal tool execution with external MCP actions. These are the "no custom code per step" patterns the framework envisions.

5. **Add persistent goals** (Agent) -- `/goal monitor tenant va health` sets a session-level objective. All queries are interpreted in this context, and the agent suggests next steps proactively after each result.

---

## CLI Summary (Quick Reference)

### Stack Score: 7.1/10 -- "Three Layers Working, Cross-System Gap"

| Layer | Score | One-Line Assessment |
|-------|-------|---------------------|
| MCP (Connection) | **7/10** | 77 internal + 39 MCP-exposed tools; no MCP client for external systems |
| RAG (Memory) | **8.5/10** | 5 memory sources, hybrid retrieval, live schema; small doc corpus |
| Agent (Doer) | **6/10** | Observe-reason-act-refine loop; no task decomposition, 3-iter cap |
| Stack Integration | **7/10** | Layers cooperate per-query; no cross-system workflows |

### The Key Insight

Roachie has all three stack layers and they integrate well **within the CockroachDB domain**: RAG provides context, the agent reasons and self-corrects, tools execute against live databases, and output feeds back to the agent for refinement. But the stack can't reach beyond CockroachDB -- no MCP client means no cross-system delegation (Slack, Jira, email).

### How the Stack Flows in Roachie

```
User query
  → RAG retrieves: tool docs + CockroachDB knowledge + live schema + past failures
  → Agent reasons: selects tool, generates flags, structures response
  → Tools execute: validated cr_* command against live cluster
  → Agent refines: observes output, re-enriches context, iterates if needed
  → Result: executed database operation with metrics logged
```

### What the Stack Gets Right
- RAG is architecturally integrated (5 memory sources), not bolted on
- Agent has genuine observe-reason-act-refine loop with reflexion
- Tool execution is real (live database operations with 6-layer validation)
- Layers feed each other (RAG enriches agent, agent output triggers re-enrichment)
- No custom code per query (same pipeline handles all 77 tools)

### What the Stack Needs
- MCP client for cross-system workflows (Slack, Jira, Grafana)
- Task decomposition for complex multi-step goals
- Larger knowledge base (14 → 50+ doc chunks)
- Persistent goals across queries (mission-scoped, not query-scoped)

### Top 5 Upgrades to Complete the Stack

1. **MCP client capability** -- consume external MCP servers for cross-system workflows
2. **Task decomposition** -- agent breaks down complex goals into step-by-step plans
3. **Scale Doc RAG** -- 14 → 50+ chunks covering full DBA topic range
4. **Cross-system workflow templates** -- "diagnose + alert" patterns chaining tools + MCP
5. **Persistent goals** -- session-level objectives that guide multi-query workflows
