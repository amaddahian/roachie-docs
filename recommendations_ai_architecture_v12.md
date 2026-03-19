# AI Architecture Review v12 (2026-03-19)

**Scope:** Comprehensive review of all 9 LLM modules (~6,400 lines), 5 provider plugins, 2 entry points, security pipeline, semantic matching, MCP integration, and test coverage. This review builds on v6-v11 (all previous items marked FIXED) and identifies **new** findings only.

**Files reviewed (14 core + 5 providers + 2 entry points + 5 test suites):**

| File | Lines | Role |
|------|-------|------|
| `src/lib/llm.sh` | 13 | Module loader |
| `src/lib/llm_config.sh` | 63 | Centralized constants |
| `src/lib/llm_init.sh` | 73 | Init/bootstrap |
| `src/lib/llm_exec.sh` | 243 | Security validation (single source of truth) |
| `src/lib/llm_metrics.sh` | 718 | Logging, cost, learning |
| `src/lib/llm_voice.sh` | 248 | Speech-to-text |
| `src/lib/llm_prompt.sh` | 1,426 | Prompt construction, tool matching, RAG |
| `src/lib/llm_assistant.sh` | 1,634 | REPL, agent loop, feedback |
| `src/lib/llm_providers.sh` | 868 | Provider dispatch, retry |
| `src/lib/providers/anthropic.sh` | 274 | Anthropic Claude API |
| `src/lib/providers/openai.sh` | 335 | OpenAI GPT/o-series API |
| `src/lib/providers/gemini.sh` | 341 | Google Gemini API |
| `src/lib/providers/vertex.sh` | 113 | Vertex AI Claude |
| `src/lib/providers/ollama.sh` | 75 | Ollama local models |

---

## Table 1: Recommendation Summary

| ID | Priority | Category | Title | Status |
|----|----------|----------|-------|--------|
| R1 | P1 | Security | Gemini API key exposed in URL query parameter | **FIXED** |
| R2 | P1 | Bug | Doc RAG makes redundant embedding API call (wasted latency) | **FIXED** |
| R3 | P1 | Bug | Metacharacter check blocks `>` and `<` in legitimate SQL comparisons | **FIXED** |
| R4 | P2 | Security | RBAC bypass for non-cr_* whitelisted commands | **FIXED** |
| R5 | P2 | Bug | `set +e` block in `_nl_try_voice_input` violates code conventions | **FIXED** |
| R6 | P2 | Performance | Cosine similarity in jq is O(n*d) per query for all 77 tools | **FIXED** |
| R7 | P2 | Robustness | No user input length validation before LLM call | **FIXED** |
| R8 | P2 | Robustness | xargs fallback degrades argument parsing on unmatched quotes | **FIXED** |
| R9 | P2 | Observability | Session cost file cleanup race on Ctrl+C during API call | **FIXED** |
| R10 | P2 | Maintenance | Model pricing table requires manual updates with no staleness detection | **FIXED** |
| R11 | P3 | Architecture | System prompt token budget still unbounded (carry-forward from v9-R2) | Deferred |
| R12 | P3 | Architecture | Parallel tool execution still sequential (carry-forward from v9-R8) | Deferred |
| R13 | P3 | Performance | Conversation history truncation loses context without summarization | **FIXED** |
| R14 | P3 | Maintenance | Hardcoded capability count "74 cr_* analysis tools" in session banner | **FIXED** |
| R15 | P3 | Code Quality | `_confirm` function dependency not documented in llm_exec.sh header | **FIXED** |

---

## Detailed Findings

### R1 (P1 / Security): Gemini API key exposed in URL query parameter

**File:** `src/lib/llm_prompt.sh:154`

```bash
curl -s --max-time 10 "https://generativelanguage.googleapis.com/v1beta/models/gemini-embedding-001:embedContent?key=${GEMINI_API_KEY}" \
```

The Gemini embedding API key is passed as a URL query parameter (`?key=...`). This means:
1. The full URL including the API key appears in `ps` process listing
2. The key may be logged by HTTP proxies, load balancers, or network monitoring tools
3. All other providers (Anthropic, OpenAI) correctly use header-based auth

**Contrast with secure pattern (OpenAI, line 147-148):**
```bash
curl -s --max-time 10 "https://api.openai.com/v1/embeddings" \
  -H "Authorization: Bearer ${OPENAI_API_KEY}" \
```

**Fix:** Move Gemini API key from URL to header. The Gemini API accepts `x-goog-api-key` header:
```bash
curl -s --max-time 10 "https://generativelanguage.googleapis.com/v1beta/models/gemini-embedding-001:embedContent" \
  -H "x-goog-api-key: ${GEMINI_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "..."
```

**Also affects:** Check `providers/gemini.sh` for the same pattern in the main LLM API calls.

---

### R2 (P1 / Bug): Doc RAG makes redundant embedding API call

**File:** `src/lib/llm_prompt.sh:243` and `src/lib/llm_prompt.sh:406-408,506-508`

In `_enrich_system_prompt_with_tool_help()`, the query embedding is computed and cached:

```bash
# Line 406: Compute embedding
_query_embedding=$(_embed_query "$user_query" "$embed_provider" 2>/dev/null) || true
# Line 408: Cache to file
printf '%s' "$_query_embedding" > "${_NL_METRICS_PREFIX}_query_embedding" 2>/dev/null
```

Later, `_doc_rag_retrieve()` is called (line 508) but it re-computes the embedding from scratch:

```bash
# Line 243 in _doc_rag_retrieve:
query_embedding=$(_embed_query "$query" "$provider")  # REDUNDANT API CALL
```

**Impact:** Wastes 50-500ms per query (depending on provider) and an unnecessary API call for cloud providers.

**Fix:** Pass the cached embedding to `_doc_rag_retrieve()` or have it read from the cache file:

```bash
# Option A: Pass cached embedding as $3
_doc_rag_retrieve() {
  local query="$1" provider="$2" cached_embedding="${3:-}"
  ...
  local query_embedding
  if [[ -n "$cached_embedding" ]]; then
    query_embedding="$cached_embedding"
  else
    query_embedding=$(_embed_query "$query" "$provider")
  fi
```

Then at the call site (line 508):
```bash
doc_context=$(_doc_rag_retrieve "$user_query" "$cached_embed_provider" "$_query_embedding" 2>/dev/null) || true
```

---

### R3 (P1 / Bug): Metacharacter check blocks `>` and `<` in legitimate SQL comparisons

**File:** `src/lib/llm_exec.sh:124`

```bash
"$cmd_to_run" == *">"* || "$cmd_to_run" == *"<"* || \
```

This blocks **any** occurrence of `>` or `<` in the command string, including inside SQL comparison operators. If the LLM generates:

```
cockroach sql -e "SELECT * FROM t WHERE created_at > '2026-03-01'"
run_roach_sql CA "SELECT count(*) FROM orders WHERE amount > 100"
```

These would be blocked with "command contains shell metacharacters" even though the `>` and `<` are inside quoted SQL strings and pose no shell injection risk.

**Impact:** Legitimate SQL queries with comparison operators, `BETWEEN` alternatives, and timestamp filters are silently blocked. Users see a confusing "metacharacters" error for valid queries.

**Fix:** Only block `>` and `<` when they appear outside of quoted strings, or refine the check to target actual shell redirections (e.g., `>file`, `>>`, `2>`, `<(`, `>(` which are already covered):

```bash
# Option A: Only block redirect patterns, not bare comparison operators
# Already covered: <( >( (process substitution) at line 123
# Add targeted redirect patterns instead of blanket > <
"$cmd_to_run" == *">>"* || "$cmd_to_run" == *" >"* || "$cmd_to_run" == *"1>"* || "$cmd_to_run" == *"2>"* || \
"$cmd_to_run" == *"< "* || \
```

**Note:** The `>` and `<` inside cr_* tool arguments (e.g., `cr_query_stats --since ">2026-03-01"`) are less likely since cr_* tools handle SQL internally. But `cockroach sql -e "..."` and `run_roach_sql` are whitelisted and commonly use comparison operators. Consider restricting `>` / `<` blocking to only the shell-dangerous patterns (start of redirect: space before `>`, `>>`).

---

### R4 (P2 / Security): RBAC bypass for non-cr_* whitelisted commands

**File:** `src/lib/llm_exec.sh:33`

```bash
# Non-cr_* commands (cockroach sql, etc.) follow existing whitelist -- RBAC only gates cr_* tools
[[ "$tool" != cr_* ]] && return 0
```

The RBAC system intentionally skips non-cr_* commands. This means a `monitor`-role user can run:
- `cockroach sql -e "SELECT ..."` -- any read query
- `cockroach workload run ...` -- load generation
- `cockroach node drain ...` -- node operations
- `cockroach debug ...` -- debug commands

While the SQL mutation check (`_check_sql_mutation`) blocks destructive SQL for `cockroach sql` and `run_roach_sql`, other whitelisted prefixes (`cockroach workload`, `cockroach node`, `cockroach debug`) have **no RBAC gating**.

**Impact:** A `monitor`-role user could run `cockroach node drain 1` or `cockroach workload run` through the NL interface, which contradicts the principle of least privilege.

**Fix:** Extend RBAC to cover non-cr_* whitelisted commands:

```bash
_check_rbac() {
  local tool="$1"
  local role="${_NL_ROLE:-admin}"
  [[ "$role" == "admin" ]] && return 0

  # Gate cr_* tools (existing logic)
  if [[ "$tool" == cr_* ]]; then
    # ... existing cr_* RBAC logic ...
  fi

  # Gate non-cr_* commands by role
  case "$role" in
    monitor)
      # Monitor can only run cockroach sql (read-only, enforced by SQL mutation check)
      [[ "$tool" == "cockroach" ]] && {
        local cmd_prefix="${1% *}"  # Would need full command context
        # Block: cockroach workload, cockroach node, cockroach debug
      }
      ;;
    analyst)
      # Analyst cannot run cockroach workload (load generation)
      ;;
  esac
  return 0
}
```

**Alternative:** Add the non-cr_* commands to per-role deny lists. This requires passing the full command (not just base) to `_check_rbac`.

---

### R5 (P2 / Bug): `set +e` block in `_nl_try_voice_input` violates code conventions

**File:** `src/lib/llm_assistant.sh:34-37`

```bash
_nl_try_voice_input() {
  local duration="${1:-}"
  local voice_result voice_rc
  set +e                                    # <-- violates convention
  voice_result=$(_get_voice_input $duration)
  voice_rc=$?
  set -e                                    # <-- violates convention
```

The CLAUDE.md code conventions explicitly state: "use `|| true` or `|| { fallback }` patterns, not broad `set +e` blocks."

**Fix:**
```bash
_nl_try_voice_input() {
  local duration="${1:-}"
  local voice_result="" voice_rc=0
  voice_result=$(_get_voice_input $duration) || voice_rc=$?
  if [[ $voice_rc -ne 0 || -z "$voice_result" ]]; then
```

---

### R6 (P2 / Performance): Cosine similarity in jq is O(n*d) per query for all 77 tools

**File:** `src/lib/llm_prompt.sh:203-212`

The cosine similarity computation runs in jq against all 77 tool embeddings. For Gemini's 3072-dimensional vectors, this is 77 * 3072 = ~237K floating-point multiplications plus magnitude calculations, all in jq (an interpreted JSON processor).

```bash
jq -r --argjson qe "$query_embedding" '
  .tools | to_entries[] |
  .key as $name |
  .value.embedding as $te |
  ([ range(0; $qe | length) ] | map($qe[.] * $te[.]) | add) as $dot |
  ([ $qe[] | . * . ] | add | sqrt) as $mag_q |
  ([ $te[] | . * . ] | add | sqrt) as $mag_t |
  ($dot / ($mag_q * $mag_t)) as $sim |
  "\($sim)\t\($name)"
' "$embed_file"
```

**Measured impact:** Gemini embeddings file is 236K lines (4.9 MB). Loading and computing 77 cosine similarities in jq takes 200-800ms depending on hardware — comparable to the API call itself.

**Fix options (in order of complexity):**

1. **Pre-compute tool magnitudes** (easy, ~30% speedup): Store `magnitude` alongside each tool embedding in the JSON. Skip `sqrt(sum(t[i]^2))` at query time.

2. **Use awk instead of jq** (moderate, ~3-5x speedup): awk handles floating-point arithmetic much faster than jq. Read the embedding file once, compute dot products in awk.

3. **Pre-compute a compact similarity index** (advanced): For each provider, pre-compute a binary format or reduced-dimension index. Use Python/NumPy at embed-time to generate a compact lookup table.

**Recommended:** Option 1 (pre-compute magnitudes) gives the best effort-to-speedup ratio. Modify `generate_embeddings.sh` to store magnitudes alongside embeddings.

---

### R7 (P2 / Robustness): No user input length validation before LLM call

**File:** `src/lib/llm_assistant.sh:1593`

User input from the REPL (`read -e -r user_input`) has no length validation. An extremely long input (e.g., pasted multi-KB text) would:
1. Be embedded in the system prompt + messages JSON
2. Potentially exceed the provider's context window
3. Cause expensive API calls that return errors
4. Consume the conversation history budget

**Fix:** Add a length guard after reading input:

```bash
# After read -e -r user_input (line 1593)
if [[ ${#user_input} -gt 2000 ]]; then
  _printc "$YELLOW_BOLD" "Input too long (${#user_input} chars, max 2000). Please shorten your query."
  continue
fi
```

The limit of 2000 chars (~500 tokens) is generous for natural language queries while preventing accidental paste floods.

---

### R8 (P2 / Robustness): xargs fallback degrades argument parsing on unmatched quotes

**File:** `src/lib/llm_exec.sh:229-231`

```bash
# Fallback to simple word splitting if xargs fails (e.g., unmatched quotes)
if [[ ${#_cmd_parts[@]} -eq 0 ]]; then
  read -ra _cmd_parts <<< "$cmd_to_run"
fi
```

If xargs fails (exit code from `2>/dev/null`), the fallback uses `read -ra` which splits on whitespace without respecting quotes. A command like:

```
cr_query_stats -d mydb --since "2026-03-10 14:00:00"
```

Would be split into `[cr_query_stats, -d, mydb, --since, "2026-03-10, 14:00:00"]` (6 parts instead of 5), breaking the timestamp argument.

**Impact:** Low probability (xargs handles most quote styles), but when triggered, the command receives wrong arguments.

**Fix:** Log a warning when falling back, and consider rejecting the command:

```bash
if [[ ${#_cmd_parts[@]} -eq 0 ]]; then
  _nl_debug "EXEC" "xargs failed to parse command, falling back to word splitting: $cmd_to_run"
  # Attempt word splitting but warn
  read -ra _cmd_parts <<< "$cmd_to_run"
  if [[ ${#_cmd_parts[@]} -eq 0 ]]; then
    echo "Error: could not parse command arguments" >&2
    return 1
  fi
fi
```

---

### R9 (P2 / Observability): Session cost file cleanup race on Ctrl+C during API call

**File:** `src/lib/llm_assistant.sh:1585-1587`

```bash
local _nl_cleanup_cmd='rm -f "${_NL_METRICS_PREFIX}"_* "$_NL_SESSION_COST_FILE" 2>/dev/null'
trap "$_nl_cleanup_cmd" RETURN
trap "$_nl_cleanup_cmd; exit 130" INT TERM
```

If the user presses Ctrl+C while a background API call or cost-write subshell is in progress, the trap deletes the cost file while a subshell may still be trying to write to it. The `_nl_add_session_cost` function uses `flock`, but `flock` on a deleted file creates a new inode — the write succeeds but is immediately orphaned.

**Impact:** Minor (session cost display may show stale value on exit), but could cause confusing "file not found" errors in debug logs.

**Fix:** Wait for background jobs before cleanup:

```bash
trap 'wait 2>/dev/null; rm -f "${_NL_METRICS_PREFIX}"_* "$_NL_SESSION_COST_FILE" 2>/dev/null; exit 130' INT TERM
```

---

### R10 (P2 / Maintenance): Model pricing table requires manual updates with no staleness detection

**File:** `src/lib/llm_metrics.sh:30-63`

The pricing table (`_llm_cost_per_1m_tokens`) is hardcoded with 20+ model entries and a comment "Last verified: 2026-03-14." There is no mechanism to:
1. Detect when new models are used that aren't in the table (falls through to `*) echo "0 0"`)
2. Alert when the table may be stale
3. Validate that pricing has been updated after adding new models

**Impact:** New models silently report $0.00 cost. Users relying on cost tracking get inaccurate data.

**Fix:** Log a warning when falling through to the default case:

```bash
*)
  _nl_debug "COST" "Unknown model for pricing: '$model' -- reporting $0 cost"
  echo "0 0"
  ;;
```

And add a comment with the date format for future maintainers:

```bash
# MAINTENANCE: Update when adding new models or when providers change pricing.
# Last verified: YYYY-MM-DD. Verify at:
#   Anthropic: https://docs.anthropic.com/en/docs/about-claude/pricing
#   OpenAI: https://openai.com/api/pricing
#   Google: https://ai.google.dev/pricing
```

---

### R11 (P3 / Architecture): System prompt token budget still unbounded (carry-forward)

**Original:** v9-R2 (P0, deferred by user)

The base system prompt is ~10,750 tokens before enrichment. With tool help (7 tools * ~300 tokens), doc RAG (3 chunks * ~350 tokens), persistent learning (15+10 entries), and conversation history, a single query can consume 15,000-20,000 tokens of context.

For Ollama (4096 context default), this means the system prompt alone exceeds the context window. The compact prompt helps but is still not budget-aware.

**Recommendation:** Track this as ongoing technical debt. Consider implementing a token budget checker that measures the system prompt before the API call and truncates sections (learning data first, then doc RAG, then tool help) to fit within a target budget.

---

### R12 (P3 / Architecture): Parallel tool execution still sequential (carry-forward)

**Original:** v9-R8 (P2, deferred by user)

When the LLM returns multiple independent commands (e.g., `cr_tables -d db1` and `cr_size -d db1`), they execute sequentially. Independent commands could run in parallel, reducing execution time for multi-command responses.

**Recommendation:** Track as enhancement. Would require the LLM to indicate command dependencies, or use a heuristic (read-only commands with different tools are independent).

---

### R13 (P3 / Performance): Conversation history truncation loses context without summarization

**File:** `src/lib/llm_assistant.sh:1558-1563`

When conversation history exceeds `max_history_turns * 2` messages, oldest messages are dropped:

```bash
_nl_messages_json=$(printf '%s' "$_nl_messages_json" | jq -c ".[-${_max_messages}:]")
```

This is a hard truncation with no summarization. In a long DBA session where the user investigated an issue across many queries, early diagnostic context is lost.

**Recommendation:** Consider a lightweight summarization approach: when dropping messages, extract key facts (databases mentioned, tools used, issues found) and inject them as a "session context" prefix. This could be as simple as keeping a running list of `{tool: "cr_tables", database: "mydb", result: "50 tables"}` entries without the full conversation text.

---

### R14 (P3 / Maintenance): Hardcoded capability count in session banner

**File:** `src/lib/llm_assistant.sh:1135`

```bash
echo "Available capabilities (powered by 74 cr_* analysis tools):"
```

The count "74" is hardcoded but the actual tool count is dynamic (currently 77 cr_* tools). The correct count is already computed dynamically on line 1116:

```bash
"    Tools: v${_NL_TOOLS_VERSION:-unknown} ($(ls -1 "$tools_dir"/cr_* 2>/dev/null | wc -l | tr -d ' ') cr_* tools)"
```

**Fix:** Use the dynamic count or remove the specific number:

```bash
echo "Available capabilities (powered by cr_* analysis tools):"
```

---

### R15 (P3 / Code Quality): `_confirm` function dependency not documented in llm_exec.sh header

**File:** `src/lib/llm_assistant.sh:934`

The `_nl_execute_commands()` function calls `_confirm` (line 934), which is defined in `bin/roachman`. The dependency header for `llm_assistant.sh` (lines 6-10) lists `_printc`, `_lower`, `_get_voice_input`, etc., but does not mention `_confirm`.

Similarly, `llm_exec.sh` lists dependencies on `_printc` (line 8) but `_confirm` is called from `llm_assistant.sh` which sources `llm_exec.sh` indirectly.

**Fix:** Add `_confirm` to the dependency header of `llm_assistant.sh`:

```bash
#   Functions: _printc, _lower, _confirm, _get_voice_input (from llm_voice.sh),
```

---

## Architecture Assessment

### Table 2: Architecture Scorecard

| Dimension | Score | Notes |
|-----------|-------|-------|
| **Modularity** | 9/10 | Clean 9-module decomposition with dependency headers. Single improvement: llm_prompt.sh at 1,426 lines could split tool matching into its own module. |
| **Security** | 8/10 | 5-layer validation pipeline is excellent. Deductions: Gemini API key in URL (R1), RBAC gap for non-cr_* commands (R4), `>/<` false positives (R3). |
| **Provider Abstraction** | 9/10 | Clean plugin architecture. All 5 providers follow consistent patterns. Native tool calling works across Anthropic/OpenAI/Gemini. |
| **Observability** | 9/10 | 19-field CSV metrics, JSONL feedback, trace IDs, debug logging with rotation. Cost tracking is well-engineered (integer millicents, flock-serialized). |
| **Performance** | 7/10 | Smart caching (session-scoped keywords, topology, schema). Deductions: redundant doc RAG embedding (R2), jq cosine similarity overhead (R6), no token budget management (R11). |
| **Test Coverage** | 8/10 | 83+ security tests, 135-query accuracy harness. Provider-specific streaming and error parsing could use more unit tests. |
| **Resilience** | 8/10 | Graceful degradation (embedding -> regex -> none), retry with backoff, reflexion loop. Deductions: xargs fallback (R8), cleanup race (R9). |

### Table 3: Module Size and Complexity

| Module | Lines | Functions | Complexity | Recommendation |
|--------|-------|-----------|------------|----------------|
| llm_assistant.sh | 1,634 | 8 major | High | Consider extracting `_nl_process_query` (365 lines) into phases |
| llm_prompt.sh | 1,426 | 30+ | High | Consider extracting semantic matching (lines 94-525) to `llm_matching.sh` |
| llm_providers.sh | 868 | 15 | Medium | Well-structured with provider plugins |
| llm_metrics.sh | 718 | 18 | Medium | Clean, focused responsibilities |
| llm_exec.sh | 243 | 6 | Low | Excellent single-source-of-truth design |
| llm_voice.sh | 248 | 5 | Low | Clean, minimal |
| llm_config.sh | 63 | 2 | Low | All constants env-overridable |
| llm_init.sh | 73 | 2 | Low | Focused bootstrap |
| llm.sh | 13 | 0 | Low | Pure import |

### Table 4: Security Validation Pipeline (Current State)

| Layer | Function | What It Blocks | Test Coverage |
|-------|----------|----------------|---------------|
| L1: Whitelist | `_validate_cmd_whitelist()` | Non-whitelisted commands, path traversal | test_nl_whitelist.sh |
| L2: Metachar | `_check_metachar()` | `;` `\|` `&&` `\|\|` `` ` `` `$(` `$((` `<(` `>(` `>` `<` `${` `\n` `\r` `$'` | test_nl_security.sh |
| L3: SQL Guard | `_check_sql_mutation()` | DROP DELETE TRUNCATE UPDATE INSERT ALTER GRANT REVOKE; `-f`/`--file` | test_nl_security.sh |
| L4: Tool Check | inline in `_validate_command()` | Non-existent cr_* tools | test_nl_pipeline.sh |
| L5: RBAC | `_check_rbac()` | Unauthorized cr_* tools by role | test_nl_rbac_e2e.sh |
| L6: Sanitize | `_sanitize_for_prompt()` | Prompt injection in reflexion output | test_nl_prompt_injection.sh |
| L7: Execution | `_safe_execute_command()` | Shell interpretation (array-based invocation) | test_nl_pipeline.sh |

### What's Working Well

1. **Defense in depth** -- 7 security layers, each independently testable
2. **Provider plugin architecture** -- Adding a new LLM provider requires only a new file in `providers/`
3. **Combined matching** -- Regex intent gate + semantic similarity achieves 99-100% accuracy
4. **Persistent learning** -- Cross-session failure/success database improves over time
5. **Cost tracking** -- Integer millicents with flock serialization prevents drift
6. **Reflexion loop** -- Self-correction on command failure is a sophisticated agent pattern
7. **Trace IDs** -- 8-char hex IDs correlate queries through the full pipeline

### Remaining Technical Debt (Carry-Forward Summary)

| Item | Original | Priority | Description |
|------|----------|----------|-------------|
| v9-R2 | 2026-03-18 | P0 (deferred) | System prompt token budget management |
| v9-R8 | 2026-03-18 | P2 (deferred) | Parallel tool execution |
| v10-R25 | 2026-03-18 | P3 | Template file parameter rule duplication |

---

## Implementation Priority

**Fix first (P1 -- bugs and security):**
1. R1: Move Gemini API key to header (security, 5-minute fix)
2. R2: Pass cached embedding to doc RAG (performance bug, 10-minute fix)
3. R3: Refine `>` / `<` metacharacter blocking (functional bug, 15-minute fix with test updates)

**Fix next (P2 -- robustness and quality):**
4. R4: Extend RBAC to non-cr_* commands (security improvement)
5. R5: Replace `set +e` with `|| true` pattern (code convention)
6. R7: Add input length validation (robustness)
7. R8: Improve xargs fallback logging (debugging)
8. R10: Add unknown model warning to pricing table (maintenance)
9. R9: Fix cleanup race on Ctrl+C (reliability)
10. R6: Pre-compute embedding magnitudes (performance)

**Track as tech debt (P3):**
11-15. R11-R15 (architecture, maintenance, code quality)
