# LoRA Fine-Tuning Pipeline — Architecture & Process Flow

## 1. End-to-End Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ROACHIE LoRA FINE-TUNING PIPELINE                        │
│                    Llama 3.1 8B: 82% → 90% Accuracy                        │
└─────────────────────────────────────────────────────────────────────────────┘

  PHASE 1              PHASE 2              PHASE 3              PHASE 4
  DIAGNOSE             QUICK FIXES          FINE-TUNE             VALIDATE
  ─────────            ───────────          ──────────            ─────────
  ┌─────────┐         ┌──────────┐         ┌──────────┐         ┌──────────┐
  │ Analyze │         │ Unify    │         │ Extract  │         │ Batch    │
  │ Failure │────────▶│ Prompt   │────────▶│ Training │────────▶│ Test     │
  │ Patterns│         │ Pipeline │         │ Data     │         │ (73      │
  │         │         │          │         │          │         │ prompts) │
  └─────────┘         └──────────┘         └──────────┘         └──────────┘
   72-78%              +7% (→83%)           +7% (→90%)           90% ✓
   baseline            code fixes           model tuning         validated
```

---

## 2. Phase 1 — Root Cause Diagnosis

```
┌───────────────────────────────────────────────────────────────┐
│                   FAILURE ANALYSIS                            │
│                                                               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐       │
│  │ Gemini      │    │ Ollama      │    │ Compare     │       │
│  │ Batch Test  │    │ Batch Test  │    │ Side-by-    │       │
│  │ 95-100%     │    │ 72-78%      │    │ Side        │       │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘       │
│         │                  │                  │               │
│         └──────────┬───────┘                  │               │
│                    ▼                          │               │
│         ┌──────────────────┐                  │               │
│         │ 3 Root Causes    │◀─────────────────┘               │
│         │ Identified       │                                  │
│         └──────────────────┘                                  │
│                    │                                          │
│     ┌──────────────┼──────────────┐                           │
│     ▼              ▼              ▼                           │
│  ┌────────┐  ┌──────────┐  ┌──────────┐                      │
│  │Factor 1│  │ Factor 2 │  │ Factor 3 │                      │
│  │Prompt  │  │ Enrich-  │  │ Model    │                      │
│  │Gap     │  │ ment     │  │ Capacity │                      │
│  │        │  │ Throttle │  │ Ceiling  │                      │
│  │Ollama  │  │          │  │          │                      │
│  │skipped │  │3 tools   │  │8B params │                      │
│  │7 tmpl  │  │vs 7 for  │  │can't     │                      │
│  │files   │  │Gemini    │  │match     │                      │
│  │(6,470  │  │          │  │Gemini    │                      │
│  │tokens) │  │          │  │          │                      │
│  └────┬───┘  └────┬─────┘  └────┬─────┘                      │
│       │           │             │                             │
│       │  Fixable  │   Fixable   │  Addressable                │
│       │  via code │   via code  │  via fine-tuning             │
│       └─────┬─────┘             │                             │
│             ▼                   ▼                             │
│       Phase 2 (fixes)     Phase 3 (LoRA)                      │
└───────────────────────────────────────────────────────────────┘
```

---

## 3. Phase 2 — Quick Fixes (Code Changes Only)

```
┌───────────────────────────────────────────────────────────────┐
│                   4 CODE FIXES APPLIED                        │
│                                                               │
│  Fix #1: UNIFY PROMPT PATH                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ BEFORE                         AFTER                    │  │
│  │                                                         │  │
│  │ if ollama:                     # All providers:         │  │
│  │   compact_prompt()  ──╳        full_prompt()            │  │
│  │   (skips 7 files)              (all 8 templates)        │  │
│  │ else:                                                   │  │
│  │   full_prompt()                                         │  │
│  │                                                         │  │
│  │ File: bin/roachie-batch (line 639)                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  Fix #2: REMOVE ENRICHMENT THROTTLE                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ BEFORE                         AFTER                    │  │
│  │                                                         │  │
│  │ if ollama: max_tools=3         max_tools=7 (all)        │  │
│  │ else:      max_tools=7                                  │  │
│  │                                                         │  │
│  │ File: bin/roachie-batch (line 832)                      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  Fix #3: REMOVE MODELFILE SYSTEM PROMPT                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ BEFORE                         AFTER                    │  │
│  │                                                         │  │
│  │ SYSTEM """You are roachie...   (no SYSTEM block)        │  │
│  │ Available tools: cr_tables,    Runtime prompt provides   │  │
│  │ cr_size, ..."""                full context              │  │
│  │                                                         │  │
│  │ Files: tools/utils/Modelfile.roachie-{8b,3b,}           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  Fix #4: TUNE MODEL PARAMETERS                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ Parameter        BEFORE          AFTER                  │  │
│  │ ─────────        ──────          ─────                  │  │
│  │ num_predict      256             512                    │  │
│  │ temperature      0.1             0.15                   │  │
│  │ num_ctx          16384           16384 (unchanged)      │  │
│  │                                                         │  │
│  │ Files: tools/utils/Modelfile.roachie-{8b,3b,}           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  RESULT: 72% ──────────────────────▶ ~83%  (+11 points)       │
└───────────────────────────────────────────────────────────────┘
```

---

## 4. Phase 3 — LoRA Fine-Tuning Pipeline

### 4.1 Training Data Extraction

```
┌───────────────────────────────────────────────────────────────┐
│              TRAINING DATA PIPELINE                            │
│              prepare_training_data.py                          │
│                                                               │
│  ┌──────────────────────┐                                     │
│  │   sql_audit/          │                                     │
│  │   nl_batch_*.jsonl    │  156 files                          │
│  │                       │  3,105 successful Gemini records    │
│  └──────────┬────────────┘                                     │
│             │                                                  │
│             ▼                                                  │
│  ┌──────────────────────┐                                     │
│  │  Filter              │                                     │
│  │  provider == "gemini" │                                     │
│  │  status == "success"  │                                     │
│  │  commands != empty    │                                     │
│  └──────────┬────────────┘                                     │
│             │                                                  │
│             ▼                                                  │
│  ┌──────────────────────┐                                     │
│  │  Deduplicate          │                                     │
│  │  by prompt.lower()    │──▶  82 unique examples              │
│  │  keep latest          │                                     │
│  └──────────┬────────────┘                                     │
│             │                                                  │
│             ▼                                                  │
│  ┌──────────────────────┐                                     │
│  │  Clean Commands       │                                     │
│  │  Strip [OK:XXms]      │                                     │
│  │  Strip absolute paths │                                     │
│  │  Strip trailing ;     │                                     │
│  └──────────┬────────────┘                                     │
│             │                                                  │
│             ▼                                                  │
│  ┌──────────────────────┐                                     │
│  │  Format as Chat-ML    │                                     │
│  │  ┌──────────────────┐ │                                     │
│  │  │ {"messages": [   │ │                                     │
│  │  │   {"role":"user",│ │                                     │
│  │  │    "content":    │ │                                     │
│  │  │    "<prompt>"},  │ │                                     │
│  │  │   {"role":       │ │                                     │
│  │  │    "assistant",  │ │                                     │
│  │  │    "content":    │ │                                     │
│  │  │    "<json>"}     │ │                                     │
│  │  │ ]}               │ │                                     │
│  │  └──────────────────┘ │                                     │
│  └──────────┬────────────┘                                     │
│             │                                                  │
│             ▼                                                  │
│  ┌──────────────────────┐                                     │
│  │  Split (seed=42)     │                                     │
│  │                       │                                     │
│  │  train.jsonl  65 (80%)│                                     │
│  │  valid.jsonl   8 (10%)│                                     │
│  │  test.jsonl    9 (10%)│                                     │
│  └───────────────────────┘                                     │
└───────────────────────────────────────────────────────────────┘
```

### 4.2 Training Example Format

```
┌───────────────────────────────────────────────────────────────┐
│                   SINGLE TRAINING EXAMPLE                      │
│                                                               │
│  INPUT (user role):                                           │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ "show me the list of columns for the rides table in     │  │
│  │  the movr database on cluster a in tenant va"           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                          │                                    │
│                          ▼                                    │
│  TARGET (assistant role):                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ {                                                       │  │
│  │   "reasoning": "The user is asking for columns for a    │  │
│  │     specific table. Using cr_columns with --filter-db,  │  │
│  │     --filter-table, and -t for tenant.",                 │  │
│  │   "user_message": "I'll run the appropriate command.",  │  │
│  │   "commands": [{                                        │  │
│  │     "description": "Show columns for rides table",      │  │
│  │     "command": "cr_columns --filter-db movr              │  │
│  │       --filter-table rides -h localhost -p 26257         │  │
│  │       -t va --insecure"                                  │  │
│  │   }],                                                   │  │
│  │   "needs_followup": false                               │  │
│  │ }                                                       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  The model learns:                                            │
│  ✓ Correct tool selection (cr_columns, not cr_tables)         │
│  ✓ Correct flags (--filter-table, not -T)                     │
│  ✓ Correct topology (localhost:26257 = Cluster A)             │
│  ✓ Correct JSON response structure                            │
└───────────────────────────────────────────────────────────────┘
```

### 4.3 LoRA Training Process

```
┌───────────────────────────────────────────────────────────────┐
│                LoRA TRAINING (MLX on Apple Silicon)            │
│                                                               │
│  Base Model                                                   │
│  ┌─────────────────────────────────────────┐                  │
│  │  Meta-Llama-3.1-8B-Instruct-4bit       │                  │
│  │  8.03 billion parameters                │                  │
│  │  Downloaded from HuggingFace            │                  │
│  │  (~4.5 GB, Q4 quantized)               │                  │
│  └──────────────────┬──────────────────────┘                  │
│                     │                                         │
│                     ▼                                         │
│  LoRA Configuration                                           │
│  ┌─────────────────────────────────────────┐                  │
│  │  Trainable params:  10.49M  (0.131%)   │                  │
│  │  Frozen params:     8,019M  (99.87%)   │                  │
│  │  LoRA rank:         8                   │                  │
│  │  LoRA layers:       16                  │                  │
│  │  Learning rate:     1e-5                │                  │
│  │  Batch size:        1                   │                  │
│  │  Iterations:        1,000               │                  │
│  └──────────────────┬──────────────────────┘                  │
│                     │                                         │
│                     ▼                                         │
│  Training Loop (33 minutes, 6.87 GB peak RAM)                 │
│  ┌─────────────────────────────────────────┐                  │
│  │                                         │                  │
│  │  Iter   Train Loss   Val Loss   Status  │                  │
│  │  ────   ──────────   ────────   ──────  │                  │
│  │     1       —         3.309    baseline  │                  │
│  │   100     0.450       0.386              │                  │
│  │   200     0.210       0.376              │                  │
│  │   300     0.085       0.873    overfit ↑ │                  │
│  │   400     0.055       0.751              │                  │
│  │   500     0.038       1.070    overfit ↑ │                  │
│  │   600     0.035       0.332  ★ BEST ★   │                  │
│  │   700     0.029       0.787    overfit ↑ │                  │
│  │   800     0.030       0.918    overfit ↑ │                  │
│  │   900     0.028       0.526              │                  │
│  │  1000     0.029       1.320    overfit ↑ │                  │
│  │                                         │                  │
│  └──────────────────┬──────────────────────┘                  │
│                     │                                         │
│            Select iter 600                                    │
│            (lowest val loss)                                  │
│                     │                                         │
│                     ▼                                         │
│  ┌─────────────────────────────────────────┐                  │
│  │  adapters/0000600_adapters.safetensors  │                  │
│  │  (LoRA weight deltas, ~42 MB)           │                  │
│  └─────────────────────────────────────────┘                  │
└───────────────────────────────────────────────────────────────┘
```

### 4.4 Model Conversion & Import Pipeline

```
┌───────────────────────────────────────────────────────────────┐
│            MODEL CONVERSION → OLLAMA IMPORT                    │
│                                                               │
│  Step 1: FUSE                                                 │
│  ┌───────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ Base Model    │    │ LoRA         │    │ Fused Model  │   │
│  │ (4-bit, 4.5GB)│ +  │ Adapters     │ =  │ (4-bit,4.2GB)│   │
│  │               │    │ (42 MB)      │    │              │   │
│  └───────────────┘    └──────────────┘    └──────┬───────┘   │
│                                                  │            │
│  Step 2: DE-QUANTIZE                             ▼            │
│  ┌───────────────────────────────────────────────────────┐    │
│  │ Ollama cannot import MLX 4-bit (U32 packed weights)   │    │
│  │                                                       │    │
│  │ mlx_lm.convert(dequantize=True)                       │    │
│  │                                                       │    │
│  │ 4-bit (4.2 GB) ──────▶ bfloat16 (15 GB)              │    │
│  └──────────────────────────────┬────────────────────────┘    │
│                                 │                             │
│  Step 3: FIX CHAT TEMPLATE      ▼                             │
│  ┌───────────────────────────────────────────────────────┐    │
│  │ MLX de-quantize strips chat_template from             │    │
│  │ tokenizer_config.json                                 │    │
│  │                                                       │    │
│  │ Fix: Copy chat_template from base model's             │    │
│  │      tokenizer_config.json (4,614 chars)              │    │
│  │                                                       │    │
│  │ Without this: Ollama processes 11 tokens instead      │    │
│  │ of full prompt → model outputs "{  }"                 │    │
│  └──────────────────────────────┬────────────────────────┘    │
│                                 │                             │
│  Step 4: OLLAMA IMPORT          ▼                             │
│  ┌───────────────────────────────────────────────────────┐    │
│  │                                                       │    │
│  │ Modelfile:                                            │    │
│  │ ┌─────────────────────────────────────────────────┐   │    │
│  │ │ FROM ./roachie-8b-finetuned-fp16                │   │    │
│  │ │                                                 │   │    │
│  │ │ PARAMETER num_ctx 16384                         │   │    │
│  │ │ PARAMETER temperature 0.15                      │   │    │
│  │ │ PARAMETER num_predict 512                       │   │    │
│  │ │ PARAMETER stop <|eot_id|>                       │   │    │
│  │ └─────────────────────────────────────────────────┘   │    │
│  │                                                       │    │
│  │ $ ollama create roachie-8b-ft -f Modelfile.finetuned  │    │
│  │                                                       │    │
│  │ ✓ Auto-detects llama3-instruct template               │    │
│  │ ✓ Converts safetensors → GGUF internally              │    │
│  │ ✓ Registers as "roachie-8b-ft" (16 GB in Ollama)      │    │
│  └───────────────────────────────────────────────────────┘    │
│                                                               │
│  Step 5: CLEANUP                                              │
│  ┌───────────────────────────────────────────────────────┐    │
│  │ Remove intermediate files:                            │    │
│  │   roachie-8b-finetuned/      (4.2 GB)  ── deleted     │    │
│  │   roachie-8b-finetuned-fp16/ (15 GB)   ── deleted     │    │
│  │   Non-best adapter checkpoints          ── deleted     │    │
│  │                                                       │    │
│  │ Keep:                                                 │    │
│  │   adapters/adapters.safetensors (best checkpoint)     │    │
│  │   data/{train,valid,test}.jsonl (training data)       │    │
│  └───────────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────────┘
```

---

## 5. Phase 4 — Validation

```
┌───────────────────────────────────────────────────────────────┐
│                   BATCH TEST VALIDATION                        │
│                                                               │
│  $ bash bin/roachie-batch \                                    │
│      --file examples/queries/example-queries.txt \             │
│      --provider ollama-roachie-8b-ft \                         │
│      --tools-version 25.4 \                                    │
│      --no-feedback --no-followup --allow-sql-files             │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                73 PROMPTS TESTED                        │  │
│  │                                                         │  │
│  │  ████████████████████████████████████████████░░░░░░░░   │  │
│  │  ◀──────── 66 SUCCESS (90%) ────────▶ ◀─ 5 FAIL ─▶    │  │
│  │                                         2 SKIP          │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ACCURACY PROGRESSION                                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                                                         │  │
│  │  100% ┤                                    ●●● Gemini   │  │
│  │   95% ┤                                ●●●             │  │
│  │   90% ┤                            ★ roachie-8b-ft     │  │
│  │   85% ┤                                                │  │
│  │   83% ┤                     ● after unified prompt      │  │
│  │   80% ┤                                                │  │
│  │   78% ┤             ● best pre-fix                      │  │
│  │   75% ┤                                                │  │
│  │   72% ┤      ● compact prompt era                       │  │
│  │   70% ┤                                                │  │
│  │       └───────┬────────┬────────┬────────┬──────────    │  │
│  │           Mar 15   Mar 21   Mar 21   Mar 22             │  │
│  │          baseline  unified   fixes   LoRA               │  │
│  │                    prompt    #1-#4   fine-tune           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  REMAINING 5 FAILURES                                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  #8  cr_backup hallucinated S3 URL (should use          │  │
│  │      cr_ceph_storage backup)                            │  │
│  │  #57 cr_ddl_table missing positional table name         │  │
│  │  #61 cr_ddl_database missing -d flag                    │  │
│  │  #68 cr_migrate truncated flags                         │  │
│  │  #70 cr_ddl_table missing positional table name         │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

## 6. Runtime Integration

```
┌───────────────────────────────────────────────────────────────┐
│           HOW THE FINE-TUNED MODEL SERVES QUERIES              │
│                                                               │
│  User: "show columns for rides in movr on cluster A"          │
│                    │                                          │
│                    ▼                                          │
│  ┌──────────────────────────────┐                             │
│  │ 1. Build System Prompt       │                             │
│  │    _nl_build_system_prompt() │  Same prompt for ALL        │
│  │    8 template files loaded   │  providers (unified)        │
│  │    ~5,500 tokens base        │                             │
│  └──────────────┬───────────────┘                             │
│                 │                                             │
│                 ▼                                             │
│  ┌──────────────────────────────┐                             │
│  │ 2. Enrich with Tool Help     │                             │
│  │    Regex + Semantic matching  │  7 tools max               │
│  │    Fetch --help for matches  │  (same as Gemini)           │
│  │    ~4,000 tokens added       │                             │
│  └──────────────┬───────────────┘                             │
│                 │                                             │
│                 ▼                                             │
│  ┌──────────────────────────────┐                             │
│  │ 3. Call Ollama API            │                             │
│  │    model: roachie-8b-ft      │  ← Fine-tuned model        │
│  │    Provider: ollama-*        │                             │
│  │    Same code path as base    │  No code changes needed     │
│  │    ${provider#ollama-}       │  → "roachie-8b-ft"          │
│  └──────────────┬───────────────┘                             │
│                 │                                             │
│                 ▼                                             │
│  ┌──────────────────────────────┐                             │
│  │ 4. Validate & Execute        │                             │
│  │    Whitelist → Metachar →    │                             │
│  │    SQL guard → RBAC →        │  Same 5-layer validation    │
│  │    Tool existence            │                             │
│  └──────────────┬───────────────┘                             │
│                 │                                             │
│                 ▼                                             │
│  ┌──────────────────────────────┐                             │
│  │ 5. Result                     │                             │
│  │    cr_columns --filter-db     │                             │
│  │    movr --filter-table rides  │  ✓ Correct flags           │
│  │    -h localhost -p 26257      │  ✓ Correct topology        │
│  │    -t va --insecure           │  ✓ Correct tenant          │
│  └──────────────────────────────┘                             │
└───────────────────────────────────────────────────────────────┘
```

---

## 7. File Inventory

```
┌───────────────────────────────────────────────────────────────┐
│                   FILES CREATED / MODIFIED                     │
│                                                               │
│  NEW FILES                                                    │
│  ─────────                                                    │
│  tools/utils/finetune/                                        │
│  ├── prepare_training_data.py   Extract Gemini → training set │
│  ├── run_lora.sh                End-to-end pipeline script    │
│  ├── .gitignore                 Exclude large model files     │
│  ├── data/                                                    │
│  │   ├── train.jsonl            65 training examples          │
│  │   ├── valid.jsonl             8 validation examples        │
│  │   └── test.jsonl              9 test examples              │
│  └── adapters/                                                │
│      └── adapters.safetensors   Best LoRA weights (iter 600)  │
│                                                               │
│  MODIFIED FILES                                               │
│  ──────────────                                               │
│  bin/roachie-batch              Remove Ollama-specific paths   │
│  tools/utils/Modelfile.roachie-8b   No SYSTEM, new params     │
│  tools/utils/Modelfile.roachie      No SYSTEM, new params     │
│  tools/utils/Modelfile.roachie-3b   No SYSTEM, new params     │
│                                                               │
│  OLLAMA MODELS                                                │
│  ─────────────                                                │
│  roachie-8b       4.9 GB   Base (rebuilt with new params)     │
│  roachie-8b-ft   16.0 GB   Fine-tuned (FP16, LoRA fused)     │
│  roachie          4.7 GB   Base llama3 (rebuilt)              │
│  roachie-3b       2.0 GB   Base llama3.2:3b (rebuilt)         │
└───────────────────────────────────────────────────────────────┘
```

---

## 8. Reproduction Steps

```
# Step 1: Prepare training data from Gemini batch results
python3 tools/utils/finetune/prepare_training_data.py

# Step 2: Run LoRA fine-tuning + Ollama import
bash tools/utils/finetune/run_lora.sh

# Step 3: Validate
bash bin/roachie-batch \
  --file examples/queries/example-queries.txt \
  --provider ollama-roachie-8b-ft \
  --tools-version 25.4 \
  --no-feedback --no-followup --allow-sql-files
```
