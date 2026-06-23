---
name: docyrus-acl-design
description: Design, configure, and validate a Docyrus tenant ACL (Access Control Layer) end-to-end using the `docyrus acl` CLI commands. Use when the user wants to define permission roles, assign operation-level permissions to roles for data sources / apps / AI tools / settings / devtools, set up row-level record filters (role queries), build an organizational hierarchy (hierarchy units), or look up available system operations. Triggers on "create a role", "set permissions for a role", "grant access to a data source", "role query", "row-level filter", "hierarchy unit", "org-chart access", "acl rules", "what operations", `docyrus acl roles`, `docyrus acl rules`, `docyrus acl role-queries`, `docyrus acl hierarchy-units`, `docyrus acl operations`, or any ACL role/permission design + configuration task in Docyrus.
---

# Docyrus ACL Design

Configure tenant access control with `docyrus acl`, then **validate** the setup before handing it off. The Docyrus ACL system has five concepts:

| Concept | What it is |
|---|---|
| **Role** | Named permission group assigned to users |
| **Hierarchy Unit** | Org-tree node for org-chart-aware row-level security (RLS) |
| **Role Query** | Row-level filter attached to one or more roles for a data source |
| **Operation** | System-defined permission slug (read-only — you can't create these) |
| **Rule** | Binds a role to a target (data source / app / AI tool / settings) with a list of allowed operation slugs |

For platform concepts see the **docyrus-platform** skill. For CLI flag tables see the **docyrus-cli-app** skill. This skill is the ACL-specific workflow.

## Workflow

Follow in order. A role without rules grants nothing useful.

1. **Confirm auth.**
   ```bash
   docyrus auth who --json          # confirm session + tenant
   ```
   No session → stop and ask the user to run `docyrus auth login`.

2. **Survey available operations.** Before creating rules, know which operations exist per target type:
   ```bash
   docyrus acl operations list --targetType data_source --json
   docyrus acl operations list --targetType settings --json
   docyrus acl operations list --targetType devtools --json
   ```
   The full catalog is in [references/operation-catalog.md](references/operation-catalog.md).

3. **Create roles.** See [Roles](#roles). One role per logical access level (e.g. "Viewer", "Editor", "Admin"). If the role is app-scoped pass `--tenantAppId`.

4. **Create hierarchy units** (skip if org-chart RLS is not needed). See [Hierarchy Units](#hierarchy-units). Build parent units before children.

5. **Create role queries** (skip if row-level record filtering is not needed). See [Role Queries](#role-queries). Each role query narrows which records a role can see in a specific data source.

6. **Create rules.** Each rule grants a role a set of operations on one specific target. See [Rules](#rules). Start with `data_source` rules, then `settings`, `devtools`, `app`, `ai_tool` as needed.

7. **Validate.** Re-read each object to confirm it landed with the right values. See [Validate](#validate).

A full worked example (two-role CRM setup with data-source rules, a settings rule, and a role query) is in [references/workflow-examples.md](references/workflow-examples.md).

## Roles

```bash
# List roles (optionally filter by name/slug)
docyrus acl roles list --json
docyrus acl roles list --search "editor" --json

# Get one role
docyrus acl roles get --roleId <uuid> --json

# Create a role
docyrus acl roles create --slug "crm-viewer" --name "CRM Viewer" --json
docyrus acl roles create --slug "crm-editor" --name "CRM Editor" \
  --tenantAppId <app-uuid> --ownership CUSTOM --json

# Update a role (PATCH — any subset of fields)
docyrus acl roles update --roleId <uuid> --name "CRM Viewer (Read-Only)" --json

# Delete a role and all its assignments
docyrus acl roles delete --roleId <uuid> --json
```

Key fields:
- `--slug` and `--name` are **required** on create.
- `--ownership` ∈ `SYSTEM | CUSTOM | APP | USER | PRODUCT` (default `CUSTOM`).
- `--tenantAppId` scopes the role to a specific app.
- `--disableLogin` (int 0/1) prevents login when the user only holds this role.
- `--activitySummaryReportQueryId` points to a custom query UUID for the role's activity summary dashboard.
- `--data '<json>'` / `--from-file ./payload.json` override individual flags.

## Hierarchy Units

Hierarchy units form an org-chart tree. Users are then assigned to units, and data-source `data_access`/`unit_peer_access` rules use that tree for RLS.

```bash
# List units (optionally filter by parent or name)
docyrus acl hierarchy-units list --json
docyrus acl hierarchy-units list --parentId <uuid> --json
docyrus acl hierarchy-units list --search "sales" --json

# Get one unit
docyrus acl hierarchy-units get --unitId <uuid> --json

# Create (build parents before children)
docyrus acl hierarchy-units create --name "Sales" --json
docyrus acl hierarchy-units create --name "Sales East" --parentId <sales-unit-uuid> --json

# Update
docyrus acl hierarchy-units update --unitId <uuid> --name "Sales West" --json

# Delete (fails if the unit has children — delete children first)
docyrus acl hierarchy-units delete --unitId <uuid> --json
```

- `--name` is **required** on create.
- `--parentId` is optional; omit for a root unit.

## Role Queries

A role query is a DSQL-style filter object attached to one or more roles for a specific data source. Users whose active role matches are restricted to only the rows that match the query.

```bash
# List role queries
docyrus acl role-queries list --json
docyrus acl role-queries list --dataSourceId <uuid> --json
docyrus acl role-queries list --roleId <uuid> --json
docyrus acl role-queries list --restrictionLevel hidden --json

# Get one
docyrus acl role-queries get --roleQueryId <uuid> --json

# Create
docyrus acl role-queries create \
  --roleIds "<role-uuid-1>,<role-uuid-2>" \
  --dataSourceId <ds-uuid> \
  --query '{"field":"status","operator":"eq","value":"active"}' \
  --restrictionLevel "read-only" \
  --json

# Update
docyrus acl role-queries update --roleQueryId <uuid> \
  --query '{"field":"status","operator":"in","value":["active","pending"]}' --json

# Delete
docyrus acl role-queries delete --roleQueryId <uuid> --json
```

Key fields:
- `--roleIds` comma-separated UUIDs — **required** on create.
- `--query` JSON string of the filter object — **required** on create.
- `--dataSourceId` scopes the query to one data source (optional — omit for a global filter).
- `--restrictionLevel` ∈ `hidden | read-only | not-deletable`.
- `--filterChildRelations` boolean — also apply the filter to child-relation queries.

## Operations

Operations are system-defined and **read-only** — you can list them but cannot create, modify, or delete them.

```bash
# List all operations
docyrus acl operations list --json

# Filter by target type
docyrus acl operations list --targetType data_source --json
docyrus acl operations list --targetType settings --json
docyrus acl operations list --targetType devtools --json
docyrus acl operations list --targetType ai --json
docyrus acl operations list --targetType ai_tool --json
docyrus acl operations list --targetType field --json

# Search by name or slug
docyrus acl operations list --search "delete" --json
docyrus acl operations list --search "manage" --targetType settings --json
```

`--targetType` values: `data_source`, `field`, `ai`, `settings`, `devtools`, `app`, `ai_tool`.
The full operation catalog (slugs + names per target type) is in [references/operation-catalog.md](references/operation-catalog.md).

## Rules

A rule grants a role a set of operations on exactly one target. One rule = one (role × target) binding.

```bash
# List rules
docyrus acl rules list --json
docyrus acl rules list --roleId <uuid> --json
docyrus acl rules list --targetType settings --json
docyrus acl rules list --dataSourceId <uuid> --json
docyrus acl rules list --appId <uuid> --json
docyrus acl rules list --aiToolId <uuid> --json
docyrus acl rules list --includeArchived --json     # default: archived rules are hidden

# Get one rule
docyrus acl rules get --ruleId <uuid> --json

# Create a rule
# data_source target (default)
docyrus acl rules create \
  --roleId <role-uuid> --dataSourceId <ds-uuid> \
  --allowedOperations "view,create,edit,delete" --json

# settings target (no FK — omit dataSourceId/appId/aiToolId)
docyrus acl rules create \
  --roleId <role-uuid> --targetType settings \
  --allowedOperations "manage_users,manage_teams" --json

# devtools target
docyrus acl rules create \
  --roleId <role-uuid> --targetType devtools \
  --allowedOperations "cody,studio" --json

# ai_tool target
docyrus acl rules create \
  --roleId <role-uuid> --targetType ai_tool --aiToolId <tool-uuid> \
  --allowedOperations "allow" --json

# Update a rule (PATCH — supply only what changes)
docyrus acl rules update --ruleId <uuid> \
  --allowedOperations "view,create,edit,delete,export" --json

# Archive a rule (soft disable — still visible with --includeArchived)
docyrus acl rules update --ruleId <uuid> --archived true --json

# Restore an archived rule
docyrus acl rules update --ruleId <uuid> --archived false --json

# Hard-delete a rule
docyrus acl rules delete --ruleId <uuid> --json
```

Key fields:
- `--roleId` is **required** on create.
- `--targetType` defaults to `data_source`; valid values: `data_source`, `field`, `ai`, `settings`, `devtools`, `app`, `ai_tool`.
- `--allowedOperations` comma-separated list of operation slugs; must match real slugs for the target type.
- FK fields are mutually exclusive per rule row: set `--dataSourceId` for `data_source` targets, `--appId` for `app` targets, `--aiToolId` for `ai_tool` targets. Leave all three null for `settings`, `devtools`, `ai`, `field` targets.

## Critical rules

- **All payload keys are camelCase.** `--roleId`, `--dataSourceId`, `--allowedOperations`, `--roleIds`, `--targetType` — never snake_case. The backend ACL DTOs are camelCase and `GlobalValidationPipe` does NOT transform snake_case → camelCase.
- **One rule per (role × target FK).** A unique database constraint prevents two rules for the same `(tenant, role, dataSource)`, `(tenant, role, app)`, or `(tenant, role, aiTool)`. Attempting to create a duplicate returns a 500 constraint error. Check with `rules list --roleId <id>` first.
- **`targetType` defaults to `data_source`** on `rules create` — always pass it explicitly when creating non-data-source rules to avoid accidentally creating a `data_source` rule with null FK.
- **`--allowedOperations` is comma-separated** on the CLI — it becomes a `string[]` in the request. Spaces around commas are stripped.
- **`--roleIds` is comma-separated** for role queries (same pattern).
- **`--query` must be a JSON string** for role queries — wrap in single quotes: `--query '{"field":"owner","operator":"eq","value":"{{userId}}"}'`.
- **Operations are read-only.** You cannot create or modify `core_acl_operation` rows — only a superadmin can. Use `acl operations list` to discover slugs; pass them into `--allowedOperations`.
- **Hierarchy unit delete rejects units with children.** Delete leaves first, then parents.
- **Archive ≠ delete.** `rules update --archived true` soft-disables a rule (the row stays and is hidden from normal list results). `rules delete` hard-removes it. Prefer archive when you may want to restore later.
- **`--targetType` uses underscores** (`data_source`, `ai_tool`), not hyphens.
- **`--restrictionLevel` for role queries uses hyphens** (`read-only`, `not-deletable`), not underscores.
- **Validate then test.** After configuring roles + rules, log in as a user with that role and confirm the access behaves as expected.

## Validate

After authoring, confirm each object landed correctly:

1. **Roles:**
   ```bash
   docyrus acl roles list --json            # count matches
   docyrus acl roles get --roleId <id> --json   # slug, name, ownership, tenantAppId
   ```

2. **Hierarchy units:**
   ```bash
   docyrus acl hierarchy-units list --json          # tree structure
   docyrus acl hierarchy-units list --parentId <id> --json  # children of a node
   ```

3. **Role queries:**
   ```bash
   docyrus acl role-queries list --roleId <id> --json   # confirm query + restrictionLevel
   ```

4. **Rules:**
   ```bash
   docyrus acl rules list --roleId <id> --json       # all rules for the role
   docyrus acl rules get --ruleId <id> --json         # confirm allowedOperations, targetType, FK
   ```
   Confirm `allowed_operations` matches the slugs you passed. Check `target_type` is correct and exactly one of `tenant_data_source_id`, `tenant_app_id`, `tenant_ai_tool_id` is set (rest null) for FK-targeted types.

## References

- **[references/operation-catalog.md](references/operation-catalog.md)** — All operation slugs grouped by target type, with display names and intended use.
- **[references/workflow-examples.md](references/workflow-examples.md)** — End-to-end worked example: create two roles with data-source and settings rules, add a role query, validate, and test.
