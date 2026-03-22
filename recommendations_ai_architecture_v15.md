# AI Architecture Review v15 — 2026-03-22

## Scope

Full review of the roachie NL/AI subsystem covering: prompt construction, provider architecture, execution pipeline, semantic matching, embeddings, MCP integration, and the new LoRA fine-tuning pipeline.

## Current State Summary

| Component | Maturity | Key Metric |
|-----------|----------|------------|
| Prompt pipeline | High | 8 templates, ~4,450 tokens base, unified across providers |
| Provider abstraction | High | 5 providers (Anthropic, OpenAI, Gemini, Vertex, Ollama) |
| Security / validation | High | 5 validation layers, 113 security tests |
| Semantic matching | High | 99-100% combined accuracy (135 queries, 3 providers) |
| Ollama accuracy | Medium | 90% with LoRA fine-tune (was 72%, Gemini: 95-100%) |
| MCP Toolbox | Medium | 39/77 tools (51% coverage) |
| LoRA fine-tuning | New | 82 training examples, MLX pipeline, reproducible |
| Native tool calling | Low | Supported but not battle-tested |

---

## Recommendations

### P1 — High Priority

#### R1. Fix SQL Injection via Multi-Statement in Monitor Role
**Risk**: Medium | **Effort**: Low

The RBAC guard for `cockroach sql` checks SQL mutation keywords only at statement start. A monitor-role user could bypass via multi-statement:
```
cockroach sql -e "SELECT 1; GRANT admin TO attacker"
```
The GRANT is in a separate statement and not caught by the keyword-start check.

**Fix**: Split SQL input on `;` boundaries and validate each statement independently. Or restrict monitor role to `--execute` with a single statement.

**File**: `src/lib/llm_assistant.sh` (SQL mutation guard, ~line 261)

---

#### R2. Add Execution Timeout to Interactive REPL
**Risk**: Medium | **Effort**: Low

Batch mode has `BATCH_CMD_TIMEOUT` but the interactive REPL (`llm.sh`) has no timeout on command execution. A hung command blocks indefinitely.

**Fix**: Add configurable `NL_CMD_TIMEOUT` (default: 120s) using `timeout` command wrapper in `_safe_execute_command()`.

**File**: `src/lib/llm_assistant.sh` (~line 295)

---

#### R3. Remove Dead Compact Prompt Code
**Risk**: None (cleanup) | **Effort**: Low

`_nl_build_system_prompt_compact()` (llm_prompt.sh:1175-1229) is no longer called anywhere — the unified prompt path replaced it. Dead code adds maintenance burden and confusion.

**Fix**: Delete `_nl_build_system_prompt_compact()` and all references.

**File**: `src/lib/llm_prompt.sh`

---

#### R4. Streaming Boilerplate Deduplication
**Risk**: None (refactor) | **Effort**: Medium

Three streaming implementations (anthropic.sh, openai.sh, gemini.sh) share ~120-150 lines of identical boilerplate:
- Temp file creation + trap cleanup
- SSE cursor loop (`while IFS= read -r line`)
- Progress dot printing (`chunk_count % 5`)
- Connection drop detection via curl rc file
- Error buffer accumulation

**Fix**: Extract shared streaming infrastructure into `src/lib/providers/_streaming_common.sh`:
```bash
_stream_sse_loop() {
  local url="$1" headers="$2" body="$3" extractor="$4"
  # Shared: temp files, trap, curl, SSE parse loop, progress dots
  # Provider-specific: $extractor callback for content extraction
}
```

**Files**: `src/lib/providers/anthropic.sh`, `openai.sh`, `gemini.sh`

---

### P2 — Medium Priority

#### R5. Consolidate Redundant Prompt Templates
**Risk**: None | **Effort**: Medium

Content overlap between templates:
- `tool_notes.txt` and `few_shot_examples.txt` both document DDL table positional args, cr_get_acl `-o` flag, and cr_ceph_storage subcommands
- `execution_rules.txt` and `parameter_rules.txt` both cover tenant parameter rules
- `hallucination_warnings.txt` overlaps significantly with `tool_notes.txt`

For 8B models, redundancy wastes scarce context tokens. For cloud models, it's harmless but increases cost.

**Fix**: Merge `hallucination_warnings.txt` into `tool_notes.txt` (they serve the same purpose — flag correction). Remove duplicate content from `few_shot_examples.txt` that's already in `tool_notes.txt`. Estimated savings: ~200-300 tokens.

**Files**: `tools/utils/prompts/*.txt`

---

#### R6. Add Streaming Support to Ollama and Vertex Providers
**Risk**: None | **Effort**: Medium

Ollama and Vertex are the only providers without streaming. For Ollama especially, the 8B model generates slowly (~0.5 it/sec), making 15-30s waits with no feedback a poor UX.

Ollama's `/api/chat` supports `"stream": true` with newline-delimited JSON (simpler than SSE). Vertex AI supports streaming via `streamGenerateContent`.

**Fix**: Add `_call_ollama_api_stream()` and `_call_vertex_claude_api_stream()`. Use the shared streaming infrastructure from R4.

**Files**: `src/lib/providers/ollama.sh`, `vertex.sh`

---

#### R7. Expand LoRA Training Data
**Risk**: None | **Effort**: Low

Current fine-tuning uses only 82 training examples (65 train / 8 valid / 9 test). This is sufficient for pattern memorization but insufficient for generalization. The validation loss curve shows oscillation (0.332 → 1.070 → 0.332 → 1.320), indicating the model overfits quickly and hasn't learned stable representations.

**Fix**:
1. Run 200+ diverse prompts through Gemini to expand training data
2. Include paraphrased versions of existing prompts (same intent, different wording)
3. Add negative examples (wrong commands with corrections) as a separate training signal
4. Consider DPO (Direct Preference Optimization) instead of SFT — pairs of (correct, incorrect) responses teach the model what to avoid

**File**: `tools/utils/finetune/prepare_training_data.py`

---

#### R8. Implement Best-of-N Sampling for Ollama
**Risk**: None | **Effort**: Medium

Generate N=3 candidate responses from Ollama, validate each against tool `--help`, and pick the one with highest flag accuracy. This is a cheap way to improve accuracy without model changes.

**Fix**: Add `_best_of_n_sample()` function that:
1. Calls `_call_ollama_api()` N times with slightly varied temperature
2. Extracts command from each response
3. Validates flags against `tool --help` output
4. Returns the response with most valid flags

**Files**: `src/lib/providers/ollama.sh`, `src/lib/llm_assistant.sh`

---

#### R9. Token Budget Monitoring and Alerting
**Risk**: Low | **Effort**: Low

No runtime check warns when the system prompt + enrichment approaches the context limit. Ollama's 16K window can be exceeded with large schema contexts + 7 enriched tools. When exceeded, Ollama silently truncates, leading to degraded accuracy.

**Fix**: After prompt construction, compare `${#system_prompt}` (chars, estimated 4 chars/token for Ollama) against `_llm_context_limit`. Warn if >80% utilized. Log in metrics.

**Files**: `src/lib/llm_prompt.sh`, `bin/roachie-batch`

---

#### R10. Validate Embedding Completeness on Startup
**Risk**: Low | **Effort**: Low

No check ensures all 77 tools appear in the embedding files. If a new tool is added but embeddings aren't regenerated, semantic matching silently misses it.

**Fix**: On NL startup, compare tool count in embedding file vs `ls tools/current/cr_* | wc -l`. Warn if mismatch. Add to CI checks.

**File**: `src/lib/llm_prompt.sh` (embedding loading section)

---

### P3 — Low Priority / Enhancements

#### R11. Add Constrained Decoding for Ollama (GBNF Grammar)
**Effort**: High | **Impact**: +3-5% accuracy

Ollama supports GBNF grammars that constrain output tokens. A grammar that restricts flag names to known valid values per tool would prevent hallucinated flags at the decoder level.

**File**: New file `tools/utils/grammars/roachie.gbnf`, changes to `providers/ollama.sh`

---

#### R12. Implement Tool Calling Validation Loop
**Effort**: High | **Impact**: +5-8% accuracy

After command generation, automatically validate the command against `tool --help`:
1. Extract flags from generated command
2. Parse `--help` output for valid flags
3. If invalid flags found, feed error back to LLM for correction
4. Max 1 correction iteration

This is lighter than best-of-N (R8) and more targeted.

**Files**: `src/lib/llm_assistant.sh`

---

#### R13. Expand MCP Toolbox Coverage
**Effort**: Medium | **Impact**: Broader integration

39/77 tools (51%) are exposed via MCP. The remaining 38 tools require bash orchestration (DDL generation, backup workflows, monitoring loops) and can't be pure SQL.

Consider exposing a "meta-tool" that wraps the bash tools as MCP endpoints, returning structured results.

**File**: `tools/mcp/tools.yaml`

---

#### R14. Implement Provider-Specific Response Parsing Hardening
**Effort**: Low | **Impact**: Reliability

Gemini sometimes returns thinking tokens that consume `maxOutputTokens`, causing truncated responses (caught by `MAX_TOKENS` finishReason check). OpenAI o-series models have similar behavior.

Add structured output mode (`response_format: { type: "json_object" }`) for OpenAI and Gemini to guarantee JSON output without thinking token leakage.

**Files**: `src/lib/providers/openai.sh`, `gemini.sh`

---

#### R15. Version-Specific Tool Descriptions and Embeddings
**Effort**: Medium | **Impact**: Accuracy on multi-version deployments

`tools/embeddings/25.2/` and `tools/embeddings/25.4/` directories exist but are empty. Tool descriptions should be version-aware — some tools have different flags or behaviors across CockroachDB versions.

**Fix**: Generate version-specific embeddings when tools differ. Fall back to shared embeddings for tools that are identical.

**Files**: `tools/utils/generate_embeddings.sh`, `tools/embeddings/`

---

#### R16. Add Automated Regression Testing for NL Accuracy
**Effort**: Medium | **Impact**: Prevents regressions

No CI step validates NL accuracy after code changes. Add a lightweight CI job that:
1. Runs 20 representative prompts in DRY-RUN mode (no cluster needed)
2. Validates JSON structure and tool selection
3. Fails if pass rate drops below 85%

**Files**: `.github/workflows/tests.yml`, new test script

---

#### R17. Persistent Learning Effectiveness Audit
**Effort**: Low | **Impact**: Data quality

The persistent failure/success DB (`NL_FAILURES_DB`, `NL_SUCCESSES_DB`) accumulates examples over time but is never pruned or validated. Stale or incorrect entries could degrade performance.

**Fix**: Add TTL (30 days default), dedup logic, and max entry cap (100). Add a `--clear-learning` flag to roachie-nl for manual reset.

**Files**: `src/lib/llm_metrics.sh`, `bin/roachie-nl`

---

#### R18. Unify Token Counting Across Providers
**Effort**: Low | **Impact**: Observability

Each provider hardcodes different JSON paths for token extraction:
- Anthropic: `.usage.input_tokens` / `.usage.output_tokens`
- OpenAI: `.usage.prompt_tokens` / `.usage.completion_tokens`
- Gemini: `.usageMetadata.promptTokenCount` / `.usageMetadata.candidatesTokenCount`
- Ollama: `.prompt_eval_count` / `.eval_count`

The `_write_token_counts` function handles this but each provider call-site duplicates the path strings.

**Fix**: Define token path constants per provider in `llm_config.sh` and pass provider name to `_write_token_counts` which looks up paths internally.

**File**: `src/lib/llm_providers.sh`, `llm_config.sh`

---

## Architecture Diagram (Current State)

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Input                              │
│              (roachie-nl / roachman menu / roachie-batch)        │
└────────────────────────────┬────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   llm.sh REPL   │
                    │  Session Mgmt   │
                    │  History Mgmt   │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │     llm_prompt.sh           │
              │  ┌────────────────────┐     │
              │  │ System Prompt Build│     │
              │  │ 8 Template Files   │     │
              │  │ Schema Context     │     │
              │  │ Topology Injection │     │
              │  └────────┬───────────┘     │
              │  ┌────────▼───────────┐     │
              │  │ Tool Enrichment    │     │
              │  │ Regex + Semantic   │     │
              │  │ --help injection   │     │
              │  │ Doc RAG (optional) │     │
              │  └────────────────────┘     │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   llm_providers.sh          │
              │   Dispatcher + Retry        │
              │  ┌─────┬─────┬─────┬─────┐  │
              │  │Anthr│OpenAI│Gemin│Ollam│  │
              │  │opic │     │i    │a    │  │
              │  └──┬──┴──┬──┴──┬──┴──┬──┘  │
              │     │     │     │     │      │
              │     └─────┴─────┴─────┘      │
              │         Vertex AI            │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   llm_assistant.sh          │
              │  ┌────────────────────┐     │
              │  │ JSON Response Parse│     │
              │  │ Command Extraction │     │
              │  └────────┬───────────┘     │
              │  ┌────────▼───────────┐     │
              │  │ 5-Layer Validation │     │
              │  │ Whitelist → Meta   │     │
              │  │ → SQL → RBAC →    │     │
              │  │ Tool Existence     │     │
              │  └────────┬───────────┘     │
              │  ┌────────▼───────────┐     │
              │  │ Safe Execution     │     │
              │  │ (array-based, no   │     │
              │  │  eval/shell interp)│     │
              │  └────────┬───────────┘     │
              │  ┌────────▼───────────┐     │
              │  │ Reflexion Loop     │     │
              │  │ (max 3 iterations) │     │
              │  └────────────────────┘     │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │   llm_metrics.sh            │
              │  CSV logging, persistent    │
              │  learning (failures/success) │
              └─────────────────────────────┘
```

---

## Summary Table

| ID | Priority | Description | Effort | Impact |
|----|----------|-------------|--------|--------|
| R1 | P1 | Fix SQL injection via multi-statement in monitor role | Low | Security |
| R2 | P1 | Add execution timeout to interactive REPL | Low | Reliability |
| R3 | P1 | Remove dead compact prompt code | Low | Cleanup |
| R4 | P1 | Extract shared streaming boilerplate | Medium | Maintainability |
| R5 | P2 | Consolidate redundant prompt templates | Medium | Token savings |
| R6 | P2 | Add streaming to Ollama and Vertex | Medium | UX |
| R7 | P2 | Expand LoRA training data (82 → 300+) | Low | Accuracy |
| R8 | P2 | Best-of-N sampling for Ollama | Medium | Accuracy +3-5% |
| R9 | P2 | Token budget monitoring and alerting | Low | Reliability |
| R10 | P2 | Validate embedding completeness on startup | Low | Reliability |
| R11 | P3 | Constrained decoding (GBNF grammar) | High | Accuracy +3-5% |
| R12 | P3 | Tool calling validation loop | High | Accuracy +5-8% |
| R13 | P3 | Expand MCP Toolbox coverage | Medium | Integration |
| R14 | P3 | Provider-specific response parsing hardening | Low | Reliability |
| R15 | P3 | Version-specific embeddings | Medium | Multi-version |
| R16 | P3 | Automated NL regression testing in CI | Medium | Quality gate |
| R17 | P3 | Persistent learning TTL and pruning | Low | Data quality |
| R18 | P3 | Unify token counting across providers | Low | Observability |
