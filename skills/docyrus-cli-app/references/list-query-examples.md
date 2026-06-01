# List Query Examples

Practical examples for `docyrus ds list` with columns, filters, sorting, pagination, and the advanced query engine (keyword search, calculations, formulas, pivots, child queries).

JSON flags (`--filters`, `--calculations`, `--formulas`, `--pivot`, `--childQueries`, etc.) take the same payload shapes the `/items` query endpoint accepts. For the full payload reference see the `docyrus-platform` skill's `data-source-query-guide.md`.

---

## Table of Contents

1. [Basic Listing](#basic-listing)
2. [Column Selection](#column-selection)
3. [Filtering](#filtering)
4. [Sorting](#sorting)
5. [Pagination](#pagination)
6. [Combined Examples](#combined-examples)
7. [Advanced Queries](#advanced-queries)

---

## Basic Listing

List all records with default columns:

```bash
docyrus ds list crm contacts
```

List with JSON output:

```bash
docyrus ds list crm contacts --format json
```

---

## Column Selection

Select specific fields:

```bash
docyrus ds list crm contacts --columns "name, email, phone"
```

Select with relation expansion (get related account's name):

```bash
docyrus ds list crm contacts --columns "name, email, related_account(account_name)"
```

Spread related fields into root object:

```bash
docyrus ds list crm contacts --columns "name, ...related_account(account_name, account_phone)"
```

Alias columns:

```bash
docyrus ds list crm contacts --columns "n:name, e:email, p:phone"
```

---

## Filtering

### Single field equals

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"status","operator":"=","value":"active"}]}'
```

### Multiple conditions (AND)

```bash
docyrus ds list crm contacts --filters '{"combinator":"and","rules":[{"field":"status","operator":"=","value":"active"},{"field":"priority","operator":">=","value":3}]}'
```

### OR conditions

```bash
docyrus ds list crm contacts --filters '{"combinator":"or","rules":[{"field":"city","operator":"=","value":"Istanbul"},{"field":"city","operator":"=","value":"Ankara"}]}'
```

### IN operator (match any value in list)

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"status","operator":"in","value":[1,2,3]}]}'
```

### Not equal

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"status","operator":"!=","value":"archived"}]}'
```

### Text search with LIKE

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"name","operator":"like","value":"John"}]}'
```

### Empty / not empty

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"email","operator":"not empty"}]}'
```

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"phone","operator":"empty"}]}'
```

### Date shortcuts

Records created this month:

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"created_on","operator":"this_month"}]}'
```

Records created today:

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"created_on","operator":"today"}]}'
```

Records from last 30 days:

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"created_on","operator":"last_30_days"}]}'
```

### Date range with between

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"created_on","operator":"between","value":["2025-01-01","2025-06-30"]}]}'
```

### Filter on related field

Filter by related account's status:

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"rel_related_account/account_status","operator":"=","value":1}]}'
```

### Current user filter

Records owned by current user:

```bash
docyrus ds list crm contacts --filters '{"rules":[{"field":"record_owner","operator":"active_user"}]}'
```

### Nested AND + OR

Active contacts created this month OR contacts with high priority:

```bash
docyrus ds list crm contacts --filters '{"combinator":"or","rules":[{"combinator":"and","rules":[{"field":"status","operator":"=","value":"active"},{"field":"created_on","operator":"this_month"}]},{"field":"priority","operator":">=","value":5}]}'
```

---

## Sorting

Sort by name ascending (default):

```bash
docyrus ds list crm contacts --orderBy "name"
```

Sort by created date descending:

```bash
docyrus ds list crm contacts --orderBy "created_on DESC"
```

---

## Pagination

First 10 records:

```bash
docyrus ds list crm contacts --limit 10
```

Next 10 records (page 2):

```bash
docyrus ds list crm contacts --limit 10 --offset 10
```

Get total count alongside results:

```bash
docyrus ds list crm contacts --limit 10 --fullCount true
```

---

## Combined Examples

### Active contacts with email, sorted by name, paginated

```bash
docyrus ds list crm contacts \
  --columns "name, email, phone, created_on" \
  --filters '{"rules":[{"field":"status","operator":"=","value":"active"},{"field":"email","operator":"not empty"}]}' \
  --orderBy "name" \
  --limit 25 \
  --fullCount true
```

### Recent high-priority deals with account info

```bash
docyrus ds list crm deals \
  --columns "name, amount, stage, ...related_account(account_name)" \
  --filters '{"combinator":"and","rules":[{"field":"priority","operator":">=","value":4},{"field":"created_on","operator":"last_30_days"}]}' \
  --orderBy "amount DESC" \
  --limit 20
```

### Search for records by keyword

```bash
docyrus ds list crm contacts \
  --columns "name, email, phone" \
  --filters '{"rules":[{"field":"name","operator":"like","value":"Smith"}]}' \
  --format json
```

### Export all records as JSON lines

```bash
docyrus ds list crm contacts \
  --columns "id, name, email, phone, status, created_on" \
  --format jsonl
```

### Records owned by current user, created this quarter

```bash
docyrus ds list crm tasks \
  --columns "name, status, priority, due_date" \
  --filters '{"combinator":"and","rules":[{"field":"record_owner","operator":"active_user"},{"field":"created_on","operator":"this_quarter"}]}' \
  --orderBy "due_date" \
  --limit 50
```

---

## Advanced Queries

### Keyword search

Full-text keyword filter across searchable fields:

```bash
docyrus ds list crm contacts --columns "name, email" --filterKeyword "renewal"
```

### Calculations (aggregations / group-by)

Group by a column (via `--columns`) and aggregate with `--calculations`. Each rule is `{ field, func, name, isDistinct? }` (`func`: `count`, `sum`, `avg`, `min`, `max`). Add `--groupSummaries` to include per-group summary rows.

```bash
# Count open tasks per owner
docyrus ds list crm tasks \
  --columns "record_owner(name)" \
  --calculations '[{"field":"id","func":"count","name":"open_tasks"}]' \
  --filters '{"rules":[{"field":"task_status","operator":"=","value":1}]}' --json

# Multiple aggregations per category, with group summaries
docyrus ds list crm deals \
  --columns "category" \
  --calculations '[{"field":"id","func":"count","name":"total"},{"field":"amount","func":"sum","name":"totalAmount"},{"field":"amount","func":"avg","name":"avgAmount"}]' \
  --groupSummaries --json

# Distinct count
docyrus ds list crm contacts \
  --calculations '[{"field":"email","func":"count","name":"unique_emails","isDistinct":true}]' --json
```

### Formulas (computed columns)

`--formulas` is a `{ name: definition }` object; each name must also appear in `--columns`.

```bash
# Inline math formula
docyrus ds list crm accounts \
  --columns "id, name, balance_pct" \
  --formulas '{"balance_pct":{"inputs":[{"kind":"math","op":"/","inputs":[{"kind":"column","name":"balance"},{"kind":"literal","literal":100}]}]}}' --json

# Subquery formula (count child records)
docyrus ds list crm accounts \
  --columns "id, name, contacts_count" \
  --formulas '{"contacts_count":{"from":"crm_contacts","with":"account","inputs":[{"kind":"aggregate","name":"count","inputs":[]}]}}' --json
```

### Pivot

`--pivot` builds a matrix of cross-joined dimensions left-joined onto the main query, so every combination appears even with zero matches.

```bash
docyrus ds list crm orders \
  --columns "...order_status(orderStatus:name)" \
  --pivot '{"matrix":[{"using":"created_on","columns":"day:to_char[DD/MM/YYYY]@created_on","dateRange":{"interval":"day","min":"2025-09-01T00:00:00Z","max":"2025-09-30T00:00:00Z"},"spread":true}]}' --json
```

### Child queries (nested records)

`--childQueries` embeds related rows under an alias that must also appear in `--columns`. Each entry is `{ alias, from, using, columns?, filters?, calculations?, orderBy?, limit? }`; `from` uses `appSlug_slug` and `using` is the child field referencing the parent `id`.

```bash
docyrus ds list crm accounts \
  --columns "id, name, recent_deals" \
  --childQueries '[{"alias":"recent_deals","from":"crm_deals","using":"account","columns":"name, amount","orderBy":"created_on DESC","limit":5}]' --json
```

### Expand and distinct columns

```bash
# Expand related/nested columns
docyrus ds list crm contacts --columns "name, email" --expand "related_account" --json

# Deduplicate rows by selected columns (e.g. one row per email)
docyrus ds list crm contacts --columns "email, name, created_on" --distinctColumns "email" --json
```
