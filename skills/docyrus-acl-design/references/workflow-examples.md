# ACL Design Workflow Examples

## End-to-end example: Two-role CRM setup

Scenario: a CRM app with two roles — `crm-viewer` (read-only on Contacts and Deals) and `crm-editor` (full CRUD on Contacts and Deals, can manage users in settings). Also add a role query so viewers only see records they own.

### Step 1 — Confirm auth and find the app

```bash
docyrus auth who --json
# → email: you@company.com, tenantName: Acme Corp

docyrus apps list --json | python3 -c "import sys,json; [print(r['slug'], r['id']) for r in json.load(sys.stdin)['data']]"
# → crm  019c4abc-0001-7000-a000-000000000001
```

### Step 2 — Survey available operations

```bash
docyrus acl operations list --targetType data_source --json | python3 -c \
  "import sys,json; [print(r['slug']) for r in json.load(sys.stdin)['data']]"
# → view create edit delete create_bulk delete_bulk export export_bulk print print_bulk import

docyrus acl operations list --targetType settings --json | python3 -c \
  "import sys,json; [print(r['slug']) for r in json.load(sys.stdin)['data']]"
# → manage_settings manage_users manage_teams ... (11 total)
```

### Step 3 — Create the roles

```bash
# Viewer role — read-only
docyrus acl roles create \
  --slug "crm-viewer" --name "CRM Viewer" \
  --tenantAppId 019c4abc-0001-7000-a000-000000000001 \
  --json
# → { data: { id: "019e11aa-...", slug: "crm-viewer", ... } }
VIEWER_ID="019e11aa-..."

# Editor role — full CRUD
docyrus acl roles create \
  --slug "crm-editor" --name "CRM Editor" \
  --tenantAppId 019c4abc-0001-7000-a000-000000000001 \
  --json
EDITOR_ID="019e11bb-..."
```

### Step 4 — Find data source IDs

```bash
docyrus studio list-data-sources --appSlug crm --json | python3 -c \
  "import sys,json; [print(r['slug'], r['id']) for r in json.load(sys.stdin)['data']]"
# → contacts 019c48d0-...
# → deals    019c48e0-...
CONTACTS_ID="019c48d0-..."
DEALS_ID="019c48e0-..."
```

### Step 5 — Create data_source rules

```bash
# Viewer: can only view contacts and deals
docyrus acl rules create \
  --roleId $VIEWER_ID --dataSourceId $CONTACTS_ID \
  --allowedOperations "view" --json

docyrus acl rules create \
  --roleId $VIEWER_ID --dataSourceId $DEALS_ID \
  --allowedOperations "view" --json

# Editor: full CRUD + bulk ops
docyrus acl rules create \
  --roleId $EDITOR_ID --dataSourceId $CONTACTS_ID \
  --allowedOperations "view,create,edit,delete,create_bulk,delete_bulk,export,import" --json

docyrus acl rules create \
  --roleId $EDITOR_ID --dataSourceId $DEALS_ID \
  --allowedOperations "view,create,edit,delete,create_bulk,delete_bulk,export,import" --json
```

### Step 6 — Create a settings rule for editors

```bash
# Editor can manage users and teams (no FK for settings rules)
docyrus acl rules create \
  --roleId $EDITOR_ID --targetType settings \
  --allowedOperations "manage_users,manage_teams" --json
```

### Step 7 — Add a role query (viewers only see their own contacts)

```bash
# Viewers can only read contacts where record_owner = the logged-in user
docyrus acl role-queries create \
  --roleIds "$VIEWER_ID" \
  --dataSourceId $CONTACTS_ID \
  --query '{"field":"record_owner","operator":"eq","value":"{{userId}}"}' \
  --restrictionLevel "hidden" \
  --json
# → { data: { id: "019e22cc-...", ... } }
```

### Step 8 — Validate

```bash
# Confirm roles exist
docyrus acl roles list --json | python3 -c \
  "import sys,json; [print(r['slug'], r['id']) for r in json.load(sys.stdin)['data'] if 'crm' in r['slug']]"

# Confirm rules for viewer
docyrus acl rules list --roleId $VIEWER_ID --json | python3 -c \
  "import sys,json; [print(r['target_type'], r['tenant_data_source_id'], r['allowed_operations']) for r in json.load(sys.stdin)['data']]"
# → data_source  019c48d0-...  ['view']
# → data_source  019c48e0-...  ['view']

# Confirm rules for editor
docyrus acl rules list --roleId $EDITOR_ID --json | python3 -c \
  "import sys,json; [print(r['target_type'], r['tenant_data_source_id'], r['allowed_operations']) for r in json.load(sys.stdin)['data']]"
# → data_source  019c48d0-...  ['view', 'create', 'edit', 'delete', ...]
# → data_source  019c48e0-...  ['view', 'create', 'edit', 'delete', ...]
# → settings     None          ['manage_users', 'manage_teams']

# Confirm role query
docyrus acl role-queries list --roleId $VIEWER_ID --json
# → 1 role query, restrictionLevel: "hidden", query: {field: "record_owner", ...}
```

---

## Validation checklist

After any ACL configuration, confirm:

- [ ] `docyrus acl roles list` shows expected roles with correct `slug`, `name`, `tenantAppId`, `ownership`
- [ ] `docyrus acl rules list --roleId <id>` shows one rule per (role × target) — no duplicates
- [ ] Each rule's `target_type` matches the intended target (`data_source`, `settings`, etc.)
- [ ] Each rule's FK is set correctly: `tenant_data_source_id` for data_source, `null` for settings/devtools
- [ ] `allowed_operations` array contains the slugs you passed — no extras, no truncation
- [ ] Archived rules are absent from default list (appear with `--includeArchived`)
- [ ] Role queries: correct `dataSourceId`, `roleIds` array, `restrictionLevel`, and `query` object
- [ ] Hierarchy units: correct parent/child tree (use `--parentId` to traverse)

---

## Common patterns

### Pattern: Archive a rule instead of deleting (reversible)

```bash
# Soft-disable
docyrus acl rules update --ruleId <uuid> --archived true --json

# Re-enable
docyrus acl rules update --ruleId <uuid> --archived false --json
```

### Pattern: Replace operations on an existing rule

Rules don't have a "patch operations" endpoint — `update` replaces `allowedOperations` wholesale:
```bash
docyrus acl rules update --ruleId <uuid> \
  --allowedOperations "view,create,edit,delete,export,print" --json
```

### Pattern: Clone a role's rules to a new role

```bash
# 1. List source role's rules
docyrus acl rules list --roleId $SOURCE_ROLE --json > /tmp/source-rules.json

# 2. For each rule, create on the new role with the same allowedOperations and target
# (manual loop — the CLI doesn't have a clone command)
```

### Pattern: Grant a role access to all data sources in an app

There is no "wildcard" rule — you must create one rule per data source. Loop:
```bash
docyrus studio list-data-sources --appSlug crm --json | \
  python3 -c "import sys,json; [print(r['id']) for r in json.load(sys.stdin)['data']]" | \
  while read DS_ID; do
    docyrus acl rules create \
      --roleId $ROLE_ID --dataSourceId "$DS_ID" \
      --allowedOperations "view,create,edit,delete" --json
  done
```

### Pattern: Devtools access for a developer role

```bash
docyrus acl rules create \
  --roleId $DEV_ROLE --targetType devtools \
  --allowedOperations "cody,studio,automations,custom_queries,developer_center" --json
```

### Pattern: Field-level masking

To mask a salary field from a standard user role, create a `field` rule pointing to that data source with the `masked` operation. (The field-level rule requires the data source FK.)
```bash
docyrus acl rules create \
  --roleId $ROLE_ID --targetType field --dataSourceId $DS_ID \
  --allowedOperations "masked" --json
```

---

## Error reference

| Error | Cause | Fix |
|---|---|---|
| `duplicate key value violates unique constraint` | Two rules for same (role × target FK) | Check with `rules list --roleId` before creating |
| `"Invalid data received for parameters"` (422) | Wrong key casing (e.g. `role_id` instead of `roleId`) | Use camelCase keys |
| `ACL rule not found` (404) | `ruleId` doesn't exist or was deleted | Run `rules list --includeArchived` to check |
| `Update ACL rule payload is empty` | `rules update` called with no flags/data | Pass at least one flag |
| `Hierarchy unit delete fails` | Unit has child units | Delete children first, then parent |
| `Provide --roleIds and --query` | `role-queries create` missing required fields | Both are always required on create |
