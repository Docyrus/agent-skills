# Docyrus Data Source Query Guide

## Table of Contents

1. [Query Payload Overview](#query-payload-overview)
2. [Columns](#columns)
3. [Filters](#filters)
4. [Filter Operators](#filter-operators)
5. [Order By](#order-by)
6. [Pagination](#pagination)
7. [Calculations (Aggregations)](#calculations)
8. [Formulas](#formulas)
9. [Block Formulas](#block-formulas)
10. [Subquery Formulas](#subquery-formulas)
11. [Child Queries](#child-queries)
12. [Pivot](#pivot)
13. [Expand](#expand)
14. [Complete Examples](#complete-examples)

---

## Query Payload Overview

All data source reads use `ICollectionListParams` (client-side) which maps to `ZodSelectQueryPayload` (server-side).

```typescript
collection.list({
  columns: ['name', 'email', 'status'],
  filters: { rules: [...], combinator: 'and' },
  filterKeyword: 'search term',
  orderBy: 'created_on desc',
  limit: 50,
  offset: 0,
  fullCount: true,
  calculations: [...],
  formulas: { ... },
  childQueries: { ... },
  pivot: { ... },
  expand: ['relation_field'],
})
```

**Critical rule**: Always send `columns` parameter. If omitted, only `id` is returned.

---

## Columns

**Type**: `Array<string>` (client) or comma-separated `string` (server)

### Basic
```typescript
columns: ['task_name', 'created_on', 'record_owner']
```

### Aliasing with `:`
```typescript
columns: ['ra:related_account']
// Result: { ra: { id: "uuid", name: "account name" } }
```

### Relation Expansion with `()`
```typescript
columns: ['task_name', 'related_account(name:account_name, phone:account_phone)']
// Result: { task_name: "...", related_account: { name: "...", phone: "..." } }
```

### Spread with `...`
Flattens relation fields to root:
```typescript
columns: ['task_name', '...related_account(account_name, phone:account_phone)']
// Result: { task_name: "...", account_name: "...", phone: "..." }
```

### Functions with `@`
```typescript
columns: ['...related_account(an:account_name@upper, ap:account_phone)']
// Result: { an: "ACCOUNT NAME", ap: "05556668899" }
```

### Date Grouping Formulas with `@`
For aggregation queries — groups records by time intervals:

| Formula | Description |
|---------|-------------|
| `hours_of_today@field` | Groups by hour for today |
| `days_of_week@field` | Groups by day for current week |
| `days_of_month@field` | Groups by day for current month |
| `weeks_of_month@field` | Groups by week for current month |
| `months_of_year@field` | Groups by month (YYYY-MM) |
| `quarters_of_year@field` | Groups by quarter (YYYY-Q) |

### to_char Formatting
```typescript
columns: ['day:to_char[DD/MM/YYYY]@created_on']
```

---

## Filters

Recursive group structure with AND/OR combinators.

```typescript
interface ICollectionFilterGroup {
  rules: Array<ICollectionFilterRule | ICollectionFilterGroup>
  combinator?: 'and' | 'or'  // default: 'and'
  not?: boolean               // negate entire group
}

interface ICollectionFilterRule {
  field?: string
  operator: string
  value?: unknown
  filterType?: string | null  // NUMERIC, ALPHA, BOOL, DATE, DATETIME, etc.
}
```

### Basic AND
```typescript
filters: {
  combinator: 'and',
  rules: [
    { field: 'task_status', operator: '=', value: 1 },
    { field: 'priority', operator: '>=', value: 3 },
  ],
}
```

### Nested AND + OR
```typescript
filters: {
  combinator: 'and',
  rules: [
    { field: 'created_on', operator: 'between', value: ['2025-10-01', '2025-11-01'] },
    {
      combinator: 'or',
      rules: [
        { field: 'email', operator: 'empty' },
        { field: 'phone', operator: 'not empty' },
      ],
    },
  ],
}
```

### Filtering by Related Record's Field
Use `rel_{{relation_field}}/{{field}}` syntax:
```typescript
{ field: 'rel_client/account_status', operator: '=', value: 2 }
```

### Negated Group
```typescript
{ combinator: 'and', not: true, rules: [{ field: 'status', operator: '=', value: 'archived' }] }
```

---

## Filter Operators

### Comparison
`=`, `!=`, `<>`, `>`, `<`, `>=`, `<=`, `between`

### Text
`like`, `not like`, `starts with`, `ends with`

### Collection
`in`, `not in`, `exists`, `contains any`, `contains all`, `not contains`

### Null/Empty
`is`, `is not`, `empty`, `not empty`, `null`, `not null`

### Boolean
`true`, `false`

### User-Related
`active_user`, `not_active_user`, `in_active_user_scope`, `in_role`, `not_in_role`, `in_team`, `in_active_user_team`, `in_unit`

### Date Shortcuts (no value needed)
`today`, `tomorrow`, `yesterday`, `last_7_days`, `last_15_days`, `last_30_days`, `last_60_days`, `last_90_days`, `last_120_days`, `next_7_days`, `next_15_days`, `next_30_days`, `next_60_days`, `next_90_days`, `next_120_days`, `last_week`, `this_week`, `next_week`, `last_month`, `this_month`, `next_month`, `before_today`, `after_today`, `last_year`, `this_year`, `next_year`, `first_quarter`, `second_quarter`, `third_quarter`, `fourth_quarter`, `last_3_months`, `last_6_months`

### Dynamic Date (value = number)
`x_days_ago`, `x_days_later`, `before_last_x_days`, `in_last_x_days`, `after_last_x_days`, `in_next_x_days`

---

## Order By

```typescript
// String
orderBy: 'created_on DESC'

// Object
orderBy: { field: 'created_on', direction: 'desc' }

// Multi-column
orderBy: [
  { field: 'status', direction: 'asc' },
  { field: 'created_on', direction: 'desc' },
]

// By related field
orderBy: 'relation_field_slug(field_name DESC), id ASC'
```

---

## Pagination

```typescript
{
  limit: 25,    // default: 100
  offset: 50,   // default: 0
  fullCount: true, // returns total matching count
}
```

---

## Calculations

Aggregate functions with grouping. Selected `columns` become GROUP BY fields.

```typescript
interface ICollectionCalculation {
  func: 'count' | 'sum' | 'avg' | 'min' | 'max' | 'jsonb_agg' | 'json_agg' | 'array_agg'
  field: string       // use 'id' for counting records
  name?: string       // alias for result column
  isDistinct?: boolean
  minValue?: number   // aggregate values greater than this
  maxValue?: number   // aggregate values less than this
  numberType?: 'bigint' | 'int' | 'decimal'
}
```

### Count per Group
```typescript
{
  columns: ['record_owner(name)'],
  calculations: [{ field: 'id', func: 'count', name: 'task_count' }],
  filters: { rules: [{ field: 'task_status', operator: '=', value: 1 }] },
}
```

### Multiple Aggregations
```typescript
{
  columns: ['category'],
  calculations: [
    { field: 'id', func: 'count', name: 'total' },
    { field: 'amount', func: 'sum', name: 'totalAmount' },
    { field: 'amount', func: 'avg', name: 'avgAmount' },
  ],
}
```

---

## Formulas

Virtual computed columns. Key = column alias, must appear in `columns`.

### Simple Formula
```typescript
formulas: {
  formatted_date: { func: 'to_char', args: ['{created_on}', 'DD/MM/YYYY'] },
  full_name: { func: 'concat', args: ['{first_name}', ' ', '{last_name}'] },
  upper_name: { func: 'upper', args: ['{name}'] },
}
```
Column refs use `{column_name}` syntax.

---

## Block Formulas

AST-based formulas for complex expressions.

### Structure
```typescript
{ inputs: [rootBlock] }  // exactly 1 root block
```

### Block Kinds

**literal** — Static value:
```json
{ "kind": "literal", "literal": 42 }
```

**column** — Column reference:
```json
{ "kind": "column", "name": "balance" }
```

**builtin** — SQL constant (`current_date`, `current_time`, `current_timestamp`, `now`):
```json
{ "kind": "builtin", "name": "current_date" }
```

**function** — Whitelisted SQL function:
```json
{ "kind": "function", "name": "round", "inputs": [{ "kind": "column", "name": "price" }, { "kind": "literal", "literal": 2 }] }
```

**math** — Arithmetic (`+`, `-`, `*`, `/`, `%`, min 2 operands):
```json
{ "kind": "math", "op": "*", "inputs": [{ "kind": "column", "name": "qty" }, { "kind": "column", "name": "price" }] }
```

**compare** — Comparison (`=`, `!=`, `>`, `<`, `>=`, `<=`, `like`, `ilike`, `in`, `not in`):
```json
{ "kind": "compare", "op": ">", "left": { "kind": "column", "name": "price" }, "right": { "kind": "literal", "literal": 100 } }
```

**boolean** — Logical (`and`, `or`, `not`):
```json
{ "kind": "boolean", "op": "and", "inputs": [compareBlock1, compareBlock2] }
```

**case** — CASE WHEN:
```json
{ "kind": "case", "cases": [{ "when": compareBlock, "then": literalBlock }], "else": literalBlock }
```

**aggregate** — `count`, `sum`, `avg`, `min`, `max`, etc.:
```json
{ "kind": "aggregate", "name": "count", "inputs": [] }
```

**extract** — Date part (`year`, `month`, `day`, `hour`, `minute`, `second`):
```json
{ "kind": "extract", "part": "month", "inputs": [columnBlock] }
```

### Type Casting
Any block can include `"cast": "decimal"` → SQL: `(expr)::decimal`

### Timezone
Any block can include `"tz": "UTC"` → SQL: `expr at time zone 'UTC'`

### Example: CASE with Multiple Conditions
```typescript
formulas: {
  deal_tier: {
    inputs: [{
      kind: 'case',
      cases: [
        { when: { kind: 'compare', op: '>=', left: { kind: 'column', name: 'amount' }, right: { kind: 'literal', literal: 100000 } }, then: { kind: 'literal', literal: 'Enterprise' } },
        { when: { kind: 'compare', op: '>=', left: { kind: 'column', name: 'amount' }, right: { kind: 'literal', literal: 25000 } }, then: { kind: 'literal', literal: 'Mid-Market' } },
      ],
      else: { kind: 'literal', literal: 'SMB' },
    }],
  },
}
```

---

## Subquery Formulas

Correlated subquery against a child data source.

```typescript
formulas: {
  active_deals: {
    from: 'crm_deal',         // child table slug: appSlug_tableSlug
    with: 'account',           // child field referencing parent id
    filters: { rules: [{ field: 'stage', operator: '!=', value: 'lost' }] },
    inputs: [{ kind: 'aggregate', name: 'count', inputs: [] }],
  },
}
```

Multi-field join:
```typescript
with: { child_field1: 'parent_field1', child_field2: 'parent_field2' }
```

---

## Child Queries

Fetch related records as nested JSON arrays per parent row.

```typescript
{
  columns: ['id', 'name', 'recent_orders'],
  childQueries: {
    recent_orders: {
      from: 'shop_order_item',    // appSlug_slug format
      using: 'product',            // child field referencing parent id
      columns: ['order_date', 'quantity', 'total_price'],
      orderBy: 'order_date DESC',
      limit: 5,
      filters: { rules: [{ field: 'order_date', operator: 'last_30_days' }] },
    },
  },
}
```

**Rules:**
- Child query key must also appear in parent's `columns`.
- `from` uses `appSlug_slug` format.
- `using` is the field in **child** DS referencing parent's `id`.

---

## Pivot

Cross-tab grouping with matrix CTEs.

```typescript
{
  columns: ['...order_status(status_name:name)'],
  pivot: {
    matrix: [
      {
        using: 'created_on',
        columns: 'day:to_char[DD/MM/YYYY]@created_on',
        dateRange: { interval: 'day', min: '2025-09-01T00:00:00Z', max: '2025-09-30T23:59:59Z' },
        spread: true,
      },
      {
        using: 'record_owner',
        columns: 'userName:name',
        spread: true,
        filters: { rules: [{ field: 'primary_role', operator: '=', value: 'role-uuid' }] },
      },
    ],
    hideEmptyRows: false,
    orderBy: 'day ASC',
  },
  calculations: [
    { field: 'id', func: 'count', name: 'total' },
    { field: 'amount', func: 'sum', name: 'totalSold' },
  ],
}
```

### Date Range Intervals
`day`, `week`, `month`, `year`, `hour`, `minute`, `second`

---

## Expand

Expand relation/user/enum fields to return full objects instead of IDs:
```typescript
expand: ['record_owner', 'related_account', 'status']
```

---

## Complete Examples

### Full-Featured Select
```typescript
collection.list({
  columns: ['id', 'task_name', '...record_owner(owner_name:name)', '...related_account(account_name:name)'],
  filters: {
    combinator: 'and',
    rules: [
      { field: 'task_status', operator: 'in', value: [1, 2] },
      { field: 'due_date', operator: 'in_next_x_days', value: 7 },
      { field: 'record_owner', operator: 'in_active_user_team' },
    ],
  },
  orderBy: 'due_date ASC',
  limit: 50,
  fullCount: true,
})
```

### Monthly Sales Report
```typescript
collection.list({
  columns: ['months_of_year@created_on', '...category(cat:name)'],
  calculations: [
    { field: 'id', func: 'count', name: 'order_count' },
    { field: 'total_amount', func: 'sum', name: 'revenue' },
    { field: 'total_amount', func: 'avg', name: 'avg_order' },
  ],
  filters: { rules: [
    { field: 'created_on', operator: 'this_year' },
    { field: 'order_status', operator: '!=', value: 'cancelled' },
  ]},
  orderBy: 'months_of_year@created_on ASC',
})
```

### Customers with Nested Orders
```typescript
collection.list({
  columns: ['id', 'name', 'email', 'recent_orders'],
  childQueries: {
    recent_orders: {
      from: 'shop_order',
      using: 'customer',
      columns: ['id', 'order_date', 'total_amount'],
      orderBy: 'order_date DESC',
      limit: 10,
      filters: { rules: [{ field: 'order_date', operator: 'last_90_days' }] },
    },
  },
  filters: { rules: [{ field: 'status', operator: '=', value: 'active' }] },
  limit: 25,
})
```

### Allowed SQL Functions (Postgres)

**String**: `length`, `lower`, `upper`, `substr`, `replace`, `concat`, `trim`, `ltrim`, `rtrim`, `btrim`, `split_part`, `initcap`, `reverse`, `strpos`, `lpad`, `rpad`

**Number**: `abs`, `ceil`, `floor`, `round`, `sqrt`, `power`, `mod`, `greatest`, `least`, `trunc`

**Date/Time**: `now`, `age`, `date_part`, `date_trunc`, `extract`, `to_timestamp`, `to_char`, `to_date`, `make_date`

**Utility**: `coalesce`

**JSON**: `jsonb_array_length`, `jsonb_extract_path`, `jsonb_extract_path_text`, `jsonb_build_object`, `json_build_object`

### Allowed Cast Types
`int`, `bigint`, `real`, `float`, `numeric`, `decimal`, `money`, `timestamp`, `timestamptz`, `date`, `time`, `interval`, `bool`, `boolean`, `uuid`, `text` (and array variants)
