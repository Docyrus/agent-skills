---
name: docyrus-dsql-query-design
description: Write, discover, and run DSQL (Docyrus Structured Query Language) queries against logical Docyrus data sources. Use when the user wants to query, report on, or aggregate data from Docyrus — list records, count tasks by status, join contacts with users, build time-series breakdowns, answer "show me all projects with more than 5 tasks", or any read-only data question. Also covers using the AI-powered `docyrus dsql ask` command (natural language → DSQL → run → results). Triggers on "query data", "show me all X", "count by Y", "join Z with W", "DSQL query", "docyrus dsql", "run a report", "SQL over data sources", "list records from", "aggregate", "how many", "docyrus dsql ask", "docyrus dsql query", "docyrus dsql schema", or any data retrieval / reporting task against Docyrus logical data sources.
---

# Docyrus DSQL Query Design

DSQL (Docyrus Structured Query Language) is a read-only, PostgreSQL-compatible SQL dialect that queries **logical data sources** — not physical tables. Tables are named `appSlug.dataSourceSlug`. This skill covers the full workflow: schema discovery → query authoring → execution.

For the complete language reference (all supported functions, tenant pseudo-functions, row limits, rejection rules), see [references/dsql-language-reference.md](references/dsql-language-reference.md). For common query patterns and examples, see [references/query-patterns.md](references/query-patterns.md).

## Workflow

Follow in order.

1. **Confirm auth.**
   ```bash
   docyrus auth who --json   # confirms session + tenant name
   ```
   No session → `docyrus auth login` first.

2. **Identify the data sources.** If app/source slugs are already known, skip to step 3. Otherwise:
   ```bash
   docyrus apps list --json                          # list apps → grab appSlug
   docyrus dsql schema app <appSlug> --json          # all queryable sources in the app
   ```
   The schema response includes a compact `CREATE TABLE appSlug.dataSourceSlug (...)` DDL per source — this lists all queryable field slugs and `-- references` join hints.

3. **Fetch the schema for the exact sources you'll query.** Three options — pick the smallest:

   | Situation | Command |
   |---|---|
   | Know appSlug + dataSourceSlug | `docyrus dsql schema data-source <appSlug> <dsSlug>` |
   | Know one or more data source IDs | `docyrus dsql schema data-sources --ids <id1,id2>` |
   | Need all sources in an app | `docyrus dsql schema app <appSlug>` |

   Always read the schema before writing joins — field slugs and reference hints are only visible there.

4. **Write the DSQL query.** Follow the rules in [references/dsql-language-reference.md](references/dsql-language-reference.md). Key constraints:
   - Tables are `appSlug.dataSourceSlug` (e.g. `base.task`, `crm.contact`)
   - Always alias every table; qualify all columns in multi-table queries
   - Bare `*` only when exactly one source is in scope
   - Default limit 100 applies if omitted; max 1000

5. **Run it.**
   ```bash
   docyrus dsql query "select t.id, t.subject from base.task t limit 10"
   # or from a file
   docyrus dsql query --from-file ./my-query.sql
   ```

6. **Quick path — AI generate + run in one shot.** For natural-language questions where you don't need to handwrite SQL:
   ```bash
   docyrus dsql ask "how many open tasks per project this month?"
   ```
   `ask` calls the DSQL Generator agent (discover schema → write DSQL → execute → return results). Use `generate` if you only want the SQL without running it:
   ```bash
   docyrus dsql generate "list contacts created in the last 30 days with their owner names"
   ```

7. **Explain the result.** Summarize what the data shows; note any relevant limit truncation or empty-result reasons.

---

## Command cheat-sheet

### Schema discovery

```bash
# All sources in an app
docyrus dsql schema app crm --json

# Single source
docyrus dsql schema data-source crm contact --json

# Multiple sources by ID
docyrus dsql schema data-sources --ids "019c48d0-...,019c48e0-..." --json
```

### Execute a query

```bash
# Inline SQL
docyrus dsql query "select p.id, p.name, p.status from base.project p order by p.created_on desc limit 25"

# From file
docyrus dsql query --from-file ./report.sql

# JSON output
docyrus dsql query "select count(*) as n from base.task t" --format json
```

### AI-assisted generation

```bash
# Generate only (returns { prompt, query })
docyrus dsql generate "total time logged per project last week"

# Generate + run (returns { prompt, query, data, meta })
docyrus dsql ask "how many tasks are overdue by assignee?"

# Override the DSQL generator agent
docyrus dsql ask "..." --agentId <custom-agent-id>
```

---

## Critical rules

- **Table notation is `appSlug.dataSourceSlug`**, never a physical Postgres table. Bare table names (`task`, `contact`) are rejected unless they are CTE aliases.
- **Always alias every table** (`from base.task t`). Qualify every column in joins — DSQL rejects ambiguous unqualified column references.
- **Bare `*` is only valid when exactly one logical source is in scope.** `select * from base.task t join base.project p ...` is rejected — use `t.*` or list columns.
- **Read-only only.** `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `DROP`, and any DDL are rejected outright.
- **No physical tables, no schema-qualified functions.** `select * from public.tenant_user` or `select pg_catalog.now()` — both rejected. Use `tenant.user` for users and the approved function list.
- **Tenant pseudo-functions** (`tenant.current_user_id()`, `tenant.user_in_current_teams(userId)`, etc.) are the only way to filter by current session identity securely. Don't substitute hard-coded UUIDs for them.
- **`tenant.user` and `tenant.enum`** are system sources for resolving user names/emails and enum labels/colors. Join on `u.id` and `e.id` respectively.
- **Inherited base fields**: `base.task` and `base.event` expose `base.activity` fields (e.g. `record_owner`, `created_on`) — query them directly on the child source.
- **Row limit**: default 100 if no `LIMIT` clause; hard cap 1000. A user-written smaller limit is preserved. `LIMIT ALL` does not bypass the cap.
- **Schema first**: never guess field slugs — fetch the schema and read the DDL. Wrong field slugs return a DSQL error.
- **`docyrus dsql ask`** is the one-liner path: natural language → AI-generate → run → results. Use it when handwriting SQL is unnecessary.
- **CLI-only — do not use in frontend pages.** The underlying endpoint is not documented in Swagger and is not intended for use in implemented React/frontend application code. All DSQL queries must go through the `docyrus dsql` CLI commands.

---

## References

- **[references/dsql-language-reference.md](references/dsql-language-reference.md)** — Full DSQL language spec: supported clauses, functions, tenant pseudo-functions, table resolution rules, rejection list, row limits.
- **[references/query-patterns.md](references/query-patterns.md)** — Common query patterns: aggregates, joins, CTEs, tenant filters, enum lookups, date ranges, and error reference.
