# Process Flow: Llama 8B Accuracy Improvement (82% → 90%) — ASCII

This is the original ASCII art version for reference and comparison with other formats.

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
│  │   system_prompt+="REMINDER: EVERY cr_* command..."    │         │
│  │ else                                                  │         │
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
│         STEP 4: QUICK FIX #3 & #4 — MODELFILE UPDATES              │
│                                                                     │
│  Files: tools/utils/Modelfile.roachie-8b (and roachie, roachie-3b) │
│                                                                     │
│  BEFORE:                               AFTER:                       │
│  ┌──────────────────────────┐          ┌──────────────────────────┐ │
│  │ FROM llama3.1:latest     │          │ FROM llama3.1:latest     │ │
│  │ SYSTEM """You are a      │          │ PARAMETER num_ctx 16384  │ │
│  │   CockroachDB..."""      │          │ PARAMETER temperature    │ │
│  │ PARAMETER num_ctx 16384  │    ──▶   │   0.15                   │ │
│  │ PARAMETER temperature    │          │ PARAMETER num_predict    │ │
│  │   0.1                    │          │   512                    │ │
│  │ PARAMETER num_predict    │          │ PARAMETER stop           │ │
│  │   256                    │          │   <|eot_id|>             │ │
│  │ PARAMETER stop           │          └──────────────────────────┘ │
│  │   <|eot_id|>             │                                       │
│  └──────────────────────────┘                                       │
│                                                                     │
│  Impact: No SYSTEM conflict, more output tokens, better temp        │
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
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│           STEP 6: PREPARE LORA TRAINING DATA                        │
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
│                                         │ Chat-ML JSONL    │       │
│                                         └────────┬─────────┘       │
│                                                   │                 │
│                                                   ▼                 │
│                                         ┌──────────────────┐       │
│                                         │ Split (seed=42): │       │
│                                         │  train.jsonl: 65 │       │
│                                         │  valid.jsonl:  8 │       │
│                                         │  test.jsonl:   9 │       │
│                                         └──────────────────┘       │
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
│  Note: System Python blocked by PEP 668 → venv required            │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│            STEP 8: RUN LORA FINE-TUNING (33 min)                    │
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
│  Best checkpoint: iteration 600 (val loss 0.332)                    │
│  Observation: Val loss oscillates — dataset too small (82 examples) │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│       STEPS 9-12: MODEL CONVERSION PIPELINE                        │
│                                                                     │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────────┐   │
│  │ Step 9   │   │ Step 10  │   │ Step 11  │   │ Step 12      │   │
│  │ Fuse     │   │ Dequant  │   │ Fix Chat │   │ Import to    │   │
│  │ LoRA     │──▶│ 4-bit to │──▶│ Template │──▶│ Ollama       │   │
│  │ Adapters │   │ bfloat16 │   │ Copy from│   │              │   │
│  │          │   │ (~15 GB) │   │ base     │   │ roachie-8b-ft│   │
│  │ 4.2 GB   │   │          │   │ model    │   │ (16 GB FP16) │   │
│  └──────────┘   └──────────┘   └──────────┘   └──────────────┘   │
│       │              │              │                               │
│       ▼              ▼              ▼                               │
│   DELETED        DELETED     Error: empty                          │
│   (4.2 GB)       (15 GB)    response fixed                        │
│                              by copying                            │
│  Errors: U32 dtype, disk full (both resolved)                      │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 13: CLEANUP                                                │
│     Delete intermediates (19.2GB), keep adapters (16.8MB)          │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│     STEP 14: BATCH TEST VALIDATION                                  │
│                                                                     │
│  Result: 66/73 = 90.4% accuracy                                    │
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
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
                              DONE
                        82% → 90% (+8 pts)


  Accuracy Progression:

  100% ┬────────────────────────────────────────────────
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
```
