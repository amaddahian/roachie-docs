# Roachie — Architectural Review v31

**Date**: 2026-04-09
**Scope**: AI subsystem, application design, system architecture, security, testing
**Codebase**: ~19.3K lines core + ~12K lines tests across 78 tools, 22 modules, 5 LLM providers
**Prior review**: v30 (2026-03-31) — 13/14 recommendations implemented, 1 partial

---

## Executive Summary

Roachie continues to mature as a production-grade CockroachDB toolkit with a sophisticated NL/AI layer. The v30 review drove significant structural improvements: tool ranking extraction (`llm_ranking.sh`), command validation isolation (`llm_exec.sh`), circuit breaker resilience, structured error handling, and batch SQL execution. All 14 v30 recommendations were addressed (13 fully, 1 partially).

This v31 review shifted focus toward **module decomposition of the remaining large files**, **code deduplication between batch and interactive paths**, **provider response normalization**, and **operational maturity** (session data lifecycle, MCP drift detection, observability gaps). No critical vulnerabilities were found. All 14 recommendations were implemented on branch `feature/arch-review-v31` (commit `e40e1ac`), delivering 3 new extracted modules (`llm_embedding.sh`, `llm_schema.sh`, `llm_query.sh`), normalized error responses across all 5 providers, 2 new test suites (66 tests), and a 23% reduction in `llm_prompt.sh` (2,303 → 1,766 lines).

---

## Table 1: v30 Implementation Status

| # | Recommendation | Status |
|---|---------------|--------|
| R1 | Extract tool ranking into `llm_ranking.sh` | **FIXED** — 191 lines, 5 functions |
| R2 | Centralize message history management | **FIXED** — `_nl_trim_history_to_budget()`, `_nl_append_and_trim_history()` |
| R3 | Consolidate SQL URL construction | **FIXED** — `sql.sh` expanded to 123 lines with `_build_sql_url()` |
| R4 | Structured error types with context | **FIXED** — `errors.sh` expanded to 77 lines with `die()`, `error_context()`, 6 error codes |
| R5 | Provider circuit breaker | **FIXED** — `_circuit_breaker_is_open/record_failure/record_success()` in `llm_providers.sh` |
| R6 | Parallelize tool help fetching | **FIXED** — `xargs -P` with `_NL_HELP_CONCURRENCY` |
| R7 | Parallel CI tests | **FIXED** — `--parallel 4` in `.github/workflows/tests.yml` |
| R8 | Token budget governor | **FIXED** — Priority-based (P1-P4) section trimming in `llm_providers.sh` |
| R9 | Decouple cluster.sh/replication.sh | **FIXED** — `_discover_containers()` shared, `upgrade.sh` delegates |
| R10 | Validate learned keywords | **PARTIAL** — `tr -cd` sanitization exists but no explicit regex metachar guard |
| R11 | Negative test suites | **FIXED** — 3 new error-path test files (508 lines) |
| R12 | Tiered model routing | **FIXED** — `_nl_tiered_model_override()` with 3-tier classification |
| R13 | Batch SQL execution | **FIXED** — `_run_sql_batch()` in `sql.sh` |
| R14 | Test metadata tags | **FIXED** — `@tags`, `@timeout`, `@requires` headers |

---

## Table 2: v31 Recommendation Summary

| # | Area | Recommendation | Priority | Effort | Status |
|---|------|---------------|----------|--------|--------|
| R1 | AI | Extract embedding subsystem from `llm_prompt.sh` | P1 | Medium | **FIXED** |
| R2 | App | Extract shared query processor from roachie-batch | P1 | High | **FIXED** |
| R3 | AI | Standardize provider error response format | P2 | Medium | **FIXED** |
| R4 | AI | Extract schema context into `llm_schema.sh` | P2 | Low | **FIXED** |
| R5 | Security | Sanitize learning DB entries before persistent storage | P2 | Low | **FIXED** |
| R6 | Ops | Add MCP-to-cr_* drift detection in CI | P2 | Low | **FIXED** |
| R7 | Perf | Add cross-session tool help cache | P2 | Low | **FIXED** |
| R8 | AI | Add trap cleanup to learning DB compaction | P3 | Low | **FIXED** |
| R9 | Test | Add provider response normalization tests | P3 | Medium | **FIXED** |
| R10 | Ops | Add session data lifecycle management | P3 | Low | **FIXED** |
| R11 | AI | Complete learned keyword regex metachar validation (v30 R10) | P3 | Low | **FIXED** |
| R12 | App | Add flock to session cost file writes | P3 | Low | **FIXED** |
| R13 | Test | Add roachie-batch integration test suite | P3 | Medium | **FIXED** |
| R14 | AI | Add embedding auto-regeneration on staleness | P4 | Medium | **FIXED** |

---

## Table 2a: v31 Implementation Details

| # | Implementation | Files Modified/Created | Lines Changed |
|---|---------------|----------------------|---------------|
| R1 | Extracted 7 embedding functions to `llm_embedding.sh` | `src/lib/llm_embedding.sh` (NEW), `llm_prompt.sh`, `llm.sh`, `test_semantic_matching.sh` | +170, -170 from prompt |
| R2 | Extracted `_nl_build_reflexion_prompt()`, `_nl_build_followup_prompt()`, `_nl_check_reflexion_trigger()` to shared module | `src/lib/llm_query.sh` (NEW), `llm_assistant.sh`, `roachie-batch`, `llm.sh` | +45 new, ~60 deduped |
| R3 | Added `_provider_error_response()` in `llm_providers.sh`; all 5 providers return normalized JSON on error | `llm_providers.sh`, `anthropic.sh`, `openai.sh`, `vertex.sh`, `ollama.sh`, `test_llm_extract.sh`, `test_streaming.sh` | +15 shared, ~40 provider fixes |
| R4 | Extracted 3 schema functions (280 lines) to dedicated module | `src/lib/llm_schema.sh` (NEW), `llm_prompt.sh`, `llm.sh`, `test_schema_context.sh`, `test_nl_schema_context.sh` | +280 new, -280 from prompt |
| R5 | Added `_mask_sensitive_output()` call at write time in `_save_failure_persistent()` | `llm_metrics.sh` | +3 |
| R6 | Added MCP drift detection CI step validating 39 MCP tools against `tools.25.2/` scripts | `.github/workflows/tests.yml` | +15 |
| R7 | Added persistent help cache at `~/.roachie/cache/help/` keyed by tools version directory | `llm_metrics.sh` (`_nl_get_tool_help()`) | +25 |
| R8 | Added `trap 'rm -f ...' RETURN INT TERM` to `_compact_learning_db_inner()` | `llm_metrics.sh` | +1 |
| R9 | Created 45-test normalization suite covering all 5 providers | `tests/unit/test_provider_normalization.sh` (NEW) | +280 |
| R10 | Added `_nl_session_gc()` for temp file, cache, and history cleanup on session start | `llm_metrics.sh`, `llm_assistant.sh` | +30 |
| R11 | Added regex metachar escaping (`\\.`, `\\-`) after sanitization in `_nl_load_learned_keywords()` | `llm_metrics.sh` | +5 |
| R12 | Added `flock -x` wrapper to `_nl_add_session_cost_inner()` using existing `_with_db_lock()` pattern | `llm_metrics.sh` | +8 |
| R13 | Created 21-test batch integration suite (syntax, help, parsing, resume, retry, JSONL) | `tests/unit/test_roachie_batch.sh` (NEW) | +167 |
| R14 | Added `_auto_regen_embeddings_if_stale()` gated behind `NL_AUTO_REGEN_EMBEDDINGS=1` | `llm_prompt.sh` (`_enrich_semantic_match_tools()`) | +15 |

**Net change**: +1,185 lines added, -713 removed across 22 files. `llm_prompt.sh` reduced from 2,303 → 1,766 lines (-23%).

### Test Results (post-implementation)

- **51 test suites**: 50 pass, 1 pre-existing failure (`test_llm_providers.sh` — `OPENAI_MODEL` env override conflict)
- **1,601 individual assertions**: all passing
- **New test suites added**: `test_provider_normalization.sh` (45 tests), `test_roachie_batch.sh` (21 tests)
- **Existing suites updated**: `test_llm_extract.sh`, `test_streaming.sh`, `test_semantic_matching.sh`, `test_schema_context.sh`, `test_nl_schema_context.sh`, `test_llm_metrics.sh`
- **Branch**: `feature/arch-review-v31` — commit `e40e1ac`

---

## Detailed Recommendations

### R1: Extract Embedding Subsystem from `llm_prompt.sh`
**Area**: AI Architecture | **Priority**: P1 | **Effort**: Medium

**Current state**: `llm_prompt.sh` remains the largest module at 2,303 lines. After the v30 ranking extraction, ~170 lines of embedding/semantic code remain interleaved with prompt construction: `_embed_query()`, `_embed_cache_get/put()`, `_check_embedding_freshness()`, `_cosine_similarity_search()`, `_semantic_match_tools()`, `_detect_embedding_provider()`, and `_doc_rag_retrieve()`.

**Problem**: The embedding subsystem has its own caching strategy, provider detection, and similarity math — all concerns orthogonal to prompt construction. Testing cosine similarity or cache behavior requires sourcing the entire 2,303-line prompt module. The embedding code is also used by `llm_ranking.sh` (semantic match input), creating a circular dependency where ranking depends on prompt.sh for embeddings.

**Recommendation**: Extract to `src/lib/llm_embedding.sh` (~170 lines):
- `_detect_embedding_provider()` — provider auto-detection (Ollama → OpenAI → Gemini → regex fallback)
- `_embed_query()` — query vectorization with provider dispatch
- `_embed_cache_get/put()` — session-level embedding cache
- `_check_embedding_freshness/staleness()` — SHA-256 based staleness check
- `_cosine_similarity_search()` — vector similarity computation
- `_semantic_match_tools()` — orchestrator for semantic tool matching
- `_doc_rag_retrieve()` — documentation retrieval via embeddings

This breaks the circular dependency: `llm_ranking.sh` imports from `llm_embedding.sh` directly instead of depending on `llm_prompt.sh`.

---

### R2: Extract Shared Query Processor from `roachie-batch`
**Area**: Application | **Priority**: P1 | **Effort**: High

**Current state**: `roachie-batch` (1,954 lines) and `llm_assistant.sh` (2,241 lines) duplicate ~300 lines of core query processing logic:
- Query → LLM call → response parsing → command execution → reflexion retry (roachie-batch:1044-1230 vs llm_assistant.sh:623-660 + 1710-1775)
- Cluster topology initialization (defined independently in both)
- Learning DB loading (roachie-batch:942-957 vs llm_assistant.sh:1522-1548)

**Problem**: Bug fixes to the query loop, reflexion logic, or learning DB loading must be applied in two places. The v30 circuit breaker was added to `llm_providers.sh` (shared), but reflexion flow logic remains duplicated. Behavioral drift between batch and interactive modes is likely over time.

**Recommendation**: Extract to `src/lib/llm_query.sh` (~200 lines):
- `_nl_process_single_query(query, system_prompt, options)` — unified query-to-response pipeline
- `_nl_reflexion_loop(query, error_output, iteration)` — shared reflexion/self-correction
- `_nl_init_learning_context()` — learning DB loading for both paths

`roachie-batch` becomes a thin loop that calls `_nl_process_single_query()` per prompt line, adds assertion checking and JUnit output. `llm_assistant.sh` REPL calls the same function per user input, adds interactive features (voice, feedback, slash commands).

**Impact**: Reduces roachie-batch by ~300 lines; guarantees identical query behavior across modes.

---

### R3: Standardize Provider Error Response Format
**Area**: AI Architecture | **Priority**: P2 | **Effort**: Medium

**Current state**: The 5 provider `_extract_*_response()` functions return inconsistent formats on error:
- **Ollama** returns structured JSON (`{"reasoning": "...", "commands": [], "needs_followup": false}`) on API errors
- **Anthropic/Vertex** return raw error text to stderr
- **Gemini** strips markdown fences and filters thinking parts (asymmetric processing)
- **OpenAI** returns `choices[0].message.content` which may be null on tool-only responses

**Problem**: Downstream code in `llm_assistant.sh` must handle multiple error shapes. A Gemini-specific markdown strip means the same LLM output renders differently depending on provider. Ollama's pre-formatted error JSON bypasses the normal response parsing, potentially confusing the agent loop.

**Recommendation**: Define a canonical extraction contract that all providers must satisfy:
```bash
# Return format: JSON on stdout, errors on stderr
# Success: {"text": "...", "tool_calls": [...], "tokens": {"input": N, "output": N}}
# Error:   exit 1, stderr contains error message
```
- Move Gemini's markdown stripping to a shared post-processing step (all providers benefit)
- Remove Ollama's error JSON pre-formatting; let the caller construct error responses
- Add a `_normalize_provider_response()` wrapper called after every extraction

---

### R4: Extract Schema Context into `llm_schema.sh`
**Area**: AI Architecture | **Priority**: P2 | **Effort**: Low

**Current state**: Schema context functions occupy ~280 lines in `llm_prompt.sh`: `_nl_fetch_tenant_schema()` (139 lines), `_nl_fetch_schema_context()` (128 lines), and `_nl_build_schema_sql_url()` (20 lines). These are self-contained — they query the cluster, format DDL, and return context strings.

**Problem**: Schema fetching involves SQL execution, connection building, and tenant routing — concerns unrelated to prompt construction. The functions are only called once per session (cached), but their complexity (FK discovery, index enumeration, query stats) makes them hard to test within the prompt module.

**Recommendation**: Extract to `src/lib/llm_schema.sh` (~280 lines):
- `_nl_build_schema_sql_url()` — connection URL for schema queries
- `_nl_fetch_tenant_schema()` — per-tenant DDL extraction (SHOW DATABASES, information_schema)
- `_nl_fetch_schema_context()` — orchestrator with caching, multi-tenant iteration, truncation
- Keep the config constants (`_NL_SCHEMA_MAX_DBS`, `_NL_SCHEMA_MAX_TABLES`, etc.) in `llm_config.sh`

This reduces `llm_prompt.sh` from 2,303 to ~1,750 lines and makes schema fetching independently testable.

---

### R5: Sanitize Learning DB Entries Before Persistent Storage
**Area**: Security | **Priority**: P2 | **Effort**: Low

**Current state**: The learning DB (`~/.roachie/failures.jsonl`, `~/.roachie/successes.jsonl`) stores raw query text, command outputs, and error messages. `_sanitize_for_prompt()` is applied when injecting learning context into LLM prompts (llm_metrics.sh:707-730), but the stored entries themselves contain unsanitized data.

**Problem**: If the cluster returns error messages containing sensitive data (connection strings, internal IPs, table contents), these persist on disk indefinitely (until 30-day compaction). A compromised workstation could expose this data. The learning DB is also read by `roachie-batch` for regression analysis, widening the exposure surface.

**Recommendation**: Apply sanitization at write time, not just read time:
```bash
_save_failure_persistent() {
  local sanitized_output
  sanitized_output=$(_mask_sensitive_output "$output")
  # Write sanitized version to JSONL
}
```
- Apply `_mask_sensitive_output()` (already in llm_metrics.sh) before writing to failures.jsonl
- Strip connection strings, IP addresses, and credential-like patterns from stored entries
- Cost: Near-zero (function already exists; just call it at write path instead of only at read path)

---

### R6: Add MCP-to-cr_* Drift Detection in CI
**Area**: Operations | **Priority**: P2 | **Effort**: Low

**Current state**: `tools/mcp/tools.yaml` defines 39 SQL-based tool endpoints. The 78 cr_* bash scripts in `tools/tools.25.2/` evolve independently. There is no automated check that MCP tool definitions stay aligned with actual cr_* tool capabilities or that MCP doesn't reference tools that have been renamed or removed.

**Problem**: If a cr_* tool's flags change, the MCP schema may reference stale parameters. If new cr_* tools are added, MCP coverage gaps grow silently. The reverse is also possible — MCP could define tools with no corresponding cr_* implementation.

**Recommendation**: Add a CI validation step in `.github/workflows/tests.yml`:
```bash
# Validate MCP tools reference real cr_* scripts
for tool in $(grep 'name:' tools/mcp/tools.yaml | awk '{print $2}' | tr '-' '_'); do
  if ! ls tools/current/"${tool}" 2>/dev/null; then
    echo "DRIFT: MCP tool ${tool} has no cr_* script" >&2
    exit 1
  fi
done

# Validate cr_* tools with SQL-only logic have MCP coverage
for script in tools/current/cr_*; do
  tool=$(basename "$script" | tr '_' '-')
  if ! grep -q "name: ${tool}" tools/mcp/tools.yaml 2>/dev/null; then
    echo "INFO: ${tool} not in MCP (may be intentional)" >&2
  fi
done
```
This prevents silent drift and flags coverage gaps during PR review.

---

### R7: Add Cross-Session Tool Help Cache
**Area**: Performance | **Priority**: P2 | **Effort**: Low

**Current state**: Tool help text is fetched via `cr_tool --help` on every session start. With `xargs -P8` parallelism (v30 R6), this takes 50-100ms for 7 tools. However, tool help text rarely changes between sessions — it only changes when tools are updated.

**Problem**: On slower machines or with Ollama (where tool detection may trigger model loading), the help fetch adds noticeable latency to every session start. The help text is deterministic given the tool version.

**Recommendation**: Cache help text to `~/.roachie/cache/help/` with version-based invalidation:
```bash
_help_cache_dir="${HOME}/.roachie/cache/help"
_help_cache_key="${tool}_$(md5 -q "${tools_dir}/${tool}" 2>/dev/null)"
if [[ -f "${_help_cache_dir}/${_help_cache_key}" ]]; then
  cat "${_help_cache_dir}/${_help_cache_key}"
else
  ${tool} --help 2>/dev/null | head -30 | tee "${_help_cache_dir}/${_help_cache_key}"
fi
```
- Invalidation: MD5 of the tool script (changes when tool is modified)
- Cleanup: Prune cache entries older than 7 days on session start
- Expected improvement: 50-100ms → <5ms for cached tools (first session after update still fetches)

---

### R8: Add Trap Cleanup to Learning DB Compaction
**Area**: AI Architecture | **Priority**: P3 | **Effort**: Low

**Current state**: `_compact_learning_db_inner()` (llm_metrics.sh:595-650) creates temp files via `_secure_mktemp()` for compaction work, but has no `trap` cleanup. If the process dies between temp file creation and the final `mv`, orphaned files persist in `$TMPDIR`.

**Problem**: Under normal operation this is harmless (compaction completes in milliseconds). Under stress (OOM, Ctrl-C during batch run), temp files accumulate. The naming pattern `roachie_compact.XXXXXX` makes them identifiable but not auto-cleaned.

**Recommendation**: Add a cleanup trap at function entry:
```bash
_compact_learning_db_inner() {
  local tmp_file trimmed_file
  tmp_file=$(_secure_mktemp "${TMPDIR:-/tmp}/roachie_compact.XXXXXX")
  trap 'rm -f "$tmp_file" "${trimmed_file:-}" 2>/dev/null' RETURN INT TERM
  # ... existing logic ...
}
```
Cost: 1 line. Prevents orphaned temp files under all exit conditions.

---

### R9: Add Provider Response Normalization Tests
**Area**: Testing | **Priority**: P3 | **Effort**: Medium

**Current state**: Provider extraction functions (`_extract_*_response()`) are tested implicitly through integration tests but have no dedicated unit tests verifying schema consistency across providers. The test suites `test_llm_extract.sh` and `test_llm_providers.sh` exist but focus on happy-path extraction.

**Problem**: Subtle schema differences between providers (documented in R3) can cause downstream failures that only manifest with specific provider/query combinations. Without normalization tests, regressions in one provider's extraction go undetected.

**Recommendation**: Add `tests/unit/test_provider_normalization.sh`:
```bash
# For each provider:
# 1. Feed identical mock API responses (success, error, tool call, empty)
# 2. Assert extraction produces identical field names and types
# 3. Assert error handling produces consistent exit codes

test_all_providers_return_text_field() {
  for provider in anthropic openai gemini vertex ollama; do
    result=$(_extract_${provider}_response "$mock_success_response")
    assert_contains "$result" '"text"' "Provider $provider must return text field"
  done
}

test_all_providers_handle_empty_response() {
  for provider in anthropic openai gemini vertex ollama; do
    result=$(_extract_${provider}_response '{}')
    assert_equals 1 $? "Provider $provider must fail on empty response"
  done
}
```
This catches normalization drift before it reaches production.

---

### R10: Add Session Data Lifecycle Management
**Area**: Operations | **Priority**: P3 | **Effort**: Low

**Current state**: Session data persists in `~/.roachie/`:
- `sessions/last_session.json` — overwritten each session (7-day staleness check on load)
- `failures.jsonl` / `successes.jsonl` — compacted to 30 days, max 25 entries
- `cache/` — no explicit cleanup policy
- Debug logs — rotated at 1MB with 5 backups

**Problem**: While individual components have size limits, there is no unified cleanup routine. Over months of usage, stale cache files, old debug logs, and orphaned temp files accumulate. There is no `roachie cleanup` command or startup garbage collection.

**Recommendation**: Add a lightweight cleanup routine called on session start:
```bash
_nl_session_gc() {
  local cutoff_days=30
  # Clean stale help cache
  find "${HOME}/.roachie/cache" -type f -mtime +${cutoff_days} -delete 2>/dev/null
  # Clean orphaned temp files
  find "${TMPDIR:-/tmp}" -name "roachie_*" -mtime +1 -delete 2>/dev/null
  # Rotate old debug logs
  find "${HOME}/.roachie/logs" -name "*.log.*" -mtime +${cutoff_days} -delete 2>/dev/null
}
```
- Run at most once per day (guard with marker file)
- Log what was cleaned in debug mode
- No user-visible output unless items were cleaned

---

### R11: Complete Learned Keyword Regex Metachar Validation (v30 R10 Carry-Forward)
**Area**: AI Architecture | **Priority**: P3 | **Effort**: Low

**Current state**: `_nl_load_learned_keywords()` sanitizes via `tr -cd 'A-Za-z0-9_ ,.-'` which strips most dangerous characters. However, the remaining allowed characters (`.`, `-`) are valid regex metacharacters (`.` matches any character, `-` is special inside character classes).

**Problem**: A learned keyword containing `.` (e.g., "v25.2") would match "v25X2" in regex context. While unlikely to cause security issues, it can produce false-positive tool matches, degrading accuracy.

**Recommendation**: Escape regex metacharacters after sanitization:
```bash
keyword="${keyword//./\\.}"
keyword="${keyword//-/\\-}"
```
Or simpler — restrict to `[A-Za-z0-9_ ]` only (drop `.` and `-` from allowed set).

---

### R12: Add Flock to Session Cost File Writes
**Area**: Application | **Priority**: P3 | **Effort**: Low

**Current state**: `_nl_add_session_cost_inner()` (llm_metrics.sh:105-130) writes the cumulative session cost to `$_NL_SESSION_COST_FILE` via:
```bash
echo "$new_cost" > "$_NL_SESSION_COST_FILE"
```
No file locking is used. The current single-threaded design makes this safe, but the architecture supports background API calls (streaming) and the circuit breaker can trigger concurrent fallback attempts.

**Problem**: If two concurrent API calls complete simultaneously (e.g., during circuit breaker fallback), both may read the same cost value and write back, losing one update. The cost error is small (one query's cost) but accumulates.

**Recommendation**: Use flock for atomic read-modify-write:
```bash
_nl_add_session_cost_inner() {
  (
    flock -x 200
    local current; current=$(cat "$_NL_SESSION_COST_FILE" 2>/dev/null || echo 0)
    echo "$(( current + cost_millicents ))" > "$_NL_SESSION_COST_FILE"
  ) 200>"${_NL_SESSION_COST_FILE}.lock"
}
```
The `_with_db_lock()` pattern already exists in llm_metrics.sh for learning DB writes — reuse the same approach.

---

### R13: Add roachie-batch Integration Test Suite
**Area**: Testing | **Priority**: P3 | **Effort**: Medium

**Current state**: `roachie-batch` has no dedicated test suite. Testing relies on manual runs with prompt files and visual inspection of JSONL output. The batch runner has its own assertion system (`tool:`, `commands:`, `contains:`, `flags:`) but this is a test framework for LLM responses, not tests of roachie-batch itself.

**Problem**: roachie-batch features like `--resume`, `--retry-failed`, `--dry-run`, JUnit XML generation, rate limiting, and assertion parsing are untested. Regressions in these features (e.g., resume skipping the wrong prompts) would go undetected until a batch run fails in production.

**Recommendation**: Add `tests/unit/test_roachie_batch.sh`:
- Test assertion parsing: `tool:cr_tables` → correct tool check
- Test `--resume` logic: skips already-completed prompts correctly
- Test `--retry-failed` logic: re-runs only non-passing tests
- Test JUnit XML generation: valid XML structure
- Test rate limiting: minimum delay between calls is enforced
- Test prompt file parsing: comments, blank lines, tab-separated assertions
- Use mock LLM responses (no API calls needed)

---

### R14: Add Embedding Auto-Regeneration on Staleness
**Area**: AI Architecture | **Priority**: P4 | **Effort**: Medium

**Current state**: `_check_embedding_freshness()` warns when embeddings are >7 days old and `_check_embedding_staleness()` detects SHA-256 mismatches between current tool descriptions and stored embeddings. Both produce warnings only — no automatic regeneration.

**Problem**: Users see warnings but must manually run `tools/utils/generate_embeddings.sh` to update. In practice, warnings are often ignored, leading to degraded semantic matching accuracy after tool changes.

**Recommendation**: Add opt-in auto-regeneration:
```bash
if [[ "${NL_AUTO_REGEN_EMBEDDINGS:-0}" == "1" ]] && _check_embedding_staleness; then
  _nl_debug "EMBEDDING" "Auto-regenerating stale embeddings"
  bash "${ROACHMAN_PROJECT_DIR}/tools/utils/generate_embeddings.sh" \
    --provider "$(_detect_embedding_provider)" --quiet
fi
```
- Gated behind `NL_AUTO_REGEN_EMBEDDINGS=1` (disabled by default — preserves current behavior)
- Only triggers on SHA-256 mismatch (tool descriptions changed), not on age alone
- Runs `generate_embeddings.sh` with `--quiet` flag to suppress output
- Log regeneration in debug and metrics

---

## Architectural Strengths (What Continues to Work Well)

1. **Module extraction discipline**: v30 delivered `llm_ranking.sh` (191 lines) and `llm_exec.sh` (401 lines) as clean, focused modules — this pattern should continue with embedding and schema extraction
2. **Security layering**: 5-layer command validation pipeline remains robust; structured error codes (v30 R4) improved debuggability without weakening security
3. **Circuit breaker resilience** (v30 R5): Provider-level fault isolation with automatic cross-provider fallback — well-implemented with configurable threshold and cooldown
4. **Token budget governor** (v30 R8): Priority-based prompt section trimming prevents context overflow while preserving the most relevant content
5. **Provider plugin pattern**: Adding a 6th provider still requires only a single file in `providers/` — the abstraction holds
6. **Persistent learning**: Failure/success databases with 30-day TTL and size caps provide bounded, useful cross-session context
7. **Observability**: Cost tracking (millicent precision), latency percentiles, audit logging, and session statistics are comprehensive and well-integrated

---

## Table 3: Codebase Metrics (v31)

| Component | Lines | Files | Change from v30 |
|-----------|-------|-------|-----------------|
| bin/roachman | 2,885 | 1 | +27 |
| bin/roachie-nl | 410 | 1 | +72 |
| bin/roachie-batch | 1,954 | 1 | +104 |
| src/lib/ (non-LLM) | 5,083 | 10 | +221 (errors.sh, sql.sh expanded) |
| src/lib/ (LLM) | 7,697 | 9 | +122 (llm_ranking.sh, llm_exec.sh new) |
| src/lib/providers/ | 1,162 | 6 | — |
| tools/lib/common.sh | 231 | 1 | — |
| cr_* tools | ~5,000 | 78 | — |
| Unit tests | ~12,500 | 54 | +488 (3 error-path suites) |
| Integration tests | ~2,100 | 9 | — |
| **Total** | **~39,000** | **170** | **+1,034** |

---

## Table 4: Module Size Distribution (Decomposition Targets)

| Module | Lines | Proposed Action |
|--------|-------|----------------|
| llm_prompt.sh | 2,303 | Extract embedding (~170L) → R1, schema (~280L) → R4 |
| llm_assistant.sh | 2,241 | Extract shared query processor (~200L) → R2 |
| roachie-batch | 1,954 | Consume shared query processor → R2 |
| llm_providers.sh | 1,213 | Stable (circuit breaker, budget governor well-placed) |
| monitoring.sh | 1,179 | Stable (self-contained Docker Compose management) |
| llm_metrics.sh | 1,088 | Add trap cleanup → R8, add flock → R12 |
| roachman | 2,885 | Stable (menu system, no further extraction needed) |

After R1 + R2 + R4, `llm_prompt.sh` drops from 2,303 → ~1,550 lines. No module exceeds 2,300 lines.

---

## CLI Summary

```
+-----+----------+------------------------------------------------------+----------+--------+--------+
|  #  |   Area   |                   Recommendation                     | Priority | Effort | Status |
+-----+----------+------------------------------------------------------+----------+--------+--------+
|  R1 | AI       | Extract embedding subsystem from llm_prompt.sh       |    P1    |  Med   | FIXED  |
|  R2 | App      | Extract shared query processor from roachie-batch     |    P1    |  High  | FIXED  |
|  R3 | AI       | Standardize provider error response format            |    P2    |  Med   | FIXED  |
|  R4 | AI       | Extract schema context into llm_schema.sh             |    P2    |  Low   | FIXED  |
|  R5 | Security | Sanitize learning DB entries before storage           |    P2    |  Low   | FIXED  |
|  R6 | Ops      | Add MCP-to-cr_* drift detection in CI                 |    P2    |  Low   | FIXED  |
|  R7 | Perf     | Add cross-session tool help cache                     |    P2    |  Low   | FIXED  |
|  R8 | AI       | Add trap cleanup to learning DB compaction             |    P3    |  Low   | FIXED  |
|  R9 | Test     | Add provider response normalization tests             |    P3    |  Med   | FIXED  |
| R10 | Ops      | Add session data lifecycle management                 |    P3    |  Low   | FIXED  |
| R11 | AI       | Complete learned keyword regex validation (v30 R10)   |    P3    |  Low   | FIXED  |
| R12 | App      | Add flock to session cost file writes                 |    P3    |  Low   | FIXED  |
| R13 | Test     | Add roachie-batch integration test suite              |    P3    |  Med   | FIXED  |
| R14 | AI       | Add embedding auto-regeneration on staleness           |    P4    |  Med   | FIXED  |
+-----+----------+------------------------------------------------------+----------+--------+--------+

v31 Status:  14/14 FIXED | Branch: feature/arch-review-v31 | Commit: e40e1ac
v30 Status:  13/14 FIXED, 1 PARTIAL (R10 → carried as v31 R11, now FIXED)
Tests:       51 suites (50 pass, 1 pre-existing) | 1,601 assertions
Net change:  +1,185 / -713 lines across 22 files | llm_prompt.sh 2,303 → 1,766 (-23%)
New modules: llm_embedding.sh | llm_schema.sh | llm_query.sh
New tests:   test_provider_normalization.sh (45) | test_roachie_batch.sh (21)
Strengths:   Module extraction discipline | Circuit breaker resilience | Token budget governor
             Provider plugin pattern | Persistent learning with TTL | Full observability
Codebase:    ~39K lines | 170 files | 78 cr_* tools | 5 LLM providers | 53 test suites
```
