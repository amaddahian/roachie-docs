# Roachie: Agentic AI Litmus Test

**Date:** 2026-03-25
**Scope:** Apply the hard "NOT Agentic vs Truly Agentic" framework to roachie -- is it a sophisticated chatbot/RAG/RPA system, or does it cross the line into genuine agentic behavior?

---

## The Framework: A Hard Line

This framework draws a clear boundary. Three things are explicitly **NOT** Agentic AI:

| Pattern | Definition | Why Not Agentic |
|---------|-----------|-----------------|
| **LLM Chatbot** | Query -> Prompt -> LLM -> Output | No planning, no autonomy, no task management |
| **RPA + LLM** | Automated scripts that trigger tools/workflows | Predefined logic, not autonomous reasoning |
| **RAG System** | Retrieval-augmented single-step responses | Single-step response system, no iteration |

Five capabilities define **truly Agentic AI**:

| Capability | Definition |
|-----------|-----------|
| **Planning** | Breaking complex goals into actionable tasks |
| **Memory** | Maintaining context across steps and interactions |
| **Tool Usage** | Dynamically selecting tools to solve problems |
| **Feedback Loops** | Evaluating outputs and refining actions |
| **Task Orchestration** | Coordinating multiple agents and workflows |

---

## The Honest Question: Which Category Is Roachie?

Let's test roachie against each NOT-agentic pattern first, then against each agentic capability.

---

## Part 1: Is Roachie One of the NOT-Agentic Patterns?

### Test 1: Is Roachie an LLM Chatbot?

**Definition:** Query -> Prompt -> LLM -> Output. No planning, no autonomy, no task management.

**Roachie's flow:**
```
Query
  → Regex + Semantic Matching (retrieval)
  → System Prompt Construction (enrichment)
  → LLM API Call
  → JSON Response Parsing
  → 6-Layer Command Validation
  → Command Execution (against live database)
  → [If needs_followup] Feed output back → LLM → Execute → ...
  → [If command failed] Reflexion → LLM → Retry differently → ...
  → Metrics Logging + Persistent Learning
```

**Verdict: Not a chatbot.** The critical differences:

1. **Output is execution, not text** -- A chatbot returns text. Roachie validates and executes commands against live CockroachDB clusters. The "output" is a database operation, not a message.

2. **Multi-step iteration** -- The agent loop (max 3 iterations) with `needs_followup` means the system can observe results and continue acting. A chatbot is single-shot.

3. **Self-correction** -- Reflexion detects command failure, analyzes the error, and tries a different approach. A chatbot returns the same wrong answer regardless of whether it worked.

4. **Persistent learning** -- Failures and successes are stored across sessions and influence future behavior. A chatbot has no memory beyond the conversation.

**However:** The base interaction pattern (user asks, system responds) IS chatbot-like. Without the user initiating each query, the system does nothing. It doesn't autonomously monitor, schedule, or act.

**Score: 70% beyond chatbot.** Execution + iteration + self-correction lift it past the chatbot pattern, but user-triggered-only operation keeps one foot in chatbot territory.

---

### Test 2: Is Roachie RPA + LLM?

**Definition:** Automated scripts that trigger tools/workflows following predefined logic rather than autonomous reasoning.

**Evidence FOR RPA + LLM:**
- The 77 cr_* tools are bash scripts with fixed logic
- `regex_keywords.tsv` maps keywords to tools via predefined patterns
- The 6-layer validation pipeline is entirely rule-based
- `roachie-batch` runs a predefined list of prompts automatically
- Flag normalization (`sed` transformations) follows hardcoded rules
- RBAC is a static role-to-permission mapping

**Evidence AGAINST RPA + LLM:**
- Tool selection is not predefined -- the LLM reasons about which tool to use based on the query, schema context, and available documentation
- The LLM generates command flags dynamically, not from a lookup table
- Reflexion adapts behavior based on runtime errors, not predefined error handlers
- The system handles novel queries it has never seen before (not just predefined workflows)
- Multi-step reasoning chains emerge from LLM output, not from scripted sequences

**Verdict: Not pure RPA + LLM, but has RPA-like components.** The tool validation pipeline is RPA (rules-based automation), but the tool selection and flag generation are LLM-driven (autonomous reasoning). The architecture is a hybrid: autonomous reasoning (LLM) operating within safety rails (RPA-like validation).

**Score: 65% beyond RPA + LLM.** The LLM does genuine reasoning for tool selection and flag generation, but the execution pipeline has significant predefined logic.

---

### Test 3: Is Roachie a RAG System?

**Definition:** Great for improving factual responses through retrieval and embeddings, but operates as a single-step response system.

**Evidence FOR being a RAG system:**
- Hybrid regex + semantic retrieval feeds tool documentation into the prompt
- Doc RAG retrieves CockroachDB knowledge chunks via cosine similarity
- Pre-computed embeddings stored in JSON files
- Live schema context retrieved via SQL queries

**Evidence AGAINST being just a RAG system:**
- RAG is one component, not the system -- retrieval feeds into execution, not just text generation
- The agent loop makes it multi-step, not single-step
- Reflexion makes the system evaluate and refine its outputs
- Persistent learning means the system improves over time
- Tool execution makes the output an action, not a response

**Verdict: Roachie uses RAG, but is not a RAG system.** RAG is the retrieval layer; the system continues with execution, evaluation, and learning. The framework says RAG systems "operate as a single-step response system" -- roachie's agent loop, reflexion, and persistent learning are multi-step by design.

**Score: 80% beyond RAG.** RAG is clearly a component, not the architecture. The multi-step execution and learning elevate it.

---

## Part 2: Does Roachie Have the Five Agentic Capabilities?

### Capability 1: Planning -- Breaking complex goals into actionable tasks

| Aspect | What True Agentic AI Does | What Roachie Does | Gap |
|--------|--------------------------|-------------------|-----|
| Goal decomposition | "Optimize database" → [analyze queries, identify missing indexes, generate recommendations, apply changes] | No decomposition -- each query is independent | **Large** |
| Task ordering | Plan step dependencies, execute in order | Multi-command responses executed sequentially, but no explicit plan | **Large** |
| Plan visibility | Show the plan to the user before execution | No plan shown -- goes directly from understanding to action | **Large** |
| Plan adaptation | Revise plan based on intermediate results | Reflexion retries on failure, but doesn't revise a plan | **Medium** |
| Proactive planning | Anticipate what the user needs next | No anticipation -- waits for user input | **Large** |

#### Pros

- **Multi-command responses are implicit plans** -- When the LLM generates a JSON response with 3 commands (`cr_query_stats`, then `cr_plan`, then `cr_index_advisor`), this is an implicit plan executed sequentially. The system supports this natively.
- **Reflexion is adaptive re-planning** -- When a command fails, the reflexion prompt says "analyze what went wrong, try a different approach." This is a limited form of plan revision: the original "plan" (command) failed, so the system generates an alternative.
- **needs_followup enables multi-step execution** -- The LLM can say "I need to see this output before deciding what to do next," which is a form of conditional planning: execute step 1, observe, decide step 2.

#### Cons

- **No explicit plan generation** -- The LLM never outputs a structured plan like `["Step 1: check slow queries", "Step 2: get explain plan", "Step 3: suggest index"]`. It goes directly from query to command. The user never sees the plan and can't modify it.
- **No goal persistence across queries** -- Each query starts fresh. There's no concept of "we're working toward optimizing the movr database" that persists across multiple user queries and guides each response.
- **No proactive task generation** -- The system never suggests "Based on what I've seen, you should also check X." It waits for the user to ask.
- **Max 3 iterations limits complex plans** -- Even if the LLM wanted to execute a 7-step plan, the agent loop caps at 3 iterations. Complex goals are truncated.

#### Room to Improve

1. **Add explicit plan output** -- When the LLM detects a complex query, require it to output a `plan` field:
   ```json
   {
     "plan": ["Check slow queries with cr_query_stats", "Analyze worst query with cr_plan", "Suggest indexes with cr_index_advisor"],
     "current_step": 1,
     "commands": [{"command": "cr_query_stats -d movr ...", "description": "Step 1"}]
   }
   ```
   Display the plan to the user, execute one step at a time, and track progress.

2. **Add goal context** -- `/goal optimize movr performance` sets a session-level objective. All subsequent queries are interpreted in this context, and the system suggests next steps after each result.

3. **Increase iteration limit for planned workflows** -- When a plan is present, allow up to 10 iterations instead of 3. The plan provides bounded scope; the iteration limit can be relaxed.

---

### Capability 2: Memory -- Maintaining context across steps and interactions

| Aspect | What True Agentic AI Does | What Roachie Does | Gap |
|--------|--------------------------|-------------------|-----|
| Short-term (working) | Full context of current task | Conversation history (10 turns), cached schema | **Small** |
| Cross-step memory | Results from step 1 available in step 3 | Agent loop carries output forward; truncated to 4K chars | **Small** |
| Cross-session memory | Remember past interactions | Persistent failure/success DBs (15/10 entries, 30-day TTL) | **Medium** |
| Episodic memory | Retrieve similar past situations | No episodic retrieval -- flat JSONL, no similarity search | **Large** |
| Semantic memory | Organized knowledge about the domain | Pre-computed embeddings, tool descriptions | **Medium** |
| User memory | Remember user preferences and patterns | No user profiles, no personalization | **Large** |

#### Pros

- **Cross-step memory within agent loop** -- Command output from iteration 1 is fed back as context for iteration 2. The LLM can reference previous results when deciding the next action. This maintains context across steps -- a core agentic capability.
- **Persistent learning is genuine cross-session memory** -- The failure/success databases survive restarts. A failure in session N is available to prevent the same mistake in session N+1. This is agentic: the system learns from experience.
- **Sticky context preservation** -- When conversation history is truncated to fit the context window, the system extracts and preserves key context (cluster name, database, tenant, host) from dropped messages (`llm_assistant.sh:581-591`). This is intelligent memory management, not just FIFO eviction.
- **Schema context is environmental memory** -- The live schema query gives the system accurate knowledge of its operating environment. This is superior to static memory -- it's real-time awareness.

#### Cons

- **Memory is shallow** -- 15 failures and 10 successes is not enough for meaningful learning. A system that processes 50 queries per day would overflow this memory in 2-3 days.
- **No episodic retrieval** -- Past interactions are stored flat (JSONL) without similarity-based retrieval. The system can't say "last time someone asked about slow queries on movr, we found missing indexes." It either has the exact query in its failure DB or starts fresh.
- **No user profiles** -- All users share the same memory. The system can't learn that User A primarily works with the movr database while User B focuses on cluster health.
- **No memory summarization** -- When entries are evicted from the persistent databases, they're simply dropped. No summarization ("cr_ddl_table commonly fails with -T flag") preserves the knowledge.

#### Room to Improve

1. **Expand persistent memory** -- Increase from 15/10 entries to 100/50 with summarization on overflow. When the 101st failure is stored, summarize the oldest 50 into 5 pattern summaries.
2. **Add user profiles** -- Store per-user preferences in `~/.roachie/users/<username>.json`: preferred database, common queries, feedback patterns. Inject relevant preferences into the system prompt.
3. **Add episodic retrieval** -- Embed past queries when storing them. On new queries, retrieve the most similar past interaction and its outcome as a few-shot example.

---

### Capability 3: Tool Usage -- Dynamically selecting tools to solve problems

| Aspect | What True Agentic AI Does | What Roachie Does | Gap |
|--------|--------------------------|-------------------|-----|
| Dynamic selection | Choose tools based on reasoning | LLM selects from 77 tools via enriched prompt | **None** |
| Tool discovery | Find and learn new tools | Fixed 77-tool registry; no runtime discovery | **Medium** |
| Tool composition | Chain tools with data flow | Sequential execution, no structured piping | **Medium** |
| Tool creation | Generate new tools on the fly | No tool generation | **Large** |
| Environmental interaction | Act on and observe the real world | Execute against live CockroachDB clusters | **None** |

#### Pros

- **Dynamic tool selection is genuinely agentic** -- The LLM receives 7 relevant tool docs (via hybrid retrieval) and autonomously decides which tool to use and what flags to pass. It's not choosing from a menu -- it's reasoning about the user's intent, matching it to a tool, and generating the correct invocation. This is the defining characteristic of agentic tool usage.
- **77-tool registry with structured metadata** -- Each tool has a curated description, pre-computed embeddings, regex keyword mappings, and `--help` documentation. This is a structured tool registry, not just a list of functions.
- **MCP integration** -- 39 tools exposed via Model Context Protocol. Native tool calling schema conversion for Anthropic, OpenAI, and Gemini. This is standard agentic tool infrastructure.
- **Environmental interaction is real** -- Tools execute against live CockroachDB clusters. The system reads real metrics, queries real data, and observes real results. This is not simulated -- it's genuine environment interaction.
- **6-layer safety validation** -- Tools are validated before execution (whitelist, normalize, metachar, SQL mutation, existence, RBAC). This is responsible agentic tool usage -- autonomy within guardrails.

#### Cons

- **No tool creation** -- The system can't generate new tools or scripts at runtime. If no existing tool handles the user's need, it can't write a SQL query or bash script to fill the gap (though `cockroach sql -e "..."` provides some flexibility).
- **No tool composition** -- The system can't express "take the output of cr_query_stats, extract the slowest query ID, and pass it to cr_plan." Each command is independent. Data flow between tools is done by the LLM rewriting the query, not by a structured piping mechanism.
- **No tool learning** -- The system doesn't learn new tool usage patterns. If cr_query_stats works better with `--format json` for a particular query type, this isn't captured.

#### Room to Improve

1. **Add SQL generation as a tool** -- When no cr_* tool matches, allow the LLM to generate and execute read-only SQL queries directly. This is "tool creation" within a safe boundary (read-only, validated).
2. **Add structured output capture** -- When a tool executes, capture its output in a structured variable that the next tool invocation can reference: `cr_plan --query-id $PREV_QUERY_ID`.

---

### Capability 4: Feedback Loops -- Evaluating outputs and refining actions

| Aspect | What True Agentic AI Does | What Roachie Does | Gap |
|--------|--------------------------|-------------------|-----|
| Output evaluation | Assess if output meets the goal | Checks command exit code (pass/fail) | **Medium** |
| Self-correction | Fix errors and retry | Reflexion analyzes failure, tries different approach | **Small** |
| Quality assessment | Score output quality (not just pass/fail) | No quality scoring beyond pass/fail | **Large** |
| User feedback integration | Learn from user corrections | Collects y/n feedback, stores in persistent DB | **Medium** |
| Continuous improvement | Get better over time automatically | Persistent learning + LoRA fine-tuning (manual) | **Medium** |

#### Pros

- **Reflexion is a genuine feedback loop** -- This is roachie's strongest agentic feature. When a command fails:
  1. The system **observes** the failure (exit code + error output)
  2. It **evaluates** what went wrong (constructs a reflexion prompt with explicit analysis instructions)
  3. It **refines** its action (generates a different command, told "do NOT repeat the same command")
  4. It **executes** the refined action

  This observe-evaluate-refine-execute cycle is textbook agentic feedback. (`llm_assistant.sh:1455-1507`)

- **needs_followup enables output-driven iteration** -- The LLM can say "I need to see the output of this command before deciding what to do next." The output is fed back (truncated to 4000 chars, sanitized against injection), and the LLM reasons about it. This is evaluation-driven execution.

- **User feedback persists and influences behavior** -- Positive feedback saves the pattern to the success DB; negative feedback saves to the failure DB. Both are injected into future prompts. The system literally learns from corrections.

- **Batch testing as systematic evaluation** -- `roachie-batch` runs 73+ prompts with assertions and reports pass/fail per prompt. This is a systematic evaluation pipeline, not ad-hoc testing.

#### Cons

- **Pass/fail is too coarse** -- The system only knows if a command succeeded (exit code 0) or failed. It doesn't evaluate whether the output actually answers the user's question. A command can succeed but be the wrong command for the intent.

- **No self-evaluation of relevance** -- After successful execution, the system doesn't ask "Did this actually answer what the user wanted?" It assumes success = relevance.

- **Feedback-to-training loop is manual** -- User feedback is collected and stored, but it only flows into the persistent learning database (prompt injection). It doesn't flow into LoRA training automatically. The learning loop is partially closed (prompt injection) but not fully closed (weight updates).

- **No comparative evaluation** -- The system doesn't compare its response to alternative approaches. It doesn't say "I chose cr_tables but cr_db_tables_rowcount might have been better for this query."

#### Room to Improve

1. **Add self-evaluation** -- After command execution, ask the LLM: "Does this output answer the user's original question? Score 1-5." If score < 3, trigger a follow-up automatically. This closes the quality evaluation gap.

2. **Close the feedback-to-training loop** -- Automate: accumulated feedback (50+ new examples) -> `prepare_training_data.py` -> LoRA training -> model deployment. This makes improvement continuous, not manual.

3. **Add comparative tool evaluation** -- When the system selects a tool, also note the second-best match. If the primary tool fails, try the alternative. Log which was better. This produces preference data for DPO training.

---

### Capability 5: Task Orchestration -- Coordinating multiple agents and workflows

| Aspect | What True Agentic AI Does | What Roachie Does | Gap |
|--------|--------------------------|-------------------|-----|
| Multi-agent coordination | Specialized agents collaborate | Single agent handles everything | **Large** |
| Central orchestrator | Manages agent assignments and results | `_nl_process_query()` is a single-pipeline orchestrator | **Large** |
| Parallel execution | Multiple agents work simultaneously | Sequential execution only | **Large** |
| Progress tracking | Track and report multi-step progress | `multi-step(N)` complexity label in metrics | **Medium** |
| Agent handoff | Routing between specialized agents | No specialization, no handoff | **Large** |

#### Pros

- **The orchestrator exists and works** -- `_nl_process_query()` manages the full pipeline: enrichment → LLM call → validation → execution → follow-up → metrics → feedback. It's a single-pipeline orchestrator, but it is an orchestrator.
- **Agent loop enables multi-step coordination** -- The loop coordinates LLM calls, tool executions, and output analysis across up to 3 iterations. Each iteration can re-enrich the system prompt with new tool documentation based on the previous step's reasoning.
- **Batch mode coordinates across prompts** -- `roachie-batch` orchestrates sequential execution of 73+ prompts with rate limiting, assertion checking, and aggregated reporting. This is workflow-level orchestration.
- **Metrics aggregate across iterations** -- Token counts, costs, and execution times are accumulated across all agent loop iterations and reported as unified metrics. Progress is tracked.

#### Cons

- **No multi-agent architecture** -- This is roachie's largest agentic gap. There are no specialized agents (no schema agent, no performance agent, no security agent). One LLM handles all query types with one system prompt.
- **No parallel execution** -- Multi-command responses are executed one at a time. Independent commands can't run simultaneously. The LLM can't spawn background tasks.
- **No agent handoff** -- There's no mechanism to say "this is a schema question, hand it to the schema specialist." All queries go through the same pipeline.
- **No inter-agent communication** -- Even the framework's basic requirement of agents sharing task information is absent. There's one agent, so there's nothing to share with.
- **No persistent workflow state** -- If a multi-step workflow is interrupted, there's no checkpoint, no resume, no state file. All progress is lost.

#### Room to Improve

1. **Add prompt-profile-based specialization** -- Create specialized prompt profiles:
   - `schema_specialist`: emphasizes schema tools, includes DDL few-shot examples
   - `performance_specialist`: emphasizes performance tools, includes optimization examples
   - `security_specialist`: emphasizes access control tools, includes audit examples

   The query classifier routes to the appropriate profile. This is "multi-agent via prompt specialization" -- lighter than true multi-agent but captures the benefit.

2. **Add parallel command execution** -- When the LLM generates multiple independent commands, execute them as background jobs and collect results. Use `wait` to synchronize. This is achievable in bash.

3. **Add workflow checkpointing** -- Save agent loop state (iteration, accumulated results, plan) to a session file. Offer `/resume` on next startup to continue an interrupted workflow.

---

## Part 3: The Evolution Assessment

The framework describes a three-stage evolution:

```
Prompt-based AI → Workflow AI → Autonomous Agent Systems
```

### Where Is Roachie on This Spectrum?

| Stage | Characteristics | Roachie's Position |
|-------|----------------|-------------------|
| **Prompt-based AI** | Query → Prompt → LLM → Output | **Past this** -- output is execution, not text |
| **Workflow AI** | Multi-step execution with tools, retrieval, validation | **Here** -- agent loop, reflexion, tool pipeline |
| **Autonomous Agent Systems** | Planning, multi-agent, self-improving, persistent | **Approaching** -- has reflexion and persistent learning, lacks planning and multi-agent |

**Verdict: Roachie is solidly in Workflow AI, reaching toward Autonomous Agent Systems.**

It has crossed the line from Prompt-based AI through:
- Tool execution (not just text generation)
- Multi-step agent loop (not single-shot)
- Reflexion (not static responses)
- Persistent learning (not stateless)
- Hybrid retrieval (not just RAG)

It has NOT yet reached Autonomous Agent Systems because:
- No explicit planning or goal decomposition
- No multi-agent coordination or specialization
- No autonomous scheduling or monitoring
- No continuous self-improvement (manual fine-tuning only)
- No persistent workflow state

---

## Summary Scorecard

### "NOT Agentic" Test Results

| Pattern | Is Roachie This? | Evidence | Verdict |
|---------|-----------------|----------|---------|
| LLM Chatbot | **30% yes** | User-triggered, single entry point | Output is execution + multi-step + self-correction elevate it |
| RPA + LLM | **35% yes** | Validation pipeline is rule-based | Tool selection and flag generation use autonomous LLM reasoning |
| RAG System | **20% yes** | Uses RAG for retrieval | RAG is one component; execution + iteration + learning make it more |

### Agentic Capability Scores

| Capability | Score | Strongest Feature | Biggest Gap |
|-----------|-------|-------------------|-------------|
| Planning | **3/10** | Multi-command implicit plans | No explicit planning, no goal decomposition |
| Memory | **6/10** | Persistent failure/success DBs, sticky context | Shallow (15/10 entries), no episodic retrieval, no user profiles |
| Tool Usage | **9/10** | 77 tools, dynamic selection, MCP, 6-layer validation | No tool creation, no structured composition |
| Feedback Loops | **7/10** | Reflexion, needs_followup, persistent learning | Pass/fail only, no quality scoring, manual fine-tuning |
| Task Orchestration | **3/10** | Single-pipeline orchestrator with bounded loop | No multi-agent, no parallel execution, no agent handoff |

### Overall Agentic Score: 5.6/10

---

## The Key Insight

Roachie is **NOT a chatbot, NOT pure RPA, and NOT just a RAG system** -- it has crossed beyond all three NOT-agentic patterns. But it has crossed **unevenly**:

**Strongly agentic** in two capabilities:
- **Tool Usage** (9/10) -- Autonomous tool selection from a 77-tool registry with MCP integration and 6-layer safety validation. This is production-grade agentic tool usage.
- **Feedback Loops** (7/10) -- Reflexion provides genuine observe-evaluate-refine-execute cycles. Persistent learning carries corrections across sessions.

**Weakly agentic** in three capabilities:
- **Planning** (3/10) -- No goal decomposition, no explicit plans, no proactive task generation. The system reacts to user queries rather than pursuing objectives.
- **Task Orchestration** (3/10) -- Single agent, single pipeline, sequential execution. No specialization, no coordination, no parallel work.
- **Memory** (6/10) -- Cross-session persistence exists but is shallow. No episodic retrieval, no user profiles, no memory summarization.

### The Bottleneck

**Planning and Orchestration are the bottleneck to full agentic status.** Tool usage and feedback loops are already production-grade. Memory is functional but shallow. But without planning (breaking complex goals into steps) and orchestration (coordinating specialized execution), the system remains a sophisticated single-step tool invoker that can self-correct -- powerful, but not autonomous.

### Evolution Position

```
Prompt-based AI ──────── Workflow AI ──────── Autonomous Agent Systems
                              ▲
                              │
                         ROACHIE IS HERE
                         (strong tools + feedback,
                          weak planning + orchestration)
```

### Top 5 Upgrades to Cross Into Autonomous Agent Territory

1. **Explicit plan generation** (Planning) -- Have the LLM output a `plan` field for complex queries. Display the plan, execute step-by-step, track progress. Single highest-impact change: it enables goal decomposition and visible multi-step reasoning.

2. **Prompt-profile specialization** (Orchestration) -- Create domain-specific prompt profiles (schema, performance, security, cluster ops). Route queries to the best profile via classifier. This is multi-agent behavior via prompt specialization, achievable without a framework rewrite.

3. **Self-evaluation after execution** (Feedback Loops) -- After successful command execution, ask the LLM to score whether the output answers the user's question (1-5). If low, trigger follow-up. Closes the quality gap between pass/fail and relevance assessment.

4. **Episodic memory with similarity retrieval** (Memory) -- Embed and store past interactions. On new queries, retrieve the most similar past interaction as a few-shot example. The system learns from experience, not just from failures.

5. **Automated fine-tuning pipeline** (Feedback Loops + continuous improvement) -- Connect data accumulation → training data extraction → LoRA training → deployment into an automated pipeline. Triggered by thresholds (e.g., 50 new examples since last training). This makes improvement continuous, not manual.

---

## CLI Summary (Quick Reference)

### Agentic AI Litmus Test: 5.6/10 -- "Workflow AI, Reaching for Autonomous"

### Is Roachie One of the NOT-Agentic Patterns?

| Pattern | Verdict |
|---------|---------|
| LLM Chatbot? | **No** -- output is execution, multi-step, self-corrects |
| RPA + LLM? | **No** -- tool selection uses LLM reasoning, not predefined logic |
| RAG System? | **No** -- RAG is one component; execution + iteration + learning make it more |

### Does Roachie Have the Five Agentic Capabilities?

| Capability | Score | Strongest Feature | Biggest Gap |
|-----------|-------|-------------------|-------------|
| Planning | **3/10** | Multi-command implicit plans | No goal decomposition, no explicit plans |
| Memory | **6/10** | Persistent failure/success DBs | Shallow, no episodic retrieval, no user profiles |
| Tool Usage | **9/10** | 77 tools, dynamic selection, MCP, validation | No tool creation or structured composition |
| Feedback Loops | **7/10** | Reflexion + persistent learning | Pass/fail only, no quality scoring |
| Task Orchestration | **3/10** | Single-pipeline orchestrator | No multi-agent, no parallel, no handoff |

### The Key Insight

Roachie has crossed beyond all three NOT-agentic patterns but **unevenly**: Tool Usage (9/10) and Feedback Loops (7/10) are production-grade agentic capabilities. Planning (3/10) and Task Orchestration (3/10) are the bottleneck to full agentic status. Without planning and orchestration, the system remains a sophisticated self-correcting tool invoker -- powerful, but not autonomous.

### Evolution Position

```
Prompt-based AI ──── Workflow AI ──── Autonomous Agent Systems
                          ▲
                     ROACHIE IS HERE
```

### Top 5 Upgrades to Cross Into Autonomous Territory

1. **Explicit plan generation** -- `plan` field in LLM output for complex queries
2. **Prompt-profile specialization** -- domain-specific profiles as lightweight multi-agent
3. **Self-evaluation after execution** -- LLM scores whether output answers the question
4. **Episodic memory with retrieval** -- embed past interactions, retrieve similar ones
5. **Automated fine-tuning pipeline** -- continuous data → training → deployment loop
