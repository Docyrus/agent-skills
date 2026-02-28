# Checklist Details

Detailed explanation, detection pattern, and fix example for every check in the Docyrus API Doctor.

---

## BUG Checks

### B1 — Missing `columns` parameter

**Detect:** A `.list()` or `.get()` call where the options object has no `columns` property.

**Why:** The API returns only the `id` field by default. Every other field is omitted unless explicitly requested.

```typescript
// BAD
const items = await collection.list({
  filters: { rules: [{ field: 'status', operator: '=', value: 'active' }] },
})
// items = [{ id: '...' }, { id: '...' }]  — no useful data

// GOOD
const items = await collection.list({
  columns: ['id', 'name', 'status'],
  filters: { rules: [{ field: 'status', operator: '=', value: 'active' }] },
})
```

---

### B2 — `limit: 0` in query payload

**Detect:** `limit` property set to `0` in any query payload.

**Why:** `limit: 0` causes an API error. If you don't need to limit results, omit `limit` entirely (defaults to 100). If you need aggregation only, use `calculations` without `limit`.

```typescript
// BAD — causes API error
const result = await collection.list({
  columns: ['id'],
  limit: 0,
  fullCount: true,
})

// GOOD — use calculations for counts
const result = await collection.list({
  columns: ['id'],
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
})
```

---

### B3 — Child query key not in `columns`

**Detect:** A `childQueries` object with a key (e.g., `orders`) that does not appear in the `columns` array/string.

**Why:** The API silently drops child query results if their key isn't in `columns`.

```typescript
// BAD — child query results won't appear
const result = await collection.list({
  columns: ['id', 'name'],
  childQueries: {
    orders: {
      from: 'base_order',
      using: 'customer',
      columns: ['id', 'total'],
    },
  },
})

// GOOD — include child query key in columns
const result = await collection.list({
  columns: ['id', 'name', 'orders'],
  childQueries: {
    orders: {
      from: 'base_order',
      using: 'customer',
      columns: ['id', 'total'],
    },
  },
})
```

---

### B4 — Formula key not in `columns`

**Detect:** A `formulas` object with a key (e.g., `total`) that does not appear in `columns`.

**Why:** Same as B3 — formula results are dropped if the key isn't in `columns`.

```typescript
// BAD
const result = await collection.list({
  columns: ['id', 'name'],
  formulas: {
    total: { func: 'sum', inputs: [{ kind: 'column', column: 'amount' }] },
  },
})

// GOOD
const result = await collection.list({
  columns: ['id', 'name', 'total'],
  formulas: {
    total: { func: 'sum', inputs: [{ kind: 'column', column: 'amount' }] },
  },
})
```

---

### B5 — Aggregation via `@` column syntax

**Detect:** Column strings containing aggregation function patterns like `count@`, `sum@`, `avg@`, `min@`, `max@`.

**Why:** The `@` syntax in columns is for pre-defined date/grouping functions (e.g., `months_of_year@created_on`), not for aggregations. Use the `calculations` parameter for count/sum/avg/min/max.

```typescript
// BAD
const result = await collection.list({
  columns: ['status', 'count@id'],
})

// GOOD
const result = await collection.list({
  columns: ['status'],
  calculations: [{ func: 'count', field: 'id', name: 'count' }],
})
```

---

### B6 — `distinctColumns` with `calculations`

**Detect:** Both `distinctColumns` and `calculations` present in the same query payload.

**Why:** These features are mutually exclusive. `distinctColumns` selects distinct rows; `calculations` performs aggregation. Combining them produces unexpected results.

```typescript
// BAD
const result = await collection.list({
  columns: ['status'],
  distinctColumns: ['status'],
  calculations: [{ func: 'count', field: 'id', name: 'count' }],
})

// GOOD — use calculations for aggregation
const result = await collection.list({
  columns: ['status'],
  calculations: [{ func: 'count', field: 'id', name: 'count' }],
})
```

---

### B7 — Formula `extract` input count

**Detect:** A formula block with `kind: 'extract'` (or `func: 'extract'`) that has `inputs.length !== 1`.

**Why:** PostgreSQL EXTRACT requires exactly one input expression. Zero or multiple inputs cause a server error.

```typescript
// BAD — 0 inputs
{ kind: 'extract', part: 'month', inputs: [] }

// BAD — 2 inputs
{ kind: 'extract', part: 'month', inputs: [
  { kind: 'column', column: 'start_date' },
  { kind: 'column', column: 'end_date' },
]}

// GOOD
{ kind: 'extract', part: 'month', inputs: [
  { kind: 'column', column: 'start_date' },
]}
```

---

### B8 — Formula `not` operand count

**Detect:** A formula block with `kind: 'not'` that has `inputs.length !== 1`.

**Why:** NOT is a unary operator. Multiple operands cause: "NOT operation requires exactly one operand".

---

### B9 — Formula `and`/`or` operand count

**Detect:** A formula block with `kind: 'and'` or `kind: 'or'` that has `inputs.length < 2`.

**Why:** Boolean AND/OR require at least 2 operands. A single operand causes: "${OP} operation requires at least 2 operands".

---

### B10 — Formula math operand count

**Detect:** A formula block with `kind: 'math'` and `operator` one of `+`, `-`, `*`, `/`, `%` that has `inputs.length < 2`.

**Why:** Math is left-associative binary. Fewer than 2 operands causes: "Math operations require at least 2 operands".

---

### B11 — Formula `case` without `when`

**Detect:** A formula block with `kind: 'case'` that has empty or missing `whens` array.

**Why:** CASE must have at least one WHEN clause. Empty cases cause: "CASE expression must have at least one WHEN clause".

---

### B12 — Uncast literal in `jsonb_build_object`

**Detect:** A formula using `jsonb_build_object` (or `json_build_object`) that contains `kind: 'literal'` blocks without a `cast` property.

**Why:** PostgreSQL cannot determine parameter types for literals inside `jsonb_build_object`. Auto-cast only works inside `concat`/`concat_ws`. All other functions need explicit `"cast": "text"` (or the appropriate type).

```typescript
// BAD — Postgres error: could not determine data type of parameter
{
  func: 'jsonb_build_object',
  inputs: [
    { kind: 'literal', literal: 'label' },  // missing cast
    { kind: 'column', column: 'name' },
  ],
}

// GOOD
{
  func: 'jsonb_build_object',
  inputs: [
    { kind: 'literal', literal: 'label', cast: 'text' },
    { kind: 'column', column: 'name' },
  ],
}
```

---

## PERFORMANCE Checks

### P1 — Unnecessary `limit` on aggregation queries

**Detect:** A query that has `calculations` but also includes a `limit` property, and the consuming code only reads the aggregated values (not raw row data).

**Why:** When using `calculations`, the result is already grouped/aggregated. Sending `limit` adds unnecessary constraint and suggests confusion about how aggregation works.

```typescript
// BAD — limit is unnecessary
const result = await collection.list({
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
  limit: 1,
})

// GOOD
const result = await collection.list({
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
})
```

---

### P2 — `fullCount` just to get a total count

**Detect:** A query with `fullCount: true` where the consuming code only reads `.fullCount` and ignores the actual rows, or uses `limit: 0`/`limit: 1` to minimize rows.

**Why:** `fullCount` is designed for pagination — showing "Page 1 of 50" alongside actual data. If you only need the count, `calculations` is more efficient because it doesn't fetch any row data.

```typescript
// BAD — fetching rows just to read fullCount
const result = await collection.list({
  columns: ['id'],
  limit: 1,
  fullCount: true,
})
const count = result.fullCount

// GOOD — aggregation only
const result = await collection.list({
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
})
const count = result[0]?.total ?? 0
```

---

### P3 — Unnecessary `columns` on calculation-only queries

**Detect:** A query that has `calculations` and the consuming code only reads the aggregated values, but also includes a `columns` parameter.

**Why:** When you only need aggregated results (e.g., a total count for a stat card), sending `columns` is unnecessary overhead. The calculation result is returned without needing column selection.

```typescript
// BAD — columns is unnecessary for a pure count
const result = await collection.list({
  columns: ['id'],
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
})

// GOOD — calculations only
const result = await collection.list({
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
})
```

**Note:** `columns` IS needed alongside `calculations` when you also need grouped data (e.g., count per status requires `columns: ['status']` to group by status).

---

### P4 — Over-fetching columns

**Detect:** Columns in the `columns` parameter that are never referenced in the code that consumes the response.

**Why:** Each column adds to query execution time and response payload size. Only select what you actually use.

```typescript
// BAD — description and notes are never rendered
const result = await collection.list({
  columns: ['id', 'name', 'status', 'description', 'notes', 'created_on', 'modified_on'],
})
// Only renders: name, status

// GOOD
const result = await collection.list({
  columns: ['id', 'name', 'status'],
})
```

---

### P5 — Large `limit` without pagination

**Detect:** `limit` set to > 200 without corresponding `offset` and `fullCount` pagination logic.

**Why:** Large result sets slow down both the API response and client-side rendering. Use pagination to load data incrementally.

```typescript
// BAD — loading everything at once
const result = await collection.list({
  columns: ['id', 'name', 'status'],
  limit: 5000,
})

// GOOD — paginated
const result = await collection.list({
  columns: ['id', 'name', 'status'],
  limit: 50,
  offset: page * 50,
  fullCount: true,
})
```

---

### P6 — Missing `expand` causing N+1

**Detect:** Code that accesses `.name` or other sub-properties of a relation/enum/user field, but that field is not in `expand`.

**Why:** Without `expand`, relation fields return only the ID string. If you then need the name, you'd have to make a separate API call per record (N+1 problem). With `expand`, the full object comes in the original response.

```typescript
// BAD — status is just an ID string without expand
const result = await collection.list({
  columns: ['id', 'name', 'status'],
})
// result[0].status = 'uuid-string' — can't render name

// GOOD
const result = await collection.list({
  columns: ['id', 'name', 'status'],
  expand: ['status'],
})
// result[0].status = { id: '...', name: 'Active', color: '#00ff00' }
```

---

### P7 — Fetching rows for existence checks

**Detect:** Fetching rows with `.list()` just to check `result.length > 0` or similar existence logic.

**Why:** Fetching full records to check existence wastes bandwidth and query time. A count calculation returns a single number.

```typescript
// BAD
const items = await collection.list({
  columns: ['id'],
  filters: { rules: [{ field: 'email', operator: '=', value: email }] },
})
const exists = items.length > 0

// GOOD
const result = await collection.list({
  calculations: [{ func: 'count', field: 'id', name: 'total' }],
  filters: { rules: [{ field: 'email', operator: '=', value: email }] },
})
const exists = (result[0]?.total ?? 0) > 0
```

---

### P8 — Redundant overlapping queries

**Detect:** Two or more `useQuery` calls hitting the same data source with overlapping columns and compatible filters that could be merged into one query.

**Why:** Each query is a separate HTTP round-trip. Combining them reduces latency and server load.

---

## CODE QUALITY Checks

### Q1 — Heavy `as` type assertions on responses

**Detect:** API response cast with `as Array<SomeType>` or `as unknown as SomeType` without runtime checks.

**Why:** `as` is a compile-time assertion — it doesn't validate data at runtime. If the API response shape changes, the code silently breaks.

```typescript
// BAD
const result = (await collection.list({
  columns: ['id', 'name'],
})) as Array<{ id: string; name: string }>

// BETTER — use collection's typed return
const result = await collection.list({
  columns: ['id', 'name'],
})
// Collection hooks should return typed results
```

---

### Q2 — Missing `enabled` on dependent queries

**Detect:** A `useQuery` call where the `queryFn` references a variable (like `recordId`, `userId`) that could be `null`/`undefined` at mount time, but there's no `enabled` option.

**Why:** Without `enabled`, the query fires immediately with the undefined value, causing a failed or meaningless API call.

```typescript
// BAD — fires with undefined recordId on mount
const { data } = useQuery({
  queryKey: ['task', recordId],
  queryFn: () => collection.get(recordId, { columns: ['id', 'name'] }),
})

// GOOD
const { data } = useQuery({
  queryKey: ['task', recordId],
  queryFn: () => collection.get(recordId!, { columns: ['id', 'name'] }),
  enabled: !!recordId,
})
```

---

### Q3 — No error handling on mutations

**Detect:** `.create()`, `.update()`, `.delete()`, `.deleteMany()` calls without try/catch, `.catch()`, or a mutation error handler.

**Why:** Mutations can fail (network errors, validation errors, permission errors). Without error handling, the user gets no feedback and the UI may be left in an inconsistent state.

```typescript
// BAD
const handleSave = async (values: Record<string, unknown>) => {
  await collection.create(values)
  onSuccess?.()
}

// GOOD
const handleSave = async (values: Record<string, unknown>) => {
  try {
    await collection.create(values)
    onSuccess?.()
  } catch (error) {
    toast.error('Failed to create record')
  }
}
```

---

### Q4 — Missing query invalidation after mutations

**Detect:** A mutation (create/update/delete) that does not call `queryClient.invalidateQueries()` afterward.

**Why:** Without invalidation, list views show stale data. The user creates/updates/deletes a record but the table still shows old data until a manual refresh.

```typescript
// BAD
await collection.update(recordId, values)
onClose()

// GOOD
await collection.update(recordId, values)
await queryClient.invalidateQueries({ queryKey: ['tasks'] })
onClose()
```

---

### Q5 — Serial cache invalidations

**Detect:** Multiple `await queryClient.invalidateQueries(...)` statements in sequence where they don't depend on each other.

**Why:** Independent invalidations can run concurrently. Sequential awaits add unnecessary latency.

```typescript
// BAD — sequential
await queryClient.invalidateQueries({ queryKey: ['task', recordId] })
await queryClient.invalidateQueries({ queryKey: ['tasks'] })
await queryClient.invalidateQueries({ queryKey: ['dashboard'] })

// GOOD — parallel
await Promise.all([
  queryClient.invalidateQueries({ queryKey: ['task', recordId] }),
  queryClient.invalidateQueries({ queryKey: ['tasks'] }),
  queryClient.invalidateQueries({ queryKey: ['dashboard'] }),
])
```

---

### Q6 — Using deprecated `expandTypes`

**Detect:** The `expandTypes` property in a query payload.

**Why:** `expandTypes` is deprecated. Use `expand` with an array of field slugs instead.

```typescript
// BAD
collection.list({
  columns: ['id', 'status'],
  expandTypes: ['enum'],
})

// GOOD
collection.list({
  columns: ['id', 'status'],
  expand: ['status'],
})
```

---

### Q7 — Hardcoded data source paths

**Detect:** Direct `client.get('/v1/apps/.../data-sources/.../items')` calls instead of using generated collection hooks.

**Why:** Collection hooks encapsulate the endpoint path, provide typed methods, and automatically use the authenticated client. Hardcoded paths are fragile and bypass these benefits.

```typescript
// BAD
const client = useDocyrusClient()
const tasks = await client!.get('/v1/apps/base/data-sources/task/items', {
  columns: 'id, name',
})

// GOOD
const collection = useBaseTaskCollection()
const tasks = await collection.list({
  columns: ['id', 'name'],
})
```

---

### Q8 — `distinctColumns` without `orderBy`

**Detect:** A query payload with `distinctColumns` but no `orderBy`.

**Why:** `distinctColumns` uses PostgreSQL `DISTINCT ON` which requires `ORDER BY` to determine which row is kept per group. Without it, the selected row is non-deterministic.

```typescript
// BAD — non-deterministic result
collection.list({
  columns: ['id', 'customer', 'status'],
  distinctColumns: ['customer'],
})

// GOOD
collection.list({
  columns: ['id', 'customer', 'status', 'created_on'],
  distinctColumns: ['customer'],
  orderBy: { field: 'created_on', direction: 'desc' },
})
```
