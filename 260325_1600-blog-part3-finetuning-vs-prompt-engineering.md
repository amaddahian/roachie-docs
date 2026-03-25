# When Fine-Tuning Loses to Prompt Engineering: Llama 8B at 97% Without Changing a Single Weight

*Part 3 of 3 — The surprising conclusion to the local LLM accuracy journey: retraining made the model worse, not better.*

---

[Part 1]() took Llama 3.1 8B from 50% to 83% through prompt engineering. [Part 2]() pushed it to 90% with LoRA fine-tuning. This was supposed to be the validation post — retrain the fine-tuned model with better data, hit 95%+, and declare victory for LoRA.

Instead, prompt engineering alone reached 97%. The fine-tuned model, retrained with sanitized data, maxed out at 90%. The base model — with zero weight changes — beat the trained one by 7 percentage points.

This post covers what happened, why, and what it reveals about the real dynamics between prompt engineering and fine-tuning for domain-specific tool selection.

## The Setup: Two Weeks of Prompt Engineering

Between Part 2 (March 22) and today, the base roachie-8b model (unmodified Llama 3.1 8B) received continuous prompt improvements:

| Date | Accuracy | What Changed |
|------|----------|-------------|
| Mar 19 | 65% | Baseline (unified prompt, semantic matching) |
| Mar 20 | 72-76% | Modelfile fixes (num_predict, temperature, removed embedded SYSTEM) |
| Mar 21 | 77-82% | Enrichment raised to 7 tools, connection reminders fixed |
| Mar 22 | 89% | Few-shot examples expanded, tool-specific rules added |
| Mar 24 | 93% | Hallucination warnings, WRONG/RIGHT pattern examples |
| **Mar 25** | **96%** | Fixed cr_ddl_table help text, tool discovery rules |

The test suite also shifted from 74 to 70 queries (5 flawed prompts removed, 1 added), but the trend held on the overlapping subset.

67 out of 70 queries now produce the correct command. The 3 remaining failures are genuinely hard:

1. **"show me tool names including view keyword"** — The model runs `cr_grep_views` (a real tool that searches views in databases) instead of `cr_help --search "view"` (meta-query about tool names). This is an ambiguity problem: the prompt doesn't make clear whether "view" refers to database views or the word "view" in tool names.

2. **"list all columns for products table"** — Uses `-T products` (a flag that doesn't exist) instead of `--filter-table products`. The base model gets this wrong despite explicit WRONG/RIGHT examples in the prompt. This is a 3% error rate on a rare flag pattern.

3. **"migrate inventory_schema to db3"** — `cr_migrate` is the most complex tool in the kit, with 12+ flags for source/target host, port, database, tenant, and schema. Every model struggles with it.

## The Retraining Hypothesis

Meanwhile, the fine-tuned model (roachie-8b-ft) from Part 2 was stuck at 86%. It had been trained on 82 examples from March 22 — before most of the prompt improvements. Its training data contained patterns that were now known to be wrong:

- `cr_ddl_table -T rides` (the `-T` flag doesn't exist — table name is positional)
- `cr_help -h localhost -p 26257` (cr_help is local-only, no connection flags)
- `cr_columns -T products` (should be `--filter-table products`)

The hypothesis was straightforward: regenerate training data from current batch results (which now include correct patterns), retrain, and the fine-tuned model should match or exceed the base model.

### Regenerating Training Data

The training data pipeline (`prepare_training_data.py`) was upgraded:

**Multi-provider deduplication.** Instead of using only Gemini outputs, all providers contribute with a priority system: Gemini (priority 3) > roachie-8b (priority 2) > others (priority 1). When the same prompt has results from multiple providers, the highest-priority successful result wins.

**Command sanitization.** A new `sanitize_command()` function fixes known bad patterns before they enter training data:

```python
def sanitize_command(cmd):
    # Fix cr_ddl_table -T TABLE -> positional arg
    cmd = re.sub(r'(cr_ddl_table\s+-d\s+\S+)\s+-T\s+(\S+)', r'\1 \2', cmd)

    # Fix cr_help with connection flags — strip -h, -p, -t, --insecure
    if 'cr_help' in cmd:
        cmd = re.sub(r'\s+-h\s+\S+', '', cmd)
        cmd = re.sub(r'\s+-p\s+\S+', '', cmd)
        # ...

    # Fix cr_columns -T TABLE -> --filter-table TABLE
    cmd = re.sub(r'(cr_columns\s.*?)\s+-T\s+(\S+)', r'\1 --filter-table \2', cmd)
    return cmd
```

This produced **93 unique training examples** (up from 82), split into 74 train / 9 valid / 10 test.

### Retraining

Same MLX LoRA configuration as Part 2: rank 8, 16 layers, learning rate 1e-5, 600 iterations on an M1 Pro MacBook. Training took about 33 minutes.

One change: the final model was exported as GGUF Q8_0 (8.5 GB) instead of FP16 (16 GB) to manage disk space. Q8_0 quantization has negligible accuracy impact on 8B models.

## The Results

| Model | Accuracy | Notes |
|-------|----------|-------|
| **roachie-8b (base)** | **96% (67/70)** | Prompt engineering only |
| roachie-nemo-ft (Mistral 12B) | 91% (64/70) | LoRA fine-tuned |
| **roachie-8b-ft (retrained)** | **90% (63/70)** | LoRA with sanitized data |
| roachie-8b-ft (old training) | 86% (60/70) | LoRA with March 22 data |
| mistral-nemo (base) | 86% (60/70) | Untuned baseline |

The retrained model improved from 86% to 90% — the sanitized training data helped. But it still falls 6 points short of the base model.

### What the Fine-Tuned Model Gets Wrong

Comparing the 7 failures of the retrained roachie-8b-ft against the 3 failures of the base model:

| Prompt | Base | Fine-Tuned | Issue |
|--------|------|-----------|-------|
| #32 list groups for test_lab | Pass | **Fail** | cr_get_acl execution error |
| #39 tool names with "view" keyword | Fail | **Fail** | Both hallucinate wrong tool |
| #40 columns for products table | Fail | **Fail** | Both use wrong flag (-T) |
| #61 top 3 longest running queries | Pass | **Fail** | FT adds hallucinated cr_plan command |
| #64 migrate inventory_schema to db3 | Fail | **Fail** | Both — cr_migrate is too complex |
| #66 DDL for rides table | Pass | **Fail** | FT uses `-T rides` despite sanitization |

The fine-tuned model fails on everything the base model fails on, plus 4 additional prompts. Two of those extra failures (#61, #66) are cases where the LoRA weights actively override the system prompt with memorized-but-wrong patterns.

## Why Fine-Tuning Makes Things Worse

### The Weight Override Problem

LoRA adapters modify the model's attention layers. When the training data contains a strong pattern — like "when the user mentions DDL and a table name, use `-T`" — the adapter learns it as a weighted bias in the attention mechanism. At inference time, this bias competes with the system prompt.

For the base model, the system prompt says:

```
WRONG: cr_ddl_table -d DB -T TABLE (the -T flag does NOT exist)
RIGHT: cr_ddl_table -d movr rides -h HOST -p PORT -t TENANT --insecure
```

The base model reads this and follows it (96% of the time). But the fine-tuned model has seen `cr_ddl_table -T` dozens of times during training — even though the retraining sanitized this specific pattern, only 2-3 training examples contained cr_ddl_table at all. The adapter weights learned the `-T` pattern from 600 iterations on the original data, and the 93 sanitized examples couldn't fully override it.

This is the core issue: **LoRA weights are persistent biases, while prompt instructions are transient context.** When they agree, fine-tuning amplifies accuracy. When they disagree, the weights often win.

### The Overfitting Tax

With only 93 training examples, the model memorizes the training set by iteration 200-300. The validation loss curve oscillates wildly (0.3 to 1.3), confirming that the model isn't learning generalizable patterns — it's learning specific prompt-to-command mappings.

This memorization creates brittleness. When a prompt is phrased slightly differently from any training example, the model falls back on its learned biases rather than following the system prompt. The base model, with no learned biases, follows the prompt more faithfully.

### The Prompt Engineering Treadmill

Each prompt improvement changes what "correct" means. New few-shot examples, hallucination warnings, and tool-specific rules shift the expected behavior. But the fine-tuned model's weights reflect the *old* definition of correct. Retraining is required after every prompt change — and retraining takes 33 minutes plus data preparation.

The base model adapts instantly to prompt changes. No retraining, no data preparation, no GGUF conversion.

## The Mistral Comparison: Where Fine-Tuning Does Help

The picture changes for Mistral Nemo 12B:

| Model | Accuracy | Delta |
|-------|----------|-------|
| mistral-nemo (base) | 86% (60/70) | — |
| roachie-nemo-ft (LoRA) | 91% (64/70) | **+5%** |

Fine-tuning improved Mistral by 5 percentage points. Why does it help here but hurt Llama?

**The base accuracy is the key variable.** Mistral's base accuracy (86%) leaves significant room for improvement. The training data teaches patterns the model couldn't infer from prompts alone. But Llama's base accuracy (96%) means the prompts already cover almost everything. Fine-tuning can only add value for the remaining 4% — and for some of those cases, the learned weights actually override correct prompt-driven behavior.

The crossover point appears to be around 90-92%. Below that, fine-tuning helps. Above that, it hurts.

## What Actually Worked: The Prompt Engineering Playbook

The 65% to 96% journey was entirely prompt-driven. Here's what moved the needle, roughly in order of impact:

### 1. Unified Prompt Path (65% to 83%)

The single biggest improvement was giving Llama the same prompt as cloud models. The original code had an Ollama-specific "compact" prompt that stripped out 7 template files. Removing the special case and using the full prompt — which fit comfortably in Llama's 16K context — added 18 points.

Lesson: **Don't assume small models need less context. They often need more.**

### 2. Few-Shot Examples with WRONG/RIGHT Patterns (83% to 89%)

Generic examples like "list tables → cr_db_tables" helped, but explicit anti-patterns were more effective:

```
WRONG: cr_ddl_table -d DB -T TABLE (the -T flag does NOT exist)
RIGHT: cr_ddl_table -d movr rides -h HOST -p PORT -t TENANT --insecure
```

The model learns what to avoid, not just what to do. Three WRONG/RIGHT pairs for cr_ddl_table alone added 2% accuracy.

### 3. Hallucination Warnings (89% to 93%)

A dedicated section listing the top mistakes:

```
MISTAKE #1: Using -T flag with cr_ddl_table — THIS FLAG DOES NOT EXIST
MISTAKE #2: Adding -h/-p/-t/--insecure to cr_help — IT IS LOCAL-ONLY
```

These are essentially "negative few-shot examples" — they address the most common failure modes directly.

### 4. Tool Help Text Fixes (93% to 96%)

The tools' own `--help` output was teaching wrong patterns. `cr_ddl_table` had:

```
Usage: cr_ddl_table [OPTIONS] [table]
Example: cr_ddl_table -d mydb -t users --insecure
```

The example confused `-t` (tenant) with the table name. Fixing it to:

```
Usage: cr_ddl_table [OPTIONS] TABLE_NAME
Example: cr_ddl_table -d mydb users -t system --insecure
```

This matters because the tool's `--help` output is injected into the prompt via enrichment. Bad help text means bad prompt context.

## The Full Model Comparison

Five models, same 70 queries, same day:

| Model | Size | Type | Accuracy | Per-Query Cost |
|-------|------|------|----------|---------------|
| Gemini 2.5 Flash | Cloud | Cloud API | 97% (68/70) | ~$0.01 |
| **roachie-8b (Llama 3.1)** | **4.9 GB** | **Base + prompts** | **96% (67/70)** | **$0.00** |
| roachie-nemo-ft (Mistral) | 13 GB | LoRA fine-tuned | 91% (64/70) | $0.00 |
| roachie-8b-ft (Llama 3.1) | 8.5 GB | LoRA fine-tuned | 90% (63/70) | $0.00 |
| mistral-nemo (base) | 7.1 GB | Untuned | 86% (60/70) | $0.00 |

The base Llama 3.1 8B — with no fine-tuning, no weight modification, just good prompts — is within 1 percentage point of Gemini 2.5 Flash. And it runs locally, offline, for free.

## When to Fine-Tune (and When Not To)

Based on this experience, fine-tuning makes sense when:

1. **Base accuracy is below ~90%.** There's enough headroom for learned patterns to help without overriding correct prompt-driven behavior.
2. **Prompts are stable.** If the prompt template, few-shot examples, and tool documentation aren't changing, trained weights won't drift out of alignment.
3. **The training data is large and diverse.** 93 examples isn't enough for stable generalization. 300+ diverse examples with paraphrased prompts would likely improve results.
4. **The task has patterns that can't be expressed in prompts.** Some mappings are too complex or numerous for in-context learning. Tool selection for 77 tools with varied flag patterns is borderline — it can be done in-context, but it's not trivial.

Fine-tuning is counterproductive when:

1. **Base accuracy is already high (90%+).** The risk of weight override exceeds the potential gain.
2. **Prompts are actively evolving.** Every prompt change invalidates the training and potentially creates conflicts.
3. **Training data is small.** With fewer than ~100 examples, the model memorizes rather than generalizes, creating brittleness.
4. **The wrong patterns exist in training data.** Even with sanitization, bad patterns from historical data can persist in LoRA weights and override correct prompt instructions.

## The Economics of Prompt Engineering vs. Fine-Tuning

| Approach | Time Investment | Accuracy Gain | Maintenance |
|----------|----------------|---------------|-------------|
| Prompt engineering | Minutes per change | 65% → 96% (+31 pts) | Zero — changes apply instantly |
| LoRA fine-tuning | ~1 hour per cycle | 65% → 90% (+25 pts) | Retrain after every prompt change |
| Combined (prompts + LoRA) | Both | 90% (worse than prompts alone) | High — must keep weights and prompts aligned |

The prompt engineering approach is not only more effective — it's dramatically cheaper to maintain. Each improvement is a text edit that takes effect immediately. Fine-tuning requires data preparation, a 33-minute training run, GGUF conversion, Ollama import, and testing. And if the prompts change (which they will), the whole cycle repeats.

## Conclusion

The original narrative was clean: prompt engineering gets the easy wins, fine-tuning pushes past the ceiling. Three weeks of work produced a different conclusion.

**There may not be a ceiling — just diminishing returns on prompt quality.** Each improvement requires more careful analysis: studying failure patterns, crafting targeted WRONG/RIGHT examples, fixing tool documentation, adding hallucination warnings for specific mistakes. The gains per change get smaller. But they compound, and they don't carry the maintenance burden of fine-tuning.

The fine-tuned model isn't useless. For Mistral Nemo (base accuracy 86%), LoRA training added 5 genuine percentage points. For environments where the prompt template is frozen and won't change, fine-tuning could provide a stable accuracy floor. And with 300+ training examples, the generalization problems would likely diminish.

But for a system where prompts evolve weekly, where tool documentation gets corrected, where new few-shot examples are added as failure patterns emerge — prompt engineering is the better investment. The base Llama 3.1 8B, running locally on a MacBook Pro, matches Gemini Flash at 96% accuracy. No training required.

---

*This is Part 3 of a 3-part series.*
- *[Part 1](): From 50% to 83% — prompt engineering and infrastructure fixes*
- *[Part 2](): From 83% to 90% — LoRA fine-tuning on Apple Silicon*
- *Part 3 (this post): Why prompt engineering won*

*The tools and scripts referenced in this post are part of [Roachie](https://github.com/amaddahian/roachie), an open-source CockroachDB DBA toolkit with a natural language interface.*
