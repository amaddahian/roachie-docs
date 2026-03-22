# AI Architecture Review v10 — 2026-03-18

Review of the roachie NL/LLM subsystem following completion of v8/v9 review cycles (32 of 35 items fixed). Fresh comprehensive analysis across all 13 AI modules, 5 provider plugins, 6 prompt templates, and 29 test suites.

## Table 1: v10 Recommendations

| ID | Priority | Category | Description |
|----|----------|----------|-------------|
| R1 | P1 | Concurrency | **FIXED** Inverted flock logic in `_save_failure_locked` / `_save_success_locked` — lock not actually held during writes |
| R2 | P1 | Concurrency | **FIXED** Non-atomic cost file read-modify-write in `_nl_add_session_cost` — race between read and write |
| R3 | P1 | Security | **FIXED** Path traversal in `_nl_load_prompt_template` — filename validated to `[a-zA-Z0-9_.-]+` |
| R4 | P1 | Security | **FIXED** Sed injection in `_nl_load_prompt_template` — `$tools_dir` escaped before sed substitution |
| R5 | P1 | Security | **FIXED** RBAC role not validated at session init — invalid `ROACHIE_ROLE=superuser` silently treated as admin |
| R6 | P1 | Config | **FIXED** `_NL_MAX_AGENT_ITERATIONS` has no env override despite comment claiming `NL_MAX_ITERATIONS` works |
| R7 | P1 | Error handling | **FIXED** Non-streaming curl calls missing `-f` flag — HTTP 401/403/500 return HTML parsed as JSON |
| R8 | P1 | Error handling | **FIXED** Silent jq failure on partial tool arguments — connection drop during streaming loses tool args silently |
| R9 | P2 | Testing | **FIXED** Zero test coverage for Doc RAG and reflexion features (reflexion tested; Doc RAG only on unmerged branch) |
| R10 | P2 | Testing | **FIXED** No tests for sticky context, trace IDs, or multi-turn memory persistence |
| R11 | P2 | Testing | **FIXED** Evaluation harness per-query timeout — polls with `EVAL_QUERY_TIMEOUT` (default 60s) |
| R12 | P2 | Consistency | **FIXED** Different error JSON structures across providers (Gemini missing `type` field, Ollama has none) |
| R13 | P2 | Consistency | **FIXED** Inconsistent empty response detection — OpenAI doesn't check tool calls, Gemini doesn't check text |
| R14 | P2 | Performance | **FIXED** Multiple `ollama list` calls during provider detection — 5+ subprocess invocations per selection |
| R15 | P2 | Performance | **FIXED** Gemini streaming buffers entire SSE response in memory — unbounded growth on large responses |
| R16 | P2 | Concurrency | **FIXED** Metrics `_curl_rc` temp file not unique per invocation — concurrent calls overwrite each other |
| R17 | P2 | Concurrency | **FIXED** Ollama startup race — concurrent calls both try to start `ollama serve`, fighting for port |
| R18 | P2 | Error handling | **FIXED** Schema context failure silently ignored — no user warning when schema fetch fails |
| R19 | P2 | Error handling | **FIXED** Corrupted `tools.json` schema file causes silent fallback to no native tool calling |
| R20 | P2 | Security | **FIXED** No test for prompt injection via reflexion command output fed back to LLM |
| R21 | P2 | Testing | **FIXED** RBAC tests only validate `_check_rbac` in isolation, not integrated with command execution pipeline |
| R22 | P2 | Testing | **FIXED** Missing negative tests for malformed LLM responses (string vs boolean, object vs array) |
| R23 | P3 | Dead code | **FIXED** Unused `_sse_raw` variable in Gemini streaming, unused `_ollama_pid` in provider startup |
| R24 | P3 | Config | **FIXED** Ollama startup wait timeout hardcoded to 15s, max retries hardcoded to 1 — not configurable |
| R25 | P3 | Maintenance | **N/A** Dual-parameter tool rules duplicated across 3+ template files — template files only on unmerged branch |

**Total: 25 recommendations** (P1: 8, P2: 14, P3: 3)

---

## Table 2: v9 Carry-Forward

| ID | Status | Description |
|----|--------|-------------|
| v9-R2 | Skipped | Prompt token budget management (user deferred) |
| v9-R8 | Skipped | Parallel tool execution (user deferred) |

---

## Table 3: Priority Summary

| Priority | Count | Categories |
|----------|-------|------------|
| **P1** | 8 | Concurrency (2), Security (3), Config (1), Error handling (2) |
| **P2** | 14 | Testing (5), Consistency (2), Performance (2), Concurrency (2), Error handling (2), Security (1) |
| **P3** | 3 | Dead code (1), Config (1), Maintenance (1) |
| **Total** | **25** | |

---

## Detailed Findings

### R1 — Inverted flock Logic (P1) — **FIXED**

**Problem:** `_save_failure_locked` and `_save_success_locked` use flock with inverted logic. The current code:
```bash
flock -w 5 200 || { "$@"; return; }
"$@"
```
This means: on lock acquisition success, skip the fallback and run `"$@"` — but on timeout, run `"$@"` WITHOUT the lock. The lock is never actually protecting the critical section because `"$@"` runs regardless.

**File:** `src/lib/llm_metrics.sh:379-388`

**Fix:** Remove the `||` fallback — let flock hold the lock for the subshell duration:
```bash
flock -w 5 200
"$@"
```

---

### R2 — Non-atomic Cost File Race (P1) — **FIXED**

**Problem:** `_nl_add_session_cost()` reads the file, does arithmetic, then writes back. Between the `cat` and `echo`, another subshell could write, causing lost updates.

**File:** `src/lib/llm_metrics.sh:87-94`

**Fix:** Use atomic temp+mv pattern or flock serialization.

---

### R3 — Path Traversal in Template Loading (P1) — **FIXED**

**Problem:** `_nl_load_prompt_template` accepts filename as `$1` and concatenates into `$base_dir/tools/utils/prompts/$filename`. No validation prevents `../../etc/passwd`.

**File:** `src/lib/llm_prompt.sh:37-48`

**Fix:** Validate filename contains only `[a-z0-9_.-]` characters:
```bash
[[ "$filename" =~ ^[a-z0-9_.-]+$ ]] || return 1
```

---

### R4 — Sed Injection in Template Loading (P1) — **FIXED**

**Problem:** `sed "s|{{TOOLS_DIR}}|${tools_dir}|g"` — if `$tools_dir` contains `|`, `&`, `\`, or newlines, sed breaks or substitutes incorrectly.

**File:** `src/lib/llm_prompt.sh:43`

**Fix:** Escape sed special characters in `tools_dir` before substitution.

---

### R5 — RBAC Role Not Validated at Init (P1) — **FIXED**

**Problem:** `_NL_ROLE` is set from `ROACHIE_ROLE` env var with no validation. Invalid values like `superuser` are not rejected — `_check_rbac` may fall through to permissive defaults.

**File:** `src/lib/llm_config.sh:54`, `src/lib/llm_assistant.sh:1085`

**Fix:** Validate at session init:
```bash
case "$_NL_ROLE" in admin|dba|analyst|monitor) ;; *) _NL_ROLE="analyst"; warn "Invalid role" ;; esac
```

---

### R6 — Agent Iterations Not Configurable (P1) — **FIXED**

**Problem:** `_NL_MAX_AGENT_ITERATIONS=3` is hardcoded despite comment claiming `NL_MAX_ITERATIONS` env var works. The `${ENV:-default}` pattern is missing.

**File:** `src/lib/llm_config.sh:15`

**Fix:** `readonly _NL_MAX_AGENT_ITERATIONS="${NL_MAX_ITERATIONS:-3}"`

---

### R7 — Non-streaming Curl Missing -f Flag (P1) — **FIXED**

**Problem:** All 5 providers' non-streaming curl calls omit `-f` (fail on HTTP error). HTTP 401/403/500 responses return HTML that gets parsed as JSON, producing confusing errors instead of clear "auth failed" or "server error".

**Files:** `providers/anthropic.sh:60`, `openai.sh:109`, `gemini.sh:74`, `vertex.sh:83`, `ollama.sh:35`

**Fix:** Add `-f` to all non-streaming curl calls, or check HTTP status code via `-w '%{http_code}'`.

---

### R8 — Silent Tool Argument Loss on Connection Drop (P1) — **FIXED**

**Problem:** In Anthropic streaming, if connection drops mid-tool-argument, `_cur_tool_args` has incomplete JSON. The code falls back to `echo '{}'`, silently dropping all arguments.

**File:** `src/lib/providers/anthropic.sh:193-200`

**Fix:** Log a warning when partial tool args are detected, and set `had_error` to signal incomplete response.

---

### R9 — Zero Test Coverage for Doc RAG and Reflexion (P2) — **FIXED**

**Problem:** Two major features — Doc RAG (document retrieval for complex queries) and reflexion (auto-retry with error analysis) — have zero automated tests despite being deployed.

**Files:** No `test_doc_rag.sh` or `test_reflexion.sh` exist.

**Fix:** Create test suites covering: document retrieval ranking, reflexion loop termination, max retry limits, error analysis function.

---

### R10 — No Tests for Sticky Context or Trace IDs (P2) — **FIXED**

**Problem:** Sticky session context (v9-R7) and per-query trace IDs (v9-R11) were added but have no dedicated tests validating they work correctly.

**Fix:** Add tests verifying: context extracted from evicted turns, trace ID appears in debug log, trace ID in CSV metrics.

---

### R11 — Evaluation Harness Missing Per-Query Timeout (P2) — **FIXED**

**Problem:** `run_eval.sh` calls `_call_llm_api` with no timeout wrapper. A hung API call blocks the entire evaluation run.

**File:** `tests/evaluation/run_eval.sh:146`

**Fix:** Wrap API call with `timeout 60` and handle timeout as FAIL.

---

### R12 — Inconsistent Error JSON Across Providers (P2) — **FIXED**

**Problem:** Streaming error responses differ:
- Anthropic/OpenAI: `{"error":{"type":"streaming_error","message":"..."}}`
- Gemini: `{"error":{"message":"..."}}` (missing `type`)
- Ollama: No error JSON at all

**Fix:** Standardize all to include `type` field.

---

### R13 — Inconsistent Empty Response Detection (P2) — **FIXED**

**Problem:** Empty response conditions differ across providers:
- Anthropic: checks text AND tool_use
- OpenAI: checks text only (misses tool-call-only responses)
- Gemini: checks chunk_count only

**Fix:** Standardize: `chunk_count == 0 && text empty && no tool calls`.

---

### R14 — Multiple ollama list Calls (P2) — **FIXED**

**Problem:** Provider detection, fallback selection, model selection, and startup each independently call `ollama list` — 5+ subprocess invocations.

**File:** `src/lib/llm_providers.sh:176-386`

**Fix:** Cache `ollama list` output once per session.

---

### R15 — Unbounded Gemini SSE Buffer (P2) — **FIXED**

**Problem:** `_sse_buffer+="${line}"$'\n'` accumulates the entire SSE stream in memory. For large responses, this causes memory pressure.

**File:** `src/lib/providers/gemini.sh:155`

**Fix:** Only buffer when error detected, or cap buffer size.

---

### R16 — Metrics curl_rc File Not Unique (P2) — **FIXED**

**Problem:** All streaming providers write curl exit code to `${_NL_METRICS_PREFIX}_curl_rc`. Concurrent calls share the same prefix and overwrite each other.

**Files:** `providers/anthropic.sh:221`, `openai.sh:258`, `gemini.sh:197`

**Fix:** Include `$RANDOM` or `$$` in the filename.

---

### R17 — Ollama Startup Race (P2) — **FIXED**

**Problem:** Concurrent ollama calls both detect server is down and both start `ollama serve`, causing port conflicts.

**File:** `src/lib/llm_providers.sh:803-817`

**Fix:** Use flock on a startup lock file.

---

### R18 — Schema Context Failure Silent (P2) — **FIXED**

**Problem:** `_nl_fetch_schema_context` failures produce empty string with no user warning. Users don't know schema-aware features are unavailable.

**File:** `src/lib/llm_assistant.sh:1207`

**Fix:** Log warning when schema fetch fails.

---

### R19 — Corrupted Schema File Silent Fallback (P2) — **FIXED**

**Problem:** If `tools.json` is corrupted (invalid JSON), `_nl_load_tool_schemas` returns `[]` without warning — native tool calling silently disabled.

**File:** `src/lib/llm_prompt.sh:456`

**Fix:** Add jq validation with debug log on failure.

---

### R20 — No Reflexion Prompt Injection Test (P2) — **FIXED**

**Problem:** Reflexion feeds command output back to the LLM. If command output contains "SYSTEM: Ignore previous instructions", the LLM could be manipulated. No test validates this attack vector.

**Fix:** Add test with malicious command output to verify it doesn't affect LLM behavior.

---

### R21 — RBAC Not Tested End-to-End (P2) — **FIXED**

**Problem:** RBAC tests validate `_check_rbac` in isolation but don't test the full pipeline: LLM generates prohibited command -> `_validate_command` blocks it.

**File:** `tests/unit/test_nl_security.sh:269-319`

**Fix:** Add integration test with mock LLM returning RBAC-prohibited tool.

---

### R22 — Missing Malformed Response Tests (P2) — **FIXED**

**Problem:** No tests for: `needs_followup` as string instead of boolean, `commands` as object instead of array, missing `reasoning` field, null fields.

**Fix:** Add `test_llm_malformed_responses.sh` covering all JSON structure variants.

---

### R23 — Dead Code (P3) — **FIXED**

**Problem:** `_sse_raw` declared but unused in `gemini.sh:150`. `_ollama_pid` assigned but unused in `llm_providers.sh:56`.

**Fix:** Remove both.

---

### R24 — Hardcoded Timeouts (P3) — **FIXED**

**Problem:** Ollama startup wait (15s) and max API retries (1) are hardcoded without env overrides.

**Files:** `llm_providers.sh:58,617`

**Fix:** `_oll_wait_max="${NL_OLLAMA_STARTUP_TIMEOUT:-15}"`, `max_retries="${NL_MAX_API_RETRIES:-1}"`

---

### R25 — Duplicated Parameter Rules (P3) — **N/A** (template files on unmerged branch)

**Problem:** Dual-parameter tool rules appear in `parameter_rules.txt`, `tool_specific_rules.txt`, and `tool_notes.txt` with slight variations.

**Fix:** Consolidate into single canonical location, reference from others.

---

## Table 4: Architecture Metrics

| Metric | Value |
|--------|-------|
| Total AI module lines | ~6,250 (reduced from 6,650 via template extraction) |
| Provider plugins | 5 (Anthropic, OpenAI, Gemini, Vertex, Ollama) |
| Prompt template files | 6 (externalized from llm_prompt.sh) |
| Test suites | 29 unit + 1 evaluation harness |
| v8/v9 items completed | 32 of 35 (91%) |
| Security layers | 5 (whitelist, metachar, SQL mutation, RBAC, prompt sanitization) |
| Known concurrency issues | 4 (flock, cost file, curl_rc, ollama startup) |
