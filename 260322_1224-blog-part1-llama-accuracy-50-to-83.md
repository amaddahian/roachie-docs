# Making a Local LLM Actually Useful: How We Took Llama 8B from 50% to 83% Accuracy for CLI Tool Selection

*Part 1 of 2 — Prompt engineering, semantic matching, and the surprising power of giving a small model the same context as a large one.*

---

We built a natural language interface for a CockroachDB toolkit — 77 bash CLI tools that DBAs use daily. Type "show me replication lag on the VA tenant" and the system picks the right tool, constructs the right flags, and runs it.

Cloud LLMs (Gemini, GPT-5, Claude) hit 95-100% accuracy almost immediately. But we wanted a local option. Privacy-sensitive environments, air-gapped networks, and the simple appeal of not paying per-token for every DBA query.

So we tried Llama 3.1 8B running locally via Ollama.

It started at roughly 50-60% accuracy. One out of every two commands was wrong.

This is the story of how we got it to 83% — without changing the model weights at all. Pure infrastructure work. Part 2 covers the LoRA fine-tuning that pushed it to 90%.

## The Architecture

The system works like this:

1. **User types a natural language query** ("list all tables in the movr database")
2. **Tool matching** finds the most relevant tools from 77 options (regex + semantic embeddings)
3. **Prompt construction** builds a system prompt with tool documentation, cluster topology, schema context, and rules
4. **LLM generates** a JSON response with the tool name and flags
5. **Validation** checks the command against a 5-layer security pipeline
6. **Execution** runs the command safely (array-based, no shell interpretation)

Every component in this chain affects accuracy. When we started diagnosing Llama's failures, we found problems at almost every layer.

## Phase 1: Semantic Matching (50-60% → 72%)

The first version used pure regex matching to find relevant tools. Given a query like "show me the backup status," it would scan for keywords like "backup" and "status" in tool names and descriptions. This worked surprisingly well — about 97% accuracy for tool *selection*.

But regex has blind spots. "How much storage am I using?" doesn't contain the word "storage" in the tool name (`cr_ceph_storage`). "What's the health of my cluster?" could match half a dozen tools.

We added **semantic matching** using embedding models:

- Pre-computed embeddings for all 77 tool descriptions (768-dim for Ollama, 1536-dim for OpenAI, 3072-dim for Gemini)
- At query time, embed the user's query and compute cosine similarity against all tools
- Combine semantic results with regex: semantic gets priority, regex supplements, deduplicated, top 7

The two-model architecture — a small embedding model for matching, the LLM for reasoning — was key. The embedding model runs in milliseconds and doesn't consume the LLM's context window.

We tested across 135 queries (71 multi-tenant + 64 single-tenant):

| Provider | Multi-Tenant | Single-Tenant | Combined |
|----------|-------------|---------------|----------|
| Gemini (3072-dim) | 100% | 98.4% | 100% |
| OpenAI (1536-dim) | 98.5% | 96.8% | 100% |
| Ollama (768-dim) | 97.1% | 93.7% | 99.3% |
| Regex only | 100% | 96.8% | — |

The combined approach (semantic + regex) hit 99-100% across all providers. Tool selection was essentially solved.

But Llama was still only at about 72% overall accuracy. It was finding the right tools but generating wrong flags, missing required parameters, or hallucinating options that didn't exist.

## Phase 2: The Compact Prompt Problem (72% → 83%)

This is where things got interesting. We had been treating Llama differently from cloud models throughout the entire pipeline, and every special case was making it worse.

### Discovery: Three Hidden Code Paths

When we compared the Ollama code path against Gemini's in our batch test runner, we found three Ollama-specific branches:

**1. Compact prompt (line 639 of roachie-batch)**

```bash
# What we had:
if [[ "$llm_provider" == ollama-* ]]; then
  system_prompt_base="$(_nl_build_system_prompt_compact \
    "$cluster_topology" "$tools_dir" "$schema_context")"
else
  system_prompt_base="$(_nl_build_system_prompt \
    "$cluster_topology" "$tools_dir" "$schema_context")"
fi
```

The "compact" prompt was a stripped-down version designed to fit Llama's smaller context window. It excluded 7 template files that cloud models received: parameter rules, tool notes, execution rules, few-shot examples, hallucination warnings, and more.

The logic seemed sound — 8B models have limited context, so give them less to process. But it was wrong. Those templates contained critical information: which flags are required, which tools share similar names, what the correct parameter format is. Without them, Llama was guessing.

**2. Enrichment cap (lines 831-839)**

```bash
local _enrich_max="$_NL_MAX_ENRICHED_TOOLS"
[[ "$llm_provider" == ollama-* ]] && _enrich_max=3
```

Cloud models got 7 tools' `--help` output injected into the prompt. Ollama got 3. This meant that for queries involving tools ranked 4th-7th in relevance, Llama had no documentation at all. It was forced to rely on the tool name alone.

**3. Hardcoded connection reminder**

```bash
if [[ "$llm_provider" == ollama-* ]]; then
  system_prompt+="REMINDER: EVERY cr_* command MUST include..."
else
  system_prompt+="$(_nl_connection_reminder)"
fi
```

Ollama got a simplified, hardcoded reminder instead of the dynamic one that included actual cluster details. Minor, but it contributed to missing `-h` and `-p` flags.

### The Key Insight: Tokenizer Efficiency

Why did the compact prompt exist in the first place? Because someone (reasonably) assumed that Llama's 16K context window couldn't handle the full prompt.

But we measured the actual token counts:

| Content | Gemini Tokens | Ollama Tokens |
|---------|--------------|---------------|
| Full system prompt | ~12,000 | ~4,000 |
| 7 enriched tools | ~3,500 | ~1,200 |
| **Total** | **~15,500** | **~5,200** |

Llama's tokenizer is roughly 3x more efficient than Gemini's for our content (bash commands, flag names, technical documentation). The full prompt fit comfortably within 16K tokens. The compact prompt was solving a problem that didn't exist.

### The Fix: Treat Every Model the Same

```bash
# After: one line, all providers
system_prompt_base="$(_nl_build_system_prompt \
  "$cluster_topology" "$tools_dir" "$schema_context")"
```

We also found and fixed Modelfile issues:

| Setting | Before | After | Why |
|---------|--------|-------|-----|
| SYSTEM prompt | 200-token embedded prompt | Removed | Conflicted with the runtime system prompt — the model received contradictory instructions |
| num_predict | 256 | 512 | Responses were being truncated mid-JSON, especially for tools with many flags |
| temperature | 0.1 | 0.15 | Slightly higher temperature improved tool selection diversity without hurting precision |

### Results

After removing all Ollama-specific code paths and fixing the Modelfile:

- **Before**: 52-53/73 prompts passing (72%)
- **After**: 60-61/73 prompts passing (83%)
- **Change**: +11 points from pure infrastructure fixes

No model changes. No fine-tuning. No additional training data. Just giving the small model the same information that the large model was receiving.

## Lessons Learned

**1. Don't handicap small models with less context.** The instinct to "simplify for smaller models" is often counterproductive. The templates we stripped out were the ones the small model needed *most* — it couldn't infer correct flag names from tool names alone the way a 400B model could.

**2. Measure before optimizing.** The compact prompt existed because of an assumption about token limits. Measuring actual token counts showed the full prompt was well within budget. An afternoon of measurement would have saved weeks of debugging accuracy issues.

**3. Provider-specific code paths are a maintenance trap.** Every `if ollama` branch was a place where Ollama's behavior diverged from the tested path. When we fixed bugs in the Gemini path, Ollama didn't benefit. Unifying the code paths meant one set of tests, one set of fixes.

**4. Semantic matching is table stakes.** Pure regex matching works until it doesn't. The combined approach (semantic + regex) was cheap to implement, fast at runtime (embeddings are pre-computed), and eliminated an entire class of tool-selection failures.

**5. The 8B ceiling is real.** Even with perfect context, 83% was the limit. The remaining failures were complex multi-parameter tools, positional arguments, and tool disambiguation — tasks that require reasoning capacity beyond what 8B parameters provide. To go further, we needed to change the model itself.

That's what Part 2 covers: LoRA fine-tuning on Apple Silicon, using cloud model outputs as training data, and the surprisingly tricky pipeline from MLX to Ollama.

---

*Part 2: [LoRA Fine-Tuning Llama 8B on Apple Silicon: From 83% to 90% Using Cloud Model Outputs as Training Data]()*

*The tools and scripts referenced in this post are part of [Roachie](https://github.com/amaddahian/roachie), an open-source CockroachDB DBA toolkit.*
