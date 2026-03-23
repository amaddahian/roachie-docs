# What If You Could Talk to Your Database Tools?

*Part 1 of 2 — From bash scripts to a conversational DBA/SRE assistant, and why the gap between CLI power and GUI simplicity needed bridging.*

> **Disclaimer:** Roachie is a personal project. All trademarks referenced
> are the property of their respective owners. The software is provided
> "as-is" without warranty of any kind. Use at your own risk.

---

## The Problem: Database Administration is Split in Two

Database tooling has always had a split personality.

On one side: **raw power**. SQL consoles, CLI tools, shell scripts. Full control, full flexibility, steep learning curve. A DBA/SRE who knows the right query can extract anything from the catalog — slow queries, lock contention, replication lag, schema diffs. But knowing the right query might also involve access to internal system tables for advanced-feature support scenarios which could be a bit more involved.

On the other side: **simplicity**. GUI dashboards, built-in monitoring UIs. Point and click, see appropriate charts, and will get a high-level view.

But the moment something goes wrong and specially involving more advanced issues —such as changefeed lags, complex job lifecycle failures, or hard-to-attribute latency regressions—the dashboard alone might not sufficient for effective diagnosis and it's back to the CLI.

The question that started Roachie: **what if there was something in between?**

![The NLP Bridge — connecting CLI power to conversational simplicity](https://github.com/amaddahian/roachie/blob/main/docs/images/nlp_bridge.png?raw=true)

Not a GUI. Not a raw SQL console. A natural language interface that understands the database — one where a DBA/SRE could type "show me the top 5 slowest queries on the VA tenant" and get the actual answer, not a tutorial on how to write the query.

## The Journey: It Started with Bash Scripts

Roachie didn't originally start as an AI project. It started as a collection of bash scripts.

Working with self-hosted multi-tenant database deployments meant running the same complex queries repeatedly — checking replication lag, comparing schemas between databases, finding who has access to what, generating DDLs, checking for range distributions and/or other distributed attributes of the overall system. Each query involved remembering the right internal system tables, the right joins, the right flags.

So those queries became scripts. `cr_query_stats` for slow queries. `cr_tables` for table listings. `cr_ddl_diff` for schema comparison. `cr_replication_lag` for PCR health. Each one a focused bash tool that wraps a complex SQL query and returns structured, actionable output.

To accommodate this, 70+ of these `cr_*` tools accumulated, organized into categories:

| Category | What It Does |
|----------|-------------|
| Schema & DDL Analysis | Generate CREATE statements, compare schemas between databases/clusters, analyze constraints and dependencies |
| Security & Access Control | List user permissions, detect ACL inconsistencies, generate GRANT statements |
| Performance & Monitoring | Identify slow queries, track replication lag, monitor wait events and lock contention |
| Data Operations | Backup, restore, migrate, clone, unload data |
| Cluster Management | Rolling upgrades, Ceph/S3 storage, Kafka CDC, Prometheus/Grafana |

The full tool categories are meant to span the entire spectrum of DBA/SRE operations:

Each tool is self-contained — runs anywhere with bash, the database CLI, `jq`, and `curl`. No Python dependencies, no npm packages, no compilation step. Copy the scripts to a server and they work.

## The Infrastructure Layer: Example Multi-Tenant Clusters with Peripherals

The tools needed an environment to run against. A self-hosted multi-tenant distributed SQL architecture — physical clusters with virtual clusters (tenants) inside — is powerful and rich for this demo. Setting up a proper test environment involves:

- **Cluster A** (primary) with multiple virtual clusters (va, vb, system)
- **Cluster B** (standby) receiving Physical Cluster Replication (PCR) from A
- **Ceph/S3GW** for backup storage
- **Kafka** for CDC changefeeds
- **Prometheus + Grafana** for monitoring
- **HAProxy** for load balancing

Managing all of this manually is tedious. So `roachman` was built — an interactive cluster manager that handles the full lifecycle: create clusters, configure tenants, start replication, set up monitoring, run upgrades, and manage failovers. All containerized, all reproducible.

![Infrastructure Architecture — containerized database clusters with full peripheral integration](https://github.com/amaddahian/roachie/blob/main/docs/images/infrastructure.png?raw=true)

At this point, the toolkit was useful but had the same limitation as every CLI tool: you had to know which tool to use and what flags to pass. `cr_ddl_table -h 10.1.1.5 -p 26257 -t va -d movr products` is powerful, but it assumes knowledge of the tool name, flag syntax, hostname, port, and tenant name.

## The Bridge: Natural Language Processing

The idea was straightforward: let the LLM figure out which tool to run.

A DBA/SRE types "show me the DDL for the products table in movr on the VA tenant." The system should:

1. Identify that `cr_ddl_table` is the right tool
2. Know that the cluster's host is 10.1.1.5, port 26257
3. Pass the right flags: `-h 10.1.1.5 -p 26257 -t va -d movr products`
4. Execute the command and show the result

Simple in concept. Hard in practice. The challenges:

### Challenge 1: Tool Selection Across 77 Options

Given a natural language query, which of 77 tools is the right one? As an example, "Show me table sizes" or "Check the health" could potentially map to different tools. 

The solution: **hybrid tool matching**. A combination of regex keyword scoring and semantic embedding similarity, using a two-model architecture:

- **Regex** acts as an intent gate — if regex finds nothing, semantic matching is skipped (avoids false positives on unrelated queries)
- **Embedding model** (small, fast) computes similarity between the query and pre-computed tool description embeddings
- Results are merged: semantic matches get priority, regex supplements, deduplicated, top 7

Three embedding providers are supported (Ollama 768-dim, OpenAI 1536-dim, Gemini 3072-dim), and the combined approach hits 99-100% tool selection accuracy across 135 test queries.

### Challenge 2: Security

Letting an LLM generate shell commands is dangerous. The system needed defense-in-depth:

1. **Command whitelist** — only `cr_*` tools and the database CLI are allowed
2. **Metacharacter blocking** — no pipes, redirects, backticks, or command substitution
3. **SQL mutation guard** — blocks DROP, DELETE, GRANT, ALTER in SQL commands (configurable by role)
4. **RBAC** — four roles (admin, dba, analyst, monitor) with different permissions
5. **Human-in-the-loop** — user confirms before execution (optional)
6. **Tool existence check** — verifies the tool actually exists on disk

This isn't a "please don't be malicious" prompt instruction. It's enforced in code, with 113 security tests validating the boundaries.

### Challenge 3: Provider Abstraction

Different teams use different LLM providers. Some have OpenAI API keys, others use Google's Gemini, others want everything local. The NL interface needed to work with all of them interchangeably.

Five providers are supported through a unified dispatcher:

| Provider | Model | Use Case |
|----------|-------|----------|
| Anthropic Claude | Sonnet/Opus | Highest accuracy, streaming |
| OpenAI | GPT-5, o-series | Enterprise standard |
| Google Gemini | 2.0 Flash | Fast, cost-effective |
| Vertex AI | Claude via GCP | Enterprise GCP environments |
| Ollama | Llama 3.1 8B (local) | Offline, air-gapped, free |

Switching providers is a single flag: `--provider gemini` or `--provider ollama-roachie-8b`. The prompt construction, tool matching, validation, and execution pipeline is identical regardless of which LLM generates the response.

### Challenge 4: Context Grounding

LLMs hallucinate. Especially for tool flags. The model might generate `cr_tables --database movr` when the correct flag is `-d movr`. Or it might invent a flag that doesn't exist.

The solution is aggressive context injection:

- **Schema-aware prompts** — on session start, query the cluster for actual databases and tables, inject them into the system prompt
- **Tool enrichment** — for the top 7 matched tools, inject their actual `--help` output into the prompt (the model sees the real flag definitions)
- **Cluster topology** — inject hostnames, ports, tenant names, connection parameters
- **Few-shot examples** — curated examples of correct tool usage in the prompt templates
- **Persistent learning** — track successes and failures across sessions, inject past failures as "avoid this" context

### Challenge 5: Offline and Air-Gapped Operation

This was a firm design requirement - air-gapped. The system had to work entirely offline if necessary with a local model.

Ollama with Llama 3.1 8B running on a MacBook meets this requirement — no API keys, no network calls, no data leaving the machine. The embedding model (nomic-embed-text) also runs locally. Even voice input works offline via whisper.cpp.

But local models are less accurate than cloud models. Gemini hits 95-100% on the test suite. Llama 8B started at 50-60%. Bridging that gap became its own project — covered in Part 2.

## The Architecture Today

After months of iteration, the system has six layers:

![AI Architecture — layered NL processing pipeline](https://github.com/amaddahian/roachie/blob/main/docs/images/ai_architecture.png?raw=true)

```
Layer 1: Input & Interface
  CLI / Voice (Whisper STT) / Batch mode

Layer 2: Hybrid Tool Matching
  Regex pre-filter → Semantic embeddings → Merge & deduplicate → Top 7

Layer 3: Prompt Assembly & LLM
  Schema context + Persistent learning + Topology + Templates → LLM API

Layer 4: Security & Execution
  JSON parse → 6-layer security gates → Human confirmation → Execute

Layer 5: Database Operations
  77 cr_* tools → Sandboxed execution → Cluster

Layer 6: Output, Metrics & Learning
  Display results → Log metrics (tokens, latency, accuracy) → User feedback
```

Three embedding providers power the hybrid matching layer: Ollama nomic-embed-text (768-dim), OpenAI text-embedding-3-small (1536-dim), and Gemini gemini-embedding-001 (3072-dim). Combined with regex, accuracy reaches 99-100% across a 135-query test suite.

The system also supports **voice input** via Whisper speech-to-text — three options: OpenAI's cloud API (~$0.006/min), whisper.cpp (local, free), or Python Whisper (local, free). Say "Roachie" as the wake word, and the rest is natural language.

It also integrates with the broader ecosystem via **MCP (Model Context Protocol)** — 39 of the 77 tools are exposed as MCP-compatible SQL endpoints through Google's GenAI Toolbox. This means Claude Desktop, Cursor, VS Code Copilot, or any LangChain agent can use Roachie's DBA/SRE tools directly.

## Why Bash?

A common question: why build all of this in bash instead of Python or Go?

**Portability.** Bash is on every Linux server, every Mac, every container. No runtime to install, no dependencies to manage, no virtual environments to activate. Copy the scripts to a server with the database CLI and `jq` installed, and everything works. In air-gapped environments where installing Python packages requires security review, this matters.

**Composability.** Each `cr_*` tool is a standalone script. It can be run directly, piped to other tools, called from cron jobs, or orchestrated by the NL interface. The tools don't know or care that an LLM chose them.

**Transparency.** When a DBA/SRE asks "what did that command actually do?", the answer is a readable bash script, not a compiled binary or a Python class hierarchy. The SQL queries are right there in the source.

It's not without trade-offs. Bash has no native JSON parsing (hence `jq`), no real data structures beyond arrays, and error handling is primitive. But for wrapping SQL queries and orchestrating CLI tools, it's hard to beat.

## What It Looks Like in Practice

![Sample prompts and responses](https://github.com/amaddahian/roachie/blob/main/docs/images/sample_prompts.png?raw=true)

A few examples of what can be asked in plain English:

- *"Show me the top 3 long running queries on cluster A system tenant"*
- *"List all tables in the test_lab database in system tenant on cluster A in inventory_schema schema"*
- *"Show me the DDL diff between db1 and db2 databases on system tenant on cluster A"*
- *"Get the ACL for the products table in system tenant on cluster A in database test_lab"*
- *"Backup database movr to crdb-test-backups bucket on system tenant on cluster A"*
- *"What is the current PCR replication lag for the vb tenant on cluster B?"*

Each query gets routed to the right `cr_*` tool with the correct flags, executed against the cluster, and the result displayed — all without the user needing to know tool names or flag syntax.

## What's Next

The NL interface works well with cloud LLMs — 95-100% accuracy on the test suite. But the local model story needed work. Llama 8B at 50-60% accuracy wasn't useful enough for daily DBA/SRE work.

Part 2 of this series covers the journey to make the local model competitive: semantic matching improvements, prompt unification, and LoRA fine-tuning that pushed accuracy from 50% to 90%.

---

*Part 2: [Making Open-Source LLMs Work for CLI Tool Selection: Llama 8B from 50% to 90%]()*
