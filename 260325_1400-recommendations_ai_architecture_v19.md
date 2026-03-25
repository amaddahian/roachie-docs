# AI Architecture Review v19 -- Recommendations

**Date:** 2026-03-25
**Reviewer:** Claude Opus 4.6
**Scope:** Full AI/NL subsystem (~6,500 lines across 14 files + 5 provider plugins), training pipeline, embeddings, MCP, test coverage
**Previous:** v18 (2026-03-24, 6 items), v17 (2026-03-24, 15/18 fixed), v16 (2026-03-23), v13 (all fixed)

---

## Maturity Assessment

| Dimension | Score | v18 | Delta | Notes |
|-----------|-------|-----|-------|-------|
| Security | 9.3/10 | 9.2 | +0.1 | v18 R1 (few-shot) fixed. R4 (reflexion sanitize) fixed. New findings: preferences file sourcing (R5), training data lacks system prompt (R1). |
| Provider Architecture | 9.0/10 | 9.0 | -- | Solid. Shared builders, unified dispatch. Streaming gap for Ollama remains (by design). |
| Tool Matching | 9.5/10 | 9.5 | -- | Combined matching at 99-100%. Regex gate + semantic priority proven stable. |
| Prompt Engineering | 8.8/10 | 8.2 | +0.6 | Few-shot examples corrected (v18-R1). Doc RAG infrastructure in place. System prompt well-structured. |
| Error Handling | 9.0/10 | 8.8 | +0.2 | Feedback timeout (v18-R3), cmd_count validation (v18-R2) fixed. |
| Observability | 8.3/10 | 8.2 | +0.1 | Good metrics pipeline. Missing: prompt size tracking vs context limit (R3). |
| Local Model (Ollama) | 8.5/10 | 8.5 | -- | 90% LoRA accuracy. Training data distribution mismatch (R1) limits ceiling. |
| Test Coverage | 8.0/10 | 7.8 | +0.2 | 38 unit suites. Gaps: no training data validation tests, no prompt size regression tests. |
| Code Quality | 8.7/10 | 8.5 | +0.2 | ShellCheck clean. Inverted return convention in metachar/SQL guards documented (R7). |
| Fine-Tuning Pipeline | 7.5/10 | N/A | New | First assessment. Pipeline functional but training data quality issues (R1, R2, R8). |
| **Overall** | **8.8/10** | **8.6** | **+0.2** | Steady improvement. Training pipeline is the main area for growth. |

---

## Recommendations

### P1 -- Critical (1 item)

#### R1: Training Data Distribution Mismatch -- No System Prompt in Fine-Tuning Examples
**Files:** `tools/utils/finetune/prepare_training_data.py:62-77`
**Issue:** Training examples contain only user/assistant message pairs:
```python
{"messages": [
    {"role": "user", "content": "show slowest queries on va tenant"},
    {"role": "assistant", "content": "{\"reasoning\": ..., \"commands\": [...]}"}
]}
```

At inference time, the model receives a rich system prompt (3-4K tokens) containing:
- Cluster topology and connection parameters
- Database schema context
- Tool catalog (77 tools)
- Parameter rules, few-shot examples
- Execution rules and JSON response format

The model was fine-tuned without ever seeing this system prompt, creating a **train/test distribution mismatch**. The model learned to generate commands from raw user queries alone, but at runtime it must parse and follow instructions from a system prompt it never trained on. This explains the 90% accuracy ceiling -- the remaining 10% likely fails on cases where the system prompt's connection rules or parameter syntax matter most.

**Fix:** Include the system prompt in training examples:
```python
def build_training_example(prompt, reasoning, command, system_prompt=None):
    messages = []
    if system_prompt:
        messages.append({"role": "system", "content": system_prompt})
    messages.append({"role": "user", "content": prompt})
    messages.append({"role": "assistant", "content": response})
    return {"messages": messages}
```

Generate a representative system prompt at training time using the same `_nl_build_system_prompt()` output (or a snapshot). This teaches the model to follow the connection parameter rules, flag syntax, and JSON format that it will see at inference.

**Severity:** Critical -- primary bottleneck for local model accuracy (90% ceiling)
**New in v19**

---

### P2 -- High Priority (3 items)

#### R2: Training Data Lacks Negative Examples and Multi-Turn Interactions
**Files:** `tools/utils/finetune/prepare_training_data.py:80-169`
**Issue:** Training data consists exclusively of single-turn, successful command examples. Two gaps:

1. **No negative examples** -- The model never sees queries that should produce zero commands (informational responses). Examples: "what does cr_tables do?", "explain the difference between tenant and database", "what version of CockroachDB is running?". Without these, the model has a bias toward always generating a command, even when the user is asking a question.

2. **No multi-turn examples** -- All training examples are single user/assistant turns. The model never sees follow-up patterns (`needs_followup: true`) or reflexion sequences. At inference, multi-step queries rely entirely on the system prompt's instructions.

**Fix:**
1. Add 15-20 informational examples with `"commands": []`
2. Add 5-10 multi-turn examples showing `needs_followup: true` followed by a follow-up response
3. Optionally mine negative examples from batch runs where the model correctly answered without commands

**Severity:** High -- model over-generates commands for informational queries
**New in v19**

---

#### R3: No System Prompt Size Guard Against Context Overflow
**Files:** `src/lib/llm_assistant.sh:1354-1358`, `src/lib/llm_prompt.sh:476-537`
**Issue:** The history trimming at `_nl_call_and_parse()` (line 556-608) guards against total context overflow by trimming messages when `est_tokens > 80% of context_limit`. However, this only considers `system_prompt + messages`. The system prompt itself is never size-checked.

With full enrichment, the system prompt can grow to:
- Base prompt: ~3-4K tokens (~12-16K chars)
- 7 enriched tools x ~500 tokens each: ~3.5K tokens
- Doc RAG chunks: up to ~1.5K tokens
- Learning data: up to ~1K tokens
- **Total: ~9-10K tokens**

For Ollama with a 16K context limit, this leaves only 6-7K tokens for the entire conversation (messages + response). After just 2-3 exchanges, history trimming kicks in aggressively, dropping context the user just provided.

For cloud providers (128K-1M context) this is irrelevant. But for Ollama, the compact prompt path (`_nl_build_system_prompt_compact`) mitigates this partially -- yet enrichment still adds up to 5 full tool `--help` docs on top.

**Fix:** After enrichment, check system prompt size against the context limit:
```bash
local _sys_chars=${#system_prompt}
local _sys_tokens=$(( _sys_chars / _NL_CHARS_PER_TOKEN ))
local _ctx_limit=$(_llm_context_limit "$llm_provider")
local _max_sys_pct=60  # System prompt should use at most 60% of context
if [[ $_sys_tokens -gt $(( _ctx_limit * _max_sys_pct / 100 )) ]]; then
    _nl_debug "PROMPT" "WARNING: system prompt ${_sys_tokens} tokens exceeds ${_max_sys_pct}% of ${_ctx_limit} limit"
    # Truncate enriched tool docs to fit
fi
```

**Severity:** High -- affects Ollama conversation depth in multi-turn sessions
**New in v19**

---

#### R4: Budget Validation Accepts Non-Numeric Input
**Files:** `src/lib/llm_assistant.sh:1585-1596`
**Issue:** The budget calculation uses awk to convert `NL_BUDGET` to millicents:
```bash
_budget_mc=$(awk -v b="$_session_budget" 'BEGIN { printf "%d", b * 100000 }')
```
If `NL_BUDGET` is non-numeric (e.g., `NL_BUDGET=five` or `NL_BUDGET=""`), awk silently returns `0`. Since `_current_mc >= 0` is always true, the **first query** immediately triggers the "budget exceeded" warning, confusing the user.

**Fix:** Validate at session init:
```bash
if [[ -n "$_session_budget" ]] && ! [[ "$_session_budget" =~ ^[0-9]+\.?[0-9]*$ ]]; then
    _printc "$YELLOW_BOLD" "Warning: NL_BUDGET='$_session_budget' is not numeric -- budget tracking disabled"
    _session_budget=""
fi
```

**Severity:** High -- confusing UX when budget is misconfigured
**New in v19**

---

### P3 -- Medium Priority (5 items)

#### R5: User Preferences File Sourced Without Validation
**Files:** `src/lib/llm_assistant.sh:1141-1147`
**Issue:** `~/.roachie/config.sh` is sourced directly:
```bash
source "$_prefs_file" 2>/dev/null
```
While this is a user-controlled file (not an attack vector in typical use), it creates a subtle risk: if the file contains syntax errors or unintended side effects (e.g., `set -e`, `trap` overrides, `PATH` modifications), the NL session silently inherits broken state. There is no validation of file contents or isolation.

**Fix:** Source in a subshell for validation first, then source for real:
```bash
if bash -n "$_prefs_file" 2>/dev/null; then
    source "$_prefs_file" 2>/dev/null && \
        _printc "$CYAN_BOLD" "Loaded preferences from $_prefs_file"
else
    _printc "$YELLOW_BOLD" "Warning: syntax error in $_prefs_file -- skipping"
fi
```

**Severity:** Medium -- defense-in-depth; prevents silent session corruption
**New in v19**

---

#### R6: Doc RAG Query Magnitude Recomputed Per-Chunk
**Files:** `src/lib/llm_prompt.sh:264-276`
**Issue:** In `_doc_rag_retrieve()`, the jq cosine similarity computation includes:
```jq
([ $qe[] | . * . ] | add | sqrt) as $mag_q |
```
This computes `$mag_q` (the query vector magnitude) **inside the `.chunks | to_entries[]` loop**, meaning it is recomputed for every doc chunk. Since the query embedding is identical across all chunks, this is redundant work.

The tool matching function `_semantic_match_tools()` has the same pattern (line 209), but with only 77 tools the overhead is negligible. Doc chunks could number in the hundreds, making the redundancy more impactful.

**Fix:** Compute `$mag_q` once outside the loop:
```jq
($qe | [.[] | . * .] | add | sqrt) as $mag_q |
.chunks | to_entries[] |
...
(if ($mag_q == 0 or $mag_t == 0) then 0 else ($dot / ($mag_q * $mag_t)) end) as $sim
```

**Severity:** Medium -- performance optimization for Doc RAG (saves ~50ms per query with 100+ chunks)
**New in v19**

---

#### R7: Inverted Return Values in Metachar/SQL Guards
**Files:** `src/lib/llm_exec.sh:141-170, 172-198`
**Issue:** Both `_check_metachar()` and `_check_sql_mutation()` use inverted return conventions:
- Returns `0` (bash "success") when the check **blocks** the command
- Returns `1` (bash "failure") when the command is **clean**

This is counterintuitive and error-prone for maintainers. The callers use `if _check_metachar "$cmd"; then ... blocked ...` which reads correctly in English but violates the standard bash convention where 0 = success.

The functions are well-documented (comments at lines 145-146 and 175-176), but every new developer reading this code will do a double-take.

**Fix:** While a rename would be ideal (`_has_metachar` or `_is_blocked_metachar`), the lowest-risk change is to add inline comments at every call site:
```bash
if _check_metachar "$cmd_to_run"; then  # returns 0=blocked
```
Or, for a cleaner long-term solution, rename to `_has_metachar()` and `_has_sql_mutation()` to make the boolean semantics clear (returns 0 = "yes, has metachar" = blocked).

**Severity:** Medium -- maintainability; risk of future regression if caller misreads convention
**New in v19**

---

#### R8: Training Data Sanitization Incomplete for Multi-Source Commands
**Files:** `tools/utils/finetune/prepare_training_data.py:32-59`
**Issue:** `sanitize_command()` fixes 3 known bad patterns (cr_ddl_table, cr_help, cr_columns). But the training data may contain other patterns that violate current prompt rules:

1. **`--flag=value` syntax** -- Several tools produce `--format=json` or `--top=5` in batch output, but `parameter_rules.txt` requires `--format json` (space-separated). The model trains on `=` syntax and reproduces it at inference, where `_validate_command()` normalizes it (line 221) -- so it works, but the model learns a suboptimal pattern.

2. **Shortened flags** -- `-h` instead of `--host`, `-p` instead of `--port`. The model may learn these shortcuts and reproduce them even though the rules say "NEVER use -h, -p shorthand."

3. **Missing `--insecure`** -- Some batch results omit `--insecure` when the cluster was configured without TLS. The model learns to omit it, then fails on clusters that need it.

**Fix:** Extend `sanitize_command()`:
```python
def sanitize_command(cmd):
    # ... existing fixes ...
    # Normalize --flag=value to --flag value
    cmd = re.sub(r'(--[a-zA-Z][-a-zA-Z0-9]*)=(\S+)', r'\1 \2', cmd)
    # Ensure --insecure is present for cr_* commands (if not already)
    if cmd.startswith('cr_') and '--insecure' not in cmd:
        cmd += ' --insecure'
    return cmd
```

**Severity:** Medium -- training data quality affects model accuracy
**New in v19**

---

#### R9: Ollama Streaming Not Implemented
**Files:** `src/lib/providers/ollama.sh:33`
**Issue:** Ollama hardcodes `"stream": false` while all cloud providers default to streaming. For complex queries generating long responses (e.g., DDL generation, multi-table analysis), the user sees no output for 5-30 seconds on slower hardware. Cloud providers show progress dots during streaming.

Ollama's `/api/chat` endpoint supports streaming (returns NDJSON chunks). The same progress-dot pattern used by Anthropic/OpenAI/Gemini could be applied.

**Fix:** Implement `_call_ollama_api_stream()` following the same pattern as other providers:
```bash
while IFS= read -r line; do
    content=$(echo "$line" | jq -r '.message.content // empty')
    [[ -n "$content" ]] && full_response+="$content"
    chunk_count=$((chunk_count + 1))
    (( chunk_count % 5 == 0 )) && [[ -t 2 ]] && printf '.' >&2
done < <(curl -sN "$url" -d "$request_body")
```

**Severity:** Medium -- UX improvement for local model users
**New in v19**

---

### P4 -- Low Priority (3 items)

#### R10: Parallel Tool Help Fetch Has No Concurrency Limit
**Files:** `src/lib/llm_prompt.sh:497-502`
**Issue:** Tool help is fetched in parallel via background jobs:
```bash
for tool in "${_help_tools[@]}"; do
    _nl_get_tool_help "$tool" "$tools_dir" > /dev/null &
done
wait
```
The number of jobs equals the number of enriched tools, which defaults to 7 but is configurable via `NL_MAX_ENRICHED_TOOLS`. Setting it to 50 would fork 50 background processes simultaneously.

**Fix:** Add a simple concurrency limit:
```bash
local _max_parallel=8
local _running=0
for tool in "${_help_tools[@]}"; do
    _nl_get_tool_help "$tool" "$tools_dir" > /dev/null &
    _running=$((_running + 1))
    (( _running >= _max_parallel )) && { wait -n 2>/dev/null || wait; _running=$((_running - 1)); }
done
wait
```

**Severity:** Low -- only affects users who override `NL_MAX_ENRICHED_TOOLS` to high values
**New in v19**

---

#### R11: Gemini Thinking Budget Defaults to Zero
**Files:** `src/lib/llm_config.sh:9`
**Issue:** `_NL_GEMINI_THINKING_BUDGET="${NL_GEMINI_THINKING:-0}"` disables Gemini 2.5 Pro's extended thinking by default. For complex multi-step queries (schema comparison, performance diagnosis), extended thinking could significantly improve accuracy.

However, thinking tokens consume the output budget (`_NL_GEMINI_MAX_TOKENS=8192`), so enabling it risks truncating the actual response. The v12 review noted this as an intentional trade-off.

**Fix:** Consider increasing the default to a modest value (e.g., `2048`) and increasing `_NL_GEMINI_MAX_TOKENS` to `12288` to compensate:
```bash
readonly _NL_GEMINI_THINKING_BUDGET="${NL_GEMINI_THINKING:-2048}"
readonly _NL_GEMINI_MAX_TOKENS="${NL_GEMINI_MAX_TOKENS:-12288}"
```

**Severity:** Low -- trade-off already documented; improvement is optional
**New in v19**

---

#### R12: No CI Regression Testing for Semantic Accuracy
**Files:** `.github/workflows/tests.yml`
**Issue:** Unit tests validate the mechanics of semantic matching (cosine similarity, provider detection, top-N filtering) but do not run the full accuracy suite (`test_embeddings.sh --mode all`). If a code change breaks the matching pipeline (e.g., regex keyword typo, embedding format change), it won't be caught until manual testing.

**Fix:** Add a CI step that runs the accuracy suite in regex-only mode (no API keys needed):
```yaml
- name: NL Semantic Matching Accuracy (regex-only)
  run: |
    bash tools/utils/test_embeddings.sh --mode all --regex-only
```
This validates the regex keywords and combined matching logic without requiring embedding API keys.

**Severity:** Low -- manual testing currently catches regressions; CI would prevent drift
**New in v19**

---

## v18 Status Update

| v18 Item | Status |
|----------|--------|
| R1: Few-shot examples for cr_ddl_diff/cr_catalog_diff | **FIXED** (verified: lines 21-22 now correct) |
| R2: cmd_count numeric validation | **FIXED** (verified: line 1402 uses `if type == "array"` + regex guard) |
| R3: Feedback read timeout | **FIXED** (verified: line 1034 uses `-t 30`) |
| R4: Reflexion unsanitized command names | **FIXED** (verified: line 1490-1491 uses `_sanitize_for_prompt`) |
| R5: Hardcoded schema fetch timeout | **Open** (still uses `timeout 5` at lines 1001, 1019, 1074) |
| R6: Anthropic streaming partial tool args | **Open** (still adds empty args to response) |

## v17 Remaining Open Items

| v17 Item | Status | Notes |
|----------|--------|-------|
| R8: Agent loop e2e tests | Open | Test infrastructure investment |
| R9: Session init tests | Open | Test infrastructure investment |
| R10: History truncation tests | Open | Test infrastructure investment |
| R17: Provider-specific error test fixtures | Open | Would require mock API responses |

---

## Architecture Strengths (Confirmed Across 13 Reviews)

1. **Unified provider interface** -- 5 providers (Anthropic, Vertex, OpenAI, Gemini, Ollama) behind a single `_call_llm_api()` dispatcher with automatic fallback
2. **Defense-in-depth security** -- 6-layer command validation (whitelist, normalize, metachar, SQL mutation, tool existence, RBAC) with array-based execution
3. **Hybrid tool matching** -- Regex intent gate + semantic cosine similarity = 99-100% accuracy across 135 test queries
4. **Schema-aware prompts** -- Cluster topology + database schema cached per session, injected into system prompt
5. **Agent loop with reflexion** -- Multi-step problem solving (max 3 iterations) with error-driven self-correction
6. **Persistent learning** -- JSONL failure/success databases with cross-session injection, dedup, pruning
7. **Cost tracking** -- Per-query and session totals with flock-serialized accumulator and budget alerts
8. **PII masking** -- Regex-based redaction of emails, SSN, credit cards, AWS keys before sending to LLM
9. **Doc RAG** -- CockroachDB documentation chunks retrieved via embedding similarity (infrastructure ready, embeddings available)
10. **Comprehensive metrics** -- CSV logging with 18+ fields per query, debug logging at 3 verbosity levels

---

## Recommended Fix Order

**Immediate (1 item, ~30 min):**
1. R1: Add system prompt to training data pipeline

**This week (3 items, ~1 hour):**
2. R4: Validate NL_BUDGET numeric input
3. R2: Add negative/multi-turn training examples
4. R3: Add system prompt size guard for Ollama

**Next sprint (5 items):**
5. R5: Validate preferences file syntax before sourcing
6. R6: Optimize Doc RAG query magnitude computation
7. R8: Extend training data sanitization
8. R7: Rename inverted check functions or add call-site comments
9. R9: Implement Ollama streaming

**Backlog (3 items):**
10. R10: Add concurrency limit to parallel tool help fetch
11. R11: Consider enabling Gemini thinking budget
12. R12: Add regex accuracy regression to CI

---

## Cumulative Review History

| Version | Date | Items | Fixed | Key Theme |
|---------|------|-------|-------|-----------|
| v6 | 2026-03-16 | 12 | 12 | Shell metachar blocking, regex keyword gaps |
| v7 | 2026-03-16 | 8 | 8 | Semantic matching accuracy tuning |
| v9 | 2026-03-18 | 10 | 10 | Provider retry logic, temp file security |
| v10 | 2026-03-18 | 9 | 9 | RBAC, input validation, history management |
| v11 | 2026-03-18 | 8 | 8 | GPT-5/o-series support, SQL file flags |
| v12 | 2026-03-19 | 15 | 13 | Gemini key security, doc RAG, metachar refinement |
| v13 | 2026-03-19 | 10 | 10 | Shared request builders, streaming cleanup traps |
| v14 | 2026-03-19 | 7 | 7 | Native tool call handling, Ollama debug |
| v15 | 2026-03-22 | 6 | 6 | Cost calculation, session exit, trace context |
| v16 | 2026-03-23 | 8 | 6 | Prompt caching, compact prompts, flag dedup |
| v17 | 2026-03-24 | 18 | 15 | Div-by-zero, error logic, execution timeout, test gaps |
| v18 | 2026-03-24 | 6 | 4 | Few-shot fix, cmd_count, feedback timeout |
| v19 | 2026-03-25 | 12 | -- | Training pipeline quality, system prompt sizing |
| **Total** | | **129** | **108** | **84% fix rate** |
