# AI Architecture Review v16 — Recommendations

**Date:** 2026-03-23
**Reviewer:** Claude Opus 4.6
**Scope:** Full AI/NL subsystem (~6,500 lines across 14 files + 5 provider plugins)
**Previous:** v15 (2026-03-22), v14 (2026-03-21), v13 (2026-03-19 — all fixed)

---

## Maturity Assessment

| Dimension | Score | Notes |
|-----------|-------|-------|
| Security | 9.2/10 | Defense-in-depth with 5 validation gates, 362 NL-specific tests. One SQL multi-statement gap (R1). |
| Provider Architecture | 8.8/10 | 5 providers unified under single dispatcher. `flag_val` triplicated (R3). Streaming boilerplate duplicated (R4). |
| Tool Matching | 9.5/10 | Hybrid regex + semantic achieves 99-100%. Well-tested, well-cached. |
| Prompt Engineering | 8.5/10 | Unified prompt path (no more Ollama-specific branches). Dead compact prompt code remains (R5). Template overlap (R6). |
| Error Handling | 8.7/10 | Retry with exponential backoff, rate-limit detection, streaming reconnection. No execution timeout (R2). |
| Observability | 8.0/10 | Debug logging, cost tracking, metrics CSV. No CI regression testing (R14). Token validation gaps (R9). |
| Local Model (Ollama) | 8.5/10 | 90% accuracy via LoRA. Non-streaming hurts UX for larger models (R7). Training data limited (R8). |
| Code Quality | 8.3/10 | ShellCheck clean. Two large files (1,703 + 1,438 lines). Some duplication. |
| **Overall** | **8.6/10** | Up from 8.5 (v13). Security and matching are strong. Main debt is duplication and dead code. |

---

## Open Items from v14/v15

28 items remain open from v14 (10) and v15 (18). Many overlap. This review consolidates the most impactful items and adds new findings, producing a unified priority list.

---

## Recommendations

### P1 — High Priority

#### R1: SQL Multi-Statement Injection in Monitor Role
**Files:** `src/lib/llm_exec.sh:176-198`
**Issue:** `_check_sql_mutation()` checks the entire command string against mutation keywords but doesn't detect multi-statement SQL. A crafted input like `cockroach sql -e "SELECT 1; DROP TABLE users"` would pass the mutation check because `SELECT` is allowed — the `DROP` is hidden after a semicolon within the quoted SQL string.

The metachar check blocks bare semicolons in the command, but semicolons *inside quotes* (which are legitimate SQL) pass through. The mutation regex sees the whole string and does match `DROP`, so this specific example is actually caught. However, a more subtle attack could use `cockroach sql -e "SELECT * FROM t WHERE id > 0 UNION SELECT 'GRANT' FROM t"` — the keyword appears in a string literal but would trigger a false positive, while actual mutations via SQL functions or dynamic SQL could evade the regex.

**Recommendation:** Add a statement-count check for SQL commands — reject any `-e` value containing `;` that isn't inside a string literal. This is simpler than full SQL parsing and catches the most practical attack vector.

**Severity:** Medium (monitor role + SQL mutation guard provides dual protection, but defense-in-depth principle says close this gap)
**Carried from:** v15-R1

---

#### R2: No Execution Timeout for Interactive Commands
**Files:** `src/lib/llm_exec.sh:295-324`
**Issue:** `_safe_execute_command()` runs commands with no timeout. A slow query, a hung `cr_monitor --watch`, or an accidental `cr_workload` with no duration limit can block the REPL indefinitely. The user's only recourse is Ctrl-C, which may leave orphaned processes.

**Recommendation:** Wrap execution with `timeout` (GNU coreutils, available on macOS via `gtimeout`):
```bash
local _timeout="${NL_CMD_TIMEOUT:-120}"
if command -v timeout >/dev/null 2>&1; then
  timeout "$_timeout" "${_cmd_parts[@]}" 2>&1 || _cmd_rc=$?
elif command -v gtimeout >/dev/null 2>&1; then
  gtimeout "$_timeout" "${_cmd_parts[@]}" 2>&1 || _cmd_rc=$?
else
  "${_cmd_parts[@]}" 2>&1 || _cmd_rc=$?
fi
```
Exit code 124 = timeout. Feed "command timed out after ${_timeout}s" back to LLM for reflexion.

**Severity:** Medium-High (impacts interactive usability)
**Carried from:** v15-R2

---

#### R3: `flag_val` jq Function Duplicated 3x in Tool Call Converter
**Files:** `src/lib/llm_providers.sh:560-625`
**Issue:** The `flag_val` jq helper function is identically defined inside three separate jq expressions (Anthropic, OpenAI, Gemini cases of `_convert_tool_call_to_json`). Each copy is 5 lines. The surrounding jq logic also shares the same `needs_followup` and `user_message` extraction pattern.

**Recommendation:** Extract the shared jq logic into a variable:
```bash
_TOOL_CALL_FLAG_VAL='
  def flag_val:
    if .value == true then "--\(.key)"
    elif .value == false then empty
    elif (.value | tostring | test(" ")) then "--\(.key) \"\(.value | tostring)\""
    else "--\(.key) \(.value | tostring)"
    end;
'
```
Then each provider case prepends `$_TOOL_CALL_FLAG_VAL` to its jq filter. Reduces 15 duplicated lines to 5.

**Severity:** Low-Medium (maintenance risk — a fix to flag formatting must be applied 3x)
**Carried from:** v14-R3

---

#### R4: Streaming Boilerplate Duplicated Across 3 Providers
**Files:** `providers/anthropic.sh`, `providers/openai.sh`, `providers/gemini.sh`
**Issue:** All three streaming implementations share ~40-50 lines of identical boilerplate:
- Progress dot output (every 5 chunks, clear on completion)
- SSE line parsing (`while IFS= read -r line`)
- curl invocation pattern with `-sN` flags and header file
- Temp file creation and `trap` cleanup
- Token count writing at stream end

Each provider then adds ~60-80 lines of provider-specific parsing. The shared portion is ~45% of each file.

**Recommendation:** Extract a `_stream_sse_request()` helper that handles curl invocation, progress dots, temp file lifecycle, and line iteration. Each provider supplies a callback or case-statement for event-specific parsing. This is the single largest duplication in the codebase.

**Severity:** Medium (277 + 245 + 289 = 811 lines total; extracting shared code would save ~120 lines and ensure progress/cleanup fixes apply universally)
**Carried from:** v15-R4

---

### P2 — Medium Priority

#### R5: Dead Code — `_nl_build_system_prompt_compact()` (55 lines)
**Files:** `src/lib/llm_prompt.sh:1173-1229`
**Issue:** This function is never called in production. The unified prompt path (established 2026-03-21) replaced all compact prompt usage. The only reference outside the definition is a test mock in `tests/unit/test_session_init.sh:180`. The function, its helper `_nl_generate_compact_tool_listing`, and related config constants (`_NL_COMPACT_SCHEMA_LINES`, `_NL_OLLAMA_MAX_TOOL_HELP`) are dead weight.

**Recommendation:** Delete `_nl_build_system_prompt_compact()` and `_nl_generate_compact_tool_listing()`. Remove the test mock. Remove `_NL_COMPACT_SCHEMA_LINES` and `_NL_OLLAMA_MAX_TOOL_HELP` from `llm_config.sh` (they have no other callers).

**Severity:** Low (no functional impact, but 55+ lines of dead code create confusion about which prompt path is active)
**Carried from:** v15-R3

---

#### R6: Prompt Template Overlap — `hallucination_warnings.txt` vs `tool_notes.txt`
**Files:** `tools/utils/prompts/hallucination_warnings.txt`, `tools/utils/prompts/tool_notes.txt`
**Issue:** Both templates warn about flag hallucination and emphasize using exact `--help` output. The overlap wastes ~200 tokens per query across all providers. For Ollama (16K context), this is 1.2% of the budget spent on redundant instructions.

**Recommendation:** Audit both files. Merge unique content into `tool_notes.txt` (the more specific file). Delete `hallucination_warnings.txt`. Update `_nl_build_system_prompt()` to load only the merged file.

**Severity:** Low (token waste, not functional)
**Carried from:** v15-R5

---

#### R7: Ollama Streaming Support
**Files:** `src/lib/providers/ollama.sh`
**Issue:** Ollama supports streaming via `"stream": true` in the API request (returns newline-delimited JSON chunks). The current implementation uses non-streaming only. For Mistral Nemo 12B (~79s per query) or any model >5s latency, this creates a poor UX — the user sees nothing for the entire inference duration.

**Recommendation:** Add `_call_ollama_api_stream()` that:
1. Sets `"stream": true` in the request body
2. Reads newline-delimited JSON chunks (each has `{message: {content: "..."}, done: false}`)
3. Accumulates content, shows progress dots
4. Final chunk has `done: true` with token counts
5. Reconstruct response in same format as non-streaming

This is simpler than SSE parsing (no `data:` prefix, no event types — just NDJSON).

**Severity:** Medium (UX impact scales with model size; critical if 12B models are adopted)
**Carried from:** v15-R6

---

#### R8: Expand LoRA Training Data (82 → 200+ examples)
**Files:** `tools/utils/finetune/data/`
**Issue:** The validation loss curve oscillates wildly (3.3 → 0.3 → 1.3 → 0.5) due to only 8 validation examples. The model memorizes training data by iteration 300 (training loss ≈ 0). More diverse training data would smooth the curve and improve generalization, especially for:
- Complex multi-flag tools (cr_ddl_diff, cr_migrate, cr_catalog_diff)
- Positional arguments (cr_ddl_table, cr_ddl_user)
- Ambiguous queries that could map to multiple tools

**Recommendation:**
1. Run 150+ new diverse prompts through Gemini batch testing
2. Include paraphrased versions of existing prompts (vary vocabulary)
3. Include negative examples (queries that should NOT trigger any tool)
4. Target 200 train / 25 validation / 25 test split
5. Re-train with same hyperparameters; compare validation stability

**Severity:** Medium (90% → 93-95% is achievable with more data; diminishing returns past 300 examples)
**Carried from:** v15-R7

---

#### R9: Token Count Validation Gap
**Files:** `src/lib/llm_providers.sh:702-712`
**Issue:** `_write_token_counts()` validates that the first line is numeric (`grep -qE '^[0-9]+$'`) but doesn't validate the second line. If the API returns input tokens but no output tokens (e.g., streaming error mid-response), the second line could be empty or non-numeric, causing downstream cost calculation errors (division by zero in cost-per-token).

**Recommendation:** Validate both lines:
```bash
local input_count output_count
input_count=$(echo "$counts" | head -1)
output_count=$(echo "$counts" | tail -1)
if ! [[ "$input_count" =~ ^[0-9]+$ && "$output_count" =~ ^[0-9]+$ ]]; then
  printf '0\n0\n' > "${_NL_METRICS_PREFIX}_tokens"
  return
fi
```

**Severity:** Low (cost display error, not functional)
**Carried from:** v14-R7

---

#### R10: Doc RAG Title Injection
**Files:** `src/lib/llm_prompt.sh` (doc RAG retrieval function)
**Issue:** Document titles from the RAG corpus are injected into the system prompt without sanitization. If a malicious document title contained prompt injection patterns (e.g., `"IGNORE ALL PREVIOUS INSTRUCTIONS"`), it would be passed directly to the LLM.

**Recommendation:** Apply `_sanitize_for_prompt()` to doc RAG titles before injection. The function already exists and handles this pattern class.

**Severity:** Low (attack requires control over the doc corpus, which is local/trusted)
**Carried from:** v14-R1

---

#### R11: Cosine Similarity Division-by-Zero Guard
**Files:** `src/lib/llm_prompt.sh` (cosine similarity jq filter)
**Issue:** If an embedding vector has zero magnitude (all zeros), the cosine similarity calculation divides by zero. This could happen with a degenerate embedding from a failed API call that returns a zero vector instead of an error.

**Recommendation:** Add a magnitude check in the jq filter:
```jq
if $query_mag == 0 or $tool_mag == 0 then 0 else ($dot / ($query_mag * $tool_mag)) end
```

**Severity:** Low (would produce `null` from jq, caught by downstream null checks, but better to handle explicitly)
**Carried from:** v14-R2

---

#### R12: Validate Embedding Completeness on Startup
**Files:** `src/lib/llm_prompt.sh` (semantic matching initialization)
**Issue:** The system loads pre-computed embeddings from `tools/embeddings/*.json` but doesn't verify that all 77 tools are present. If a tool was added after embeddings were generated, it would have no embedding vector and never appear in semantic matches (regex would still catch it, but combined accuracy could degrade silently).

**Recommendation:** On first semantic match call, count tool entries in the embedding file and compare against the count of `cr_*` executables in the tools directory. Log a warning if mismatched:
```bash
_nl_debug "EMBEDDINGS" "Warning: embedding file has $n_embedded tools but tools dir has $n_actual (regenerate with generate_embeddings.sh)"
```

**Severity:** Low (regex fallback provides safety net; warning prevents silent degradation)
**Carried from:** v15-R10

---

#### R13: `_split_tools` Declared 4x Inside Loop Body
**Files:** `src/lib/llm_prompt.sh:343-344, 354, 366`
**Issue:** `local -a _split_tools; read -ra _split_tools <<< "$_tools"` appears 4 times inside the keyword matching loop (once per gate type + standard). Each iteration re-declares the local array.

**Recommendation:** Declare once before the loop; reuse inside:
```bash
local -a _split_tools=()
while IFS= read -r _line; do
  ...
  read -ra _split_tools <<< "$_tools"
  relevant_tools+=("${_split_tools[@]}")
  ...
done
```

**Severity:** Low (performance: negligible per-query, but cleaner code)
**Carried from:** v14-R4

---

### P3 — Low Priority / Enhancements

#### R14: NL Accuracy Regression Testing in CI
**Issue:** The 73-prompt batch test suite runs manually. There's no CI gate that catches accuracy regressions when prompt templates, tool matching, or provider code changes. A prompt template edit could silently degrade Ollama accuracy from 90% to 80%.

**Recommendation:** Add a lightweight CI job that:
1. Mocks the LLM API (returns canned responses for 10-15 canonical prompts)
2. Validates tool matching: input query → expected tool in top-7
3. Validates command construction: expected flags present
4. Runs on PR (fast, no cluster or API keys needed)

This doesn't replace the full 73-prompt live test but catches structural regressions.

**Severity:** Low (currently caught by manual testing; CI would prevent merge-time surprises)
**Carried from:** v15-R16

---

#### R15: Best-of-N Sampling for Ollama
**Issue:** Ollama at 90% accuracy means 1 in 10 commands is wrong. A simple accuracy boost without retraining: generate N=3 responses at slightly varied temperature (0.15, 0.2, 0.25), validate each against the tool's `--help` output, pick the response with the most valid flags.

**Recommendation:** Implement as opt-in (`NL_BEST_OF_N=3`). Cost: 3x latency per query. Benefit: eliminates most flag-level errors (the dominant failure mode at 90%).

**Severity:** Low (optimization; 90% may be sufficient for most users)
**Carried from:** v15-R8

---

#### R16: GBNF Grammar for Ollama JSON Output
**Issue:** Ollama supports GBNF grammars that constrain output tokens at the decoder level. A grammar that enforces valid JSON structure with known field names (`reasoning`, `commands`, `user_message`, `needs_followup`) would eliminate malformed JSON responses entirely — no retraining needed.

**Recommendation:** Create `tools/utils/roachie_response.gbnf` defining the response schema. Pass via Ollama's `grammar` parameter. Test with all 4 Ollama models.

**Severity:** Low (malformed JSON is already handled by fallback parsing; grammar prevents rather than recovers)
**Carried from:** v15-R11

---

#### R17: Persistent Learning Database Audit
**Issue:** The failures DB (`NL_FAILURES_DB`) and successes DB accumulate entries over time. There's no mechanism to audit what's in them, detect contradictory entries (same query with different outcomes), or export for analysis.

**Recommendation:** Add `/audit-learning` special command that:
1. Prints entry count, date range, and top-5 most frequent tools
2. Flags any query that appears in both success and failure DBs
3. Optionally exports to CSV for external analysis

**Severity:** Low (observability improvement)
**Carried from:** v15-R17

---

#### R18: Vertex AI Streaming Support
**Issue:** Vertex AI only supports non-streaming (`rawPredict`). Anthropic's `streamRawPredict` endpoint exists but returns SSE chunks that the current parser can't handle. This creates a "Waiting..." UX for Vertex users.

**Recommendation:** Investigate `streamRawPredict` with the Anthropic SSE format. If the chunk format matches Anthropic's standard SSE, reuse `_call_anthropic_api_stream()` with the Vertex endpoint and auth header.

**Severity:** Low (Vertex is the least-used provider; non-streaming is functional)
**Carried from:** v15-R6 (partial)

---

## Summary Table

| # | Priority | Issue | Files | Effort | Carried From |
|---|----------|-------|-------|--------|-------------|
| R1 | P1 | SQL multi-statement in monitor role | llm_exec.sh | Small | v15-R1 |
| R2 | P1 | No execution timeout | llm_exec.sh | Small | v15-R2 |
| R3 | P1 | `flag_val` triplicated in jq | llm_providers.sh | Small | v14-R3 |
| R4 | P1 | Streaming boilerplate duplication | providers/*.sh | Medium | v15-R4 |
| R5 | P2 | Dead compact prompt code | llm_prompt.sh, llm_config.sh | Small | v15-R3 |
| R6 | P2 | Template overlap (hallucination/tool_notes) | tools/utils/prompts/ | Small | v15-R5 |
| R7 | P2 | Ollama streaming support | providers/ollama.sh | Medium | v15-R6 |
| R8 | P2 | Expand LoRA training data | finetune/data/ | Medium | v15-R7 |
| R9 | P2 | Token count validation gap | llm_providers.sh | Small | v14-R7 |
| R10 | P2 | Doc RAG title injection | llm_prompt.sh | Small | v14-R1 |
| R11 | P2 | Cosine similarity div-by-zero | llm_prompt.sh | Small | v14-R2 |
| R12 | P2 | Validate embedding completeness | llm_prompt.sh | Small | v15-R10 |
| R13 | P2 | `_split_tools` declared 4x in loop | llm_prompt.sh | Small | v14-R4 |
| R14 | P3 | NL regression testing in CI | tests/, .github/ | Medium | v15-R16 |
| R15 | P3 | Best-of-N sampling for Ollama | llm_assistant.sh | Medium | v15-R8 |
| R16 | P3 | GBNF grammar for Ollama JSON | tools/utils/ | Small | v15-R11 |
| R17 | P3 | Persistent learning audit command | llm_assistant.sh | Small | v15-R17 |
| R18 | P3 | Vertex AI streaming | providers/vertex.sh | Medium | v15-R6 |

**P1:** 4 items (2 small, 1 small, 1 medium)
**P2:** 9 items (7 small, 2 medium)
**P3:** 5 items (2 small, 3 medium)

---

## Architecture Strengths (No Action Needed)

These areas are well-implemented and should be preserved as-is:

1. **Security validation pipeline** — 5-gate defense-in-depth (whitelist → metachar → SQL → existence → RBAC) with 362 tests
2. **Hybrid tool matching** — regex intent gate + semantic embeddings, 99-100% combined accuracy, version-aware resource resolution
3. **Provider abstraction** — unified dispatcher with shared request builders, centralized error detection, consistent retry logic
4. **Persistent learning** — flock-protected append-only DBs with dedup, TTL, and max-size pruning
5. **Session caching** — topology, schema context, regex keywords, and embedding provider cached per session
6. **Safe execution** — xargs-based argument parsing, no shell interpretation, PATH-scoped tool resolution
7. **Cost tracking** — millicent-precision integer arithmetic, flock-serialized accumulation, per-provider pricing tables
8. **Sensitive data masking** — regex-based redaction of emails, SSNs, credit cards, AWS keys, passwords

---

## Historical Fix Rate

| Review | Items | Fixed | Rate |
|--------|-------|-------|------|
| v10 | 25 | 25 | 100% |
| v11 | 15 | 15 | 100% |
| v12 | 15 | 13 | 87% (2 deferred by user) |
| v13 | 10 | 10 | 100% |
| v14 | 10 | 0 | Open |
| v15 | 18 | 0 | Open |
| **v16** | **18** | — | **Consolidated from v14+v15 + new** |

**Cumulative:** 93 items identified across v10-v15, 63 fixed (68%). v14-v15 items are recent (1-2 days old) and consolidated here as v16.
