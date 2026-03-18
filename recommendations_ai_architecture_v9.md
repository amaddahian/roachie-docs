# AI Architecture Review v9 — 2026-03-18

Review of the roachie NL/LLM subsystem (~6,650 lines across 13 AI modules + 5 provider plugins) following v8 review cycle. Includes new findings from comprehensive architectural analysis, Doc RAG feature review, and reflexion/self-correction assessment.

## Table 1: v9 New Recommendations

| ID | Priority | Category | Description |
|----|----------|----------|-------------|
| R1 | P0 | Reliability | Structured output / constrained decoding — enforce JSON via provider-native mechanisms |
| R2 | P0 | Cost/Performance | Prompt token budget management — system prompt too large, no per-provider limits |
| R3 | P0 | Completeness | ~~Doc RAG missing Ollama embeddings~~ **FIXED** (on feature/doc-rag branch) |
| R4 | P1 | Reliability | Context window overflow protection — no hard limit enforced before API call |
| R5 | P1 | Correctness | ~~Response normalization~~ **FIXED** — native tool-call path now extracts needs_followup |
| R6 | P1 | Performance | Deduplicate embedding calls — `_embed_query()` called twice per query |
| R7 | P1 | UX | Conversation memory compression — FIFO eviction loses connection context |
| R8 | P2 | Performance | Parallel tool execution — independent commands run sequentially |
| R9 | P2 | Testing | Structured evaluation harness — no end-to-end query-to-command accuracy testing |
| R10 | P2 | Architecture | Tool schema versioning — `tools/schemas/tools.json` not version-aware |
| R11 | P2 | Observability | Per-query trace ID — no correlation across enrichment, API, execution phases |
| R12 | P3 | Maintainability | Prompt template externalization — 1,807-line module mixes bash and prompt text |

**Total: 12 new recommendations** (P0: 3, P1: 4, P2: 4, P3: 1)

---

## Table 2: v8 Carry-Forward (23 open items)

| ID | Priority | Status | Description |
|----|----------|--------|-------------|
| v8-R1 | P1 | **FIXED** | NL pipeline lacks tool version awareness |
| v8-R2 | P0 | **FIXED** | Batch metrics prefix uses predictable PID-only naming |
| v8-R3 | P1 | **FIXED** | Output masking only covers first 500 chars before follow-up context |
| v8-R4 | P1 | **FIXED** | Batch follow-up truncation hardcoded to 2000, not using constant |
| v8-R5 | P1 | Open | Context trimming doesn't re-estimate tokens after prepending summary |
| v8-R6 | P1 | Open | Ollama silently ignores tool calling parameter |
| v8-R7 | P1 | Open | Ollama missing error detection on empty/malformed responses |
| v8-R8 | P1 | Open | Vertex gcloud token not validated before API call |
| v8-R9 | P1 | **FIXED** | 5 constants in `llm_config.sh` not overridable via env vars |
| v8-R10 | P2 | Open | Retry backoff missing jitter (thundering herd risk) |
| v8-R11 | P2 | Open | Ollama token counts never captured to metrics |
| v8-R12 | P2 | Open | Duplicate request JSON building in streaming vs non-streaming |
| v8-R13 | P2 | Open | OpenAI tool call merging is O(n^2) via repeated jq |
| v8-R14 | P2 | **FIXED** | Ollama embedding check hardcodes `localhost:11434` instead of `_NL_OLLAMA_HOST` |
| v8-R15 | P2 | Open | `_NL_OUTPUT_TRUNCATE_CHARS=2000` too small for agent follow-up |
| v8-R16 | P2 | Open | 6 functions with zero unit test coverage |
| v8-R17 | P2 | Open | `test_llm_extract.sh` has always-pass logic for fenced JSON |
| v8-R18 | P2 | Open | `test_llm_metrics.sh` silently skips jq-dependent tests without SKIP status |
| v8-R19 | P2 | **FIXED** | Feedback CSV user comments not escaped via `_csv_escape()` |
| v8-R20 | P2 | Open | Duplicate error-handling pattern in `_call_llm_api_once` (4x rate-limit blocks) |
| v8-R21 | P3 | Open | Streaming progress dots not cleared on early error (< 5 chunks) |
| v8-R22 | P3 | Open | "no reasoning" literal in message history when empty |
| v8-R23 | P3 | Open | No debug log when semantic matching falls back to regex-only |
| v8-R24 | P3 | Open | `ROACHIE_BATCH` global flag not documented in CLAUDE.md |

---

## Table 3: Fixes Applied During v9 Review

| Item | Description | Files Modified |
|------|-------------|----------------|
| Ceph `--top` hallucination | Expanded ceph storage guidance in system prompt; tightened generic flag guidance to require checking `--help` docs | `llm_prompt.sh:1252, 1321-1325` |
| Failure DB error output | Store actual command error output (`_NL_LAST_CMD_OUTPUT`) in persistent failure DB with ANSI stripping and sanitization | `llm_metrics.sh`, `llm_assistant.sh`, `test_llm_metrics.sh` (+2 tests) |
| v8-R9: Config env overrides | All 10 constants in `llm_config.sh` now overridable via env vars | `llm_config.sh` |
| v8-R2: Batch metrics collision | Metrics prefix includes PID + timestamp + RANDOM (was PID-only) | `bin/roachie-batch` |
| v8-R3: Output masking gap | PII masking now covers full `_NL_OUTPUT_TRUNCATE_CHARS` (was hardcoded 500) | `llm_assistant.sh` |
| v8-R4: Batch truncation | Follow-up truncation uses `_NL_OUTPUT_TRUNCATE_CHARS` constant (was hardcoded 2000) | `bin/roachie-batch` |
| v8-R14: Ollama host | Embedding detection uses `_NL_OLLAMA_HOST` (was hardcoded `localhost:11434`) | `llm_prompt.sh` |
| v8-R19: CSV escaping | Verified already fixed — both JSONL (jq --arg) and CSV (_csv_escape) paths escape comments | `llm_metrics.sh` |
| v9-R3: Ollama doc embeddings | Generated 14 chunks x nomic-embed-text 768d; added startup warning for missing embeddings | `tools/embeddings/docs/`, `llm_assistant.sh` |
| v9-R5: Tool-call followup | Native tool-call path extracts `needs_followup` from LLM text via pattern matching (all 3 providers) | `llm_providers.sh` |

---

## Table 4: Combined Priority Summary

| Priority | v8 Open | v8 Fixed | v9 Open | v9 Fixed | Total Open |
|----------|---------|----------|---------|----------|------------|
| **P0** | 0 | 1 | 2 | 1 | **2** |
| **P1** | 4 | 3 | 3 | 1 | **7** |
| **P2** | 7 | 3 | 4 | 0 | **11** |
| **P3** | 5 | 0 | 1 | 0 | **6** |
| **Total** | **16** | **7** | **10** | **2** | **26** |

---

## Detailed Findings

### R1 — Structured Output / Constrained Decoding (P0)

**Problem:** The system relies on LLMs returning valid JSON via prompt instructions (`llm_prompt.sh:1122-1171`). When models return malformed JSON (especially Ollama/Gemini), the pipeline fails silently or produces garbled output. The Gemini provider already has a fallback for plain-text responses (`providers/gemini.sh:319`), indicating this is a recurring issue.

**Files:** `providers/anthropic.sh`, `providers/openai.sh`, `providers/gemini.sh`, `providers/ollama.sh`

**Fix:**
- **Anthropic:** Use `response_format` parameter or tool-use-only mode (partially implemented via native tool calling)
- **OpenAI:** Add `response_format: { type: "json_object" }` to API calls
- **Gemini:** Use `responseMimeType: "application/json"` with `responseSchema`
- **Ollama:** Use `format: "json"` parameter in `/api/generate`
- Add a JSON repair/extraction fallback — extract JSON from markdown code fences, fix trailing commas

---

### R2 — Prompt Token Budget Management (P0)

**Problem:** The system prompt is massive. `_nl_build_system_prompt()` generates a static prompt (`llm_prompt.sh:700-1085`) listing all 77 tools with flags/examples, then `_enrich_system_prompt_with_tool_help()` appends full `--help` output for up to 7 matched tools, plus doc RAG chunks, plus schema context. The only budget awareness is a rough char-to-token warning at `_NL_CONTEXT_WARN_TOKENS=100000` (`llm_config.sh:20`).

**Impact:** Ollama models with 4096 context are likely truncating. Cloud providers charge per token — bloated system prompt on every query accumulates cost.

**Files:** `llm_prompt.sh:700-1085` (system prompt), `llm_prompt.sh:260-488` (enrichment)

**Fix:**
- Measure actual system prompt size per provider and enforce hard limits
- For Ollama (4096 ctx): cap total prompt to ~3000 tokens
- Remove full 77-tool catalog from system prompt — tool matching already selects 5-7 relevant tools. Catalog should list tool *names* only (one line each)
- Move hallucination warnings and parameter rules into enrichment (only when relevant), not base prompt
- Track actual system prompt token counts via API `usage` response

---

### R3 — Doc RAG Missing Ollama Embeddings (P0)

**Problem:** Doc RAG (`llm_prompt.sh:195-254`) supports three embedding providers, but `tools/embeddings/docs/` only contains `openai_text-embedding-3-small.json` and `gemini_gemini-embedding-001.json`. No `ollama_nomic-embed-text.json` for docs. Users running Ollama-only (no API keys) silently get zero doc RAG results.

**Files:** `tools/embeddings/docs/` (missing file), `llm_prompt.sh:195-212` (provider resolution)

**Fix:**
- Generate Ollama doc embeddings: extend `tools/utils/generate_embeddings.sh` to support `--type docs`
- Add startup diagnostic warning when doc RAG is enabled but no compatible embedding file exists

---

### R4 — Context Window Overflow Protection (P1)

**Problem:** The agent loop (`llm_assistant.sh:1280-1424`) appends command output (truncated to 2000 chars) plus full system prompt on every iteration. Over 3 iterations, message history grows with no mechanism to ensure it fits within the model's context window. The token estimate warning doesn't prevent the API call.

**Files:** `llm_assistant.sh:1280-1424`, `llm_config.sh:15-16`

**Fix:**
- Before each API call, estimate total tokens and compare against model context limit
- If over: truncate oldest turns first, then command output
- For Ollama (4096 ctx), reduce `_NL_MAX_AGENT_ITERATIONS` to 2 and `_NL_OUTPUT_TRUNCATE_CHARS` to 1000
- Add provider-aware context limits:
```bash
_llm_context_limit() {
  case "$1" in
    anthropic|vertex-claude) echo 200000 ;;
    openai) echo 128000 ;;
    gemini) echo 1000000 ;;
    ollama-*) echo 4096 ;;
  esac
}
```

---

### R5 — Provider-Specific Response Normalization (P1)

**Problem:** Response extraction has two code paths:
1. **Text-mode:** Each provider's `_extract_*_response()` parses into `{reasoning, commands, user_message, needs_followup}`
2. **Native tool-calling:** `_convert_tool_call_to_json()` in `llm_providers.sh:526-592` handles conversion separately

The tool-calling path hard-codes `needs_followup: false` and doesn't extract `user_message`. With native tool calling enabled, the agent loop's reflexion logic won't trigger because `needs_followup` is always false.

**Files:** `llm_providers.sh:526-592`, each provider's `_extract_*_response()`

**Fix:**
- Unify into a single `_normalize_llm_response()` function handling both text and tool-call responses
- Ensure `needs_followup` and `user_message` are properly extracted in all paths
- Add response validation checking all required fields before passing to command executor

---

### R6 — Deduplicate Embedding Calls (P1)

**Problem:** Every query triggers a real-time embedding API call via `_embed_query()` for both tool matching and doc RAG — potentially 2 calls per query. For Ollama this is fast (~50ms local), but for OpenAI/Gemini adds 200-500ms latency per query.

**Files:** `llm_prompt.sh:130-185` (`_semantic_match_tools`), `llm_prompt.sh:195-254` (`_doc_rag_retrieve`), `llm_prompt.sh:350-488` (orchestrator)

**Fix:** Compute the query embedding once in `_enrich_system_prompt_with_tool_help()` and pass it to both `_semantic_match_tools()` and `_doc_rag_retrieve()` as a parameter.

---

### R7 — Conversation Memory Compression (P1)

**Problem:** `_NL_MAX_HISTORY_TURNS=10` limits conversation history with naive FIFO eviction. Early context (e.g., "I'm working on the movr database on cluster CA") is lost, causing the LLM to forget connection parameters and re-ask.

**Files:** `llm_assistant.sh` (history management), `llm_config.sh:19`

**Fix options:**
- Sliding window with "sticky" context: extract and preserve connection parameters (host, port, tenant, database) from initial turns as a persistent preamble
- Summary-based: when evicting old turns, summarize into a single "conversation so far" message preserving key facts
- Simplest: extract structured session context into a separate variable that persists across history truncation

---

### R8 — Parallel Tool Execution (P2)

**Problem:** When the LLM returns multiple commands (e.g., "check health on both clusters"), they execute sequentially in `_nl_execute_commands()`. Independent commands could run in parallel.

**Files:** `llm_assistant.sh` (`_nl_execute_commands`)

**Fix:**
- Add optional `parallel: true` field to response format
- When true with multiple commands, execute concurrently with `&` and `wait`, merging outputs
- Default to sequential for safety

---

### R9 — Structured Evaluation Harness (P2)

**Problem:** Tool matching accuracy is well-tested (135 queries, 99-100%), but no end-to-end evaluation of the full NL pipeline exists — from query to correct command generation. Unit tests cover individual functions but not whether the LLM produces correct commands.

**Files:** New: `tests/integration/test_nl_e2e.sh`

**Fix:**
- Create evaluation dataset of 50-100 query-to-expected_command pairs
- Build harness that runs queries through full pipeline, compares extracted commands against expected (tool name + key flags), reports accuracy per provider
- Store results in `test-results/` for tracking over time

---

### R10 — Tool Schema Versioning (P2)

**Problem:** `tools/schemas/tools.json` contains MCP tool schemas for native tool calling, but there's only one file — doesn't account for version differences between v25.2 and v25.4 tools.

**Files:** `llm_prompt.sh:490-520` (`_nl_load_tool_schemas`)

**Fix:**
- Use existing `_nl_resolve_resource()` pattern for `tools/schemas/25.2/tools.json` overrides
- Generate schemas automatically from `--help` output to prevent drift

---

### R11 — Observability / Per-Query Trace ID (P2)

**Problem:** Debug logging (`_nl_debug()` at `NL_DEBUG=1`) writes timestamped entries, but no request-level trace ID correlates enrichment, API call, extraction, validation, and execution for a single query. CSV metrics capture aggregates but not intermediate decisions.

**Files:** `llm_metrics.sh:180-196`, `llm_assistant.sh`

**Fix:**
- Generate per-query trace ID at query start
- Pass through all phases, include in debug log entries
- Add `NL_DEBUG=2` level logging full system prompt + LLM response (redacted)

---

### R12 — Prompt Template Externalization (P3)

**Problem:** System prompt is built by 7+ functions across `llm_prompt.sh` (1,807 lines), mixing bash heredocs, concatenation, and conditional sections. Changing prompt wording requires modifying bash code.

**Files:** `llm_prompt.sh:700-1085` (full prompt), `llm_prompt.sh:1500-1600` (compact prompt)

**Fix:**
- Extract templates to `tools/utils/prompts/system.txt`, `compact.txt`
- Use simple variable substitution at load time
- Separates prompt engineering from code changes

---

## Table 5: Architecture Metrics

| Metric | Value |
|--------|-------|
| Total AI module lines | 6,646 |
| Provider plugins | 5 (Anthropic, OpenAI, Gemini, Vertex, Ollama) |
| LLM models supported | 15+ |
| Tool matching accuracy | 99-100% (combined regex+semantic) |
| Pre-computed embeddings | 3 providers x tools + 2 providers x doc chunks |
| Security layers | 5 (whitelist, metachar, SQL mutation, RBAC, prompt sanitization) |
| Agent loop max iterations | 3 |
| Conversation history limit | 10 turns |
| MCP tools exposed | 39 (via toolbox) |
| Native tool calling | Anthropic, OpenAI, Gemini (schema conversion) |
| Unit test suites (AI) | 13 suites, ~520+ assertions |
| Full test suite | 29 suites, 100% pass rate |

## Table 6: Module Coupling Analysis

| Module | Lines | Depends On | Depended By | Risk |
|--------|-------|-----------|-------------|------|
| `llm_config.sh` | 47 | none | all llm_* | Low |
| `llm_metrics.sh` | 692 | llm_config | llm_providers, llm_assistant | Medium |
| `llm_exec.sh` | 242 | llm_config | llm_assistant, roachie-batch | Low |
| `llm_prompt.sh` | 1,807 | llm_config, llm_metrics | llm_assistant | **High** |
| `llm_providers.sh` | 848 | llm_config, llm_metrics, 5 providers | llm_assistant | Medium |
| `llm_assistant.sh` | 1,585 | all llm_* modules | bin/roachman | **High** |
| `llm_voice.sh` | 247 | none | llm_assistant | Low |

`llm_prompt.sh` and `llm_assistant.sh` together account for 51% of all AI code. `llm_prompt.sh` mixes four concerns: version detection, semantic matching, doc RAG retrieval, and prompt construction. Consider splitting into `llm_matching.sh` + `llm_prompt.sh` when the module grows further.

## Table 7: Test Coverage Gaps Mapped to Recommendations

| Gap | Recommendation | Severity |
|-----|---------------|----------|
| No actual embedding computation tests — cosine similarity only mocked | R9 | High |
| No end-to-end query-to-command evaluation | R9 | High |
| Streaming token count parsing not verified against real responses | R5 | Medium |
| `test_persistent_learning.sh` manual-only, not in automated suite | R11 | Medium |
| No concurrent session isolation tests | R4 | Medium |
| No version-specific tool availability test (v25.2 vs v25.4) | R10 | Low |

## Suggested Implementation Order

1. **v9-R1** (Structured output) — eliminates #1 source of JSON parse failures
2. **v8-R2** (Batch metrics collision) — only P0 from v8, 15-minute fix
3. **v9-R2** (Token budget) — biggest cost reduction opportunity
4. **v8-R3** (Output masking) — security gap, PII beyond 500 chars unmasked
5. **v9-R6** (Deduplicate embeddings) — easy win, saves 200-500ms/query
6. **v9-R3** (Ollama doc embeddings) — completes Doc RAG for offline users
7. **v9-R5** (Response normalization) — fixes reflexion in native tool-call mode
