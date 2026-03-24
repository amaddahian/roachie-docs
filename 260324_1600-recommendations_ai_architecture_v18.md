# AI Architecture Review v18 — Recommendations

**Date:** 2026-03-24
**Reviewer:** Claude Opus 4.6
**Scope:** Full AI/NL subsystem — fresh review after v17 fixes (15/18 items resolved)
**Previous:** v17 (2026-03-24, 15/18 fixed), v16 (2026-03-23), v13 (2026-03-19 — all fixed)

---

## Maturity Assessment

| Dimension | Score | v17 | Delta | Notes |
|-----------|-------|-----|-------|-------|
| Security | 9.2/10 | 9.0 | +0.2 | v17 fixes improved: div-by-zero, malformed JSON, path traversal, execution timeout. |
| Provider Architecture | 9.0/10 | 8.8 | +0.2 | `flag_val` deduplicated. Shared request builders. |
| Tool Matching | 9.5/10 | 9.3 | +0.2 | 3 missing tools added to regex_keywords. Template contradiction fixed. |
| Prompt Engineering | 8.2/10 | 8.5 | -0.3 | New few-shot examples for cr_ddl_diff/cr_catalog_diff are incorrect (R1). |
| Error Handling | 8.8/10 | 8.5 | +0.3 | Execution timeout added, token validation improved. |
| Observability | 8.2/10 | 8.0 | +0.2 | Corruption logging added to persistent DBs. |
| Local Model (Ollama) | 8.5/10 | 8.5 | — | 90% accuracy via LoRA. |
| Test Coverage | 7.8/10 | 7.5 | +0.3 | Success persistence tests, SQL mutation edge cases added. |
| Code Quality | 8.5/10 | 8.3 | +0.2 | Agent loop history capped. Temp file cleanup on startup. |
| **Overall** | **8.6/10** | **8.5** | **+0.1** | Steady improvement. One regression in prompt engineering (R1). |

---

## Recommendations

### P1 — Critical (1 item)

#### R1: Few-Shot Examples for cr_ddl_diff and cr_catalog_diff Are Incorrect
**Files:** `tools/utils/prompts/few_shot_examples.txt:12-13`
**Issue:** The examples added in the v17 P3 fix (R15) directly contradict `parameter_rules.txt`:

**Line 12 (cr_ddl_diff):**
```
cr_ddl_diff --source-host H1 --source-port P1 --target-host H2 --target-port P2 --source-db DB --target-db DB -t TENANT --insecure
```
- Uses `-t TENANT` — parameter_rules line 52 says: "NEVER EVER use -h, -p, -t, -d shorthand flags"
- Uses `--insecure` — parameter_rules line 57 says: "ALWAYS use --source-insecure AND --target-insecure (not just --insecure)"
- Missing `--source-tenant` and `--target-tenant` — parameter_rules line 53 says: "ALWAYS include"

**Line 13 (cr_catalog_diff):**
```
cr_catalog_diff --source-host H1 --source-port P1 --target-host H2 --target-port P2 -t TENANT --insecure
```
- Uses `-t TENANT` — parameter_rules line 27 says: "cr_catalog_diff does NOT support --source-tenant or --target-tenant" (and certainly not `-t`)
- Uses `--insecure` — should be `--source-insecure --target-insecure`
- Missing `--source-db` and `--target-db` which are required

**Fix:**
```
- DDL diff: cr_ddl_diff --source-host H1 --source-port P1 --source-tenant T --source-db DB --target-host H2 --target-port P2 --target-tenant T --target-db DB --source-insecure --target-insecure
- Catalog diff: cr_catalog_diff --source-host H1 --source-port P1 --source-db DB --target-host H2 --target-port P2 --target-db DB --source-insecure --target-insecure
```

**Severity:** Critical — few-shot examples are weighted heavily by LLMs and will override the parameter rules, causing wrong commands for these tools
**Regression from:** v17-R15

---

### P2 — High Priority (2 items)

#### R2: Command Count Validation Missing Numeric Check
**Files:** `src/lib/llm_assistant.sh:1401-1409`
**Issue:** `cmd_count` is extracted via jq but not validated as numeric:
```bash
cmd_count=$(echo "$_NL_RESPONSE" | jq -r '.commands | length' 2>/dev/null || echo "0")
if [[ "$cmd_count" == "0" || "$cmd_count" == "null" ]]; then
```
If `.commands` is not an array (e.g., a string or object from a malformed LLM response), jq returns `null`. The comparison `"null" == "0"` is false, so the query is misclassified as `multi-command` instead of `informational`. This affects complexity labeling in metrics.

**Fix:**
```bash
cmd_count=$(echo "$_NL_RESPONSE" | jq -r '.commands | if type == "array" then length else 0 end' 2>/dev/null || echo "0")
[[ "$cmd_count" =~ ^[0-9]+$ ]] || cmd_count=0
```

**Severity:** High — affects metrics accuracy and complexity classification
**New in v18**

---

#### R3: Feedback Collection `read` Without Timeout
**Files:** `src/lib/llm_assistant.sh:1034, 1065`
**Issue:** Feedback collection uses `read -r -n 1` (line 1034) and `read -r` (line 1065) without timeouts. If the user walks away during feedback, the REPL hangs indefinitely. While feedback mode is opt-in (`feedback_mode == "true"`), it's still a usability issue.

**Fix:** Add timeout to both reads:
```bash
read -r -t 30 -n 1 feedback_response    # Line 1034
read -r -t 30 user_comments             # Line 1065
```
If timeout expires, treat as "skip."

**Severity:** High — REPL hangs in feedback mode
**New in v18**

---

### P3 — Medium Priority (3 items)

#### R4: Reflexion Prompt Embeds Unsanitized Command Names
**Files:** `src/lib/llm_assistant.sh:1490-1502`
**Issue:** The reflexion prompt embeds `${_NL_EXECUTED_CMDS}` and `${followup_output}` directly. While `followup_output` is sanitized via `_sanitize_for_prompt()`, `_NL_EXECUTED_CMDS` is not. It contains tool names and flags that passed validation, so the risk is low — but for defense-in-depth, sanitization is appropriate.

**Fix:**
```bash
local _safe_cmds
_safe_cmds=$(_sanitize_for_prompt "${_NL_EXECUTED_CMDS}" 300)
```
Then use `$_safe_cmds` in the reflexion prompt.

**Severity:** Medium — defense-in-depth
**New in v18**

---

#### R5: Hardcoded Schema Fetch Timeout
**Files:** `src/lib/llm_prompt.sh:1001, 1019, 1074`
**Issue:** Schema fetching queries use `timeout 5` without configurability. On slow networks or large clusters, these may timeout prematurely, silently dropping schema context.

**Fix:** Use `${NL_SCHEMA_TIMEOUT:-5}`.

**Severity:** Medium — affects schema context availability on slow networks
**New in v18**

---

#### R6: Anthropic Streaming Continues After Partial Tool Args
**Files:** `src/lib/providers/anthropic.sh:159-162`
**Issue:** When tool arguments are partially received (connection drop), `had_error` is set but streaming continues. The tool call is added to the response with empty args `{}`. Downstream validation will catch the invalid command, but it wastes an API call and a reflexion iteration.

**Fix:** After detecting partial args, set a flag to skip adding the tool block:
```bash
if [[ -z "$parsed_args" ]]; then
    _nl_debug "STREAM" "partial tool args for $_cur_tool_name — skipping"
    parsed_args='{}'
    had_error="partial tool arguments for $_cur_tool_name"
    _cur_tool_id=""  # prevent adding to _tool_blocks
    continue
fi
```

**Severity:** Medium — wastes one agent loop iteration on invalid tool call
**New in v18**

---

## v17 Remaining Open Items

| v17 Item | Status |
|----------|--------|
| R8: Agent loop e2e tests | Open (test infrastructure) |
| R9: Session init tests | Open (test infrastructure) |
| R10: History truncation tests | Open (test infrastructure) |
| R17: Provider-specific error tests | Open (test fixtures) |

---

## False Positives Rejected

Several agent findings were investigated and rejected:

1. **"Unsafe jq field merging via `($tlf)`"** — `$tlf` is passed via `--arg` which ensures it's a plain string in jq. Computed keys from `--arg` values are safe; jq doesn't interpret them as expressions.

2. **"Gemini SSE buffer unbounded"** — The check `${#_sse_buffer} -lt 4096` is on total buffer size, not per-line. Once the buffer reaches 4096 chars, no more is appended. The cap works correctly.

3. **"Schema sanitization insufficient"** — The `$` character is NOT in the `tr -cd` whitelist (`A-Za-z0-9_,. -`), so `$(whoami)` would be stripped. Period and hyphen are legitimate SQL identifier characters.

4. **"History loading unbounded array growth"** — `_load_recent_failures()` already limits output to `max_entries` (15). The caller reads bounded output.

5. **"Cost calculation drift"** — awk floating-point is accurate to 15+ significant digits. For dollar amounts under $1000, rounding to 2 decimal places is exact.

6. **"TOCTOU in flock"** — The code already checks: `flock -w 5 200 || return 1`. The return code IS checked.

---

## Recommended Fix Order

**Immediate (1 item, ~5 min):**
1. R1: Fix few-shot examples for cr_ddl_diff and cr_catalog_diff

**This week (2 items, ~15 min):**
2. R2: Add numeric validation to cmd_count
3. R3: Add timeout to feedback reads

**Next sprint (3 items):**
4. R4-R6: Medium-priority items
