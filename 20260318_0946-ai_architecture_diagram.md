# Roachie AI Architecture Diagram — 2026-03-18

## Full NL Pipeline Architecture

```mermaid
flowchart TB
    subgraph Entry["Entry Points"]
        RN["bin/roachie-nl<br/>(interactive REPL)"]
        RB["bin/roachie-batch<br/>(scripted/pipeline)"]
        RM["bin/roachman<br/>(menu option 14)"]
    end

    subgraph SessionInit["Session Initialization (once per session)"]
        direction TB
        SI_TOPO["Cluster Topology Detection<br/>_nl_cluster_topology()"]
        SI_SCHEMA["Schema Context Fetch<br/>_nl_fetch_schema_context()<br/>SHOW DATABASES + information_schema<br/>per tenant (max 3 tenants, 8 DBs, 50 tables)"]
        SI_PROMPT["Base System Prompt Build<br/>_nl_build_system_prompt()<br/>~6,935 tokens static"]
        SI_EMBED["Embedding Provider Detection<br/>_detect_embedding_provider()<br/>Ollama > OpenAI > Gemini"]
        SI_LEARN["Load Persistent Learning<br/>_load_recent_failures() (15 max)<br/>_load_recent_successes() (10 max)<br/>from JSONL databases"]
        SI_TOPO --> SI_SCHEMA --> SI_PROMPT
        SI_EMBED
        SI_LEARN
    end

    subgraph PerQuery["Per-Query Pipeline"]
        direction TB

        Q_INPUT["User Query Input<br/>+ wake word detection<br/>+ voice transcription (optional)"]

        subgraph PromptBuild["Prompt Construction (~10,750 tokens typical)"]
            direction TB
            PB_BASE["Cached Base Prompt<br/>~6,935 tokens"]
            PB_PERSONALITY["Personality Mode<br/>(friendly/professional/balanced)<br/>~75 tokens"]
            PB_LEARNING["Learning Context Injection<br/>persistent failures + successes<br/>~750-2,250 tokens"]
            PB_BASE --> PB_PERSONALITY --> PB_LEARNING
        end

        subgraph Enrichment["Tool Enrichment Phase (timed)"]
            direction TB
            subgraph ToolMatch["Combined Tool Matching"]
                direction LR
                TM_REGEX["Regex Keyword Matching<br/>regex_keywords.tsv (TSV)<br/>bash [[ =~ ]] built-in<br/>~1ms, no subprocesses"]
                TM_SEMANTIC["Semantic Embedding Match<br/>cosine similarity via jq<br/>against pre-computed embeddings<br/>(768/1536/3072 dim)"]
                TM_COMBINE["Combine + Deduplicate<br/>semantic priority, regex supplements<br/>top 7 (cloud) / top 5 (Ollama)"]
                TM_REGEX --> TM_COMBINE
                TM_SEMANTIC --> TM_COMBINE
            end
            EN_HELP["Append Tool --help Output<br/>parallel fetch (background jobs)<br/>~3,265 tokens for 7 tools"]
            EN_DOC["Doc RAG Retrieval<br/>_doc_rag_retrieve()<br/>cosine similarity on doc chunks<br/>top 3, min 0.45 similarity<br/>~1,045 tokens"]
            ToolMatch --> EN_HELP --> EN_DOC
        end

        subgraph AgentLoop["Agent Loop (max 3 iterations)"]
            direction TB
            subgraph LLMCall["LLM API Call"]
                direction TB
                AC_HISTORY["Build Messages Array<br/>conversation history (10 turns max)<br/>+ context trimming if over budget"]
                AC_SCHEMA_LOAD["Load MCP Tool Schemas<br/>_nl_load_tool_schemas()<br/>(when NL_NATIVE_TOOLS=1)"]
                AC_DISPATCH["Provider Dispatcher<br/>_call_llm_api()<br/>with retry + exponential backoff"]
                AC_EXTRACT["Response Extraction<br/>_extract_*_response()<br/>per-provider JSON parsing<br/>or native tool call conversion"]
                AC_HISTORY --> AC_SCHEMA_LOAD --> AC_DISPATCH --> AC_EXTRACT
            end

            subgraph Validation["5-Layer Security Validation"]
                direction TB
                V1["1. Whitelist Check<br/>cr_* tools + cockroach sql/workload/node/debug<br/>rejects path-qualified commands"]
                V2["2. Metachar Block<br/>pipes, redirects, $(), backticks,<br/>semicolons, &&, newlines"]
                V3["3. SQL Mutation Guard<br/>DROP, DELETE, TRUNCATE, UPDATE,<br/>INSERT, ALTER, GRANT, REVOKE, -f"]
                V4["4. RBAC Check<br/>admin / dba / analyst / monitor<br/>per-tool access control"]
                V5["5. Tool Existence Check<br/>verify cr_* exists in tools_dir<br/>and is executable"]
                V1 --> V2 --> V3 --> V4 --> V5
            end

            EXEC["Safe Execution<br/>_safe_execute_command()<br/>array-based invocation<br/>output capture + PII masking"]

            subgraph Followup["Follow-up Decision"]
                direction TB
                FU_CHECK{"needs_followup?<br/>or command failed?"}
                FU_REFLEXION["Reflexion: Self-Correction<br/>error analysis prompt<br/>'analyze what went wrong'<br/>'do NOT repeat same command'"]
                FU_CONTINUE["Multi-step Follow-up<br/>output fed back to LLM<br/>re-enrichment with reasoning"]
                FU_DONE["Done - Exit Loop"]
                FU_CHECK -->|"true (LLM requested)"| FU_CONTINUE
                FU_CHECK -->|"true (failure + reflexion)"| FU_REFLEXION
                FU_CHECK -->|"false"| FU_DONE
            end

            LLMCall --> Validation --> EXEC --> Followup
            FU_REFLEXION -->|"next iteration"| LLMCall
            FU_CONTINUE -->|"next iteration"| LLMCall
        end

        subgraph PostQuery["Post-Query Processing"]
            direction TB
            PQ_COST["Cost Tracking<br/>_llm_calculate_cost_cents()<br/>integer millicents, per-model pricing<br/>session accumulation + budget check"]
            PQ_LOG["Metrics Logging<br/>_log_nl() to CSV<br/>provider, tokens, timing, complexity,<br/>match method, enriched tools"]
            PQ_FEEDBACK["User Feedback Collection<br/>positive -> _save_success_persistent()<br/>negative -> _save_failure_persistent()<br/>with error_output (ANSI-stripped, 500 char)"]
            PQ_HISTORY["Conversation History Update<br/>append user + assistant messages<br/>FIFO eviction at 10 turns"]
            PQ_COST --> PQ_LOG --> PQ_FEEDBACK --> PQ_HISTORY
        end

        Q_INPUT --> PromptBuild --> Enrichment --> AgentLoop --> PostQuery
    end

    Entry --> SessionInit --> PerQuery

    subgraph Providers["LLM Provider Abstraction"]
        direction TB
        P_DISPATCH["_call_llm_api() dispatcher<br/>+ retry with backoff"]
        subgraph ProviderPlugins["Provider Plugins (src/lib/providers/)"]
            direction LR
            PP_ANTH["anthropic.sh<br/>Claude 3.5/4<br/>200K context"]
            PP_OAI["openai.sh<br/>GPT-4o / GPT-5 / o3/o4<br/>128K context"]
            PP_GEM["gemini.sh<br/>Gemini 2.0/2.5<br/>1M context"]
            PP_VTX["vertex.sh<br/>Claude via Vertex AI<br/>200K context"]
            PP_OLL["ollama.sh<br/>roachie-8b/3b, llama3<br/>4K context (compact prompt)"]
        end
        P_DISPATCH --> ProviderPlugins
    end

    subgraph EmbedProviders["Embedding Providers"]
        direction LR
        EP_OLL["Ollama<br/>nomic-embed-text<br/>768-dim"]
        EP_OAI["OpenAI<br/>text-embedding-3-small<br/>1536-dim"]
        EP_GEM["Gemini<br/>gemini-embedding-001<br/>3072-dim"]
    end

    subgraph Storage["Persistent Storage"]
        direction LR
        ST_FAIL["failures.jsonl<br/>cross-session learning<br/>(query, reasoning, cmd, issue, error_output)"]
        ST_SUCC["successes.jsonl<br/>positive patterns<br/>(query, command, description)"]
        ST_CSV["feedback.csv<br/>all query outcomes"]
        ST_METRICS["metrics CSV<br/>provider, tokens, timing,<br/>cost, complexity"]
    end

    subgraph Embeddings["Pre-Computed Embeddings"]
        direction LR
        EMB_TOOLS["tools/embeddings/<br/>3 provider files<br/>77 tool embeddings each"]
        EMB_DOCS["tools/embeddings/docs/<br/>2 provider files (14 chunks)<br/>CockroachDB documentation"]
    end

    subgraph Tools["77 cr_* Tools (tools/current/)"]
        direction LR
        T_CAT1["Schema/DDL (12)"]
        T_CAT2["Performance (11)"]
        T_CAT3["Security (8)"]
        T_CAT4["Tables/Size (5)"]
        T_CAT5["Data Ops (6)"]
        T_CAT6["+ 35 more"]
    end

    AC_DISPATCH --> Providers
    TM_SEMANTIC --> EmbedProviders
    TM_SEMANTIC --> Embeddings
    EN_DOC --> EmbedProviders
    EN_DOC --> Embeddings
    PQ_FEEDBACK --> Storage
    PQ_LOG --> Storage
    SI_LEARN --> Storage
    EXEC --> Tools

    classDef entry fill:#e1f5fe,stroke:#01579b,color:#000
    classDef security fill:#fce4ec,stroke:#b71c1c,color:#000
    classDef llm fill:#f3e5f5,stroke:#4a148c,color:#000
    classDef storage fill:#fff3e0,stroke:#e65100,color:#000
    classDef tools fill:#e8f5e9,stroke:#1b5e20,color:#000
    classDef enrichment fill:#fffde7,stroke:#f57f17,color:#000

    class RN,RB,RM entry
    class V1,V2,V3,V4,V5 security
    class PP_ANTH,PP_OAI,PP_GEM,PP_VTX,PP_OLL,AC_DISPATCH,P_DISPATCH llm
    class ST_FAIL,ST_SUCC,ST_CSV,ST_METRICS storage
    class T_CAT1,T_CAT2,T_CAT3,T_CAT4,T_CAT5,T_CAT6 tools
    class TM_REGEX,TM_SEMANTIC,TM_COMBINE,EN_HELP,EN_DOC enrichment
```

## Module Dependency Graph

```mermaid
flowchart LR
    subgraph Core["Core Modules"]
        CONFIG["llm_config.sh<br/>47 lines<br/>constants + env overrides"]
        METRICS["llm_metrics.sh<br/>692 lines<br/>logging, CSV, JSONL, cost"]
        EXEC["llm_exec.sh<br/>242 lines<br/>whitelist, metachar, SQL guard, RBAC"]
    end

    subgraph Pipeline["Pipeline Modules"]
        PROMPT["llm_prompt.sh<br/>1,807 lines<br/>prompt build, tool matching,<br/>doc RAG, schema fetch"]
        PROVIDERS["llm_providers.sh<br/>848 lines<br/>dispatcher, retry, model select"]
        ASSISTANT["llm_assistant.sh<br/>1,585 lines<br/>REPL, agent loop, execution"]
    end

    subgraph ProviderPlugins["Provider Plugins"]
        P_A["anthropic.sh"]
        P_O["openai.sh"]
        P_G["gemini.sh"]
        P_V["vertex.sh"]
        P_L["ollama.sh"]
    end

    subgraph Optional["Optional Modules"]
        VOICE["llm_voice.sh<br/>247 lines<br/>speech-to-text"]
        INIT["llm_init.sh<br/>setup + detection"]
    end

    CONFIG --> METRICS
    CONFIG --> EXEC
    CONFIG --> PROMPT
    CONFIG --> PROVIDERS
    METRICS --> PROMPT
    METRICS --> PROVIDERS
    METRICS --> ASSISTANT
    EXEC --> ASSISTANT
    PROMPT --> ASSISTANT
    PROVIDERS --> ASSISTANT
    VOICE --> ASSISTANT

    PROVIDERS --> P_A
    PROVIDERS --> P_O
    PROVIDERS --> P_G
    PROVIDERS --> P_V
    PROVIDERS --> P_L

    ASSISTANT --> ROACHMAN["bin/roachman"]
    ASSISTANT --> ROACHIE_NL["bin/roachie-nl"]
    ASSISTANT --> ROACHIE_BATCH["bin/roachie-batch"]

    classDef core fill:#e3f2fd,stroke:#1565c0,color:#000
    classDef pipeline fill:#f3e5f5,stroke:#6a1b9a,color:#000
    classDef plugin fill:#e8f5e9,stroke:#2e7d32,color:#000
    classDef optional fill:#fff8e1,stroke:#f9a825,color:#000

    class CONFIG,METRICS,EXEC core
    class PROMPT,PROVIDERS,ASSISTANT pipeline
    class P_A,P_O,P_G,P_V,P_L plugin
    class VOICE,INIT optional
```

## Data Flow: Query to Command Execution

```mermaid
sequenceDiagram
    participant U as User
    participant R as REPL Loop<br/>(llm_assistant.sh)
    participant P as Prompt Builder<br/>(llm_prompt.sh)
    participant E as Enrichment<br/>(llm_prompt.sh)
    participant API as LLM Provider<br/>(llm_providers.sh)
    participant V as Validator<br/>(llm_exec.sh)
    participant T as cr_* Tool<br/>(tools/current/)
    participant S as Storage<br/>(llm_metrics.sh)

    U->>R: "show slow queries on cluster A"
    R->>P: _nl_build_prompt(cached_base)
    P-->>R: system_prompt + personality + learning

    R->>E: _enrich_system_prompt_with_tool_help(query, prompt)
    Note over E: Regex match: "slow|queries" -> cr_query_stats
    Note over E: Semantic match: cosine similarity -> cr_query_stats, cr_plan
    E->>E: Fetch --help for matched tools (parallel)
    E->>E: Doc RAG: retrieve relevant doc chunks
    E-->>R: enriched system_prompt (~10,750 tokens)

    loop Agent Loop (max 3 iterations)
        R->>API: _call_llm_api(provider, prompt, messages, temp)
        Note over API: Provider dispatch + retry
        API-->>R: JSON response {reasoning, commands[], needs_followup}

        loop For each command
            R->>V: _validate_command(cmd, tools_dir)
            Note over V: 1. Whitelist  2. Metachar<br/>3. SQL mutation  4. RBAC<br/>5. Tool exists
            V-->>R: allowed / blocked

            alt Command allowed
                R->>U: "Run this command? [y/N]"
                U-->>R: y
                R->>T: _safe_execute_command(cmd)
                T-->>R: output (captured + PII masked)
                R->>U: Display output
            end
        end

        alt needs_followup = true OR command failed
            Note over R: Feed output back to LLM<br/>(reflexion if failure)
        else done
            Note over R: Exit agent loop
        end
    end

    R->>S: _log_nl(provider, tokens, timing, cost)
    R->>U: "Was this helpful? [y/n]"
    U-->>R: n (with comments)
    R->>S: _save_failure_persistent(query, reasoning, cmd, issue, error_output)
```

## Token Budget Composition

```mermaid
pie title System Prompt Token Budget (Typical Query ~10,750 tokens)
    "Tool Catalog (77 tools)" : 2125
    "Execution Rules + Response Format" : 2558
    "Parameter Rules" : 1043
    "Per-Tool Flag Notes" : 910
    "Hallucination Warnings" : 300
    "Tool --help (5 matched)" : 2330
    "Doc RAG (3 chunks)" : 1045
    "Persistent Learning" : 750
    "Schema Context (3 DBs)" : 475
    "Intro + Topology" : 125
```

## Security Architecture

```mermaid
flowchart TB
    subgraph Input["Input Sanitization"]
        IS1["_sanitize_for_prompt()<br/>control char stripping<br/>length truncation"]
        IS2["Wake word stripping<br/>clean_input extraction"]
    end

    subgraph PromptSec["Prompt Security"]
        PS1["Metachar rule in system prompt<br/>no $(), backticks, pipes, redirects"]
        PS2["Data delimiters on learning<br/>&lt;DATA type=historical_failure_log&gt;<br/>prevents prompt injection"]
        PS3["Schema name sanitization<br/>tr -cd 'A-Za-z0-9_-'"]
    end

    subgraph ExecSec["Execution Security (5 layers)"]
        ES1["Whitelist<br/>cr_* + cockroach sql/workload/node/debug"]
        ES2["Metachar Block<br/>;  |  &&  ||  $()  backtick  > < \\n"]
        ES3["SQL Mutation Guard<br/>DROP DELETE TRUNCATE UPDATE INSERT<br/>ALTER GRANT REVOKE + -f/--file block"]
        ES4["RBAC Access Control<br/>admin: all<br/>dba: no cluster create/remove<br/>analyst: read-only<br/>monitor: health/monitoring only"]
        ES5["Tool Existence Verification<br/>-x tools_dir/cr_*"]
    end

    subgraph OutputSec["Output Security"]
        OS1["PII Masking<br/>_mask_sensitive_output()<br/>first 500 chars of output"]
        OS2["Error Output Sanitization<br/>ANSI stripping + 500 char limit<br/>before storage in failure DB"]
        OS3["Temp File Security<br/>umask 0077 before mktemp"]
    end

    LLM_RESPONSE["LLM Response"] --> ES1 --> ES2 --> ES3 --> ES4 --> ES5 --> EXECUTE["Safe Execution"]
    EXECUTE --> OS1 --> DISPLAY["User Display"]
    EXECUTE --> OS2 --> STORE["Persistent Storage"]

    classDef security fill:#ffebee,stroke:#c62828,color:#000
    class ES1,ES2,ES3,ES4,ES5 security
```

## File Reference

| Module | Lines | Role in Pipeline |
|--------|------:|------------------|
| `llm_config.sh` | 47 | Constants with env var overrides |
| `llm_init.sh` | ~150 | Provider/model setup, first-run detection |
| `llm_prompt.sh` | 1,807 | Prompt construction, tool matching (regex+semantic), doc RAG, schema fetch |
| `llm_providers.sh` | 848 | Provider dispatch, retry, model selection, native tool call conversion |
| `llm_assistant.sh` | 1,585 | REPL loop, agent loop, command execution, feedback, cost tracking |
| `llm_exec.sh` | 242 | Whitelist, metachar, SQL mutation, RBAC validation |
| `llm_metrics.sh` | 692 | CSV/JSONL logging, cost calculation, persistent failure/success DBs |
| `llm_voice.sh` | 247 | Speech-to-text transcription |
| `providers/anthropic.sh` | ~200 | Anthropic Claude API (streaming + non-streaming) |
| `providers/openai.sh` | ~250 | OpenAI GPT/o-series API (streaming, tool calling) |
| `providers/gemini.sh` | ~350 | Google Gemini API (thinking budget, tool calling) |
| `providers/vertex.sh` | ~200 | Vertex AI Claude (gcloud auth) |
| `providers/ollama.sh` | ~250 | Local Ollama models (compact prompt path) |
| **Total** | **~6,650** | |

---

*Diagrams rendered with Mermaid. View in any Mermaid-compatible renderer (GitHub, VS Code, mermaid.live).*
