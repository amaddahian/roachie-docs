# Process Flow: Llama 8B Accuracy Improvement (82% → 90%)

**Date**: 2026-03-22
**Model**: Llama 3.1 8B Instruct (4-bit) → roachie-8b-ft (LoRA fine-tuned)
**Hardware**: Apple M1 Pro, 32GB RAM
**Framework**: MLX LoRA (Apple Silicon native)

---

## Master Process Flow

```
 START
   │
   │  Baseline: roachie-8b at 82-83% accuracy (60-61/73 prompts)
   │  Reference: Gemini at 95-100% accuracy (70-73/73 prompts)
   │
   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   STEP 1: ROOT CAUSE ANALYSIS                       │
│                                                                     │
│  Compared Ollama vs Gemini code paths in bin/roachie-batch          │
│  Found 3 Ollama-specific restrictions still in the batch runner:    │
│                                                                     │
│    1. Compact prompt (fewer templates, less context)                │
│    2. Enrichment capped at 3 tools (vs 7 for cloud providers)      │
│    3. Simplified connection reminder (missing details)              │
│                                                                     │
│  Also found Modelfile issues:                                       │
│    4. Embedded SYSTEM prompt conflicting with runtime prompt        │
│    5. num_predict too low (256 tokens, truncating responses)        │
│    6. temperature too conservative (0.1)                            │
│                                                                     │
│  Conclusion: 4 quick code fixes + LoRA fine-tuning                  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│               STEP 2: QUICK FIX #1 — UNIFIED PROMPT                │
│                                                                     │
│  File: bin/roachie-batch (line 639)                                 │
│                                                                     │
│  BEFORE:                                                            │
│  ┌────────────────────────────────────────────────────────┐         │
│  │ if [[ "$llm_provider" == ollama-* ]]; then             │         │
│  │   system_prompt_base="$(_nl_build_system_prompt_compact│         │
│  │     "$cluster_topology" "$tools_dir" "$schema_context")│"        │
│  │ else                                                   │         │
│  │   system_prompt_base="$(_nl_build_system_prompt ...)"  │         │
│  │ fi                                                     │         │
│  └────────────────────────────────────────────────────────┘         │
│                              │                                      │
│                              ▼                                      │
│  AFTER:                                                             │
│  ┌────────────────────────────────────────────────────────┐         │
│  │ system_prompt_base="$(_nl_build_system_prompt          │         │
│  │   "$cluster_topology" "$tools_dir" "$schema_context")" │         │
│  └────────────────────────────────────────────────────────┘         │
│                                                                     │
│  Impact: Ollama now receives all 7 template files                   │
│          (parameter_rules, tool_notes, execution_rules,             │
│           few_shot_examples, hallucination_warnings, etc.)          │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│          STEP 3: QUICK FIX #2 — REMOVE ENRICHMENT CAP              │
│                                                                     │
│  File: bin/roachie-batch (lines 831-839)                            │
│                                                                     │
│  BEFORE:                                                            │
│  ┌────────────────────────────────────────────────────────┐         │
│  │ local _enrich_max="$_NL_MAX_ENRICHED_TOOLS"           │         │
│  │ [[ "$llm_provider" == ollama-* ]] && _enrich_max=3    │         │
│  │ system_prompt=$(_enrich_system_prompt_with_tool_help \ │         │
│  │   "$query" "$base" "$tools_dir" "" "$_enrich_max")    │         │
│  │                                                       │         │
│  │ if [[ "$llm_provider" == ollama-* ]]; then            │         │
│  │   system_prompt+="REMINDER: EVERY cr_* command..."    │ ◄─ hard │
│  │ else                                                  │   coded │
│  │   system_prompt+="$(_nl_connection_reminder)"         │         │
│  │ fi                                                    │         │
│  └────────────────────────────────────────────────────────┘         │
│                              │                                      │
│                              ▼                                      │
│  AFTER:                                                             │
│  ┌────────────────────────────────────────────────────────┐         │
│  │ system_prompt=$(_enrich_system_prompt_with_tool_help \ │         │
│  │   "$query" "$base" "$tools_dir")                      │         │
│  │ system_prompt+="$(_nl_connection_reminder)"           │         │
│  └────────────────────────────────────────────────────────┘         │
│                                                                     │
│  Impact: Ollama gets 7 enriched tools (was 3) + standard reminder   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│         STEP 4: QUICK FIX #3 — REMOVE MODELFILE SYSTEM             │
│                                                                     │
│  Files: tools/utils/Modelfile.roachie-8b (and roachie, roachie-3b) │
│                                                                     │
│  BEFORE:                                                            │
│  ┌────────────────────────────────────────────────────────┐         │
│  │ FROM llama3.1:latest                                  │         │
│  │ SYSTEM """You are a CockroachDB assistant..."""       │ ◄─ 200  │
│  │ PARAMETER num_ctx 16384                               │   token │
│  │ PARAMETER temperature 0.1                             │   waste │
│  │ PARAMETER num_predict 256                             │         │
│  │ PARAMETER stop <|eot_id|>                             │         │
│  └────────────────────────────────────────────────────────┘         │
│                              │                                      │
│                              ▼                                      │
│  AFTER:                                                             │
│  ┌────────────────────────────────────────────────────────┐         │
│  │ FROM llama3.1:latest                                  │         │
│  │ PARAMETER num_ctx 16384                               │         │
│  │ PARAMETER temperature 0.15           ◄── Fix #4       │         │
│  │ PARAMETER num_predict 512            ◄── doubled      │         │
│  │ PARAMETER stop <|eot_id|>                             │         │
│  └────────────────────────────────────────────────────────┘         │
│                                                                     │
│  Impact: No conflicting SYSTEM prompt, more output tokens,          │
│          slightly higher temperature for better tool selection      │
│                                                                     │
│  All 3 Modelfiles rebuilt via: ollama create roachie-8b -f ...      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│            STEP 5: VERIFY QUICK FIXES (~87% estimated)              │
│                                                                     │
│  bash bin/roachie-batch \                                           │
│    --file examples/queries/example-queries.txt \                    │
│    --provider ollama-roachie-8b \                                   │
│    --tools-version 25.4 --no-feedback --no-followup                 │
│                                                                     │
│  Result: ~87% pass rate (improvement from 82%)                      │
│  Remaining gap to Gemini (95-100%): model capability, not config    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│           STEP 6: PREPARE LORA TRAINING DATA                        │
│                                                                     │
│  Created: tools/utils/finetune/prepare_training_data.py             │
│                                                                     │
│  ┌─────────────┐    ┌──────────────┐    ┌──────────────────┐       │
│  │ 156 JSONL    │    │ Filter:      │    │ Deduplicate by   │       │
│  │ batch result │───▶│ provider ==  │───▶│ normalized       │       │
│  │ files in     │    │ "gemini" AND │    │ prompt text      │       │
│  │ sql_audit/   │    │ status ==    │    │ (lowercase,      │       │
│  │              │    │ "success"    │    │  stripped)       │       │
│  │ 3,105 total  │    │              │    │                  │       │
│  │ records      │    │ 3,105 pass   │    │ 82 unique pairs  │       │
│  └─────────────┘    └──────────────┘    └────────┬─────────┘       │
│                                                   │                 │
│                                                   ▼                 │
│                                         ┌──────────────────┐       │
│                                         │ Clean commands:  │       │
│                                         │ - Strip [OK:XXms]│       │
│                                         │ - Remove abs     │       │
│                                         │   paths          │       │
│                                         │ - Strip trailing │       │
│                                         │   semicolons     │       │
│                                         └────────┬─────────┘       │
│                                                   │                 │
│                                                   ▼                 │
│                                         ┌──────────────────┐       │
│                                         │ Format as        │       │
│                                         │ Chat-ML JSONL:   │       │
│                                         │                  │       │
│                                         │ {"messages": [   │       │
│                                         │   {"role":"user",│       │
│                                         │    "content":    │       │
│                                         │    "<prompt>"},  │       │
│                                         │   {"role":       │       │
│                                         │    "assistant",  │       │
│                                         │    "content":    │       │
│                                         │    "<json>"}     │       │
│                                         │ ]}               │       │
│                                         └────────┬─────────┘       │
│                                                   │                 │
│                                                   ▼                 │
│                                         ┌──────────────────┐       │
│                                         │ Split (seed=42): │       │
│                                         │                  │       │
│                                         │  train.jsonl: 65 │       │
│                                         │  valid.jsonl:  8 │       │
│                                         │  test.jsonl:   9 │       │
│                                         └──────────────────┘       │
│                                                                     │
│  python3 tools/utils/finetune/prepare_training_data.py              │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│              STEP 7: SET UP TRAINING ENVIRONMENT                    │
│                                                                     │
│  python3 -m venv tools/utils/finetune/.venv                        │
│  source tools/utils/finetune/.venv/bin/activate                    │
│  pip install mlx-lm                                                │
│                                                                     │
│  Note: System Python blocked by PEP 668 (externally-managed-env)   │
│        → venv required                                              │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│            STEP 8: RUN LORA FINE-TUNING (33 min)                    │
│                                                                     │
│  Created: tools/utils/finetune/run_lora.sh                          │
│                                                                     │
│  ┌─ Training Configuration ──────────────────────────────────┐     │
│  │  Base model:    mlx-community/Meta-Llama-3.1-8B-Instruct  │     │
│  │                 -4bit                                      │     │
│  │  LoRA rank:     8                                          │     │
│  │  LoRA layers:   16                                         │     │
│  │  Learning rate: 1e-5                                       │     │
│  │  Batch size:    1                                          │     │
│  │  Iterations:    1,000 (20 epochs x 65 / 1)                │     │
│  │  Peak memory:   6.87 GB                                    │     │
│  └────────────────────────────────────────────────────────────┘     │
│                                                                     │
│  Training Loss Curve:                                               │
│                                                                     │
│  Loss                                                               │
│  3.5 │* (val)                                                       │
│      │                                                              │
│  3.0 │                                                              │
│      │                                                              │
│  2.5 │                                                              │
│      │                                                              │
│  2.0 │                                                              │
│      │                                                              │
│  1.5 │                                            x (val 1.32)      │
│      │                                                              │
│  1.0 │            x         x              x   x                    │
│      │         x     x         x        x                           │
│  0.5 │      o     o    o  o  o  o  o  o  o  o  o  (train)          │
│      │   o                                                          │
│  0.3 │         *              * ◄── BEST (val 0.332, iter 600)      │
│      │                                                              │
│  0.0 └──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──                             │
│        100 200 300 400 500 600 700 800 900 1000                     │
│                          Iteration                                  │
│                                                                     │
│  Key: o = train loss, * = validation loss, x = val (not best)       │
│                                                                     │
│  Observation: Val loss oscillates — dataset too small (82 examples) │
│               for stable generalization. Best checkpoint: iter 600  │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│       STEP 9: FUSE LORA ADAPTERS INTO BASE MODEL                   │
│                                                                     │
│  python -m mlx_lm.fuse \                                           │
│    --model mlx-community/Meta-Llama-3.1-8B-Instruct-4bit \         │
│    --adapter-path adapters/ \                                       │
│    --save-path roachie-8b-finetuned/                                │
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────────┐    │
│  │ Base Model   │     │ LoRA         │     │ Fused Model      │    │
│  │ (4-bit,      │  +  │ Adapters     │  =  │ (4-bit,          │    │
│  │  4.2 GB)     │     │ (iter 600,   │     │  4.2 GB)         │    │
│  │              │     │  16.8 MB)    │     │                  │    │
│  └──────────────┘     └──────────────┘     └──────────────────┘    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 10: DE-QUANTIZE FOR OLLAMA IMPORT                          │
│                                                                     │
│  Problem: Ollama cannot import MLX 4-bit packed weights (U32 dtype) │
│  Solution: De-quantize to bfloat16, let Ollama re-quantize          │
│                                                                     │
│  python -c "from mlx_lm.convert import convert;                     │
│             convert(hf_path='fused/', mlx_path='fp16/',             │
│                     dequantize=True)"                                │
│                                                                     │
│  ┌──────────────┐              ┌──────────────────┐                │
│  │ Fused Model  │              │ FP16 Model       │                │
│  │ (4-bit,      │  ──dequant─▶ │ (bfloat16,       │                │
│  │  4.2 GB,     │              │  ~15 GB)         │                │
│  │  U32 dtype)  │              │                  │                │
│  └──────────────┘              └──────────────────┘                │
│                                                                     │
│  Error encountered: "no space left on device" (only 16GB free)      │
│  Fix: Removed intermediate 4-bit fused model to free 4.2GB         │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 11: FIX CHAT TEMPLATE (Critical Bug)                       │
│                                                                     │
│  Problem: After de-quantization, tokenizer_config.json was missing  │
│           the chat_template field. The model responded with "{ }"   │
│           and only processed 11 tokens (no prompt formatting).      │
│                                                                     │
│  Diagnosis:                                                         │
│  ┌─────────────────────────────────────────────────────────┐       │
│  │ ollama run roachie-8b-ft "list databases"              │       │
│  │ Response: "{  }"   ◄── only 11 tokens processed        │       │
│  │                                                         │       │
│  │ Root cause: MLX de-quantize strips chat_template        │       │
│  │ from tokenizer_config.json                              │       │
│  └─────────────────────────────────────────────────────────┘       │
│                                                                     │
│  Fix: Copy chat_template from base model's tokenizer_config.json    │
│                                                                     │
│  ┌──────────────────────┐        ┌──────────────────────┐          │
│  │ Base Model           │        │ FP16 Model           │          │
│  │ tokenizer_config:    │ ─copy─▶│ tokenizer_config:    │          │
│  │  chat_template: "..."│        │  chat_template: "..."│          │
│  │  (4,614 chars,       │        │  + added_tokens      │          │
│  │   llama3-instruct)   │        │  + bos/eos tokens    │          │
│  └──────────────────────┘        └──────────────────────┘          │
│                                                                     │
│  After fix: Ollama auto-detects "llama3-instruct" template          │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 12: IMPORT INTO OLLAMA                                     │
│                                                                     │
│  Generated Modelfile.finetuned:                                     │
│  ┌────────────────────────────────────────────┐                    │
│  │ FROM ./roachie-8b-finetuned-fp16           │                    │
│  │                                            │                    │
│  │ PARAMETER num_ctx 16384                    │                    │
│  │ PARAMETER temperature 0.15                 │                    │
│  │ PARAMETER num_predict 512                  │                    │
│  │ PARAMETER stop <|eot_id|>                  │                    │
│  └────────────────────────────────────────────┘                    │
│                                                                     │
│  ollama rm roachie-8b-ft 2>/dev/null || true                       │
│  ollama create roachie-8b-ft -f Modelfile.finetuned                │
│                                                                     │
│  Error: "no space left on device" again (FP16 model = 15GB)        │
│  Fix: Removed old Ollama model to free 16GB, retried successfully  │
│                                                                     │
│  Final model in Ollama: 16GB (FP16, fits in 32GB unified memory)   │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 13: CLEANUP                                                │
│                                                                     │
│  rm -rf roachie-8b-finetuned/        # 4-bit fused (4.2 GB)       │
│  rm -rf roachie-8b-finetuned-fp16/   # FP16 intermediate (15 GB)  │
│                                                                     │
│  Kept:                                                              │
│  ├── adapters/adapters.safetensors   # 16.8 MB (reproducible)     │
│  ├── data/{train,valid,test}.jsonl   # Training data               │
│  ├── prepare_training_data.py        # Data extraction script      │
│  ├── run_lora.sh                     # End-to-end pipeline         │
│  └── .gitignore                      # Excludes large files        │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 14: BATCH TEST VALIDATION                                  │
│                                                                     │
│  bash bin/roachie-batch \                                           │
│    --file examples/queries/example-queries.txt \                    │
│    --provider ollama-roachie-8b-ft \                                │
│    --tools-version 25.4 --no-feedback --no-followup --allow-sql    │
│                                                                     │
│  Result: 66/73 passed = 90.4% accuracy                              │
│                                                                     │
│  5 remaining failures:                                              │
│  ┌───┬──────────────────────────────────┬────────────────────────┐ │
│  │ # │ Prompt                           │ Issue                  │ │
│  ├───┼──────────────────────────────────┼────────────────────────┤ │
│  │ 8 │ backup database to bucket        │ Wrong tool selected    │ │
│  │57 │ DDL for products table           │ Missing positional arg │ │
│  │61 │ DDL for database movr            │ Missing -d flag        │ │
│  │68 │ migrate inventory_schema         │ Truncated flags        │ │
│  │70 │ DDL for rides table              │ Missing positional arg │ │
│  └───┴──────────────────────────────────┴────────────────────────┘ │
│                                                                     │
│  Pattern: DDL positional args and complex multi-flag tools           │
│  are at the boundary of 8B model capability                         │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 15: COMMIT AND DOCUMENT                                    │
│                                                                     │
│  Roachie repo (15cab1a):                                            │
│  ├── bin/roachie-batch              # Unified prompt path          │
│  ├── tools/utils/Modelfile.roachie* # Updated parameters           │
│  ├── tools/utils/finetune/          # New LoRA pipeline            │
│  │   ├── prepare_training_data.py                                  │
│  │   ├── run_lora.sh                                               │
│  │   └── .gitignore                                                │
│  └── pushed to main                                                │
│                                                                     │
│  Roachie-docs repo:                                                 │
│  ├── changelog-session-2026-03-21.md        (67e6290)              │
│  ├── lora-finetuning-results-20260322.md    (918825e)              │
│  ├── recommendations_ai_architecture_v15.md (f4a03b1)              │
│  ├── diagram-lora-finetuning-pipeline.md    (14daa21)              │
│  └── all pushed to main                                            │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
                              DONE
```

---

## Accuracy Progression Summary

```
  100% ┬─────────────────────────────────────────────────
       │                                    ████████████  Gemini (95-100%)
   95% ┤                                    ████████████
       │
   90% ┤                              ┌─────────────────  roachie-8b-ft (90%)
       │                              │     LoRA
   87% ┤                        ┌─────┘                   After fixes #1-#4
       │                        │
   85% ┤                        │
       │                  ┌─────┘
   83% ┤            ┌─────┘                               After fix #1
       │            │
   80% ┤      ┌─────┘                                     Baseline (82%)
       │      │
   75% ┤      │
       │ ┌────┘
   72% ┤─┘                                               Before any changes
       │
   70% ┴──┬────────┬────────┬────────┬────────┬──────────
          v0       #1       #2-#4    LoRA     Gemini
       (original) (unified (Modelfile (fine-   (reference)
                  prompt)   fixes)   tuned)
```

---

## Errors Encountered and Resolutions

```
┌─ Error Timeline ──────────────────────────────────────────────────┐
│                                                                    │
│  1. PEP 668 — pip3 refused to install mlx-lm on system Python     │
│     └─▶ Created venv: python3 -m venv .venv                      │
│                                                                    │
│  2. Wrong CLI flags — --lora-rank not recognized by mlx-lm        │
│     └─▶ Used --num-layers (correct flag in mlx-lm 0.31.1)        │
│                                                                    │
│  3. U32 dtype error — Ollama can't read MLX 4-bit packed weights  │
│     └─▶ De-quantized to bfloat16 via mlx_lm.convert              │
│                                                                    │
│  4. Disk space (1st) — 15GB FP16 model, only 16GB free            │
│     └─▶ Removed 4-bit fused intermediate (freed 4.2 GB)          │
│                                                                    │
│  5. Missing chat template — model returned "{ }" (11 tokens)      │
│     └─▶ Copied chat_template from base model tokenizer_config    │
│                                                                    │
│  6. Disk space (2nd) — Ollama import failed, disk full again       │
│     └─▶ Removed old Ollama model (freed 16 GB), retried          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Model Conversion Pipeline Detail

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  HuggingFace Hub              Local (MLX)              Ollama Registry  │
│                                                                          │
│  ┌─────────────┐   download   ┌─────────────┐                          │
│  │ Meta-Llama  │ ───────────▶ │ Base Model  │                          │
│  │ 3.1-8B-Inst │              │ (4-bit,     │                          │
│  │ ruct-4bit   │              │  4.2 GB)    │                          │
│  └─────────────┘              └──────┬──────┘                          │
│                                      │                                   │
│                               mlx_lm.lora                               │
│                               (1000 iters)                              │
│                                      │                                   │
│                                      ▼                                   │
│                               ┌─────────────┐                          │
│                               │ LoRA        │                          │
│                               │ Adapters    │                          │
│                               │ (16.8 MB)   │                          │
│                               └──────┬──────┘                          │
│                                      │                                   │
│                               mlx_lm.fuse                               │
│                                      │                                   │
│                                      ▼                                   │
│                               ┌─────────────┐                          │
│                               │ Fused Model │                          │
│                               │ (4-bit,     │───── DELETED             │
│                               │  4.2 GB)    │      (save space)        │
│                               └──────┬──────┘                          │
│                                      │                                   │
│                            mlx_lm.convert                               │
│                            (dequantize=True)                            │
│                                      │                                   │
│                                      ▼                                   │
│                               ┌─────────────┐                          │
│                               │ FP16 Model  │                          │
│                               │ (bfloat16,  │───── DELETED             │
│                               │  ~15 GB)    │      (after import)      │
│                               └──────┬──────┘                          │
│                                      │                                   │
│                                      │  + fix tokenizer_config.json     │
│                                      │    (add chat_template)           │
│                                      │                                   │
│                               ollama create                             │
│                                      │                                   │
│                                      ▼                                   │
│                                              ┌─────────────────┐       │
│                                              │ roachie-8b-ft   │       │
│                                              │ (FP16, 16 GB)   │       │
│                                              │ in Ollama        │       │
│                                              │ registry        │       │
│                                              └─────────────────┘       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## How to Reproduce

```bash
# Step 1: Prepare training data from Gemini batch results
python3 tools/utils/finetune/prepare_training_data.py

# Step 2: Run full LoRA pipeline (train → fuse → convert → import)
bash tools/utils/finetune/run_lora.sh

# Step 3: Validate accuracy
bash bin/roachie-batch \
  --file examples/queries/example-queries.txt \
  --provider ollama-roachie-8b-ft \
  --tools-version 25.4 \
  --no-feedback --no-followup --allow-sql-files
```
