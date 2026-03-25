# Roachie: "Real AI Stack" Analysis

**Date:** 2026-03-25
**Scope:** Evaluate roachie against the production AI infrastructure framework -- is roachie optimizing prompts or designing systems?

---

## The Framework

The "real AI stack" argues that production AI is not "LLM + RAG" but a layered system:

```
User → Orchestrator → Agents → Tools → Retrieval → LLM → Guardrails → Output
```

Surrounded by cross-cutting concerns:
- Memory (short + long term)
- Identity (for agents)
- Observability (logs, traces)
- Evaluation pipelines
- Cost & governance controls

The key shift: **prompts -> workflows, chatbots -> execution systems, LLMs -> AI infrastructure.**

---

## Roachie's Stack Map

Before evaluating each layer, here's how roachie maps to the reference architecture:

```
Reference Layer          Roachie Implementation                          Status
─────────────────────────────────────────────────────────────────────────────────
User                     CLI REPL (roachie-nl), batch file (roachie-     Complete
                         batch), roachman menu, MCP client

Orchestrator             _nl_process_query() — single orchestrator      Partial
                         Agent loop (3 iterations, reflexion)

Agents                   Single agent (no specialization)               Minimal
                         Same LLM handles all query types

Tools                    77 cr_* CLI tools, 39 MCP-exposed              Complete
                         Tool registry with descriptions + embeddings

Retrieval (RAG)          Hybrid regex+semantic tool matching             Strong
                         Doc RAG (14 chunks), live schema context

LLM                      5 providers (Claude, GPT, Gemini, Vertex,      Complete
                         Ollama), 2 fine-tuned local models

Guardrails               6-layer validation pipeline                    Strong
                         RBAC, SQL mutation guard, metachar blocking

Output                   Structured JSON → validated commands →          Complete
                         executed against live CockroachDB

Memory                   Session history + persistent failure/success   Partial
                         DBs (cross-session, 30-day TTL)

Identity                 RBAC roles (admin/dba/analyst/monitor)         Minimal
                         No agent identity, no user profiles

Observability            CSV metrics, JSONL audit, debug logs,          Partial
                         cost tracking, PII masking

Evaluation               Batch testing (73+ prompts), assertions,       Partial
                         multi-provider comparison

Cost & Governance        Per-query cost tracking, session budgets,      Partial
                         20+ model pricing, RBAC
```

**Overall: Roachie is designing a system, not optimizing a prompt.** But the system has strong and weak layers.

---

## Layer-by-Layer Evaluation

### 1. User Layer

**What the framework expects:** Multiple input channels, clear user experience, context preservation.

**What roachie has:**

| Entry Point | Mode | Interaction | Context |
|-------------|------|-------------|---------|
| `roachie-nl` | Interactive REPL | NL queries, voice input, special commands | Session-scoped |
| `roachie-batch` | File-driven batch | One prompt per line, assertions, JSONL output | Stateless per prompt |
| `roachman` menu 8 | Integrated NL | Same REPL within cluster management context | Inherits CA/CB topology |
| MCP Toolbox | API-driven | Any MCP client (Claude Desktop, custom agents) | Stateless |

#### Pros

- **Four distinct user interfaces** -- Interactive, batch, integrated, and API. This is not a chatbot with one input channel; it's a system with multiple access patterns for different use cases (exploration, testing, management, integration).
- **Voice input** -- `llm_voice.sh` provides speech-to-text via Whisper API. Type `v` at the prompt for 3-second recording, `v 5` for 5 seconds. This is a genuine multimodal input channel.
- **Wake word personalities** -- `roachie:` (friendly) and `roachman:` (professional) change the response tone. Small touch, but shows user experience design beyond basic prompt I/O.
- **Input validation** -- 2000-character limit prevents token budget overflow. Empty/whitespace inputs rejected. Special commands (`/help`, `/verbose`, `/quiet`, `switch`) handled before LLM call.

#### Cons

- **No web UI** -- All interfaces are terminal-based. No browser dashboard, no Slack bot, no mobile access. For a DBA tool this is appropriate, but limits accessibility.
- **No session persistence across restarts** -- If the user closes the terminal, all conversation context is lost. Only the persistent failure/success databases survive.
- **No user onboarding** -- First-time users face a provider selection menu with no guidance on which to choose. No tutorial, no guided setup.

#### Room to Improve

1. **Add a `/status` command** -- Show current connection, provider, model, session cost, query count, loaded memories. Gives users situational awareness without leaving the REPL.
2. **Add session export/import** -- `roachie-nl --save-session session.json` and `--load-session session.json` to preserve and restore conversation context across restarts.

---

### 2. Orchestrator Layer

**What the framework expects:** Central coordinator that routes requests, manages workflow state, handles branching logic.

**What roachie has:** `_nl_process_query()` in `llm_assistant.sh:1323-1645` -- a single orchestration function.

**Orchestration flow:**
```
_nl_process_query():
  1. Clean input (strip wake words, detect voice)
  2. Load persistent learning (failures + successes)
  3. Enrich system prompt (regex + semantic tool matching + doc RAG)
  4. Agent loop (max 3 iterations):
     a. Call LLM → parse JSON
     b. Classify complexity (informational / simple / multi-command)
     c. Execute commands → capture output
     d. Check needs_followup or reflexion trigger
     e. Re-enrich prompt if continuing
  5. Calculate cost, log metrics
  6. Collect feedback (if enabled)
```

#### Pros

- **Single orchestrator with clear phases** -- The pipeline has distinct, measurable phases: enrichment (timed), API call (timed), execution (timed), logging. This is orchestration, not ad-hoc prompting.
- **Dynamic re-enrichment** -- Each agent loop iteration re-runs tool matching on the combined query + reasoning from the previous step. This means the orchestrator adapts the context per iteration, not just replaying the same prompt.
- **Complexity classification** -- The orchestrator classifies each query as `informational`, `simple`, `multi-command`, or `multi-step(N)`. This classification drives different execution paths (informational queries skip execution and cost tracking).
- **Batch orchestration** -- `roachie-batch` wraps the same orchestrator with file-based input, rate limiting, assertion checking, and JSONL reporting. Same core pipeline, different execution context.

#### Cons

- **No routing** -- The orchestrator sends every query through the same pipeline. There's no branching: "if this is a schema question, route to schema-specialist; if it's a performance question, route to performance-specialist." Every query gets the same system prompt structure.
- **No workflow state machine** -- The orchestrator is a linear pipeline with a bounded loop. It cannot model complex workflows like "first do X, then if Y, do Z, otherwise do W." Each iteration is independent except for accumulated context.
- **No parallel orchestration** -- Multi-command responses are executed sequentially. If the LLM generates three independent commands, they wait for each other. The orchestrator cannot fork and join.
- **No checkpoint/resume** -- If the orchestrator is interrupted mid-execution (Ctrl-C, network timeout), there's no recovery. The partial work is logged but cannot be resumed.

#### Room to Improve

1. **Add query routing** -- Before the main pipeline, classify the query into categories (schema, performance, cluster ops, informational) and select a category-specific prompt profile. This adds routing without requiring separate agents.
2. **Add a plan-then-execute mode** -- For complex queries, have the LLM first output a plan (list of steps). Show the plan to the user for approval, then execute each step. This adds workflow state management to the existing loop.
3. **Add parallel command execution** -- When the LLM generates multiple independent commands, execute them in parallel (background jobs). Join results before presenting to the user. The validation pipeline already processes commands independently.

---

### 3. Agents Layer

**What the framework expects:** Specialized agents with distinct roles, capabilities, and identities. Agents are composed, not monolithic.

**What roachie has:** A single, general-purpose agent.

#### Pros

- **The single agent is well-constructed** -- It has a clear role ("CockroachDB DBA assistant"), structured output (JSON with reasoning, commands, needs_followup), domain-specific knowledge (77 tools, 14 doc chunks, live schema), and self-correction (reflexion). As a single agent, it's production-quality.
- **Agent behavior is configurable** -- Temperature, max iterations, reflexion toggle, Doc RAG toggle, streaming toggle, native tool calling toggle, RBAC role. The agent's behavior can be tuned without code changes.
- **Batch mode as "headless agent"** -- `roachie-batch` runs the agent non-interactively with auto-fallback provider selection. This is effectively a headless agent execution mode.

#### Cons

- **No agent specialization** -- The same agent handles everything: schema inspection, performance analysis, cluster management, informational queries, DDL generation. A system with specialized agents could have deeper domain expertise per agent.
- **No multi-agent composition** -- There's no supervisor agent, no planner agent, no critic agent. Complex tasks can't be decomposed across agents with different capabilities.
- **No agent identity** -- The agent has no persistent identity, no name (beyond wake word personalities), no memory of who it is across sessions. Each session starts from zero self-knowledge, relying entirely on the system prompt.
- **No inter-agent communication** -- Even if multiple agents existed, there's no message-passing infrastructure, no shared blackboard, no coordination protocol.

#### Room to Improve

1. **Add prompt profiles as lightweight agents** -- Create specialized system prompt profiles for different domains: `schema_agent.txt`, `performance_agent.txt`, `cluster_ops_agent.txt`. The query classifier routes to the appropriate profile. Same LLM, different specialization via prompt.
2. **Add a critic/validator pass** -- After the LLM generates a command, run a second (cheaper) LLM call to validate: "Is this the right tool for this query? Are the flags correct?" This acts as a quality-check agent without requiring a full multi-agent framework.

---

### 4. Tools Layer

**What the framework expects:** APIs, databases, external systems. Tools are registered, versioned, and access-controlled.

**What roachie has:** This is roachie's **strongest layer**.

| Aspect | Implementation | Quality |
|--------|---------------|---------|
| Tool count | 77 cr_* CLI tools | Complete |
| Tool registry | `tool_descriptions.txt` (77 curated descriptions) | Complete |
| Tool discovery | Regex + semantic matching (99-100% accuracy) | Strong |
| Tool documentation | `--help` output injected per-query | Complete |
| Tool versioning | `tools/tools.25.2/` and `tools/tools.25.4/` with symlink | Complete |
| Tool access control | 4-tier RBAC (admin/dba/analyst/monitor) | Complete |
| Tool validation | 6-layer pipeline (whitelist, normalize, metachar, SQL, existence, RBAC) | Strong |
| Tool execution | Safe array-based invocation, no shell interpretation | Secure |
| External integration | MCP Toolbox (39 tools), native tool calling | Complete |
| Environment interaction | Live CockroachDB clusters (read + write with guards) | Complete |

#### Pros

- **Production-grade tool infrastructure** -- 77 tools with curated descriptions, pre-computed embeddings (3 providers), regex keyword matching, RBAC gating, 6-layer validation, and safe execution. This is not "function calling with a list of APIs" -- it's a designed tool system.
- **MCP as a standard interface** -- Exposing 39 tools via Model Context Protocol means any MCP-compatible agent can use roachie's tools. This decouples the tools from roachie's orchestrator.
- **Version-aware tooling** -- Tools are versioned per CockroachDB release (25.2 vs 25.4) with a symlink-based version selection. The system validates that tool version matches cluster version at startup.
- **Security-first design** -- SQL mutations are blocked at the validation layer. Shell metacharacters are blocked. Tool existence is verified. RBAC limits tool access by role. The LLM cannot bypass these guardrails regardless of prompt.

#### Cons

- **Tools are opaque to the LLM** -- The LLM sees tool descriptions and `--help` output, but it doesn't see tool source code, execution patterns, or common flag combinations. It must infer correct usage from documentation alone.
- **No tool output schema** -- Tools return unstructured text output. The LLM must parse freeform text to extract query IDs, table names, or metrics. A structured output schema (JSON) would enable reliable data extraction between tool calls.
- **No tool composition primitives** -- There's no way to express "pipe output of tool A into tool B" or "run tool A, extract field X, pass to tool B as flag Y." Each tool call is independent.

#### Room to Improve

1. **Add structured output mode to key tools** -- For tools frequently chained (cr_query_stats -> cr_plan), add a `--json` flag that outputs structured data. This enables reliable field extraction for multi-step workflows.
2. **Add tool chain documentation** -- Create a `tool_chains.txt` file documenting common sequences: "For query optimization: cr_query_stats -> cr_plan -> cr_indexes." Inject relevant chains into the system prompt when the first tool in a chain is selected.

---

### 5. Retrieval (RAG) Layer

**What the framework expects:** Retrieval-augmented generation with context-aware, adaptive retrieval.

**What roachie has:** A three-source hybrid retrieval system.

| Source | Method | Content | Adaptivity |
|--------|--------|---------|------------|
| Tool catalog | Regex + cosine similarity | 77 tool descriptions | Static corpus, dynamic matching |
| Doc RAG | Cosine similarity | 14 CockroachDB knowledge chunks | Static corpus |
| Live schema | SQL queries to cluster | Databases, tables, columns | **Real-time** (queried at session start) |

#### Pros

- **Hybrid retrieval is the right design** -- Regex (high precision, zero latency) + semantic (broad recall, synonym-aware) + live data (real-time accuracy). This is not basic RAG; it's a designed retrieval system with complementary strategies.
- **Live schema is genuinely adaptive** -- Unlike static RAG corpora, the schema context is queried from the actual database. When tables are added or dropped, the system's knowledge updates automatically at next session start.
- **Embedding reuse** -- A single query embedding is computed once and reused for both tool matching and Doc RAG retrieval, eliminating redundant API calls (`llm_prompt.sh:417-422`).
- **Three-provider fallback** -- Ollama (free/local) -> OpenAI (API) -> Gemini (API) -> regex-only. The system never fails due to missing embeddings; it degrades gracefully.

#### Cons

- **Static document corpus** -- The 14 CockroachDB knowledge chunks are hand-curated and never auto-updated. New CockroachDB versions, features, and best practices require manual corpus updates.
- **No query rewriting** -- The user's raw query is embedded directly. No spelling correction, no query expansion, no synonym injection. "show me running quries" (typo) relies on embedding similarity to match "queries."
- **No re-ranking** -- Retrieved results are ranked by bi-encoder cosine similarity only. No cross-encoder re-ranking stage to refine the top-K results.
- **No retrieval feedback loop** -- The system doesn't learn which retrievals led to successful tool selections. Retrieval quality is never evaluated or improved from usage data.

#### Room to Improve

1. **Add retrieval quality tracking** -- Log which retrieved tools were actually used by the LLM. If the LLM consistently ignores a retrieved tool, it's a false positive. Use this data to adjust similarity thresholds or tool descriptions.
2. **Add query normalization** -- Before embedding, normalize the query: lowercase, strip filler words, expand abbreviations ("db" -> "database", "tbl" -> "table"). This improves retrieval precision for informal queries.

---

### 6. LLM Layer

**What the framework expects:** The LLM is one component in a larger system, not the system itself.

**What roachie has:** The LLM is correctly positioned as a component -- it receives a constructed context and returns structured output that the system validates and executes.

| Aspect | Implementation |
|--------|---------------|
| Provider count | 5 (Anthropic, OpenAI, Gemini, Vertex AI, Ollama) |
| Model count | 15+ (cloud) + 5 local (roachie variants) |
| Fine-tuned models | 2 (roachie-8b-ft at 90%, roachie-nemo-ft at 89%) |
| Context management | Per-provider limits (16K-1M), 80% trim threshold |
| Output format | Structured JSON (reasoning, commands, needs_followup) |
| Streaming | Supported for Anthropic, OpenAI, Gemini (not Ollama) |
| Token tracking | Per-query prompt/completion token counts |
| Temperature | Provider-specific defaults, runtime configurable |

#### Pros

- **LLM is a pluggable component, not the system** -- The LLM receives a fully constructed system prompt (tools, schema, history, persistent learning) and returns structured JSON. The system validates, executes, and logs independently of which LLM was used. This is the correct architectural position.
- **Provider abstraction** -- `_call_llm_api()` in `llm_providers.sh` presents a unified interface. The orchestrator doesn't know or care which LLM backend is executing. Adding a new provider requires only a plugin file in `src/lib/providers/`.
- **Fine-tuned local models** -- Two LoRA-trained models (Llama 8B, Mistral Nemo 12B) achieve 89-90% accuracy on the domain task. This demonstrates that roachie treats the LLM as a trainable component, not a fixed external service.
- **Structured output contract** -- The LLM must return JSON with specific fields (reasoning, user_message, commands[], needs_followup). This contract decouples the LLM from the execution layer.

#### Cons

- **No model routing by query type** -- All queries go to the same model regardless of complexity. Simple tool lookups use the same expensive model as complex multi-step analyses. No tiered model selection.
- **No fallback on output quality** -- If the LLM returns malformed JSON, the system retries with the same model. It doesn't escalate to a more capable model. A production system might try GPT-4.1-mini first, then escalate to GPT-4.1 if the output is low-quality.
- **No caching of LLM responses** -- Identical queries always trigger a new LLM call. A response cache for common queries (e.g., "show databases") would eliminate redundant API calls.

#### Room to Improve

1. **Add tiered model selection** -- Route simple queries (identified by the query classifier) to cheaper/faster models (GPT-4.1-mini, Gemini Flash) and complex queries to more capable models (GPT-4.1, Claude Sonnet). This reduces cost without reducing quality.
2. **Add response caching** -- Cache LLM responses for deterministic queries (no schema/topology dependency). "What does cr_tables do?" always has the same answer. Cache hit = zero cost, zero latency.

---

### 7. Guardrails Layer

**What the framework expects:** Safety checks, output validation, content filtering, compliance enforcement.

**What roachie has:** This is roachie's **second strongest layer** after tools.

**6-layer validation pipeline** (`llm_exec.sh:1-336`):

| Layer | What It Checks | File:Line |
|-------|---------------|-----------|
| 1. Whitelist | Only cr_* tools + `cockroach sql/workload/node/debug` allowed | `llm_exec.sh:50-85` |
| 2. Flag normalization | Strips hallucinated `=` in flags, fixes known patterns | `llm_exec.sh:90-120` |
| 3. Metachar blocking | Pipes, redirects, $(), backticks, process substitution blocked | `llm_exec.sh:125-180` |
| 4. SQL mutation guard | DROP, DELETE, TRUNCATE, UPDATE, INSERT, ALTER, etc. blocked | `llm_exec.sh:185-230` |
| 5. Tool existence | Verifies the tool binary exists in tools_dir | `llm_exec.sh:235-250` |
| 6. RBAC | Role-based tool access (admin/dba/analyst/monitor) | `llm_exec.sh:255-320` |

**Additional guardrails:**
- Prompt injection sanitization (`_sanitize_for_prompt()` in `llm_metrics.sh:412-436`)
- Input length validation (2000 char max)
- Output truncation (4000 chars for follow-up context)
- PII masking in logs (`_mask_sensitive_output()`)
- User confirmation before execution (unless batch/quiet mode)

#### Pros

- **Defense in depth** -- Six independent validation layers means a single bypass doesn't compromise the system. Even if the LLM generates a valid-looking destructive command, the SQL mutation guard blocks it. Even if it bypasses that, RBAC blocks it for non-admin roles.
- **Deterministic, not probabilistic** -- Guardrails are rule-based, not LLM-based. The whitelist check is a literal string match. The metachar check is a regex. These cannot be confused, hallucinated past, or prompt-injected.
- **Prompt injection defense** -- `_sanitize_for_prompt()` strips 10+ injection patterns (case-insensitive) from all data injected into prompts. Error messages, command output, and persistent learning entries are all sanitized before reaching the LLM.
- **Separation of validation from execution** -- `_validate_command()` returns a normalized command and a block reason. `_safe_execute_command()` only receives pre-validated commands. The validation decision is made before execution starts.

#### Cons

- **No output content guardrails** -- The LLM's `user_message` and `reasoning` fields are displayed to the user without content filtering. If the LLM hallucinates incorrect information ("this cluster has 5 nodes" when it has 3), there's no factual verification.
- **No rate limiting on tool execution** -- Within a single query, the LLM can generate unlimited commands (within the agent loop's 3-iteration cap). There's no guard against "execute 50 commands in a row."
- **Guardrails are bypass-able via SQL** -- While direct mutations are blocked, `cockroach sql -e "SELECT * FROM ..."` is allowed. Complex SQL that modifies data via CTEs or function calls could potentially bypass the mutation guard.

#### Room to Improve

1. **Add factual verification for key claims** -- When the LLM says "this cluster has N nodes," verify against the cached topology. When it says "table X has Y rows," verify against cached schema. Flag discrepancies to the user.
2. **Add per-query command count limits** -- Cap at 5-10 commands per query to prevent runaway execution. Most legitimate queries need 1-3 commands.

---

### 8. Output Layer

**What the framework expects:** Structured, validated, actionable output. Not just text -- execution results.

**What roachie has:**

| Output Type | Format | Destination |
|-------------|--------|-------------|
| User message | Formatted text | Terminal (colored, styled) |
| Command execution | Tool stdout/stderr | Terminal |
| Metrics | CSV row | `sql_audit/nl_queries_*.csv` |
| Batch results | JSONL record | `sql_audit/nl_batch_*.jsonl` |
| Feedback | JSONL + CSV | Feedback files |
| Debug trace | Timestamped log | Debug log file |

#### Pros

- **Output is execution, not text** -- Roachie's primary output is executed commands against live databases, not prose. The `user_message` field is secondary; the real value is the validated, executed cr_* tool invocation. This is the "chatbot -> execution system" shift.
- **Structured JSON contract** -- The LLM's output is parsed JSON, not freeform text. Fields are extracted programmatically (`jq`), not pattern-matched. This makes the output layer reliable and testable.
- **Multiple output channels** -- Terminal display (user-facing), CSV metrics (analytics), JSONL audit (compliance), debug logs (development). Each channel serves a different consumer.
- **Execution timing in output** -- Every command's execution time is measured and displayed: `[OK:123ms]`. This gives users immediate performance feedback.

#### Cons

- **No structured data output** -- Command results are displayed as raw text. There's no JSON/table/chart rendering. A result like "15 tables found" could be rendered as a formatted table instead of raw `cockroach sql` output.
- **No output history** -- Past results are not stored in a queryable format. Users can't say "show me the results from my last query" or "compare today's slow queries with yesterday's."

#### Room to Improve

1. **Add output formatting** -- Detect tabular output from commands and render as aligned tables with headers. Many cr_* tools output TSV/CSV that could be displayed more readably.
2. **Add output persistence** -- Store the last N query results in a session file. Allow `!!` to re-display the last result, `!3` for the third-to-last, etc.

---

## Cross-Cutting Concerns

### Memory

| Type | Roachie Implementation | Maturity |
|------|----------------------|----------|
| Short-term (session) | `_nl_messages_json` (10 turns max), `recent_failures[]`/`recent_successes[]` (3 each) | **Adequate** |
| Long-term (persistent) | `NL_FAILURES_DB` (15 entries, 30-day TTL), `NL_SUCCESSES_DB` (10 entries, 30-day TTL) | **Basic** |
| Working memory | Cached topology, cached schema, cached embedding provider | **Good** |
| Semantic memory | Pre-computed tool + doc embeddings | **Good** |
| Episodic memory | None (no interaction trace replay) | **Missing** |

**Verdict:** Memory exists at multiple levels but is shallow. The persistent databases are too small (15/10 entries) and too lossy (no summarization on eviction) to serve as true long-term memory.

### Identity

| Aspect | Implementation |
|--------|---------------|
| Agent identity | None (agent has no name, no persistent self-model) |
| User identity | RBAC role only (admin/dba/analyst/monitor) |
| User profiles | None (no per-user preferences or history) |
| Session identity | Trace ID per query, session cost file by PID |

**Verdict:** Identity is minimal. RBAC provides role-based access control, but there's no user modeling, no agent identity, and no personalization.

### Observability

| Observable | Storage | Queryable? |
|-----------|---------|-----------|
| Per-query metrics | CSV | Yes (any spreadsheet/SQL tool) |
| Batch results | JSONL | Yes (jq, Python) |
| Debug traces | Log file | Manually (grep) |
| Session cost | Temp file | No (session-scoped) |
| Persistent learning | JSONL | Yes (jq) |

**Verdict:** Observability data is comprehensive but requires external tools to analyze. No built-in dashboards, no trend analysis, no anomaly detection.

### Evaluation Pipelines

| Evaluation | Method | Automation |
|-----------|--------|------------|
| Accuracy testing | `roachie-batch` with assertions | Manual trigger |
| Multi-provider comparison | Same prompts across providers | Manual trigger |
| Embedding accuracy | `test_embeddings.sh` (135 queries) | Manual trigger |
| Unit tests | 38 suites, ~500 tests | CI (GitHub Actions) |
| Integration tests | 9 suites (cluster required) | Manual trigger |

**Verdict:** Evaluation infrastructure is solid but manually triggered. No continuous evaluation, no automated regression detection, no quality gates in the deployment pipeline.

### Cost & Governance

| Control | Implementation |
|---------|---------------|
| Per-query cost tracking | `_llm_calculate_cost_cents()` for 20+ models |
| Session cost accumulator | Displayed after each query |
| Session budget | `NL_BUDGET` env var (warns on approach) |
| RBAC | 4-tier role system gating tool access |
| SQL mutation blocking | Deterministic guard in validation pipeline |
| PII masking | `_mask_sensitive_output()` in logs |
| Audit trail | CSV + JSONL per query |

**Verdict:** Cost tracking and governance are strong for a CLI tool. Per-query cost display with model-specific pricing for 20+ models is production-grade. RBAC provides governance. Main gap: no organization-level cost controls or usage quotas.

---

## The Key Shift: Where Does Roachie Stand?

The framework identifies three key shifts happening in production AI:

### 1. From Prompts -> Workflows

| Aspect | Prompt-Oriented | Workflow-Oriented | Roachie |
|--------|----------------|-------------------|---------|
| Input processing | Raw user text to LLM | Classified, routed, enriched | **Workflow** (enrichment pipeline) |
| Execution | LLM output displayed as text | LLM output drives tool execution | **Workflow** (validated execution) |
| Error handling | User retries manually | System self-corrects (reflexion) | **Workflow** (reflexion loop) |
| Multi-step | One prompt, one response | Iterative plan-execute cycles | **Transitional** (bounded loop, no planning) |

**Verdict: Roachie has moved past prompts into workflows**, but the workflows are bounded (3 iterations) and reactive (no planning). The enrichment pipeline (regex + semantic + Doc RAG + schema + persistent learning) is a genuine workflow, not prompt engineering.

### 2. From Chatbots -> Execution Systems

| Aspect | Chatbot | Execution System | Roachie |
|--------|---------|-------------------|---------|
| Primary output | Text responses | Executed actions | **Execution** (validated tool invocation) |
| Environment awareness | None | Real-time state | **Execution** (live schema, topology) |
| Safety | "Don't do harmful things" prompt | Deterministic guardrails | **Execution** (6-layer validation) |
| Persistence | Chat history | Operational logs + learning | **Execution** (metrics, audit, learning DBs) |
| Batch operation | N/A | Automated test suites | **Execution** (roachie-batch) |

**Verdict: Roachie is firmly an execution system.** It executes validated commands against live databases, not just generates text. The 6-layer guardrail pipeline is a security architecture, not a prompt instruction.

### 3. From LLMs -> AI Infrastructure

| Aspect | LLM-Centric | AI Infrastructure | Roachie |
|--------|------------|-------------------|---------|
| LLM role | The product | One component | **Infrastructure** (pluggable, abstracted) |
| Model independence | Tied to one model | Provider-agnostic | **Infrastructure** (5 providers, unified API) |
| Custom models | Use as-is | Fine-tuned for domain | **Infrastructure** (2 LoRA models) |
| Tooling | Prompt engineering | Tool registry + validation | **Infrastructure** (77 tools, MCP, embeddings) |
| Testing | Manual | Automated evaluation | **Partial** (batch testing exists, not continuous) |

**Verdict: Roachie treats the LLM as infrastructure, not the product.** The LLM is one component in a larger system with its own tool registry, validation pipeline, retrieval system, and evaluation framework.

---

## Summary Scorecard

| Layer / Concern | Score | Strongest Aspect | Biggest Gap |
|----------------|-------|-------------------|-------------|
| User | **7/10** | 4 input modes (REPL, batch, menu, MCP) | No web UI, no session persistence |
| Orchestrator | **6/10** | Phased pipeline with re-enrichment | No routing, no planning, no parallelism |
| Agents | **4/10** | Well-constructed single agent | No specialization, no composition |
| Tools | **9/10** | 77 tools, MCP, RBAC, 6-layer validation | No structured output, no composition |
| Retrieval | **8/10** | Hybrid regex+semantic+live schema | Static doc corpus, no re-ranking |
| LLM | **8/10** | 5 providers, 2 fine-tuned, pluggable | No model routing, no response caching |
| Guardrails | **9/10** | 6-layer deterministic validation pipeline | No output factual verification |
| Output | **7/10** | Execution-based, structured JSON, multi-channel | No formatted display, no history |
| Memory | **5/10** | Session + persistent + cached schema | Shallow, lossy, no episodic replay |
| Identity | **3/10** | RBAC roles | No user profiles, no agent identity |
| Observability | **6/10** | Rich CSV/JSONL metrics, cost tracking | No dashboards, no anomaly detection |
| Evaluation | **6/10** | Batch testing with assertions | Manual triggers, no continuous eval |
| Cost & Governance | **7/10** | Per-query cost, 20+ models, RBAC, PII masking | No org-level controls |

### Overall System Design Score: 6.5/10

### The Bottom Line

**Roachie is designing a system, not optimizing a prompt.**

It has a clear layered architecture with separation of concerns: the LLM is a pluggable component (not the system), tools have their own registry and validation pipeline (not just function definitions), retrieval uses multiple complementary strategies (not just embeddings), and guardrails are deterministic (not probabilistic).

The strongest layers are **Tools** (9/10) and **Guardrails** (9/10) -- exactly the layers that distinguish an execution system from a chatbot.

The weakest layers are **Identity** (3/10) and **Agents** (4/10) -- these are the layers that distinguish a multi-agent platform from a single-agent tool.

### Top 5 Upgrades to Close the Gaps

1. **Add query routing + prompt profiles** (Orchestrator + Agents) -- Classify queries and route to domain-specific prompt profiles. This adds lightweight multi-agent behavior without a framework rewrite. Elevates both the Orchestrator (routing) and Agent (specialization) layers.

2. **Add daemon/watch mode** (User + Orchestrator + Output) -- Persistent monitoring transforms the system from reactive to proactive. `roachie-watch "cluster health" --interval 60 --alert webhook` adds event-driven orchestration and persistent output.

3. **Automate evaluation pipeline** (Evaluation + Learning) -- Run batch tests automatically after each model update. Compare accuracy against baseline. Block deployment if accuracy drops. This closes the evaluation -> learning -> deployment loop.

4. **Add response caching + tiered model selection** (LLM + Cost) -- Cache responses for deterministic queries. Route simple queries to cheap models, complex queries to capable models. Reduces cost by 50-70% on typical workloads.

5. **Deepen memory** (Memory + Identity) -- Add episodic replay (store full interaction traces, retrieve similar past interactions), user profiles (per-user preferences and common queries), and memory summarization (compact old entries instead of dropping them).

---

## CLI Summary (Quick Reference)

### System Design Score: 6.5/10 -- "Designing Systems, Not Optimizing Prompts"

| Layer / Concern | Score | One-Line Assessment |
|----------------|-------|---------------------|
| User | **7/10** | 4 input modes; no web UI or session persistence |
| Orchestrator | **6/10** | Phased pipeline with re-enrichment; no routing or planning |
| Agents | **4/10** | Well-built single agent; no specialization or composition |
| Tools | **9/10** | 77 tools, MCP, RBAC, 6-layer validation; best-in-class |
| Retrieval | **8/10** | Hybrid regex+semantic+live schema; static doc corpus |
| LLM | **8/10** | 5 providers, 2 fine-tuned, pluggable; no model routing |
| Guardrails | **9/10** | 6-layer deterministic validation; no output fact-checking |
| Output | **7/10** | Execution-based with multi-channel logging; raw display |
| Memory | **5/10** | Session + persistent + cached; shallow and lossy |
| Identity | **3/10** | RBAC roles only; no user profiles or agent identity |
| Observability | **6/10** | Rich metrics + cost tracking; no dashboards |
| Evaluation | **6/10** | Batch testing + assertions; manually triggered |
| Cost & Governance | **7/10** | Per-query cost for 20+ models, RBAC, PII masking |

### The Key Shift Assessment

| Shift | Status | Evidence |
|-------|--------|----------|
| Prompts -> Workflows | **Achieved** | Enrichment pipeline, reflexion, re-enrichment per iteration |
| Chatbots -> Execution Systems | **Achieved** | Validated tool execution against live databases, not text generation |
| LLMs -> AI Infrastructure | **Mostly achieved** | LLM is pluggable component; gaps in evaluation automation and continuous learning |

### What Roachie Gets Right (System Design)
- LLM is a component, not the product (5 providers, unified API, fine-tuned models)
- Tools have their own infrastructure (registry, embeddings, validation, MCP)
- Guardrails are deterministic, not probabilistic (6-layer pipeline)
- Retrieval uses complementary strategies (regex + semantic + live data)
- Output is execution, not text (validated commands against live databases)

### What Roachie Needs (Infrastructure Gaps)
- No query routing or agent specialization (everything through one pipeline)
- No persistent/daemon execution (reactive only, no monitoring)
- No automated evaluation -> learning -> deployment loop
- Shallow memory (15 failures, 10 successes, no episodic replay)
- No user identity or personalization

### Top 5 Upgrades by Impact
1. **Query routing + prompt profiles** -- lightweight multi-agent via specialized prompts
2. **Daemon/watch mode** -- reactive -> proactive, copilot -> monitoring agent
3. **Automated evaluation pipeline** -- continuous quality gates, regression detection
4. **Response caching + tiered models** -- 50-70% cost reduction on typical workloads
5. **Deeper memory** -- episodic replay, user profiles, summarization on eviction
