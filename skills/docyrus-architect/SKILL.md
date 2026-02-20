---
name: docyrus-architect
description: Use the Docyrus Architect MCP tools to manage data sources, fields, enums, apps, custom queries, and query data in the Docyrus platform. Use when the user asks to create, update, delete, or query data sources, fields, enum options, apps, or custom query templates via the docyrus-architect MCP server. Also use when building reports, dashboards, or performing data analysis that requires querying Docyrus data sources or running custom SQL templates with filters, aggregations, formulas, pivots, or child queries.
---

# Docyrus Architect

Guide for using `docyrus-architect` MCP tools to manage and query data sources in Docyrus.

## Tool Overview

### Discovery Tools
- `get_apps` — List tenant apps. Use before `create_data_source` to find the target `tenantAppId`.
- `get_data_source_list` — Search data sources by name/description or app ID.
- `get_data_source_list_with_fields` — Same as above but includes field names and types.
- `get_data_source_metadata` — Get full metadata (fields with IDs, types, slugs, enums, relations) for a data source. **Always call this before querying** to discover field slugs and relation targets.
- `get_enums_by_field_id` — Get enum options for select/status/tagSelect fields.
- `read_current_user` / `read_tenant_user` — Get user info.

### Data Source CRUD
- `create_data_source` — Create a new data source (table). Default fields auto-created: `id`, `autonumber_id`, `name`, `record_owner`, `created_on`, `created_by`, `last_modified_by`, `last_modified_on`.
- `update_data_source` — Update data source properties.
- `delete_data_source` — Delete a data source and all its data.

### Field Management
- `create_fields` — Batch create fields. Set `relationDataSourceId` for `field-relation` types.
- `update_fields` — Batch update fields. Non-CUSTOM fields get customization records.
- `delete_fields` — Batch delete fields by ID.

### Enum Management
- `create_enums` — Create enum options for select/tagSelect/status fields. Pass `fieldId` for field-specific enums or `enumSetId` for shared enum sets.
- `update_enums` — Update enum option name/slug/color/icon.
- `delete_enums` — Delete enum options.

### OpenAPI Spec
- `regenerate_openapi_spec` — Regenerate and upload the tenant's OpenAPI spec after data source or field changes. Accepts optional `dataSourceSlugs` (string array) to limit scope; omit to include all data sources. Returns the `publicUrl` of the uploaded spec. **Call this after any `create_data_source`, `update_data_source`, `delete_data_source`, `create_fields`, `update_fields`, or `delete_fields` operation** to keep the spec in sync.

### Query & Compute
- `query_data_source` — Read data with filtering, sorting, aggregation, formulas, pivots, child queries. **See [references/data-source-query-guide.md](references/data-source-query-guide.md) for complete query syntax.**
- `evaluate_jsonata` — Test JSONata expressions. Use for validating computed field formulas.

### Custom Queries
- `get_custom_queries` — List non-archived custom queries for the tenant.
- `get_custom_query_by_id` — Read full custom query definition (`name`, `description`, `query`, `filters`).
- `create_custom_query` — Create a saved SQL template using Handlebars variables and optional filter definitions.
- `update_custom_query` — Update selected fields of an existing custom query (partial update).
- `delete_custom_query` — Soft-delete (archive) a custom query.
- `run_custom_query` — Execute a saved custom query with runtime filters, pagination offset, and optional simulate mode.

## Common Workflows

### Create a Data Source with Fields and Enums

1. Call `get_apps` to find the target app ID
2. Call `create_data_source` with title (plural), name (singular), slug (singular snake_case)
3. Call `create_fields` with all custom fields (default fields already exist)
4. For select/tagSelect/status fields, call `create_enums` with the field ID from step 3
5. Call `regenerate_openapi_spec` to update the OpenAPI spec

### Query Data

1. Call `get_data_source_metadata` to discover field slugs, types, and relations
2. Call `query_data_source` with appropriate columns, filters, and sorting
3. For advanced queries (aggregations, formulas, pivots, child queries), read [references/data-source-query-guide.md](references/data-source-query-guide.md)

### Modify Existing Data Source

1. Call `get_data_source_metadata` to see current fields
2. Use `create_fields` / `update_fields` / `delete_fields` as needed
3. For enum changes, use `get_enums_by_field_id` first, then `create_enums` / `update_enums` / `delete_enums`
4. Call `regenerate_openapi_spec` to update the OpenAPI spec

### Manage a Custom Query Lifecycle

1. Call `get_custom_queries` to find an existing query or confirm naming
2. Call `create_custom_query` with `name`, `query`, and optional `description`/`filters`
3. Call `get_custom_query_by_id` to verify the saved template and filter definitions
4. Use `update_custom_query` for targeted edits (rename, revise SQL template, update filter definitions)
5. Use `delete_custom_query` only when archival is explicitly requested

### Run and Debug a Custom Query

1. Call `get_custom_query_by_id` first to inspect SQL template and available filter slugs
2. Build runtime `filters` with `rules[].field` values that match the query's filter slugs
3. Call `run_custom_query` with `simulate: true` for complex or untrusted queries to inspect plan output
4. Call `run_custom_query` with `simulate: false` (or omitted) to fetch real data
5. Use `offset` for pagination and expect a max of 50,000 rows per execution

## Key Rules

### Data Source Creation
- `title` is plural (e.g., "Sales Orders"), `name` is singular (e.g., "Sales Order"), `slug` is singular snake_case (e.g., "sales_order")
- Use `defaultEditFormTarget: "tab"` for complex forms, `"side"` for simple ones
- Enable `pluginActivityView` for CRM-type data sources (leads, contacts, deals)
- Enable `pluginComments` for collaborative data sources
- Enable `pluginFile` when users need to attach files to records
- Enable `pluginDocyment` when users need rich text documents per record

### Field Types
- `field-relation` requires `relationDataSourceId` — the ID of the related data source
- `field-list` is a virtual field showing child records (one-to-many) — not stored in DB
- `field-select` / `field-tagSelect` / `field-status` need enum options created after the field
- `field-formula` uses JSONata expressions — test with `evaluate_jsonata` first
- `field-inlineData` stores array of objects, `field-inlineForm` stores single nested object
- Field `slug` must be snake_case matching `^[a-z][a-z0-9_]*$`

### Querying
- Use `dataSourceId` (UUID) to identify which data source to query
- `columns` is a comma-separated string of field slugs, not an array
- For aggregations, always use `id` field for `count` calculations
- Relation expansion: `relation_field(sub_field1, sub_field2)` selects nested columns
- Spread operator: `...relation_field(alias:sub_field)` flattens into root object
- Filter on related fields: `rel_{{relation_slug}}/{{field_slug}}`
- Date filters have shortcut operators like `today`, `this_month`, `last_30_days`

### query_data_source Required Parameters
All parameters are required in the MCP tool schema (most accept `null`):
- `dataSourceId`: string (required, non-null)
- `columns`: string | null
- `filters`: object | null
- `filterKeyword`: string | null
- `orderBy`: array | null
- `limit`: number | null (default: 1000)
- `offset`: number | null
- `fullCount`: boolean | null
- `recordId`: string | null (fetch single record by ID)
- `calculations`: array | null
- `distinctColumns`: array | null
- `formulas`: array | null
- `childQueries`: array | null
- `pivot`: object | null

### Custom Query Rules
- Treat custom query `query` content as a Handlebars SQL template, not static SQL.
- Use built-in context variables in templates when relevant: `TENANT_ID`, `TENANT_SCHEMA`, `USER_ID`, `USER_EMAIL`, `USER_FIRSTNAME`, `USER_LASTNAME`, `USER_FULLNAME`.
- Use `{{filter FILTERS.<slug> <column_expression>}}` for optional runtime filtering. If no runtime value is provided, the helper resolves to `1=1`.
- Keep `filters` definitions in `create_custom_query` / `update_custom_query` aligned with template usage:
  - template usage: `FILTERS.<slug>`
  - runtime rule field: `<slug>`
  - filter definition slug: `<slug>`
- Prefer `simulate: true` before running expensive queries to inspect `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)` output.
- Respect runtime limits on `run_custom_query`: 50,000 row cap, 15s timeout for normal execution, 30s timeout for simulate mode.
- Use `delete_custom_query` as an archival action (soft delete), not permanent deletion.
- For JSONB-backed simple data sources, access custom fields with `(table_alias."data"->>'field-uuid')::type`.

## References

- **[Data Source Query Guide](references/data-source-query-guide.md)** — Up-to-date reference for `query_data_source` including columns, filters, orderBy, pagination, calculations, formulas, pivots, child queries, and operator details.
- **[Formula Design Guide (LLM)](references/formula-design-guide-llm.md)** — Up-to-date guide for designing formula payloads used in query requests.
- **[Custom Query Guide](references/custom-query-guide.md)** — Full lifecycle and execution reference for `get_custom_queries`, `get_custom_query_by_id`, `create_custom_query`, `update_custom_query`, `delete_custom_query`, and `run_custom_query`.
