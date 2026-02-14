# SQL Block Formula Reference

## Formula Types

`IQueryFormula = IQuerySimpleFormula | IQueryBlockFormula`

**Simple** — legacy flat function call: `{ func: string, args: (string|number)[] }`. Column refs use `{col}` syntax. Detected by `func` property.

**Block Inline** — AST expression in SELECT: `{ alias?: string, inputs: IQueryFormulaBlock[] }`. Detected by `inputs` without `from`/`with`.

**Block Subquery** — correlated subquery on child table: `{ alias?, inputs, from: string, with: string | Record<string,string>, filters?: IQueryFilterGroup }`. Detected by `from`+`with`.

Compat wrapper: `{ expression: { from, with, inputs } }` is also accepted.

## Block Schema

Top-level requires exactly 1 element in `inputs[]`. Optional `alias` becomes SQL alias.

Every block has optional `tz?: string` (timezone) and `cast?: string` (type cast). Processing: compile → tz → cast.

## Block Kinds

### literal
`{ kind: "literal", literal: string|number|boolean|Date|null|Array }`
- Scalars → parameterized `$N`. Arrays → `($1, $2, ...)`.
- Inside `concat`/`concat_ws` parent: auto-casts (`::text`, `::boolean`, `::timestamptz`, `::jsonb`).

### column
`{ kind: "column", name: string|string[] }`
- Advanced DS: `"alias"."slug"`
- Simple DS custom fields: `"alias".data->>'<field-uuid>'` with auto-cast by field type:
  - number/money/duration(decimal≠false) → `::decimal`, (decimal=false) → `::int`
  - DB types jsonb/date/time/timestamptz/boolean/int* → `::<type>`, uuid[] → `::jsonb`
- Simple DS system fields: direct reference. Field not found → error.

### builtin
`{ kind: "builtin", name: "current_date"|"current_time"|"current_timestamp"|"now" }`
- Emitted as raw SQL. Other names → error.

### function
`{ kind: "function", name: string, inputs?: Block[] }`
- Validated against per-dialect whitelist. Inputs compiled recursively, joined by commas.

### extract
`{ kind: "extract", part: "year"|"month"|"day"|"hour"|"minute"|"second", inputs: [Block] }`
- Exactly 1 input required.
- Postgres/MySQL: `extract(<part> from <expr>)`. ClickHouse: `toYear(...)`, `toMonth(...)`, etc.

### aggregate
`{ kind: "aggregate", name: "count"|"sum"|"avg"|"min"|"max"|"jsonb_agg"|"json_agg"|"array_agg", distinct?: boolean, inputs: Block[] }`
- `count` with empty inputs → `count(*)`. `distinct` → `DISTINCT` keyword.

### math
`{ kind: "math", op: "+"|"-"|"*"|"/"|"%", inputs: Block[] }`
- Min 2 operands. Left-associative with parens: `((a op b) op c)`.

### case
`{ kind: "case", cases: [{when: Block, then: Block}], else?: Block }`
- Min 1 case required. `else` optional (defaults NULL).

### compare
`{ kind: "compare", op: "="|"!="|"<>"|">"|"<"|">="|"<="|"like"|"ilike"|"in"|"not in", left: Block, right: Block }`
- `in`/`not in`: `left in right`. `ilike` → `like` on MySQL.

### boolean
`{ kind: "boolean", op: "and"|"or"|"not", inputs: Block[] }`
- `not`: exactly 1 input → `not (<expr>)`. `and`/`or`: min 2 → `((<a>) op (<b>))`.

## Subquery Details

- `from`: child table full slug (`appSlug_tableSlug`), matched via `dataSource.children`.
- `with` (string): child field joins to parent `id`. `with` (object): `{ childField: parentField }`.
- Simple child DS: table rewritten to `tenant_record`, fields use `data->>'uuid'` refs.
- Optional `filters` apply WHERE on child table.
- Child alias: `t0_child`. Parent alias: `t0`.

## Allowed Functions

**Postgres**: length, lower, upper, substr, replace, concat, trim, ltrim, rtrim, btrim, split_part, initcap, reverse, strpos, lpad, rpad, abs, ceil, floor, round, sqrt, power, mod, gcd, lcm, exp, ln, log, log10, log1p, pi, sign, width_bucket, trunc, greatest, least, now, age, clock_timestamp, date_part, date_trunc, extract, isfinite, justify_days, justify_hours, make_date, make_time, make_timestamp, make_timestamptz, timeofday, to_timestamp, to_char, to_date, to_time, coalesce, jsonb_array_length, jsonb_extract_path, jsonb_extract_path_text, jsonb_object_keys, jsonb_build_object, json_build_object, jsonb_agg, json_agg, array_agg, array_to_json, row_to_json

**MySQL**: length, lower, upper, substr, substring, replace, concat, trim, ltrim, rtrim, left, right, reverse, locate, lpad, rpad, abs, ceil, ceiling, floor, round, sqrt, power, pow, mod, exp, ln, log, log10, sign, truncate, greatest, least, now, curdate, curtime, current_date, current_time, current_timestamp, date, time, year, month, day, hour, minute, second, date_format, str_to_date, unix_timestamp, from_unixtime, coalesce, json_value, json_extract, ifnull, format

**ClickHouse**: length, lower, upper, substr, substring, replace, concat, trim, ltrim, rtrim, reverse, position, leftPad, rightPad, abs, ceil, floor, round, sqrt, pow, mod, exp, ln, log, log10, sign, trunc, greatest, least, now, today, yesterday, toYear, toMonth, toDayOfMonth, toHour, toMinute, toSecond, formatDateTime, parseDateTime, coalesce, ifnull, multiif

**Aggregates** (all dialects): count, sum, avg, min, max, jsonb_agg, json_agg, array_agg

## Cast Types

Allowed: int, int2, int4, int8, bigint, real, float, float4, float8, numeric, double, decimal, money, timestamp, timestamptz, date, time, interval, bool, boolean, uuid, text (+ array variants like `int[]`, `text[]`).

## Timezone

`tz` property: validated `/^[a-zA-Z0-9_]+$/`. Postgres: `<expr> at time zone '<tz>'`. Column/function blocks omit outer parens.

## Validation Errors

| Condition | Error |
|---|---|
| Empty inputs | "Formula must have at least one input block" |
| >1 root input | "Multiple input blocks not yet supported" |
| Bad function | `Function "${name}" is not allowed for dialect "${dialect}"` |
| Bad aggregate | `Aggregate function "${name}" is not allowed` |
| Extract ≠1 input | "EXTRACT requires exactly one input expression" |
| Math <2 ops | "Math operations require at least 2 operands" |
| NOT ≠1 op | "NOT operation requires exactly one operand" |
| AND/OR <2 ops | "${OP} operation requires at least 2 operands" |
| CASE 0 whens | "CASE expression must have at least one WHEN clause" |
| Bad tz | "Unsupported timezone: ${tz}" |
| Bad builtin | "Unsupported formula function: ${name}" |

## SelectQueryBuilder Integration

1. Formulas in `ISelectQueryParams.formulas` as `Record<string, IQueryFormula>`.
2. Column alias matching formula key → formula replaces column ref in SELECT.
3. Dispatch: `func` → `buildSimpleFormula()`, `from`/`expression` → `buildBlockFormula()` (subquery), `inputs` only → `buildBlockFormula()` (inline).
4. Calculations with `func:"formula"` also route through `buildFormula()`.
5. `usedFormulas` set prevents duplicate application across SELECT and aggregations.
6. Subquery formulas trigger async `resolveChildDatasources()` before build.

## Examples

**Inline math** (balance / 100):
```json
{ "inputs": [{ "kind": "math", "op": "/", "inputs": [{ "kind": "column", "name": "balance" }, { "kind": "literal", "literal": 100 }] }] }
```

**Subquery count**:
```json
{ "from": "app_child", "with": "parent_id", "inputs": [{ "kind": "aggregate", "name": "count", "inputs": [] }] }
```

**CASE with AND**:
```json
{ "inputs": [{ "kind": "case", "cases": [{ "when": { "kind": "boolean", "op": "and", "inputs": [{ "kind": "compare", "op": ">", "left": { "kind": "column", "name": "price" }, "right": { "kind": "literal", "literal": 100 } }, { "kind": "compare", "op": "ilike", "left": { "kind": "column", "name": "name" }, "right": { "kind": "literal", "literal": "%pro%" } }] }, "then": { "kind": "literal", "literal": "premium" } }], "else": { "kind": "literal", "literal": "standard" } }] }
```

**Nested aggregate**: `round(sum(qty * price), 2)`:
```json
{ "alias": "total", "inputs": [{ "kind": "function", "name": "round", "inputs": [{ "kind": "aggregate", "name": "sum", "inputs": [{ "kind": "math", "op": "*", "inputs": [{ "kind": "column", "name": "qty" }, { "kind": "column", "name": "price" }] }] }, { "kind": "literal", "literal": 2 }] }] }
```

**Timezone**: `to_char(now() at time zone 'UTC', 'YYYY-MM-DD')`:
```json
{ "inputs": [{ "kind": "function", "name": "to_char", "inputs": [{ "kind": "function", "name": "now", "tz": "UTC" }, { "kind": "literal", "literal": "YYYY-MM-DD" }] }] }
```
