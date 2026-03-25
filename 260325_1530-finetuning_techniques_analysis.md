# Roachie Fine-Tuning Techniques Analysis

**Date:** 2026-03-25
**Scope:** Evaluation of roachie's model optimization practices against 8 fine-tuning and alignment techniques

---

## Overview -- Current Fine-Tuning Landscape

| Technique | Used in Roachie? | Implementation | Maturity |
|-----------|-----------------|----------------|----------|
| 1. LoRA | **Yes (primary)** | MLX LoRA via `run_lora.sh` on Llama 3.1 8B + Mistral Nemo 12B | Production |
| 2. QLoRA | **Effectively yes** | Base models are 4-bit quantized (MLX 4-bit), adapters in FP16 | Production |
| 3. Full Fine-Tuning | No | -- | -- |
| 4. DPO | No | Infrastructure exists (feedback data) but not connected | Opportunity |
| 5. PPO | No | -- | -- |
| 6. GRPO | No | -- | -- |
| 7. RLHF | No | Feedback collection exists but doesn't train reward model | Partial |
| 8. LLM-as-a-Judge | **Partially** | Gemini generates training data; batch testing evaluates | Emerging |

---

## 1. LoRA (Low-Rank Adaptation)

**Technique:** Freeze base model weights, train small low-rank adapter matrices on top. Adapters are typically <1% of base model parameters.

### How Roachie Uses LoRA

Roachie has a **complete LoRA fine-tuning pipeline**:

| Component | Implementation | File |
|-----------|---------------|------|
| Data preparation | Extract from Gemini batch results, deduplicate, sanitize, split 80/10/10 | `prepare_training_data.py` |
| Training | MLX LoRA with configurable rank, layers, epochs, learning rate | `run_lora.sh:121-133` |
| Fusion | LoRA adapters merged into base model weights | `run_lora.sh:139-142` |
| De-quantization | MLX 4-bit -> BF16 for Ollama import | `run_lora.sh:152-160` |
| GGUF conversion | BF16 -> Q8_0 GGUF for Mistral (Ollama requirement) | `run_lora.sh:201-222` |
| Deployment | Ollama model with explicit chat template + stop tokens | `run_lora.sh:228-242` |
| Validation | Batch testing against 73 prompts with accuracy reporting | `roachie-batch` |

**Configuration:**
```
Base model:    mlx-community/Meta-Llama-3.1-8B-Instruct-4bit
LoRA rank:     8
LoRA layers:   16
Epochs:        20
Batch size:    1
Learning rate: 1e-5
Training size: ~60-80 examples (80% of ~75-100 unique prompts)
```

**Results:**
- roachie-8b-ft (Llama 3.1 8B): 90% accuracy (73 prompts, batch test)
- roachie-nemo-ft (Mistral Nemo 12B): 89% accuracy (69 queries, live)

### Pros

- **End-to-end pipeline is automated and reproducible** -- A single `bash run_lora.sh` command takes a base model through training, fusion, conversion, and Ollama import. The pipeline handles both Llama (safetensors import) and Mistral (GGUF required) architectures automatically.
- **Low resource requirements** -- LoRA rank 8 with 16 layers means training <1M adapter parameters on top of an 8B model. This runs on 16GB Apple Silicon in under an hour. No GPU cluster needed.
- **Data pipeline is well-designed** -- `prepare_training_data.py` handles deduplication by normalized prompt, provider priority (Gemini > roachie-8b > others), command sanitization (fixing known bad patterns), and multi-command support. The 80/10/10 split with fixed seed ensures reproducible evaluation.
- **Chat template handling is sophisticated** -- The pipeline correctly handles the critical but often-missed detail of chat templates: Llama 3.x uses `<|start_header_id|>role<|end_header_id|>` format while Mistral uses `[INST] content [/INST]`. Without explicit templates, Ollama defaults to raw text which breaks fine-tuned models. The pipeline also copies `chat_template` from the base model's tokenizer_config when MLX de-quantization strips it.
- **Model family auto-detection** -- The script automatically detects Llama vs Mistral architectures and configures stop tokens, GGUF requirements, and chat templates accordingly.

### Cons

- **Training data is very small** -- 60-80 examples is at the absolute minimum for LoRA. Research suggests 200-500 examples for domain-specific tasks produce significantly better results. The 90% accuracy ceiling is likely a data quantity issue as much as a data quality issue.
- **No system prompt in training examples** -- Training examples contain only user/assistant pairs. At inference, the model receives a 3-10K token system prompt it never saw during training. This is the most impactful gap (discussed in v19 review R1).
- **No hyperparameter search** -- The pipeline uses fixed hyperparameters (rank 8, lr 1e-5, 20 epochs). No grid search, no validation-loss-based early stopping, no rank comparison (4 vs 8 vs 16). The current values may not be optimal.
- **Single-command bias** -- Nearly all training examples have `"commands": [single_command]`. The model rarely sees multi-command or zero-command (informational) responses. This creates a bias toward always generating exactly one command.
- **No adapter versioning** -- Adapter weights are overwritten on each training run. There is no version tracking, no comparison between adapter versions, and no rollback mechanism.
- **Validation loss is the only training metric** -- The pipeline reports training and validation loss but doesn't compute task-specific metrics during training (e.g., exact-match accuracy on tool names, flag correctness rate).

### Room to Improve

1. **Scale training data to 300-500 examples** by running batch tests with Gemini across a wider query set. Include edge cases, multi-tool queries, and informational queries.
2. **Add system prompt to training examples** -- Generate a representative system prompt at training time and include it as the `system` message. This teaches the model to follow connection rules and flag syntax.
3. **Implement hyperparameter sweep** -- Test LoRA rank {4, 8, 16, 32} and learning rate {5e-6, 1e-5, 2e-5}. Use validation loss as the selection criterion. The `--save-every 200` flag already saves checkpoints.
4. **Add early stopping** -- Monitor validation loss every 100 steps and stop training when it hasn't improved for 3 consecutive evaluations. MLX LoRA supports this via `--early-stop`.
5. **Version adapter outputs** -- Save adapters with timestamps (e.g., `adapters/20260325_llama8b_r8_e20/`) and keep a `TRAINING_LOG.md` with loss curves and accuracy results.

---

## 2. QLoRA (Quantized LoRA)

**Technique:** LoRA on a 4-bit quantized base model. Adapters remain in higher precision (FP16/BF16). Dramatically reduces memory requirements.

### How Roachie Uses QLoRA

Roachie **effectively uses QLoRA** without calling it that:

| Aspect | Implementation |
|--------|---------------|
| Base model quantization | MLX 4-bit (e.g., `mlx-community/Meta-Llama-3.1-8B-Instruct-4bit`) |
| Adapter precision | FP16 (MLX default for LoRA adapters) |
| Training precision | Mixed precision (4-bit base + FP16 adapters + FP32 gradients) |
| Deployment | De-quantized to BF16, then Ollama re-quantizes on import (typically Q4_K_M) |

The key QLoRA characteristic -- training adapters on a quantized base -- is present because the base models are MLX 4-bit quantized variants from the `mlx-community` Hub.

### Pros

- **Memory efficient** -- Training Llama 3.1 8B in 4-bit requires ~6GB VRAM (vs ~16GB for FP16). This is critical for Apple Silicon where unified memory is shared between CPU and GPU. Training on 16GB Macs is possible only because of 4-bit quantization.
- **MLX 4-bit is high quality** -- MLX uses GPTQ-style 4-bit quantization which is more accurate than naive round-to-nearest. The quality loss from 4-bit base weights is minimal for the domain-specific fine-tuning task.
- **Automatic de-quantization for deployment** -- The pipeline de-quantizes fused weights to BF16 before Ollama import (lines 149-160). This means the final deployed model can be quantized by Ollama using its preferred quantization scheme, rather than inheriting the training quantization.
- **Mistral GGUF path handles the edge case** -- Ollama cannot import Mistral safetensors directly. The pipeline correctly detects this and adds the GGUF Q8_0 conversion step. This is a nuance most tutorials miss.

### Cons

- **No comparison with FP16 base training** -- It's unknown how much accuracy is lost from QLoRA vs standard LoRA with an FP16 base. For the small 60-80 example dataset, the quantization-induced noise might be significant relative to the small gradient updates.
- **Double quantization not used** -- QLoRA's original paper (Dettmers et al.) uses double quantization (quantizing the quantization constants themselves) for further memory savings. MLX's 4-bit implementation doesn't use this.
- **Quantization format differs between training and deployment** -- Training uses MLX 4-bit, deployment uses Q4_K_M (Ollama default) or Q8_0 (GGUF). This quantization mismatch means the model is evaluated on different precision than it was trained on. The BF16 de-quantization step mitigates this, but it's an unnecessary round-trip.
- **No INT8 option for quality-sensitive use** -- For users with 32GB+ Apple Silicon, an 8-bit base model would provide better training quality with acceptable memory usage. The pipeline doesn't offer this choice.

### Room to Improve

1. **Add `--precision` flag** -- Allow `run_lora.sh --precision 4bit|8bit|fp16` to let users with more memory trade compute for quality. The 8-bit option (`mlx-community/Meta-Llama-3.1-8B-Instruct-8bit`) would be a good quality midpoint.
2. **Benchmark QLoRA vs LoRA** -- Train with both the 4-bit and FP16 base models on the same data and compare accuracy on the test set. If the gap is significant (>2%), document the trade-off.
3. **Consider matching training and deployment quantization** -- If the deployment target is Q8_0 GGUF, consider training on the 8-bit base to minimize the training/deployment mismatch.

---

## 3. Full Fine-Tuning

**Technique:** Update all model weights (no frozen layers). Maximum accuracy but requires the most compute, memory, and data.

### How Roachie Uses Full Fine-Tuning

**Roachie does not use full fine-tuning.** This is the correct decision.

### Pros (of NOT using full fine-tuning)

- **Resource appropriate** -- Full fine-tuning of Llama 3.1 8B requires ~32GB+ VRAM for FP16 or ~64GB for mixed precision with gradients. This exceeds consumer Apple Silicon capabilities. LoRA is the right choice for single-machine training.
- **Avoids catastrophic forgetting** -- Full fine-tuning on a small dataset (60-80 examples) would likely cause catastrophic forgetting of the base model's general capabilities. LoRA's frozen base preserves the model's language understanding while adding domain-specific behavior.
- **Faster iteration** -- LoRA training takes <1 hour on Apple Silicon. Full fine-tuning would take 8-24 hours for the same data. Given the rapid iteration cycle (change prompt rules -> regenerate data -> retrain), LoRA's speed is essential.

### Cons (of NOT using full fine-tuning)

- **Ceiling on adaptation depth** -- LoRA can only modify the low-rank subspace of attention weights. Deep behavioral changes (e.g., learning CockroachDB's entire SQL dialect) may require full fine-tuning. The 90% accuracy ceiling might partially be a LoRA capacity limitation.
- **No base model specialization** -- Full fine-tuning could produce a CockroachDB-specialized model with internalized knowledge of CRDB concepts (ranges, leaseholders, tenants). LoRA adapts behavior but doesn't add new knowledge.

### Room to Improve

1. **Consider full fine-tuning when data reaches 1000+ examples** -- At the current 60-80 examples, full fine-tuning would overfit. But if the dataset grows to 1000+ through expanded batch testing and user feedback, a full fine-tune could surpass LoRA's ceiling.
2. **Explore continued pre-training** -- A middle ground: continue pre-training the base model on CockroachDB documentation (unsupervised) to inject domain knowledge, then apply LoRA for task-specific fine-tuning. This two-stage approach avoids catastrophic forgetting while adding knowledge that LoRA alone cannot.

---

## 4. DPO (Direct Preference Optimization)

**Technique:** Train the model to prefer "chosen" outputs over "rejected" outputs for the same input. No reward model needed -- preference learning is built into the loss function.

### How Roachie Uses DPO

**Roachie does not use DPO**, but has the **infrastructure to generate preference data**:

| Data Source | Available? | Format | How It Maps to DPO |
|-------------|-----------|--------|---------------------|
| Successful batch results | Yes | JSONL with prompt + command + status | `chosen` response (status=success) |
| Failed batch results | Yes | JSONL with prompt + command + error | `rejected` response (status=fail) |
| User feedback (positive) | Yes | JSONL with prompt + response + feedback="positive" | `chosen` |
| User feedback (negative) | Yes | JSONL with prompt + response + feedback="negative" | `rejected` |
| Persistent failures | Yes | JSONL with prompt + commands + error_output | `rejected` |
| Persistent successes | Yes | JSONL with prompt + commands | `chosen` |

### Pros (of the current state + DPO potential)

- **Preference data already exists** -- The batch testing pipeline naturally produces (chosen, rejected) pairs: same prompt, different providers, one succeeds and one fails. No additional labeling effort needed.
- **User feedback is preference data** -- The `_nl_collect_feedback()` function collects y/n ratings with optional comments. A "y" response paired with a "n" response for similar queries creates DPO training pairs.
- **DPO would address the biggest accuracy gap** -- The 10% failure rate on local models is concentrated in specific failure patterns: wrong flags, missing parameters, wrong tool selection. DPO directly teaches the model "for this query, this output is correct and this output is wrong."
- **MLX supports DPO** -- The `mlx-lm` library supports DPO training with the same adapter-based approach used for LoRA. No new tooling required.

### Cons (of NOT using DPO)

- **Valuable feedback data is wasted** -- The persistent failure/success databases contain exactly the preference data DPO needs, but this data is only used for prompt injection (injecting past failures into the system prompt). It is never used to update model weights.
- **Missing structured pairing** -- The current data stores successes and failures independently. DPO requires explicit (chosen, rejected) pairs for the **same prompt**. Building these pairs requires matching failures to the correct response from a different provider or run.
- **No negative examples in training** -- Without DPO, the model only learns what to do (from successful examples). It never learns what NOT to do. This is why the model sometimes generates commands with `-T TABLE` instead of positional args -- it has never been penalized for this pattern.

### Room to Improve

1. **Build a DPO data pipeline** -- Extend `prepare_training_data.py` with a `--dpo` flag:
   ```python
   # For each prompt that has both a success and a failure:
   {
       "prompt": "show tables in movr",
       "chosen": '{"commands": [{"command": "cr_tables -d movr -h H -p P -t T --insecure"}]}',
       "rejected": '{"commands": [{"command": "cr_tables -T movr -h H -p P --insecure"}]}'
   }
   ```
   Match by normalized prompt across batch runs. Prefer Gemini successes as `chosen` and Ollama failures as `rejected`.

2. **Mine DPO pairs from persistent learning** -- The `NL_FAILURES_DB` and `NL_SUCCESSES_DB` files contain failures and successes that can be paired by query similarity (cosine similarity on embeddings). Even approximate matches are useful for DPO.

3. **Start with 30-50 DPO pairs after LoRA** -- Apply DPO as a second training stage after the initial LoRA SFT. This is the standard recipe: SFT first (learn the format), DPO second (learn preferences).

---

## 5. PPO (Proximal Policy Optimization)

**Technique:** Reinforcement learning with a reward model. The policy (LLM) generates responses, the reward model scores them, and PPO updates the policy to maximize reward while staying close to the original policy.

### How Roachie Uses PPO

**Roachie does not use PPO.** This is appropriate for the current stage.

### Pros (of NOT using PPO)

- **Complexity not justified** -- PPO requires training and maintaining a separate reward model, managing the policy/reference model pair, handling KL divergence, and tuning RL hyperparameters (clip ratio, GAE lambda, etc.). For a 60-80 example dataset, this infrastructure overhead far exceeds the benefit.
- **DPO achieves similar results with less complexity** -- For the preference learning objective (choose correct commands over incorrect ones), DPO is mathematically equivalent to PPO with an implicit reward model. DPO is the better starting point.

### Cons (of NOT using PPO)

- **No online learning capability** -- PPO can learn from the model's own generated outputs (on-policy learning). This means the model could self-improve by generating commands, executing them, and using the execution result as a reward signal. This creates a self-reinforcing loop that offline methods (LoRA, DPO) cannot achieve.
- **Cannot optimize for execution success directly** -- PPO could use a reward function based on actual command execution: `reward = 1.0 if exit_code == 0 else -1.0`. This directly optimizes for what matters (commands that work) rather than matching a static training set.

### Room to Improve

1. **Not recommended at current scale** -- PPO requires hundreds to thousands of on-policy samples. At 60-80 training examples and local model inference speed (~5-30s per sample), generating enough PPO samples would take hours per training iteration.
2. **Consider for future scale** -- If roachie reaches 500+ users generating thousands of feedback samples, a PPO setup with `command_exit_code` as the reward signal would be the highest-quality fine-tuning method. The infrastructure already exists (command execution with exit codes, output capture, feedback collection).

---

## 6. GRPO (Group Relative Policy Optimization)

**Technique:** Sample a group of responses for each prompt, rank them, and use relative rankings as the reward signal. No separate reward model needed. More stable than PPO for reasoning tasks.

### How Roachie Uses GRPO

**Roachie does not use GRPO**, but the batch testing infrastructure provides a natural foundation.

### Pros (of GRPO potential)

- **Multi-provider batch testing is a group sampling mechanism** -- When `roachie-batch` runs the same prompts against multiple providers (Gemini, OpenAI, Ollama), it produces exactly the "group of responses per prompt" that GRPO needs. The provider comparison already ranks them by success/failure.
- **Natural reward signal** -- For each prompt group, the ranking is deterministic: `executed_successfully > validation_failed > wrong_tool > no_command`. No subjective human judgment needed.
- **Better for reasoning** -- GRPO was designed for improving chain-of-thought reasoning. Roachie's `reasoning` field in responses is exactly this: the model explains why it chose a particular tool and flags. GRPO could improve the quality of this reasoning.

### Cons (of NOT using GRPO)

- **Not supported by MLX** -- GRPO is relatively new (DeepSeek-R1, 2024) and not yet implemented in the MLX ecosystem. Implementation would require a custom training loop.
- **Requires multiple samples per prompt** -- GRPO needs 4-16 responses per prompt to form meaningful group rankings. At local inference speeds, generating 8 responses x 75 prompts = 600 inference calls would take several hours.

### Room to Improve

1. **Collect GRPO-ready data passively** -- When running batch tests across providers, store all responses (not just the best) in a structured format:
   ```json
   {"prompt": "...", "responses": [
     {"provider": "gemini", "command": "...", "status": "success", "rank": 1},
     {"provider": "ollama-roachie-8b", "command": "...", "status": "fail", "rank": 3},
     {"provider": "openai", "command": "...", "status": "success", "rank": 2}
   ]}
   ```
   When GRPO support arrives in MLX (or via a Python GRPO library), this data is ready.

2. **Use GRPO for reasoning quality** -- The highest-impact GRPO application would be improving the `reasoning` field. Good reasoning leads to correct tool selection. Train the model to produce reasoning that consistently leads to the right command.

---

## 7. RLHF (Reinforcement Learning from Human Feedback)

**Technique:** Humans label preference data -> train a reward model -> use PPO to optimize the policy against the reward model. The canonical alignment technique.

### How Roachie Uses RLHF

Roachie has **half of the RLHF pipeline** -- the human feedback collection -- but doesn't close the loop with reward model training or policy optimization.

| RLHF Component | Roachie Status | Implementation |
|----------------|---------------|----------------|
| Human feedback collection | **Yes** | `_nl_collect_feedback()` -- y/n/skip with optional comments |
| Feedback storage | **Yes** | JSONL (`NL_FEEDBACK_FILE`) + CSV (`NL_FEEDBACK_CSV`) |
| Preference pairs | **Partial** | Positive/negative feedback stored independently, not paired |
| Reward model training | **No** | -- |
| Policy optimization | **No** | -- |
| Feedback-to-prompt injection | **Yes** | Persistent failures/successes injected into system prompt |

### Pros

- **Feedback infrastructure is well-built** -- The feedback collection captures: timestamp, provider, user query, LLM reasoning, response, executed commands, execution results, user feedback (y/n), and free-text comments. This is rich enough for reward model training.
- **Implicit RLHF via prompt injection** -- While roachie doesn't train a reward model, it achieves a similar effect by injecting past failures into the system prompt. The LLM sees "this query failed with these commands" and adjusts its behavior. This is "RLHF without the RL" -- using the feedback data at inference rather than training time.
- **Persistent learning provides immediate benefit** -- Unlike RLHF which requires a training cycle, the persistent learning database takes effect immediately. A failure logged in session N is available to prevent the same mistake in session N+1.

### Cons

- **Feedback data is sparse and opt-in** -- Most users skip feedback (press Enter). The persistent learning database holds at most 15 failures and 10 successes. This is too small for reward model training.
- **No feedback on partial correctness** -- The y/n binary doesn't capture "the right tool but wrong flags" or "correct command but wrong database." A 1-5 scale or multi-dimensional feedback would be more useful for RLHF.
- **Feedback loop doesn't close** -- The most critical gap: feedback data is collected but never used to update model weights. It's injected into prompts (a weak form of learning) but the model itself never improves from feedback.
- **No reward model** -- Without a reward model, there's no automated way to score new model outputs. Every response requires human evaluation to determine quality.

### Room to Improve

1. **Add structured feedback dimensions** -- Replace y/n with:
   ```
   Tool correct? (y/n)  Flags correct? (y/n)  Output useful? (y/n)
   ```
   This provides richer signal for reward model training and helps identify whether failures are tool selection, flag syntax, or response format issues.

2. **Build a simple reward model** -- Train a classifier on the feedback data that predicts `feedback=positive` from (query, response) pairs. Even a logistic regression on TF-IDF features would provide automated scoring. Use this to evaluate new model versions without manual testing.

3. **Close the feedback loop via DPO** -- Convert feedback data into DPO pairs (positive feedback = chosen, negative feedback = rejected for similar queries). This is easier than full RLHF and achieves most of the benefit.

---

## 8. LLM-as-a-Judge

**Technique:** Use a capable LLM (e.g., GPT-4, Claude) to evaluate the outputs of another LLM at scale. Score for quality, correctness, safety, and adherence to instructions.

### How Roachie Uses LLM-as-a-Judge

Roachie **partially implements** LLM-as-a-Judge through its multi-provider training data pipeline:

| LLM-as-a-Judge Aspect | Roachie Implementation | Status |
|------------------------|----------------------|--------|
| Teacher model generates training data | Gemini batch results as training source | **Yes** |
| Automated quality scoring | Command execution success/failure | **Yes (execution-based, not LLM-based)** |
| Multi-dimensional evaluation | Only pass/fail; no flag accuracy, reasoning quality, or response coherence | **Partial** |
| Comparative evaluation | Same prompts run across providers with accuracy comparison | **Yes** |
| Automated at scale | `roachie-batch` runs 73+ prompts across providers | **Yes** |
| LLM judges LLM output | Not implemented -- uses execution result as proxy | **No** |

### Pros

- **Execution-as-judge is superior to LLM-as-judge for this domain** -- For tool-calling, the ultimate ground truth is "did the command execute successfully?" This is more reliable than asking an LLM to evaluate whether a command looks correct. Roachie's approach of running the command and checking the exit code is a form of **environment-as-judge** that is strictly more accurate than LLM-based evaluation.
- **Multi-provider comparison provides relative quality** -- Running the same prompts against Gemini, OpenAI, and Ollama creates a natural leaderboard. The provider with the highest pass rate on the same test set is the best model. No LLM judge needed.
- **Batch assertions enable rule-based judging** -- `roachie-batch --assertions` allows regex-based validation of expected output patterns. This is a deterministic, auditable evaluation that LLM judges cannot match for consistency.
- **Knowledge distillation through data generation** -- Using Gemini (a more capable model) to generate training data for Ollama (a smaller model) is a form of LLM-as-a-teacher. The smaller model learns to mimic the teacher's behavior.

### Cons

- **No LLM evaluation of reasoning quality** -- The `reasoning` field is never evaluated for correctness, clarity, or logical consistency. An LLM judge could assess: "Does the reasoning correctly identify the user's intent? Does it justify the tool choice? Are the flag selections logical?"
- **No LLM evaluation of response coherence** -- The `user_message` field (what the user sees) is never evaluated for helpfulness, accuracy, or tone. An LLM judge could catch responses like "I'll run the appropriate command for you" (generic, unhelpful) vs. "I'll check the row counts in the movr database on tenant va" (specific, informative).
- **Pass/fail is too coarse** -- A command can succeed but be suboptimal (e.g., `cr_tables` when `cr_db_tables_rowcount` would better match the intent). Execution-based evaluation misses these quality distinctions.
- **No automated regression detection** -- When a new model version is deployed, there's no automated way to check if previously correct responses are now wrong. An LLM judge could compare new vs old responses and flag regressions.

### Room to Improve

1. **Add an LLM-as-a-judge evaluation stage to batch testing** -- After executing each prompt, send the (query, reasoning, command, output) tuple to a capable LLM (Claude or Gemini) with a structured rubric:
   ```
   Score 1-5 on each dimension:
   - Tool selection: Did the model choose the best tool for this query?
   - Flag accuracy: Are all flags correct and complete?
   - Reasoning quality: Does the reasoning logically lead to the command?
   - Response helpfulness: Is the user_message informative and specific?
   ```
   Store scores alongside execution results for comprehensive quality tracking.

2. **Use LLM-as-a-judge for training data quality** -- Before using Gemini's outputs as LoRA training data, have a second LLM (Claude) verify each example. Flag examples where the command is correct but the reasoning is wrong, or where the response is technically correct but not the best approach.

3. **Build a regression detection pipeline** -- After retraining, run the test set with both old and new models. Send any cases where the old model succeeded and the new model failed to an LLM judge for analysis. This catches regressions before deployment.

4. **Combine execution-as-judge with LLM-as-judge** -- The strongest evaluation combines both:
   - **Execution judge** (deterministic): Did the command run? Exit code 0?
   - **LLM judge** (qualitative): Was it the *best* command? Was the reasoning sound?
   - **Composite score**: `0.6 * execution_pass + 0.4 * llm_quality_score`

---

## Summary Scorecard

| Technique | Roachie Score | Impact If Adopted | Effort to Implement | Priority |
|-----------|--------------|-------------------|---------------------|----------|
| 1. LoRA | **8/10** | Already primary method | Already done | Improve data quality |
| 2. QLoRA | **7.5/10** | Already in use (4-bit base) | Already done | Add 8-bit option |
| 3. Full Fine-Tuning | **N/A** | Low (data too small) | High | Not recommended now |
| 4. DPO | **2/10** (potential: 8/10) | **High** -- directly teaches preferences | **Medium** -- data exists, needs pairing + training | **#1 priority** |
| 5. PPO | **0/10** | Medium -- online learning | High -- needs RL infrastructure | Not recommended now |
| 6. GRPO | **0/10** | Medium -- better reasoning | High -- no MLX support | Collect data passively |
| 7. RLHF | **3/10** (feedback exists) | Medium -- alignment | High -- needs reward model | Close loop via DPO instead |
| 8. LLM-as-a-Judge | **5/10** | **High** -- automated quality at scale | **Low** -- add to batch pipeline | **#2 priority** |

### Recommended Roadmap

**Phase 1 -- Immediate (improve existing LoRA):**
1. Scale training data to 300+ examples
2. Include system prompt in training examples
3. Add negative examples (informational queries with empty commands)
4. Run hyperparameter sweep (LoRA rank, learning rate)

**Phase 2 -- Next sprint (add DPO):**
5. Build DPO data pipeline from batch test results (pair successes with failures for same prompt)
6. Apply DPO as post-SFT training stage using MLX
7. Expected impact: +3-5% accuracy (90% -> 93-95%)

**Phase 3 -- Next month (add LLM-as-a-Judge):**
8. Add LLM evaluation stage to `roachie-batch` (4-dimension rubric scored by Claude/Gemini)
9. Use LLM judge scores for training data quality filtering
10. Build regression detection pipeline

**Phase 4 -- Future (GRPO/online learning):**
11. Collect multi-response group data from batch tests passively
12. Implement GRPO when MLX support is available
13. Consider PPO with execution-as-reward when dataset reaches 1000+ examples

### Key Insight

Roachie has a **solid LoRA/QLoRA foundation** and the **infrastructure for preference learning** (feedback data, batch testing, multi-provider comparison) -- but doesn't connect these systems. The feedback data flows into prompt injection (weak learning) instead of model weight updates (strong learning). **DPO is the highest-leverage next step** because:
- The preference data already exists (batch successes vs failures)
- MLX supports DPO training
- It directly addresses the model's failure patterns (wrong flags, wrong tool)
- It requires no reward model, no RL infrastructure, and no additional human labeling

---

## CLI Summary (Quick Reference)

This section mirrors the summary provided in the Claude Code terminal for easy cross-referencing.

### Current State

| Technique | Roachie Score | Impact If Adopted | Effort | Priority |
|-----------|--------------|-------------------|--------|----------|
| 1. LoRA | **8/10** | Already primary method | Done | Improve data quality |
| 2. QLoRA | **7.5/10** | Already in use (4-bit base) | Done | Add 8-bit option |
| 3. Full Fine-Tuning | **N/A** | Low (data too small) | High | Not recommended |
| 4. DPO | **2/10** (potential: 8/10) | **High** | **Medium** | **#1 priority** |
| 5. PPO | **0/10** | Medium | High | Not recommended now |
| 6. GRPO | **0/10** | Medium | High | Collect data passively |
| 7. RLHF | **3/10** | Medium | High | Close loop via DPO |
| 8. LLM-as-a-Judge | **5/10** | **High** | **Low** | **#2 priority** |

### The Key Insight

Roachie has a solid LoRA/QLoRA foundation and the infrastructure for preference learning (feedback data, batch testing, multi-provider comparison) -- but **doesn't connect these systems**. The feedback data flows into prompt injection (weak learning) instead of model weight updates (strong learning). The infrastructure exists; it just needs to be wired together.

### Top 3 Improvements by Impact

1. **Implement DPO** -- The preference data already exists from batch testing (success = chosen, failure = rejected for the same prompt). MLX supports DPO. This directly addresses the model's failure modes (wrong flags, wrong tool selection) and requires no reward model, no RL infrastructure, and no additional human labeling. Expected impact: +3-5% accuracy (90% -> 93-95%).

2. **Add LLM-as-a-Judge to batch pipeline** -- After executing each prompt, send the (query, reasoning, command, output) tuple to Claude or Gemini with a 4-dimension rubric (tool selection, flag accuracy, reasoning quality, response helpfulness). This catches quality issues that pass/fail execution testing misses (e.g., correct command but poor reasoning, or working but suboptimal tool choice).

3. **Scale and improve LoRA training data** -- Three specific improvements to the existing pipeline:
   - Include system prompt in training examples (the model currently trains without the context it sees at inference)
   - Add negative examples (informational queries that should produce empty command lists)
   - Scale from ~75 to 300+ unique training examples for better generalization

### What NOT to Do Yet

- **Full Fine-Tuning**: With only ~75 training examples, full fine-tuning would catastrophically overfit. LoRA's parameter efficiency is the right choice.
- **PPO**: Requires RL infrastructure (value network, advantage estimation, clipping) that is disproportionate to the dataset size. DPO achieves similar results with far less complexity.
- **GRPO**: No MLX support yet. Collect group response data passively from batch testing for future use.
- **Reward Model Training**: DPO bypasses the need for an explicit reward model. Only invest in a reward model if/when the training set exceeds 1000+ examples.
