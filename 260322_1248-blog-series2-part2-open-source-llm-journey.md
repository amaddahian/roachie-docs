# Making Open-Source LLMs Work for CLI Tool Selection: Llama 8B from 50% to 90%

*Part 2 of 2 — The full journey of making a local, open-source model competitive with cloud LLMs for database tool selection.*

---

In [Part 1](), the origin story covered why Roachie exists: 77 CockroachDB CLI tools that needed a natural language interface, with a firm requirement that everything works offline using local models. Cloud LLMs (Gemini, Claude, GPT-5) hit 95-100% accuracy immediately. The open-source model — Llama 3.1 8B running on Ollama — started at roughly 50-60%.

This is the story of closing that gap. Not all the way — 90% isn't 95% — but enough to make the local model genuinely useful for daily DBA work.

## The Starting Point: Why 50%?

Llama 3.1 8B is a capable general-purpose model. It can write code, explain concepts, and follow instructions. But asking it to select the right tool from 77 options and construct the correct flags is a specialized task. At 50-60% accuracy, every other command was wrong — wrong tool, wrong flags, hallucinated parameters, or truncated JSON output.

The failures fell into three categories:

1. **Wrong tool selection** — "backup database to S3" triggered `cr_backup` instead of `cr_ceph_storage backup`
2. **Wrong or hallucinated flags** — generating `--database movr` when the correct flag is `-d movr`
3. **Truncated output** — JSON responses cut off mid-stream because the output token limit was too low

Each category had a different root cause and needed a different fix.

## Phase 1: Semantic Matching (50% → 72%)

The original tool matching used pure regex — scan the query for keywords and match against tool names. "Show tables" matches `cr_tables`. "Query stats" matches `cr_query_stats`. This works for obvious cases but fails when the user's vocabulary doesn't match the tool name.

"How much disk space is my cluster using?" has no keyword overlap with `cr_db_size`. "What's wrong with my replication?" doesn't contain "lag" (the keyword that would match `cr_replication_lag`).

**Semantic matching** fixed this. Pre-computed embeddings for all 77 tool descriptions, cosine similarity at query time, combined with regex as a safety gate:

- If regex finds matches → combine with semantic results (semantic gets priority)
- If regex finds nothing → skip semantic too (avoids false positives on completely unrelated queries)
- Deduplicate and take top 7

Three embedding providers were implemented — Ollama's nomic-embed-text (768-dim), OpenAI's text-embedding-3-small (1536-dim), and Gemini's gemini-embedding-001 (3072-dim). Even the smallest (Ollama) achieves 99.3% combined accuracy on a 135-query test suite.

This brought tool selection close to perfect. But Llama was still generating wrong flags for the right tools, landing at 72% overall accuracy.

## Phase 2: Prompt Unification (72% → 83%)

This phase produced the biggest single improvement, and it came from deleting code.

The batch test runner had three Ollama-specific code paths that gave Llama a stripped-down experience compared to cloud models:

**1. A "compact" prompt** that excluded 7 template files — parameter rules, tool notes, execution rules, few-shot examples, hallucination warnings. The reasoning: 8B models have limited context, so give them less to process. The reality: those templates contained exactly the information Llama needed most. Without the hallucination warnings, it invented flags. Without the parameter rules, it got tenant syntax wrong. Without the few-shot examples, it had no reference for correct output format.

**2. An enrichment cap of 3 tools** (vs 7 for cloud models). Only 3 tools' `--help` output was injected into the prompt. For queries where the correct tool ranked 4th-7th in matching, Llama had zero documentation to work from.

**3. A hardcoded connection reminder** instead of the dynamic one that included actual cluster hostnames and ports.

The assumption behind all three: Llama's 16K context window can't handle the full prompt.

The assumption was wrong. Llama's tokenizer is roughly **3x more efficient** than Gemini's for technical content. The full prompt with all 7 templates and 7 enriched tools consumed ~5,200 Ollama tokens — well within the 16K budget. The compact prompt was solving a problem that didn't exist.

The fix was three lines of code deletion. Remove the `if ollama` branches. Give every provider the same prompt.

Additionally, three Modelfile settings were fixed:

| Setting | Before | After | Impact |
|---------|--------|-------|--------|
| SYSTEM prompt | 200-token embedded prompt | Removed | Eliminated conflict with runtime system prompt |
| num_predict | 256 | 512 | Stopped JSON truncation for tools with many flags |
| temperature | 0.1 | 0.15 | Better tool selection diversity |

Result: **72% → 83%** from pure infrastructure fixes. No model changes, no training, no new data. Just treating the small model with the same respect as the large one.

## Phase 3: LoRA Fine-Tuning (83% → 90%)

At 83%, the remaining failures were beyond what prompt engineering could fix. Llama was receiving the right context but still making mistakes that Gemini didn't — choosing the wrong tool for ambiguous queries, omitting positional arguments, generating nearly-correct-but-wrong flag combinations.

The gap was model capability. The fix: teach Llama how Gemini solves these problems.

### Training Data: Cloud Model Distillation

Months of batch testing had produced 3,105 records of Gemini solving the same prompts successfully. Each record: a natural language query, the JSON response with tool name and flags, and confirmation that the command executed correctly.

After deduplication by normalized prompt text: **82 unique prompt/command pairs**. Small, but enough for a first attempt.

A Python script extracts, deduplicates, cleans (strips timing prefixes, absolute paths, trailing semicolons), and formats the data as Chat-ML JSONL — the format MLX expects for instruction tuning:

```json
{
  "messages": [
    {"role": "user", "content": "list all tables in the movr database on VA tenant"},
    {"role": "assistant", "content": "{\"tool\": \"cr_db_tables\", \"flags\": \"-h 10.1.1.5 -p 26257 -t va -d movr\"}"}
  ]
}
```

Split: 65 train / 8 validation / 9 test.

### Why Llama 3.1? Why Not a Newer Version?

Llama 3.1 8B Instruct remains the best option at the ~8B size. Meta's subsequent releases don't offer an 8B text-instruct model — Llama 3.2's text models stop at 3B (the 11B/90B are multimodal vision models), Llama 3.3 is 70B only, and Llama 4's Mixture-of-Experts architecture has a much larger total footprint. For a model that needs to run on a standard laptop with 16-32 GB RAM, Llama 3.1 8B is still the sweet spot.

### Training: LoRA on Apple Silicon

Full fine-tuning of 8B parameters is impractical on a laptop. **LoRA** (Low-Rank Adaptation) freezes the base weights and trains small adapter matrices — 16.8 MB of new parameters on top of 4.2 GB.

The training ran on **MLX**, Apple's native ML framework for Apple Silicon:

| Setting | Value |
|---------|-------|
| Base model | Meta-Llama-3.1-8B-Instruct-4bit |
| LoRA rank | 8 |
| LoRA layers | 16 |
| Learning rate | 1e-5 |
| Iterations | 1,000 |
| Peak memory | 6.87 GB |
| Training time | 33 minutes (M1 Pro) |

The validation loss curve told the story of a dataset that's too small:

```
Val Loss
3.3  *
1.3  .  .  .  .  .  .  .  .  .  x
1.0  .  .  .  x  .  x  .  .  .  .
0.8  .  .  x  .  .  .  x  x  .  .
0.5  .  .  .  .  .  .  .  .  x  .
0.3  .  x  .  .  .  *  .  .  .  .   ← Best: iter 600 (0.332)
     100 200 300 400 500 600 700 800 900 1000
```

Training loss dropped to near-zero by iteration 300 (full memorization). Validation loss oscillated wildly — the hallmark of overfitting on limited data. But iteration 600 hit a sweet spot at 0.332 validation loss, and those adapter weights were selected.

### The Conversion Pipeline: Three Roadblocks

Getting from MLX weights to a running Ollama model required solving three compatibility issues:

**1. Ollama can't read MLX 4-bit weights.** MLX stores 4-bit quantization as packed U32 integers. Ollama rejects them with `Error: unknown data type: U32`. The workaround: de-quantize to bfloat16 (~15 GB), let Ollama re-quantize on import.

**2. Disk space.** The de-quantization inflates the model from 4.2 GB to 15 GB. With limited disk space, intermediate files had to be deleted at each step. Budget 3x the model size for the pipeline.

**3. Missing chat template.** MLX's de-quantization step silently strips the `chat_template` field from `tokenizer_config.json`. Without it, Ollama doesn't format the prompt with Llama 3.1's special tokens (`<|begin_of_text|>`, `<|start_header_id|>`, etc.). The model receives raw text, produces empty output, and appears broken. The fix: copy the chat template from the original base model's tokenizer config. This was the hardest bug to diagnose — the error manifested as the model returning `{ }` after processing only 11 tokens.

All three fixes are automated in `run_lora.sh`, so future training runs handle them transparently.

## Results: The Full Accuracy Journey

| Stage | Accuracy | Method |
|-------|----------|--------|
| Baseline (regex matching, compact prompt) | ~50-60% | Original implementation |
| + Semantic embeddings | 72% | Two-model architecture (embedding + LLM) |
| + Unified prompt (all 7 templates) | 83% | Deleted Ollama-specific code paths |
| + Modelfile fixes (num_predict, temperature) | ~87% | Configuration changes |
| + LoRA fine-tuning (82 Gemini examples) | **90%** | Model weight adaptation |
| Gemini (reference) | 95-100% | Cloud model |

The 5 remaining failures (out of 73 test prompts) are:
- 2 DDL table commands missing positional arguments
- 1 wrong tool selection (backup)
- 1 missing database flag
- 1 complex migration with truncated flags

These represent the boundary of what 8B parameters can handle for this task.

## Storage Requirements

The fine-tuning and conversion pipeline has significant disk demands:

| Component | Size | Notes |
|-----------|------|-------|
| Base model (4-bit) | 4.2 GB | Cached in HuggingFace hub |
| LoRA adapters | 16.8 MB | Permanent — the only new weights |
| Fused model (4-bit) | 4.2 GB | Temporary — deleted after de-quantize |
| De-quantized model (bfloat16) | ~15 GB | Temporary — deleted after Ollama import |
| **Final model in Ollama** | **16 GB** | Permanent (FP16) |
| **Peak disk during conversion** | **~25 GB** | Budget this much free space |

The non-fine-tuned roachie models in Ollama are much smaller (~4.5 GB each at Q4_K_M quantization). The fine-tuned model is larger because it's stored at FP16 precision — a future optimization would be to re-quantize it to Q4_K_M after import, halving the memory footprint with minimal accuracy loss.

## The Economics of Local vs. Cloud

| | Local (Ollama) | Cloud (Gemini) |
|-|---------------|----------------|
| Accuracy | 90% | 95-100% |
| Per-query cost | $0.00 | ~$0.005-0.01 |
| Latency | ~2-5s (M1 Pro) | ~1-3s (network + inference) |
| Privacy | 100% local | Data sent to Google |
| Offline | Yes | No |
| Setup cost | 33 min training + free hardware | API key |
| Model size | 16 GB | N/A |

For 100 queries/day, the cloud model costs $0.50-1.00/day (~$200-365/year). The local model costs nothing after a one-time 33-minute training run. More importantly, it works in environments where sending database metadata to a cloud API isn't an option.

## What's Next: Reaching 95%

The gap from 90% to 95% is harder than the gap from 50% to 90%. Several approaches are on the roadmap:

**More training data.** 82 examples isn't enough for stable generalization. Running 300+ diverse prompts through Gemini and including paraphrased versions of existing prompts would smooth the validation curve and improve coverage of edge cases.

**Best-of-N sampling.** Generate 3 candidate responses from Ollama with slightly varied temperature, validate each against the tool's `--help` output, pick the one with the most valid flags. A cheap accuracy boost without retraining.

**Constrained decoding.** Ollama supports GBNF grammars that restrict output tokens at the decoder level. A grammar that only allows valid flag names per tool would eliminate hallucinated flags entirely — no retraining needed.

**Tool calling validation loop.** After generating a command, extract the flags and validate against `--help`. If invalid flags are found, feed the error back to the LLM for one correction attempt. Lighter than best-of-N and more targeted.

**Larger model.** Llama 3.1 70B would handle the complex tools (migration, backup, DDL with positional args) better, but requires ~40 GB of memory and is 5-10x slower. Worth testing on machines with 64+ GB RAM.

**Alternative 12B models.** Early testing of Mistral Nemo 12B (no fine-tuning) achieved 87.7% accuracy on the same 73-prompt suite — nearly matching the fine-tuned Llama's 90%. Tool selection was near-perfect (72/73); all failures were flag-level errors. The trade-off: ~79s average latency per query (vs ~3-5s for Llama 8B) and ~3 GB more RAM. With LoRA fine-tuning, 93-95% is plausible. Google's Gemma 3 12B is also under evaluation.

## Lessons for Anyone Building with Local LLMs

**1. Don't simplify the prompt for small models.** The instinct is to strip context to save tokens. Resist it. Measure the actual token count first. Small models need *more* context, not less — they can't infer what large models can.

**2. Cloud model outputs are free supervised training data.** If queries are already running against a cloud LLM, log every successful response. That's a dataset for fine-tuning a local model that costs nothing to collect.

**3. Infrastructure fixes before model fixes.** 33 of the 40 percentage points gained came from code changes, not model training. Prompt engineering, correct tokenizer settings, and unified code paths are cheaper and faster than fine-tuning.

**4. LoRA on Apple Silicon is surprisingly practical.** 33 minutes, 6.87 GB peak memory, on a standard MacBook Pro. The MLX ecosystem has made laptop-scale fine-tuning viable for domain-specific tasks.

**5. The conversion pipeline needs automation.** MLX → Ollama has three non-obvious failure modes (U32 dtype, disk space, chat template). Automate the entire pipeline end-to-end on the first successful run, because manual steps will be forgotten.

**6. 90% is good enough to be useful.** A local model at 90% with zero cost, full privacy, and offline capability is more valuable in many environments than a cloud model at 98% that requires internet access and sends data externally. Perfect is the enemy of deployed.

---

*This is Part 2 of a 2-part series. [Part 1]() covers why Roachie was built: the gap in database tooling, the 77-tool CLI toolkit, and the architecture of the natural language interface.*

*For the full technical deep-dive with code snippets and loss curves, see the companion posts:*
- *[Making a Local LLM Actually Useful: Llama 8B from 50% to 83%]()*
- *[LoRA Fine-Tuning Llama 8B on Apple Silicon: 83% to 90%]()*

*Roachie is open source: [github.com/amaddahian/roachie](https://github.com/amaddahian/roachie)*
