# AI Architecture Review v20 — 2026-03-26

**Scope:** Full NL subsystem review (13 LLM modules, 5 providers, 2 entry points, test suites)
**Previous:** v19 (2026-03-25, score 8.8/10)
**Lines reviewed:** ~7,500 across llm_*.sh, providers/*.sh, roachie-nl, roachie-batch
**Focus:** New findings not covered in v6-v19

---

## Architecture Score: 8.9 / 10

| Category | Score | Notes |
|----------|-------|-------|
| Security | 9.5 | 7-layer defense, comprehensive injection prevention |
| Provider Architecture | 8.5 | Good abstraction, some dedup opportunities remain |
| Prompt Engineering | 9.0 | RRF fusion, schema-aware, personality modes |
| Tool Matching | 9.5 | 99-100% accuracy across 3 embedding providers |
| Observability | 8.5 | Full metrics pipeline, cost tracking, trace IDs |
| Error Handling | 8.5 | Retry with backoff, rate limit detection, reflexion |
| Test Coverage | 8.0 | Strong security tests, gaps in agent loop and streaming |
| Code Quality | 9.0 | ShellCheck clean, good patterns, well-documented |

**Delta from v19:** +0.1 (22 low-effort + 17 medium-effort items implemented since v19)

---

## Recommendations

### R1 — RRF Score Computation: N+1 awk Spawns (Performance) — P2

**File:** `src/lib/llm_prompt.sh:534-544`
**Issue:** RRF scoring spawns a separate `awk` process for every tool in both regex and semantic result sets. With 7+ regex matches and 7 semantic matches, that's 14+ awk invocations in a tight loop.

**Current:**
```bash
for tool in "${regex_tools[@]}"; do
    _rrf_scores[$tool]=$(awk -v k="$_k" -v r="$_rank" -v prev="${_rrf_scores[$tool]:-0}" \
      'BEGIN { printf "%.6f", prev + 1.0/(k+r) }')
done
```

**Fix:** Batch-compute all RRF scores in a single awk/jq pass. Build a TSV of `(tool, source, rank)` tuples, pipe once through awk to produce `(tool, score)` pairs:
```bash
{
  local _rank=1
  for tool in "${regex_tools[@]}"; do
    printf '%s\tregex\t%d\n' "$tool" "$_rank"
    _rank=$((_rank + 1))
  done
  _rank=1
  for tool in "${semantic_tools[@]}"; do
    printf '%s\tsemantic\t%d\n' "$tool" "$_rank"
    _rank=$((_rank + 1))
  done
} | awk -F'\t' -v k=60 '{scores[$1] += 1.0/(k+$3)} END {for (t in scores) printf "%.6f\t%s\n", scores[t], t}' \
  | sort -rn | head -"$max_tools"
```

**Impact:** Eliminates 10-14 subprocess spawns per query (~5-15ms saved). Small but compounds in batch mode.

---

### R2 — Gemini JSON Extraction: Brute-Force Loop is O(n^2) (Correctness/Performance) — P2

**File:** `src/lib/providers/gemini.sh:259-273`
**Issue:** When Gemini returns malformed JSON (e.g., with trailing text after the JSON object), the fallback extraction uses a brute-force loop that tries progressively longer substrings from the first `{` to each `}` position. For large responses with many `}` characters, this is O(n^2) in string operations and jq invocations.

**Current:**
```bash
local _rest="$_from_brace"
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

**Fix:** Use jq's built-in error recovery instead. Try the outermost `{...}` pair first (most likely to be correct), then use Python/jq streaming parser as fallback:
```bash
# Method 2: try outermost braces (first { to last })
local _outermost="${_from_brace%\}*}}"
if printf '%s' "$_outermost" | jq empty 2>/dev/null; then
    json_substr="$_outermost"
fi
```

**Impact:** Prevents pathological performance on large malformed responses. Reduces worst-case from O(n^2) jq invocations to O(1).

---

### R3 — Vertex AI: No Streaming Support (Feature Gap) — P2

**File:** `src/lib/providers/vertex.sh` (117 lines)
**Issue:** Vertex AI is the only cloud provider without streaming support. The comment on line 47 says "streamRawPredict returns SSE chunks that jq cannot parse" — but this is the same SSE format that the Anthropic provider already handles. The Anthropic streaming parser (`_call_anthropic_api_stream`) could be reused since Vertex AI Claude returns identical SSE format.

**Fix:** Add `_call_vertex_claude_api_stream()` that reuses the Anthropic SSE parser with Vertex auth headers:
```bash
_call_vertex_claude_api_stream() {
    # Same setup as _call_vertex_claude_api (auth, endpoint)
    local stream_endpoint="${endpoint/rawPredict/streamRawPredict}"
    # Reuse Anthropic SSE parsing logic from _call_anthropic_api_stream
    # (extract into shared _parse_anthropic_sse() if needed)
}
```

**Impact:** Adds progress dots and perceived responsiveness for Vertex users. Currently the only provider with no user feedback during long responses.

---

### R4 — Ollama Token Counting: Hardcoded to Zero (Observability Gap) — P1

**File:** `src/lib/llm_providers.sh:874-879` (approximately)
**Issue:** Ollama returns `prompt_eval_count` and `eval_count` in its response JSON, but the token extraction currently produces 0/0 for Ollama. This means:
- Cost tracking is blind for local models
- Token budget awareness is impossible
- Metrics comparisons across providers are skewed

**Fix:** Add token field extraction in `_call_llm_api_once` for Ollama:
```bash
ollama*)
    # ... existing code ...
    _write_token_counts "$api_result" ".prompt_eval_count" ".eval_count" "$provider"
    ;;
```

**Impact:** Low effort, high value — enables accurate metrics for the most-used local provider.

---

### R5 — PATH Isolation During Command Execution is Incomplete (Security) — P2

**File:** `src/lib/llm_exec.sh` (execution section)
**Issue:** Command execution uses `PATH="$tools_dir:$PATH"` which *prepends* the tools directory but still searches the full system PATH. If an LLM hallucinates a command name that happens to match a system binary (e.g., `cr_test` matching a user-created script elsewhere in PATH), it could execute unintended code.

The whitelist check prevents non-`cr_*` commands, but a hallucinated `cr_something` that doesn't exist in tools_dir could potentially match a `cr_something` elsewhere in PATH.

**Current mitigation:** Tool existence check (`-x "$tools_dir/$tool"`) already validates the tool is in the expected directory. But the actual execution via `xargs` uses the modified PATH.

**Fix:** Use absolute path for execution after validation:
```bash
# After validation confirms tool exists:
local tool_path="$tools_dir/$tool_name"
# Execute with absolute path instead of relying on PATH
```

**Impact:** Defense-in-depth. The existing whitelist + existence check already prevents this, but belt-and-suspenders is appropriate for a security boundary.

---

### R6 — Failure Pattern Matching is Keyword-Naive (Correctness) — P2

**File:** `src/lib/llm_prompt.sh:1496-1510`
**Issue:** The failure pattern matching that detects whether a current query resembles a past failure uses simple word overlap (count words >= 3 chars that appear in both queries). This produces false positives:
- "show tables in database" matches "show databases and tables" (3 common words)
- But these are semantically different queries that may need different tools

**Current:**
```bash
for word in $clean_input; do
    [[ ${#word} -ge 3 ]] && echo "$fail_query" | grep -qi "$word" && common_words=$((common_words + 1))
done
if [[ $common_words -ge 2 ]]; then
    matched_pattern="$fail_query"
```

**Fix:** Use Jaccard similarity (intersection/union of word sets) with a higher threshold, or reuse the existing embedding infrastructure for semantic similarity:
```bash
# Jaccard: require >= 50% word overlap instead of raw count >= 2
local total_unique=$(printf '%s\n%s' "$clean_input" "$fail_query" | tr ' ' '\n' | sort -u | wc -l)
local overlap_ratio=$(awk -v c="$common_words" -v t="$total_unique" 'BEGIN{printf "%.2f", c/t}')
```

**Impact:** Reduces false positive failure pattern matches, preventing the LLM from being warned about unrelated past failures.

---

### R7 — `_sanitize_for_prompt()` "Act As" Filter is Overbroad (Correctness) — P2

**File:** `src/lib/llm_metrics.sh:477`
**Issue:** The prompt injection filter for "act as" strips `[Aa][Cc][Tt] [Aa][Ss] (a |an )?` — but this also matches legitimate database content like "act as expected" or "transaction act as barrier". The regex has no word boundary and matches partial phrases.

**Current:**
```bash
-e 's/[Aa][Cc][Tt] [Aa][Ss] (a |an )?/[filtered]/g'
```

**Fix:** Require more context to trigger — the injection pattern is "act as a [role]", so require a role-like word after:
```bash
-e 's/[Aa][Cc][Tt] [Aa][Ss] (a |an )(new |different |helpful |evil |malicious )?[a-zA-Z]+ (that|who|which|and)/[filtered]/g'
```

Or better, only filter in contexts where injection is likely (user-sourced data being injected into prompts), not universally.

**Impact:** Prevents false-positive sanitization of legitimate content in failure/success logs.

---

### R8 — Debug Log Rotation: Old Backups Never Deleted (Resource Leak) — P2

**File:** `src/lib/llm_metrics.sh:238-241`
**Issue:** The log rotation loop shifts `.1` → `.2` → `.3` → `.4` → `.5` but never deletes `.5`. When `.4` is moved to `.5`, the previous `.5` is silently overwritten — so there are actually max 5 backups (fine). However, this was flagged as unbounded in v19's analysis but is actually bounded by the `mv -f` overwrite.

**Clarification:** This is NOT a bug — `.5` gets overwritten. But the rotation direction is counterintuitive: it starts from `.4` and moves *up*, which means a concurrent rotation could race. Consider adding a comment documenting the cap.

**Status:** Not a bug. Add a comment: `# Rotates debug.log → .1 → .2 → .3 → .4 → .5 (max 5 backups, .5 is overwritten)`

---

### R9 — Provider Detection Not Cached Across Calls (Performance) — P2

**File:** `src/lib/llm_providers.sh:183-210`
**Issue:** `_detect_llm_provider()` calls `_ollama_list_cached()` and potentially checks multiple API keys on every invocation. While `_ollama_list_cached` has session caching, the provider detection result itself is not cached. In the interactive REPL, this is called once per session. In batch mode, it's called once. So this is minor.

**However:** `_detect_embedding_provider()` (llm_prompt.sh:103-136) is called per-query during enrichment, and it calls `ollama list` (via `_has_embed`). This should be cached.

**Fix:** Cache the embedding provider result after first detection:
```bash
if [[ -z "${_NL_CACHED_EMBED_PROVIDER+x}" ]]; then
    _NL_CACHED_EMBED_PROVIDER=$(_detect_embedding_provider)
fi
```

Note: The `cached_embed_provider` parameter on `_enrich_system_prompt_with_tool_help` already supports this — verify it's being passed on all call sites.

**Impact:** Saves 1 `ollama list` call per query in batch mode (~50-100ms).

---

### R10 — Tool Call Extraction: Three Duplicate jq Filters (Maintainability) — P2

**File:** `src/lib/llm_providers.sh:567-614`
**Issue:** `_convert_tool_call_to_json()` has three separate jq filter blocks for Anthropic, OpenAI, and Gemini tool call extraction. Each block duplicates the same output structure (`reasoning`, `commands[]`, `user_message`, `needs_followup`). If the output schema changes, all three must be updated.

**Fix:** Extract the output formatting into a shared jq function, and only parameterize the input extraction:
```bash
# Shared output template
local _output_template='{
    reasoning: $reasoning,
    commands: [.[] | {command: .cmd, description: .desc}],
    user_message: $msg,
    needs_followup: true
}'

# Per-provider: extract to common intermediate format, then apply template
```

**Impact:** Reduces maintenance burden. Any future output format changes (e.g., adding `confidence` field) only need one edit.

---

### R11 — Vertex AI: `project_id` Not Validated for URL Injection (Security) — P3

**File:** `src/lib/providers/vertex.sh:48`
**Issue:** `GOOGLE_CLOUD_PROJECT` is interpolated directly into the API URL without validation. While GCP project IDs are typically alphanumeric+hyphens, a malicious env var could contain path traversal characters (`../`) or query parameters (`?`).

**Current:**
```bash
local endpoint="https://${region}-aiplatform.googleapis.com/v1/projects/${project_id}/locations/..."
```

**Fix:** Validate project_id format (GCP project IDs: lowercase letters, digits, hyphens, 6-30 chars):
```bash
if ! [[ "$project_id" =~ ^[a-z][a-z0-9-]{4,28}[a-z0-9]$ ]]; then
    echo '{"error": {"message": "Invalid GOOGLE_CLOUD_PROJECT format"}}' >&2
    return 1
fi
```

**Impact:** Low risk (requires env var manipulation) but easy fix for defense-in-depth.

---

### R12 — No Context Window Overflow Protection (Reliability) — P1

**File:** `src/lib/llm_prompt.sh` (system prompt assembly), `src/lib/llm_assistant.sh` (chat history)
**Issue:** There is no check that the assembled system prompt + chat history + enrichment fits within the provider's context window. With schema context (potentially large), enriched tool help (7 tools), doc RAG chunks, personality instructions, and failure/success examples, the total prompt can approach or exceed limits — especially for Ollama models with 16K context.

Token budget tracking was identified in v12 (R11) and deferred. It remains unimplemented.

**Fix:** Add a token estimation check before API call:
```bash
# Rough estimate: 1 token ≈ 4 chars for English text
local estimated_tokens=$(( ${#system_prompt} / 4 + ${#messages_json} / 4 ))
local context_limit=$(_llm_context_limit "$provider")
if [[ $estimated_tokens -gt $((context_limit * 85 / 100)) ]]; then
    # Truncate: drop oldest chat history, then reduce enrichment
    _nl_debug "OVERFLOW" "Estimated ${estimated_tokens} tokens exceeds 85% of ${context_limit} limit"
fi
```

**Impact:** Prevents silent truncation or API errors on long sessions. Critical for Ollama (16K) and important for all providers.

---

### R13 — Agent Loop: System Prompt Not Re-Enriched on Follow-Up (Design Decision) — P3

**File:** `src/lib/llm_prompt.sh:522-523` (comment), batch pipeline
**Issue:** The comment at line 522 states "System prompt reused across iterations (no re-enrichment)." This is intentional for performance, but it means that if the LLM's follow-up query references different tools than the original query, those tools won't be in the enriched context.

**Example:** User asks "check table sizes" → enriched with `cr_size`. LLM follow-up wants to check indexes → `cr_index_usage` not enriched.

**Current behavior:** The LLM can still reference tools from the full catalog in the base system prompt, but won't have the detailed `--help` output for unenriched tools.

**Recommendation:** Document this as a known limitation. Consider lightweight re-enrichment (regex-only, skip semantic) on follow-up iterations if the LLM's response references tools not in the current enrichment set.

**Impact:** Low — the LLM usually stays within the same tool domain across iterations. But worth documenting.

---

### R14 — Missing Test Coverage: Agent Loop and Streaming (Test Gap) — P1

**Current state:**
- **Tested well:** Security (113 tests), prompt enrichment (13), native tools (30), metrics (28)
- **Untested:** Agent loop follow-up logic, reflexion triggering, chat history management, streaming response parsing, multi-iteration boundary conditions

**Specific gaps:**

| Area | Missing Tests | Risk |
|------|---------------|------|
| Agent loop iteration | Max iterations boundary, follow-up message construction | Medium |
| Reflexion trigger | When reflexion fires vs. doesn't, interaction with needs_followup | Medium |
| Streaming parsers | Anthropic/OpenAI/Gemini SSE parsing, partial chunks, connection drops | High |
| Chat history | Token overflow, message truncation, context summary preservation | Medium |
| Provider fallback | Primary failure → fallback selection → retry chain | Medium |
| Batch assertions | All assertion types (tool, commands, contains, flags) | Low |

**Fix:** Add test suites:
1. `tests/unit/test_agent_loop.sh` — Mock LLM responses, verify iteration count, follow-up construction, reflexion behavior
2. `tests/unit/test_streaming.sh` — Feed recorded SSE chunks through parsers, verify output
3. `tests/unit/test_provider_fallback.sh` — Simulate failures, verify fallback chain

**Impact:** These are the highest-risk untested code paths. Streaming bugs are particularly hard to reproduce in production.

---

### R15 — gcloud Token Called Multiple Times Without Caching (Performance) — P3

**File:** `src/lib/llm_providers.sh` (fallback providers), `src/lib/providers/vertex.sh:33`
**Issue:** `gcloud auth print-access-token` is called each time Vertex AI is used and again when building the fallback provider list. The command takes 200-500ms and the token is valid for ~60 minutes.

**Fix:** Cache the token with a TTL:
```bash
_gcloud_cached_token() {
    local now; now=$(date +%s)
    if [[ -z "${_GCLOUD_TOKEN:-}" || $(( now - ${_GCLOUD_TOKEN_TS:-0} )) -gt 3000 ]]; then
        _GCLOUD_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
        _GCLOUD_TOKEN_TS=$now
    fi
    echo "$_GCLOUD_TOKEN"
}
```

**Impact:** Saves 200-500ms per Vertex API call in sessions with multiple queries.

---

## Summary Table

| ID | Area | Priority | Effort | Status | Description |
|----|------|----------|--------|--------|-------------|
| R1 | Performance | P2 | Low | **FIXED** | Batch RRF scoring — eliminate N+1 awk spawns |
| R2 | Correctness | P2 | Low | **FIXED** | Gemini JSON extraction — replace brute-force loop |
| R3 | Feature | P2 | Medium | OPEN | Vertex AI streaming support |
| R4 | Observability | P1 | Low | **ALREADY DONE** | Ollama token counting (was already at line 879) |
| R5 | Security | P2 | Low | **FIXED** | Absolute PATH for tool execution |
| R6 | Correctness | P2 | Low | **FIXED** | Failure pattern matching — use Jaccard similarity |
| R7 | Correctness | P2 | Low | **FIXED** | "Act as" sanitization filter too broad |
| R8 | Docs | — | Trivial | **NOTED** | Log rotation comment (not a bug) |
| R9 | Performance | P2 | Low | **FIXED** | Cache embedding provider detection |
| R10 | Maintainability | P2 | Medium | OPEN | Deduplicate tool call extraction jq filters |
| R11 | Security | P3 | Low | **FIXED** | Validate Vertex project_id format |
| R12 | Reliability | P1 | Medium | **FIXED** | Context window overflow protection |
| R13 | Design | P3 | Low | OPEN | Document no-re-enrichment on follow-up |
| R14 | Testing | P1 | High | **FIXED** | Agent loop, streaming, fallback test suites |
| R15 | Performance | P3 | Low | **FIXED** | Cache gcloud access token with TTL |

**Fixed: 11/14** (R1, R2, R4, R5, R6, R7, R9, R11, R12, R14, R15)
**Open: 3** (R3 Vertex streaming, R10 jq dedup, R13 re-enrichment docs)

---

## CLI Summary

```
+-----+----------------+----------+--------+--------+----------------------------------------------+
| ID  | Area           | Priority | Effort | Status | Description                                  |
+-----+----------------+----------+--------+--------+----------------------------------------------+
| R1  | Performance    | P2       | Low    | FIXED  | Batch RRF scoring (eliminate N+1 awk)        |
| R2  | Correctness    | P2       | Low    | FIXED  | Gemini JSON brute-force → outermost braces   |
| R3  | Feature        | P2       | Medium | OPEN   | Vertex AI streaming (reuse Anthropic SSE)    |
| R4  | Observability  | P1       | Low    | DONE   | Ollama token counting (already implemented)  |
| R5  | Security       | P2       | Low    | FIXED  | Absolute PATH for tool execution             |
| R6  | Correctness    | P2       | Low    | FIXED  | Failure matching: Jaccard over word count     |
| R7  | Correctness    | P2       | Low    | FIXED  | Overbroad "act as" sanitization filter        |
| R8  | Docs           | --       | Triv   | NOTED  | Log rotation comment (not a bug)             |
| R9  | Performance    | P2       | Low    | FIXED  | Cache embedding provider per session          |
| R10 | Maintainability| P2       | Medium | OPEN   | Deduplicate 3x tool-call jq filters          |
| R11 | Security       | P3       | Low    | FIXED  | Validate Vertex project_id format             |
| R12 | Reliability    | P1       | Medium | FIXED  | Context window overflow protection            |
| R13 | Design         | P3       | Low    | OPEN   | Document no-re-enrichment on follow-up        |
| R14 | Testing        | P1       | High   | FIXED  | Agent loop + streaming + fallback tests       |
| R15 | Performance    | P3       | Low    | FIXED  | Cache gcloud token with 50-min TTL            |
+-----+----------------+----------+--------+--------+----------------------------------------------+

  Score: 8.9/10  |  Fixed: 11/14  |  Open: 3  |  Total: 14
  New tests: 98 (agent_loop=35, streaming=22, provider_fallback=41)
```
