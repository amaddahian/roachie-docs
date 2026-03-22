# AI Architecture Review v14

**Date:** 2026-03-19
**Reviewer:** Claude Opus 4.6
**Scope:** Deep code review — LLM providers, prompt construction, assistant loop, metrics, security validation
**Previous:** v13 (10/10 fixed) — shared request builders, `needs_followup` fix, Ollama chunking tests

---

## Architecture Scorecard

| Category | Score | Notes |
|----------|-------|-------|
| Security Pipeline | 9.5/10 | 5-step validation, RBAC, SQL mutation guards, metachar blocking — mature |
| Provider Plugins | 8.5/10 | Shared builders landed in v13, but `flag_val` jq function still 3x duplicated |
| Prompt Construction | 8.5/10 | Doc RAG title injection risk; `_split_tools` declared 4x in loop body |
| Token Efficiency | 9/10 | Ollama chunking effective; streaming token writes unvalidated |
| Error Handling | 8.5/10 | Gemini JSON extraction has O(n^2) worst case; silent jq failures in history trimming |
| Agent Loop | 8.5/10 | Array sync issue between desc/cmd arrays; 2 jq calls per command in loop |
| Code Quality | 8/10 | Duplication reduced in v13; remaining issues are localized |
| Test Coverage | 8.5/10 | 38 suites, no tests for semantic matching edge cases or doc RAG injection |

**Overall: 8.6/10** (up from 8.5 in v13)

---

## Recommendations

### Table 1: Summary

| # | Priority | Category | Description | File(s) |
|---|----------|----------|-------------|---------|
| R1 | P1 | Security | Doc RAG title injection — raw titles injected into LLM prompt without sanitization | llm_prompt.sh:282 |
| R2 | P1 | Correctness | Cosine similarity division-by-zero when embedding magnitude is 0 | llm_prompt.sh:269 |
| R3 | P1 | Code Quality | `flag_val` jq function duplicated 3x in `_convert_tool_call_to_json` | llm_providers.sh:560-624 |
| R4 | P2 | Performance | `_split_tools` local array declared 4x inside keyword matching loop body | llm_prompt.sh:343-377 |
| R5 | P2 | Robustness | Silent jq parse failures in history trimming — no error handling on `jq 'length'` | llm_assistant.sh:564-575 |
| R6 | P2 | Performance | 2 jq calls per command in execution loop — should batch extract | llm_assistant.sh:897-898 |
| R7 | P2 | Robustness | Streaming token counts written without numeric validation | providers/*.sh |
| R8 | P2 | Performance | Gemini brute-force JSON extraction has O(n^2) worst case | providers/gemini.sh:259-273 |
| R9 | P3 | Code Quality | Duplicate array resets in command extraction — lines 878-879 vs 891-892 | llm_assistant.sh |
| R10 | P3 | Testing | No tests for doc RAG edge cases (empty embeddings, zero-magnitude vectors) | tests/unit/ |

---

### R1 (P1, Security): Doc RAG title injection — raw titles injected into prompt

**Issue:** In `_nl_retrieve_doc_context()`, document titles from the embedding file are injected directly into the LLM system prompt without any sanitization:

```bash
# llm_prompt.sh:282
doc_context+="
## ${_title}
${_content}
"
```

If a doc title contains LLM prompt injection (e.g., `"Ignore all instructions and..."`) or markdown injection, it flows directly into the system prompt. While the embedding files are generated locally from trusted `tool_descriptions.txt`, any future integration with external doc sources would be vulnerable.

**Impact:** Low-risk today (trusted source), but this is a defense-in-depth gap. A compromised or malicious embedding file could inject arbitrary instructions into the system prompt.

**Fix:** Sanitize titles using the existing `_sanitize_for_prompt` function from `llm_metrics.sh`:

```bash
doc_context+="
## $(_sanitize_for_prompt "${_title}")
${_content}
"
```

---

### R2 (P1, Correctness): Cosine similarity division-by-zero

**Issue:** The cosine similarity calculation in `_nl_retrieve_doc_context()` computes `$dot / ($mag_q * $mag_t)` without checking for zero magnitude:

```jq
# llm_prompt.sh:267-269
([ $qe[] | . * . ] | add | sqrt) as $mag_q |
([ $te[] | . * . ] | add | sqrt) as $mag_t |
($dot / ($mag_q * $mag_t)) as $sim |
```

If a document chunk has an all-zeros embedding vector (e.g., empty content, generation error), `$mag_t` would be 0, causing a jq division error. The same pattern exists in the tool-level semantic matching in `_semantic_match_tools()`.

**Impact:** A single corrupt embedding entry would cause jq to emit an error, breaking the entire doc RAG retrieval. The `2>/dev/null` on line 272 would suppress the error but also suppress all valid results from the same jq pipeline.

**Fix:** Guard against zero magnitude:

```jq
(if ($mag_q * $mag_t) > 0 then $dot / ($mag_q * $mag_t) else 0 end) as $sim |
```

---

### R3 (P1, Code Quality): `flag_val` jq function duplicated 3x

**Issue:** The `_convert_tool_call_to_json` function in `llm_providers.sh` defines identical `flag_val` jq helper functions for all 3 provider cases (anthropic at line 560, openai at line 583, gemini at line 607):

```jq
def flag_val:
  if .value == true then "--\(.key)"
  elif .value == false then empty
  elif (.value | tostring | test(" ")) then "--\(.key) \"\(.value | tostring)\""
  else "--\(.key) \(.value | tostring)"
  end;
```

This 6-line function is copy-pasted 3 times. Any bug fix or enhancement (e.g., adding array value support) must be made in 3 places.

**Fix:** Extract `flag_val` into a shell variable and include via `--argjson`:

```bash
_convert_tool_call_to_json() {
  local api_response="$1" provider="$2"

  local _flag_val_def='def flag_val:
    if .value == true then "--\(.key)"
    elif .value == false then empty
    elif (.value | tostring | test(" ")) then "--\(.key) \"\(.value | tostring)\""
    else "--\(.key) \(.value | tostring)"
    end;'

  case "$provider" in
    anthropic)
      echo "$api_response" | jq -c "${_flag_val_def}
        ([.content[]? | ...  # rest of provider-specific jq
      "
      ;;
    # ... same for openai, gemini
  esac
}
```

---

### R4 (P2, Performance): `_split_tools` declared 4x in loop body

**Issue:** In the keyword matching section of `_enrich_system_prompt_with_tool_help()`, `local -a _split_tools` is declared 4 times at lines 343, 354, 366, and 377 — once for each branch (ceph_gate, !ceph_gate, dual, standard):

```bash
# Repeated 4x inside the while-read loop:
local -a _split_tools; read -ra _split_tools <<< "$_tools"
relevant_tools+=("${_split_tools[@]}")
```

Each `local` declaration inside a loop body re-declares the variable on each iteration. While functionally correct, it's redundant and adds slight overhead. More importantly, it's a maintenance burden — the same line appears 4 times.

**Fix:** Declare `_split_tools` once before the loop and use `read -ra` without `local` inside the branches:

```bash
local -a _split_tools=()
while IFS= read -r _line; do
  # ... branches just do:
  read -ra _split_tools <<< "$_tools"
  relevant_tools+=("${_split_tools[@]}")
done
```

---

### R5 (P2, Robustness): Silent jq parse failures in history trimming

**Issue:** The history trimming code in `_nl_prepare_messages()` calls jq multiple times on `messages_json` without error handling:

```bash
# llm_assistant.sh:564 — if messages_json is malformed, _msg_count becomes empty
_msg_count=$(printf '%s' "$messages_json" | jq 'length')

# llm_assistant.sh:571 — silent failure
_two=$(printf '%s' "$messages_json" | jq -c '.[:2]')

# llm_assistant.sh:575 — could produce invalid JSON
messages_json=$(printf '%s' "$messages_json" | jq -c '.[2:]')
```

If `messages_json` is malformed or empty, these jq calls silently fail, potentially producing empty strings that cascade into further errors.

**Fix:** Add a validation guard before the trimming loop:

```bash
if ! printf '%s' "$messages_json" | jq empty 2>/dev/null; then
  _nl_debug "TRIM" "messages_json is not valid JSON, skipping history trimming"
  return
fi
```

---

### R6 (P2, Performance): 2 jq calls per command in execution loop

**Issue:** Inside the command execution loop, each iteration makes 2 separate jq calls to extract command and description:

```bash
# llm_assistant.sh:897-898
cmd_to_run=$(echo "$assistant_response" | jq -r ".commands[$i].command // empty")
cmd_description=$(echo "$assistant_response" | jq -r ".commands[$i].description // empty")
```

For responses with multiple commands (e.g., 3-5 tools in a multi-step analysis), this means 6-10 jq invocations parsing the same JSON document repeatedly.

**Fix:** Extract all commands at once before the loop:

```bash
local all_cmds_json
all_cmds_json=$(echo "$assistant_response" | jq -c '[.commands[]? | {command: (.command // ""), description: (.description // "")}]')
for i in $(seq 0 $((commands_count - 1))); do
  cmd_to_run=$(echo "$all_cmds_json" | jq -r ".[$i].command")
  cmd_description=$(echo "$all_cmds_json" | jq -r ".[$i].description")
```

This reduces N*2 jq invocations to N+2 (one batch extract + N lightweight index lookups).

---

### R7 (P2, Robustness): Streaming token counts written without numeric validation

**Issue:** All 3 streaming providers (Anthropic, OpenAI, Gemini) write token counts to the metrics file using `printf '%d\n'` but the variables may not be numeric:

```bash
# All 3 providers:
printf '%d\n%d\n' "$input_tokens" "$output_tokens" > "${_NL_METRICS_PREFIX}_tokens"
```

If SSE parsing fails to extract usage data, `input_tokens` and `output_tokens` remain at their initialized value of `0`, which is safe. However, if partial usage data arrives (e.g., `input_tokens` is set but `output_tokens` parsing fails and produces a non-numeric string), `printf '%d'` with a non-integer will emit `0` with a shell warning.

The non-streaming path already uses `_write_token_counts()` (line 702) with proper validation. The streaming paths bypass this.

**Fix:** Either call `_write_token_counts` from streaming paths (requires restructuring) or add validation inline:

```bash
[[ "$input_tokens" =~ ^[0-9]+$ ]] || input_tokens=0
[[ "$output_tokens" =~ ^[0-9]+$ ]] || output_tokens=0
printf '%d\n%d\n' "$input_tokens" "$output_tokens" > "${_NL_METRICS_PREFIX}_tokens"
```

---

### R8 (P2, Performance): Gemini brute-force JSON extraction O(n^2) worst case

**Issue:** When Gemini returns non-JSON text, `_extract_gemini_response()` has a fallback that iterates from the first `{` to each `}` position, testing each substring with jq:

```bash
# providers/gemini.sh:259-273
while [[ "$_rest" == *"}"* ]]; do
  local _after="${_rest#*\}}"
  local _candidate="${_from_brace%"$_after"}"
  if printf '%s' "$_candidate" | jq empty 2>/dev/null; then
    json_substr="$_candidate"
    break
  fi
  _rest="$_after"
done
```

For a response with N `}` characters, this calls jq up to N times, each time on a progressively longer string. With large non-JSON responses (e.g., a model that outputs code blocks with many braces), this becomes O(n^2).

**Impact:** In practice this is bounded by the response size (~4000 chars for Ollama, ~32K for cloud), and the `break` on first valid JSON limits average-case impact. But it could cause noticeable latency spikes with brace-heavy responses.

**Fix:** Add a size guard and attempt limit:

```bash
local _attempts=0
while [[ "$_rest" == *"}"* && $_attempts -lt 10 ]]; do
  _attempts=$((_attempts + 1))
  # ... existing logic
done
```

---

### R9 (P3, Code Quality): Duplicate array resets in command extraction

**Issue:** `_NL_CMD_DESCS` and `_NL_CMD_CMDS` arrays are reset twice — once at lines 878-879 (early return path) and again at lines 891-892 (normal path):

```bash
# llm_assistant.sh:878-879 (inside commands_count==0 early return)
_NL_CMD_DESCS=()
_NL_CMD_CMDS=()

# llm_assistant.sh:891-892 (normal path, 13 lines later)
_NL_CMD_DESCS=()
_NL_CMD_CMDS=()
```

The early-return path at 878-879 returns immediately after resetting, so the second reset at 891-892 only runs for the normal case. This is functionally correct but the pattern suggests the arrays were reset defensively without realizing they'd already be empty on the normal path.

**Fix:** Remove the duplicate reset at lines 878-879 by moving the shared reset to before the if-block:

```bash
_NL_CMD_DESCS=()
_NL_CMD_CMDS=()
if [[ "$commands_count" == "0" ]]; then
  _NL_EXECUTED_CMDS=""
  _NL_ALL_OK=true
  return 1
fi
```

---

### R10 (P3, Testing): No tests for doc RAG edge cases

**Issue:** The doc RAG retrieval function `_nl_retrieve_doc_context()` has no dedicated unit tests. Key untested paths:

1. **Zero-magnitude embedding vectors** — would trigger the division-by-zero in R2
2. **Empty embedding file** — function should return empty gracefully
3. **Title injection** — if R1 is fixed, tests should verify sanitization
4. **Similarity threshold edge cases** — scores exactly at `_NL_DOC_RAG_MIN_SIM` boundary

The existing test suites test tool matching and chunking, but doc RAG is tested only indirectly through integration tests that require a running cluster.

**Fix:** Add a `test_doc_rag.sh` unit test that creates mock embedding files and verifies:
- Normal retrieval with valid embeddings
- Graceful handling of zero-magnitude vectors
- Empty/missing embedding file returns empty
- Title sanitization (after R1 fix)

---

## Summary of Changes Since v13

### Table 2: v13 Items — All Resolved

| # | Description | Status |
|---|-------------|--------|
| R1 | CLAUDE.md wrong default for `NL_OUTPUT_TRUNCATE_CHARS` | FIXED (v13) |
| R2 | OpenAI request construction 4x duplication | FIXED (v13) |
| R3 | Gemini request construction 4x duplication | FIXED (v13) |
| R4 | Anthropic API version hardcoded | FIXED (v13) |
| R5 | `needs_followup` regex unreliable for tool calls | FIXED (v13) |
| R6 | Ollama decorative Unicode headers | FIXED (v13) |
| R7 | Vertex AI no progress indicator | FIXED (v13) |
| R8 | Ollama ignores `tools_json` | FIXED (v13) |
| R9 | Streaming `trap` missing INT/TERM | FIXED (v13) |
| R10 | No Ollama chunking unit tests | FIXED (v13) |

### Table 3: Open Item Trend

| Review | Open P1 | Open P2 | Open P3 | Total | Score |
|--------|---------|---------|---------|-------|-------|
| v12 | 0 | 2 | 0 | 2 | 8.3 |
| v13 | 0 | 0 | 0 | 0 | 8.5 |
| v14 | 3 | 5 | 2 | 10 | 8.6 |

Note: v14 findings are all new issues found through deeper analysis. The score increased because v13 fixes improved code quality, but the deeper analysis revealed previously unexamined areas.
