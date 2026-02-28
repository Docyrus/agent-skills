# List Query Examples

Practical examples for `docyrus ds list` with columns, filters, sorting, and pagination.

---

## Table of Contents

1. [Basic Listing](#basic-listing)
2. [Column Selection](#column-selection)
3. [Filtering](#filtering)
4. [Sorting](#sorting)
5. [Pagination](#pagination)
6. [Combined Examples](#combined-examples)

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
