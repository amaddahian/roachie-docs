# Llama Version Selection and Storage Requirements for Roachie Fine-Tuning

## Why Llama 3.1 8B?

Llama 3.1 8B Instruct was chosen as the base model for Roachie's local inference and LoRA fine-tuning. As of March 2026, it remains the best option at the ~8B parameter size for pure text instruction tasks. Here's how it compares to later releases:

| Version | Release | Available Sizes | Text-Instruct 8B? | Notes |
|---------|---------|----------------|-------------------|-------|
| **Llama 3.1** | Jul 2024 | 8B, 70B, 405B | **Yes** | Last release with an 8B text-instruct model. Used for all roachie fine-tuning. |
| Llama 3.2 | Sep 2024 | 1B, 3B (text), 11B, 90B (vision) | **No** | Text lineup stopped at 3B. The 11B/90B variants are multimodal (vision+text) — unnecessarily large and not optimized for pure text CLI tasks. The 3B is used in roachie as `roachie-3b` but accuracy is lower (~75%). |
| Llama 3.3 | Dec 2024 | 70B only | **No** | No small model variant. 70B requires ~40 GB RAM and is 5-10x slower for inference. Viable on 64+ GB machines but impractical for standard laptop use. |
| Llama 4 | Apr 2025 | Scout (17B MoE), Maverick (17B MoE) | **No** | Mixture-of-Experts architecture. Despite having ~17B active parameters per token, the total model size is much larger (Scout: 109B total, Maverick: larger still). Memory and disk footprint far exceeds what an 8B dense model requires. |

### Key Decision Factors

1. **Parameter count vs. hardware fit**: 8B parameters in 4-bit quantization = 4.2 GB on disk, ~6.9 GB peak memory during LoRA training. This fits comfortably on any Apple Silicon Mac with 16+ GB unified memory. Moving to 70B or MoE architectures requires 40-100+ GB.

2. **Text-only instruction tuning**: Roachie's task is pure text — natural language query → JSON with tool name and flags. Multimodal capabilities (vision in Llama 3.2 11B/90B) add model size without benefiting the task.

3. **MLX ecosystem support**: `mlx-community/Meta-Llama-3.1-8B-Instruct-4bit` is well-tested in the MLX LoRA pipeline. Newer architectures (MoE) may have limited MLX support for fine-tuning.

4. **Ollama compatibility**: Llama 3.1 uses the `llama3-instruct` chat template, which Ollama auto-detects and handles correctly. Newer architectures may require manual template configuration.

### Roachie's Model Lineup

| Ollama Model | Base | Parameters | Use Case |
|-------------|------|-----------|----------|
| `roachie` | Llama 3 | 8B | Legacy, original model |
| `roachie-8b` | Llama 3.1 | 8B | Standard local inference |
| `roachie-3b` | Llama 3.2 | 3B | Low-resource environments |
| `roachie-8b-ft` | Llama 3.1 + LoRA | 8B | Fine-tuned, highest local accuracy (90%) |

---

## Storage Requirements

### Model Sizes

| Component | Size | Format | Persistence |
|-----------|------|--------|-------------|
| Base model (HuggingFace cache) | 4.2 GB | MLX 4-bit (U32 packed) | Cached in `~/.cache/huggingface/` |
| LoRA adapters | 16.8 MB | MLX safetensors | Kept in `tools/utils/finetune/adapters/` |
| Fused model | 4.2 GB | MLX 4-bit | **Temporary** — deleted after de-quantization |
| De-quantized model | ~15 GB | bfloat16 safetensors | **Temporary** — deleted after Ollama import |
| Final model in Ollama | 16 GB | FP16 | Stored in Ollama's model registry |

### Non-Fine-Tuned Models in Ollama

| Model | Quantization | Size |
|-------|-------------|------|
| `roachie` (Llama 3) | Q4_K_M | ~4.5 GB |
| `roachie-8b` (Llama 3.1) | Q4_K_M | ~4.5 GB |
| `roachie-3b` (Llama 3.2) | Q4_K_M | ~2.0 GB |

### Disk Space Budget for Fine-Tuning Pipeline

```
Pipeline Step                    Disk Used    Cumulative    Free Needed
─────────────────────────────────────────────────────────────────────
Base model (cached)              4.2 GB       4.2 GB        -
Training (adapters)              16.8 MB      4.2 GB        ~7 GB (peak mem)
Fuse adapters                    +4.2 GB      8.4 GB
De-quantize (delete fused first) +15 GB       19.2 GB       ← PEAK
Import to Ollama                 +16 GB       35.2 GB       ← Exceeds most laptops
  (delete FP16 model first)      -15 GB       20.2 GB

After cleanup:
  Base model (cached)            4.2 GB
  Adapters                       16.8 MB
  Ollama model                   16 GB
  ────────────────────────────────────────
  Total permanent                ~20.2 GB
```

### Recommendations

1. **Minimum free disk space**: 25 GB before starting the pipeline. The conversion steps create large intermediate files.

2. **Delete intermediates sequentially**: The `run_lora.sh` script deletes the fused 4-bit model before de-quantizing, and deletes the FP16 model after Ollama import. This keeps peak usage manageable.

3. **Old Ollama models**: If reimporting a fine-tuned model, delete the previous version first (`ollama rm roachie-8b-ft`) to free 16 GB before the new import.

4. **HuggingFace cache**: The base model is cached in `~/.cache/huggingface/hub/`. It's downloaded once and reused across training runs. Clearing this cache saves 4.2 GB but requires re-download on next run.

### Memory Requirements (Runtime)

| Activity | Peak Memory | Hardware |
|----------|------------|----------|
| LoRA training | 6.87 GB | Apple Silicon unified memory |
| Inference (roachie-8b, Q4) | ~5-6 GB | Any 8+ GB machine |
| Inference (roachie-8b-ft, FP16) | ~17-18 GB | Requires 32 GB machine |
| Inference (roachie-3b, Q4) | ~3 GB | Any 8+ GB machine |

The fine-tuned model (`roachie-8b-ft`) runs in FP16 because Ollama re-imports at full precision. This means it needs roughly 3x the memory of the Q4-quantized base model. On a 32 GB M1 Pro, it fits comfortably but leaves less room for other applications. Future optimization: export the fine-tuned model at Q4_K_M quantization to halve the memory footprint with minimal accuracy loss.
