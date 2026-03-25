# Roachie Model Architecture Analysis -- Evaluation Against LLM Architecture Paradigms

**Date:** 2026-03-25
**Scope:** How roachie uses (and could benefit from) the six major neural architecture paradigms across its AI pipeline

---

## Overview -- Architecture Usage Map

Roachie's AI pipeline uses models from multiple architecture families across two distinct roles: **embedding** (understanding/matching) and **generation** (reasoning/commanding). Here is how the six paradigms map to what roachie actually deploys:

| Architecture | Used in Roachie? | Where | Models |
|-------------|-----------------|-------|--------|
| 1. Encoder-Only | **Yes** | Embedding generation | nomic-embed-text (Ollama), text-embedding-3-small (OpenAI), gemini-embedding-001 (Gemini) |
| 2. Decoder-Only | **Yes (primary)** | Response generation | Claude Sonnet 4.6, GPT-4.1/5, Gemini 2.5, Llama 3.1 8B, Mistral Nemo 12B |
| 3. Encoder-Decoder | **No** | -- | -- |
| 4. Mixture of Experts | **Indirectly** | Response generation | Gemini 2.5 (MoE internally), Mistral Nemo (sparse MoE) |
| 5. State Space Models | **No** | -- | -- |
| 6. Hybrid Architectures | **Indirectly** | Response generation | Gemini 2.5 (attention + MoE), potentially future Ollama models |

---

## 1. Encoder-Only (Autoencoders)

**Architecture:** Bidirectional attention over input; produces contextualized embeddings. No text generation. Examples: BERT, RoBERTa, nomic-embed-text.

### How Roachie Uses Encoder-Only Models

Roachie uses encoder-only models exclusively for **semantic matching** -- converting text (tool descriptions and user queries) into fixed-dimensional vectors for cosine similarity retrieval.

| Model | Provider | Dimensions | Role | File |
|-------|----------|-----------|------|------|
| nomic-embed-text | Ollama (local) | 768 | Tool matching + Doc RAG | `llm_prompt.sh:146-148` |
| text-embedding-3-small | OpenAI | 1536 | Tool matching + Doc RAG | `llm_prompt.sh:153-155` |
| gemini-embedding-001 | Gemini | 3072 | Tool matching + Doc RAG | `llm_prompt.sh:158-162` |

**Pipeline:**
1. **Offline:** Each of the 77 tool descriptions + 14 doc chunks is embedded and stored as a JSON array
2. **Runtime:** User query is embedded (single API call), cosine similarity computed against all pre-computed embeddings via jq
3. **Output:** Top-7 most similar tools and top-3 doc chunks injected into system prompt

### Pros

- **Right model for the right job** -- Encoder-only models produce superior embeddings compared to decoder-only models because bidirectional attention captures full context in both directions. Using an embedding-specific model rather than extracting embeddings from a generative model is the correct architectural choice.
- **Three-provider redundancy** -- Having three embedding providers with automatic fallback (Ollama -> OpenAI -> Gemini -> regex) means the system degrades gracefully. If no embedding model is available, regex matching takes over.
- **Local-first option** -- nomic-embed-text runs locally via Ollama with zero API cost and zero data leakage. This is critical for sensitive DBA environments.
- **Pre-computation amortizes cost** -- Embeddings are generated once offline and shipped with the code. Runtime cost is a single query embedding per user query (~1ms local, ~100ms API).
- **Embedding reuse** -- A single query embedding is computed once and reused for both tool matching and Doc RAG retrieval, eliminating redundant API calls.

### Cons

- **No fine-tuned embedding model** -- The embedding models are used off-the-shelf. A domain-fine-tuned encoder (trained on CockroachDB terminology: "range," "leaseholder," "changefeed," "gc.ttlseconds") would produce embeddings where domain-specific terms cluster correctly. Currently, "range" in CockroachDB context (a data partition) maps similarly to "range" in general English (a span of values).
- **Single-vector representation** -- Each tool gets one embedding from its description. Multi-function tools (e.g., `cr_backup` handles backup, restore, schedule, list) are represented by a single averaged vector. An encoder-only model fine-tuned with contrastive learning on (query, tool) pairs would produce better task-specific embeddings.
- **Dimension mismatch across providers** -- The three providers produce 768, 1536, and 3072 dimensional vectors respectively. There is no normalization or dimensionality alignment. Switching providers mid-session would require recomputing all embeddings (though this doesn't happen in practice since provider is fixed per session).
- **No cross-encoder re-ranking** -- Encoder-only models can be used as cross-encoders (encoding query+document together) for re-ranking after initial retrieval. Roachie's pipeline does initial retrieval only, with no second-stage re-ranking.

### Room to Improve

1. **Fine-tune a domain-specific encoder** -- Use contrastive learning (e.g., sentence-transformers) on (user query, correct tool) pairs from the 135-query test set + batch results. Train on nomic-embed-text base and deploy via Ollama. This would improve embedding quality for CockroachDB-specific vocabulary.
2. **Add cross-encoder re-ranking** -- After cosine similarity retrieval returns the top-7, run a cross-encoder (e.g., `cross-encoder/ms-marco-MiniLM-L-6-v2` via Ollama) to re-rank by encoding (query, tool_description) pairs together. Cross-encoders are more accurate than bi-encoders for re-ranking but too expensive for full-corpus search.
3. **Consider Matryoshka embeddings** -- OpenAI's text-embedding-3-small supports Matryoshka dimensionality reduction (truncate to 256 or 512 dims with minimal quality loss). This would reduce JSON file sizes and jq computation by 3-6x.

---

## 2. Decoder-Only (Autoregressive)

**Architecture:** Left-to-right (causal) attention; generates tokens one at a time. The dominant paradigm for modern LLMs. Examples: GPT-4, Claude, Llama, Mistral.

### How Roachie Uses Decoder-Only Models

This is roachie's **primary architecture** -- all response generation uses decoder-only models.

| Model | Provider | Parameters | Context | Role |
|-------|----------|-----------|---------|------|
| Claude Sonnet 4.6 | Anthropic / Vertex AI | Unknown (large) | 200K | Primary cloud generation |
| GPT-4.1 / GPT-5 | OpenAI | Unknown (large) | 128K | Alternative cloud generation |
| Gemini 2.5 Flash/Pro | Google | Unknown (MoE) | 1M | Alternative cloud generation (with thinking) |
| Llama 3.1 8B (LoRA) | Ollama (local) | 8B | 16K | Local fine-tuned generation |
| Mistral Nemo 12B (LoRA) | Ollama (local) | 12B | 16K | Local fine-tuned generation |
| roachie-3b | Ollama (local) | 3B | 16K | Lightweight local generation |

**Pipeline:**
1. System prompt (3-10K tokens) + conversation history + user query sent to decoder-only model
2. Model generates structured JSON response: `{reasoning, user_message, commands[], needs_followup}`
3. Commands extracted, validated (6-layer), and executed
4. If `needs_followup`, output fed back for next iteration (agent loop, max 3)

### Pros

- **Multi-provider architecture with unified interface** -- Five providers behind a single `_call_llm_api()` dispatcher is excellent abstraction. The user can switch between a $0.003/query cloud model and a free local model without code changes. Provider-specific quirks (o-series temperature, Gemini markdown fences, Mistral stop tokens) are handled transparently.
- **LoRA fine-tuning for local models** -- The fine-tuning pipeline (prepare_training_data.py -> run_lora.sh -> Ollama import) bridges the accuracy gap between 8B local models and 100B+ cloud models. The 90% accuracy on LoRA-tuned Llama 3.1 8B vs ~95%+ on Claude/GPT demonstrates effective knowledge distillation.
- **Structured JSON output** -- Constraining decoder-only models to produce structured JSON (with Ollama's `format: "json"` and cloud providers' `responseMimeType`) is the right approach. It eliminates the parsing ambiguity that plagues free-form text generation.
- **Low temperature (0.2)** -- For tool-calling, you want deterministic output. The 0.2 temperature setting minimizes sampling randomness while allowing minor variation for paraphrase.
- **Streaming with progress indication** -- SSE streaming for cloud providers (Anthropic, OpenAI, Gemini) provides immediate feedback. This is a UX advantage of autoregressive models -- you can display partial output as tokens are generated.
- **Agent loop with reflexion** -- The multi-iteration agent loop exploits the autoregressive nature: each iteration builds on the previous output, allowing the model to self-correct. This is a natural fit for decoder-only models which excel at conditional generation.
- **Model selection menus** -- The OpenAI sub-menu (GPT-4.1, GPT-4.1-mini, GPT-5, o3-mini, o4-mini) and Gemini sub-menu (2.5-flash, 2.5-pro, 2.0-flash) give users cost/quality control within the decoder-only paradigm.

### Cons

- **No speculative decoding** -- For local Ollama models (8B-12B), generation is slow (5-30 seconds for complex queries). Speculative decoding with a smaller draft model (e.g., roachie-3b drafting for roachie-8b-ft) could significantly reduce latency. Ollama supports this but roachie doesn't configure it.
- **No constrained decoding beyond JSON format** -- While Ollama uses `format: "json"`, there is no schema-level constraint (e.g., "output must have exactly these fields"). The LLM can produce valid JSON that doesn't match the expected schema (missing `commands` array, wrong field types). Cloud providers' structured output modes (Anthropic's `tool_use`, OpenAI's `response_format`) are used for native tool calling but not for the default text-based response.
- **Ollama streaming not implemented** -- Despite decoder-only models being inherently streamable (token-by-token generation), the Ollama provider hardcodes `stream: false`. Users see no output for 5-30 seconds on local models, which is the worst case for the architecture that best supports progressive output.
- **Context window underutilized for cloud providers** -- Anthropic offers 200K tokens and Gemini offers 1M, but the system caps history at 10 turns and trims aggressively at 80% capacity. For long analysis sessions, the models could handle far more context than the system provides.
- **No model-specific prompt optimization** -- The same system prompt is sent to Claude, GPT, Gemini, and Ollama (with only a compact variant for Ollama). Different decoder-only architectures have different instruction-following strengths. Claude excels with XML tags, GPT with structured instructions, Gemini with examples. Prompt formatting could be provider-optimized.
- **Training data distribution mismatch** -- LoRA fine-tuned models are trained on user/assistant pairs without the system prompt they receive at inference. This underutilizes the autoregressive model's ability to follow system instructions -- it never learned to do so during training.

### Room to Improve

1. **Enable Ollama streaming** -- Implement `_call_ollama_api_stream()` to show progress dots and partial output during local generation. This is straightforward since Ollama's API returns NDJSON chunks.
2. **Configure speculative decoding for Ollama** -- Use roachie-3b as a draft model for roachie-8b-ft to accelerate local generation by 2-3x. Ollama supports this via the `--draft` flag or Modelfile `FROM` chain.
3. **Add schema validation on responses** -- After JSON parsing, validate against an expected schema (commands must be an array, each command must have `command` and `description` fields). Reject and retry if schema doesn't match.
4. **Include system prompt in fine-tuning data** -- The autoregressive model needs to see the system prompt during training to learn how to follow its instructions. This is the single highest-impact change for local model accuracy.
5. **Consider provider-specific prompt formatting** -- Wrap system prompt sections in `<section>` XML tags for Anthropic, use numbered headings for OpenAI, and use `**bold**` emphasis for Gemini. This aligns with each model's training distribution.

---

## 3. Encoder-Decoder (Seq2Seq)

**Architecture:** Separate encoder (bidirectional, processes input) and decoder (autoregressive, generates output). Examples: T5, BART, Flan-T5.

### How Roachie Uses Encoder-Decoder Models

**Roachie does not use encoder-decoder models.** All generation is decoder-only, and all encoding is encoder-only.

### Pros (of NOT using Seq2Seq)

- **Correct architectural choice** -- Encoder-decoder models excel at tasks with a clear input-output mapping (translation, summarization, question-answering with short answers). Roachie's task -- generating structured JSON with reasoning, commands, and follow-up decisions from a complex multi-part prompt -- is better suited to decoder-only models which handle long, instruction-heavy prompts natively.
- **No architecture overhead** -- Seq2Seq models require managing both encoder and decoder states, cross-attention, and potentially different tokenizers. Avoiding this simplifies the pipeline.
- **Ecosystem alignment** -- The industry has converged on decoder-only for general-purpose generation. All five of roachie's providers offer decoder-only models. Using Seq2Seq would mean abandoning the multi-provider architecture.

### Cons (of NOT using Seq2Seq)

- **Missed opportunity for lightweight query classification** -- A small Seq2Seq model (e.g., Flan-T5-small at 80M parameters) could serve as a fast, local query classifier: "Does this query need a command or is it informational?" This would be faster than sending every query to an 8B+ decoder-only model.
- **No specialized SQL generation** -- Encoder-decoder models like NSQL or SQLCoder are specifically trained for natural language to SQL translation. For roachie's `cockroach sql` commands, a specialized Seq2Seq model could generate more accurate SQL than a general-purpose decoder-only model.

### Room to Improve

1. **Consider a Seq2Seq query classifier** -- A Flan-T5-small (~80M params) model running locally could classify queries in <50ms as: `command` (needs cr_* tool), `sql` (needs cockroach sql), `informational` (no command), `ambiguous` (ask for clarification). This would eliminate wasted LLM calls for informational queries and could route SQL queries to a specialized model.
2. **Evaluate Seq2Seq for SQL generation** -- For the subset of queries that produce `cockroach sql -e "SELECT ..."` commands, a specialized NL-to-SQL model (e.g., fine-tuned T5 on CockroachDB's SQL dialect) could be more accurate than a general-purpose LLM, especially for complex JOINs and window functions.

---

## 4. Mixture of Experts (MoE)

**Architecture:** Multiple "expert" sub-networks with a gating/routing mechanism that activates only a subset of experts per token. Reduces compute cost for large models. Examples: Gemini 2.5, Mixtral, GPT-4 (rumored).

### How Roachie Uses MoE Models

Roachie uses MoE models **indirectly** through two providers:

| Model | MoE Architecture | Active Parameters | Total Parameters | How Used |
|-------|-----------------|-------------------|------------------|----------|
| Gemini 2.5 Flash/Pro | Dense attention + MoE FFN | ~40-80B (estimated) | 300B+ (estimated) | Cloud generation via API |
| Mistral Nemo 12B | Sparse MoE (Nemo variant) | ~12B (dense in this version) | 12B | Local LoRA fine-tuned generation |

**Note:** Mistral Nemo Instruct 12B is actually a dense model (not MoE), despite Mistral's MoE reputation. The Mixtral series (8x7B, 8x22B) uses MoE, but roachie's Nemo variant does not.

### Pros

- **Gemini MoE benefits are free** -- Roachie benefits from Gemini's MoE architecture (faster inference, lower cost per token) without any special code. The API abstracts away the MoE routing entirely. Gemini 2.5 Flash offers the best cost-per-quality ratio among cloud providers.
- **MoE-aware thinking budget** -- Roachie's `_NL_GEMINI_THINKING_BUDGET` configuration (defaulting to 0, configurable via `NL_GEMINI_THINKING`) leverages Gemini 2.5's extended thinking capability, which benefits from MoE's efficient routing during the thinking phase.
- **Cost advantage** -- MoE models activate fewer parameters per token, making them cheaper to run. Gemini 2.5 Flash's pricing reflects this: significantly cheaper per token than dense models of equivalent quality.

### Cons

- **No local MoE option** -- Roachie's local models (Llama 3.1 8B, Mistral Nemo 12B) are dense architectures. A local MoE model (e.g., Mixtral 8x7B via Ollama) could provide better quality at the same memory footprint by activating only 2 of 8 experts per token (12.9B active out of 46.7B total).
- **MoE routing not observable** -- Since MoE is behind the API, roachie cannot observe which experts are being activated. There is no way to know if CockroachDB-related queries consistently activate the same experts, which would be useful information for evaluating model suitability.
- **Gemini thinking budget defaults to zero** -- With thinking disabled, Gemini 2.5 doesn't leverage its MoE architecture for extended reasoning. For complex multi-step queries (schema comparison, performance diagnosis), the thinking budget should be non-zero to take advantage of MoE's efficient reasoning.
- **No MoE-specific fine-tuning** -- LoRA fine-tuning is only applied to dense models (Llama 8B, Mistral 12B). Fine-tuning a local MoE model could provide the quality benefits of a larger model within the same memory budget.

### Room to Improve

1. **Add Mixtral 8x7B as a local option** -- Mixtral 8x7B runs on 32GB Apple Silicon and activates 12.9B parameters per token (vs 8B for Llama). This would provide better local model quality. Add to Ollama model detection: `ollama-mixtral`.
2. **Enable Gemini thinking by default** -- Set `_NL_GEMINI_THINKING_BUDGET` to 2048 and increase `_NL_GEMINI_MAX_TOKENS` to 12288. This leverages MoE's efficient routing during extended reasoning without significantly increasing cost (MoE thinking tokens are cheaper than dense thinking tokens).
3. **Evaluate Mixtral for LoRA fine-tuning** -- If Mixtral 8x7B can be fine-tuned with LoRA on the expert layers (selecting which experts to fine-tune), it could provide 12B-equivalent quality with 8B-equivalent compute. MLX supports MoE LoRA.

---

## 5. State Space Models (SSM)

**Architecture:** Linear-time sequence modeling (no quadratic attention). O(n) complexity vs O(n^2) for transformers. Examples: Mamba, Mamba-2, RWKV, Jamba.

### How Roachie Uses SSM Models

**Roachie does not currently use SSM models.** All generation uses transformer-based decoder-only models, and all embedding uses transformer-based encoders.

### Pros (Potential Benefits for Roachie)

- **Long-context efficiency** -- SSMs process sequences in O(n) time vs O(n^2) for attention. For Ollama's 16K context window, this is irrelevant. But if roachie wanted to process long command outputs (e.g., full `cr_ddl` output for a 500-table database, which could be 100K+ tokens), an SSM could handle it efficiently on local hardware where a transformer would OOM.
- **Faster local inference** -- SSMs have lower memory and compute requirements per token. A Mamba-based model at 8B parameters would generate tokens faster than Llama 3.1 8B on the same hardware, reducing the 5-30 second local generation time.
- **Constant-memory decoding** -- SSMs use a fixed-size state that doesn't grow with sequence length (unlike KV cache in transformers). For long interactive sessions with many exchanges, this prevents the memory growth that eventually forces context trimming.

### Cons (Why SSMs Don't Fit Today)

- **Instruction following is weaker** -- Current SSM models (Mamba, RWKV) lag behind transformers on instruction following and structured output generation. Roachie requires precise JSON output with exact tool names and flag syntax. SSMs' weaker in-context learning would reduce accuracy.
- **No Ollama-ready SSM models** -- As of March 2026, no SSM models are well-supported in Ollama's model library with chat templates and instruction tuning. The ecosystem isn't ready.
- **No LoRA tooling for SSMs** -- The LoRA fine-tuning pipeline (`mlx-lm`) doesn't support SSM architectures (Mamba uses a different parameterization than transformers). Fine-tuning would require a different training framework.
- **Embedding models are encoder-only** -- SSMs are a replacement for decoder-only generation, not for encoder-only embedding. The embedding pipeline would remain unchanged.

### Room to Improve

1. **Monitor Mamba-2 / Jamba ecosystem** -- Jamba (AI21's hybrid Mamba+attention model) is available via API and could be added as a provider. If Jamba models become available in Ollama, they would offer faster local inference with reasonable instruction following.
2. **Consider SSM for command output processing** -- If a future feature requires processing very long command outputs (100K+ tokens of DDL or log data), an SSM could summarize the output before feeding it to the main LLM. This would be a specialized SSM-as-preprocessor role.
3. **Watch for RWKV v6 in Ollama** -- RWKV v6 (Eagle architecture) is approaching transformer-quality instruction following with O(n) inference. When available in Ollama with proper chat templates, it would be a strong candidate for resource-constrained environments.

---

## 6. Hybrid Architectures

**Architecture:** Combines elements from multiple paradigms (attention + SSM, dense + MoE, encoder + decoder with shared weights). Examples: Jamba (Mamba + attention), StripedHyena, Gemini (attention + MoE).

### How Roachie Uses Hybrid Architectures

Roachie uses hybrid architectures **indirectly** through models that internally combine multiple paradigms:

| Model | Hybrid Nature | Components | How Used |
|-------|-------------|------------|----------|
| Gemini 2.5 | Attention + MoE | Dense attention layers + MoE FFN layers | Cloud generation |
| GPT-5 | Potentially hybrid (undisclosed) | Unknown internals | Cloud generation |

More importantly, **roachie's own pipeline is a hybrid architecture** at the system level:

| Component | Architecture Paradigm | Model/Method |
|-----------|---------------------|--------------|
| Query understanding | Encoder-only | nomic-embed-text / text-embedding-3-small |
| Intent classification | Rule-based (regex) | Bash `[[ =~ ]]` keyword matching |
| Tool retrieval | Hybrid (regex + vector) | Keyword scoring + cosine similarity |
| Response generation | Decoder-only | Claude / GPT / Gemini / Llama / Mistral |
| Command validation | Rule-based (deterministic) | 6-layer whitelist + metachar + RBAC |
| Error correction | Decoder-only (iterative) | Reflexion agent loop |
| Learning | Statistical (pattern matching) | Persistent failure/success JSONL |

### Pros

- **System-level hybrid design is the key strength** -- Roachie doesn't rely on a single architecture for everything. It composes:
  - **Encoder-only** for understanding (embeddings)
  - **Rule-based** for safety (validation) and precision (regex matching)
  - **Decoder-only** for reasoning and generation
  - **Statistical** for learning (persistent patterns)

  This composition means each component uses the architecture best suited to its task. The rule-based validation layer is deterministic and unbypassable -- no amount of prompt engineering can override `_check_metachar()`.

- **Graceful degradation through architectural layers** -- If the embedding model fails, regex takes over. If the LLM fails, the system retries with a different provider. If the command fails, reflexion self-corrects. This layered resilience is a direct benefit of the hybrid approach.

- **Two-model architecture separates concerns** -- The embedding model (understanding) and the generation model (reasoning) are completely independent. You can use Ollama for embeddings and Anthropic for generation, or OpenAI for both. This decoupling allows optimizing each component independently.

- **Rule-based components are auditable** -- Unlike neural models, the regex matching, command validation, and RBAC gating are fully transparent and deterministic. A security auditor can verify that `_check_metachar()` blocks all shell injection patterns without understanding neural network weights.

- **Hybrid matching exceeds either component alone** -- The regex+semantic combined matching (99-100% accuracy) outperforms regex-only (96.8%) and semantic-only (~95%). This is empirical validation of the hybrid approach.

### Cons

- **No hybrid model architectures for local inference** -- While the system pipeline is hybrid, the actual neural models used locally are standard dense transformers (Llama, Mistral). Hybrid models like Jamba (Mamba + attention) would offer faster inference with similar quality but are not yet available in the pipeline.

- **No neural-symbolic integration for validation** -- The command validation is purely rule-based (regex, whitelist). A neural-symbolic approach could learn new validation rules from failure patterns. For example, if the model repeatedly generates `--format=json` (with `=`), the system could learn to auto-correct this pattern rather than relying on the generic `sed` normalization.

- **No learned routing between providers** -- The provider selection is static (manual or priority-based detection). A learned router could predict which provider will give the best response for a given query type (e.g., Claude for complex reasoning, Gemini for SQL generation, Ollama for simple tool lookups) based on historical performance data.

- **No attention/SSM hybrid for local models** -- Hybrid architectures like StripedHyena or Jamba combine attention layers (good for precise instruction following) with SSM layers (good for long-range dependencies and speed). This would be ideal for roachie's use case: precise flag syntax (attention) + understanding long command outputs (SSM).

- **Component boundaries are coarse** -- The pipeline switches between architectures at fixed boundaries (embedding -> regex -> semantic -> LLM -> validation). A more fine-grained hybrid could interleave neural and rule-based processing within the generation step (e.g., constrain generation token-by-token to only produce valid flag names).

### Room to Improve

1. **Add learned provider routing** -- Track per-provider accuracy by query category (schema inspection, performance analysis, DDL generation, etc.) in the metrics CSV. Use this historical data to route queries to the provider with the best track record for that category. This is a simple statistical hybrid: neural (LLM) + statistical (routing).

2. **Implement neural-guided validation** -- When persistent learning shows repeated failures with a specific pattern (e.g., model always generates `-T TABLE` for cr_ddl_table), add an auto-correction rule to `sanitize_command()` that catches this before execution. This bridges the rule-based (validation) and statistical (learning) components.

3. **Evaluate Jamba when available in Ollama** -- AI21's Jamba (Mamba + attention hybrid) offers near-transformer quality with significantly faster inference. When available as an Ollama model with proper chat templates, it would be the ideal local model for roachie: fast generation + strong instruction following.

4. **Consider constrained decoding for critical fields** -- For the `commands[].command` field in the JSON response, use a trie of valid tool names + flags to constrain the decoder's token generation. This would be a hybrid of neural generation (decoder-only) and deterministic validation (trie-based constraint) that prevents invalid tool names from ever being generated.

5. **Build a lightweight query classifier** -- A small model (even logistic regression on TF-IDF features) could classify queries before they reach the LLM:
   - `tool_query`: Route to normal pipeline
   - `informational`: Skip LLM, return cached answer or tool documentation
   - `sql_query`: Route to SQL-specialized handling
   - `multi_step`: Set `max_iterations=3`, enable thinking budget

   This classifier adds an explicit routing layer to the hybrid pipeline.

---

## Summary Scorecard

| Architecture | Relevance to Roachie | Current Usage | Score | Key Opportunity |
|-------------|---------------------|---------------|-------|-----------------|
| 1. Encoder-Only | **High** | 3-provider embeddings for semantic matching | 8/10 | Fine-tune domain-specific encoder; add cross-encoder re-ranking |
| 2. Decoder-Only | **Critical** | 5-provider generation (cloud + local LoRA) | 9/10 | Enable Ollama streaming; speculative decoding; include system prompt in training |
| 3. Encoder-Decoder | **Low** | Not used | 5/10 | Consider for query classification or specialized SQL generation |
| 4. MoE | **Medium** | Indirect via Gemini 2.5 | 6/10 | Add Mixtral as local option; enable Gemini thinking; MoE LoRA |
| 5. SSM | **Future** | Not used | 3/10 | Monitor Mamba-2/Jamba/RWKV for fast local inference |
| 6. Hybrid | **System-level** | Pipeline is a hybrid of all paradigms | 8.5/10 | Learned provider routing; neural-guided validation; constrained decoding |

### Key Insight

Roachie's greatest architectural strength is not any single model choice, but the **system-level hybrid design** that composes encoder-only models, decoder-only models, rule-based systems, and statistical learning into a pipeline where each component does what it does best. The rule-based validation layer is especially important -- it provides deterministic safety guarantees that no neural model can offer.

The largest opportunities are:
1. **Include system prompt in LoRA training** (decoder-only improvement -- highest impact on local model accuracy)
2. **Enable Ollama streaming** (decoder-only UX -- most visible improvement for local users)
3. **Fine-tune a domain encoder** (encoder-only improvement -- better embedding quality for CockroachDB terminology)
4. **Add Mixtral 8x7B** (MoE -- better local model quality within same memory budget)
5. **Build a query classifier** (hybrid -- reduce unnecessary LLM calls, enable specialized routing)

---

## CLI Summary (Quick Reference)

This section mirrors the summary provided in the Claude Code terminal for easy cross-referencing.

### Architecture Usage Map

| Architecture | Relevance | Current Usage | Score | Key Opportunity |
|-------------|-----------|---------------|-------|-----------------|
| 1. Encoder-Only | **High** | 3-provider embeddings for semantic matching | 8/10 | Fine-tune domain-specific encoder; add cross-encoder re-ranking |
| 2. Decoder-Only | **Critical** | 5-provider generation (cloud + local LoRA) | 9/10 | Enable Ollama streaming; speculative decoding; include system prompt in training |
| 3. Encoder-Decoder | **Low** | Not used | 5/10 | Consider for query classification or specialized SQL generation |
| 4. MoE | **Medium** | Indirect via Gemini 2.5 | 6/10 | Add Mixtral as local option; enable Gemini thinking; MoE LoRA |
| 5. SSM | **Future** | Not used | 3/10 | Monitor Mamba-2/Jamba/RWKV for fast local inference |
| 6. Hybrid | **System-level** | Pipeline is a hybrid of all paradigms | 8.5/10 | Learned provider routing; neural-guided validation; constrained decoding |

### Key Insight

Roachie's greatest architectural strength is not any single model choice, but the **system-level hybrid design** that composes encoder-only models, decoder-only models, rule-based systems, and statistical learning into a pipeline where each component does what it does best. The rule-based validation layer is especially important -- it provides deterministic safety guarantees that no neural model can offer.

### Top 5 Opportunities by Impact

1. **Include system prompt in LoRA training** (decoder-only) -- Highest impact on local model accuracy. Training without the system prompt means the fine-tuned model has never seen the context it receives at inference time.
2. **Enable Ollama streaming** (decoder-only) -- Most visible UX improvement for local users. Currently Ollama responses appear all at once after full generation.
3. **Fine-tune a domain encoder** (encoder-only) -- Better embedding quality for CockroachDB-specific terminology. nomic-embed-text is general-purpose; a domain-adapted encoder would improve tool matching precision.
4. **Add Mixtral 8x7B** (MoE) -- Better local model quality within similar memory budget. MoE activates only 2 of 8 experts per token, so a 46B parameter model runs at ~12B speed.
5. **Build a query classifier** (hybrid) -- Reduce unnecessary LLM calls, enable specialized routing by query type (tool lookup vs informational vs SQL vs multi-step).

### Where Each Architecture Would Help Most

- **Encoder-Only**: Cross-encoder re-ranking would eliminate false positive tool matches without adding latency to the initial retrieval
- **Decoder-Only**: Streaming + speculative decoding would make local inference feel as responsive as cloud APIs
- **Encoder-Decoder**: A small T5-based query normalizer could standardize user input before it reaches the generative model
- **MoE**: Mixtral offers the best quality/speed tradeoff for local inference when 8B dense models aren't accurate enough
- **SSM**: Mamba-based models would excel at processing long command outputs in the agent loop (faster than attention for long sequences)
- **Hybrid**: Constrained decoding would prevent invalid tool names from ever being generated, reducing validation failures to zero
