# Building a CockroachDB DBA Assistant with Google GenAI Toolbox

Every DBA has a mental playbook: check who's connected, find the slow queries, verify
replication health, audit permissions. The queries are mostly the same — what changes
is the context. "Why is the app slow right now?" leads to the same diagnostic steps
whether it's Tuesday morning or Saturday at 2 AM.

What if an LLM could run that playbook for you — safely, against your live cluster,
using the exact SQL you'd write yourself?

This tutorial shows how to expose CockroachDB administration queries as MCP-compatible
tools using [Google GenAI Toolbox for Databases](https://googleapis.github.io/genai-toolbox/).
Once configured, any MCP client — Claude Desktop, Cursor, VS Code Copilot, or a custom
LangChain agent — can query your cluster through parameterized, read-only SQL.

## What We're Building

A `tools.yaml` configuration file that turns DBA queries into callable tools. No
application code, no middleware, no custom API. Just SQL statements with parameter
bindings that Toolbox serves over MCP.

Here's what the end result looks like in Claude Desktop:

> **You**: "Show me any slow queries from the last 2 hours"
>
> **Claude**: *calls cr-query-stats with period_hours=2, min_duration_sec=0.5*
>
> | query_text | latency_sec | rows_read | rows_written |
> |------------|-------------|-----------|--------------|
> | SELECT * FROM orders WHERE ... | 2.341 | 148,203 | 0 |
> | UPDATE inventory SET ... | 1.892 | 12 | 12 |

The LLM picks the right tool, fills in the parameters, and presents the results.
You never hand it raw SQL access — it can only call the tools you've defined.

## Prerequisites

- A running CockroachDB cluster (v22.2+, tested on v25.2)
- [GenAI Toolbox](https://googleapis.github.io/genai-toolbox/getting-started/introduction/) installed (`brew install googleapis/tap/toolbox`)
- An MCP client (Claude Desktop, Cursor, or similar)

## Step 1: Configure the Connection

Toolbox connects to CockroachDB using a source definition. Environment variables
keep credentials out of the config file:

```yaml
kind: sources
name: crdb
type: cockroachdb
host: ${CRDB_HOST:localhost}
port: ${CRDB_PORT:26257}
user: ${CRDB_USER:root}
password: ${CRDB_PASSWORD:}
database: ${CRDB_DATABASE:defaultdb}
queryParams:
  sslmode: ${CRDB_SSLMODE:disable}
  application_name: cockroachdb-toolbox
```

Set your environment before starting Toolbox:

```bash
export CRDB_HOST=localhost
export CRDB_PORT=26257
export CRDB_USER=root
export CRDB_SSLMODE=disable    # or 'require' for CockroachDB Cloud
```

### The System Tenant Source

CockroachDB supports virtual clusters (multi-tenancy). Some administrative queries —
node liveness, virtual cluster listing, event history — are only available from the
system tenant. Define a second source that routes to it:

```yaml
---
kind: sources
name: crdb-system
type: cockroachdb
host: ${CRDB_HOST:localhost}
port: ${CRDB_PORT:26257}
user: ${CRDB_USER:root}
password: ${CRDB_PASSWORD:}
database: ${CRDB_DATABASE:defaultdb}
queryParams:
  sslmode: ${CRDB_SSLMODE:disable}
  application_name: cockroachdb-toolbox
  options: -ccluster=system
```

The key is `options: -ccluster=system` — this tells CockroachDB to route the
connection to the system tenant. Tools that need this source reference `crdb-system`
instead of `crdb`.

## Step 2: Build Your First Tool — Cluster Version

Start simple. A tool with no parameters that returns the CockroachDB version:

```yaml
---
kind: tools
name: cr-health-version
type: cockroachdb-sql
source: crdb
description: >
  Get the CockroachDB version running on the cluster.
parameters: []
statement: |
  SELECT value AS version
  FROM crdb_internal.node_build_info
  WHERE field = 'Version'
```

Every tool needs:
- **name**: What clients call it (use lowercase with hyphens)
- **type**: Always `cockroachdb-sql` for CockroachDB
- **source**: Which connection to use
- **description**: The LLM reads this to decide when to use the tool
- **parameters**: Input bindings (empty array if none)
- **statement**: The SQL to execute

The `crdb_internal` schema is CockroachDB-specific — it exposes cluster metadata
that isn't available through standard `information_schema`. You'll use it extensively
for DBA tools.

Test it:

```bash
toolbox --tools-file tools.yaml --ui
# Open http://localhost:5000 and invoke cr-health-version
```

## Step 3: Add Parameterized Filtering — Active Sessions

Most DBA queries need filtering. Toolbox uses prepared statement parameters (`$1`, `$2`)
for safe parameter binding — no SQL injection risk:

```yaml
---
kind: tools
name: cr-session
type: cockroachdb-sql
source: crdb
description: >
  Show active database sessions with query text, duration, and resource usage.
  Filter by session status (ACTIVE/IDLE) or username.
parameters:
  - name: status_filter
    type: string
    description: Filter by session status — ACTIVE, IDLE, or empty string for all
  - name: user_filter
    type: string
    description: Filter by username (empty string for all users)
statement: |
  SELECT
    session_id,
    user_name,
    application_name,
    client_address,
    status,
    EXTRACT(EPOCH FROM (NOW() - session_start))::INT AS session_duration_sec,
    active_queries,
    last_active_query,
    node_id
  FROM [SHOW CLUSTER SESSIONS]
  WHERE ($1 = '' OR status = $1)
    AND ($2 = '' OR user_name = $2)
  ORDER BY session_duration_sec DESC
```

A few patterns worth noting:

**Optional filters with empty-string convention.** The pattern `($1 = '' OR column = $1)`
makes every filter optional. Pass an empty string to skip it, or a value to filter.
This avoids needing separate tools for "all sessions" vs "sessions for user X".

**`[SHOW CLUSTER SESSIONS]` syntax.** CockroachDB lets you use `SHOW` commands as
subqueries by wrapping them in brackets. This returns session data from all nodes,
not just the one you're connected to.

**Toolbox auto-appends `LIMIT 1000`.** Don't add your own `LIMIT` clause — Toolbox
handles it. If you add one, you'll get a SQL error from the double `LIMIT`.

## Step 4: Cross-Database Queries — Template Parameters

Here's where CockroachDB requires a Toolbox-specific pattern. The `information_schema`
is scoped per database — to query columns in database "movr", you need
`movr.information_schema.columns`, not just `information_schema.columns`.

Regular parameters (`$1`) use prepared statements and can't appear in identifiers
like database names. Toolbox solves this with **template parameters** — values injected
into the SQL text before execution:

```yaml
---
kind: tools
name: cr-columns
type: cockroachdb-sql
source: crdb
description: >
  List all columns for a table including data types, defaults, and nullability.
parameters:
  - name: schema
    type: string
    description: Schema name filter (empty string for all schemas)
  - name: table
    type: string
    description: Table name filter (empty string for all tables)
templateParameters:
  - name: database
    type: string
    description: Database name
statement: |
  SELECT
    table_schema AS schema_name,
    table_name,
    column_name,
    ordinal_position,
    data_type,
    is_nullable,
    column_default
  FROM {{.database}}.information_schema.columns
  WHERE table_schema NOT IN ('information_schema', 'pg_catalog', 'crdb_internal', 'pg_extension')
    AND ($1 = '' OR table_schema = $1)
    AND ($2 = '' OR table_name = $2)
  ORDER BY table_schema, table_name, ordinal_position
```

Notice the distinction:
- `{{.database}}` — template parameter, injected into SQL text (for the database identifier)
- `$1`, `$2` — prepared statement parameters (for filter values, safe from injection)

Use template parameters only for trusted identifiers like database or schema names.
User-supplied filter values should always go through prepared statement parameters.

## Step 5: System Tenant Queries — Node Health

Some queries only work from the system tenant. Node liveness data, virtual cluster
listings, and event logs live there. Reference the `crdb-system` source:

```yaml
---
kind: tools
name: cr-health-nodes
type: cockroachdb-sql
source: crdb-system
description: >
  Show node health status — membership, draining status, and seconds since
  last heartbeat. Requires system tenant connection.
parameters: []
statement: |
  SELECT
    node_id,
    epoch,
    membership,
    draining,
    decommissioning,
    EXTRACT(EPOCH FROM (NOW() - updated_at))::INT AS seconds_since_update
  FROM crdb_internal.gossip_liveness
  ORDER BY node_id
```

The only difference from a regular tool is `source: crdb-system`. Toolbox handles
the connection routing transparently.

## Step 6: Organize with Toolsets

With multiple tools, clients benefit from loading subsets. Toolsets group tools by
purpose:

```yaml
---
kind: toolsets
name: dba-schema
description: Schema exploration — tables, columns, constraints
tools:
  - cr-tables
  - cr-columns
  - cr-table-constraints
  - cr-find-object

---
kind: toolsets
name: dba-monitoring
description: Live activity — sessions, locks, query performance
tools:
  - cr-session
  - cr-show-locks
  - cr-query-stats

---
kind: toolsets
name: dba-health
description: Cluster health — version, nodes, ranges, storage
tools:
  - cr-health-version
  - cr-health-nodes
  - cr-health-ranges
  - cr-cluster-top-databases
  - cr-list-tenants
```

A Python client can load just what it needs:

```python
from toolbox_core import ToolboxClient

async with ToolboxClient("http://127.0.0.1:5000") as client:
    # Load only monitoring tools
    tools = await client.load_toolset("dba-monitoring")

    # Check active sessions
    result = await tools["cr-session"].invoke({
        "status_filter": "ACTIVE",
        "user_filter": ""
    })
```

## Step 7: Connect to Claude Desktop

Add Toolbox to your Claude Desktop configuration (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "cockroachdb": {
      "command": "toolbox",
      "args": ["--tools-file", "/path/to/tools.yaml"]
    }
  }
}
```

Restart Claude Desktop. You can now ask natural language questions and Claude will
invoke the appropriate tools:

- "Who's connected to the cluster right now?"
- "Are there any under-replicated ranges?"
- "Show me the schema for the orders table in the movr database"
- "What are the top databases by storage size?"
- "List all users with admin privileges"

Claude reads the tool descriptions to decide which tool to call, maps your question
to parameters, and formats the results.

## CockroachDB Patterns Reference

Here's a cheat sheet of patterns you'll encounter when building CockroachDB tools
for Toolbox:

| Pattern | When to Use | Example |
|---------|-------------|---------|
| `crdb_internal.*` tables | Cluster metadata not in information_schema | `crdb_internal.tables`, `crdb_internal.ranges` |
| `[SHOW ... ]` as subquery | Using SHOW commands in WHERE/SELECT | `FROM [SHOW CLUSTER SESSIONS]` |
| `{{.database}}.information_schema.*` | Cross-database schema queries | Template parameter for database name |
| `source: crdb-system` | Node, tenant, or event log queries | gossip_liveness, system.eventlog |
| `($1 = '' OR col = $1)` | Optional filters | Every filter parameter |
| `($1::INT \|\| ' hours')::INTERVAL` | Time-based lookback | `make_interval()` doesn't work through Toolbox |
| No `LIMIT` clause | Always | Toolbox auto-appends `LIMIT 1000` |
| `EXTRACT(EPOCH FROM ...)::INT` | Duration in seconds | Session duration, lock duration |

## What NOT to Expose

Some DBA operations don't fit the single-SQL-statement model. These require
multi-step orchestration, file I/O, or write access that shouldn't be exposed
through an LLM interface:

- **DDL generation** — Requires formatting, assembly, and multi-query logic
- **Backup / restore / migrate** — Multi-step operations with state management
- **Cluster creation / upgrade** — Container or infrastructure orchestration
- **Statistics generation** — Write operations (`CREATE STATISTICS`)
- **Data loading / unloading** — File I/O that Toolbox can't handle

Keep your Toolbox tools read-only and single-query. If you need multi-step
workflows, build them in your application layer and use Toolbox tools as
building blocks.

## Going Further

The example in this post covers 15 tools across 4 categories. A production setup
might include 30-40+ tools covering query history, workload analysis, data skew
detection, foreign key exploration, view inspection, and replication lag monitoring.

The full configuration with 39 tools is available at:
[github.com/amaddahian/roachie/tools/mcp](https://github.com/amaddahian/roachie)

Key resources:
- [GenAI Toolbox documentation](https://googleapis.github.io/genai-toolbox/)
- [CockroachDB crdb_internal reference](https://www.cockroachlabs.com/docs/stable/crdb-internal)
- [MCP protocol specification](https://modelcontextprotocol.io/)

---

*Ali Maddahian is the creator of [roachie](https://github.com/amaddahian/roachie),
an open-source CockroachDB DBA toolkit with 77 CLI tools, a natural language
interface, and MCP integration.*
