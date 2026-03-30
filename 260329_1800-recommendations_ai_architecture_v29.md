# AI Architecture Review v29 — Roachie NL Subsystem

**Date:** 2026-03-29
**Roachie Version:** 1.2.0
**Scope:** Full AI/LLM architecture + application + system design review
**LLM Subsystem:** 18,589 lines across 13 modules + 6 provider plugins
**Test Coverage:** 47 unit test suites, 9 integration suites

---

## Overall Assessment

**Score: 9.5 / 10** (adjusted from 9.7 in v28 due to expanded scope)

The v28 review implemented all 10 recommendations: Vertex builder extraction, enrichment decomposition (428->6 functions), special command dispatcher, process_query decomposition (500->3 phases), jq batch optimization (~60-80 calls eliminated), header consolidation, 21/47 test failures fixed to 45/45, IDF scaling, composite arrays, and base dir caching.

This v29 review expands beyond the NL subsystem to cover application architecture, system design, operational concerns, and test infrastructure.

**Maturity Matrix:**

| # | Dimension | Score | Delta | Notes |
|---|-----------|-------|-------|-------|
| 1 | Provider architecture | 9.5 | = | 6 providers, credential helper, shared streaming, shared builders |
| 2 | Prompt engineering | 9.5 | = | Enrichment decomposed into 6 functions; semantic+regex combined matching |
| 3 | Agent loop | 9.5 | = | Reflexion tracking, 3-phase decomposition, needs_followup heuristics |
| 4 | Security | 10 | = | 7-layer validation, PII masking, RBAC, 113 security tests |
| 5 | Metrics & observability | 9.5 | = | Cache hit/miss, reflexion success/failure, context trim counters |
| 6 | Batch testing | 9.0 | -0.5 | _run_batch() still 973 lines; no parallel test execution |
| 7 | Fine-tuning pipeline | 9.0 | = | CI flag, regression gate, JSON manifest |
| 8 | Error handling | 9.5 | = | Multi-layer streaming error detection, connection drops |
| 9 | Code maintainability | 9.0 | +0.5 | v28 decompositions landed; one 973-line function remains |
| 10 | Test coverage | 8.5 | -0.5 | 47 suites but zero unit tests for cluster/upgrade/replication/ceph |
| 11 | System design | 9.0 | NEW | Solid module architecture; minor operational gaps |

---

## Recommendations

### R1: Decompose `_run_batch()` (Medium effort, High priority)

**Problem:** `_run_batch()` in `bin/roachie-batch` spans 973 lines (819-1791), making it the largest function in the codebase. It handles argument parsing, file I/O, prompt iteration, LLM calls, command execution, assertion evaluation, summary generation, and JSONL logging in a single function.

**Fix:** Split into 5 focused functions:

| Function | Responsibility | Est. Lines |
|----------|---------------|------------|
| `_batch_parse_args()` | CLI flag parsing, validation, defaults | ~120 |
| `_batch_process_prompt()` | Single prompt lifecycle: LLM call, extraction, execution | ~250 |
| `_batch_evaluate_assertions()` | Judge/assertion evaluation logic | ~150 |
| `_batch_generate_summary()` | Summary stats, pass/fail reporting | ~100 |
| `_batch_write_jsonl()` | JSONL log entry construction | ~80 |

**Impact:** Each function becomes independently testable. The orchestrator `_run_batch()` shrinks to ~150 lines of flow control.

---

### R2: Fix context trim threshold inconsistency (Low effort, Medium priority)

**Problem:** `llm_config.sh` defines `_NL_CONTEXT_TRIM_PCT=80` (line 68), and `llm_assistant.sh` correctly uses it (line 758). However, `llm_providers.sh` (line 769) hardcodes `85`:

```bash
# llm_providers.sh:769
local _threshold=$(( _ctx_limit * 85 / 100 ))
```

This means history trimming triggers at 80% (correct) but the provider-level overflow check uses a different 85% threshold, creating a 5% gap where trimmed history could still trigger provider warnings.

**Fix:** Replace `85` with `${_NL_CONTEXT_TRIM_PCT}` in `llm_providers.sh`.

---

### R3: Extract inline reflexion prompts to template files (Low effort, Medium priority)

**Problem:** `llm_assistant.sh` contains hardcoded multi-line reflexion prompts (lines 1914-1928) embedded directly in the agent loop. These instruction blocks define how the LLM should handle command failures and follow-ups. Other prompts (system prompt, tool catalog, few-shot examples) are already templated.

**Fix:** Create two template files:
- `tools/utils/templates/reflexion_failure.txt` — the "previous command failed, analyze what went wrong" prompt
- `tools/utils/templates/reflexion_followup.txt` — the "based on this output, continue your analysis" prompt

Load them via `cat` at runtime (same pattern as existing template loading in `_nl_build_system_prompt()`).

---

### R4: Add unit tests for cluster/upgrade/replication/ceph modules (Medium effort, High priority)

**Problem:** Four core modules have zero unit test coverage:

| Module | Lines | Functions | Test Coverage |
|--------|-------|-----------|---------------|
| `src/lib/cluster.sh` | ~800 | ~15 | Integration only (input validation) |
| `src/lib/upgrade.sh` | ~600 | ~12 | None |
| `src/lib/replication.sh` | ~400 | ~8 | None |
| `src/lib/ceph.sh` | ~350 | ~7 | Integration only (E2E) |

These modules contain pure functions (semver parsing, version comparison, config validation, image tag construction) that can be unit tested without Docker or live clusters.

**Fix:** Create 4 test suites targeting pure-function logic:
- `tests/unit/test_upgrade.sh` — semver parsing, version comparison, upgrade path validation
- `tests/unit/test_cluster.sh` — config generation, node count validation, network naming
- `tests/unit/test_replication.sh` — lag threshold parsing, status interpretation
- `tests/unit/test_ceph.sh` — bucket name validation, endpoint URL construction

---

### R5: Add parallel test execution to test runner (Medium effort, Medium priority)

**Problem:** `tests/run-all-tests.sh` runs all 47 unit test suites sequentially. On the current codebase, a full test run takes 2-3 minutes. With the test suite count growing, this will become a bottleneck.

**Fix:** Add a `--parallel N` flag that runs up to N test suites concurrently using background jobs:

```bash
if [[ "$parallel" -gt 1 ]]; then
  local running=0
  for test_file in "${test_files[@]}"; do
    run_test_suite "$test_file" &
    running=$((running + 1))
    if [[ $running -ge $parallel ]]; then
      wait -n
      running=$((running - 1))
    fi
  done
  wait
fi
```

Capture stdout/stderr per suite to avoid interleaved output. Default to sequential (backward compatible).

---

### R6: Add file locking for concurrent batch runs (Low effort, Low priority)

**Problem:** `roachie-batch` writes to shared state files (metrics CSV, learning DB, audit log) without file locking. Two concurrent batch runs against the same cluster could produce corrupt CSV entries or duplicated JSON records.

**Fix:** Add `flock` around critical file writes in `llm_metrics.sh`:

```bash
(
  flock -x 200
  echo "$csv_line" >> "$NL_FEEDBACK_CSV"
) 200>"${NL_FEEDBACK_CSV}.lock"
```

This is low priority because concurrent batch runs are rare in practice, but it's a correctness issue.

---

### R7: Pin Docker images with version variables (Low effort, Low priority)

**Problem:** Several modules hardcode Docker image versions inline:

| Module | Image | Version |
|--------|-------|---------|
| `replication.sh` | `cockroachdb/cockroach` | `v25.2.7` |
| `kafka.sh` | `apache/kafka` | `3.9.0` |
| `monitoring.sh` | `prom/prometheus` | `v3.2.1` |
| `monitoring.sh` | `grafana/grafana` | `11.5.2` |

Some use env var overrides (monitoring.sh does `${PROMETHEUS_VERSION:-v3.2.1}`), but not all.

**Fix:** Ensure all image references follow the pattern `${ENV_VAR:-default}` and document the available overrides in `config/roachman.conf.example`. Digest pinning (`@sha256:...`) is not recommended for this project — version tags with env var overrides provide the right balance of reproducibility and upgradeability.

---

### R8: Fix Anthropic SSE partial tool args handling (Low effort, Medium priority)

**Problem:** In `src/lib/providers/anthropic_sse.sh` (lines 90-96), when partial/malformed tool arguments are received during streaming, the code uses `continue` to skip the current SSE event and proceed to the next one. This is technically correct for streaming (more data may follow), but if the tool args are permanently malformed (e.g., truncated JSON), the loop will process all remaining events without acting on the error.

**Fix:** Track consecutive partial-arg events. After 3 consecutive failures, `break` out of the loop and surface the error. Single partials should still `continue` (streaming may complete the JSON in the next event).

---

### R9: Add system prompt size guard (Low effort, Low priority)

**Problem:** `llm_assistant.sh` logs a debug warning when the system prompt exceeds 60% of the context window, but takes no action. For small-context providers (Ollama at 16K), a large prompt can consume the entire context budget, leaving no room for conversation history.

**Fix:** When system prompt exceeds 70% of context for Ollama providers, auto-truncate tool help entries (already limited to 5) and skip doc RAG. Log the truncation action. This prevents silent degradation where the model receives a prompt but has no room for the user's question.

---

### R10: Standardize VERSION bumping workflow (Low effort, Low priority)

**Problem:** VERSION file is manually edited. The current version (1.2.0) was bumped during v28 implementation, but there's no automation or convention for when to bump.

**Fix:** Add a `bump-version.sh` script in `tools/utils/` that:
1. Reads current version from `VERSION`
2. Accepts `--major`, `--minor`, or `--patch`
3. Updates `VERSION` file
4. Outputs the new version for use in commit messages

This is documentation/tooling — not a code fix.

---

## Summary

| # | Recommendation | Effort | Priority | Category |
|---|---------------|--------|----------|----------|
| R1 | Decompose `_run_batch()` 973 lines | Medium | High | Code maintainability |
| R2 | Fix context trim 80% vs 85% inconsistency | Low | Medium | Correctness |
| R3 | Extract reflexion prompts to templates | Low | Medium | Maintainability |
| R4 | Unit tests for cluster/upgrade/replication/ceph | Medium | High | Test coverage |
| R5 | Parallel test execution (`--parallel N`) | Medium | Medium | Developer experience |
| R6 | File locking for concurrent batch runs | Low | Low | Correctness |
| R7 | Pin Docker images with env var overrides | Low | Low | Operational |
| R8 | Fix SSE partial tool args (continue vs break) | Low | Medium | Correctness |
| R9 | System prompt size guard for small-context models | Low | Low | Robustness |
| R10 | VERSION bump automation script | Low | Low | Tooling |

---

## CLI Summary

```
┌────────────────────────────────────────────────────────────────┐
│          AI Architecture Review v29 — Summary                  │
│          2026-03-29 | Roachie v1.2.0                          │
├────────────────────────────────────────────────────────────────┤
│  Overall Score: 9.5 / 10                                      │
│  Codebase: 18,589 lines | 47 unit suites | 9 integration      │
│  Providers: 6 | Tools: 78 | Modules: 20                       │
├────┬─────────────────────────────────────────┬────────┬────────┤
│ #  │ Recommendation                          │ Effort │ Pri    │
├────┼─────────────────────────────────────────┼────────┼────────┤
│ R1 │ Decompose _run_batch() (973 lines)      │ Med    │ HIGH   │
│ R2 │ Fix context trim 80%/85% inconsistency  │ Low    │ Med    │
│ R3 │ Extract reflexion prompts to templates   │ Low    │ Med    │
│ R4 │ Unit tests: cluster/upgrade/repl/ceph   │ Med    │ HIGH   │
│ R5 │ Parallel test execution (--parallel N)   │ Med    │ Med    │
│ R6 │ File locking for concurrent batch runs   │ Low    │ Low    │
│ R7 │ Docker image env var overrides           │ Low    │ Low    │
│ R8 │ SSE partial tool args break after 3      │ Low    │ Med    │
│ R9 │ System prompt size guard (Ollama)        │ Low    │ Low    │
│ R10│ VERSION bump automation script           │ Low    │ Low    │
└────┴─────────────────────────────────────────┴────────┴────────┘
```
