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
| Current roachie accuracy | 90% (fine-tuned) | **Testing in progress** | **Testing in progress** |

---

## The Opportunity

Either Gemma 3 12B or Mistral Nemo 12B could potentially **match or exceed the fine-tuned Llama's 90% without any fine-tuning** — their larger parameter count and newer architectures handle structured output better. With LoRA fine-tuning on top, 93-95% might be achievable, significantly closing the gap to Gemini's 95-100%.

The trade-off is ~3 GB more RAM and slightly slower inference. On a 32 GB Mac, both fit comfortably.

---

## Batch Test Results

*Results will be updated once the 73-prompt batch tests complete for both models.*

| Model | Pass Rate | Prompts Passed | Notes |
|-------|-----------|---------------|-------|
| Llama 3.1 8B (base, no fine-tune) | 83% | 60-61/73 | Unified prompt, no LoRA |
| **Llama 3.1 8B (LoRA fine-tuned)** | **90%** | **66/73** | **Current best local model** |
| Gemma 3 12B (base, no fine-tune) | *pending* | */73 | First test run |
| Mistral Nemo 12B (base, no fine-tune) | *pending* | */73 | First test run |
| Gemini 2.0 Flash (cloud reference) | 95-100% | 70-73/73 | Cloud model baseline |

---

## Recommendations

1. **If Gemma 3 or Mistral Nemo scores >85% without fine-tuning**, it's worth LoRA fine-tuning them with the same 82 Gemini examples. The extra 4B parameters should allow better generalization from limited training data.

2. **If either scores >90% without fine-tuning**, it may replace roachie-8b-ft as the default local model — simpler (no training pipeline needed) and potentially more accurate.

3. **The 12B models use ~3 GB more RAM** than the 8B. On 16 GB machines, this could be tight. Llama 3.1 8B remains the better choice for resource-constrained environments.

4. **LoRA fine-tuning feasibility**: Both models have MLX-compatible variants on HuggingFace. The existing `run_lora.sh` pipeline should work with minimal changes (update `BASE_MODEL` variable).
