# Querying & Data Operations

## Record CRUD

- Create, read, update, delete individual records
- Bulk create, bulk update, bulk delete (batched for performance)
- Insert/update/delete with custom return value selection

## Record Comments & Files

- Record-scoped comments with create, list, fetch by ID, update, and delete operations
- Comment payloads can include threading (`parentId`), assignee targeting (`assignedTo`), attachments metadata, level, status, and done state
- Record-scoped file attachments with upload, list, fetch by ID, insert-without-upload, copy/move, and delete operations
- File uploads support multipart form data, public/private storage selection, and record association

## Query Engine

Every list/get call accepts a structured query payload:

- **Column selection** — Pick specific fields, alias them, spread related data, apply functions
- **Filtering** — 50+ operators across comparison, text matching, date shortcuts (`today`, `this_week`, `last_30_days`), user-relative filters (`active_user`, `in_active_user_scope`), array containment, and null checks. Filter groups support AND/OR/NOT nesting
- **Sorting** — Multi-field, directional
- **Pagination** — Limit/offset with optional full count
- **Aggregations** — count, sum, avg, min, max with grouping
- **Formulas** — Virtual computed columns via JSONata expressions
- **Pivot** — Cross-tab matrix generation with date range series support
- **Child queries** — Fetch related records as nested JSON arrays in a single request
- **Expand** — Return full objects for relation and enum fields instead of IDs
- **Keyword search** — Full-text search across multiple fields

## Detailed References

- For complete query payload reference with all parameters, operators, and examples, see [data-source-query-guide.md](data-source-query-guide.md).
- For block formula design (inline expressions and correlated subqueries), see [formula-design-guide-llm.md](formula-design-guide-llm.md).
