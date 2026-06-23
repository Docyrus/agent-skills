# ACL Operation Catalog

System-defined operations (`core_acl_operation`) grouped by target type. These slugs are passed to `--allowedOperations` when creating or updating rules.

Fetch live with:
```bash
docyrus acl operations list --targetType <type> --json
```

---

## `data_source` — Record-level CRUD and bulk actions

| Slug | Display name | When to grant |
|---|---|---|
| `view` | View Record | Read records at all |
| `create` | Create Records | Add new records |
| `edit` | Edit Record | Modify existing records |
| `delete` | Delete Record | Remove a single record |
| `create_bulk` | Create Bulk Records | Mass import / batch create |
| `delete_bulk` | Delete Bulk Records | Batch delete |
| `export` | Export Record | Export a single record (PDF/Excel/CSV) |
| `export_bulk` | Export Bulk Records | Bulk export |
| `print` | Print Record | Print a single record |
| `print_bulk` | Print Bulk Records | Batch print |
| `import` | Import | Import records from file |

**Typical configurations:**
- Read-only viewer: `view`
- Standard user: `view,create,edit`
- Power user: `view,create,edit,delete,export,print`
- Admin: all operations

---

## `field` — Per-field visibility overrides

| Slug | Display name | When to grant |
|---|---|---|
| `hidden` | Hidden | Completely hide this field from the role |
| `masked` | Masked | Show field exists but mask the value (e.g. `****`) |
| `readOnly` | Read-Only | Show field value but prevent editing |

> These operations are typically used in field-scoped rules to restrict specific sensitive fields (e.g. salary, SSN) from less-privileged roles.

---

## `ai` — AI feature access

| Slug | Display name | When to grant |
|---|---|---|
| `ai-access` | AI Features | Enable AI-powered features (Cody chat, AI suggestions) for the role |

---

## `settings` — Tenant administration

| Slug | Display name | When to grant |
|---|---|---|
| `manage_settings` | Manage Settings | General settings access |
| `manage_account_billing` | Manage Account & Billing | Billing and subscription management |
| `manage_regional_settings` | Manage Regional Settings | Date, locale, timezone settings |
| `manage_users` | Manage Users | Invite, deactivate, and manage tenant users |
| `manage_teams` | Manage Teams | Create and manage teams |
| `manage_organization_hierarchy` | Manage Organization Hierarchy | Create and manage hierarchy units (org chart) |
| `manage_roles_and_permissions` | Manage Roles And Permissions | Create roles and assign permissions |
| `manage_integrations` | Manage Integrations | Configure third-party integrations |
| `view_audit_ogs` | View Audit Logs | Read audit trail |
| `view_usage_analytics` | View Usage Analytics | Read tenant usage stats |
| `manage_branding` | Manage Branding | Customize tenant branding |

**Typical configurations:**
- HR admin: `manage_users, manage_teams, manage_organization_hierarchy`
- IT admin: `manage_settings, manage_regional_settings, manage_integrations`
- Super admin: all settings operations

> `settings` rules have **no FK target** (no `dataSourceId`/`appId`/`aiToolId`) — they apply tenant-wide.

---

## `devtools` — Developer tool access

| Slug | Display name | When to grant |
|---|---|---|
| `cody` | Cody | Access to Cody coding agent |
| `studio` | Studio | Access to Studio (no-code schema designer) |
| `data_sources` | Data Sources | Manage data sources in dev tools |
| `email_print_templates` | Email/Print Templates | Edit HTML/email export templates |
| `automations` | Automations | Build and manage automations |
| `custom_queries` | Custom Queries | Create and manage custom queries |
| `developer_center` | Developer Center | Access to the developer center panel |

> `devtools` rules have **no FK target** — they apply to the dev tooling globally.

---

## `app` — App-level access

Currently no operations defined for the `app` target type. Reserved for future app-gating rules.

---

## `ai_tool` — AI tool access

| Slug | Display name | When to grant |
|---|---|---|
| `allow` | Allow | Allow the role to use this specific AI tool |

> `ai_tool` rules require `--aiToolId <uuid>` — one rule per role per tool.

---

## Usage notes

- **Slugs are case-sensitive.** `readOnly` (camelCase) for field, `ai-access` (hyphenated) for ai — use exactly as shown.
- **Run `docyrus acl operations list --json` to get live UUIDs** if your code needs the operation `id` (rare — most integrations only use slugs).
- **`app` has no operations yet.** Passing `--targetType app` with any `allowedOperations` will result in an empty grant until platform defines app-level ops.
