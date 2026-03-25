# Roachie — Internal DBA Toolkit Overview

**Prepared for**: CockroachDB Labs Legal Review
**Date**: March 25, 2026
**Author**: Ali Maddahian

---

## What Is Roachie?

Roachie is an **internal command-line toolkit** for CockroachDB database administration. It wraps 77 bash-based DBA tools (`cr_*` commands) with a natural language interface, allowing DBAs to perform routine operational tasks by describing them in plain English instead of remembering complex CLI flags.

**It is not** a product, SaaS offering, or commercial software. It is an internal productivity tool used by our DBA team.

## What Does It Do?

| Category | Examples |
|----------|----------|
| Schema inspection | List tables, columns, views, DDL generation |
| Monitoring | Query performance stats, disk scan speeds, node health |
| Operations | Statistics generation, backup/restore, data migration |
| Cluster management | Replication lag, tenant listing, configuration history |
| Security | ACL inspection, user/role DDL, grant management |

A DBA types something like _"show me the DDL for the products table in test_lab"_ and roachie translates that into the correct `cr_ddl_table` command with the right flags, executes it, and returns the result.

## How Does It Work?

```
User prompt  -->  Tool Matching  -->  LLM generates command  -->  Safety checks  -->  Execute
                  (regex +                                       (whitelist,
                   embeddings)                                    metachar block,
                                                                  RBAC)
```

1. **User describes a task** in natural language
2. **Tool matching** identifies which of the 77 tools are relevant (semantic embeddings + keyword matching)
3. **An LLM** generates the correctly-flagged command based on tool documentation injected into the prompt
4. **Safety checks** validate the command against a whitelist, block shell metacharacters, and enforce role-based access
5. **The command executes** and results are returned to the user

## LLM Providers

Roachie supports multiple LLM backends. The DBA team can choose based on their security and performance needs:

| Provider | Data Residency | Use Case |
|----------|---------------|----------|
| **Ollama (local)** | All data stays on the DBA's machine | Air-gapped / sensitive environments |
| Anthropic Claude | Anthropic API (enterprise terms) | Higher accuracy when allowed |
| Google Gemini | Google AI API (enterprise terms) | Alternative cloud provider |
| OpenAI | OpenAI API (enterprise terms) | Alternative cloud provider |

**The Ollama path is fully offline** — no data, schema metadata, or query results leave the local machine. Fine-tuned models (Llama 8B and Mistral Nemo 12B) run locally with 90-97% accuracy on our DBA task benchmarks.

## What Data Flows to LLM Providers?

When using a cloud LLM provider (not Ollama):

| Data Sent | Example | Sensitive? |
|-----------|---------|------------|
| System prompt with tool docs | "cr_ddl_table accepts -d DATABASE TABLE_NAME..." | No |
| Database/table names | "test_lab", "products", "inventory_schema" | Low |
| User's natural language query | "show me the DDL for products table" | Low |
| Command output (optional, follow-up only) | Table DDL text, query stats | Moderate |

**Not sent**: Row-level data, credentials, connection strings, API keys. Command output feedback is optional (`--no-followup` disables it) and truncated to 4,000 characters when enabled.

## Security Controls

| Control | Implementation |
|---------|---------------|
| Command whitelist | Only approved `cr_*` tools and `cockroach sql` can execute |
| Shell injection prevention | Metacharacter blocking (`$()`, backticks, pipes, redirects) |
| Role-based access (RBAC) | 4 roles: admin, dba, analyst, monitor — each restricts available operations |
| Input validation | 2,000-character input limit, command length limits |
| No stored credentials | API keys via environment variables only |
| Temp file security | `umask 0077` on all temporary files |
| Test coverage | 113 security-specific tests, 500+ total unit tests |

## Relationship to CockroachDB

- Roachie is **built for** CockroachDB, not a modification or fork of CockroachDB
- It calls the standard `cockroach` CLI and connects via standard PostgreSQL protocol
- It does not embed, redistribute, or modify any CockroachDB source code
- The name "roachie" is an informal internal name, not intended to imply endorsement by or affiliation with Cockroach Labs
- It is comparable to third-party database management tools (like DBeaver for PostgreSQL or Toad for Oracle)

## Relationship to CockroachDB's Managed MCP Server

CockroachDB Cloud recently launched a managed MCP server for AI agent access. Roachie's scope is different:

| | CockroachDB Managed MCP | Roachie |
|---|---|---|
| Target | Cloud customers, AI agent integration | Internal DBA team, operational tasks |
| Scope | ~6 tools (schema exploration, SELECT, INSERT) | 77 DBA tools (DDL, backup, migrate, monitor, upgrade) |
| Deployment | Cloud-managed | Self-hosted / local |
| Auth | OAuth 2.1, Cloud RBAC | CockroachDB native auth (certs or insecure) |
| Clusters | CockroachDB Cloud only | Any CockroachDB cluster (self-hosted, on-prem, Cloud) |

The tools are complementary, not competing. Roachie addresses operational DBA workflows that the managed MCP server does not cover.

## Open Source Status

Roachie is currently hosted in a **private GitHub repository**. It is not published as open source software and is not distributed outside our organization.

---

**Questions?** Happy to walk through a live demo or discuss any specific area in more detail.
