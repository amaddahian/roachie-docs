# AI Architecture Review v6 — 2026-03-16

Comprehensive review of roachie's AI/NL subsystem: LLM providers, prompt construction,
command execution, batch pipeline, MCP integration, test coverage, and security.

**Scope**: ~7,500 lines across 9 LLM modules + MCP toolbox + test suites.

**Prior reviews**: v1–v5 (v1–v4 deleted; v5 items carried forward below).

---

## Table 1: Module Inventory

| Module | Lines | Purpose | Test Coverage |
|--------|-------|---------|---------------|
| `llm_providers.sh` | 1,773 | 5 LLM providers, streaming, tool calling, retry | 85% |
| `llm_assistant.sh` | 1,460 | REPL, command extraction, agent loop, feedback | 60% |
| `llm_prompt.sh` | 1,650 | Prompt construction, semantic matching, schema context | 50% |
| `llm_metrics.sh` | 650 | Cost tracking, CSV logging, persistent learning | 75% |
| `llm_exec.sh` | 162 | Whitelist, metachar, SQL mutation guards | 95% |
| `llm_voice.sh` | ~300 | Voice input, STT (Whisper cloud + local) | 40% |
| `llm_init.sh` | 58 | Module sourcing, project root detection | 0% |
| `roachie-nl` | 275 | Standalone interactive NL entry point | N/A |
| `roachie-batch` | 1,254 | Batch NL test runner | N/A |
| **Total** | **~7,582** | | |

---

## Table 2: Recommendations

| ID | Priority | Category | Recommendation | Effort | Status |
|----|----------|----------|----------------|--------|--------|
| R1 | P0 | Security | Sanitize LLM reasoning before follow-up re-enrichment | 30 min | |
| R2 | P0 | Correctness | Initialize `_VALIDATE_BLOCK_REASON` per command iteration | 10 min | |
| R3 | P1 | Correctness | Fix multi-iteration message JSON construction in batch | 30 min | |
| R4 | P1 | Providers | Standardize streaming error detection across providers | 2 hrs | |
| R5 | P1 | Providers | Fix OpenAI rate-limit case-sensitivity (`"Rate limit"` vs `"rate limit"`) | 10 min | |
| R6 | P1 | Providers | Stop treating all non-JSON Gemini responses as rate limits | 20 min | |
| R7 | P1 | Metrics | Add cost calculation to batch pipeline | 1 hr | |
| R8 | P1 | Testing | Add semantic matching test suite (0 tests today) | 2 hrs | |
| R9 | P1 | Testing | Add schema context injection tests (0 tests today) | 1.5 hrs | |
| R10 | P2 | Correctness | Add persistent learning DB locking for concurrent batch runs | 1 hr | |
| R11 | P2 | Testing | Add agent loop iteration-limit and output-truncation tests | 1 hr | |
| R12 | P2 | Testing | Fix always-pass history trimming test | 30 min | |
| R13 | P2 | Providers | Add empty response handling for Anthropic/OpenAI (Ollama validates, they don't) | 30 min | |
| R14 | P2 | Metrics | Reset iteration-level token counters on API error | 15 min | |
| R15 | P2 | Batch | Fix dry-run mode false positives (marks blocked commands as "success") | 30 min | |
| R16 | P3 | Docs | Document `NL_MAX_ITERATIONS`, `NL_MIN_SIMILARITY` env vars in `--help` | 20 min | |
| R17 | P3 | Docs | Add voice feature to `roachie-nl --help` | 15 min | |
| R18 | P3 | Prompt | Add schema fetch timeout (`timeout 5 cockroach sql ...`) | 15 min | |
| R19 | P3 | Prompt | Pre-load regex keywords TSV at session init (avoid re-read per query) | 30 min | |

### Carry-Forward from v5

| v5 ID | Status | Description |
|-------|--------|-------------|
| v5-R7 | **Open (= R4)** | Standardize streaming error detection — port Gemini's `_sse_raw` to Anthropic/OpenAI |
| v5-R9 | Deferred (P2) | Rate limiting parity between interactive and batch |
| v5-R11 | Deferred (P3) | Reduce jq subprocess overhead in command extraction loop |

### Carry-Forward from v4

| v4 ID | Status | Description |
|-------|--------|-------------|
| v4-R5 | Open (P3) | Consolidate tool flag docs (duplicate in two prompt sections) |
| v4-R6 | **Open (= R11)** | Tests for history trimming and agent loop |
| v4-R7 | **Open (= R12)** | Fix "always-pass" test patterns |

---

## Detailed Findings

### R1 — Sanitize LLM Reasoning Before Re-Enrichment (P0)

**File**: `bin/roachie-batch` lines 746–756, `src/lib/llm_assistant.sh` line 1294

In the agent follow-up loop, the LLM's `reasoning` output is concatenated with the user's
original query to re-enrich the system prompt:

```bash
_enrich_query="${line} ${reasoning:-}"
system_prompt=$(_enrich_system_prompt_with_tool_help "$_enrich_query" ...)
```

The `reasoning` field comes directly from the LLM response and is **not sanitized**.
A compromised or manipulated LLM could inject tool names into reasoning to steer follow-up
tool selection.

**Fix**: Re-enrich using only the original user query, or truncate reasoning to first 100 chars.

---

### R2 — Initialize `_VALIDATE_BLOCK_REASON` Per Command (P0)

**File**: `src/lib/llm_exec.sh` lines 108–138

`_VALIDATE_BLOCK_REASON` is set on validation failure but never cleared on success.
If command N fails validation (sets `BLOCKED`), then command N+1 passes, the variable
still holds the previous value.

**Fix**: Add `_VALIDATE_BLOCK_REASON=""` at the top of `_validate_command()`.

---

### R3 — Fix Multi-Iteration Message JSON in Batch (P1)

**File**: `bin/roachie-batch` lines 982–998

Follow-up message construction appends both user + assistant messages in step 1,
then appends another user message with the same content in step 2, creating duplicates.

**Fix**: Consolidate into single jq pipeline appending assistant then user messages.

---

### R4 — Standardize Streaming Error Detection (P1, v5-R7)

**File**: `src/lib/llm_providers.sh`

Gemini streaming (lines 1323–1403) has robust post-stream diagnostics (raw SSE capture,
empty response detection, non-SSE response detection, zero text extraction).
Anthropic and OpenAI only check `jq has("error")` inside the SSE loop.

Also, Gemini's error JSON format lacks the `"type"` field that Anthropic/OpenAI include.

**Fix**: Port Gemini's diagnostics pattern to Anthropic and OpenAI. Standardize error format.

---

### R5 — Fix OpenAI Rate-Limit Case Sensitivity (P1)

**File**: `src/lib/llm_providers.sh` line 1672

Uses `"Rate limit"` with capital R. Some OpenAI errors use lowercase. Anthropic safely
matches `"rate"` (lowercase).

**Fix**: Use case-insensitive match or match lowercase `"rate"`.

---

### R6 — Gemini Non-JSON Response Handling (P1)

**File**: `src/lib/llm_providers.sh` lines 1694–1695

Any non-JSON response (including 401/403 HTML pages) is treated as `RATE_LIMITED`,
masking auth errors behind retry logic.

**Fix**: Check HTTP status code. Only treat 429 as rate limited; 401/403 are auth errors.

---

### R7 — Add Cost Calculation to Batch Pipeline (P1)

**File**: `bin/roachie-batch` lines 777–781, 1218–1227

Token counts are tracked but `_llm_calculate_cost_cents()` is never called.
Interactive mode tracks costs correctly.

**Fix**: Call cost calculation after token extraction. Add `cost_millicents` to batch CSV.

---

### R8 — Add Semantic Matching Test Suite (P1)

**File**: `src/lib/llm_prompt.sh` lines 57–152

The semantic matching pipeline has 0 unit tests. Needs tests for: embedding provider
fallback chain, cosine similarity, similarity threshold, intent gate, dedup, combined top-7.

---

### R9 — Add Schema Context Injection Tests (P1)

**File**: `src/lib/llm_prompt.sh` lines 1192–1346

Schema context injection has 0 unit tests. Needs tests for: missing CLI fallback,
connection failure, database truncation, table truncation, session caching, SSL URLs.

---

### R10 — Persistent Learning DB Locking (P2)

**File**: `src/lib/llm_metrics.sh` lines 444–450

Concurrent batch runs can race on persistent failures/successes JSONL files.

**Fix**: Use `flock` for atomic read-modify-write.

---

### R11 — Add Agent Loop Tests (P2, v4-R6)

Needs tests for: iteration cap at `NL_MAX_ITERATIONS`, output truncation to 2000 chars,
`NL_MAX_ITERATIONS` env var respected, system prompt reused across iterations.

---

### R12 — Fix Always-Pass History Trimming Test (P2, v4-R7)

**File**: `tests/unit/test_nl_pipeline.sh` lines 323–341

Test creates 40 short messages (~1,000 tokens), far below 100K budget. Trimming never activates.

**Fix**: Lower budget for test or generate larger messages.

---

### R13 — Empty Response Handling for Cloud Providers (P2)

Ollama validates empty responses explicitly. Anthropic/OpenAI silently return empty strings.

**Fix**: Add empty response detection after text extraction.

---

### R14 — Reset Token Counters on API Error (P2)

On API failure in later agent iterations, token counters retain values from prior iterations.

**Fix**: Reset `prompt_tokens=0` and `completion_tokens=0` on error.

---

### R15 — Dry-Run Mode False Positives (P2)

Dry-run treats all prompts as "success" even when commands would be blocked.

**Fix**: Run validation checks in dry-run; only skip execution.

---

### R16–R19 — Documentation and Minor Improvements (P3)

- R16: Document env vars in `--help`
- R17: Add voice feature to `roachie-nl --help`
- R18: Add schema fetch timeout
- R19: Pre-load regex keywords at session init
