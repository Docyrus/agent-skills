# Custom Query MCP Tools Guide

Reference for managing and executing custom queries through Docyrus Architect MCP tools.

Custom queries are saved SQL templates (stored per tenant) that support Handlebars templating and runtime filters.

---

## Table of Contents

1. [Overview](#overview)
2. [Tools Summary](#tools-summary)
3. [Custom Query Structure](#custom-query-structure)
4. [Handlebars Templating](#handlebars-templating)
5. [Tool Reference](#tool-reference)
6. [Filter Definitions](#filter-definitions)
7. [Runtime Filters](#runtime-filters)
8. [Simulation Mode](#simulation-mode)
9. [Complete Examples](#complete-examples)

---

## Overview

The custom query toolset supports full lifecycle operations and execution:

- CRUD operations: list, get, create, update, delete
- Execution: run with runtime filters, pagination offset, and simulate mode

Use these tools when a user wants reusable SQL reports, dashboards, or advanced analysis that is difficult to express with `query_data_source` alone.

---

## Tools Summary

| Tool | Description | Read-Only | Destructive |
|---|---|---|---|
| `get_custom_queries` | List all non-archived custom queries | Yes | No |
| `get_custom_query_by_id` | Get full details of a custom query | Yes | No |
| `create_custom_query` | Create a new custom query | No | No |
| `update_custom_query` | Update an existing custom query | No | No |
| `delete_custom_query` | Soft-delete (archive) a custom query | No | Yes |
| `run_custom_query` | Execute a custom query | Yes | No |

---

## Custom Query Structure

| Property | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | Yes | Display name |
| `description` | `string \| null` | No | Human-readable description |
| `query` | `string` | Yes | SQL template with Handlebars syntax |
| `filters` | `array \| null` | No | Filter field definitions for runtime filtering |

---

## Handlebars Templating

Custom query SQL templates can use runtime variables and helper functions.

### Available Template Variables

| Variable | Description |
|---|---|
| `{{TENANT_ID}}` | Current tenant UUID |
| `{{TENANT_SCHEMA}}` | Tenant schema name (for table references) |
| `{{USER_ID}}` | Current user UUID |
| `{{USER_EMAIL}}` | Current user email |
| `{{USER_FIRSTNAME}}` | Current user first name |
| `{{USER_LASTNAME}}` | Current user last name |
| `{{USER_FULLNAME}}` | Current user full name |

### `filter` Helper

Use the helper to convert runtime filter rules into SQL predicates.

Syntax:

```handlebars
{{filter FILTERS.field_slug column_expression}}
```

- `FILTERS.field_slug`: filter slug declared in custom query `filters`
- `column_expression`: SQL expression to filter

Example:

```sql
SELECT id, name, amount, created_on
FROM {{TENANT_SCHEMA}}.crm_account
WHERE tenant_id = '{{TENANT_ID}}'
  AND {{filter FILTERS.status status}}
  AND {{filter FILTERS.created_on created_on}}
  AND {{filter FILTERS.amount (crm_account."data"->>'0192919d-c16d-7c8b-848e-6069d3205395')::numeric}}
```

If no runtime value is provided for a rule, the helper resolves to `1=1` so the filter stays optional.

### JSONB Data Field Expressions

For simple data sources using JSONB custom fields:

```sql
(table_alias."data"->>'field-uuid')::type
```

---

## Tool Reference

### `get_custom_queries`

List all non-archived custom queries for the current tenant.

Input: none

Output shape:

```json
{
  "queries": [
    {
      "id": "uuid-string",
      "name": "Monthly Sales Report",
      "description": "Aggregated sales by month and category"
    }
  ]
}
```

### `get_custom_query_by_id`

Get full definition of one query.

Input:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `queryId` | `string` | Yes | Query UUID |

Output shape:

```json
{
  "id": "uuid-string",
  "name": "Monthly Sales Report",
  "description": "Aggregated sales by month and category",
  "query": "SELECT ... FROM {{TENANT_SCHEMA}}.shop_order WHERE ...",
  "filters": []
}
```

### `create_custom_query`

Create a new custom query template.

Input:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | `string` | Yes | Display name |
| `description` | `string \| null` | No | Description |
| `query` | `string` | Yes | SQL template with Handlebars syntax |
| `filters` | `array \| null` | No | Filter field definitions |

Output shape:

```json
{
  "id": "newly-created-uuid",
  "name": "Monthly Sales Report"
}
```

### `update_custom_query`

Update an existing custom query. Omitted fields are unchanged.

Input:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `queryId` | `string` | Yes | Query UUID |
| `name` | `string \| null` | No | New name |
| `description` | `string \| null` | No | New description |
| `query` | `string \| null` | No | New SQL template |
| `filters` | `array \| null` | No | New filter definitions |

Output shape:

```json
{
  "success": true
}
```

### `delete_custom_query`

Soft-delete (archive) a custom query.

Input:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `queryId` | `string` | Yes | Query UUID |

Output shape:

```json
{
  "success": true
}
```

### `run_custom_query`

Execute a saved query with runtime options.

Input:

| Parameter | Type | Required | Description |
|---|---|---|---|
| `queryId` | `string` | Yes | Query UUID |
| `offset` | `number \| null` | No | Rows to skip (default 0) |
| `filters` | `object \| null` | No | Runtime filters |
| `simulate` | `boolean` | No | If true, return explain plan instead of records |

Output shape:

```json
{
  "data": [],
  "meta": {
    "count": 0,
    "compiledQuery": "SELECT ... LIMIT 50000 OFFSET 0"
  }
}
```

Execution constraints:

- Max rows: 50,000
- Timeout: 15 seconds for normal run
- Timeout: 30 seconds for `simulate: true`

---

## Filter Definitions

Filter definitions are saved with the custom query and exposed to runtime filtering/UI:

```json
{
  "slug": "created_on",
  "name": "Created Date",
  "type": "field-dateTime"
}
```

| Property | Type | Required | Description |
|---|---|---|---|
| `slug` | `string` | Yes | Identifier used by `FILTERS.slug` |
| `name` | `string` | Yes | Display label |
| `type` | `string` | Yes | Field type controlling operator options |

---

## Runtime Filters

`run_custom_query` accepts a filter group payload:

```json
{
  "filters": {
    "rules": [
      { "field": "created_on", "operator": "last_30_days", "value": null },
      { "field": "status", "operator": "=", "value": "active" }
    ],
    "combinator": "and",
    "not": false
  }
}
```

Rule properties:

| Property | Type | Required | Description |
|---|---|---|---|
| `field` | `string` | Yes | Filter slug, matching custom query definition |
| `operator` | `string` | Yes | Filter operator |
| `value` | `any \| null` | No | Value (optional for shortcut operators) |
| `filterType` | `string` | No | Type hint for value casting |

Supported operator categories:

- Comparison: `=`, `!=`, `<>`, `>`, `<`, `>=`, `<=`, `between`
- Text: `like`, `not like`, `starts with`, `ends with`, `contains`
- Collection: `in`, `not in`, `contains any`, `contains all`
- Null/empty: `empty`, `not empty`, `null`, `not null`
- Boolean: `true`, `false`
- Date shortcuts: `today`, `yesterday`, `last_7_days`, `last_30_days`, `this_week`, `this_month`, `this_year`, `last_month`, `next_30_days`
- Dynamic date: `x_days_ago`, `x_days_later`, `in_last_x_days`, `in_next_x_days`
- User/team: `active_user`, `not_active_user`, `in_active_user_scope`, `in_role`, `in_team`, `in_active_user_team`

For the full operator list and behavior details, read [data-source-query-guide.md](./data-source-query-guide.md#filter-operators-reference).

---

## Simulation Mode

Set `simulate: true` in `run_custom_query` to execute:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
```

Use this mode to:

- debug performance
- detect sequential scans and costly joins
- verify query behavior before fetching large data

Example:

```json
{
  "queryId": "query-uuid",
  "simulate": true
}
```

Typical response metadata includes `meta.explained` with plan details.

---

## Complete Examples

### Example 1: Create and Run Monthly Sales

Create:

```json
{
  "name": "Sales by Month",
  "description": "Monthly sales aggregation with optional filters",
  "query": "SELECT to_char(o.created_on, 'YYYY-MM') as month, COUNT(*) as order_count, SUM(o.amount) as total_amount FROM {{TENANT_SCHEMA}}.shop_order o WHERE o.tenant_data_source_id = 'ds-uuid' AND {{filter FILTERS.created_on o.created_on}} AND {{filter FILTERS.status o.status}} GROUP BY to_char(o.created_on, 'YYYY-MM') ORDER BY month DESC",
  "filters": [
    { "slug": "created_on", "name": "Order Date", "type": "field-dateTime" },
    { "slug": "status", "name": "Order Status", "type": "field-select" }
  ]
}
```

Run:

```json
{
  "queryId": "created-query-uuid",
  "filters": {
    "rules": [
      { "field": "created_on", "operator": "this_year", "value": null },
      { "field": "status", "operator": "!=", "value": "cancelled" }
    ],
    "combinator": "and",
    "not": false
  }
}
```

### Example 2: Simulate Before Running

```json
{
  "queryId": "complex-query-uuid",
  "simulate": true,
  "filters": {
    "rules": [
      { "field": "created_on", "operator": "this_year", "value": null }
    ],
    "combinator": "and",
    "not": false
  }
}
```

Then fetch records:

```json
{
  "queryId": "complex-query-uuid",
  "simulate": false,
  "offset": 0
}
```

### Example 3: JSONB Custom Field Filtering

```json
{
  "name": "CRM Pipeline Report",
  "description": "Deal pipeline with JSONB custom field access",
  "query": "SELECT r.id, r.name, (r.\"data\"->>'0192919d-c16d-7c8b-848e-6069d3205395')::numeric as deal_amount, r.\"data\"->>'019291a2-b8e4-7f3a-9c2d-4a1b3c5d7e8f' as stage, r.created_on FROM {{TENANT_SCHEMA}}.tenant_record r WHERE r.tenant_data_source_id = 'deal-ds-uuid' AND {{filter FILTERS.deal_amount (r.\"data\"->>'0192919d-c16d-7c8b-848e-6069d3205395')::numeric}} AND {{filter FILTERS.stage r.\"data\"->>'019291a2-b8e4-7f3a-9c2d-4a1b3c5d7e8f'}} ORDER BY deal_amount DESC",
  "filters": [
    { "slug": "deal_amount", "name": "Deal Amount", "type": "field-number" },
    { "slug": "stage", "name": "Stage", "type": "field-select" }
  ]
}
```

