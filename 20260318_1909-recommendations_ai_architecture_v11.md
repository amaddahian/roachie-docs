# AI Architecture Review v11 — 2026-03-18

Review of the roachie NL/LLM subsystem following completion of v10 cycle (24 of 25 items fixed). Fresh comprehensive analysis across all 13 AI modules, 5 provider plugins, 7 prompt templates, 35 test suites, and evaluation harness.

## Table 1: v11 Recommendations

| ID | Priority | Category | Description |
|----|----------|----------|-------------|
| R1 | P1 | Bug | **FIXED** OpenAI streaming: `accumulated_tool_calls` referenced before declaration — empty response check uses undefined variable |
| R2 | P1 | Security | **FIXED** Sticky context extraction via jq regex can be poisoned by crafted user queries injecting fake cluster/database/tenant values |
| R3 | P1 | Error handling | **FIXED** `_ms_now()` on bash < 5 spawns `python3` subprocess — fails silently if python3 not installed, breaks timing metrics |
| R4 | P1 | Concurrency | **FIXED** Cost file `_nl_add_session_cost_inner` runs inside flock but `mv -f` may complete after lock released — atomic write not guaranteed |
| R5 | P2 | Performance | Prompt token budget: system prompt ~10K tokens with full tool catalog + enrichment + schema; no per-provider budget enforcement beyond history trimming |
| R6 | P2 | Consistency | **FIXED** Cost formatting in `_csv_escape` and awk arithmetic not locale-safe — `LC_ALL=C` not set, decimal separator varies by locale |
| R7 | P2 | Error handling | **FIXED** Debug log rotation keeps only 1 backup (`.1`) — each rotation silently discards all prior history |
| R8 | P2 | Testing | **FIXED** Test isolation: `test_nl_pipeline.sh` reuses global variables across sections without reset; cleanup not trap-guarded |
| R9 | P2 | Testing | **FIXED** No test coverage for schema-aware prompts (`_nl_fetch_schema_context`, caching, truncation, graceful fallback) |
| R10 | P2 | Testing | **FIXED** No test coverage for semantic tool matching (embedding provider detection, cosine similarity, combined regex+semantic) |
| R11 | P2 | Testing | **FIXED** Evaluation dataset has ~70 queries for 77 tools — insufficient coverage; no multi-tenant query scenarios |
| R12 | P2 | Testing | **FIXED** Inconsistent assertion frameworks — `test_llm_*.sh` files define custom `assert_eq`/`assert_contains` instead of sourcing `test_helpers.sh` |
| R13 | P2 | Dead code | **FIXED** `_NL_CONTEXT_WARN_TOKENS` defined in `llm_config.sh` but never referenced — actual overflow protection uses `_llm_context_limit()` |
| R14 | P2 | Security | **FIXED** Prompt injection via reflexion: command output fed back to LLM not truncated by `_sanitize_for_prompt` — only raw char truncation applied |
| R15 | P2 | Error handling | **FIXED** Gemini JSON extraction regex (`scan`) handles only 1 level of nesting — deeply nested responses silently fail to parse |
| R16 | P3 | Consistency | Ollama has no streaming implementation (`_call_ollama_api_stream` missing) — undocumented limitation |
| R17 | P3 | Maintenance | Prompt template files have no version comments showing when rules were added/modified |
| R18 | P3 | Dead code | `_log_nl()` parameter 16 (`successes_loaded`) accepted but never logged — API compatibility shim with no consumer |
| R19 | P3 | Config | Gemini `thinkingConfig` always included in requests even for non-thinking models — wastes tokens on incompatible models |
| R20 | P3 | Maintenance | Tool descriptions in `tool_descriptions.txt` have no automated sync check against actual tools in `tools/current/` |

**Total: 20 recommendations** (P1: 4, P2: 11, P3: 5)

---

## Table 2: Carry-Forward from Previous Reviews

| ID | Status | Description |
|----|--------|-------------|
| v10-R25 | Open | Dual-parameter rules duplicated across template files (P3) |
| v9-R2 | Skipped | Prompt token budget management — partially addressed by history trimming (user deferred) |
| v9-R8 | Skipped | Parallel tool execution (user deferred) |

---

## Table 3: Priority Summary

| Priority | Count | Categories |
|----------|-------|------------|
| **P1** | 4 | Bug (1), Security (1), Error handling (1), Concurrency (1) |
| **P2** | 11 | Performance (1), Consistency (1), Error handling (2), Testing (5), Dead code (1), Security (1) |
| **P3** | 5 | Consistency (1), Maintenance (2), Dead code (1), Config (1) |
| **Total** | **20** | |

---

## Detailed Findings

### R1 — OpenAI Streaming: Undefined Variable in Empty Response Check (P1) — **FIXED**

**Problem:** `accumulated_tool_calls` is referenced at line 271 in the empty response detection check, but not declared until line 285. When no tool calls arrive and text is empty, bash evaluates `"$accumulated_tool_calls" == "[]"` against an undefined variable, which expands to `"" == "[]"` (false). This means the empty response check falls through to produce a response with no content instead of reporting an error.

**File:** `src/lib/providers/openai.sh:271,285`

**Code:**
```bash
# Line 271 — references accumulated_tool_calls BEFORE declaration
if [[ -z "$had_error" && $chunk_count -eq 0 && -z "$accumulated_text" && "$accumulated_tool_calls" == "[]" ]]; then

# Line 285 — declared here, 14 lines later
local accumulated_tool_calls="[]"
```

**Fix:** Move declaration to line 192 alongside other accumulator variables:
```bash
local accumulated_text=""
local accumulated_tool_calls="[]"  # Must be before empty response check
```

---

### R2 — Sticky Context Extraction Vulnerable to Injection (P1) — **FIXED**

**Problem:** `_nl_sticky_context` is extracted from dropped conversation history using jq regex patterns like `capture("cluster[: ]+(?<v>[A-Z]{2})")`. A user query containing "use cluster: XX" or "database: malicious_db" will be captured and injected into subsequent queries' system prompt context, potentially directing the LLM to target unintended clusters or databases.

**File:** `src/lib/llm_assistant.sh:587-594`

**Code:**
```bash
_nl_sticky_context=$(printf '%s' "$_dropped_json" | jq -r '
  [.[].content // empty] | join(" ") |
  [
    (capture("cluster[: ]+(?<v>[A-Z]{2})"; "g") | "cluster: " + .v),
    (capture("database[: ]+(?<v>[a-z_][a-z0-9_]*)"; "g") | "database: " + .v),
    ...
  ]')
```

**Fix:** Only extract from assistant responses (role == "assistant"), not user messages:
```bash
_nl_sticky_context=$(printf '%s' "$_dropped_json" | jq -r '
  [.[] | select(.role == "assistant") | .content // empty] | join(" ") |
  ...
```

---

### R3 — `_ms_now()` Silent Failure on bash < 5 Without Python3 (P1) — **FIXED**

**Problem:** On bash < 5, `_ms_now()` calls `python3 -c 'import time; ...'`. If python3 is not installed (common on minimal Docker containers), it silently falls back to `$(date +%s) * 1000)` which loses millisecond precision. But worse, the `python3` invocation spawns a full interpreter per call — 10+ times per query during agent iterations, adding ~500ms overhead per query.

**File:** `src/lib/llm_metrics.sh:13`

**Fix:** Use `date +%s` multiplication directly as primary fallback (skip python3 entirely):
```bash
elif [[ -n "${EPOCHREALTIME:-}" ]]; then
  _ms_now() { echo $(( ${EPOCHREALTIME/./} / 1000 )); }
else
  _ms_now() { echo $(( $(date +%s) * 1000 )); }
fi
```

---

### R4 — Cost File Atomic Write Not Guaranteed Under Lock (P1) — **FIXED**

**Problem:** `_nl_add_session_cost_inner()` runs under flock via `_with_db_lock`, but the `mv -f "$tmp" "$file"` operation may not complete atomically on all filesystems. If the lock is released (subshell exits) before `mv` finishes, a concurrent reader can see a partial file. Additionally, `echo $((...))` creates a file without newline-terminated content, which `cat` reads correctly but some tools may not.

**File:** `src/lib/llm_metrics.sh:93-100`

**Fix:** Use `printf` with newline and sync the write:
```bash
_nl_add_session_cost_inner() {
  local query_cost="$1"
  local prev
  prev=$(cat "$_NL_SESSION_COST_FILE" 2>/dev/null || echo "0")
  printf '%d\n' $(( prev + query_cost )) > "${_NL_SESSION_COST_FILE}.tmp.$$"
  mv -f "${_NL_SESSION_COST_FILE}.tmp.$$" "$_NL_SESSION_COST_FILE"
}
```

---

### R5 — System Prompt Token Budget Not Managed (P2)

**Problem:** The system prompt grows unbounded with enrichment:
- Base prompt with 77-tool catalog: ~6,900 tokens
- Tool `--help` enrichment (up to 7 tools): ~3,200 tokens
- Doc RAG chunks (up to 3): ~1,000 tokens
- Schema context: ~500 tokens
- Total: ~11,600 tokens per query

History trimming at 80% of context limit addresses conversation growth but not the system prompt itself. Ollama with 4096 context can't fit both the system prompt and a meaningful conversation. Cloud providers charge for every input token.

**Files:** `src/lib/llm_prompt.sh` (prompt construction), `src/lib/llm_config.sh:29-37` (context limits)

**Fix:**
1. Compact the base prompt: list tool names only (not full catalog), move detailed rules into enrichment
2. Add system prompt token estimation before API call
3. Warn if system prompt exceeds 50% of provider context limit
4. For Ollama: cap enrichment to 3 tools, skip doc RAG

---

### R6 — Locale-Dependent Arithmetic and Formatting (P2) — **FIXED**

**Problem:** Cost formatting uses `awk` without `LC_ALL=C`. On locales with `,` as decimal separator (e.g., `de_DE`), `awk` arithmetic like `printf "%.4f"` outputs `0,0150` instead of `0.0150`, breaking downstream parsing and display.

**File:** `src/lib/llm_metrics.sh` (cost calculation and formatting functions)

**Fix:** Prefix all awk and printf calls with `LC_ALL=C`:
```bash
LC_ALL=C awk -v pt="$prompt_tokens" ...
```

---

### R7 — Debug Log Rotation Keeps Only 1 Backup (P2) — **FIXED**

**Problem:** Debug log rotation uses `mv -f "$debug_file" "${debug_file}.1"`, which overwrites any existing `.1` backup. On high-volume sessions with `NL_DEBUG=1`, each rotation discards all prior debug history except the current file.

**File:** `src/lib/llm_metrics.sh:198`

**Fix:** Implement N-backup rotation:
```bash
for i in 4 3 2 1; do
  [[ -f "${debug_file}.$i" ]] && mv -f "${debug_file}.$i" "${debug_file}.$((i+1))"
done
mv -f "$debug_file" "${debug_file}.1"
```

---

### R8 — Test Isolation and Cleanup in Pipeline Tests (P2) — **FIXED**

**Problem:** `test_nl_pipeline.sh` initializes global variables at the top (lines 20-34) that persist across test sections. Variables like `_nl_messages_json`, `clean_input`, `llm_provider` are reused without explicit reset. Mock tool directory cleanup at line 595 only runs if the script completes — no `trap` guards against early exit.

**File:** `tests/unit/test_nl_pipeline.sh:20-34,162,595`

**Fix:**
1. Add `trap 'rm -rf "$tools_dir" 2>/dev/null' EXIT` at setup
2. Add `_reset_test_state()` function called before each major section
3. Reset `_nl_messages_json="[]"` at start of each section

---

### R9 — No Test Coverage for Schema-Aware Prompts (P2) — **FIXED**

**Problem:** `_nl_fetch_schema_context()` queries the cluster for databases and tables, caches per session, and injects schema into the system prompt. No test validates: schema fetching, caching behavior, truncation at 5 databases / 50 tables, graceful fallback on connection failure, or correct injection into prompts.

**Fix:** Create `tests/unit/test_nl_schema_context.sh` with mocked SQL queries.

---

### R10 — No Test Coverage for Semantic Tool Matching (P2) — **FIXED**

**Problem:** Semantic matching (embeddings + cosine similarity) is active in the pipeline via `_semantic_match_tools()` and `_enrich_system_prompt_with_tool_help()`. The only validation is `tools/utils/test_embeddings.sh` which requires live embedding APIs. No unit tests cover: embedding provider detection, cosine similarity computation, regex+semantic combining logic, or top-7 filtering.

**Fix:** Create `tests/unit/test_llm_semantic_matching.sh` with mock embedding files and precomputed similarity values.

---

### R11 — Evaluation Dataset Insufficient Coverage (P2) — **FIXED**

**Problem:** `tests/evaluation/eval_dataset.tsv` has ~70 queries covering a subset of 77 tools. Missing: multi-tenant scenarios, tool-specific flag combinations, ambiguous queries requiring disambiguation, and edge cases where multiple tools could match.

**Fix:** Expand to 150+ queries. Incorporate queries from `examples/queries/example-queries.va.txt` and the 135-query test set from `test_embeddings.sh`.

---

### R12 — Inconsistent Test Assertion Frameworks (P2) — **FIXED**

**Problem:** `test_llm_extract.sh` and `test_llm_streaming.sh` define custom assertion functions (`assert_eq`, `assert_empty`, `assert_contains`) instead of sourcing `tests/test_helpers.sh` which provides standard `assert_equals`, `assert_contains`, `assert_command_succeeds`. This creates maintenance burden and inconsistent error message formatting.

**Files:** `tests/unit/test_llm_extract.sh:14-36`, `tests/unit/test_llm_streaming.sh:18-63`

**Fix:** Refactor both files to source `test_helpers.sh` and use standard assertion functions.

---

### R13 — Dead Code: `_NL_CONTEXT_WARN_TOKENS` (P2) — **FIXED**

**Problem:** `_NL_CONTEXT_WARN_TOKENS` is defined in `llm_config.sh:20` with env override `NL_CONTEXT_WARN_TOKENS` and listed in `llm_assistant.sh:14` dependency header, but is never actually referenced in any code. The actual overflow protection uses `_llm_context_limit()` with 80% threshold trimming.

**File:** `src/lib/llm_config.sh:20`

**Fix:** Remove the unused constant:
```bash
# Remove: readonly _NL_CONTEXT_WARN_TOKENS="${NL_CONTEXT_WARN_TOKENS:-100000}"
```
Also remove from `llm_assistant.sh:14` dependency comment.

---

### R14 — Reflexion Output Not Sanitized for Prompt Injection (P2) — **FIXED**

**Problem:** When reflexion triggers, command stderr/stdout output is truncated to `_NL_OUTPUT_TRUNCATE_CHARS` (4000) and fed back to the LLM as context. This output passes through `_mask_sensitive_output()` for PII but NOT through `_sanitize_for_prompt()` for injection patterns. Command output containing "SYSTEM: Ignore previous instructions" could manipulate the LLM.

**Fix:** Apply `_sanitize_for_prompt()` to command output before including in reflexion context.

---

### R15 — Gemini JSON Extraction Regex Limited to 1-Level Nesting (P2) — **FIXED**

**Problem:** `_extract_gemini_response()` uses `jq scan("\\{[^{}]*(?:\\{[^{}]*\\}[^{}]*)*\\}")` to find JSON objects in plain text when Gemini wraps responses in markdown or explanatory text. This regex only handles one level of nested braces. Responses with `{"commands": [{"args": {"key": "value"}}]}` (3 levels) won't be extracted.

**File:** `src/lib/providers/gemini.sh:311-312`

**Fix:** Use iterative deepening or a more robust extraction:
```bash
json_substr=$(printf '%s' "$raw_text" | jq -R -s 'try fromjson catch
  (split("\n") | map(try fromjson catch empty) | first) // empty')
```

---

### R16 — Ollama Streaming Not Implemented (P3)

**Problem:** All other providers have `_call_*_api_stream()` for progress dots during generation. Ollama only has non-streaming `_call_ollama_api()`. Users get no visual feedback during Ollama generation. This limitation is undocumented.

**File:** `src/lib/providers/ollama.sh`

**Fix:** Add comment documenting the limitation. Optionally implement streaming using Ollama's `/api/generate` with `stream: true`.

---

### R17 — Prompt Template Files Lack Version/Change Comments (P3)

**Problem:** The 7 files in `tools/utils/prompts/` have no comments indicating when rules were added, modified, or why. When a rule needs updating, there's no way to know its history without checking git blame.

**Fix:** Add a header comment to each template file:
```
# parameter_rules.txt — v10 (2026-03-18)
# Defines parameter formatting rules for LLM tool invocation.
```

---

### R18 — Unused Parameter in `_log_nl()` (P3)

**Problem:** Parameter `$16` (`successes_loaded`) is accepted by `_log_nl()` with comment "accepted for API compatibility but not logged". No caller uses this parameter meaningfully, and the compatibility shim has no documented consumer.

**File:** `src/lib/llm_metrics.sh:215`

**Fix:** Remove the compatibility comment and document that position 16 is reserved/unused, or remove it and update all call sites.

---

### R19 — Gemini thinkingConfig Applied to All Models (P3)

**Problem:** `thinkingConfig: {thinkingBudget: $gthink}` is always included in Gemini API requests regardless of whether the model supports thinking. For non-thinking models, this parameter is silently ignored but adds unnecessary payload. Default `_NL_GEMINI_THINKING_BUDGET=0` means thinking is disabled but the config key is still sent.

**File:** `src/lib/providers/gemini.sh:44-55,70-71,113-124,139-141`

**Fix:** Only include `thinkingConfig` when budget > 0:
```bash
$(if [[ "$_NL_GEMINI_THINKING_BUDGET" -gt 0 ]]; then
  echo ", thinkingConfig: {thinkingBudget: $gthink}"
fi)
```

---

### R20 — Tool Descriptions Not Validated Against Actual Tools (P3)

**Problem:** `tools/utils/tool_descriptions.txt` contains 77 curated descriptions. If a tool is renamed, added, or removed in `tools/current/`, descriptions become stale with no automated detection. Embedding accuracy depends on descriptions matching actual tools.

**File:** `tools/utils/tool_descriptions.txt`

**Fix:** Add validation test:
```bash
# Count tools in tools/current vs descriptions
tool_count=$(ls tools/current/cr_* | wc -l)
desc_count=$(grep -c '^cr_' tools/utils/tool_descriptions.txt)
[[ "$tool_count" -eq "$desc_count" ]] || fail "tool count mismatch"
```

---

## Table 4: Architecture Metrics

| Metric | Value |
|--------|-------|
| Total AI module lines | ~6,400 |
| Provider plugins | 5 (Anthropic, OpenAI, Gemini, Vertex, Ollama) |
| Prompt template files | 7 (externalized from llm_prompt.sh) |
| Test suites | 35 unit + 1 evaluation harness |
| v10 items completed | 24 of 25 (96%) |
| Security layers | 5 (whitelist, metachar, SQL mutation, RBAC, prompt sanitization) |
| Known open issues | 20 new + 3 carry-forward |

---

## Table 5: Files with Most Issues

| File | Issue Count | Categories |
|------|-------------|------------|
| `src/lib/llm_metrics.sh` | 5 | Concurrency, locale, dead code, debug rotation, unused param |
| `src/lib/llm_assistant.sh` | 3 | Security (sticky context), prompt budget, context warn dead code |
| `src/lib/providers/openai.sh` | 1 | Undefined variable (P1) |
| `src/lib/providers/gemini.sh` | 2 | JSON extraction, thinkingConfig |
| `tests/unit/` | 5 | Isolation, coverage gaps, assertion consistency |
| `tests/evaluation/` | 1 | Dataset coverage |
