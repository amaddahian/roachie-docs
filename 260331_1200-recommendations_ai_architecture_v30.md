# Roachie — Architectural Review v30

**Date**: 2026-03-31
**Scope**: AI subsystem, application design, security, testing, system architecture
**Codebase**: ~19K lines core + ~12K lines tests across 78 tools, 20 modules, 5 LLM providers

---

## Executive Summary

Roachie is a mature, production-grade CockroachDB toolkit with a sophisticated NL/AI layer. The architecture is well-modularized with strong security practices (5-layer command validation, 4-tier RBAC, PII masking, audit logging). The AI subsystem features multi-provider abstraction, RRF-based tool ranking, persistent learning, and reflexion-based self-correction.

This review identifies **14 recommendations** across 4 priority tiers. No critical vulnerabilities were found. The primary opportunities are in **architectural decoupling**, **performance optimization**, and **testing maturity**.

---

## Table 1: Recommendation Summary

| # | Area | Recommendation | Priority | Effort | Status |
|---|------|---------------|----------|--------|--------|
| R1 | AI | Extract tool ranking into standalone module | P1 | Medium | NEW |
| R2 | AI | Centralize message history management | P1 | Medium | NEW |
| R3 | App | Consolidate SQL URL construction (3 implementations) | P1 | Medium | NEW |
| R4 | App | Extract structured error types with context | P2 | Medium | NEW |
| R5 | AI | Add provider-level circuit breaker with cross-provider fallback | P2 | Medium | NEW |
| R6 | Perf | Parallelize tool help fetching with xargs -P | P2 | Low | NEW |
| R7 | Test | Enable --parallel in CI and add test isolation | P2 | Medium | NEW |
| R8 | AI | Add token budget governor to prompt construction | P2 | Medium | NEW |
| R9 | App | Decouple cluster.sh from replication.sh | P3 | High | NEW |
| R10 | Security | Validate learned keywords before regex injection | P3 | Low | NEW |
| R11 | Test | Add negative/error-path test suites | P3 | Medium | NEW |
| R12 | AI | Implement tiered model routing for cost optimization | P3 | Medium | NEW |
| R13 | App | Batch SQL execution within single docker exec | P3 | Low | NEW |
| R14 | Test | Add test metadata tags and execution time tracking | P4 | Low | NEW |

---

## Detailed Recommendations

### R1: Extract Tool Ranking Into Standalone Module
**Area**: AI Architecture | **Priority**: P1 | **Effort**: Medium

**Current state**: `llm_prompt.sh` (2,380 lines) handles prompt construction AND 4-stage tool enrichment (regex matching, semantic search, RRF scoring, help loading). The RRF ranking, failure penalties, tool graph traversal, and re-ranking are all interleaved within `_enrich_score_and_rank_tools()`.

**Problem**: Changing the ranking algorithm requires editing the prompt module. Testing ranking in isolation requires sourcing the entire prompt subsystem. The 4-stage pipeline has grown organically and the stages are tightly coupled.

**Recommendation**: Extract into `src/lib/llm_ranking.sh`:
- `_rank_tools_rrf()` — pure RRF scoring (input: two ranked lists -> output: merged ranked list)
- `_apply_failure_penalties()` — penalty multipliers from learning DB
- `_expand_tool_graph()` — relationship-based augmentation
- `_rerank_with_llm()` — optional cross-encoder re-ranking

This enables unit testing of ranking logic without LLM infrastructure and makes the algorithm pluggable.

---

### R2: Centralize Message History Management
**Area**: AI Architecture | **Priority**: P1 | **Effort**: Medium

**Current state**: History trimming logic is split between `_nl_process_query()` (sliding window) and `_nl_call_and_parse()` (context overflow check). Three related variables track reflexion state (`_any_reflexion_triggered`, `_agent_attempted_cmds`, `_agent_attempted_normalized`).

**Problem**: Inconsistent truncation behavior across iterations. When context overflows mid-conversation, different code paths handle it differently — one truncates from the end of the system prompt, the other drops oldest messages.

**Recommendation**: Create `_nl_manage_history()` in `llm_assistant.sh`:
- Single function that owns message array mutation
- Handles: append, truncate-oldest, context-budget check, boundary markers
- Encapsulates the 3 reflexion state variables into a coherent lifecycle
- Called at one point per iteration (before LLM call), not scattered

---

### R3: Consolidate SQL URL Construction
**Area**: Application | **Priority**: P1 | **Effort**: Medium

**Current state**: SQL connection URLs are constructed in 3 places:
1. `bin/roachman` -> `_secure_sql_url()` (with TLS, tenant routing)
2. `tools/lib/common.sh` -> `build_conn_url()` (similar but subtly different)
3. `src/lib/replication.sh` -> inline URL construction with inline TLS

**Problem**: Protocol parameter mismatches between implementations. If tenant routing semantics change (e.g., new CockroachDB version changes `options=-ccluster=` behavior), 3 files need updating.

**Recommendation**: Consolidate into `src/lib/sql.sh` (currently only 29 lines):
- Single `build_sql_url(host, port, tenant, user, tls_mode)` function
- Single `build_sql_opts(host, port, tenant, user, tls_mode)` for docker exec style
- Both `roachman` and `common.sh` call the shared implementation
- Add unit tests for URL construction edge cases (special chars in tenant names, IPv6)

---

### R4: Extract Structured Error Types
**Area**: Application | **Priority**: P2 | **Effort**: Medium

**Current state**: `errors.sh` (32 lines) provides `error()`, `warning()`, `info()`, `success()` — all print-only, none exit. Callers must manually check return codes. Error context (call stack, variable state) is lost.

**Problem**: Silent failures propagate. Example: `run_roach_sql` can fail if a cluster is down, but the caller may not check `$?`. No structured error information for debugging.

**Recommendation**: Extend `errors.sh` with:
- `die()` — print error + exit 1 (for unrecoverable failures)
- `error_context()` — capture `${FUNCNAME[@]}` call stack + key variables
- Return structured codes: 1=general, 2=connection, 3=validation, 4=timeout, 5=rbac
- Add `|| die "context"` pattern to critical paths (cluster create, upgrade, replication setup)

---

### R5: Provider Circuit Breaker with Cross-Provider Fallback
**Area**: AI Architecture | **Priority**: P2 | **Effort**: Medium

**Current state**: `_call_llm_api()` retries within a single provider (exponential backoff on transient errors). `_nl_get_fallback_providers()` exists but is only used in interactive mode. In batch mode, a provider failure fails the entire batch.

**Problem**: A rate-limited or degraded provider blocks all queries. No automatic failover to alternate providers in batch mode.

**Recommendation**: Implement circuit breaker pattern:
- Track consecutive failures per provider (file-based counter)
- After N failures (default: 3), mark provider as "open" (skip for 60s)
- Auto-try next provider in fallback list
- After cooldown, try "half-open" (single request to test recovery)
- In `roachie-batch`: enable `ROACHIE_BATCH=1` to auto-select fallback (already partially implemented, extend to circuit breaker)

---

### R6: Parallelize Tool Help Fetching
**Area**: Performance | **Priority**: P2 | **Effort**: Low

**Current state**: `_enrich_load_tool_help()` spawns background jobs but waits sequentially. Bash job management overhead limits true parallelism.

**Problem**: Tool help loading is the slowest enrichment stage (200-400ms). With 7 tools, sequential `cr_tool --help` calls dominate latency.

**Recommendation**: Replace background job pattern with:
```bash
printf '%s\n' "${tools[@]}" | xargs -P8 -I{} bash -c '{} --help 2>/dev/null | head -30' > "$help_cache"
```
This is simpler, truly parallel, and eliminates bash IPC overhead. Expected improvement: 200-400ms -> 50-100ms.

---

### R7: Enable Parallel Tests in CI + Test Isolation
**Area**: Testing | **Priority**: P2 | **Effort**: Medium

**Current state**: 51 unit test suites run serially in CI despite `--parallel N` support in `run-all-tests.sh`. Integration tests share cluster state with no isolation between suites.

**Problem**: CI takes longer than necessary. Integration test failures can cascade (one suite corrupts state for the next).

**Recommendation**:
- Add `--parallel 4` to CI unit test step (safe — unit tests are pure functions)
- For integration tests: use test-specific databases (`CREATE DATABASE test_suite_N`) instead of shared default
- Add cleanup trap per integration suite (not just per-run)
- Track test execution times and report slowest suites

---

### R8: Token Budget Governor for Prompt Construction
**Area**: AI Architecture | **Priority**: P2 | **Effort**: Medium

**Current state**: Token budget checks happen in `_enrich_load_tool_help()` (estimates tokens as chars/4) and in `_call_llm_api()` (overflow warning at 85%). If the system prompt exceeds the context window, it is truncated from the end — potentially losing tool help or schema context.

**Problem**: The truncation is indiscriminate. Critical sections (tool help for the most relevant tool) may be cut while less important sections (schema for unrelated databases) survive.

**Recommendation**: Implement priority-based budget allocation:
1. Reserve 20% for messages + response
2. Allocate remaining 80% to prompt sections by priority:
   - Priority 1: Base system prompt + role (always included)
   - Priority 2: Top-3 tool help texts (critical for accuracy)
   - Priority 3: Schema context (trim databases beyond top-3)
   - Priority 4: Few-shot examples, doc RAG, remaining tool help
3. Each section declares its token estimate; governor trims lowest-priority first
4. Log which sections were trimmed (for debugging accuracy issues)

---

### R9: Decouple cluster.sh from replication.sh
**Area**: Application | **Priority**: P3 | **Effort**: High

**Current state**: `cluster.sh` creates virtual cluster VA. `replication.sh` creates VB as a read-replica of VA. Both manipulate virtual clusters and share naming conventions. `upgrade.sh` duplicates container discovery logic from `cluster.sh`.

**Problem**: Tight coupling means changes to virtual cluster semantics require coordinated edits across 3 modules. Container discovery is reimplemented in `upgrade.sh` (`_list_role_containers`) vs `cluster.sh` (`_get_names_by_role`).

**Recommendation**: Extract shared primitives:
- `_discover_containers(role)` — single implementation in `cluster.sh`, exported
- `_virtual_cluster_create(name, mode)` — unified VC lifecycle
- `replication.sh` calls cluster primitives rather than reimplementing
- `upgrade.sh` uses `_discover_containers` instead of custom discovery

---

### R10: Validate Learned Keywords Before Regex Injection
**Area**: Security | **Priority**: P3 | **Effort**: Low

**Current state**: `_nl_load_learned_keywords()` mines the success database for query word->tool mappings and prepends them to the regex keyword list. The success DB entries are sanitized via `_sanitize_for_prompt()`, but the extracted keywords themselves are not validated as safe regex patterns.

**Problem**: A carefully crafted query stored in the success DB could inject regex metacharacters into the keyword matching engine, causing either false matches or regex errors.

**Recommendation**: Add keyword validation after extraction:
```bash
# Only allow alphanumeric keywords (no regex metacharacters)
[[ "$keyword" =~ ^[a-zA-Z0-9_-]+$ ]] || continue
```
This is a defense-in-depth measure — the existing sanitization makes exploitation unlikely, but the validation cost is near-zero.

---

### R11: Add Negative/Error-Path Test Suites
**Area**: Testing | **Priority**: P3 | **Effort**: Medium

**Current state**: Tests are predominantly happy-path. 800+ lines of security tests validate blocking, but few tests verify error recovery behavior (e.g., what happens when a cluster node dies mid-upgrade, or when an LLM API returns malformed JSON).

**Recommendation**: Add targeted error-path suites:
- `test_cluster_errors.sh` — node failure during create/upgrade, network partition simulation
- `test_llm_error_recovery.sh` — malformed JSON, partial streaming, rate limit recovery
- `test_replication_errors.sh` — replication lag timeout, tenant not found
- Use function stubbing (already used for `_spinner_start`) to inject failures

---

### R12: Tiered Model Routing
**Area**: AI Architecture | **Priority**: P3 | **Effort**: Medium

**Current state**: `NL_TIERED_MODELS=1` flag exists in config but implementation is minimal. All queries go to the same model regardless of complexity.

**Problem**: Simple queries ("show databases") use the same expensive model as complex multi-step queries ("analyze slow queries and suggest index changes"). Cost grows linearly with usage.

**Recommendation**: Implement query complexity classifier:
- **Tier 1** (simple lookup): Single-tool queries matching regex with high confidence -> use cheapest model (Haiku/GPT-4.1-mini/Flash)
- **Tier 2** (standard): Multi-tool or moderate confidence -> use default model
- **Tier 3** (complex): Multi-step plans, reflexion needed -> use premium model
- Classification based on: regex match count, query word count, presence of analytical keywords
- Log tier selection in metrics for cost analysis

---

### R13: Batch SQL Execution
**Area**: Application Performance | **Priority**: P3 | **Effort**: Low

**Current state**: Each `run_roach_sql` call creates a fresh `docker exec` process. The NL agent loop may execute 3-5 sequential SQL commands per query, each with full connection setup overhead.

**Recommendation**: Add `run_roach_sql_batch()` that concatenates multiple SQL statements:
```bash
run_roach_sql_batch "CA" "SHOW DATABASES; SHOW TABLES FROM mydb; SELECT count(*) FROM mydb.t1" "va"
```
Single docker exec, single connection, multiple results. Parse output by statement separator. Expected improvement: 3-5x reduction in execution latency for multi-command queries.

---

### R14: Test Metadata Tags and Execution Tracking
**Area**: Testing | **Priority**: P4 | **Effort**: Low

**Current state**: No test categorization (fast/slow/flaky/requires-cluster). No per-suite execution time tracking. No way to identify which tests are slowest or most flaky.

**Recommendation**: Add metadata header to each test file:
```bash
# @tags: unit, fast, llm
# @requires: none
# @timeout: 30
```
Modify `run-all-tests.sh` to parse tags and:
- Filter by tag: `--tag fast` (run only fast tests)
- Report execution time per suite in summary
- Track flaky tests (pass/fail ratio over last N runs)

---

## Architectural Strengths (What is Working Well)

1. **Security depth**: 5-layer command validation pipeline (whitelist -> metachar -> SQL mutation -> flag check -> RBAC) is enterprise-grade
2. **Provider abstraction**: Adding a new LLM provider requires only a single file in `providers/` — clean plugin pattern
3. **Persistent learning**: Failure/success/correction databases enable continuous improvement and DPO fine-tuning data collection
4. **Tool enrichment pipeline**: RRF fusion of regex + semantic matching achieves 99-100% accuracy across 135 test queries
5. **Observability**: Cost tracking, token counting, latency percentiles, audit logging, and session statistics are comprehensive
6. **Sensitive data handling**: PII masking, secure temp files (0600), prompt injection defense — all well-tested

---

## Table 2: Codebase Metrics

| Component | Lines | Files | Notes |
|-----------|-------|-------|-------|
| bin/roachman | 2,858 | 1 | Main CLI, 13 menus |
| bin/roachie-nl | 338 | 1 | Interactive NL |
| bin/roachie-batch | 1,850 | 1 | Batch NL |
| src/lib/ (non-LLM) | 4,862 | 9 | cluster, replication, monitoring, etc. |
| src/lib/ (LLM) | 7,575 | 8 | prompt, assistant, providers, metrics |
| src/lib/providers/ | 1,162 | 6 | 5 providers + SSE parser |
| tools/lib/common.sh | 231 | 1 | Shared tool library |
| cr_* tools | ~5,000 | 78 | CockroachDB admin tools |
| Unit tests | 12,012 | 51 | ~500+ assertions |
| Integration tests | ~2,100 | 9 | Requires live cluster |
| **Total** | **~38,000** | **165** | |

---

## CLI Summary

```
+-----+----------+------------------------------------------------------+----------+--------+--------+
|  #  |   Area   |                   Recommendation                     | Priority | Effort | Status |
+-----+----------+------------------------------------------------------+----------+--------+--------+
|  R1 | AI       | Extract tool ranking into standalone module          |    P1    |  Med   |  NEW   |
|  R2 | AI       | Centralize message history management                |    P1    |  Med   |  NEW   |
|  R3 | App      | Consolidate SQL URL construction (3 impls)           |    P1    |  Med   |  NEW   |
|  R4 | App      | Extract structured error types with context           |    P2    |  Med   |  NEW   |
|  R5 | AI       | Provider circuit breaker + cross-provider fallback    |    P2    |  Med   |  NEW   |
|  R6 | Perf     | Parallelize tool help fetching with xargs -P          |    P2    |  Low   |  NEW   |
|  R7 | Test     | Enable --parallel in CI + test isolation              |    P2    |  Med   |  NEW   |
|  R8 | AI       | Token budget governor for prompt construction         |    P2    |  Med   |  NEW   |
|  R9 | App      | Decouple cluster.sh from replication.sh               |    P3    |  High  |  NEW   |
| R10 | Security | Validate learned keywords before regex injection      |    P3    |  Low   |  NEW   |
| R11 | Test     | Add negative/error-path test suites                  |    P3    |  Med   |  NEW   |
| R12 | AI       | Implement tiered model routing for cost optimization  |    P3    |  Med   |  NEW   |
| R13 | App      | Batch SQL execution within single docker exec         |    P3    |  Low   |  NEW   |
| R14 | Test     | Add test metadata tags + execution time tracking      |    P4    |  Low   |  NEW   |
+-----+----------+------------------------------------------------------+----------+--------+--------+

Strengths:  Security (5-layer validation) | Provider plugin pattern | RRF tool ranking (99%+)
            Persistent learning (DPO data) | Full observability | PII masking + audit logging
Codebase:   ~38K lines | 165 files | 78 cr_* tools | 5 LLM providers | 60 test suites
```
