# Ollama/Llama Chunking Strategies for Limited Context Windows

**Date:** 2026-03-19
**Status:** Planned (not yet implemented)
**Context:** Llama open-source models have ~4-5K token context limits. The roachie Modelfiles set `num_ctx 4096` for all three roachie variants (roachie, roachie-8b, roachie-3b).

---

## Current Token Budget (Ollama Compact Prompt)

| Component | Tokens | Notes |
|-----------|--------|-------|
| Compact system prompt | ~440 | Already optimized vs full (~2,700) |
| Tool help (5 tools) | ~1,575 | Biggest chunk -- avg 315 tokens/tool |
| Connection reminder | ~50 | Mandatory flags reminder |
| Personality + learning | ~162 | Minimal for Ollama |
| User query | ~25 | Typical single query |
| **Total input** | **~2,250** | |
| **Remaining for output** | **~1,840** | Out of 4096 context |

### Where It Breaks

1. **Reflexion loop** -- a follow-up iteration adds ~1,000 tokens of command output, leaving only ~840 for the response. A second follow-up would overflow.
2. **Conversation history** -- any multi-turn history pushes past the budget fast.
3. **Learning data** -- if 15 failures + 10 successes are loaded, that could add 500-1,000 tokens.
4. **5K context** -- would give ~2,850 tokens for output, comfortable for single-turn but still tight for reflexion.

### Actual Tool Help Sizes (measured)

| Tool | Chars | ~Tokens |
|------|-------|---------|
| cr_tables | 941 | 235 |
| cr_query_stats | 1,376 | 344 |
| cr_health | 1,241 | 310 |
| cr_size | 1,394 | 348 |
| cr_columns | 1,351 | 338 |
| **Average** | **1,261** | **315** |

---

## Proposed Chunking Strategies

### Strategy 1: Tool Help Chunking (highest impact, ~1,000 tokens saved)

Instead of injecting all 5 matched tool `--help` outputs at full length, inject only the **top 1-2** most relevant tools' full help, and provide just tool names for the rest.

```
# Current (5 tools x ~315 tokens = ~1,575 tokens):
TOOL: cr_tables
<full --help output>
TOOL: cr_size
<full --help output>
...3 more...

# Proposed (2 full + 3 names = ~630 + 30 = ~660 tokens):
TOOL: cr_tables
<full --help output>
TOOL: cr_size
<full --help output>
Also relevant: cr_db_size, cr_db_tables_rowcount, cr_get
```

**Implementation:** In `_enrich_system_prompt_with_tool_help()`, when `llm_provider == ollama-*`, only fetch `--help` for top 2 tools (by regex score), list remaining by name only.

### Strategy 2: Truncate Tool Help (moderate impact, ~500 tokens saved)

Cap each tool's `--help` output to the first 10-15 lines (usage line + key flags). Skip examples and verbose descriptions.

```bash
# In _nl_get_tool_help, for Ollama:
help_output=$("$tools_dir/$tool" --help 2>&1 | head -15)
```

Most tools have a usage line + flag list that fits in ~150 tokens (vs 315 full).

**Implementation:** Add a `max_lines` parameter to `_nl_get_tool_help()` or truncate in the enrichment function when provider is Ollama.

### Strategy 3: Skip Learning Data for Ollama (~200-500 tokens saved)

Persistent failures/successes are valuable for cloud models with large context windows, but they consume precious tokens on Ollama. Skip injecting learning memory for Ollama providers.

**Implementation:** In `_nl_build_prompt()`, skip the learning data section when `llm_provider == ollama-*`.

### Strategy 4: Reduce Reflexion Output for Ollama (~750 tokens saved)

Currently `_NL_OUTPUT_TRUNCATE_CHARS` is 4000 (~1,000 tokens). For Ollama, this should be ~500-800 chars (~125-200 tokens) to leave room for the response.

**Implementation:** In `_nl_process_query()`, when building follow-up context for Ollama, use a smaller truncation limit:
```bash
local _truncate_chars="$_NL_OUTPUT_TRUNCATE_CHARS"
[[ "$llm_provider" == ollama-* ]] && _truncate_chars=800
```

### Strategy 5: Skip Doc RAG for Ollama (~350 tokens saved per chunk)

Doc RAG chunks consume ~350 tokens each (up to 3 chunks = ~1,050 tokens). With 4K context, doc RAG should be disabled for Ollama.

**Implementation:** Already partially handled -- doc RAG requires an embedding provider, and Ollama's embedding is separate from the LLM. But if embeddings are available, doc RAG still runs. Add explicit skip:
```bash
# In _enrich_system_prompt_with_tool_help:
if [[ "$llm_provider" == ollama-* ]]; then
  # Skip doc RAG for Ollama -- context too limited
  doc_context=""
fi
```

### Strategy 6: Dynamic Context Budget (most robust, catches all edge cases)

Measure the prompt size before calling the API, and progressively drop sections until it fits within the context budget.

**Drop order (least to most important):**
1. Doc RAG chunks
2. Learning data (persistent failures/successes)
3. Extra tool help (keep top 2, drop rest)
4. Conversation history (keep only last turn)
5. Tool help truncation (head -10)

```bash
_enforce_ollama_budget() {
  local prompt="$1" max_tokens="${2:-4096}"
  local reserve=512  # minimum tokens for output
  local budget=$(( max_tokens - reserve ))
  local prompt_tokens=$(( ${#prompt} / 4 ))

  if [[ $prompt_tokens -le $budget ]]; then
    echo "$prompt"
    return
  fi

  # Progressive truncation...
  # 1. Strip doc RAG section
  # 2. Strip learning data
  # 3. Truncate tool help to top 2
  # 4. Drop conversation history
  # 5. Truncate remaining tool help
}
```

**Implementation:** Add as a final pass in `_nl_process_query()` before the Ollama API call.

---

## Recommended Implementation Order

| Priority | Strategy | Tokens Saved | Complexity |
|----------|----------|-------------|------------|
| 1 | Strategy 1: Tool help chunking | ~1,000 | Low |
| 2 | Strategy 4: Reduce reflexion output | ~750 | Low |
| 3 | Strategy 5: Skip doc RAG | ~350-1,050 | Low |
| 4 | Strategy 2: Truncate tool help | ~500 | Low |
| 5 | Strategy 3: Skip learning data | ~200-500 | Low |
| 6 | Strategy 6: Dynamic budget | All edge cases | Medium |

Strategies 1-5 are simple conditionals on `llm_provider == ollama-*`. Strategy 6 is a general-purpose budget enforcer that handles all edge cases.

**Combined effect of strategies 1-5:** Reduces Ollama input from ~2,250 tokens to ~800-1,000 tokens, leaving ~3,000 tokens for output and reflexion -- comfortable even within 4K context.

---

## Provider-Agnostic Budget Management (Recommended Approach)

The chunking strategies above were originally framed as Ollama-specific, but the underlying problem -- **no budget awareness** -- affects all providers. The system currently assembles prompts and sends them without checking whether they fit or whether they're cost-efficient.

### Context Headroom by Provider

| Provider | Context | Current Prompt | Headroom | Real Concern |
|----------|---------|---------------|----------|-------------|
| Ollama/Llama | 4K | ~2,250 | Tight, breaks on reflexion | **Overflow** |
| OpenAI GPT-4.1 | 128K | ~10,750 | Fine | Cost |
| Anthropic Claude | 200K | ~10,750 | Fine | Cost |
| Gemini 2.5 Flash | 1M | ~10,750 | Fine | Minimal |
| Gemini 2.5 Pro | 1M | ~10,750 | Fine | Minimal |

For cloud providers, "fine" doesn't mean "optimal." Sending 10,750 tokens of system prompt when only 3,000 are relevant wastes money. At Sonnet pricing (300 cents/1M input tokens), every query costs ~0.3 cents just for the system prompt. Over hundreds of queries, that adds up.

### Gemini Model Context Windows

| Model | Context Window | Notes |
|-------|---------------|-------|
| Gemini 2.5 Pro | 1M tokens | 2M token preview tier available |
| Gemini 2.5 Flash | 1M tokens | Same as Pro, faster/cheaper |
| Gemini 2.0 Flash | 1M tokens | |
| Gemini 1.5 Pro | 1M (default), 2M (extended) | 2M available via API flag |
| Gemini 1.5 Flash | 1M tokens | |

Context budget is effectively a non-issue for Gemini -- even the full prompt with enrichment, doc RAG, learning data, and multi-turn history is <1.1% of the context window. The concern is purely cost, not overflow.

### Recommended: Single Budget-Aware Prompt Assembler

Instead of hardcoding `ollama-*` checks in strategies 1-5, implement **Strategy 6 as the primary approach** -- a provider-agnostic budget enforcer:

1. Query the context limit from `_llm_context_limit()` (already exists in `llm_config.sh`)
2. Reserve tokens for output (e.g., `_NL_MAX_OUTPUT_TOKENS`)
3. Assemble the prompt in priority order, stopping when the budget is reached
4. Drop sections in the same order for all providers:
   - Doc RAG chunks (lowest priority)
   - Learning data (persistent failures/successes)
   - Extra tool help (keep top 2, drop rest)
   - Conversation history (keep only last turn)
   - Tool help truncation (head -10)

For Ollama (4K), most optional sections get dropped. For Claude/GPT (128-200K), everything fits. For Gemini (1M), everything fits trivially. Same code path, different behavior based on the provider's capacity.

This aligns with the deferred v9-R2 recommendation (token budget management) -- not Ollama-specific, but a general architectural improvement that benefits all providers by being cost-aware and overflow-safe.

---

## Modelfile Context Configuration

Current Modelfiles (`tools/utils/Modelfile.roachie*`) all use:
```
PARAMETER num_ctx 4096
```

If Llama models support 5K+ context reliably, bumping to 5120 or 6144 would provide additional headroom without requiring chunking changes. However, chunking strategies are still recommended for robustness.
