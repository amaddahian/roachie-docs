# AI Architecture Review v13

**Date:** 2026-03-19
**Reviewer:** Claude Opus 4.6
**Scope:** Full AI subsystem review — providers, prompt construction, security, metrics, chunking
**Previous:** v12 (13/15 fixed, 2 deferred) — all prior P1/P2 items resolved

---

## Architecture Scorecard

| Category | Score | Notes |
|----------|-------|-------|
| Security Pipeline | 9.5/10 | 5-step validation, array-based exec, RBAC, sanitization — mature |
| Provider Plugins | 8/10 | Streaming works well, but heavy duplication in request construction |
| Prompt Construction | 9/10 | Ollama chunking just landed, schema-aware, combined matching |
| Token Efficiency | 8.5/10 | Chunking strategies effective, minor waste in decorative headers |
| Error Handling | 9/10 | Rate limiting, retry with backoff, connection drop detection |
| Code Quality | 7.5/10 | Provider plugins have significant duplication (4x in OpenAI) |
| Test Coverage | 8.5/10 | 37 suites pass, 113 security tests, but no chunking-specific tests |
| Documentation | 8/10 | One config constant documented with wrong default |

**Overall: 8.5/10** (up from 8.3 in v12)

---

## Recommendations

### Table 1: Summary

| # | Priority | Category | Description | File(s) |
|---|----------|----------|-------------|---------|
| R1 | P1 | Documentation | CLAUDE.md documents wrong default for `NL_OUTPUT_TRUNCATE_CHARS` | CLAUDE.md |
| R2 | P1 | Code Quality | OpenAI request construction duplicated 4x — extract shared builder | providers/openai.sh |
| R3 | P1 | Code Quality | Gemini request construction duplicated 4x — extract shared builder | providers/gemini.sh |
| R4 | P2 | Maintenance | Anthropic API version hardcoded as 2023-06-01 — make configurable | providers/anthropic.sh |
| R5 | P1 | Robustness | `needs_followup` regex heuristic unreliable for native tool call responses | llm_providers.sh |
| R6 | P2 | Token Efficiency | Ollama prompt section headers still include decorative Unicode bars | llm_prompt.sh |
| R7 | P2 | Robustness | Vertex AI has no streaming — no progress indicator for users | llm_providers.sh |
| R8 | P2 | Feature Gap | Ollama provider ignores `tools_json` — no native tool calling path | providers/ollama.sh |
| R9 | P2 | Robustness | Streaming `trap ... RETURN` may not clean up headers on SIGINT | providers/*.sh |
| R10 | P2 | Testing | No dedicated unit tests for Ollama chunking strategies | tests/unit/ |

---

### R1 (P1, Documentation): CLAUDE.md documents wrong default for NL_OUTPUT_TRUNCATE_CHARS

**Issue:** CLAUDE.md documents `NL_OUTPUT_TRUNCATE_CHARS` default as `2000`, but the actual default in `llm_config.sh:16` is `4000`.

```
# CLAUDE.md says:
| `NL_OUTPUT_TRUNCATE_CHARS` | `2000` | Max chars of command output for follow-up context |

# llm_config.sh says:
readonly _NL_OUTPUT_TRUNCATE_CHARS="${NL_OUTPUT_TRUNCATE_CHARS:-4000}"
```

**Impact:** Users relying on CLAUDE.md for configuration will have wrong expectations. The Ollama chunking strategy 4 reduces this to 800 for Ollama, making the base value even more important to document correctly.

**Fix:** Update CLAUDE.md to show `4000` as the default.

---

### R2 (P1, Code Quality): OpenAI request construction duplicated 4x

**Issue:** `providers/openai.sh` has 4 nearly-identical request JSON construction blocks for the 2x2 matrix of `{streaming, non-streaming} x {with tools, without tools}`. Additionally, within each, there's a `{with temperature, without temperature}` fork, creating 8 code paths total. Anthropic solved this with a shared `_build_anthropic_request()` function.

**Location:** `providers/openai.sh:10-112` (non-streaming) and `providers/openai.sh:118-189` (streaming)

**Fix:** Extract `_build_openai_request()` like Anthropic's pattern:

```bash
_build_openai_request() {
  local system_prompt="$1" messages_json="$2" temperature="$3"
  local tools_json="${4:-}" stream="${5:-false}"
  local model="${OPENAI_MODEL:-gpt-4.1}"

  local token_limit_field="max_tokens"
  local include_temp="true"
  [[ "$model" == gpt-5* || "$model" == o3* || "$model" == o4* ]] && {
    token_limit_field="max_completion_tokens"; include_temp="false"
  }

  local all_messages
  all_messages=$(jq -nc --arg system "$system_prompt" --argjson turns "$messages_json" \
    '[{role: "system", content: $system}] + $turns')

  # Build with jq — one path handles all combinations
  local base_args=(--arg model "$model" --arg tlf "$token_limit_field"
    --argjson messages "$all_messages" --argjson maxtok "$_NL_MAX_OUTPUT_TOKENS")
  [[ "$include_temp" == "true" ]] && base_args+=(--argjson temp "$temperature")
  # ... etc
}
```

**Impact:** Reduces ~180 lines to ~60 lines. Prevents drift between streaming and non-streaming paths.

---

### R3 (P1, Code Quality): Gemini request construction duplicated 4x

**Issue:** Same pattern as R2. `providers/gemini.sh` has 4 nearly-identical request JSON blocks: `{streaming, non-streaming} x {with tools, without tools}`. Lines 11-78 (non-streaming) are almost identical to lines 84-143 (streaming).

**Fix:** Extract `_build_gemini_request()`:

```bash
_build_gemini_request() {
  local system_prompt="$1" messages_json="$2" temperature="$3" tools_json="${4:-}"
  local gemini_contents
  gemini_contents=$(printf '%s' "$messages_json" | jq '[.[] | {role: (if .role == "assistant" then "model" else .role end), parts: [{text: .content}]}]')

  local base_config
  base_config=$(jq -nc --argjson temp "$temperature" --argjson gmaxtok "$_NL_GEMINI_MAX_TOKENS" \
    --argjson gthink "$_NL_GEMINI_THINKING_BUDGET" \
    '{temperature: $temp, maxOutputTokens: $gmaxtok, responseMimeType: "application/json", thinkingConfig: {thinkingBudget: $gthink}}')

  # Single jq call with optional tools
  # ...
}
```

**Impact:** Reduces ~130 lines to ~40 lines.

---

### R4 (P2, Maintenance): Anthropic API version hardcoded

**Issue:** The Anthropic API version is hardcoded as `anthropic-version: 2023-06-01` in both `_call_anthropic_api` and `_call_anthropic_api_stream`. This version is nearly 3 years old. While it still works, newer API features may require a version bump.

**Location:** `providers/anthropic.sh:54,82`

**Fix:** Make configurable via env var:

```bash
local _api_version="${ANTHROPIC_API_VERSION:-2023-06-01}"
```

Also update the Vertex version string (`vertex-2023-10-16` in `providers/vertex.sh:72`):

```bash
local _api_version="${VERTEX_ANTHROPIC_VERSION:-vertex-2023-10-16}"
```

---

### R5 (P1, Robustness): `needs_followup` unreliable for native tool calls

**Issue:** When using native tool calling (`NL_NATIVE_TOOLS=1`), `_convert_tool_call_to_json` in `llm_providers.sh` detects `needs_followup` via regex on the text content:

```jq
needs_followup: (($text | test("needs_followup|follow.?up|next step|then I.ll"; "i")) // false)
```

This has two problems:
1. **False positives**: Text containing "next step" in an informational context triggers follow-up
2. **Missing true positives**: If the LLM's text block doesn't mention follow-up but the tool call itself implies it (e.g., calling a diagnostic tool that requires analysis of output), the follow-up won't trigger

**Impact:** Agent loop may iterate unnecessarily or miss needed follow-ups when native tool calling is enabled.

**Fix:** When native tool calls are present, always set `needs_followup: true` (the tool output needs to be fed back for the LLM to interpret). Only set false when the LLM explicitly produces text with no tool calls:

```jq
needs_followup: (if (.content[]? | select(.type == "tool_use") | length > 0) then true
  else (($text | test("needs_followup"; "i")) // false) end)
```

---

### R6 (P2, Token Efficiency): Ollama prompt headers waste tokens

**Issue:** After implementing Ollama chunking strategies 1-5, the enriched prompt still includes decorative Unicode section bars:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
RELEVANT TOOL DOCUMENTATION (3 tools detected via regex)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 TOOL: cr_health
────────────────────────────────────────────────────────────────────────────
...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 CRITICAL: Use EXACT flag names...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Each decorative line is ~20 tokens. With 5 separator lines = ~100 wasted tokens on Ollama's 4K budget.

**Location:** `llm_prompt.sh:472-534`

**Fix:** For Ollama, use plain text section headers:

```bash
if [[ "$_is_ollama" == "true" ]]; then
  enriched_prompt+="
TOOLS (${#unique_tools[@]} matched):
"
else
  enriched_prompt+="
━━━━...
RELEVANT TOOL DOCUMENTATION (${#unique_tools[@]} tools detected via ${match_method})
━━━━...
"
fi
```

---

### R7 (P2, Robustness): Vertex AI has no progress indicator

**Issue:** All cloud providers except Vertex AI use streaming with progress dots during generation. Vertex uses `rawPredict` (single response), so users see no feedback during long API calls (up to 60s timeout). This can make the system appear frozen.

**Location:** `llm_providers.sh:753-757`

**Fix:** Show a "waiting..." message before the Vertex call:

```bash
vertex-claude)
  _printc "$YELLOW_BOLD" "Waiting for Vertex AI response..." >&2
  api_response=$(_call_vertex_claude_api ...)
```

---

### R8 (P2, Feature Gap): Ollama ignores native tool calling

**Issue:** `_call_ollama_api` in `providers/ollama.sh` doesn't accept or pass the `tools_json` parameter. When `NL_NATIVE_TOOLS=1` is set with an Ollama provider, native tool calling silently degrades to JSON-in-text mode without any warning.

**Location:** `providers/ollama.sh:9-38`

**Fix:** Log a debug warning when tools are provided but provider doesn't support them:

```bash
_call_ollama_api() {
  local model="$1" system_prompt="$2" messages_json="$3" temperature="${4:-0.2}"
  # Ollama does not support native tool calling — tools are passed via system prompt
  [[ -n "${5:-}" && "${5:-}" != "[]" ]] && \
    _nl_debug "OLLAMA" "native tool calling not supported, tools passed via system prompt" 2>/dev/null || true
  # ...existing code...
```

---

### R9 (P2, Robustness): Streaming trap cleanup on signals

**Issue:** In streaming functions (`_call_anthropic_api_stream`, `_call_openai_api_stream`, `_call_gemini_api_stream`), the headers file cleanup uses `trap ... RETURN`. This fires on function return but NOT on `SIGINT`/`SIGTERM` sent to the process. If a user Ctrl-C's during streaming, the temp file containing the API key may persist.

**Location:** `providers/anthropic.sh:77`, `providers/openai.sh:135`, `providers/gemini.sh:94`

**Fix:** Add signal traps alongside RETURN:

```bash
trap 'rm -f "$headers_file"' RETURN INT TERM
```

Note: The non-streaming variants in Anthropic already have this same pattern. The risk is mitigated by `umask 0077` (files are 0600), but explicit cleanup is better.

---

### R10 (P2, Testing): No unit tests for Ollama chunking strategies

**Issue:** The 5 Ollama chunking strategies implemented in PR #3 have no dedicated unit tests. The 37 existing test suites pass (no regressions), but there's no verification that:
- Strategy 1 actually limits tool help to top 2
- Strategy 2 truncates to 15 lines
- Strategy 3 skips learning data
- Strategy 4 reduces truncation to 800 chars
- Strategy 5 skips doc RAG

**Fix:** Add test cases to `tests/unit/test_nl_prompt.sh` or a new `test_ollama_chunking.sh`:

```bash
test_ollama_chunking_tool_help_limit() {
  llm_provider="ollama-roachie"
  # Mock 5 tools, verify only top 2 get full help
  # ...
}

test_ollama_chunking_learning_skip() {
  llm_provider="ollama-roachie"
  persistent_failures=("test failure")
  local prompt
  prompt=$(_nl_build_prompt "$base_prompt")
  # Assert no "MISTAKES TO AVOID" section in output
  assert_not_contains "$prompt" "MISTAKES TO AVOID"
}
```

---

## What's Working Well

1. **Security pipeline**: 5-step validation (whitelist → metachar → SQL mutation → tool existence → RBAC) with array-based execution is robust. 113 security tests provide good coverage.

2. **Ollama chunking** (just implemented): Reduces input from ~2,250 to ~800-1,000 tokens. All 5 strategies are properly gated on `ollama-*` provider prefix.

3. **Provider plugin architecture**: Clean separation with per-provider files. Consistent error handling patterns across all 5 providers. Streaming implementations handle connection drops, partial data, and rate limiting.

4. **Semantic tool matching**: Combined regex+embedding approach achieves 99-100% accuracy across all providers. Query embedding reuse between tool matching and doc RAG avoids redundant API calls.

5. **Cost tracking**: Accurate per-model pricing with millicent precision, session accumulator with flock serialization for concurrent writes.

6. **Persistent learning**: JSONL-based failure/success database with deduplication, age-based pruning, and sanitization of all stored data.

---

## Implementation Priority

| Phase | Items | Impact |
|-------|-------|--------|
| Phase 1 (Quick fixes) | R1, R6 | Doc accuracy + Ollama token savings |
| Phase 2 (Refactoring) | R2, R3 | Reduce ~300 lines of duplication |
| Phase 3 (Robustness) | R4, R5, R7, R8, R9 | Better native tool calling, cleanup, UX |
| Phase 4 (Testing) | R10 | Prevent chunking regressions |
