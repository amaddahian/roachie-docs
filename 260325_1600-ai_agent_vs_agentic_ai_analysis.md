# Roachie: AI Agent vs Agentic AI Analysis

**Date:** 2026-03-25
**Scope:** Evaluate roachie's AI/NL subsystem against the 10-dimension AI Agent -> Agentic AI evolution framework

---

## Position Summary

Roachie sits **between AI Agent and Agentic AI** -- it has outgrown the "intelligent responder" pattern in several dimensions (memory, tool usage, decision making) but hasn't yet reached full autonomous goal-driven execution. It's best described as a **semi-agentic system**: it autonomously selects tools, executes multi-step plans, self-corrects on failure, and learns from experience -- but it still operates within a user-triggered, single-objective loop rather than pursuing persistent goals.

| Dimension | AI Agent (1-3) | Agentic AI (7-10) | Roachie Score | Position |
|-----------|---------------|-------------------|---------------|----------|
| 1. Intelligence Models | Single LLM | Multi-model, planner-executor | **7/10** | Agentic |
| 2. Architecture & Frameworks | Single-agent, linear | Multi-agent, orchestrated | **5/10** | Transitional |
| 3. Memory Systems | Session memory | Long-term, episodic, semantic | **6/10** | Transitional |
| 4. Tool Usage & Actions | Predefined, user-triggered | Autonomous selection, structured registry | **8/10** | Agentic |
| 5. Knowledge & Retrieval | Basic RAG | Adaptive RAG, hybrid search | **7/10** | Agentic |
| 6. Orchestration & Workflows | Sequential flows | Planning loops, event-driven | **6/10** | Transitional |
| 7. Decision Making | Reactive, prompt-driven | Goal-oriented, self-evaluating | **7/10** | Agentic |
| 8. Deployment | Chatbot/copilot | Autonomous platform | **4/10** | Agent-side |
| 9. Monitoring & Observability | Basic logs | Deep analytics, feedback loops | **7/10** | Agentic |
| 10. Learning & Improvement | Manual tuning | Continuous feedback pipelines | **6/10** | Transitional |

**Overall: 6.3/10** -- Solidly in the transitional zone, with notable agentic capabilities in tool usage, knowledge retrieval, and decision making.

---

## 1. Intelligence Models

**AI Agent pattern:** Single LLM, stateless interactions, prompt -> response.
**Agentic AI pattern:** Multi-model reasoning, planner-executor, hybrid local + cloud.

### How Roachie Works

Roachie implements a **multi-model architecture** across two distinct roles:

| Role | Models | Purpose |
|------|--------|---------|
| Embedding (understanding) | nomic-embed-text, text-embedding-3-small, gemini-embedding-001 | Semantic tool matching + Doc RAG retrieval |
| Generation (reasoning) | Claude Sonnet 4.6, GPT-4.1/5, Gemini 2.5, Llama 3.1 8B LoRA, Mistral Nemo 12B LoRA | Command generation + reasoning |

Provider selection in `llm_providers.sh` dispatches to 5 backend plugins (anthropic, openai, gemini, vertex, ollama), each with model-specific configuration. The system supports **hybrid local + cloud inference** -- Ollama models run locally with zero API cost while cloud providers handle complex queries.

### Pros

- **Multi-model by design** -- Separate embedding and generation models is a genuine multi-model architecture, not just "one LLM with different prompts." The embedding model (encoder-only) and generation model (decoder-only) are architecturally different and optimized for their specific roles.
- **Hybrid local + cloud** -- Users can run fully local (Ollama) or fully cloud (Gemini/Claude/OpenAI) or mixed. The `_detect_llm_provider()` auto-detection in `llm_providers.sh` finds the best available model without user configuration.
- **Provider-agnostic dispatch** -- `_call_llm_api()` presents a unified interface; adding a new provider requires only a new plugin file in `src/lib/providers/`. The shared request builders (`_build_openai_request()`, `_build_gemini_request()`) eliminate duplication.
- **Fine-tuned local models** -- roachie-8b-ft and roachie-nemo-ft are domain-specialized via LoRA, achieving 90% accuracy locally. This goes beyond the AI Agent pattern of using generic models.

### Cons

- **No planner-executor separation** -- The same LLM both plans (decides which tool to use) and executes (generates the command). A true agentic system would have a lightweight planner model that decomposes goals into sub-tasks, with a separate executor model that generates specific commands for each sub-task.
- **No model routing** -- Provider selection is manual (user chooses) or priority-based (auto-detect picks the first available). There's no learned routing that sends schema queries to Gemini (best at SQL) and troubleshooting queries to Claude (best at reasoning) based on historical performance.
- **No multimodal intelligence** -- The system processes only text. Voice input exists (`llm_voice.sh`) but only as speech-to-text transcription, not multimodal reasoning. An agentic system could process screenshots of dashboards, error logs from images, or architecture diagrams.

### Room to Improve

1. **Add a lightweight query classifier/planner** -- Before calling the main LLM, run a small classifier (even regex-based or a tiny local model) that categorizes the query as: `tool_lookup` (direct to tool), `informational` (skip LLM, return docs), `multi_step` (enable full agent loop), `sql_query` (SQL-specialized handling). This adds the planner layer without adding a full separate model.

2. **Implement learned provider routing** -- Track per-provider accuracy by query category in the metrics CSV. Route queries to the provider with the best historical accuracy for that category. Data already exists in `sql_audit/nl_batch_*.jsonl`.

3. **Add image input for error analysis** -- Accept screenshots of CockroachDB dashboard panels or error messages. Cloud providers (Claude, Gemini, GPT-4.1) all support vision. This would make roachie genuinely multimodal.

---

## 2. Architecture & Frameworks

**AI Agent pattern:** Single-agent, linear execution, task-based prompting.
**Agentic AI pattern:** Multi-agent systems, goal-driven workflows, planner-executor architecture.

### How Roachie Works

Roachie uses a **single-agent architecture** with an iterative execution loop:

```
User Query
  -> Tool Matching (regex + semantic)
  -> System Prompt Construction (schema + tools + history + persistent learning)
  -> LLM API Call
  -> Response Parsing (JSON extraction)
  -> Command Validation (6-layer pipeline)
  -> Command Execution
  -> [If needs_followup=true] Feed output back -> LLM API Call -> ...
  -> [If command failed + reflexion enabled] Self-correction -> LLM API Call -> ...
  -> Metrics Logging
```

The agent loop (`llm_assistant.sh:1381-1533`) supports up to 3 iterations (`_NL_MAX_AGENT_ITERATIONS`), with two follow-up triggers:
- **needs_followup** -- LLM explicitly requests follow-up to analyze command output
- **reflexion** -- System detects command failure and forces self-correction (lines 1456-1464)

### Pros

- **Beyond linear execution** -- The agent loop with reflexion is a genuine step beyond simple prompt -> response. The system can observe failure, analyze the error, and try a different approach -- a core agentic capability.
- **Tool-enriched prompting** -- The system dynamically constructs prompts based on query content, injecting relevant tool documentation, schema context, and persistent learning. This goes beyond static task-based prompting.
- **Multi-step execution within a single query** -- A single user request can trigger multiple commands (multi-command JSON responses), each validated and executed independently. The system tracks success/failure per command.
- **Batch execution mode** -- `roachie-batch` runs prompts programmatically without human interaction, a step toward autonomous operation.

### Cons

- **Single-agent, no specialization** -- One LLM handles everything: understanding intent, selecting tools, generating commands, analyzing output. A multi-agent system would have specialized agents: a Schema Agent (knows database structure), a Performance Agent (analyzes slow queries), a DDL Agent (generates schema changes), each with domain-specific prompts and tools.
- **No goal decomposition** -- The system handles one query at a time. It cannot decompose a high-level goal like "optimize the performance of the movr database" into sub-tasks: analyze slow queries -> identify missing indexes -> generate index recommendations -> validate with explain plans. The user must drive each step manually.
- **No inter-agent communication** -- Even within the agent loop iterations, there's no structured handoff between specialized roles. Each iteration uses the same system prompt (re-enriched, but structurally identical).
- **No framework abstraction** -- The architecture is implemented directly in bash rather than using an orchestration framework. While this is appropriate for a CLI tool, it limits extensibility and makes complex multi-agent patterns difficult to implement.

### Room to Improve

1. **Implement goal decomposition for complex queries** -- When the LLM detects a multi-step objective, have it output a `plan` field in the JSON response with ordered sub-tasks. Execute each sub-task sequentially, feeding results forward. This turns the flat agent loop into a plan-execute loop.

2. **Add specialized prompt profiles** -- Instead of one monolithic system prompt, create domain-specific profiles (schema inspection, performance analysis, cluster operations) that are selected based on the query classifier. Each profile emphasizes different tools and includes domain-specific few-shot examples.

3. **Consider a supervisor pattern** -- For complex multi-step operations (e.g., "migrate data from tenant A to tenant B"), add a supervisor layer that coordinates multiple tool executions, validates intermediate results, and decides whether to proceed or abort. This is achievable within the existing agent loop by extending the iteration logic.

---

## 3. Memory Systems

**AI Agent pattern:** Session memory, short-term embeddings, basic caches.
**Agentic AI pattern:** Long-term memory, episodic + semantic memory, knowledge graphs, vector databases.

### How Roachie Works

| Memory Type | Implementation | Persistence | File |
|------------|---------------|-------------|------|
| **Conversation history** | JSON message array (`_nl_messages_json`) | Session only | `llm_assistant.sh:1278` |
| **Session successes** | In-memory array (`recent_successes[]`) | Session only | `llm_assistant.sh:1047-1052` |
| **Session failures** | In-memory array (`recent_failures[]`) | Session only | `llm_assistant.sh:1069-1080` |
| **Persistent failures** | JSONL file (`NL_FAILURES_DB`) | Cross-session | `llm_metrics.sh:438-486` |
| **Persistent successes** | JSONL file (`NL_SUCCESSES_DB`) | Cross-session | `llm_metrics.sh:506+` |
| **Cached schema context** | Bash variable (`_cached_schema_context`) | Session only | `llm_prompt.sh` |
| **Cached topology** | Bash variable (`_cached_topology`) | Session only | `llm_prompt.sh` |
| **Cached embeddings** | JSON files on disk | Permanent | `tools/embeddings/` |
| **Metrics log** | CSV file (`NL_METRICS_CSV`) | Permanent | `llm_metrics.sh` |
| **Batch test results** | JSONL files (`sql_audit/nl_batch_*.jsonl`) | Permanent | `roachie-batch` |

### Pros

- **Persistent learning is genuinely agentic** -- The failure/success databases (`NL_FAILURES_DB`, `NL_SUCCESSES_DB`) persist across sessions and are injected into the system prompt (`_load_recent_failures()`, `_load_recent_successes()`). This means the system remembers its mistakes and successes across restarts -- a form of **episodic memory** that goes beyond simple session context.
- **Schema context as working memory** -- The system queries the actual database schema at session start (`_nl_fetch_schema_context()`) and caches it, giving the LLM accurate, real-time knowledge of the environment. This is superior to static knowledge.
- **Conversation history with truncation** -- History is maintained across turns with intelligent truncation (keeps last 10 turns, truncates within agent loop to 6 messages) to prevent context overflow while maintaining conversational coherence.
- **Pre-computed embeddings as semantic memory** -- The 77 tool descriptions and 14 doc chunks are pre-embedded and stored permanently. This is a form of static semantic memory.

### Cons

- **No episodic replay** -- While failures/successes are stored, they're injected as flat text ("these queries failed before"). A true episodic memory would store full interaction traces (query -> plan -> execution -> result -> feedback) and retrieve similar episodes for similar queries, providing richer context.
- **No knowledge graph** -- Tool relationships, database schema dependencies, and troubleshooting sequences are not modeled as a graph. For example, the system doesn't know that `cr_query_stats` findings often lead to `cr_plan` for detailed analysis, which often leads to `cr_indexes` for optimization. These workflow chains exist only in the LLM's training data.
- **No semantic search over history** -- Past conversations cannot be retrieved by semantic similarity. If a user asked the same question last week, the system starts fresh (unless the exact query was in the failure/success database).
- **Memory is capped and lossy** -- Persistent failures are capped at 15 entries, successes at 10. Conversation history at 10 turns. Once these limits are hit, older memories are discarded with no summarization or consolidation.
- **No user-specific memory** -- The system doesn't distinguish between different users. All memory is shared globally. An agentic system would maintain per-user preferences, frequently used tools, and common query patterns.

### Room to Improve

1. **Add workflow memory (tool chains)** -- Track sequences of tools used together successfully. When the user asks about query performance, automatically suggest the `cr_query_stats -> cr_plan -> cr_indexes` workflow based on historical patterns. Store these chains in a simple JSON file.

2. **Implement semantic history search** -- Embed past queries and store them with their results. When a new query arrives, retrieve the most similar past interaction and include it as a few-shot example. This would let the system say "A similar query last week was resolved with cr_query_stats."

3. **Add memory summarization** -- When the persistent failure database hits its cap, summarize common failure patterns into a compact summary rather than just dropping old entries. "cr_ddl_table often fails with -T flag" is more useful than 15 individual failure records.

4. **Add per-user profiles** -- Store user-specific preferences (preferred database, common tools, feedback patterns) in `~/.roachie/user_profile.json`. Inject relevant preferences into the system prompt.

---

## 4. Tool Usage & Actions

**AI Agent pattern:** Predefined tools, user-triggered function calling, REST APIs.
**Agentic AI pattern:** Autonomous tool selection, multi-step execution, structured tool registries, MCP.

### How Roachie Works

This is roachie's **strongest agentic dimension**:

| Capability | Implementation | Status |
|-----------|---------------|--------|
| Tool registry | 77 cr_* tools with curated descriptions (`tool_descriptions.txt`) | **Complete** |
| Autonomous tool selection | LLM chooses tools based on user intent, no explicit tool request needed | **Complete** |
| Multi-step tool execution | Agent loop with `needs_followup` chains multiple tool calls | **Complete** |
| Structured tool catalog | Pre-computed embeddings + regex keywords for 77 tools | **Complete** |
| Tool validation | 6-layer pipeline: whitelist -> normalize -> metachar -> SQL guard -> existence -> RBAC | **Complete** |
| MCP integration | 39 tools exposed via Google GenAI Toolbox (`tools/mcp/tools.yaml`) | **Complete** |
| Native tool calling | MCP schema conversion for providers that support function calling (`NL_NATIVE_TOOLS=1`) | **Complete** |
| Environment interaction | Tools execute against live CockroachDB clusters, read real state | **Complete** |

### Pros

- **Fully autonomous tool selection** -- The user says "show me slow queries" and the system autonomously selects `cr_query_stats`, generates correct flags (`-d movr -h localhost -p 26257`), and executes. No predefined mapping or user-triggered function call -- the LLM decides based on semantically matched tool documentation.
- **Structured tool registry with dual matching** -- 77 tools are cataloged with curated descriptions, pre-computed embeddings (3 providers), and regex keyword patterns. The combined matching achieves 99-100% accuracy across 135 test queries. This goes beyond simple function calling.
- **MCP integration** -- 39 tools are exposed via the Model Context Protocol (`tools/mcp/tools.yaml`), making them accessible to any MCP-compatible client. This is a standard agentic tool interface.
- **Safety-first execution** -- The 6-layer validation pipeline prevents dangerous operations (SQL mutations, shell metachar injection, unauthorized tools). RBAC gates tool access by role. This is critical for a DBA tool operating on production databases.
- **Tools interact with real environments** -- Unlike chatbot tools that return static data, roachie's tools execute against live CockroachDB clusters, read real metrics, and modify real state (with appropriate guards).

### Cons

- **No dynamic tool discovery** -- The tool set is fixed at 77 cr_* tools. The system cannot discover or install new tools at runtime. An agentic system could search for relevant tools, download them, and integrate them dynamically.
- **No tool composition/chaining by the system** -- While the LLM can generate multi-command responses, there's no formal tool composition language. The LLM can't say "pipe the output of cr_query_stats into cr_plan" -- it must generate two separate commands and manually extract the relevant query ID.
- **No tool feedback loop** -- When a tool fails, the system retries via reflexion, but it doesn't update the tool registry to note "this tool fails with this flag combination." The persistent failure database captures query-level failures, not tool-level metadata.
- **No tool versioning awareness** -- The system doesn't know that `cr_tables` behavior changed between CockroachDB v25.2 and v25.4. It relies on the tools directory symlink (`tools/current`) to point to the right version.

### Room to Improve

1. **Add tool output piping** -- Allow the LLM to express data flow between tools: "take the query_id from cr_query_stats output and pass it to cr_plan." Currently, the LLM must parse output and hardcode values in the second command. A simple `$PREV_OUTPUT` variable or structured output capture would enable this.

2. **Track per-tool success rates** -- Log which tools succeed/fail in the metrics CSV and use this to inform the LLM: "cr_ddl_table has a 70% success rate with the -T flag; use positional argument instead." This closes the tool feedback loop.

3. **Add tool dependency declarations** -- Annotate tools with prerequisites: "cr_plan requires a query_id, which can be obtained from cr_query_stats." This lets the system automatically chain dependent tools.

---

## 5. Knowledge & Retrieval

**AI Agent pattern:** Basic RAG, static retrieval, embedding search.
**Agentic AI pattern:** Adaptive RAG, context prioritization, hybrid search, GraphRAG, continuously updated knowledge.

### How Roachie Works

Roachie implements a **hybrid retrieval system** with three knowledge sources:

| Source | Type | Content | Retrieval Method |
|--------|------|---------|-----------------|
| Tool catalog | Structured | 77 tool descriptions + --help output | Regex keywords + cosine similarity (combined) |
| Doc RAG | Unstructured | 14 curated CockroachDB knowledge chunks | Cosine similarity (embedding search) |
| Live schema | Structured | Database names, tables, columns from cluster | SQL queries at session start |

### Pros

- **Hybrid search is genuinely advanced** -- The combined regex + semantic matching is a true hybrid retrieval system. Regex acts as an intent gate (high precision), semantic provides ranking (broad recall). This outperforms either approach alone, achieving 99-100% accuracy.
- **Context prioritization** -- Not all retrieved knowledge has equal weight. Tool --help output is prioritized (injected verbatim), doc chunks are secondary, and schema context is compact (15-line summary). The system manages a token budget to avoid overwhelming the LLM.
- **Live schema provides real-time knowledge** -- Unlike static RAG that retrieves from a fixed corpus, the schema context is queried from the actual database at session start. This means the system always has accurate, current knowledge about the user's environment.
- **Three-provider embedding redundancy** -- Ollama (local, 768-dim) -> OpenAI (API, 1536-dim) -> Gemini (API, 3072-dim) with automatic fallback to regex if no embedding model is available.

### Cons

- **Static document corpus** -- The 14 CockroachDB knowledge chunks are hand-curated and never updated automatically. When CockroachDB releases new features (e.g., v25.4 changefeeds), the knowledge base is stale until manually updated.
- **No GraphRAG** -- Tool relationships, database schema dependencies, and troubleshooting workflows are not modeled as a graph. A GraphRAG approach could answer "what tools are related to performance optimization?" by traversing a tool relationship graph rather than relying on embedding similarity.
- **No continuous knowledge updates** -- The system doesn't learn new facts from successful interactions. If the user teaches it "in our environment, the movr database is on tenant va," this knowledge is lost after the session.
- **No re-ranking** -- Retrieved tools are ranked by cosine similarity, but there's no cross-encoder re-ranking stage. A cross-encoder could assess (query, tool_description) pairs more accurately than bi-encoder similarity, reducing false positives.
- **No query-adaptive retrieval** -- The same retrieval pipeline is used for all queries. An adaptive system would use different strategies: keyword search for specific tool lookups, semantic search for conceptual queries, SQL-based retrieval for schema questions.

### Room to Improve

1. **Add a tool relationship graph** -- Model connections between tools as a simple JSON adjacency list: `{"cr_query_stats": ["cr_plan", "cr_indexes"], "cr_tables": ["cr_columns", "cr_ddl_table"]}`. When a tool is selected, also suggest related tools in the system prompt. This is lightweight GraphRAG.

2. **Implement knowledge base auto-refresh** -- Periodically (weekly?) scrape CockroachDB release notes and add new chunks to the doc RAG corpus. Re-compute embeddings automatically. This keeps the knowledge base current.

3. **Add cross-encoder re-ranking** -- After the initial cosine similarity retrieval, use a cross-encoder (e.g., a small model or API call) to re-rank the top-7 tools. This adds ~100ms but significantly improves precision for ambiguous queries.

4. **Implement query-adaptive retrieval strategy** -- Use the query classifier (from Dimension 1) to select the retrieval strategy: `tool_lookup` -> regex-first, `conceptual` -> semantic-first, `schema_question` -> SQL-first.

---

## 6. Orchestration & Workflows

**AI Agent pattern:** Sequential flows, simple backend automation, API chaining.
**Agentic AI pattern:** Orchestration engines, planning loops, event-driven workflows, reflection cycles.

### How Roachie Works

Roachie's orchestration is a **bounded iteration loop** with two feedback triggers:

```
while iteration < max_iterations (3):
    call LLM -> parse response -> validate -> execute commands
    if needs_followup == true:
        feed command output back as context -> continue
    elif command_failed AND reflexion_enabled:
        feed error + self-correction instructions -> continue
    else:
        break
```

Key orchestration files:
- Agent loop: `llm_assistant.sh:1381-1533`
- Reflexion trigger: `llm_assistant.sh:1456-1464`
- Follow-up context construction: `llm_assistant.sh:1478-1516`
- Re-enrichment per iteration: `llm_assistant.sh:1518-1532`

### Pros

- **Reflection cycles are implemented** -- The reflexion pattern (observe failure -> analyze error -> try different approach) is a core agentic capability. Roachie implements this cleanly: on command failure, it constructs a reflexion prompt with explicit self-correction instructions (lines 1493-1507) and re-enriches the system prompt with the relevant tool documentation for the retry.
- **Output-driven iteration** -- The `needs_followup` mechanism lets the LLM decide when it needs more information. This is event-driven (triggered by the LLM's assessment of its output) rather than purely sequential.
- **Re-enrichment per iteration** -- Each iteration re-runs tool matching on the combined query + reasoning, so the system prompt is updated with the most relevant tool documentation for the next step. This is context-adaptive orchestration.
- **Batch automation** -- `roachie-batch` automates the full pipeline: read prompts from file, execute each through the NL pipeline, report results as JSONL. This enables CI/CD integration and regression testing.

### Cons

- **No task planning** -- The system doesn't decompose complex objectives into sub-tasks with dependencies. Each iteration is reactive (respond to what happened) rather than proactive (plan what should happen next). There's no "Step 1: check indexes, Step 2: analyze query plans, Step 3: recommend changes" planning.
- **No parallel execution** -- Multi-command responses are executed sequentially. If the LLM generates three independent commands, they run one after another. An agentic orchestrator could parallelize independent tasks.
- **No event-driven triggers** -- The system only responds to user input. It cannot watch for events (e.g., "alert me when CPU exceeds 80%") or schedule recurring tasks (e.g., "check cluster health every hour"). All execution is user-initiated.
- **No workflow persistence** -- If the agent loop is interrupted (Ctrl-C, timeout), there's no checkpoint or resume capability. The entire interaction is lost. An agentic system would persist workflow state and allow resumption.
- **Bounded iteration limits flexibility** -- The hard cap of 3 iterations means complex multi-step analyses are truncated. Some legitimate workflows (e.g., investigating a performance issue across multiple tables) may require 5-10 steps.

### Room to Improve

1. **Add a plan-execute pattern** -- When the LLM detects a complex query, have it emit a `plan` field: `["check slow queries", "get explain plan for slowest", "suggest index"]`. Execute each step, feeding results forward. Track plan progress and allow the user to see/modify the plan.

2. **Add watch/monitor mode** -- Implement a simple event loop: `roachie-watch "cluster health" --interval 60s` that runs a query periodically and alerts on changes. This adds persistent, event-driven execution without requiring a full orchestration engine.

3. **Make iteration limit adaptive** -- Instead of a fixed max of 3, let the LLM request more iterations by setting `needs_more_iterations: true` in its response. Cap at a higher limit (e.g., 10) but let the system dynamically decide how many steps it needs.

4. **Add workflow checkpointing** -- Save agent loop state (current iteration, accumulated results, remaining plan) to a temp file. If interrupted, offer to resume from the last checkpoint on next startup.

---

## 7. Decision Making

**AI Agent pattern:** Reactive, prompt-driven, rule-assisted outputs.
**Agentic AI pattern:** Goal-oriented, planning, self-evaluation, iterative reasoning, ReAct/Plan-Execute.

### How Roachie Works

Roachie's decision making is a **hybrid of reactive and goal-oriented patterns**:

| Decision | Mechanism | Pattern |
|----------|-----------|---------|
| Tool selection | LLM reasoning + semantic matching + regex | Reactive (per-query) |
| Flag generation | LLM reasoning from tool --help docs | Reactive (per-query) |
| Follow-up decision | `needs_followup` field in LLM response | Semi-autonomous |
| Self-correction | Reflexion on command failure | Goal-oriented (retry toward success) |
| Provider fallback | Priority-based auto-detection | Rule-based |
| Command validation | 6-layer deterministic pipeline | Rule-based |
| RBAC gating | Role-based tool/command filtering | Rule-based |

### Pros

- **ReAct-like pattern** -- The agent loop follows a Reason-Act-Observe cycle: the LLM reasons about the query (reasoning field), acts (generates commands), observes (command output fed back), and re-reasons. This is structurally similar to the ReAct framework.
- **Self-evaluation via reflexion** -- When a command fails, the system doesn't just retry blindly. It constructs a specific self-correction prompt: "analyze what went wrong, check the error message, try a different approach, do NOT repeat the same command." This is explicit self-evaluation.
- **Deterministic safety layer** -- The 6-layer validation pipeline provides rule-based guardrails that the LLM cannot override. Even if the LLM "decides" to run a destructive command, the validation layer blocks it. This hybrid (neural decision + rule-based guard) is stronger than either alone.
- **RBAC-driven decision scoping** -- The system constrains the LLM's decision space based on the user's role. A `monitor` role user can only access health/monitoring tools; the LLM physically cannot select `cr_create_cluster` regardless of the prompt.

### Cons

- **No explicit planning** -- The LLM doesn't produce a plan before acting. It goes directly from understanding to action. A Plan-Execute pattern would first generate an explicit plan ("I need to check X, then Y, then Z"), let the user approve, then execute step-by-step.
- **No self-evaluation of output quality** -- After successful command execution, the system doesn't evaluate whether the output actually answers the user's question. It relies on the user's feedback (optional, often skipped) to assess quality. An agentic system would use self-evaluation: "Did I answer the question? Is there more information the user needs?"
- **No confidence scoring** -- The LLM doesn't express confidence in its tool selection. All recommendations are treated equally. A confidence score ("I'm 90% sure this is cr_query_stats, but it could be cr_statements") would enable fallback strategies.
- **No goal persistence** -- Each query is independent. The system doesn't maintain a goal across queries (e.g., "we're troubleshooting performance -- all subsequent queries should be interpreted in this context").

### Room to Improve

1. **Add confidence scoring to tool selection** -- Have the LLM output a `confidence` field (0-1) alongside each command. When confidence is low (<0.5), present alternatives to the user instead of executing. When high (>0.9), execute without confirmation.

2. **Implement self-evaluation** -- After command execution, ask the LLM: "Does this output answer the user's question? Score 1-5." If the score is low, trigger a follow-up. This closes the evaluation loop without requiring user feedback.

3. **Add goal context** -- Allow the user to set a session goal: `/goal optimize movr performance`. All subsequent queries are interpreted in this context, and the system can suggest next steps proactively.

---

## 8. Deployment

**AI Agent pattern:** Chatbots, copilots, API-based assistants, SaaS, serverless.
**Agentic AI pattern:** Autonomous platforms, digital workforce, persistent execution, Kubernetes, Ray Serve.

### How Roachie Works

| Deployment Mode | Entry Point | Interaction | Persistence |
|----------------|-------------|-------------|-------------|
| Interactive CLI (roachman) | `bin/roachman` | Menu-driven + NL | Session only |
| Standalone NL | `bin/roachie-nl` | REPL loop | Session only |
| Batch testing | `bin/roachie-batch` | File-driven, non-interactive | JSONL report output |
| MCP server | `tools/mcp/tools.yaml` via Toolbox | API-driven | Stateless |

### Pros

- **Multiple deployment modes** -- Interactive, standalone, batch, and MCP API cover a range of use cases. The batch mode enables CI/CD integration for regression testing.
- **MCP integration enables platform interop** -- Exposing 39 tools via MCP makes roachie's capabilities available to any MCP-compatible client (Claude Desktop, custom agents, other tools). This is a step toward platform-level integration.
- **Zero-dependency local operation** -- With Ollama, roachie runs entirely locally with no cloud APIs, no SaaS dependencies, and no data leakage. This is critical for air-gapped or security-sensitive DBA environments.

### Cons

- **No persistent execution** -- Roachie runs only while the terminal is open. There's no daemon mode, no background service, no persistent agent that monitors the database and alerts on issues. This is firmly in the "copilot" deployment pattern.
- **No API server** -- Roachie is CLI-only. There's no HTTP API, no WebSocket interface, no way for other systems to programmatically invoke NL queries. The MCP Toolbox exposes SQL-based tools but not the NL reasoning pipeline.
- **No containerized deployment** -- There's no Dockerfile, no Helm chart, no Kubernetes manifest. The system depends on the user's local environment (bash, jq, curl, cockroach CLI, optionally Ollama).
- **No multi-user support** -- The system is single-user, single-session. Multiple users cannot share a roachie instance. There's no authentication, no session isolation, no concurrent access.

### Room to Improve

1. **Add a daemon/watch mode** -- `roachie-watch --config watches.yaml` that runs persistent health checks and sends alerts (Slack, email, webhook) when thresholds are breached. This transforms roachie from a copilot into an autonomous monitoring agent.

2. **Add an HTTP API wrapper** -- A thin HTTP server (even a bash+socat or Python Flask wrapper) that accepts NL queries via POST and returns JSON responses. This enables integration with dashboards, Slack bots, and other automation tools.

3. **Create a Dockerfile** -- Package roachie with all dependencies (bash 5, jq, curl, cockroach CLI, Ollama) in a container. This enables Kubernetes deployment, multi-instance scaling, and reproducible environments.

---

## 9. Monitoring & Observability

**AI Agent pattern:** Basic logs, error tracking.
**Agentic AI pattern:** Deep analytics, response monitoring, system-level feedback loops, LangSmith-like dashboards.

### How Roachie Works

| Observable | Storage | Content | File |
|-----------|---------|---------|------|
| Per-query metrics | CSV (`NL_METRICS_CSV`) | Provider, model, tokens, cost, latency, accuracy, match method | `llm_metrics.sh:_log_nl()` |
| Batch test results | JSONL (`sql_audit/nl_batch_*.jsonl`) | Full query/response/execution detail per prompt | `roachie-batch` |
| Debug logs | File (`NL_DEBUG_LOG`) | Detailed trace of every pipeline stage | `llm_metrics.sh` |
| Session cost | Temp file | Running cost accumulator per session | `llm_metrics.sh:21` |
| Persistent failures | JSONL | Failed queries with error context | `llm_metrics.sh:438+` |
| Persistent successes | JSONL | Successful queries with commands | `llm_metrics.sh:506+` |

### Pros

- **Comprehensive per-query metrics** -- Every query logs: provider, model, prompt/completion tokens, cost (in cents), enrichment time, API time, execution time, query complexity, matched tools, match method, failure match, trace ID. This is rich enough for detailed analytics.
- **Cost tracking is built-in** -- Real-time cost display per query and per session, with model-specific pricing for 20+ models. This is critical for production use.
- **Batch testing as observability** -- Running the same 73+ prompts across providers creates a built-in benchmark. Accuracy regressions are immediately visible when comparing JSONL output files.
- **Debug logging at multiple levels** -- `NL_DEBUG=1/2/3` provides increasingly detailed tracing, from high-level decisions to raw API payloads. The 1MB size cap prevents log bloat.
- **PII masking** -- Sensitive output is masked before logging (`_mask_sensitive_output`), preventing credentials and connection strings from appearing in logs.

### Cons

- **No dashboard** -- All metrics are raw CSV/JSONL files. There's no visualization, no trend analysis, no alerting on metric changes. A user must manually parse files to understand system health.
- **No real-time monitoring** -- Metrics are write-only. There's no live stream of events, no WebSocket feed, no way to watch the system's behavior in real-time beyond the terminal output.
- **No automated anomaly detection** -- The system doesn't alert when accuracy drops, costs spike, or latency increases. All analysis is manual and after-the-fact.
- **No distributed tracing** -- While each query has a `_NL_TRACE_ID`, there's no correlation across the full pipeline (embedding call -> LLM call -> tool execution). If a query is slow, isolating the bottleneck requires manual log analysis.
- **No feedback loop from metrics to behavior** -- Metrics are logged but never used to adjust behavior. If the system notices that Gemini is 20% cheaper with the same accuracy, it doesn't automatically prefer Gemini. The data exists but isn't acted upon.

### Room to Improve

1. **Add a metrics summary command** -- `roachie-stats` that reads the CSV and reports: queries per day, average accuracy, cost by provider, latency percentiles, most common tools. Simple text output, no external dependencies.

2. **Add automated accuracy alerting** -- After each batch test, compare accuracy against the previous run. If any provider drops by more than 5%, output a warning. This is a simple diff-based feedback loop.

3. **Build a metrics-to-routing feedback loop** -- Use historical accuracy and cost data to automatically select the optimal provider for each query type. The data already exists in the CSV; it just needs to be consumed.

---

## 10. Learning & Improvement

**AI Agent pattern:** Manual tuning, prompt iteration, occasional fine-tuning.
**Agentic AI pattern:** Continuous feedback pipelines, performance adaptation, RL feedback, evaluation frameworks.

### How Roachie Works

| Learning Mechanism | Type | Frequency | Impact |
|-------------------|------|-----------|--------|
| Persistent failure injection | Prompt-time learning | Every query | Prevents repeating known mistakes |
| Persistent success injection | Prompt-time learning | Every query | Reinforces known-good patterns |
| User feedback collection | Data collection | Per-query (optional) | Stored but not used for training |
| LoRA fine-tuning | Weight-level learning | Manual (ad-hoc) | Permanent model improvement |
| Batch test regression | Evaluation | Manual (ad-hoc) | Detects accuracy changes |
| Training data sanitization | Data quality | At training time | Corrects known bad patterns |
| Prompt template iteration | Prompt engineering | Manual (developer-driven) | System prompt improvements |

### Pros

- **Multi-layered learning** -- Roachie learns at three levels: prompt injection (immediate, per-session), persistent databases (cross-session), and LoRA fine-tuning (permanent). This layered approach provides both short-term adaptation and long-term improvement.
- **Complete fine-tuning pipeline** -- `prepare_training_data.py` + `run_lora.sh` form an end-to-end pipeline: extract data from batch tests -> deduplicate -> sanitize -> split -> train LoRA adapters -> fuse -> convert -> deploy to Ollama. This is production-grade tooling.
- **Data quality controls** -- `sanitize_command()` fixes known bad patterns (e.g., `cr_ddl_table -T TABLE` -> positional arg) before they enter training data. `clean_command()` strips execution metadata. Deduplication by prompt prevents over-representation.
- **Evaluation framework exists** -- Batch testing against 73+ prompts with assertions provides a built-in evaluation suite. Multi-provider comparison shows relative quality.

### Cons

- **No continuous learning pipeline** -- Fine-tuning is manual: a developer runs `prepare_training_data.py` and then `run_lora.sh`. There's no automated pipeline that retrains when new data accumulates. The gap between data collection (continuous) and model training (ad-hoc) means the model is always behind.
- **Feedback data isn't used for training** -- Users provide y/n feedback, but this data never flows into LoRA training. It sits in JSONL files unused for weight updates. This is the biggest gap between AI Agent (logs feedback) and Agentic AI (uses feedback to improve).
- **No online learning** -- The system cannot adapt its behavior within a session based on real-time feedback. If the user corrects a tool choice, this correction helps for the next query (via session memory) but doesn't persist beyond the session unless the user gives explicit feedback.
- **No A/B testing** -- There's no mechanism to compare two model versions on live traffic. A new LoRA model replaces the old one completely. There's no gradual rollout, no canary testing, no shadow mode.
- **No reward signal from execution** -- Command success/failure is the richest signal available, but it's not formally connected to a reward function. An agentic system would use execution success rate as a reward signal for RL-based optimization.

### Room to Improve

1. **Automate the fine-tuning pipeline** -- Add a cron job or post-batch-test hook that checks if enough new data has accumulated (e.g., 50+ new successful examples since last training), runs `prepare_training_data.py`, trains a new LoRA adapter, and deploys it -- all automatically. Include a regression test before deployment.

2. **Connect feedback to training** -- Use positive feedback as training examples and negative feedback as DPO rejection examples. The data pipeline already exists; it just needs a `--include-feedback` flag in `prepare_training_data.py`.

3. **Add A/B testing** -- Deploy two Ollama models simultaneously (e.g., `roachie-8b-ft` and `roachie-8b-ft-v2`). Run batch tests against both. Only promote the new model if it matches or exceeds the old one's accuracy. This prevents regressions.

4. **Implement execution-as-reward** -- Track per-tool, per-flag success rates. Weight training examples by their execution success rate. Examples that always succeed get higher weight; examples that sometimes fail get lower weight or are filtered out.

---

## Summary Scorecard

| # | Dimension | AI Agent Traits | Agentic AI Traits | Roachie Has | Score |
|---|-----------|----------------|-------------------|-------------|-------|
| 1 | Intelligence Models | Single LLM | Multi-model, planner-executor | Multi-model (embed + generate), 5 providers, local+cloud | **7/10** |
| 2 | Architecture | Single-agent, linear | Multi-agent, orchestrated | Single-agent with iterative loop, reflexion | **5/10** |
| 3 | Memory | Session only | Long-term, episodic, semantic | Session + persistent failures/successes, cached schema | **6/10** |
| 4 | Tool Usage | Predefined, user-triggered | Autonomous, structured registry, MCP | 77-tool registry, autonomous selection, MCP, 6-layer validation | **8/10** |
| 5 | Knowledge & Retrieval | Basic RAG | Adaptive, hybrid, GraphRAG | Hybrid regex+semantic, Doc RAG, live schema | **7/10** |
| 6 | Orchestration | Sequential | Planning loops, event-driven | Bounded agent loop, reflexion, re-enrichment | **6/10** |
| 7 | Decision Making | Reactive | Goal-oriented, self-evaluating | ReAct-like loop, reflexion, RBAC guards | **7/10** |
| 8 | Deployment | Chatbot/copilot | Autonomous platform | CLI + batch + MCP, no daemon/API | **4/10** |
| 9 | Monitoring | Basic logs | Deep analytics, feedback loops | Rich CSV/JSONL metrics, cost tracking, PII masking | **7/10** |
| 10 | Learning | Manual tuning | Continuous pipelines | LoRA pipeline, persistent learning, but manual triggers | **6/10** |

### Overall: 6.3/10 -- Semi-Agentic System

---

## The Key Insight

Roachie is **not an AI Agent** anymore -- it has outgrown that pattern in tool usage, knowledge retrieval, decision making, and monitoring. But it's **not fully Agentic AI** either -- it lacks goal persistence, autonomous orchestration, continuous learning, and persistent deployment.

The most accurate description is a **semi-agentic specialist**: a system that exhibits agentic behavior (autonomous tool selection, self-correction, multi-step execution, persistent memory) within a bounded, user-triggered scope. It "executes objectives" within a single query or agent loop, but it doesn't autonomously pursue long-running goals.

### What Makes Roachie Agentic (already strong):
- **Autonomous tool selection** -- The system chooses tools without being told which one to use
- **Self-correction** -- Reflexion observes failure, analyzes errors, and retries with a different approach
- **Persistent memory** -- Failures and successes persist across sessions and influence future behavior
- **Structured tool registry** -- 77 tools with embeddings, descriptions, and MCP integration
- **Hybrid retrieval** -- Combined regex + semantic matching with live schema context

### What Keeps Roachie as an Agent (upgrade opportunities):
- **No goal persistence** -- Each query starts fresh; no persistent objectives
- **No autonomous execution** -- Everything is user-triggered; no watching, no scheduling
- **No planning** -- No explicit goal decomposition or step-by-step plans
- **No continuous learning** -- Training is manual; feedback doesn't flow to weights
- **No API/daemon mode** -- CLI-only; no persistent service, no multi-user

### Top 5 Upgrades to Move Toward Agentic AI (by impact):

1. **Add goal decomposition + plan-execute** (Architecture + Decision Making + Orchestration) -- When the LLM detects a multi-step objective, output a plan, execute steps sequentially, and track progress. This single change elevates three dimensions simultaneously.

2. **Add daemon/watch mode** (Deployment + Orchestration) -- `roachie-watch` that persistently monitors database health and alerts on anomalies. Transforms roachie from a copilot into an autonomous monitoring agent.

3. **Automate the learning pipeline** (Learning) -- Connect batch testing -> training data extraction -> LoRA training -> deployment into an automated pipeline triggered by data accumulation thresholds.

4. **Implement learned provider routing** (Intelligence + Decision Making) -- Use historical accuracy data to automatically route queries to the optimal provider for each query type.

5. **Add workflow memory and tool chaining** (Memory + Tool Usage) -- Remember successful tool sequences and suggest them for similar queries. Enable structured data flow between tool executions.

---

## CLI Summary (Quick Reference)

### Position: Semi-Agentic System (6.3/10)

| # | Dimension | Score | Key Strength | Key Gap |
|---|-----------|-------|--------------|---------|
| 1 | Intelligence Models | **7/10** | Multi-model (embed + generate), hybrid local/cloud | No planner-executor separation, no model routing |
| 2 | Architecture | **5/10** | Iterative agent loop with reflexion | Single-agent, no goal decomposition |
| 3 | Memory | **6/10** | Persistent failure/success DBs, cached schema | No episodic replay, no knowledge graph, no user profiles |
| 4 | Tool Usage | **8/10** | 77-tool registry, autonomous selection, MCP, 6-layer validation | No dynamic discovery, no tool composition |
| 5 | Knowledge & Retrieval | **7/10** | Hybrid regex+semantic, Doc RAG, live schema | Static corpus, no GraphRAG, no cross-encoder re-ranking |
| 6 | Orchestration | **6/10** | Reflexion cycles, re-enrichment per iteration | No planning loops, no parallel execution, no event triggers |
| 7 | Decision Making | **7/10** | ReAct-like loop, self-correction, RBAC | No explicit planning, no confidence scoring |
| 8 | Deployment | **4/10** | CLI + batch + MCP, zero-dependency local | No daemon, no API, no container, no multi-user |
| 9 | Monitoring | **7/10** | Rich metrics (CSV/JSONL), cost tracking, PII masking | No dashboard, no anomaly detection, no metrics-to-routing |
| 10 | Learning | **6/10** | Complete LoRA pipeline, persistent learning DBs | Manual triggers, feedback not used for training |

### The Key Insight

Roachie is a **semi-agentic specialist** -- it exhibits agentic behavior (autonomous tool selection, self-correction, persistent memory, structured tool registry) within a bounded, user-triggered scope. It "executes objectives" per-query but doesn't pursue persistent goals.

### What Makes It Agentic Already
- Autonomous tool selection from 77-tool registry (no user specification needed)
- Self-correction via reflexion (observe failure -> analyze -> retry differently)
- Persistent memory across sessions (failures/successes influence future behavior)
- Hybrid retrieval (regex + semantic + live schema)
- MCP integration for platform interop

### What Keeps It Agent-Level
- No goal persistence (each query is independent)
- No autonomous execution (everything user-triggered)
- No planning (no goal decomposition, no step-by-step plans)
- No continuous learning (training is manual, feedback disconnected from weights)
- No daemon/API mode (CLI-only, no persistent service)

### Top 5 Upgrades by Impact
1. **Goal decomposition + plan-execute pattern** -- elevates Architecture + Decision Making + Orchestration simultaneously
2. **Daemon/watch mode** (`roachie-watch`) -- transforms copilot into autonomous monitoring agent
3. **Automated learning pipeline** -- connect batch testing -> training -> deployment
4. **Learned provider routing** -- auto-select optimal provider per query type from historical data
5. **Workflow memory + tool chaining** -- remember successful tool sequences, enable structured data flow
