---
name: docyrus-custom-query
description: Create, read, update, delete, run, and consume saved Docyrus custom query records (tenant_custom_query) — reusable, parameterized SQL query templates whose body is DSQL — using the `docyrus dsql *-custom-query` CLI commands, and call them from a frontend via `@docyrus/api-client`'s `RestApiClient.runCustomQuery`. Use when the user wants to save a DSQL query as a reusable named report/template, parameterize it with runtime filters (Handlebars `{{filter}}` binding), test/run a saved query (`--debug` compiled SQL, `--simulate` EXPLAIN ANALYZE), manage the lifecycle of custom query records, or run a saved custom query from a React/frontend app. Triggers on "custom query", "saved query", "query template", "reusable report", "parameterized query", "run a saved query", "runCustomQuery", `docyrus dsql create-custom-query`, `docyrus dsql run-custom-query`, `list/get/update/delete-custom-query`, or wiring a saved query into an app. For authoring the DSQL body itself and ad-hoc one-off queries, see docyrus-dsql-query-design.
---

# Docyrus Custom Query

A **custom query record** (`tenant_custom_query`) is a saved, named, parameterized
query template. Its body is **DSQL** (logical SQL over `appSlug.dataSourceSlug`
tables — the `dsql_query` column, successor to the legacy raw-SQL `query`), plus
declared output `fields`, declared runtime `filters` (parameters), and optional
pivot/calc config. Unlike an ad-hoc `dsql query`, a saved record is reusable,
parameterized, and callable from app code.

Two API surfaces back these records, both reached through `docyrus dsql`
subcommands:
- **CRUD** of the record → dev/architect API, scoped to an app.
- **Run** (execute) → reports API, by query id, with runtime filter values.

For the record/column shape, `fields`/`filters` JSON structure, `{{filter}}`
operators, and the run contract, see
[references/custom-query-record-reference.md](references/custom-query-record-reference.md).

## Workflow

Follow in order.

1. **Confirm auth.**
   ```bash
   docyrus auth who --json
   ```
   No session → `docyrus auth login`.

2. **Author the DSQL body first.** Use **docyrus-dsql-query-design** to discover
   schema and write/validate the `SELECT`. Run it ad-hoc until correct:
   ```bash
   docyrus dsql query "select t.id, t.subject, t.status from base.task t limit 5"
   ```
   Only save a query once the raw DSQL returns what you expect.

3. **Parameterize it.** Replace hard-coded predicates with `{{filter}}` bindings
   and decide the output `fields`. Absent filters compile to `1=1`, so one body
   serves any subset of parameters:
   ```sql
   select t.id, t.subject, t.status
   from base.task t
   where {{filter FILTERS.status "t.status"}}
   order by t.created_on desc
   ```

4. **Create the record.** `name`, `fields`, and one of `--dsqlQuery`/`--query`
   are required by the API; DSQL-first means `--dsqlQuery`:
   ```bash
   docyrus dsql create-custom-query --appSlug base \
     --name "Tasks by status" \
     --dsqlQuery 'select t.id, t.subject, t.status from base.task t where {{filter FILTERS.status "t.status"}} order by t.created_on desc' \
     --fields '[{"slug":"id","name":"ID","type":"text"},{"slug":"subject","name":"Subject","type":"text"},{"slug":"status","name":"Status","type":"text"}]' \
     --filters '[{"slug":"status","name":"Status","type":"text"}]'
   ```
   Grab the returned `id` — every other command needs it.

5. **Test / run it.** Verify the template compiles and returns rows. Inspect the
   compiled SQL first, then run for real:
   ```bash
   # See the SQL the template produced (no execution)
   docyrus dsql run-custom-query --queryId <id> \
     --filters '{"logic":"and","rules":[{"field":"status","operator":"eq","value":"open"}]}' \
     --debug true

   # Actually run it
   docyrus dsql run-custom-query --queryId <id> \
     --filters '{"logic":"and","rules":[{"field":"status","operator":"eq","value":"open"}]}'
   ```
   Each run rule's `field` is what `{{filter FILTERS.<field> ...}}` reads. Result
   is `{ data: rows, meta: { count, compiledQuery } }`.

6. **Iterate & manage.** `get-custom-query` to inspect, `update-custom-query` to
   change the body/fields/filters, `list-custom-query` to find ids,
   `delete-custom-query` to soft-archive.

7. **Wire it into a frontend** (optional) — see [Consuming from a frontend](#consuming-from-a-frontend).

## Command cheat-sheet

All under `docyrus dsql`. CRUD needs an app selector (`--appId` **or**
`--appSlug`); run needs only `--queryId`.

```bash
# List saved records for an app
docyrus dsql list-custom-query --appSlug base

# Get one record (full body, fields, filters)
docyrus dsql get-custom-query --appSlug base --queryId <id>

# Create (name + fields + dsqlQuery required; JSON flags take JSON strings)
docyrus dsql create-custom-query --appSlug base --name "…" \
  --dsqlQuery '…' --fields '[…]' --filters '[…]'

# Update (partial — only the flags you pass change)
docyrus dsql update-custom-query --appSlug base --queryId <id> --name "…"

# Delete (soft-archive)
docyrus dsql delete-custom-query --appSlug base --queryId <id>

# Run (execute). --debug = compiled SQL only; --simulate = EXPLAIN ANALYZE
docyrus dsql run-custom-query --queryId <id> --filters '{…}' [--offset N] [--debug true] [--simulate true]
```

Write-command flags: `--name`, `--description`, `--dsqlQuery` (→ `dsql_query`,
preferred), `--query` (legacy raw SQL), `--fields` (JSON), `--filters` (JSON),
`--calculations` (JSON), `--defaultColumns` (JSON), `--defaultRows` (JSON),
`--balanceQuery`, `--ownerProductId`. Anything not covered by a flag — or a whole
payload — can go through `--data '<json>'` / `--from-file <path.json>`; flags are
merged over that base.

## Consuming from a frontend

Saved custom queries have **no generated collection helper**. Run one directly
with `RestApiClient.runCustomQuery(id, options)` from `@docyrus/api-client`:

```ts
import type { RestApiClient } from "@docyrus/api-client";

// options is the run body: { offset?, filters?, debug?, simulate? }
const result = await client.runCustomQuery<{
  data: Array<Record<string, unknown>>;
  meta: { count: number };
}>(customQueryId, {
  filters: {
    logic: "and",
    rules: [{ field: "status", operator: "eq", value: "open" }],
  },
});

const rows = result.data;          // the result rows
const total = result.meta.count;   // total row count
```

- The runtime filter `rules[].field` must match a `{{filter FILTERS.<field> …}}`
  binding in the saved query — that's how a UI parameter reaches the SQL.
- The call returns the `{ data, meta }` envelope; read `.data` for rows.
- Build filter groups with the exported `prepareFilterQueryForApi` helper when you
  need the query-string form for other endpoints.

## Critical rules

- **DSQL-first.** Put the query in `dsql_query` (`--dsqlQuery`). It runs through
  the DSQL runner over `appSlug.dataSourceSlug` tables and follows every
  docyrus-dsql-query-design rule (read-only, alias tables, qualify columns). The
  legacy raw-SQL `query` is only for pre-existing records.
- **Author the DSQL before saving.** Validate with `dsql query` first; a saved
  record with a broken body just fails at run time.
- **`name` + `fields` + a query body are required on create.** Keep `fields[].slug`
  in lockstep with the `SELECT` list.
- **Parameters bind through `{{filter}}`.** Declared `filters` do nothing unless
  the body references them; run-time values arrive in the run body's `filters`
  group keyed by `field`. Never string-concatenate user input into the SQL — use
  `{{filter FILTERS.x "col"}}`, which escapes and validates.
- **`--debug` / `--simulate` before trusting output.** `--debug true` returns the
  compiled SQL without executing; `--simulate true` returns EXPLAIN ANALYZE. Both
  return `data: []` with the SQL in `meta.compiledQuery`.
- **CRUD is app-scoped; run is not.** `create/get/update/delete/list-custom-query`
  need `--appId`/`--appSlug`; `run-custom-query` needs only `--queryId`.
- **Delete is soft.** `delete-custom-query` archives the record; it stops
  appearing in list/get but is not physically removed.
- **Frontend uses `runCustomQuery`.** There is no collection hook for saved
  queries — call `RestApiClient.runCustomQuery(id, options)` and wire it into your
  own data-fetching layer.

## References

- **[references/custom-query-record-reference.md](references/custom-query-record-reference.md)** —
  record columns, `fields`/`filters` JSON structure, Handlebars `{{filter}}`
  binding and operators, the run request/response contract, and ownership/RLS.
