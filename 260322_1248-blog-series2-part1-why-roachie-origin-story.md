# Why Build a Natural Language Interface for CockroachDB? The Origin Story Behind Roachie

*Part 1 of 2 — From 77 bash scripts to a conversational DBA assistant, and why the gap between CLI power and GUI simplicity needed bridging.*

---

## The Problem: Database Administration is Split in Two

Database tooling has always had a split personality.

On one side: **raw power**. SQL consoles, CLI tools, `cockroach sql`, shell scripts. Full control, full flexibility, steep learning curve. A DBA who knows the right query can extract anything from `crdb_internal` — slow queries, lock contention, replication lag, schema diffs. But knowing the right query is the hard part.

On the other side: **simplicity**. GUI dashboards, DB Console, monitoring UIs. Point and click, see pretty charts, get a high-level view. But the moment something goes wrong — a migration fails, a table has skewed data distribution, a changefeed is lagging — the dashboard can't help. It's back to the CLI.

The question that started Roachie: **what if there was something in between?**

Not a GUI. Not a raw SQL console. A natural language interface that understands CockroachDB — one where a DBA could type "show me the top 5 slowest queries on the VA tenant" and get the actual answer, not a tutorial on how to write the query.

## The Journey: It Started with Bash Scripts

Roachie didn't start as an AI project. It started as a collection of bash scripts.

Working with CockroachDB's self-hosted multi-tenant deployments meant running the same complex queries repeatedly — checking replication lag, comparing schemas between databases, finding who has access to what, generating DDL for migration. Each query involved remembering the right `crdb_internal` tables, the right joins, the right flags.

So those queries became scripts. `cr_query_stats` for slow queries. `cr_tables` for table listings. `cr_ddl_diff` for schema comparison. `cr_replication_lag` for PCR health. Each one a focused bash tool that wraps a complex SQL query and returns structured, actionable output.

Over time, 77 of these `cr_*` tools accumulated, organized into categories:

| Category | What It Does |
|----------|-------------|
| Schema & DDL Analysis | Generate CREATE statements, compare schemas between databases/clusters, analyze constraints and dependencies |
| Security & Access Control | List user permissions, detect ACL inconsistencies, generate GRANT statements |
| Performance & Monitoring | Identify slow queries, track replication lag, monitor wait events and lock contention |
| Data Operations | Backup, restore, migrate, clone, unload data |
| Cluster Management | Rolling upgrades, Ceph/S3 storage, Kafka CDC, Prometheus/Grafana |

Each tool is self-contained — runs anywhere with bash, `cockroach`, `jq`, and `curl`. No Python dependencies, no npm packages, no compilation step. Copy the scripts to a server and they work.

## The Infrastructure Layer: Multi-Tenant Clusters with Peripherals

The tools needed an environment to run against. CockroachDB's self-hosted multi-tenant architecture — physical clusters with virtual clusters (tenants) inside — is powerful but complex. Setting up a proper test environment involves:

- **Cluster A** (primary) with multiple virtual clusters (va, vb, system)
- **Cluster B** (standby) receiving Physical Cluster Replication (PCR) from A
- **Ceph/S3GW** for backup storage
- **Kafka** for CDC changefeeds
- **Prometheus + Grafana** for monitoring
- **HAProxy** for load balancing

Managing all of this manually is tedious. So `roachman` was built — an interactive cluster manager that handles the full lifecycle: create clusters, configure tenants, start replication, set up monitoring, run upgrades, and manage failovers. All containerized, all reproducible.

At this point, the toolkit was useful but had the same limitation as every CLI tool: you had to know which tool to use and what flags to pass. `cr_ddl_table -h 10.1.1.5 -p 26257 -t va -d movr products` is powerful, but it assumes knowledge of the tool name, flag syntax, hostname, port, and tenant name.

## The Bridge: Natural Language Processing

The idea was straightforward: let the LLM figure out which tool to run.

A DBA types "show me the DDL for the products table in movr on the VA tenant." The system should:

1. Identify that `cr_ddl_table` is the right tool
2. Know that the cluster's host is 10.1.1.5, port 26257
3. Pass the right flags: `-h 10.1.1.5 -p 26257 -t va -d movr products`
4. Execute the command and show the result

Simple in concept. Hard in practice. The challenges:

### Challenge 1: Tool Selection Across 77 Options

Given a natural language query, which of 77 tools is the right one? "Show me table sizes" could be `cr_size`, `cr_db_size`, `cr_tables`, or `cr_db_tables_rowcount`. "Check the health" could be `cr_health`, `cr_health_check`, `cr_monitor`, or `cr_watch`.

The solution: **hybrid tool matching**. A combination of regex keyword scoring and semantic embedding similarity, using a two-model architecture:

- **Embedding model** (small, fast) computes similarity between the query and pre-computed tool description embeddings
- **Regex** acts as an intent gate — if regex finds nothing, semantic matching is skipped (avoids false positives on unrelated queries)
- Results are merged: semantic matches get priority, regex supplements, deduplicated, top 7

Three embedding providers are supported (Ollama 768-dim, OpenAI 1536-dim, Gemini 3072-dim), and the combined approach hits 99-100% tool selection accuracy across 135 test queries.

### Challenge 2: Security

Letting an LLM generate shell commands is dangerous. The system needed defense-in-depth:

1. **Command whitelist** — only `cr_*` tools and `cockroach sql` are allowed
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

This was a firm design requirement. Many database environments have no internet access. The system had to work entirely offline with a local model.

Ollama with Llama 3.1 8B running on a MacBook meets this requirement — no API keys, no network calls, no data leaving the machine. The embedding model (nomic-embed-text) also runs locally. Even voice input works offline via whisper.cpp.

But local models are less accurate than cloud models. Gemini hits 95-100% on the test suite. Llama 8B started at 50-60%. Bridging that gap became its own project — covered in Part 2.

## The Architecture Today

After months of iteration, the system has six layers:

```
Layer 1: Input & Interface
  CLI / Voice (Whisper STT) / Batch mode

Layer 2: Hybrid Tool Matching
  Regex pre-filter → Semantic embeddings → Merge & deduplicate → Top 7

Layer 3: Prompt Assembly & LLM
  Schema context + Persistent learning + Topology + Templates → LLM API

Layer 4: Security & Execution
  JSON parse → 6-layer security gates → Human confirmation → Execute

Layer 5: CockroachDB Operations
  77 cr_* tools → Sandboxed execution → Cluster

Layer 6: Output, Metrics & Learning
  Display results → Log metrics (tokens, latency, accuracy) → User feedback
```

It also integrates with the broader ecosystem via **MCP (Model Context Protocol)** — 39 of the 77 tools are exposed as MCP-compatible SQL endpoints through Google's GenAI Toolbox. This means Claude Desktop, Cursor, VS Code Copilot, or any LangChain agent can use Roachie's DBA tools directly.

## Why Bash?

A common question: why build all of this in bash instead of Python or Go?

**Portability.** Bash is on every Linux server, every Mac, every container. No runtime to install, no dependencies to manage, no virtual environments to activate. Copy the scripts to a server with `cockroach` and `jq` installed, and everything works. In air-gapped environments where installing Python packages requires security review, this matters.

**Composability.** Each `cr_*` tool is a standalone script. It can be run directly, piped to other tools, called from cron jobs, or orchestrated by the NL interface. The tools don't know or care that an LLM chose them.

**Transparency.** When a DBA asks "what did that command actually do?", the answer is a readable bash script, not a compiled binary or a Python class hierarchy. The SQL queries are right there in the source.

It's not without trade-offs. Bash has no native JSON parsing (hence `jq`), no real data structures beyond arrays, and error handling is primitive. But for wrapping SQL queries and orchestrating CLI tools, it's hard to beat.

## What's Next

The NL interface works well with cloud LLMs — 95-100% accuracy on the test suite. But the local model story needed work. Llama 8B at 50-60% accuracy wasn't useful enough for daily DBA work.

Part 2 of this series covers the journey to make the local model competitive: semantic matching improvements, prompt unification, and LoRA fine-tuning that pushed accuracy from 50% to 90%.

---

*Part 2: [Making Open-Source LLMs Work for CLI Tool Selection: Llama 8B from 50% to 90%]()*

*Roachie is open source: [github.com/amaddahian/roachie](https://github.com/amaddahian/roachie)*
