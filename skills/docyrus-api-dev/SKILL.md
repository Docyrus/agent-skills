---
name: docyrus-api-dev
description: Develop applications using the Docyrus API with @docyrus/api-client and @docyrus/signin libraries. Use when building apps that authenticate with Docyrus OAuth2 (PKCE, iframe, client credentials, device code), make REST API calls to Docyrus data source endpoints, construct query payloads with filters, aggregations, formulas, pivots, and child queries, integrate with external connectors (discover connectors, send requests through provider auth, run actions), manage dev-app schema and saved views via the studio endpoints (data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX export templates, email templates), wire up tenant automations (CRUD plus typed triggers and action nodes), or send transactional email through tenant messaging accounts. Triggers on tasks involving Docyrus API integration, @docyrus/api-client usage, @docyrus/signin authentication, data source query building, Docyrus REST endpoint consumption, connector discovery, external provider requests, automation CRUD, automation triggers (record-created/modified/deleted, recurrence, app-event, webhook, emailhook, webform, button-activation, manual-activation) and action nodes (external-action, send-email, send-notification, create-record, update-records, request-approval, request-input, http-request, data-source-query, custom-query, generate-document, ai-prompt, ai-agent, execute-script), tenant email account discovery, or sending email via `/v1/messaging/email`.
---

# Docyrus API Developer

Integrate with the Docyrus API using `@docyrus/api-client` (REST client) and `@docyrus/signin` (React auth provider). Authenticate via OAuth2 PKCE, query data sources with powerful filtering/aggregation, and consume REST endpoints.

## Authentication Quick Start

### React Apps — Use @docyrus/signin

```tsx
import { DocyrusAuthProvider, useDocyrusAuth, useDocyrusClient, SignInButton } from '@docyrus/signin'

// 1. Wrap root
<DocyrusAuthProvider
  apiUrl={import.meta.env.VITE_API_BASE_URL}
  clientId={import.meta.env.VITE_OAUTH2_CLIENT_ID}
  redirectUri={import.meta.env.VITE_OAUTH2_REDIRECT_URI}
  scopes={['offline_access', 'Read.All', 'DS.ReadWrite.All', 'Users.Read']}
  callbackPath="/auth/callback"
>
  <App />
</DocyrusAuthProvider>

// 2. Use hooks
function App() {
  const { status, signOut } = useDocyrusAuth()
  const client = useDocyrusClient()  // RestApiClient | null

  if (status === 'loading') return <Spinner />
  if (status === 'unauthenticated') return <SignInButton />

  // client is ready — make API calls
  const user = await client!.get('/v1/users/me')
}
```

### Non-React / Server — Use OAuth2Client Directly

```typescript
import { RestApiClient, OAuth2Client, OAuth2TokenManagerAdapter, BrowserOAuth2TokenStorage } from '@docyrus/api-client'

const tokenStorage = new BrowserOAuth2TokenStorage(localStorage)
const oauth2 = new OAuth2Client({
  baseURL: 'https://api.docyrus.com',
  clientId: 'your-client-id',
  redirectUri: 'http://localhost:3000/callback',
  usePKCE: true,
  tokenStorage,
})

// Auth Code flow
const { url } = await oauth2.getAuthorizationUrl({ scope: 'openid offline_access Users.Read' })
window.location.href = url
// After redirect:
const tokens = await oauth2.handleCallback(window.location.href)

// Create API client with auto-refresh
const client = new RestApiClient({
  baseURL: 'https://api.docyrus.com',
  tokenManager: new OAuth2TokenManagerAdapter(tokenStorage, async () => {
    return (await oauth2.refreshAccessToken()).accessToken
  }),
})
```

## API Endpoints

### Data Source Items (Dynamic per tenant)
```
GET    /v1/apps/{appSlug}/data-sources/{slug}/items          — List with query payload
GET    /v1/apps/{appSlug}/data-sources/{slug}/items/{id}     — Get one
POST   /v1/apps/{appSlug}/data-sources/{slug}/items          — Create
PATCH  /v1/apps/{appSlug}/data-sources/{slug}/items/{id}     — Update
DELETE /v1/apps/{appSlug}/data-sources/{slug}/items/{id}     — Delete one
DELETE /v1/apps/{appSlug}/data-sources/{slug}/items          — Delete many (body: { recordIds })
```

Endpoints exist only if the data source is defined in the tenant. Check the tenant's OpenAPI spec at `GET /v1/api/openapi.json`.

### System Endpoints (Always Available)
```
GET    /v1/users          — List users
POST   /v1/users          — Create user
GET    /v1/users/me       — Current user profile
PATCH  /v1/users/me       — Update current user
```

### Connector Discovery & External Request Endpoints
```
GET    /v1/connectors?q=&limit=&offset=                                    — List connectors with keyword search
GET    /v1/connectors/{dataProviderSlug}                                    — Get connector detail (dataSources + actions)
GET    /v1/connectors/{dataProviderSlug}/actions/{actionKey}                — Get action detail (input/output schemas, API endpoint)
GET    /v1/connectors/{dataProviderSlug}/connections                        — Get tenant connections + user connection status
PUT    /v1/connectors/{dataProviderSlug}                                    — Send HTTP request through connector provider auth
```

Scopes: `Read.All`, `ReadWrite.All`, or `Connectors.Read.All`. The `PUT` endpoint requires `ReadWrite.All`.

**PUT request body** for sending requests through a connector:
```json
{
  "endpoint": "relative/path/or/absolute-url",
  "requestMethod": "GET",
  "data": { "fields": "id,name", "limit": 20 },
  "contentType": "application/json",
  "headers": { "Authorization": "Bearer <override-token>" },
  "connectionId": "optional-tenant-connection-uuid",
  "connectionAccountId": "optional-connection-account-uuid"
}
```

The connector resolves auth credentials (OAuth tokens, base URL) from the provider configuration and stored connections. Custom `headers.Authorization` overrides the stored token.

### Action Run Endpoints
```
GET    /v1/apps/base/actions                                               — List base actions
GET    /v1/apps/{appSlug}/actions/{actionSlug}                             — Get action metadata
POST   /v1/apps/{appSlug}/actions/{actionSlug}/run                         — Run action directly
```

Action run accepts arbitrary JSON body as input. Optional headers: `x-connection-id`, `x-connection-account-id`.

### Studio (Dev) Schema Endpoints

The studio surface manages dev-app schema objects. Most routes are gated by the `Architect.Read.All` / `Architect.ReadWrite.All` scopes, and the app is identified by its tenant `app_id` UUID.

```
# Apps (mutations only — list uses /v1/apps)
DELETE /v1/dev/apps/{appId}                                                 — Archive app
POST   /v1/dev/apps/{appId}/restore                                         — Restore archived app
DELETE /v1/dev/apps/{appId}/permanent                                       — Permanently delete app

# Data sources
GET    /v1/dev/apps/{appId}/data-sources                                    — List data sources (?expand=fields,...)
GET    /v1/dev/apps/{appId}/data-sources/{dataSourceId}                     — Get data source
POST   /v1/dev/apps/{appId}/data-sources                                    — Create data source
PATCH  /v1/dev/apps/{appId}/data-sources/{dataSourceId}                     — Update data source
DELETE /v1/dev/apps/{appId}/data-sources/{dataSourceId}                     — Archive data source
POST   /v1/dev/apps/{appId}/data-sources/{dataSourceId}/restore             — Restore archived data source
DELETE /v1/dev/apps/{appId}/data-sources/{dataSourceId}/permanent           — Permanently delete data source
POST   /v1/dev/apps/{appId}/data-sources/bulk                               — Bulk create (body: { dataSources })

# Fields
GET    /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields                       — List fields
GET    /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}             — Get field
POST   /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields                       — Create field
PATCH  /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}             — Update field
DELETE /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}             — Delete field
POST   /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/batch                 — Bulk create (body: { fields })
PATCH  /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/batch                 — Bulk update (body: { fields[].fieldId })
DELETE /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/batch                 — Bulk delete (body: { fieldIds })

# Field enums
GET    /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}/enums       — List enum options
POST   /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}/enums       — Create enums (body: { enums })
PATCH  /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}/enums       — Update enums (body: { enums[].enumId })
DELETE /v1/dev/apps/{appId}/data-sources/{dataSourceId}/fields/{fieldId}/enums       — Delete enums (body: { enumIds })

# Data views (saved views) — slug-scoped
GET    /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/views                        — List views
GET    /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/views/{viewId}               — Get view
POST   /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/views                        — Create view
PUT    /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/views/{viewId}               — Update view
DELETE /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/views/{viewId}               — Delete view

# Forms (record-entry layouts) — slug-scoped
GET    /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/forms                        — List forms
GET    /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/forms/{formId}               — Get form
POST   /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/forms                        — Create form
PUT    /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/forms/{formId}               — Update form
DELETE /v1/apps/{appSlug}/data-sources/{dataSourceSlug}/forms/{formId}               — Delete form

# Webforms (public-facing forms)
GET    /v1/dev/webforms                                                              — List webforms (?dataSourceId)
GET    /v1/dev/webforms/{webformId}                                                  — Get webform
POST   /v1/dev/webforms                                                              — Create webform
PATCH  /v1/dev/webforms/{webformId}                                                  — Update webform
DELETE /v1/dev/webforms/{webformId}                                                  — Delete webform

# HTML / PDF / DOCX export templates
GET    /v1/dev/html-templates                                                        — List (?dataSourceId,&isDefault,&limit,&offset)
GET    /v1/dev/html-templates/{templateId}                                           — Get template
POST   /v1/dev/html-templates                                                        — Create template
PUT    /v1/dev/html-templates/{templateId}                                           — Update template
DELETE /v1/dev/html-templates/{templateId}                                           — Delete template

# Email templates
GET    /v1/dev/email-templates                                                       — List (?dataSourceId,&limit,&offset)
GET    /v1/dev/email-templates/{templateId}                                          — Get template
POST   /v1/dev/email-templates                                                       — Create template
PUT    /v1/dev/email-templates/{templateId}                                          — Update template
DELETE /v1/dev/email-templates/{templateId}                                          — Delete template
```

Notes:
- The bulk update DTOs do not mirror the list/get response shapes. Send `fields[].fieldId` and `enums[].enumId` (not `id`).
- A webform created without `dataSourceId` posts submissions into the tenant-schema `webform_record` table instead of a data source.
- Archived data sources cannot be reliably resolved by slug; use the ID for `restore` and `permanent` routes.

### Automation Endpoints

Tenant-app automation CRUD plus typed trigger and action node mutations. Gated by `Architect.Read.All` / `Architect.ReadWrite.All`.

```
# Automations
GET    /v1/dev/apps/{appId}/automations                                              — List automations
GET    /v1/dev/apps/{appId}/automations/{id}                                         — Get automation (includes triggers)
POST   /v1/dev/apps/{appId}/automations                                              — Create automation + first trigger
PATCH  /v1/dev/apps/{appId}/automations/{id}                                         — Update automation (name, status, source_data_source_id)
DELETE /v1/dev/apps/{appId}/automations/{id}                                         — Delete automation (204)

# Triggers — typed create/update, type-independent delete
POST   /v1/dev/apps/{appId}/automations/{automationId}/triggers/{type}               — Create trigger
PATCH  /v1/dev/apps/{appId}/automations/{automationId}/triggers/{type}/{triggerId}   — Update trigger
DELETE /v1/dev/apps/{appId}/automations/{automationId}/triggers/{triggerId}          — Delete trigger (204)

# Action nodes — typed create/update, type-independent delete
GET    /v1/dev/apps/{appId}/automations/{automationId}/nodes                         — List nodes
GET    /v1/dev/apps/{appId}/automations/{automationId}/nodes/{nodeId}                — Get node
POST   /v1/dev/apps/{appId}/automations/{automationId}/nodes/{type}                  — Create node
PATCH  /v1/dev/apps/{appId}/automations/{automationId}/nodes/{type}/{nodeId}         — Update node
DELETE /v1/dev/apps/{appId}/automations/{automationId}/nodes/{nodeId}                — Delete node (204)
```

Trigger `{type}` values (kebab-case URL segments): `record-created`, `record-modified`, `record-deleted`, `recurrence`, `app-event`, `webhook`, `emailhook`, `webform`, `button-activation`, `manual-activation`.

Action node `{type}` values: `external-action`, `send-email`, `send-notification`, `create-record`, `update-records`, `request-approval`, `request-input`, `http-request`, `data-source-query`, `custom-query`, `generate-document`, `ai-prompt`, `ai-agent`, `execute-script`.

`POST /v1/dev/apps/{appId}/automations` accepts `trigger_type` in camelCase (e.g. `recordCreated`, `recordModified`, `recordDeleted`, `recurrence`, `appEvent`, `webhook`, `emailhook`, `webform`, `buttonActivation`, `manualActivation`) on `CreateAutomationDto`. The typed trigger CRUD endpoints use the kebab-case form in the URL.

Request bodies use `snake_case` keys (e.g. `source_data_source_id`, `max_run_per_record`, `modified_columns`, `recurrence_frequency`, `core_data_provider_id`, `webhook_id`, `tenant_webform_id`, `action_type_id`, `field_mapping`, `dynamic_field_mapping`, `condition`, `input_template`, `input_transformer`, `custom_headers`, `pre_action_request`, `post_action_request`, `target_data_source_condition`).

Important: creating a node with `type=external-action` requires `action_type_id` (maps to `core_action.id`). The backend validates the supplied `data` against `core_action.input_json_schema` and inserts the linked `tenant_action` row in the same transaction.

### Action / Approval RPC (Production)

These are separate from the dev-app automation CRUD above. They drive the runtime engine.

```
PUT  /v1/automation/processAction                                                    — Execute action payload (IActionPayload)
POST /v1/automation/exchange-rates    (alias /v1/automation/syncExchangeRates)       — Fetch/save FX rates (admin or api)
PUT  /v1/automation/sendApprovalRequests                                             — { approvalStatusFieldId, recordId }
PUT  /v1/automation/sendApprovalResponse                                             — Approve response
PUT  /v1/automation/sendApprovalRevisionRequest                                      — Approval revision request
PUT  /v1/automation/sendPushNotification/{notificationId}                            — Push notification (admin or api)
```

### Messaging Endpoints

Tenant email accounts and transactional send. All routes require the `Messaging.Email.Send` OAuth2 scope.

```
GET  /v1/messaging/email/accounts                                                    — List active tenant email accounts (no credentials)
POST /v1/messaging/email/accounts/{accountId}/send                                   — Send email through an account
```

`POST /v1/messaging/email/accounts/{accountId}/send` body (`SendEmailDto`):
```json
{
  "to": ["user@example.com"],
  "cc": ["manager@example.com"],
  "bcc": ["audit@example.com"],
  "replyTo": ["support@example.com"],
  "subject": "Daily summary",
  "body": "<p>Hello</p>",
  "sendAsUser": false,
  "attachments": [
    { "filePath": "records/abc/attachments/foo.pdf", "fileName": "foo.pdf", "mimeType": "application/pdf" }
  ]
}
```

Limits: `to`/`cc`/`bcc`/`replyTo` accept up to 50 RFC-5322 addresses each, subject is capped at 998 characters, body at 1 000 000 characters, attachments at 10 items, and `filePath` at 2048 characters. `sendAsUser` only takes effect when the account allows it (see `allowOverrideName` / `allowOverrideEmail` from the accounts list).

`EmailAccountDto` (returned by list) exposes: `id`, `name`, `provider`, `senderEmail`, `senderName`, `isUserAccessible`, `allowOverrideName`, `allowOverrideEmail`, `createdOn`. Credentials, tokens, and provider secrets are never returned.

`SendEmailResponseDto`: `{ messageId, provider, accepted, rejected }`.

### ACL / Role Management Endpoints
```
GET    /v1/users/acl?dataSourceId={uuid}&recordId={uuid}   — Read record ACL rows
POST   /v1/users/acl/share                                 — Upsert record shares
DELETE /v1/users/acl/share                                 — Revoke record shares
PUT    /v1/users/acl/owner                                 — Transfer record ownership

GET    /v1/users/acl/roles                                 — List roles
GET    /v1/users/acl/roles/{roleId}                        — Get one role
POST   /v1/users/acl/roles                                 — Create role
PATCH  /v1/users/acl/roles/{roleId}                        — Update role
DELETE /v1/users/acl/roles/{roleId}                        — Delete role

GET    /v1/users/acl/user-roles                            — List user-role assignments
GET    /v1/users/acl/users/{userId}/roles                  — List one user's roles
POST   /v1/users/acl/users/{userId}/roles                  — Add roles to a user
PUT    /v1/users/acl/users/{userId}/roles                  — Replace a user's full role set
DELETE /v1/users/acl/users/{userId}/roles/{roleId}         — Remove one role assignment

GET    /v1/users/acl/role-queries                          — List role queries
GET    /v1/users/acl/role-queries/{roleQueryId}            — Get one role query
POST   /v1/users/acl/role-queries                          — Create role query
PATCH  /v1/users/acl/role-queries/{roleQueryId}            — Update role query
DELETE /v1/users/acl/role-queries/{roleQueryId}            — Delete role query
```

ACL routes require the normal authenticated API session, but they may not appear in generated Swagger/OpenAPI output because the backend currently excludes them from public docs. Integrate them with direct `RestApiClient` calls when you need record sharing, role CRUD, user-role assignment management, or role-query management.

For all ACL role operations, prefer using role `uid` values returned by the API. Nested role objects expose both `id` and `uid`, and both map to the role UID value.

### Making API Calls

```typescript
// List items with query payload
const items = await client.get('/v1/apps/base/data-sources/project/items', {
  columns: 'name, status, record_owner(firstname,lastname)',
  filters: { rules: [{ field: 'status', operator: '!=', value: 'archived' }] },
  orderBy: 'created_on DESC',
  limit: 50,
})

// Get single item
const item = await client.get('/v1/apps/base/data-sources/project/items/uuid-here', {
  columns: 'name, description, status',
})

// Create
const newItem = await client.post('/v1/apps/base/data-sources/project/items', {
  name: 'New Project',
  status: 'status-enum-id',
})

// Update
await client.patch('/v1/apps/base/data-sources/project/items/uuid-here', {
  name: 'Updated Name',
})

// Delete
await client.delete('/v1/apps/base/data-sources/project/items/uuid-here')
```

## Query Payload Summary

The GET items endpoint accepts a powerful query payload:

| Feature | Purpose |
|---------|---------|
| `columns` | Select fields, expand relations `field(subfields)`, alias `alias:field`, spread `...field()` |
| `filters` | Nested AND/OR groups with 50+ operators (comparison, date shortcuts, user-related) |
| `filterKeyword` | Full-text search across all searchable fields |
| `orderBy` | Sort by fields with direction, including related fields |
| `limit`/`offset` | Pagination (default limit: 100) |
| `fullCount` | Return total matching count alongside results |
| `calculations` | Aggregations: count, sum, avg, min, max with grouping |
| `formulas` | Computed virtual columns (simple functions, block AST, correlated subqueries) |
| `childQueries` | Fetch related child records as nested JSON arrays |
| `pivot` | Cross-tab matrix queries with date range series |
| `expand` | Return full objects for relation/user/enum fields instead of IDs |

**For full query and formula references, read**:
- `references/data-source-query-guide.md`
- `references/formula-design-guide-llm.md`

## Critical Rules

1. **Always send `columns`** in list/get calls. Without it, only `id` is returned.
2. **Data source endpoints are dynamic** — they exist only for data sources defined in the tenant.
3. **Use `id` field** for `count` calculations. Use the actual field slug for `sum`, `avg`, `min`, `max`.
4. **Child query keys must appear in `columns`** — if childQuery key is `orders`, include `orders` in columns.
5. **Formula keys must appear in `columns`** — if formula key is `total`, include `total` in columns.
6. **Filter by related field** using `rel_{{relation_field}}/{{field}}` syntax.
7. **ACL routes may be hidden from generated OpenAPI** — call them directly via `RestApiClient` instead of expecting generated collection support.
8. **Prefer role `uid` values** for ACL role writes, user-role `roleIds`, and role-query `roleIds`.
9. **Treat `PUT /v1/users/acl/users/:userId/roles` as full replacement** and `POST /v1/users/acl/users/:userId/roles` as additive.
10. **Send role-query `query` as raw JSON** and let backend derive `tenantAppId` from `dataSourceId` when applicable.
11. **After deleting a role, refresh dependent ACL state** — role lists, user-role lists, role-query lists, and any UI showing primary-role labels.
12. **Studio bulk update DTOs use scoped IDs** — send `fields[].fieldId` and `enums[].enumId` (not `id`) for the `PATCH .../fields/batch` and `PATCH .../enums` routes; the list/get response shapes do not match the bulk update DTOs.
13. **Automation request bodies use `snake_case`** keys (e.g. `source_data_source_id`, `field_mapping`). Trigger and node create/update URLs are typed (`/triggers/<type>`, `/nodes/<type>`), but delete URLs are type-independent. `POST /automations` accepts `trigger_type` in camelCase (`recordCreated`, etc.) while typed trigger routes use kebab-case (`record-created`, etc.).
14. **`external-action` automation nodes require `action_type_id`** — the backend validates the supplied `data` against `core_action.input_json_schema` and creates the matching `tenant_action` row in the same transaction.
15. **Messaging endpoints require the `Messaging.Email.Send` scope** and never return credentials. `sendAsUser` only takes effect when the listed account allows the override.

## References

Read these files when you need detailed information:

- **`references/api-client.md`** — Full RestApiClient API, OAuth2Client (all flows: PKCE, client credentials, device code), token managers, interceptors, error classes, SSE/streaming, file upload/download, HTML to PDF, retry logic
- **`references/authentication.md`** — @docyrus/signin React provider, useDocyrusAuth/useDocyrusClient hooks, hasRole/hasPermission authorization helpers, SignInButton, standalone vs iframe auth modes, env vars, API client access pattern
- **`references/data-source-query-guide.md`** — Up-to-date query payload guide: columns, filters, orderBy, pagination, calculations, formulas, child queries, pivots, and operator reference
- **`references/formula-design-guide-llm.md`** — Up-to-date formula design guide for building and validating `formulas` payloads
- **`references/acl-endpoints-frontend.md`** — Hidden ACL endpoint reference covering record sharing, roles, user-role assignment flows, role queries, identifier rules, and expected frontend integration behavior
