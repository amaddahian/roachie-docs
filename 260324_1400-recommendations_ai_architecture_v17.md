# AI Architecture Review v17 — Recommendations

**Date:** 2026-03-24
**Reviewer:** Claude Opus 4.6
**Scope:** Full AI/NL subsystem (~6,500 lines across 14 files + 5 provider plugins), prompt templates, embeddings, test coverage
**Previous:** v16 (2026-03-23), v15 (2026-03-22), v14 (2026-03-21), v13 (2026-03-19 — all fixed)

---

## Maturity Assessment

| Dimension | Score | v16 | Delta | Notes |
|-----------|-------|-----|-------|-------|
| Security | 9.0/10 | 9.2 | -0.2 | New finding: cosine similarity div-by-zero (R1), path traversal via _NL_TOOLS_VERSION (R6). |
| Provider Architecture | 8.8/10 | 8.8 | — | `flag_val` still triplicated (v16-R3). Streaming boilerplate still duplicated (v16-R4). |
| Tool Matching | 9.3/10 | 9.5 | -0.2 | 3 tools missing from regex_keywords.tsv (R4). Template contradiction on cr_config_event_history (R5). |
| Prompt Engineering | 8.5/10 | 8.5 | — | Templates are consistent on flag syntax. Few-shot examples sparse for complex tools. |
| Error Handling | 8.5/10 | 8.7 | -0.2 | `_check_api_error()` logic potentially inverted (R2). No execution timeout (v16-R2). |
| Observability | 8.0/10 | 8.0 | — | Token validation gaps remain. No CI regression testing. |
| Local Model (Ollama) | 8.5/10 | 8.5 | — | 90% accuracy via LoRA. Training data still limited. |
| Test Coverage | 7.5/10 | N/A | New | 27 NL test files, ~4000+ tests. 5 critical gaps in persistence, semantic API, session init, agent loop. |
| Code Quality | 8.3/10 | 8.3 | — | ShellCheck clean. Two large files (1,703 + 1,438 lines). |
| **Overall** | **8.5/10** | **8.6** | **-0.1** | New issues found in error checking and tool matching offset continued quality. Test coverage assessed for first time. |

---

## Recommendations

### P1 — Critical (3 items)

#### R1: Cosine Similarity Division by Zero — FIXED
**Files:** `src/lib/llm_prompt.sh:206-216`, `src/lib/llm_prompt.sh:260-270`
**Issue:** Both `_semantic_match_tools()` and `_doc_rag_retrieve()` compute cosine similarity as:
```
($dot / ($mag_q * $mag_t)) as $sim
```
If either magnitude is zero (malformed embedding API response, corrupted embedding file, degenerate vector), this causes a jq division-by-zero error, silently returning no matches. Semantic matching would fail without fallback notification.

**Fix:**
```jq
(if ($mag_q == 0 or $mag_t == 0) then 0 else ($dot / ($mag_q * $mag_t)) end) as $sim
```

**Severity:** Critical — silent failure of core matching feature
**New in v17**

---

#### R2: `_check_api_error()` Logic Potentially Inverted — FIXED
**Files:** `src/lib/llm_providers.sh:727`
**Issue:** The function checks:
```bash
if ! echo "$api_response" | jq -e 'has("error")' >/dev/null 2>&1; then
    return 0  # No error
fi
```
The `!` negation means: "if response does NOT have an error field, return 0 (success)." This appears correct at first glance — but `jq -e` exits non-zero for both `false` results AND parse failures. If the response is malformed JSON (not parseable), `jq -e` fails, `!` inverts it to true, and the function returns 0 (success) — silently treating unparseable responses as valid.

A malformed API response (e.g., truncated JSON from a network error) would bypass error detection entirely and be passed to the response extraction layer, which would then fail with an opaque error.

**Fix:** Separate parse validation from error field checking:
```bash
if ! echo "$api_response" | jq -e '.' >/dev/null 2>&1; then
    # Not valid JSON — treat as error
    echo "Malformed API response (not valid JSON)"
    return 1
fi
if echo "$api_response" | jq -e 'has("error")' >/dev/null 2>&1; then
    # Has error field — extract and report
    ...
fi
return 0
```

**Severity:** Critical — malformed responses silently treated as valid
**New in v17**

---

#### R3: No Execution Timeout for Commands — FIXED
**Files:** `src/lib/llm_exec.sh:290-324`, `src/lib/llm_assistant.sh:951`
**Issue:** `_safe_execute_command()` runs `cr_*` tools with no timeout. A hung tool (e.g., `cr_watch`, slow query, `cr_workload` with no duration) blocks the REPL indefinitely. Agent loop max iterations (3) doesn't help because timeout is per-command, not per-iteration.

**Fix:** Wrap execution with `timeout`:
```bash
local _timeout="${NL_EXEC_TIMEOUT:-120}"
if command -v timeout >/dev/null 2>&1; then
    timeout "$_timeout" "${_cmd_parts[@]}" 2>&1 || _cmd_rc=$?
elif command -v gtimeout >/dev/null 2>&1; then
    gtimeout "$_timeout" "${_cmd_parts[@]}" 2>&1 || _cmd_rc=$?
else
    "${_cmd_parts[@]}" 2>&1 || _cmd_rc=$?
fi
```
Exit code 124 = timeout. Feed "command timed out after ${_timeout}s" back to LLM for reflexion.

**Severity:** Critical — blocks interactive use
**Carried from:** v16-R2

---

### P2 — High Priority (8 items)

#### R4: Three Tools Missing from `regex_keywords.tsv` — FIXED
**Files:** `tools/utils/regex_keywords.tsv`
**Issue:** `cr_create_cluster`, `cr_remove_cluster`, and `cr_upgrade_version` are present in `tool_descriptions.txt` (77 tools) and all embedding files (77 tools) but absent from `regex_keywords.tsv` (74 tools). Since regex acts as the intent gate (if regex finds nothing, semantic is skipped), queries about cluster creation/removal/upgrades may fail to match if no other tool's keywords overlap.

**Fix:** Add entries:
```tsv
cr_create_cluster	create cluster deploy provision new cluster setup
cr_remove_cluster	remove cluster destroy delete teardown decommission
cr_upgrade_version	upgrade version update rolling upgrade finalize
```

**Severity:** High — tools unreachable via primary matching path
**New in v17**

---

#### R5: cr_config_event_history Flag Contradiction Between Templates — FIXED
**Files:** `tools/utils/prompts/tool_notes.txt:59`, `tools/utils/prompts/tool_specific_rules.txt:3-4`
**Issue:** `tool_notes.txt` line 59 shows:
```
cr_config_event_history --event-type restart --limit 10 -t system --insecure
```
But `tool_specific_rules.txt` line 3 explicitly states: "cr_config_event_history: Does NOT support -t or -d."

The LLM sees both files in the prompt and receives contradictory instructions — one example uses `-t`, the rules say it's invalid. This causes unpredictable behavior depending on which template the model weights more heavily.

**Fix:** Remove `-t system` from the example in `tool_notes.txt`:
```
cr_config_event_history --event-type restart --limit 10 --insecure
```

**Severity:** High — contradictory prompt context confuses LLM
**New in v17**

---

#### R6: Path Traversal via `_NL_TOOLS_VERSION` — FIXED
**Files:** `src/lib/llm_prompt.sh:24-31`
**Issue:** `_nl_resolve_resource()` interpolates `$_NL_TOOLS_VERSION` directly into a file path:
```bash
local version_file="$utils_dir/${_NL_TOOLS_VERSION}/$filename"
```
Normally safe because `_detect_tools_directory()` sets the value via regex parsing of cluster version output. But if manually exported as `NL_TOOLS_VERSION="25.2/../../../etc"`, arbitrary files could be read.

**Fix:** Validate format:
```bash
[[ "$_NL_TOOLS_VERSION" =~ ^[0-9]+\.[0-9]+$ ]] || _NL_TOOLS_VERSION=""
```

**Severity:** High (low probability, high impact if exploited)
**New in v17**

---

#### R7: `_save_success_persistent()` Has No Test Coverage — FIXED
**Files:** `src/lib/llm_metrics.sh:595-655`
**Issue:** The symmetric `_save_failure_persistent()` function has tests in `test_llm_metrics.sh` (lines 136-165), but `_save_success_persistent()` and `_load_recent_successes()` have zero test coverage. These functions handle deduplication, timestamp filtering (30 days), and max entry limits (15) — all untested.

**Fix:** Add test cases mirroring the failure persistence tests:
- Save success with timestamp, verify dedup (same query stored once)
- Verify load returns newest-first ordering
- Test max entry limit (15)
- Test age filtering (30 days)

**Severity:** High — persistent learning depends on success tracking
**New in v17**

---

#### R8: Agent Loop End-to-End Not Tested
**Files:** `src/lib/llm_assistant.sh:1323+`
**Issue:** Individual components (call_and_parse, execute_commands, reflexion) have unit tests, but the full agent loop (`_nl_process_query()`) has no end-to-end test. Untested behaviors include:
- Iteration limit enforcement (4th iteration blocked)
- Reflexion trigger: command fails → error analysis → retry with adjusted command
- `needs_followup` state transitions across iterations
- History eviction during multi-iteration queries

**Fix:** Create `test_nl_agent_loop_e2e.sh` with mocked LLM responses covering:
- 3-iteration success path
- Reflexion-triggered retry
- Iteration limit enforcement
- `needs_followup` transitions (true → command → output → follow-up)

**Severity:** High — core execution engine untested end-to-end
**New in v17**

---

#### R9: Session Init (`_nl_session_init()`) Minimally Tested
**Files:** `src/lib/llm_assistant.sh:1138-1320`
**Issue:** Session initialization — topology fetching, topology caching, message history init, cost tracker init, metrics directory creation — has minimal test coverage. Connection failures during init, permission errors on metrics directory, and multi-node topology parsing are all untested.

**Fix:** Create `test_nl_session_init.sh` with mocked cluster topology covering:
- Topology cache hit/miss
- Connection timeout during init (graceful fallback)
- Metrics directory permission failures
- Multi-node cluster topology extraction

**Severity:** High — single point of failure for all NL sessions
**New in v17**

---

#### R10: History Truncation with Token Budget Not Tested
**Files:** `src/lib/llm_assistant.sh:~500-545`
**Issue:** Simple eviction (keep last 20 messages) is tested in `test_nl_trace_context.sh`. But token-based truncation, dropped message summary generation (`_nl_summarize_dropped_turns()`), and summary injection into the system prompt are all untested. Long conversations can exceed token budgets and cause API errors.

**Fix:** Create `test_history_truncation.sh` covering:
- 50-turn conversation → truncate to 20 messages
- Verify summary generated for dropped messages
- Test with different model token budgets

**Severity:** High — long sessions may hit silent failures
**New in v17**

---

#### R11: SQL Mutation Detection Gaps (Multi-Statement, Comments, String Literals) — FIXED
**Files:** `src/lib/llm_exec.sh:176-204`, `tests/unit/test_nl_security.sh`
**Issue:** `_check_sql_mutation()` tests cover single-line mutations well but miss:
- SQL comments hiding mutations: `SELECT 1; -- DROP TABLE users`
- Multi-line SQL: `SELECT 1\nDROP TABLE users`
- Mutations inside string literals (false positives): `SELECT 'DROP TABLE users'` — this would be blocked even though it's harmless
- Nested SQL in cockroach commands: `cockroach sql -e "DROP TABLE"` vs `-e "SELECT * FROM drop_log"`

**Fix:** Add targeted tests for these edge cases. Consider a statement-count check for `-e` values — reject any containing `;` that isn't inside a string literal.

**Severity:** High (defense-in-depth)
**Carried from:** v16-R1, expanded scope

---

### P3 — Medium Priority (7 items)

#### R12: Agent Loop Message History Grows Unbounded — FIXED
**Files:** `src/lib/llm_assistant.sh:1475-1603`
**Issue:** During agent loop iterations, `_nl_messages_json` grows by 2 messages per iteration (6 total for max 3 iterations) plus 2 more at the session level. No per-iteration trimming occurs — only at session boundary. Risk: context overflow on long agent loops with verbose command output (each output truncated to 4000 chars but still significant).

**Fix:** Add per-iteration history budget check. If messages exceed a threshold within the agent loop, drop the oldest iteration's output before appending the new one.

**Severity:** Medium
**New in v17**

---

#### R13: Persistent Learning DB Corruption Recovery — FIXED
**Files:** `src/lib/llm_metrics.sh:437-510, 595-655`
**Issue:** When loading persistent DB files, malformed JSON lines are silently skipped. If a file is corrupted (partial write, disk error), important failure/success patterns are lost without notification. The `flock` mechanism prevents concurrent write corruption, but power loss or signal interruption during write could still produce partial lines.

**Fix:** Add a validation/recovery function that rewrites the file excluding malformed entries, with a warning message:
```bash
_validate_jsonl_db() {
    local db_file="$1"
    while IFS= read -r line; do
        echo "$line" | jq -e . >/dev/null 2>&1 && echo "$line"
    done < "$db_file" > "${db_file}.clean"
    mv "${db_file}.clean" "$db_file"
}
```

**Severity:** Medium
**New in v17**

---

#### R14: Streaming Temp File Orphans on Signal — FIXED
**Files:** `src/lib/providers/anthropic.sh:51,81`, `openai.sh:64,91`, `gemini.sh:55,83`
**Issue:** Streaming functions create temp files for headers/rc with `trap ... RETURN INT TERM` cleanup. However, curl runs in a process substitution (`< <(curl ...)`), so signal delivery to the subshell is complex. Accumulated orphaned temp files in `/tmp` over multiple interrupted sessions is possible.

**Fix:** Add periodic cleanup at session level (already partially done at `llm_assistant.sh:1648`). Consider adding startup cleanup:
```bash
find /tmp -name ".roachie_metrics_*" -mtime +1 -delete 2>/dev/null
```

**Severity:** Medium
**New in v17**

---

#### R15: Few-Shot Examples Missing Complex Tools — FIXED
**Files:** `tools/utils/prompts/few_shot_examples.txt`
**Issue:** Only 10 examples in 12 lines. Complex tools with dual-parameter design (`cr_ddl_diff`, `cr_migrate`, `cr_catalog_diff`) are not represented. These are the tools most likely to cause flag errors because they use `--source-host`/`--target-host` instead of the standard `-h` flag.

**Fix:** Add 2-3 examples for dual-parameter tools:
```
cr_ddl_diff --source-host 10.1.1.5 --source-port 26257 --target-host 10.1.1.6 --target-port 26257 --source-db movr --target-db movr -t system --insecure
```

**Severity:** Medium
**New in v17**

---

#### R16: Token Count Range Validation — FIXED
**Files:** `src/lib/llm_providers.sh:702-712`
**Issue:** `_write_token_counts()` validates that the first line is numeric but doesn't check for negative values or impossibly large values (>1M). If a provider returns garbage, cost calculations could be distorted.

**Fix:** Add range validation:
```bash
if [[ "$prompt_tokens" =~ ^[0-9]+$ ]] && [[ "$prompt_tokens" -le 1000000 ]]; then
    # valid
else
    prompt_tokens=0
fi
```

**Severity:** Medium
**New in v17**

---

#### R17: Provider-Specific Error Handling Tests Incomplete
**Files:** `tests/unit/test_llm_providers.sh`, `tests/unit/test_llm_streaming.sh`
**Issue:** Rate limit detection and token counting are well-tested. But provider-specific error modes are not:
- Vertex AI authentication errors (different error format)
- Ollama model not found (auto-download vs error)
- OpenAI context length exceeded
- Gemini safety/policy violations
- Network timeout vs API error distinction

**Fix:** Add `test_provider_errors_comprehensive.sh` with fixtures for each provider's unique error responses.

**Severity:** Medium
**New in v17**

---

#### R18: `flag_val` jq Function Duplicated 3x — FIXED
**Files:** `src/lib/llm_providers.sh:560-625`
**Issue:** The `flag_val` jq helper is identically defined inside three separate jq expressions (Anthropic, OpenAI, Gemini cases of `_convert_tool_call_to_json`). Each copy is 5 lines.

**Fix:** Extract into a shared variable:
```bash
_TOOL_CALL_FLAG_VAL='def flag_val: if .value == true then "--\(.key)" ...'
```
Then reference it in each jq expression.

**Severity:** Medium — maintenance burden, not a bug
**Carried from:** v16-R3

---

## Carried from v16 (Still Open)

| v16 Item | Status | v17 Item |
|----------|--------|----------|
| R1: SQL multi-statement injection | Open → expanded | R11 |
| R2: No execution timeout | Open | R3 |
| R3: `flag_val` triplicated | Open | R18 |
| R4: Streaming boilerplate duplication | Open | Deferred (P3, low risk) |
| R5: Dead compact prompt code | Open | Deferred (cleanup, no risk) |
| R6-R18: Various P2/P3 items | Open | Some consolidated above |

---

## New Findings Summary

| # | Category | Count | Source |
|---|----------|-------|--------|
| 1 | Code bugs (div-by-zero, inverted logic) | 2 | Core module review |
| 2 | Tool matching gaps (missing keywords, template contradictions) | 2 | Prompt/embedding review |
| 3 | Security (path traversal, SQL edge cases) | 2 | Core + test review |
| 4 | Test coverage gaps | 5 | Test coverage review |
| 5 | Code quality (duplication, orphans, validation) | 5 | All three reviews |

---

## Positive Findings

1. **Security fundamentals are strong** — 5-gate validation pipeline, `umask 0077` for temp files, credential masking, API keys in headers not URLs, all user input passed to jq via `--arg` (not interpolation)
2. **Flag syntax consistency** — all 8 prompt template files use space syntax (`--flag value`), no contradictory equals syntax found
3. **Embedding quality** — all 77 tools have descriptions and embeddings across 3 providers, 99-100% combined accuracy on 135-query test suite
4. **Test infrastructure mature** — 27 NL test files, mock-based (no API keys needed), fixture files for streaming, isolated suites
5. **Persistent learning architecture** — flock-based atomic writes, deduplication, timestamp filtering, age-based cleanup

---

## Recommended Fix Order

**Phase 1 (Immediate — 3 items):**
1. R1: Cosine similarity div-by-zero guard (~5 min, 2 lines per function)
2. R2: `_check_api_error()` malformed JSON handling (~15 min)
3. R4: Add 3 missing tools to regex_keywords.tsv (~5 min)

**Phase 2 (This week — 5 items):**
4. R3: Execution timeout (~30 min)
5. R5: Fix cr_config_event_history template contradiction (~5 min)
6. R6: Path traversal validation on `_NL_TOOLS_VERSION` (~5 min)
7. R7: Tests for `_save_success_persistent()` (~1 hr)
8. R11: SQL mutation edge case tests (~1 hr)

**Phase 3 (Next sprint — 8 items):**
9. R8: Agent loop e2e tests
10. R9: Session init tests
11. R10: History truncation tests
12. R12-R18: Medium-priority items
