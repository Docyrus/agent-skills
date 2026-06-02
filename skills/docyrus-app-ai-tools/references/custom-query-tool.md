# `custom_query` tool

Runs a **read-only** SQL `SELECT` you hand-write, rendered from a Handlebars template with the LLM's arguments and tenant/user context. Use when a single data source query isn't enough — joins, aggregations, window functions, CTEs.

Runtime: the template is compiled once, rendered per call, then executed inside a **read-only** `DbUtils.transaction` (Postgres rejects any DML/DDL at the engine) with RLS enforced and a 15s timeout. Only `SELECT` / `WITH … SELECT` is allowed; the query is wrapped as `SELECT * FROM (<your sql>) _q LIMIT <limit> OFFSET <offset>`. Result returned to the LLM:

```json
{ "type": "custom_query_result", "data": [ /* rows */ ], "meta": { "count": <n> } }
```

## Config fields

| Flag | Key | Required | Meaning |
| --- | --- | --- | --- |
| `--customQuerySqlQuery` | `custom_query_sql_query` | **Yes** | Handlebars-templated SELECT. |
| `--inputJsonSchema` | `input_json_schema` | **Yes** | LLM argument schema. The model may also supply `limit`/`offset` (numbers) — include them in the schema if you want the model to page; otherwise they default to limit 1000 (max 50000), offset 0. |
| `--customQueryFilters` | `custom_query_filters` | No | Author "inline" filters exposed as `{{FILTERS.<field>}}`; the LLM cannot override a field an inline filter already claims. |

## Template variables

| Variable | Value |
| --- | --- |
| `INPUT` | The LLM's validated arguments object. `{{INPUT.customerId}}`. |
| `FILTERS` | Merged inline + LLM filters keyed by field → `{ operator, field, value }`. `{{FILTERS.status.value}}`. |
| `TENANT_ID`, `TENANT_SCHEMA` | Current tenant id / schema name. |
| `USER_ID`, `USER_EMAIL`, `USER_FIRSTNAME`, `USER_LASTNAME`, `USER_FULLNAME` | The acting user. |
| `{{q <value>}}` helper | Renders a **safe single-quoted SQL string literal**, or `NULL` for null/undefined. |

## SQL-injection contract (read carefully)

Inputs are sanitized by doubling single quotes only. That is **only** safe when every interpolated value is a quoted string literal. Two safe forms:

```sql
WHERE name = '{{INPUT.name}}'      -- explicit quotes
WHERE name = {{q INPUT.name}}      -- q helper quotes + handles NULL
```

**Never** bare-interpolate identifiers, numbers, or operators (`WHERE id = {{INPUT.id}}`, `SELECT {{INPUT.col}} …`, `ORDER BY {{INPUT.sort}}`). The read-only transaction limits blast radius (no writes/DDL) but does **not** stop data exfiltration via crafted input. For numeric inputs, validate the range/`type` in `input_json_schema` and still wrap or cast explicitly.

## Example: `revenue_by_month`

`revenue-by-month.json`:

```json
{
  "name": "Revenue by month",
  "key": "revenue_by_month",
  "type": "custom_query",
  "description": "Total paid invoice revenue grouped by month for a given year. Pass the 4-digit year.",
  "input_json_schema": {
    "type": "object",
    "properties": { "year": { "type": "integer", "minimum": 2000, "maximum": 2100 } },
    "required": ["year"]
  },
  "custom_query_sql_query": "SELECT date_trunc('month', paid_on) AS month, sum(amount) AS revenue FROM invoices WHERE status = 'paid' AND extract(year FROM paid_on) = {{q INPUT.year}}::int GROUP BY 1 ORDER BY 1"
}
```

```bash
docyrus apps ai-tools create --appSlug finance --from-file revenue-by-month.json
```

Note `{{q INPUT.year}}::int` — the value is rendered as a quoted literal then cast, never bare-interpolated. Tenant scoping is automatic via RLS; add `WHERE tenant_id = {{q TENANT_ID}}` only if the table isn't already RLS-scoped.
