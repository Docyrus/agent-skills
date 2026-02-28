---
name: docyrus-cli-app
description: Use the Docyrus CLI (`docyrus`) to interact with the Docyrus platform from the terminal. Use when the user asks to authenticate, list apps, query or manage data source records (CRUD), send API requests, switch environments/tenants, or discover OpenAPI specs via the `docyrus` command-line tool. Triggers on tasks involving docyrus CLI commands, terminal-based Docyrus operations, `docyrus ds list`, `docyrus curl`, `docyrus auth`, or any shell-based Docyrus workflows.
---

# Docyrus CLI

Guide for using the `docyrus` CLI to interact with the Docyrus platform from the terminal.

## Command Overview

| Command | Description |
|---------|-------------|
| `docyrus auth login` | Authenticate via OAuth2 device flow |
| `docyrus auth who` | Show current user |
| `docyrus auth tenants list` | List available tenants |
| `docyrus auth tenants use` | Switch active tenant |
| `docyrus env list` / `env use` | Manage environments |
| `docyrus apps list` | List tenant apps |
| `docyrus ds get` | Get data source metadata |
| `docyrus ds list` | Query records with filters, sorting, pagination |
| `docyrus ds create` | Create a record |
| `docyrus ds update` | Update a record |
| `docyrus ds delete` | Delete a record |
| `docyrus curl` | Send arbitrary API requests |
| `docyrus discover api` | Download tenant OpenAPI spec |

**See [references/cli-manifest.md](references/cli-manifest.md) for complete command reference with all flags and arguments.**

## Common Workflows

### First-Time Setup

1. Authenticate: `docyrus auth login`
2. Select tenant: `docyrus auth tenants use --tenantId <id>`
3. Verify: `docyrus auth who`

### Discover Data Sources

1. List apps: `docyrus apps list`
2. Get metadata: `docyrus ds get <appSlug> <dataSourceSlug>`

### Query Records (`ds list`)

Basic listing:

```bash
docyrus ds list crm contacts --columns "name, email, phone" --limit 20
```

With filters (JSON object):

```bash
docyrus ds list crm contacts \
  --columns "name, email" \
  --filters '{"rules":[{"field":"status","operator":"=","value":"active"}]}'
```

With relation expansion:

```bash
docyrus ds list crm contacts \
  --columns "name, ...related_account(account_name, account_phone)"
```

Date shortcut filter:

```bash
docyrus ds list crm tasks --filters '{"rules":[{"field":"created_on","operator":"this_month"}]}'
```

**See [references/list-query-examples.md](references/list-query-examples.md) for comprehensive filter, sort, pagination, and combined query examples.**

### CRUD Operations

Create:

```bash
docyrus ds create crm contacts --data '{"name":"Jane Doe","email":"jane@example.com"}'
```

Update:

```bash
docyrus ds update crm contacts <recordId> --data '{"phone":"+1234567890"}'
```

Delete:

```bash
docyrus ds delete crm contacts <recordId>
```

### Arbitrary API Calls

```bash
docyrus curl /v1/users/me
docyrus curl /v1/dev/apps -X GET --format json
docyrus curl /v1/some/endpoint -X POST -d '{"key":"value"}'
```

## Key Rules

- Arguments use `appSlug` and `dataSourceSlug` (not IDs) — run `docyrus apps list` and `docyrus ds get` to discover slugs
- `--filters` accepts a JSON string following the filter group structure: `{"combinator":"and","rules":[...]}`
- Filter operators include: `=`, `!=`, `>`, `>=`, `<`, `<=`, `like`, `not like`, `in`, `not in`, `empty`, `not empty`, `between`, `today`, `this_month`, `this_quarter`, `last_30_days`, `active_user`
- Filter on related fields using `rel_<relation_slug>/<field_slug>` syntax
- `--columns` uses comma-separated field slugs with support for relation expansion `()`, spread `...`, aliasing `:`, and functions `@`
- `--format` controls output: `toon` (default table), `json`, `yaml`, `md`, `jsonl`
- `--verbose` wraps response in full envelope with metadata

## References

- **[CLI Manifest](references/cli-manifest.md)** — Complete command reference with all flags, arguments, and options.
- **[List Query Examples](references/list-query-examples.md)** — Practical `ds list` examples covering columns, filters, sorting, pagination, and combined queries.
