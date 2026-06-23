# Workflow Examples

A complete build → validate → test walkthrough, then reusable checklists. Commands assume an active session (`docyrus auth who --json`) and target the `crm` app. Append `--json` everywhere you need to capture IDs.

## Table of Contents

1. [Worked example: Accounts + Contacts](#worked-example-accounts--contacts)
2. [Validation checklist](#validation-checklist)
3. [Test playbook](#test-playbook)

## Worked example: Accounts + Contacts

Goal: a two-table mini-CRM. `Accounts` is the parent; `Contacts` belongs to one account and has a pipeline status.

### 1. Resolve the app

```bash
docyrus apps list --json        # note the appSlug ("crm") and appId
```

### 2. Create the parent data source (Accounts)

Create parents first — you need the Accounts data source ID to wire the relation from Contacts.

```bash
docyrus studio create-data-source --appSlug crm \
  --title "Accounts" --name "Account" --slug "accounts" --icon "building" --json
```

Capture the returned `data.id` as `ACCOUNTS_DS_ID`.

### 3. Add Accounts fields

`name` is built-in (the record title), so only add the extras:

```bash
docyrus studio create-fields-batch --appSlug crm --dataSourceSlug accounts --data '[
  { "name": "Website",  "slug": "website",  "type": "url" },
  { "name": "Industry", "slug": "industry", "type": "select" },
  { "name": "ARR",      "slug": "arr",      "type": "money" }
]' --json
```

### 4. Add Industry options

```bash
docyrus studio create-enums --appSlug crm --dataSourceSlug accounts --fieldSlug industry --data '[
  { "name": "Software",   "sortOrder": 1 },
  { "name": "Finance",    "sortOrder": 2 },
  { "name": "Healthcare", "sortOrder": 3 }
]' --json
```

### 5. Create the child data source (Contacts) and its fields

```bash
docyrus studio create-data-source --appSlug crm \
  --title "Contacts" --name "Contact" --slug "contacts" --icon "users" --json

# Wire the relation with the Accounts data source ID captured in step 2.
# NB: batch items use snake_case `relation_data_source_id` — camelCase here is silently dropped.
docyrus studio create-fields-batch --appSlug crm --dataSourceSlug contacts --data '[
  { "name": "Email",   "slug": "email",   "type": "email" },
  { "name": "Phone",   "slug": "phone",   "type": "phone" },
  { "name": "Status",  "slug": "status",  "type": "status" },
  { "name": "Account", "slug": "account", "type": "relation", "relation_data_source_id": "ACCOUNTS_DS_ID" }
]' --json
```

> If a relation field comes back with `relation_data_source_id: null` (e.g. a batch item used camelCase), repair it with the single-command flag:
> `docyrus studio update-field --appSlug crm --dataSourceSlug contacts --fieldId <id> --relationDataSourceId ACCOUNTS_DS_ID --json`

### 6. Add Contact status options

```bash
docyrus studio create-enums --appSlug crm --dataSourceSlug contacts --fieldSlug status --data '[
  { "name": "Lead",    "color": "#94a3b8", "sortOrder": 1 },
  { "name": "Active",  "color": "#22c55e", "sortOrder": 2 },
  { "name": "Churned", "color": "#ef4444", "sortOrder": 3, "isFinalOption": true }
]' --json
```

### 7. (Optional) surface contacts from the account side

Add a virtual reverse-lookup `list` field on Accounts that points at Contacts, then confirm it resolved with `list-fields`:

```bash
docyrus studio create-field --appSlug crm --dataSourceSlug accounts \
  --name "Contacts" --slug "contacts_list" --type "list" \
  --relationDataSourceId "CONTACTS_DS_ID" --json
```

Now validate, then test.

## Validation checklist

Run these reads and check each box. Capture enum IDs here — you need them for testing defaults and inserts.

- [ ] **Data source exists with the right shape:**
  `docyrus studio get-data-source --dataSourceId <id> --json` (resolves by ID only — get the ID from `list-data-sources`)
  → `type` is `simple`; `slug`, `title`, `name` correct; `data_access`/`unit_peer_access` as intended.
- [ ] **All fields present and typed:**
  `docyrus studio list-fields --appSlug crm --dataSourceSlug contacts --json`
  → every planned field is there; `type` matches; selection fields are `field-select`/`field-status`/…; the relation field has a non-null `relation_data_source_id` pointing at the correct parent.
- [ ] **No accidental reserved-slug fields** and no duplicate of a built-in (`name`, `description`, …).
- [ ] **Enums exist for every selection field:**
  `docyrus studio list-enums --appId <id> --dataSourceId <id> --fieldId <statusFieldId> --json`
  → options present, ordered, IDs captured.
- [ ] **Defaults are valid:** any `default_value` on a select/status/relation field is a UUID that appears in that field's enum/parent set (not a label); virtual/computed fields carry no default.
- [ ] **Engine sees the schema:**
  `docyrus dsql schema data-source crm contacts`
  → the queryable columns and relation joins you expect are listed.

If any box fails, fix with `update-field` / `create-field` / `create-enums` and re-run the checklist.

## Test playbook

Prove the schema accepts and returns data. Always delete throwaway records at the end.

1. **Insert a parent record** and capture its ID:
   ```bash
   docyrus ds create crm accounts --data '{"name":"Globex","website":"https://globex.com","industry":"<industry-enum-id>"}' --json
   # → capture data.id as ACCOUNT_ID
   ```

2. **Insert a child record** referencing the parent (relation = parent record ID) and a status (enum row ID):
   ```bash
   docyrus ds create crm contacts --data '{"name":"Jane Doe","email":"jane@globex.com","phone":"+15551234567","status":"<status-enum-id>","account":"ACCOUNT_ID"}' --json
   # → capture data.id as CONTACT_ID
   ```

3. **Read back with expansion** — confirms types resolve and the relation joins:
   ```bash
   docyrus ds list crm contacts \
     --columns "name, email, phone, status, account" --expand relation \
     --filters '{"rules":[{"field":"id","operator":"=","value":"CONTACT_ID"}]}' --json
   ```
   Expect `status` and `account` to come back as nested `{ id, name }` objects. (Enum fields like `status` auto-expand even without `--expand`.) To flatten a related field inline instead, spread it: `...account(name)` — alias to avoid the related `name` overwriting the row's own `name`.

4. **Exercise a relation filter** (filters across the relation use `rel_<relationSlug>/<fieldSlug>`):
   ```bash
   docyrus ds list crm contacts \
     --columns "name, email" \
     --filters '{"rules":[{"field":"rel_account/name","operator":"=","value":"Globex"}]}' --json
   ```

5. **Default-value check** (if you set one): create a record omitting that field and confirm the default fills in:
   ```bash
   docyrus ds create crm contacts --data '{"name":"Default Test"}' --json
   docyrus ds list crm contacts --columns "name, status" \
     --filters '{"rules":[{"field":"name","operator":"=","value":"Default Test"}]}' --json
   ```

6. **Validation boundaries** — know what is and isn't enforced:
   - **Record write on a simple data source is NOT type-checked.** `docyrus ds create crm contacts --data '{"name":"Bad","status":"Lead"}'` **succeeds** and stores `status: "Lead"` as a raw string — it just won't resolve to the enum on read. So always pass enum row IDs / record IDs; a wrong value fails silently, not loudly.
   - **Schema-level validation DOES bite.** These return HTTP 400:
     ```bash
     docyrus studio create-field --appSlug crm --dataSourceSlug contacts --name "X" --slug "x" --type select --defaultValue "Lead" --json
     # → "Invalid default value for field-select: expected a UUID ... pass the enum row ID, not the label."
     docyrus studio create-field --appSlug crm --dataSourceSlug contacts --name "Dup" --slug "name" --type text --json
     # → "Slug \"name\" is a reserved system field and cannot be used"
     ```

7. **Clean up every test record:**
   ```bash
   docyrus ds delete crm contacts CONTACT_ID
   docyrus ds delete crm accounts ACCOUNT_ID
   # …and any others created above
   ```

Report results to the user: which fields round-tripped, how relations/enums resolved, and confirmation that the schema is ready (and that test data was removed).
