# `data_source_query` tool

Reads from **one** data source. The tool author fixes every aspect of the query shape; the **LLM only supplies simple scalar filter parameters** declared in `input_json_schema`. Those values are bound into the author's filter template. The model cannot pass raw filters, columns, limits, offsets, formulas, or aggregations.

Runtime: `DataSource.readData` inside an RLS-enforced transaction. Result returned to the LLM:

```json
{ "type": "data_source_query_result", "data": [ /* rows */ ], "meta": { "count": <n> } }
```

## Contents
- [Config fields](#config-fields)
- [Parameter binding in filters (`{{param}}`)](#parameter-binding-in-filters)
- [Full-text keyword search (`filter_keyword`)](#full-text-keyword-search)
- [Filter rule shape & operators](#filter-rule-shape--operators)
- [Columns, limit, formulas, child queries](#columns-limit-formulas-child-queries)
- [Worked example: `get_customer_balance`](#worked-example-get_customer_balance)

## Config fields

| Flag | Key (`data_source_query_*`) | Required | Meaning |
| --- | --- | --- | --- |
| `--dataSourceQueryDataSourceId` | `..._data_source_id` | **Yes** | UUID of the `tenant_data_source` to read. Find it with `docyrus studio list-data-sources --appSlug <slug>` (returns `id` + `slug`). |
| `--inputJsonSchema` | `input_json_schema` | **Yes** | The scalar filter params the LLM supplies. Required even if empty (`{"type":"object","properties":{}}`). Keep params simple: `customerId` (uuid), `searchQuery` (text), `startYear` (number/date). |
| `--dataSourceQueryFilters` | `..._filters` | No | Author filter template (`IQueryFilterGroup`). Binds params via `{{param}}`; see below. |
| `--dataSourceQueryFilterKeyword` | `..._filter_keyword` | No | Full-text search keyword. A static string, or an LLM param bound via a whole `{{param}}` token; see below. |
| `--dataSourceQueryColumns` | `..._columns` | No | Fixed projection. String `"a,b,c"` or string array. Default `"*"` (all fields). |
| `--dataSourceQueryLimit` | `..._limit` | No | Fixed row cap. Default 1000, hard max 50000. Values ≤ 0 are ignored. |
| `--dataSourceQueryFormulas` | `..._formulas` | No | Fixed formulas object keyed by alias (`{ "<alias>": { ...formula } }`). |
| `--dataSourceQueryChildQueries` | `..._child_queries` | No | Fixed array of child (sub-record) queries; **each entry needs a non-empty `alias`**. |

Offset is always 0 and there is no LLM-driven pagination, ordering, calculations, or pivot — those are intentionally not exposed for this tool type. If you need them, use `custom_query`.

## Parameter binding in filters

A filter rule references an input parameter by setting its `value` to the **exact token** `{{paramName}}` (the whole value — not embedded like `%{{q}}%`). At call time:

- The token is replaced with the parameter's value, **keeping its native JSON type** (uuid string, number, boolean, array for `in`).
- If the parameter is **absent** — `null`, `undefined`, `""`, or `[]` — the rule is **dropped**, and any group left with no rules is pruned. So an omitted **optional** parameter simply means "don't filter on this".
- `0` and `false` are real values and are kept.
- Rules **without** a token are static and always apply — use them for tenant/owner/status scoping that the LLM must never be able to drop.

This is what makes optional filters work: declare params as optional in `input_json_schema`, add one bound rule per param, and the agent filters by whichever it provides.

## Full-text keyword search

`data_source_query_filter_keyword` runs a full-text search across the data source's text content (the same `filterKeyword` the REST query payload accepts), independent of the structured `filters` above. It uses the **same parameter binding** as filters, but it is a single string rather than a rule tree:

- Set it to the **exact token** `{{paramName}}` to bind the LLM's value. As with filters, partial interpolation (`%{{q}}%`) is **not** supported — the whole value must be the token.
- The bound value is coerced to a string (full-text search is text-only).
- If the parameter is **absent** — `null`, `undefined`, `""`, or `[]` — the keyword is **dropped** (no keyword filter applied). So an omitted optional param means "don't keyword-search".
- A plain string with no token is a **static** keyword that always applies.

Typical use: declare an optional `searchQuery` param and set `data_source_query_filter_keyword` to `"{{searchQuery}}"`. The agent gets a free-text search lever without being able to shape the query. Combine it with structured `filters` to mix scoped filtering and keyword search in one tool.

## Filter rule shape & operators

`data_source_query_filters` is an `IQueryFilterGroup`:

```json
{
  "combinator": "and",
  "rules": [
    { "field": "status", "operator": "=", "value": "active" },
    { "field": "customer_id", "operator": "=", "value": "{{customerId}}" },
    { "field": "name", "operator": "like", "value": "{{searchQuery}}" }
  ]
}
```

- `combinator`: `"and"` | `"or"`. Groups can nest (`rules` may contain other groups).
- `field`: a column/field slug on the data source.
- `operator`: comparison `= != <> > < >= <= between`; text `like "starts with" "ends with"`; collection `in "not in" "contains any" "contains all"`; null `"is null" "not null" empty "not empty"`; plus date operators (`today`, `last_7_days`, `this_month`, `before_today`, …) and user/role/team operators. `between` and `in` take array `value`s.
- For column-to-column comparison use `valueField` instead of `value` (not parameterizable).

## Columns, limit, formulas, child queries

These are author-fixed. When you set explicit `columns`, the aliases of any `formulas` / `child_queries` are appended automatically so their computed values are still projected. Leave `columns` unset for `"*"`.

The detailed shapes for `filters`, `formulas`, `childQueries`, and `pivot` match the REST data-source query payload. If the `docyrus-api-dev` skill is available, its `references/data-source-query-guide.md` is the full reference for those shapes.

## Worked example: `get_customer_balance`

Two optional params; either filters or both.

`get-customer-balance.json`:

```json
{
  "name": "Get Customer Balance",
  "key": "get_customer_balance",
  "type": "data_source_query",
  "description": "Returns customer balance rows. Filter by customerId for one customer, and/or by searchQuery to match the customer name. With no arguments, returns active customers' balances.",
  "data_source_query_data_source_id": "8f1c…-uuid-of-customer_balances-ds",
  "data_source_query_columns": "customer_id,customer_name,balance,overdue_amount",
  "data_source_query_limit": 200,
  "input_json_schema": {
    "type": "object",
    "properties": {
      "customerId":  { "type": "string", "format": "uuid", "description": "Restrict to one customer" },
      "searchQuery": { "type": "string", "description": "Case-insensitive match on customer name" }
    }
  },
  "data_source_query_filters": {
    "combinator": "and",
    "rules": [
      { "field": "status",      "operator": "=",    "value": "active" },
      { "field": "customer_id", "operator": "=",    "value": "{{customerId}}" },
      { "field": "customer_name","operator": "like", "value": "{{searchQuery}}" }
    ]
  }
}
```

```bash
docyrus apps ai-tools create --appSlug crm --from-file get-customer-balance.json
```

Behavior: `customerId` only → `status=active AND customer_id=<id>`. `searchQuery` only → `status=active AND customer_name like <q>`. Neither → `status=active`. The `status=active` rule always applies.
