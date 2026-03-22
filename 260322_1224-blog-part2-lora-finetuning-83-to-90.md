# LoRA Fine-Tuning Llama 8B on Apple Silicon: From 83% to 90% Using Cloud Model Outputs as Training Data

*Part 2 of 2 — Training a local model to mimic Gemini's tool selection, the MLX-to-Ollama conversion pipeline, and what 82 training examples can (and can't) teach.*

---

In [Part 1](), Llama 3.1 8B went from ~50% to 83% accuracy on CLI tool selection through prompt engineering and infrastructure fixes — no model changes at all. But 83% means 1 in 6 commands is wrong. For a DBA tool that might run against production databases, that's not good enough.

Gemini was hitting 95-100% on the same prompts. The question: could Llama learn to behave more like Gemini, while still running locally?

The answer is LoRA fine-tuning. And the training data? Gemini's own outputs.

## The Idea: Use Cloud Outputs to Train Local Models

Months of batch test results had accumulated — thousands of records where the same prompts were sent to Gemini and the responses logged. Each record contained:

- The natural language prompt ("list all tables in the movr database on VA tenant")
- The JSON response (tool name, flags, explanation)
- Whether the command executed successfully

This is a ready-made supervised dataset. Gemini already solved the problem. Llama just needed to learn the mapping.

This approach — distilling a large model's behavior into a smaller one — isn't new. But applying it to bash CLI tool selection on commodity hardware (a MacBook Pro) made it practical in a way that GPU cluster training wouldn't be.

## Step 1: Preparing the Training Data

A Python script (`prepare_training_data.py`) handles the data pipeline:

1. **Reads 156 JSONL batch result files** from the test audit directory (3,105 total records)
2. **Filters** to only Gemini responses with `status: "success"` — the model shouldn't learn from wrong answers
3. **Deduplicates** by normalized prompt text (lowercase, whitespace-stripped) — many prompts ran multiple times across test runs
4. **Cleans commands** — strips timing prefixes like `[OK:247ms]`, removes absolute paths (the model should generate relative tool names), drops trailing semicolons
5. **Formats as Chat-ML JSONL** — the format MLX expects for instruction tuning

The deduplication step was critical. 3,105 records collapsed to just **82 unique prompt/command pairs**. Without dedup, the model would memorize the 15 most-common prompts (which appeared 50+ times each) and ignore the tail.

The final split (seed=42 for reproducibility):
- **65 training** examples
- **8 validation** examples
- **9 test** examples

Here's what a training example looks like:

```json
{
  "messages": [
    {
      "role": "user",
      "content": "list all tables in the movr database on VA tenant"
    },
    {
      "role": "assistant",
      "content": "{\"tool\": \"cr_db_tables\", \"flags\": \"-h 10.1.1.5 -p 26257 -t va -d movr\", \"explanation\": \"Lists all tables in the movr database on the VA tenant\"}"
    }
  ]
}
```

82 examples is small. Very small. But the goal was to establish a baseline before investing in data collection.

## Why Llama 3.1 8B?

A reasonable question: why not use a newer Llama release? As of March 2026, Llama 3.1 8B remains the best option at the ~8B size for pure text instruction tasks:

| Version | Sizes | Text 8B? | Why Not |
|---------|-------|----------|---------|
| **Llama 3.1** (Jul 2024) | 8B, 70B, 405B | **Yes** | **Used** — last 8B text-instruct model |
| Llama 3.2 (Sep 2024) | 1B, 3B, 11B, 90B | No | Text models stop at 3B. The 11B/90B are vision-focused. |
| Llama 3.3 (Dec 2024) | 70B only | No | No small variant. 70B needs ~40 GB RAM. |
| Llama 4 (Apr 2025) | Scout, Maverick (MoE) | No | Mixture-of-Experts — much larger total footprint despite ~17B active params. |

Meta hasn't released a Llama 8B text-instruct model since 3.1. The 3.2 text lineup stopped at 3B (used in roachie as `roachie-3b`, but accuracy is lower at ~75%).

### Storage Requirements

The fine-tuning pipeline has significant disk demands due to format conversion:

| Component | Size | Lifetime |
|-----------|------|----------|
| Base model (4-bit, MLX) | 4.2 GB | Cached in HuggingFace |
| LoRA adapters | 16.8 MB | Permanent (reproducible) |
| Fused model (4-bit) | 4.2 GB | Temporary — deleted after de-quantize |
| De-quantized model (bfloat16) | ~15 GB | Temporary — deleted after Ollama import |
| **Final model in Ollama** | **16 GB** | Permanent (FP16) |
| **Peak disk during conversion** | **~25 GB** | Intermediates coexisting briefly |

Budget at least 25 GB of free disk space before starting the pipeline. The `run_lora.sh` script deletes intermediates sequentially to minimize peak usage, but two "disk full" errors were hit on a drive with only 16 GB free.

## Step 2: LoRA Training on Apple Silicon

The training framework is **MLX** — Apple's machine learning framework designed for Apple Silicon. It runs natively on the Metal GPU, uses unified memory (no CPU-GPU transfers), and supports LoRA out of the box via `mlx-lm`.

### Why LoRA, Not Full Fine-Tuning?

Full fine-tuning of an 8B model requires modifying all ~8 billion parameters. Even in 4-bit quantization, that's impractical on a laptop. LoRA (Low-Rank Adaptation) freezes the base weights and trains small adapter matrices — in this case, adding only **16.8 MB** of new parameters on top of the 4.2 GB base model.

### Training Configuration

| Setting | Value | Notes |
|---------|-------|-------|
| Base model | `Meta-Llama-3.1-8B-Instruct-4bit` | From MLX Community hub |
| LoRA rank | 8 | Low rank = fewer parameters, faster training |
| LoRA layers | 16 | How many transformer layers get adapters |
| Learning rate | 1e-5 | Conservative — small dataset, avoid overfitting quickly |
| Batch size | 1 | Limited by dataset size (65 examples) |
| Iterations | 1,000 | ~20 epochs over the training set |
| Peak memory | 6.87 GB | Fits easily in 32GB unified memory |
| Training time | ~33 minutes | On M1 Pro |

### The Training Script

The entire pipeline is wrapped in a single bash script (`run_lora.sh`) with 5 stages:

```bash
# Stage 1: Train
python -m mlx_lm.lora --model $BASE --train --data ./data/ \
  --adapter-path ./adapters/ --iters 1000 --batch-size 1 \
  --learning-rate 1e-5 --num-layers 16

# Stage 2: Fuse adapters into base model
python -m mlx_lm.fuse --model $BASE --adapter-path ./adapters/ \
  --save-path ./fused/

# Stage 3: De-quantize (MLX 4-bit → bfloat16)
# Stage 4: Fix tokenizer chat template
# Stage 5: Import into Ollama
```

Stages 3-5 deserve their own section, because that's where everything went wrong.

### The Loss Curve: A Story of Overfitting

| Iteration | Train Loss | Val Loss | Notes |
|-----------|-----------|----------|-------|
| 0 | — | 3.309 | Baseline (untrained) |
| 100 | 0.450 | 0.386 | Rapid learning |
| 200 | 0.210 | 0.376 | Still improving |
| 300 | 0.085 | 0.873 | Overfitting begins |
| 400 | 0.055 | 0.751 | |
| 500 | 0.038 | 1.070 | |
| **600** | **0.035** | **0.332** | **Best checkpoint** |
| 700 | 0.029 | 0.787 | |
| 800 | 0.030 | 0.918 | |
| 900 | 0.028 | 0.526 | |
| 1000 | 0.029 | 1.320 | Heavy overfitting |

The training loss drops smoothly to near-zero — the model memorized all 65 training examples by iteration 300. But validation loss oscillates wildly: 0.376 → 0.873 → 0.332 → 1.320. This is the signature of a dataset that's too small for stable generalization.

The saving grace: iteration 600 hit a validation loss of 0.332, the lowest point. MLX saves checkpoints every 200 iterations, so the iteration-600 adapter weights were selected.

With 300+ examples, a smoother validation curve and better generalization to unseen prompts would be expected. That's future work.

## Step 3: The MLX-to-Ollama Pipeline (aka "Everything That Can Go Wrong")

Training was the easy part. Getting the trained model into Ollama was a series of roadblocks, each requiring a different workaround.

### Problem 1: Ollama Can't Read MLX 4-Bit Weights

After fusing the LoRA adapters into the base model, the result was a 4.2 GB model in MLX's native format. Ollama's import rejected it:

```
Error: unknown data type: U32
```

MLX stores 4-bit quantized weights as packed U32 integers. Ollama doesn't understand this format — it expects standard floating-point tensors and does its own quantization on import.

**Fix**: De-quantize the model from 4-bit to bfloat16 using MLX's conversion utility:

```python
from mlx_lm.convert import convert
convert(hf_path='./fused/', mlx_path='./fp16/', dequantize=True)
```

This inflated the model from 4.2 GB to ~15 GB. A necessary intermediate step.

### Problem 2: Disk Space

With only 16 GB free on the drive:
- 4.2 GB fused model (4-bit)
- 15 GB de-quantized model (bfloat16)
- Total needed: ~19 GB

```
Error: no space left on device
```

**Fix**: Delete the 4-bit fused model before de-quantizing — it's no longer needed. That freed enough space. Later, Ollama's import also ran out of space (it writes the model to its own registry). The old Ollama model had to be deleted first, then reimported.

Lesson: budget 3x the model size for the conversion pipeline.

### Problem 3: The Missing Chat Template

After importing into Ollama, testing the model produced:

```bash
$ ollama run roachie-8b-ft "list databases on VA tenant"
{  }
```

Empty JSON. The model processed only 11 tokens before stopping. Something was fundamentally wrong with how it was parsing the prompt.

**Root cause**: MLX's de-quantization step strips the `chat_template` field from `tokenizer_config.json`. Without a chat template, Ollama doesn't know how to format the prompt into the `<|begin_of_text|><|start_header_id|>system<|end_header_id|>` structure that Llama 3.1 expects. The raw text gets fed in without any special tokens, and the model has no idea what to do with it.

**Fix**: Copy the `chat_template` (4,614 characters of Jinja2 template) from the original base model's tokenizer config into the de-quantized model's config. After that, Ollama auto-detected the `llama3-instruct` template and everything worked.

This fix was added to `run_lora.sh` so future fine-tuning runs handle it automatically:

```python
import json, glob, os
# Find base model tokenizer in HuggingFace cache
base_paths = glob.glob(os.path.expanduser(
    '~/.cache/huggingface/hub/models--*Meta-Llama-3.1-8B-Instruct*'
    '/snapshots/*/tokenizer_config.json'))
with open(base_paths[0]) as f:
    src = json.load(f)
# Patch the de-quantized model's tokenizer
with open('fp16/tokenizer_config.json') as f:
    tgt = json.load(f)
tgt['chat_template'] = src['chat_template']
with open('fp16/tokenizer_config.json', 'w') as f:
    json.dump(tgt, f, indent=2)
```

### The Complete Conversion Pipeline

```
HuggingFace Hub
    │
    ▼ download
Base Model (4-bit, 4.2 GB)
    │
    ├── mlx_lm.lora (33 min) ──▶ LoRA Adapters (16.8 MB)
    │
    ▼ mlx_lm.fuse
Fused Model (4-bit, 4.2 GB) ──▶ DELETED
    │
    ▼ mlx_lm.convert (dequantize)
FP16 Model (bfloat16, ~15 GB) ──▶ DELETED after import
    │
    ▼ fix tokenizer chat_template
    │
    ▼ ollama create
Ollama: roachie-8b-ft (16 GB)
```

Total time from "start training" to "model serving in Ollama": about 45 minutes, including the troubleshooting.

## Results

The same 73-prompt batch test used for all providers:

```bash
bash bin/roachie-batch \
  --file examples/queries/example-queries.txt \
  --provider ollama-roachie-8b-ft \
  --tools-version 25.4 \
  --no-feedback --no-followup --allow-sql-files
```

**66 out of 73 passed — 90.4% accuracy.**

### The Accuracy Journey

| Stage | Accuracy | What Changed |
|-------|----------|-------------|
| Initial (regex only, compact prompt) | ~50-60% | Baseline — wrong tools, wrong flags |
| + Semantic matching | 72% | Right tools, still wrong flags |
| + Unified prompt (all 7 templates) | 83% | Correct flags for simple tools |
| + Modelfile fixes (num_predict, temp) | ~87% | Fewer truncated responses |
| **+ LoRA fine-tuning** | **90%** | Learned Gemini's flag patterns |

### The 5 Remaining Failures

| Prompt | Issue |
|--------|-------|
| "backup database movr to crdb-test-backups bucket" | Used `cr_backup` instead of `cr_ceph_storage backup` — wrong tool |
| "DDL for products table in test_lab" | Missing positional table argument |
| "DDL for database movr on VA tenant" | Missing `-d movr` flag |
| "migrate inventory_schema in test_lab to db3" | Complex dual-parameter tool, flags truncated |
| "DDL for rides table in movr" | Missing positional table argument |

The pattern is clear: **positional arguments and complex multi-flag tools** are the remaining weak spot. Two of five failures are the same issue (DDL table missing the table name as a positional arg). These require understanding that some tools take arguments *without* a flag prefix, which is hard to learn from 82 examples.

## What Could Be Done Differently

**1. Start with 300+ training examples, not 82.** The oscillating validation loss tells the whole story. 82 examples is enough for pattern memorization but not generalization. Running 300 diverse prompts through Gemini would take about an hour and dramatically improve stability.

**2. Include paraphrased prompts.** The training data had one phrasing per intent. "List tables in movr" should also appear as "show tables in the movr database," "what tables are in movr," etc. This teaches the model that intent matters more than exact wording.

**3. Add negative examples.** The model doesn't know what *wrong* looks like. DPO (Direct Preference Optimization) pairs — correct command vs. incorrect command for the same prompt — would teach it to avoid common mistakes like wrong tool selection.

**4. Consider constrained decoding.** Ollama supports GBNF grammars that restrict output tokens at the decoder level. A grammar that only allows valid flag names per tool would eliminate hallucinated flags entirely, without any retraining.

## The Economics

| Resource | Cost |
|----------|------|
| Hardware | M1 Pro MacBook (already owned) |
| Cloud API calls for training data | ~$2 (3,105 Gemini queries over time) |
| Training time | 33 minutes |
| Disk space (peak) | ~25 GB (temporary) |
| Final model size | 16 GB in Ollama |
| Per-query inference cost | $0.00 |

Compare that to cloud pricing. At Gemini's input/output token rates, 100 queries per day costs roughly $0.50-1.00/day. Over a year, the local model pays for itself many times over — and it works offline.

## Reproducing This

Everything is scripted and reproducible:

```bash
# 1. Prepare training data from batch test results
python3 tools/utils/finetune/prepare_training_data.py

# 2. Run the full pipeline: train → fuse → convert → import
bash tools/utils/finetune/run_lora.sh

# 3. Test
bash bin/roachie-batch \
  --file examples/queries/example-queries.txt \
  --provider ollama-roachie-8b-ft \
  --tools-version 25.4 \
  --no-feedback --no-followup --allow-sql-files
```

On an Apple Silicon Mac with 16+ GB RAM, the entire process takes under an hour.

## Key Takeaways

1. **Cloud model outputs are free training data.** If queries are already running against a cloud LLM, every successful response is a supervised example. Log them.

2. **LoRA on Apple Silicon is practical.** 33 minutes, 6.87 GB peak memory, no GPU cluster needed. The MLX ecosystem makes this accessible to anyone with a recent Mac.

3. **The conversion pipeline is the hard part.** Training was straightforward. Getting from MLX weights to a running Ollama model required solving three separate compatibility issues (U32 dtype, disk space, missing chat template). Automate this pipeline early.

4. **82 examples isn't enough, but it's a start.** Even with heavy overfitting, the model learned enough to gain 7 percentage points. More data will yield diminishing returns on each example but much more stable generalization.

5. **Infrastructure fixes come first.** The prompt engineering and code fixes gained 33 points (50% → 83%) — more than the 7 points from fine-tuning. Always exhaust the easy wins before training.

6. **The 8B ceiling is around 90-92%.** For complex tool selection with positional args and multi-flag commands, the options are a larger model (70B), constrained decoding, or a validation loop that checks generated commands against `--help` output.

The local LLM isn't as accurate as Gemini. But at 90%, it's accurate enough to be genuinely useful — and it runs entirely on the local machine, with zero latency, zero cost, and zero data leaving the network.

---

*This is Part 2 of a 2-part series. [Part 1]() covers the prompt engineering and infrastructure work that took accuracy from 50% to 83%.*

*The tools and scripts referenced in this post are part of [Roachie](https://github.com/amaddahian/roachie), an open-source CockroachDB DBA toolkit with a natural language interface.*
