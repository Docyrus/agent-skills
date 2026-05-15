# Core Building Blocks

## Data Sources

Data sources are the fundamental building block. Each data source represents a structured collection of records with a defined schema. Every data source — whether backed by an internal database table or a connected external REST API — is exposed through a single, unified CRUD endpoint. This means consumers interact with all data sources the same way regardless of where the data lives.

**Types:**

- **Simple** — Schema-flexible records. Quick to set up, ideal for lightweight or rapidly evolving data.
- **Advanced** — Fully structured data sources with dedicated columns per field. Supports base data sources (shared field inheritance across variants) and high-performance querying.
- **External** — Connected external REST API resources or databases mapped as data sources. External resources are accessed through the same unified API as internal data sources, abstracting away the underlying connection.
- **System** — Pre-built templates for common entities: documents, threads, messages, contacts, organizations, activities, tasks, events, calendars, time entries, projects, and sections.

**Ownership models:** App-scoped, tenant-custom, product-provided, system-built-in, or user-specific.

## Fields

Every data source is composed of fields. 45+ field types cover all data modeling needs:

| Category | Field Types |
|---|---|
| **Text** | text, textarea, email, phone, URL, color, icon, display |
| **Rich content** | document editor, HTML editor, email editor, code editor |
| **Numeric** | number, money, currency, duration, rating, autonumber, identity |
| **Date & time** | date, dateTime, time, dateRange |
| **Boolean** | checkbox, switch |
| **Selection** | select, multiSelect, tagSelect, status, radioGroup |
| **Users** | userSelect, userMultiSelect |
| **Relations** | relation (lookup to another data source), list (virtual related records) |
| **Nested data** | inlineData (nested JSON arrays), inlineForm (nested objects) |
| **Files** | file, image, fileStorageFolder |
| **Computed** | formula (JSONata expressions), display (read-only computed) |
| **Workflow** | approvalStatus, taskList, todo |
| **Advanced** | json, queryBuilder, dynamic, schema, schemaRepeater, locationSelect |
| **System** | systemEnum, systemBuffer, systemVector, systemTextArray, systemUuidArray |

## Enumerations

Multi-option fields (select, multiSelect, status, tagSelect) support enumerations with color, icon, and ordering. Enum sets allow sharing option lists across multiple fields.

## Apps

Apps are the top-level container that groups data sources, views, forms, pages, navigation, and configurations into a deployable unit.

**Supported app types:** web, mobile, AI agent, integration, inline (embedded), client portal (web & mobile), Chrome extension, MS Office add-in, MS Outlook add-in, external (white-label), website, and website widget.

**App lifecycle:** draft → design → development → active → inactive, with archiving and permanent deletion.

**Customization layers:** Apps support per-tenant overrides for pages, views, menus, fields, and enums — allowing product apps to be tailored without forking.

## Custom Queries

SQL-based analytics templates with variable interpolation. Support dynamic runtime filters, pagination, and multiple database targets. Built-in variables include tenant context, user identity, and custom filter expressions.
