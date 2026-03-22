# AI Architecture Review v14 — 2026-03-21

## Executive Summary

This review examines the complete AI/LLM subsystem of the roachie project: a natural language interface that translates English queries into CockroachDB DBA tool invocations. The system spans ~6,500 lines across 13 modules, supporting 5 LLM providers, 3 embedding providers, 77 CLI tools, and both interactive and batch execution modes.

**Overall Assessment**: The architecture is well-structured with clear separation of concerns, comprehensive security guards, and mature multi-provider support. The primary improvement opportunities are in token budget management (especially for Ollama), context quality optimization, and reducing LLM non-determinism through better prompt engineering.

---

## 1. Architecture Overview

### 1.1 Module Dependency Graph

```
Entry Points:
  bin/roachman ──────────────────────────┐
  bin/roachie-nl ──→ _llm_init_source_roachman() ──→ bin/roachman (silent)
  bin/roachie-batch ──→ _llm_init_source_roachman() ──→ bin/roachman (silent)
                                        │
                                        ▼
                               src/lib/llm.sh (orchestrator)
                                        │
          ┌─────────┬─────────┬─────────┼─────────┬─────────┬─────────┐
          ▼         ▼         ▼         ▼         ▼         ▼         ▼
    llm_config  llm_metrics  llm_exec  llm_providers  llm_voice  llm_prompt  llm_assistant
       (1)        (2)         (3)        (4)           (5)        (6)          (7)
                                         │
                    ┌────────┬───────┬───┴────┬─────────┐
                    ▼        ▼       ▼        ▼         ▼
               anthropic  openai  gemini   vertex    ollama
```

### 1.2 Data Flow (Per Query)

```
User Query
  │
  ├──→ [Input Validation] (2000 char limit, sanitization)
  │
  ├──→ [Tool Matching] regex_keywords.tsv + semantic embeddings
  │         │
  │         ├── Regex: bash [[ =~ ]] against 70 patterns (~1ms)
  │         └── Semantic: cosine similarity against 77 pre-computed embeddings (~100-500ms)
  │              └── Combined: semantic first, regex supplements, top 7, deduped
  │
  ├──→ [Prompt Construction]
  │         ├── Full prompt (~2,700 tokens) for cloud providers
  │         └── Compact prompt (~440 tokens) for Ollama
  │         ├── + Schema context (databases, tables, columns from cluster)
  │         ├── + Tool --help for top 5-7 matched tools
  │         ├── + Learning data (15 failures + 10 successes)
  │         └── + Connection reminder (mandatory flags)
  │
  ├──→ [LLM API Call] provider-specific (streaming or non-streaming)
  │         └── Response: JSON { reasoning, commands[], user_message, needs_followup }
  │
  ├──→ [Command Validation] 5-layer pipeline
  │         ├── L1: Whitelist (cr_*, cockroach sql/workload/node/debug)
  │         ├── L2: Metacharacter blocking (pipes, semicolons, subshells)
  │         ├── L3: SQL mutation guard (DROP, DELETE, ALTER, etc.)
  │         ├── L4: Tool existence check
  │         └── L5: RBAC role enforcement
  │
  ├──→ [Execution] array-based (no shell interpretation)
  │
  └──→ [Agent Loop] if needs_followup or command failed (max 3 iterations)
            ├── Reflexion: error analysis + alternative approach prompt
            └── Output truncated (4000 chars cloud, 800 chars Ollama)
```

### 1.3 Component Summary

| Module | Lines | Purpose |
|--------|-------|---------|
| llm_config.sh | ~90 | Centralized constants with env var overrides |
| llm_metrics.sh | ~450 | CSV logging, cost tracking, persistent learning, sensitive data masking |
| llm_exec.sh | ~300 | 5-layer command validation, RBAC, flag normalization |
| llm_providers.sh | ~750 | Provider dispatch, retry/backoff, fallback, token counting |
| llm_prompt.sh | ~1,540 | Prompt construction, tool enrichment, schema context, embeddings |
| llm_assistant.sh | ~1,660 | REPL loop, agent loop, reflexion, history management |
| llm_voice.sh | ~200 | Speech-to-text (Whisper API / local whisper) |
| llm_init.sh | ~120 | Shared initialization for entry points |
| providers/*.sh | ~1,400 | 5 provider plugins (API calls, streaming, response parsing) |
| **Total** | **~6,510** | |

---

## 2. Strengths

### 2.1 Multi-Provider Architecture
- 5 LLM providers (Anthropic, OpenAI, Gemini, Vertex AI, Ollama) with unified interface
- 3 embedding providers (Ollama/nomic-embed-text, OpenAI/text-embedding-3-small, Gemini/gemini-embedding-001)
- Automatic provider detection with priority chain and fallback
- Batch mode auto-selects fallback on API failure (no interactive prompt)

### 2.2 Two-Model Matching System
- Embedding model handles tool matching (semantic similarity)
- LLM handles reasoning and command generation
- Combined regex + semantic matching achieves 99-100% accuracy on 135 test queries
- Regex acts as intent gate: prevents semantic false positives on unrelated queries

### 2.3 Security Defense-in-Depth
- 5-layer command validation pipeline prevents injection attacks
- Metacharacter blocking prevents shell escapes (pipes, subshells, redirects)
- SQL mutation guard blocks destructive operations
- RBAC with 4 roles (admin, dba, analyst, monitor)
- Sensitive data masking in logs (emails, SSNs, credit cards, AWS keys, passwords)
- Prompt injection neutralization in `_sanitize_for_prompt()`
- Secure temp files (umask 0077) for API keys
- 113 security tests in test_nl_security.sh

### 2.4 Observability
- 19-column CSV query log with trace IDs
- JSONL feedback files for detailed analysis
- Per-query cost tracking (millicent precision, 50+ model pricing table)
- Session cost budgets with warnings at 80% and hard stop at 100%
- Debug log rotation (1MB, 5 backups)
- Batch reporting: JSONL + failures log + summary CSV

### 2.5 Persistent Learning
- Failure patterns (last 30 days, max 15) injected into prompts to avoid repeat mistakes
- Success patterns (last 10) reinforce known-good command mappings
- File-locked JSONL storage for concurrent access safety

---

## 3. Recommendations

### Priority Definitions
- **P1**: High impact on accuracy or reliability; fix recommended before next release
- **P2**: Moderate impact; improves robustness or maintainability
- **P3**: Low impact; nice-to-have improvements

---

### R1 (P1): Ollama Token Budget Validation

**Problem**: The compact prompt (~440 tokens) + enriched tool help (~750 tokens) + schema context (~150 tokens) + COMMON MISTAKES section (~300 tokens) can total ~1,640 tokens. With reflexion follow-up adding ~800 chars of output, the 4,096 token context limit is under pressure. There is no explicit validation that the assembled prompt fits within the context window before calling the API.

**Impact**: When the prompt exceeds context, Ollama silently truncates from the end — losing the COMMON MISTAKES section and connection reminder, which are the most critical sections for correct flag usage. This directly causes the wrong-flag failures seen in batch tests.

**Current State**: 5 chunking strategies (tool help limit, truncation, skip learning data, reduce reflexion output, skip doc RAG) provide headroom but no measurement.

**Recommendation**: Add explicit token budget accounting before the API call:
```
budget = _llm_context_limit("ollama") - _NL_MAX_OUTPUT_TOKENS
estimated = length(system_prompt) / _NL_CHARS_PER_TOKEN
if estimated > budget:
  progressively drop sections (learning data → extra tool help → schema)
  log warning with actual vs budget
```

**Files**: `llm_prompt.sh` (prompt assembly), `llm_assistant.sh` (pre-call check)

---

### R2 (P1): Move COMMON MISTAKES to Top of Compact Prompt

**Problem**: The COMMON MISTAKES section (20 tool-specific flag rules) is placed at the end of the compact prompt. When Ollama truncates context, this section is the first to be lost — yet it contains the most actionable guidance for correct command generation.

**Impact**: Batch tests show the model repeatedly uses wrong flags (e.g., `--name-pattern` instead of `--pattern` for cr_find_views, positional args instead of `-T TABLE` for cr_table_info) even though the correct flags are documented in COMMON MISTAKES.

**Recommendation**: Restructure the compact prompt to place COMMON MISTAKES immediately after the response format specification (before the tool listing), ensuring it survives any truncation. The tool listing is less critical since tool `--help` is injected via enrichment.

**Proposed order**:
1. Role + timestamp + topology
2. Response format (JSON spec)
3. COMMON MISTAKES (critical flag rules)
4. Flag rules (connection flags)
5. Tool listing (compact categories)
6. Dual-parameter exceptions

**Files**: `llm_prompt.sh:_nl_build_system_prompt_compact()`

---

### R3 (P1): Fix Semicolon Command Chaining

**Problem**: The model frequently chains multiple commands with semicolons in a single `command` field (e.g., `cr_ddl_function -d db1 ...; cr_ddl_procedure -d db1 ...`). This is blocked by the metacharacter guard, causing failures even though both commands individually are valid.

**Impact**: 3-5 failures per batch run are caused by semicolons alone. The COMMON MISTAKES section already documents this but the model often ignores it.

**Recommendation**: Two-pronged fix:
1. **Prompt**: Add a more prominent one-command-per-entry rule with a negative example directly in the response format spec (closer to where the model generates JSON).
2. **Runtime recovery**: In `_validate_command()`, when a semicolon is detected, attempt to split the command at `;` boundaries and validate/execute each part independently. This recovers valid commands that were incorrectly chained.

**Files**: `llm_prompt.sh` (prompt), `llm_exec.sh` or `llm_assistant.sh` (runtime split)

---

### R4 (P2): Reduce Compact Prompt Tool Listing Size

**Problem**: The compact tool listing includes all 77 tools grouped by category (~200 tokens). Since tool enrichment already injects the relevant 5 tools with `--help` output, the full listing is redundant for the model's decision-making and consumes precious context.

**Recommendation**: For Ollama, reduce the tool listing to only the enriched tools (top 5) plus a count of total available tools. The enriched tools already have full `--help`; the model doesn't need to see the other 72 tool names.

**Before** (~200 tokens):
```
AVAILABLE TOOLS:
  Performance: cr_query_stats, cr_query_history, cr_plan, cr_workload, cr_wait_events
  Tables/Size: cr_tables, cr_size, cr_db_size, ...
  [12 more categories]
```

**After** (~30 tokens):
```
77 tools available. The most relevant tools for your query are listed below with --help docs.
```

**Savings**: ~170 tokens, giving more room for COMMON MISTAKES and flag rules.

**Files**: `llm_prompt.sh:_nl_build_system_prompt_compact()`

---

### R5 (P2): Provider-Specific Similarity Thresholds

**Problem**: The cosine similarity threshold `_NL_MIN_SIMILARITY=0.35` is applied uniformly across all embedding providers, but higher-dimensional embeddings (Gemini 3072-dim) naturally produce tighter similarity distributions than lower-dimensional ones (Ollama 768-dim).

**Impact**: Ollama embeddings may return false-positive tool matches at 0.35, while Gemini may miss valid matches.

**Recommendation**: Use provider-specific thresholds:
| Provider | Dimensions | Recommended Threshold |
|----------|-----------|----------------------|
| Ollama | 768 | 0.40 |
| OpenAI | 1536 | 0.35 |
| Gemini | 3072 | 0.30 |

**Files**: `llm_prompt.sh:_semantic_match_tools()`, `llm_config.sh`

---

### R6 (P2): Cache gcloud Access Token

**Problem**: `_detect_llm_provider()` and every Vertex AI API call invoke `gcloud auth print-access-token`, which adds ~500ms latency per call. For a 3-iteration agent loop, this adds ~1.5s of unnecessary overhead.

**Recommendation**: Cache the token with a TTL (tokens are valid for 3600s; use 3000s cache):
```bash
_get_gcloud_token() {
  if [[ -n "${_GCLOUD_TOKEN:-}" && $(( $(date +%s) - ${_GCLOUD_TOKEN_TS:-0} )) -lt 3000 ]]; then
    echo "$_GCLOUD_TOKEN"; return
  fi
  _GCLOUD_TOKEN=$(gcloud auth print-access-token 2>/dev/null)
  _GCLOUD_TOKEN_TS=$(date +%s)
  echo "$_GCLOUD_TOKEN"
}
```

**Files**: `providers/vertex.sh`, `llm_providers.sh`

---

### R7 (P2): RBAC Silent Fallback

**Problem**: `_nl_validate_role()` silently falls back to "admin" for invalid role values. A typo like `ROACHIE_ROLE=analyse` grants full admin access without warning.

**Recommendation**: Log a warning when fallback occurs:
```bash
_nl_validate_role() {
  case "${1:-}" in
    admin|dba|analyst|monitor) echo "$1" ;;
    *) echo "WARNING: Invalid role '${1:-}', defaulting to admin" >&2; echo "admin" ;;
  esac
}
```

**Files**: `llm_config.sh`

---

### R8 (P2): Batch Learning Data Updates

**Problem**: Batch mode loads persistent failures/successes at startup but never saves new entries during the run. A 73-prompt batch run generates valuable signal that is discarded.

**Impact**: Interactive sessions learn from past mistakes; batch runs don't contribute to this learning.

**Recommendation**: Add `--enable-learning` flag to `roachie-batch` that saves failures and successes to the persistent databases after each prompt. Default: off (to avoid noisy writes during testing).

**Files**: `bin/roachie-batch`

---

### R9 (P2): Exponential Backoff Cap

**Problem**: The retry logic in `_call_llm_api()` doubles the backoff delay on each retry (2s -> 4s -> 8s -> ...) with no upper bound. While current max retries (1-3) limit this, the lack of a cap is fragile.

**Recommendation**: Cap backoff at 30 seconds:
```bash
local backoff_delay=$(( base_delay * (2 ** attempt) ))
[[ $backoff_delay -gt 30 ]] && backoff_delay=30
```

**Files**: `llm_providers.sh`

---

### R10 (P2): Input Length Validation in Batch Mode

**Problem**: Interactive REPL validates input length (2000 chars), but batch mode does not check per-prompt length. Long prompts could exceed token budgets.

**Recommendation**: Apply the same 2000-char validation in batch mode with a warning and skip.

**Files**: `bin/roachie-batch`

---

### R11 (P3): Streaming for Ollama

**Problem**: Ollama uses `stream: false` for all requests, meaning users see no output until the entire response is generated (~8-15s for roachie-8b). Cloud providers stream tokens progressively.

**Recommendation**: Implement SSE streaming for Ollama using the same pattern as Anthropic/OpenAI. Ollama's API supports `stream: true` with JSON chunks.

**Files**: `providers/ollama.sh`

---

### R12 (P3): Parallel Command Execution

**Problem**: When the LLM generates multiple commands, they are executed sequentially. Independent commands (e.g., `cr_ddl_function` and `cr_ddl_procedure` for the same database) could run in parallel.

**Impact**: Minor latency improvement for multi-command queries.

**Recommendation**: Detect independent commands (no output dependency) and execute in parallel with `&` + `wait`. Defer to a future release — complexity outweighs benefit for most queries.

**Files**: `llm_assistant.sh`, `bin/roachie-batch`

---

### R13 (P3): Structured Error Categories in Batch Reports

**Problem**: Batch reports use a single `failed` status for all failure types. There's no distinction between:
- Validation failures (wrong flags, blocked metachar)
- Execution failures (tool returned error)
- Timeout failures
- API failures

**Recommendation**: Add a `failure_reason` field to JSONL results:
```json
{"status": "failed", "failure_reason": "validation_blocked_metachar", ...}
```

This enables targeted analysis of failure causes without parsing the free-text `commands_executed` field.

**Files**: `bin/roachie-batch`

---

### R14 (P3): Cost Tracking in Batch Summary

**Problem**: Per-query cost is calculated but not aggregated in the batch summary CSV or console output.

**Recommendation**: Add `total_cost` field to the summary section and CSV output.

**Files**: `bin/roachie-batch`

---

### R15 (P3): Token Count Validation

**Problem**: `_NL_MAX_OUTPUT_TOKENS` is not validated against provider-specific limits. A misconfigured value (e.g., 100,000 for Ollama) would cause silent API failures.

**Recommendation**: Validate max tokens against `_llm_context_limit()` at initialization:
```bash
if [[ $_NL_MAX_OUTPUT_TOKENS -gt $(_llm_context_limit "$provider") ]]; then
  warning "max_tokens ($tokens) exceeds $provider context ($limit), capping"
fi
```

**Files**: `llm_config.sh` or `llm_assistant.sh`

---

## 4. Provider Capability Matrix

| Capability | Anthropic | OpenAI | Gemini | Vertex AI | Ollama |
|-----------|-----------|--------|--------|-----------|--------|
| Streaming | Yes | Yes | Yes | No | No |
| Native Tool Calling | Yes | Yes | Yes | Yes | No |
| System Prompt | Full (~2.7K tokens) | Full | Full | Full | Compact (~440 tokens) |
| Schema Context | Full | Full | Full | Full | Truncated (15 lines) |
| Learning Data | Yes (15+10) | Yes | Yes | Yes | Skipped |
| Doc RAG | Yes (3 chunks) | Yes | Yes | Yes | Skipped |
| Tool Help | Top 7 (full) | Top 7 | Top 7 | Top 7 | Top 5 (truncated 15 lines) |
| Token Budget | 200K | 128K | 1M | 200K | 4K (configurable) |
| Cost Tracking | Yes (50+ models) | Yes | Yes | Yes | Yes (free) |
| Retry on Error | 1 retry + backoff | 1 retry | 1 retry | 1 retry | 1 retry |
| Rate Limit Handling | 3 retries, 5s backoff | 3 retries | 3 retries | 3 retries | N/A |
| Fallback Provider | Yes | Yes | Yes | Yes | Yes |

---

## 5. Test Coverage

| Test Suite | Tests | Scope |
|-----------|-------|-------|
| test_llm_prompt.sh | 14 | Regex matching, enrichment, tool count limits |
| test_ollama_chunking.sh | 25 | All 5 chunking strategies |
| test_nl_security.sh | 113 | Metachar blocking, SQL injection, RBAC, sanitization |
| test_provider_fallback.sh | ~15 | Fallback logic, batch mode, infinite loop prevention |
| test_llm_streaming.sh | ~20 | SSE parsing, token extraction, error handling |
| test_llm_providers.sh | ~15 | Provider detection, dispatch, cost calculation |
| test_embeddings.sh | 135 | Semantic + regex accuracy (3 providers x 2 modes) |
| **Total** | **~337** | |

**Coverage Gaps**:
- No tests for Ollama token budget overflow scenarios
- No tests for gcloud token caching (feature doesn't exist yet)
- No integration tests with real LLM APIs
- No tests for semicolon command splitting (R3 recommendation)
- No tests for prompt section ordering impact on accuracy

---

## 6. Accuracy Metrics

### 6.1 Tool Matching (135 queries, pre-computed)

| Provider | Multi-Tenant (71) | Single-Tenant (64) | Combined |
|----------|-------------------|---------------------|----------|
| Gemini (3072-dim) | 100% | 98.4% | 100% |
| OpenAI (1536-dim) | 98.5% | 96.8% | 100% |
| Ollama (768-dim) | 97.1% | 93.7% | 99.3% |
| Regex baseline | 100% | 96.8% | — |

### 6.2 End-to-End Batch (73 prompts, Ollama roachie-8b)

| Run | Pass | Fail | Rate | Notes |
|-----|------|------|------|-------|
| Baseline (pre-fixes) | 45 | 28 | 61.6% | Before COMMON MISTAKES, regex improvements |
| After prompt fixes | 56 | 17 | 76.7% | Best observed run |
| Re-run (LLM variance) | 53 | 20 | 72.6% | Same code, different outputs |

**Failure Categories** (from best run, 17 failures):
- Expected blocks (SQL file, Cluster B down): 3
- Wrong flags despite COMMON MISTAKES: 4
- Semicolon chaining: 3
- Tool limitations (cr_migrate schema filter): 2
- LLM non-determinism: 5

---

## 7. Recommendation Summary

| ID | Priority | Title | Effort | Impact |
|----|----------|-------|--------|--------|
| R1 | P1 | Ollama token budget validation | Medium | Prevents silent context truncation |
| R2 | P1 | Move COMMON MISTAKES to top of compact prompt | Small | Survives truncation; fixes wrong-flag failures |
| R3 | P1 | Fix semicolon command chaining | Medium | Recovers 3-5 failures per batch run |
| R4 | P2 | Reduce compact prompt tool listing | Small | Saves ~170 tokens for Ollama |
| R5 | P2 | Provider-specific similarity thresholds | Small | Reduces false positives/negatives |
| R6 | P2 | Cache gcloud access token | Small | Saves ~500ms per Vertex AI call |
| R7 | P2 | RBAC silent fallback warning | Trivial | Security improvement |
| R8 | P2 | Batch learning data updates | Small | Batch runs contribute to learning |
| R9 | P2 | Exponential backoff cap | Trivial | Prevents unbounded waits |
| R10 | P2 | Input length validation in batch | Trivial | Consistency with REPL |
| R11 | P3 | Streaming for Ollama | Medium | Better UX for local models |
| R12 | P3 | Parallel command execution | High | Minor latency improvement |
| R13 | P3 | Structured error categories in batch | Small | Better failure analysis |
| R14 | P3 | Cost tracking in batch summary | Trivial | Budget visibility |
| R15 | P3 | Token count validation | Small | Prevents misconfiguration |

**P1 items (R1-R3)**: Expected to improve Ollama batch pass rate from 72-77% to 80-85%.

---

## 8. Architecture Diagram

```
                          ┌─────────────────────────────────────────┐
                          │           User Interface Layer           │
                          │  roachie-nl (REPL)  roachie-batch (CI)  │
                          └────────────┬───────────┬────────────────┘
                                       │           │
                          ┌────────────▼───────────▼────────────────┐
                          │         Orchestration Layer              │
                          │  llm_assistant.sh                       │
                          │  - Agent loop (max 3 iterations)        │
                          │  - Reflexion (self-correction on error)  │
                          │  - History management & truncation       │
                          │  - Feedback collection & learning        │
                          └──┬──────────┬──────────┬────────────────┘
                             │          │          │
              ┌──────────────▼──┐  ┌────▼────┐  ┌─▼──────────────┐
              │  Prompt Layer   │  │ Security │  │ Provider Layer │
              │  llm_prompt.sh  │  │ llm_exec │  │ llm_providers  │
              │                 │  │          │  │                │
              │ - Full/compact  │  │ Whitelist│  │ Dispatch +     │
              │ - Enrichment    │  │ Metachar │  │ retry/backoff  │
              │ - Schema inject │  │ SQL guard│  │ fallback       │
              │ - Embeddings    │  │ RBAC     │  │ cost tracking  │
              └──┬──────────────┘  └──────────┘  └──┬─────────────┘
                 │                                   │
    ┌────────────▼──────────────┐     ┌──────────────▼──────────────┐
    │    Matching Layer         │     │     Provider Plugins        │
    │                          │     │                             │
    │ regex_keywords.tsv (70)  │     │ anthropic.sh  (streaming)  │
    │ tool_descriptions.txt    │     │ openai.sh     (streaming)  │
    │ embeddings/ (3 providers)│     │ gemini.sh     (streaming)  │
    │ cosine similarity (jq)   │     │ vertex.sh     (non-stream) │
    └──────────────────────────┘     │ ollama.sh     (non-stream) │
                                     └──────────────────────────────┘
              ┌────────────────────────────────────────┐
              │         Support Layer                   │
              │  llm_config.sh  (constants + env vars) │
              │  llm_metrics.sh (CSV, cost, learning)  │
              │  llm_voice.sh   (Whisper STT)          │
              │  llm_init.sh    (bootstrap utilities)  │
              └────────────────────────────────────────┘
```

---

*Review performed: 2026-03-21*
*Modules analyzed: 13 files, ~6,510 lines*
*Previous review: ai-architecture-review-v13 (2026-03-19)*
