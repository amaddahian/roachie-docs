# Model Comparison: Gemma 3, Mistral Nemo, and Llama 3.1 for CLI Tool Selection

## Context

Roachie's local inference uses Llama 3.1 8B, fine-tuned with LoRA to 90% accuracy on a 73-prompt test suite. This document compares it against two other open-source models in the same size class: Google's Gemma 3 12B and Mistral's Nemo 12B.

---

## Gemma 3 (Google, Feb 2025)

| Size | Parameters | Context | Quantized Size (Q4) | Fit on 32GB Mac? |
|------|-----------|---------|---------------------|-----------------|
| 1B | 1B | 32K | ~0.7 GB | Yes |
| 4B | 4B | 128K | ~2.5 GB | Yes |
| **12B** | **12B** | **128K** | **~7 GB** | **Yes** |
| 27B | 27B | 128K | ~16 GB | Tight |

The **12B** is the interesting one. It's 50% more parameters than Llama 3.1 8B but still fits comfortably in 32 GB. Gemma 3 generally benchmarks above Llama 3.1 at equivalent sizes on instruction following tasks. The 128K native context is also a significant advantage — no risk of exceeding context limits with large prompt templates.

---

## Mistral Models

| Model | Parameters | Context | Quantized Size (Q4) | Notes |
|-------|-----------|---------|---------------------|-------|
| Mistral 7B | 7B | 32K | ~4 GB | Older (Sep 2023), good but superseded |
| **Mistral Nemo** | **12B** | **128K** | **~7 GB** | Joint with NVIDIA (Jul 2024), strong instruction following |
| Mixtral 8x7B | 46.7B (12.9B active) | 32K | ~26 GB | MoE — won't fit on 32GB easily |
| Mistral Small 3 | 24B | 128K | ~14 GB | Jan 2025, very capable but 3x the size |

**Mistral Nemo 12B** is the closest competitor to Llama 3.1 8B — similar size class, 128K context, strong at structured output (JSON generation, which is exactly what Roachie needs).

---

## Head-to-Head for Roachie's Task

| Factor | Llama 3.1 8B | Gemma 3 12B | Mistral Nemo 12B |
|--------|-------------|-------------|-----------------|
| Parameters | 8B | 12B | 12B |
| Context window | 128K (native) | 128K | 128K |
| Q4 model size | ~4.5 GB | ~8.1 GB | ~7.1 GB |
| Peak RAM (inference) | ~6 GB | ~9 GB | ~9 GB |
| MLX LoRA support | Mature | Available | Available |
| Ollama support | Mature | Yes | Yes |
| Instruction following | Good | Better (newer arch) | Better (trained for structured output) |
| JSON output reliability | Decent, needs fine-tuning | Likely better out of the box | Strong — designed for tool use |
| Current roachie accuracy | 90% (fine-tuned) | **Testing in progress** | **87.7% (no fine-tuning)** |

---

## The Opportunity

Either Gemma 3 12B or Mistral Nemo 12B could potentially **match or exceed the fine-tuned Llama's 90% without any fine-tuning** — their larger parameter count and newer architectures handle structured output better. With LoRA fine-tuning on top, 93-95% might be achievable, significantly closing the gap to Gemini's 95-100%.

The trade-off is ~3 GB more RAM and significantly slower inference (~79s avg vs ~3-5s for Llama 8B). On a 32 GB Mac, both fit comfortably in memory.

---

## Batch Test Results

73-prompt batch test suite with live command execution against CockroachDB v25.2.12.

| Model | Pass Rate | Prompts Passed | Notes |
|-------|-----------|---------------|-------|
| Llama 3.1 8B (base, no fine-tune) | 83% | 60-61/73 | Unified prompt, no LoRA |
| **Llama 3.1 8B (LoRA fine-tuned)** | **90%** | **66/73** | **Current best local model** |
| Gemma 3 12B (base, no fine-tune) | *retest needed* | 6/33* | Initial run failed — 27/33 API timeouts (memory contention). Solo rerun in progress. |
| **Mistral Nemo 12B (base, no fine-tune)** | **87.7%** | **64/73** | No fine-tuning. 8 failed, 1 error, 1 null. |
| Gemini 2.0 Flash (cloud reference) | 95-100% | 70-73/73 | Cloud model baseline |

*\*Gemma 3's initial run had both 12B models loaded simultaneously (~16 GB), causing memory contention and API timeouts on a 32 GB Mac. Rerunning solo.*

---

## Mistral Nemo 12B — Detailed Results

### Performance Summary

| Metric | Value |
|--------|-------|
| Success rate | 64/73 (87.7%) |
| Failed (wrong flags) | 8 |
| Error (API timeout) | 1 |
| Null response | 1 |
| Avg prompt tokens | 13,613 |
| Avg completion tokens | 132 |
| Avg latency per query | 78.8 seconds |

### Failure Analysis

All 8 failures selected the **correct tool** but generated **wrong flags**. Tool selection accuracy was ~99% (72/73, excluding the 1 timeout).

| # | Prompt | Tool Selected | Failure Type |
|---|--------|--------------|-------------|
| 1 | backup database movr to S3 bucket | cr_backup (correct) | Hallucinated `--location` flag with placeholder S3 URL |
| 2 | list views in system tenant | cr_find_views (correct) | Used `--pattern %` and `--t` (invalid double-dash + short flag) |
| 3 | list all users in test_lab | cr_list_users (correct) | Used full path `tools/tools.25.4/cr_list_users` — blocked by whitelist |
| 4 | DDL for all users | cr_ddl_user (correct) | Missing user positional argument |
| 5 | DDL for products table | cr_ddl_table (correct) | Used `-t products` overwriting the tenant `-t system` flag |
| 6 | show grants for test_engineer | cr_my_grants (correct) | Used `-U` (capital) instead of `-u` (lowercase) |
| 7 | DDL for rides table | cr_ddl_table (correct) | Used `-T rides` instead of positional `rides` argument |
| 8 | list columns for rides table | cr_columns (correct) | Used `--filter-db` and `-T` (non-existent flags) |

### Key Observations

1. **Tool selection is excellent without fine-tuning** — Mistral Nemo correctly identified the right tool in 72/73 cases. The 12B parameter count and instruction-tuning give it a strong understanding of tool descriptions.

2. **Flag generation is the weak point** — same pattern as Llama 3.1 8B before fine-tuning. The model sees the `--help` output in the prompt but still hallucinates flag syntax, confuses short and long flags, or misses positional arguments.

3. **Latency is a concern** — 78.8s average per query is ~15-25x slower than Llama 3.1 8B (~3-5s). The 12B model on Q4 quantization is noticeably heavier on inference. This may be acceptable for batch operations but is too slow for interactive use.

4. **Flag collision** — Prompt #5 is a notable failure: the model used `-t products` (meaning "table") which collided with `-t system` (meaning "tenant"). This is a fundamental ambiguity in the flag design that even fine-tuning may not fully resolve.

---

## Recommendations

1. **Mistral Nemo 12B at 87.7% without fine-tuning nearly matches the fine-tuned Llama 3.1 8B at 90%.** With LoRA fine-tuning on the same 82 Gemini examples, 93-95% is plausible. The extra 4B parameters should allow better generalization from limited training data.

2. **Latency is the primary trade-off.** At ~79s per query, Mistral Nemo is impractical for interactive use on current hardware. For batch operations or environments where accuracy matters more than speed, it's viable.

3. **LoRA fine-tuning would target exactly the right weakness** — flag generation. Since tool selection is already near-perfect, even a small training set focused on correct flag syntax could push accuracy past 90%.

4. **The 12B models use ~3 GB more RAM** than the 8B. On 16 GB machines, this could be tight. Llama 3.1 8B remains the better choice for resource-constrained environments.

5. **LoRA fine-tuning feasibility**: Both models have MLX-compatible variants on HuggingFace. The existing `run_lora.sh` pipeline should work with minimal changes (update `BASE_MODEL` variable).

6. **Gemma 3 results pending** — solo rerun in progress. Initial results suggest potential API compatibility issues with the Ollama provider that may need investigation beyond just memory contention.
