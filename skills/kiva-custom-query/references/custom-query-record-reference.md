# Custom Query Record Reference

The physical shape of a `tenant_custom_query` record, the JSON structure of its
`fields`/`filters`, how runtime parameters bind via Handlebars, and the run
contract. Read this when writing a `create-custom-query` payload, designing
parameters, or interpreting a run response.

## Table of contents

- [Record columns](#record-columns)
- [`fields` — output columns](#fields--output-columns)
- [`filters` — declared runtime parameters](#filters--declared-runtime-parameters)
- [Parameter binding (Handlebars)](#parameter-binding-handlebars)
- [Run contract](#run-contract)
- [Ownership & visibility](#ownership--visibility)

## Record columns

A custom query record is one row in `public.tenant_custom_query`. The columns a
create/update payload can set:

| Column | Type | Meaning |
|---|---|---|
| `name` | text, **required** | Display name. |
| `description` | text | Free text. |
| `dsql_query` | text | **DSQL (logical SQL) body — the preferred format.** Runs through the DSQL runner (`appSlug.dataSourceSlug` tables). Takes precedence over `query` when set. |
| `query` | text | Legacy raw Postgres SQL body. Kept for old queries; stored as `""` for DSQL-only records. |
| `balance_query` | text | Optional legacy raw-SQL "balance" variant (no DSQL equivalent). |
| `fields` | jsonb array, **required** | Output column definitions — see below. Drives `output_json_schema`. |
| `filters` | jsonb array | Declared runtime parameters — see below. Drives `input_json_schema`. |
| `calculations` | jsonb | Aggregation definitions `{func, name, field, ...}`. |
| `default_rows` / `default_columns` | jsonb | Default pivot grouping. |
| `owner_product_id` | uuid | Owning product (defaulted server-side). |

Server-derived / read-only: `id`, `tenant_app_id` (from the URL), `tenant_id`,
`ownership`, `input_json_schema`, `output_json_schema`, `archived`, audit columns.
`input_json_schema` / `output_json_schema` are **recomputed by the API** from
`name` + `filters` + `fields` on every create/update — never send them.

The API requires `name`, a non-empty `fields` array, and **at least one of**
`dsql_query` / `query`. DSQL-first: always set `dsql_query`.

## `fields` — output columns

Array describing the columns the query returns. Each entry:

```jsonc
{
  "slug": "total_amount",   // required — must match a column the SQL SELECTs
  "name": "Total Amount",   // required — display label
  "type": "number",         // text | number | date | datetime | boolean | ...
  "id": "…",                // optional
  "icon": "…",              // optional
  "options": { }            // optional (e.g. enum options, format)
}
```

Keep `slug` values in lockstep with the `SELECT` list of `dsql_query`. A field
whose `slug` is not produced by the query simply renders empty.

## `filters` — declared runtime parameters

Array declaring the parameters callers may pass at run time. Each entry:

```jsonc
{ "slug": "status", "name": "Status", "type": "text" }
```

These are the *declarations* (used to build the input schema and any parameter
UI). The *values* are supplied per run in the run body's `filters` group (below).
A parameter is only meaningful if the query body references it via `{{filter}}`.

## Parameter binding (Handlebars)

Both `dsql_query` and `query` are compiled with **Handlebars** before execution.
Available template variables:

- Session context: `{{TENANT_ID}}`, `{{TENANT_SCHEMA}}`, `{{USER_ID}}`,
  `{{USER_FIRSTNAME}}`, `{{USER_LASTNAME}}`, `{{USER_FULLNAME}}`, `{{USER_EMAIL}}`.
- `FILTERS` — a map keyed by the run body's filter `field`, each value
  `{ operator, field, value }`.

The `{{filter}}` helper turns a run-time filter rule into a **safe SQL
predicate**, or `1=1` when the caller didn't pass that filter:

```sql
-- dsql_query
select
  t.id, t.subject, t.status
from base.task t
where {{filter FILTERS.status "t.status"}}
  and {{filter FILTERS.created "t.created_on"}}
order by t.created_on desc
```

- First arg: `FILTERS.<field>` — `<field>` matches the run body rule's `field`.
- Second arg: the SQL column expression the predicate targets (identifiers and
  JSONB `"data"->>'uuid'` expressions are validated; literals are escaped).
- The helper supports ~80 operators (comparison, `like`/`ilike`, `between`,
  `contains any/all`, relative-date operators like `last_7_days`/`this_month`,
  and user/team/role/hierarchy-unit membership operators). Membership/date
  operators that need identity read `USER_ID` / `TENANT_ID` from context.

Because absent filters compile to `1=1`, a single query body safely serves calls
with any subset of its declared parameters.

## Run contract

Run body (`RunCustomQueryDto`):

```jsonc
{
  "offset": 0,
  "filters": {                         // an IQueryFilterGroup
    "logic": "and",
    "rules": [
      { "field": "status", "operator": "eq", "value": "open" }
    ]
  },
  "debug": false,                      // true → return compiled SQL, do not execute
  "simulate": false                    // true → EXPLAIN ANALYZE (raw) / compile-only (DSQL)
}
```

Each rule's `field` is what `{{filter FILTERS.<field> ...}}` reads. Response:

```jsonc
{
  "data": [ /* result rows */ ],
  "meta": { "count": 123, "compiledQuery": "…" }   // compiledQuery present with debug/simulate
}
```

Rows are capped server-side (max 50000). `debug`/`simulate` return `data: []` and
put the compiled/explained SQL in `meta.compiledQuery` — use them to inspect what
the template produced without running the full query.

## Ownership & visibility

- Tenant-authored records have `ownership = "CUSTOM"` and are scoped to the
  tenant + app. RLS only lets a tenant insert/update/delete its own `CUSTOM`
  rows. `create/update/delete-custom-query` always operate on `CUSTOM` rows.
- `ownership = "SYSTEM"` rows with `tenant_id IS NULL` are product-owned and
  visible to every tenant (read-only from a tenant's perspective).
- `delete-custom-query` is a **soft delete** (`archived = true`); list/get never
  return archived rows.
