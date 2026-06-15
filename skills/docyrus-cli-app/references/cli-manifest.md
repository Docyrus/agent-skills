# Docyrus CLI Manifest

LLM-ready reference for the `docyrus` CLI tool (`@docyrus/docyrus`). Generated from the live command graph (`docyrus --llms`).

---

## Global Options

| Flag | Type | Description |
|------|------|-------------|
| `-g`, `--global` | boolean | Force global `~/.docyrus/` settings instead of local `./.docyrus/` |
| `--format <toon\|json\|yaml\|md\|jsonl>` | string | Output format |
| `--help` | boolean | Show help (prints flags in kebab-case) |
| `--llms` | boolean | Print the full LLM-readable manifest |
| `--mcp` | boolean | Start as an MCP stdio server |
| `--verbose` | boolean | Show full output envelope |
| `--version` | boolean | Show version |

**Flag forms:** `--help` prints flags in kebab-case (`--app-slug`, `--from-file`); the parser also accepts the camelCase schema keys (`--appSlug`, `--fromFile`). Both forms work. This manifest uses camelCase.

## Environment Variables

| Name | Description |
|------|-------------|
| `DOCYRUS_API_CLIENT_ID` | Default Docyrus OAuth2 client id |
| `DOCYRUS_SANDBOX_APP_ID` | Active sandbox app id (injected by the sandbox runtime; default for `--appId` on sandbox/release commands) |

## Settings Scope

- Default scope is local: `./.docyrus/`
- Use `-g`/`--global` for global scope: `~/.docyrus/`
- Auth state: `<settings-root>/auth.json`
- Environment config: `<settings-root>/config.json`
- OpenAPI cache: `<settings-root>/tenans/<tenantId>/openapi.json`
- `docyrus` without a subcommand returns active environment, help commands, and auth `context`

## Output Model

Every successful command injects a top-level `context` object (`email`, `tenantName`, `tenantNo`, `tenantDisplay`); it is `null` when there is no active session. Object payloads are merged with `context`; non-object payloads become `{ data, context }`.

---

## docyrus env

Environment commands.

### docyrus env list

List available environments.

### docyrus env use

Switch active environment by id or name.

```
docyrus env use <selector>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `selector` | string | yes | Environment id or name (`live`, `prod`, `beta`, `alpha`, `dev`, `local-development`) |

Built-in environments:
- `live` (`prod` alias) -> `https://api.docyrus.com`
- `beta` -> `https://beta-api.docyrus.com`
- `alpha` -> `https://alpha-api.docyrus.com`
- `dev` (`local-development` alias) -> `https://localhost:3366`

### docyrus env which

Show which environment is active for the current folder, plus the resolved settings scope (`local`/`global`), `settingsRoot`, `configFilePath`, `authFilePath`, `cwd`, and whether a local `.docyrus/` exists.

---

## docyrus auth

Authentication commands.

### docyrus auth login

Authorize CLI using OAuth2 device flow, or provide tokens manually.

| Flag | Type | Description |
|------|------|-------------|
| `--clientId` | string | OAuth2 client id |
| `--scope` | string | OAuth2 scopes (default below) |
| `--accessToken` | string | Manual access token; skips device flow |
| `--refreshToken` | string | Manual refresh token; requires `--accessToken` |

Default scope: `openid email profile offline_access ReadWrite.All Architect.ReadWrite.All Automations.Run Reports.Run.CustomQuery Messaging.Email.Send Messaging.Sms.Send Messaging.Whatsapp.Send MCP.Connect`

Login notes:
- Client ID resolution order: `--clientId` -> `DOCYRUS_API_CLIENT_ID` -> saved local config -> saved global config -> `manual-token` (manual-token auth only)
- Manual token login falls back to `manual-token` only when no client ID can be resolved
- Local login can reuse the globally saved client ID when local config has none

### docyrus auth set-tokens

Set custom access and refresh tokens for the active environment. Same flags as `auth login` (`--clientId`, `--scope`, `--accessToken`, `--refreshToken`).

### docyrus auth logout

Revoke and clear all tenant sessions for the active account in the current environment.

| Flag | Type | Description |
|------|------|-------------|
| `--clientId` | string | OAuth2 client id override |

### docyrus auth who

Return the current authenticated user (`/v1/users/me`).

### docyrus auth tenant

Return the active tenant record (`GET /v1/tenant/current`). Read-only; no flags. Returns `id`, `no`, `name`, `accountStatus`, product/subscription refs, seat counts, `paymentChannel`, trial and subscription dates, and `onboardingStatus`. Distinct from the `auth tenants` (plural) account-management group.

### docyrus auth accounts list

List saved user accounts for the current API base URL.

### docyrus auth accounts use

Switch active account by user ID.

| Flag | Type | Description |
|------|------|-------------|
| `--userId` | string | User ID to activate |

### docyrus auth tenants list

List available tenants for an account.

| Flag | Type | Description |
|------|------|-------------|
| `--userId` | string | User ID; defaults to active account |

### docyrus auth tenants use

Switch active tenant for an account.

```
docyrus auth tenants use <tenantSelector> [options]
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `tenantSelector` | string | yes | Numeric tenant no or UUID tenant id |

| Flag | Type | Description |
|------|------|-------------|
| `--userId` | string | User ID; defaults to active account |
| `--scope` | string | Scope used only when tenant bootstrap login is required |

Selector rules: numeric -> match by `tenantNo`; non-numeric must be a UUID tenant id.

### Sandbox / CI token helpers

Mostly invoked by the sandbox runtime; all default `--appId` to `DOCYRUS_SANDBOX_APP_ID`.

| Command | Description | Key flags |
|---------|-------------|-----------|
| `docyrus auth sandbox` | Refresh and inject fresh auth tokens into the active sandbox | `--appId` |
| `docyrus auth github` | Regenerate the GitHub token and inject it into the active sandbox | `--appId`, `--cwd` |
| `docyrus auth git-credential` | Git credential helper supplying a repo-scoped GitHub token | `--operation`, `--appId` |
| `docyrus auth sso-session` | Create a short-lived SSO session token for headless browser auth | `--clientId`, `--scope`, `--targetOrigin` |

---

## docyrus apps

App commands. `list` uses `/v1/apps`; mutations and AI sub-resources route through `/v1/dev/apps/:appId`. `--appSlug` resolution first consults the bundled SYSTEM/APP catalog, then falls back to `GET /v1/apps`.

### docyrus apps list

List apps (`/v1/apps`).

| Flag | Type | Description |
|------|------|-------------|
| `--appType` | string | Optional app type filter |
| `--noCache` | boolean | Bypass the server cache and read all apps directly from the database |

### docyrus apps update

`PATCH /dev/apps/:appId`. Store fields and array/object values must be supplied via `--data`/`--from-file`. An empty resulting payload is rejected.

| Flag | Type | Description |
|------|------|-------------|
| `--appId` / `--appSlug` | string | App selector |
| `--data` / `--from-file` | string | JSON payload / file |
| `--name` | string | App name |
| `--slug` | string | New app slug |
| `--description` | string | App description |
| `--icon` | string | App icon |
| `--color` | string | App color |
| `--status` | string | App status (`active`, `design`, `development`, `draft`, `inactive`) |
| `--betaUrl` | string | Beta URL |
| `--chromeExtensionPath` | string | Chrome extension path |
| `--mobileVersionPath` | string | Mobile version path |
| `--agentContext` | string | Agent context |
| `--routePath` | string | Route path under the shared repo host (e.g. `/crm`) |

### docyrus apps set-agent-context

`PATCH /dev/apps/:appId` (`agent_context`). Provide exactly one of `--value`, `--from-file`, or `--clear`.

| Flag | Type | Description |
|------|------|-------------|
| `--appId` / `--appSlug` | string | App selector |
| `--value` | string | Agent context value (inline) |
| `--from-file` | string | Path to a text/markdown file holding the value |
| `--clear` | boolean | Clear the agent context (set to empty string) |

### docyrus apps delete / restore / permanent-delete

Archive / restore / hard-delete an app.

| Flag | Type | Description |
|------|------|-------------|
| `--appId` / `--appSlug` | string | App selector |

### docyrus apps ai-tools

CRUD over app-scoped `tenant_ai_tool` rows (`/dev/apps/:appId/ai-tools`). All commands take `--appId`/`--appSlug`.

- `list` — list AI tools for an app
- `get` / `delete` — `--toolId`
- `create` — `--name` and `--key` required
- `update` — `--toolId` plus changed fields

Create/update convenience flags: `--name`, `--key`, `--description`, `--icon`, `--type`, `--clientSideExecution`, `--needsApproval`, `--environments` (CSV), `--dynamicApprovalFormula`, `--inputJsonSchema` (JSON), `--outputJsonSchema` (JSON), `--secureExecCode`, `--customQuerySqlQuery`, `--customQueryFilters` (JSON), `--dataSourceQueryDataSourceId`, `--dataSourceQueryColumns` (JSON), `--dataSourceQueryFilters` (JSON), `--dataSourceQueryFilterKeyword`, `--dataSourceQueryFormulas` (JSON), `--dataSourceQueryChildQueries` (JSON), `--dataSourceQueryLimit`, plus `--data`/`--from-file`.

Platform-managed fields (`group`, `avatar`, `restricted`, `cost`, `development_status`, `owner_product_id`, `core_action_id`, `core_data_provider_id`) are not settable on app-scoped tools — the endpoint leaves them at their defaults.

### docyrus apps actions

CRUD over standalone `tenant_action` rows (`/dev/apps/:appId/actions`), plus the action-type picker and run. All commands take `--appId`/`--appSlug`.

- `list` — list app actions
- `types` — list selectable action types (`/dev/apps/:appId/actions/types`); excludes client-only and automation-flow-only actions (create/update record, wait-for); each row carries `color` (Tailwind color name)
- `get` / `delete` — `--actionId`
- `create` — `--name` and `--coreActionId` required (`--coreActionId` is create-only / immutable)
- `update` — `--actionId` plus changed fields (`--coreActionId` is rejected)
- `run` — `--actionId`; posts the body as the action input record to the slug-based public endpoint `POST /v1/apps/:appSlug/actions/:actionId/run` (accepts `--appId`, reverse-resolved to the slug); body via `--data`/`--from-file` (an empty body is allowed)

Create/update convenience flags: `--name`, `--coreActionId` (create only), `--status`, `--options` (JSON), `--conditions` (JSON), `--sourceDataSourceId`, `--inputDataSourceId`, `--targetDataSourceId`, `--targetDataSourceFieldId`, `--targetDataSourceCondition` (JSON), `--connectionId`, `--connectionAccountId`, `--webhookId`, `--requestMethod`, `--contentType`, `--customEndpoint`, `--relativeEndpoint`, `--batch`, `--batchSize`, `--customHeaders` (JSON), `--inputTransformer` (JSON), `--outputTransformer`, `--batchTransformer`, `--errorTransformer`, `--inputTemplate` (JSON), `--preActionRequest` (JSON), `--postActionRequest` (JSON), `--inputJsonSchema` (JSON), `--outputJsonSchema` (JSON), plus `--data`/`--from-file`.

---

## docyrus ds

Data source commands for CRUD on records.

### docyrus ds get

Get data source metadata (fields, types, relations).

```
docyrus ds get <appSlug> <dataSourceSlug>
```

### docyrus ds list

List data source items with the full query engine.

```
docyrus ds list <appSlug> <dataSourceSlug> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--columns` | string | Columns to select; comma-separated or JSON array (supports relation `()`, spread `...`, alias `:`, function `@`) |
| `--distinctColumns` | string | Distinct columns; comma-separated or JSON array |
| `--filters` | string | JSON filter group (`{"combinator":"and","rules":[...]}`) |
| `--filterKeyword` | string | Full-text keyword filter |
| `--orderBy` | string | Sort order string or JSON object/array |
| `--limit` | number | Max records |
| `--offset` | number | Skip N records |
| `--fullCount` | boolean | Include total count |
| `--formulas` | string | JSON formulas object |
| `--calculations` | string | JSON calculations array |
| `--groupSummaries` | boolean | Return per-group summaries when calculations are used |
| `--collapseRows` | boolean | Collapse rows into a single aggregated array |
| `--expand` | string | Expand columns; comma-separated or JSON array |
| `--pivot` | string | JSON pivot configuration |
| `--childQueries` | string | JSON child query array |

### docyrus ds create

Create item(s). Array payload triggers `POST /apps/:appSlug/data-sources/:dataSourceSlug/items/bulk` (max 50 items).

```
docyrus ds create <appSlug> <dataSourceSlug> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |
| `--from-file` | string | Path to a `.json` or `.csv` payload file |

### docyrus ds update

Update item(s). Object payload uses the single-item endpoint and requires `recordId`; array payload uses the bulk endpoint, requires `id` in each item, and must not include positional `recordId`. Max 50 per batch.

```
docyrus ds update <appSlug> <dataSourceSlug> [recordId] [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |
| `--from-file` | string | Path to a `.json` or `.csv` payload file |

### docyrus ds delete

Delete an item.

```
docyrus ds delete <appSlug> <dataSourceSlug> <recordId>
```

### docyrus ds comments create

Create a record-scoped comment.

```
docyrus ds comments create <appSlug> <dataSourceSlug> <recordId> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--message` | string | Comment message |
| `--data` / `--from-file` | string | Full comment DTO as JSON / file |
| `--parentId` | string | Parent comment ID |
| `--assignedTo` | string | Assigned user ID |
| `--attachments` | string | JSON attachments payload |
| `--level` | number | Comment level |
| `--status` | number | Comment status |
| `--done` | boolean | Mark comment as done |

### docyrus ds files upload

Upload a record-scoped file attachment (`multipart/form-data`).

```
docyrus ds files upload <appSlug> <dataSourceSlug> <recordId> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--file` | string | Path to the local file |
| `--contentType` | string | Override the inferred MIME type |
| `--publicFile` | boolean | Store the file in the public tenant bucket |

---

## docyrus dsql

Run read-only logical SQL queries and inspect the token-efficient DSQL schema. Logical tables are named `appSlug.dataSourceSlug` (e.g. `base.contact`). Requires `DS.Read.*` / `DS.ReadWrite.*` scopes.

### docyrus dsql query

Run a read-only PostgreSQL-compatible `SELECT` over logical data-source tables (`PUT /dsql/query`). The SQL is resolved from the positional argument, `--from-file`, or stdin (in that order). Throttled to 60 requests/minute.

```
docyrus dsql query "select id, email from base.contact limit 100"
docyrus dsql query --from-file ./report.sql
echo "select count(*) from base.task" | docyrus dsql query
```

| Flag | Type | Description |
|------|------|-------------|
| `--from-file` | string | Path to a file containing the SQL query |

Returns `{ data: [...], meta: { count } }`.

### docyrus dsql generate

Generate a DSQL query from a natural-language question using the base **DSQL generator agent** (`POST /ai/agents/:agentId/chat`). Returns the query text only — it does **not** run it. The question is resolved from the positional argument, `--from-file`, or stdin.

```
docyrus dsql generate "top 10 contacts by revenue this year"
docyrus dsql generate --from-file ./question.txt
```

| Flag | Type | Description |
|------|------|-------------|
| `--from-file` | string | Path to a file containing the question |
| `--agentId` | string | Override the default DSQL generator agent id |
| `--deploymentId` | string | Optional agent deployment id |

The agent returns structured output: `{ prompt, query }` (it echoes a normalized `prompt` alongside the generated `query`).

### docyrus dsql ask

Generate a DSQL query from a natural-language question, then run it through `PUT /dsql/query` and return the rows. Combines `generate` + `query`.

```
docyrus dsql ask "how many open tasks are assigned to me?"
```

| Flag | Type | Description |
|------|------|-------------|
| `--from-file` | string | Path to a file containing the question |
| `--agentId` | string | Override the default DSQL generator agent id |
| `--deploymentId` | string | Optional agent deployment id |

Returns `{ prompt, query, data: [...], meta: { count } }`.

### docyrus dsql schema app

Return the DSQL schema for every queryable data source in an app (`GET /dsql/schema/apps/:appSlug`).

```
docyrus dsql schema app base
```

### docyrus dsql schema data-source

Return the DSQL schema for a single data source (`GET /dsql/schema/apps/:appSlug/data-sources/:dataSourceSlug`).

```
docyrus dsql schema data-source base contact
```

### docyrus dsql schema data-sources

Return the DSQL schema for the data sources matching the given ids (`GET /dsql/schema/data-sources?ids=...`).

```
docyrus dsql schema data-sources --ids ds-1,ds-2
```

| Flag | Type | Description |
|------|------|-------------|
| `--ids` | string | Comma-separated data source ids (required) |

---

## docyrus studio

Dev-app schema CRUD. Data-source/field commands route through `/v1/dev/apps/:appId/data-sources...`; data-view/form commands route through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/...`; webform/html-template/email-template commands route through `/v1/dev/webforms`, `/v1/dev/html-templates`, `/v1/dev/email-templates`.

Selector rules:
- app selector: exactly one of `--appId` or `--appSlug`
- data source selector: exactly one of `--dataSourceId` or `--dataSourceSlug` when required
- field selector: exactly one of `--fieldId` or `--fieldSlug` when required

Write payload rules:
- use `--data '<json>'` or `--from-file ./payload.json` (JSON only), not both
- flags override conflicting keys from the JSON payload
- batch commands accept a root array or a root object containing the expected DTO key

### Data source commands

| Command | Route | Notes |
|---------|-------|-------|
| `list-data-sources` | `GET /dev/apps/:app/data-sources` | `--expand fields` |
| `get-data-source` | `GET .../:id` | app + data source selector |
| `create-data-source` | `POST .../data-sources` | `--title`, `--name`, `--slug`, `--type`, `--icon`, `--dataSharing`, `--meta` (JSON) |
| `update-data-source` | `PATCH .../:id` | create flags + data source selector |
| `delete-data-source` | `DELETE .../:id` | archives the data source |
| `restore-data-source` | restore | requires `--dataSourceId` (slug unavailable for archived rows) |
| `permanent-delete-data-source` | hard delete | requires `--dataSourceId` |
| `bulk-create-data-sources` | `POST .../bulk` | DTO key `dataSources` |

### Field commands

| Command | Route | Notes |
|---------|-------|-------|
| `list-fields` | `GET .../:ds/fields` | app + data source selector |
| `get-field` | `GET .../fields/:id` | + field selector |
| `create-field` | `POST .../fields` | `--name`, `--slug`, `--type`, `--readOnly`, `--status`, `--defaultValue`, `--relationDataSourceId`, `--sortOrder`, `--tenantEnumSetId`, `--options` (JSON), `--validations` (JSON) |
| `update-field` | `PATCH .../fields/:id` | create flags + field selector |
| `delete-field` | `DELETE .../fields/:id` | app + data source + field selector |
| `create-fields-batch` | `POST .../fields/batch` | DTO key `fields` |
| `update-fields-batch` | `PATCH .../fields/batch` | DTO key `fields`; backend expects `fields[].fieldId` |
| `delete-fields-batch` | `DELETE .../fields/batch` | DTO key `fieldIds` |

The CLI normalizes common live-output shapes before sending batches: `id -> fieldId`, `id -> enumId`, `read_only -> readOnly`, `default_value -> defaultValue`, `relation_data_source_id -> relationDataSourceId`, `options -> editorOptions`.

### Enum commands

| Command | Route | DTO key |
|---------|-------|---------|
| `list-enums` | `GET .../fields/:id/enums` | — |
| `create-enums` | `POST .../fields/:id/enums` | `enums` |
| `update-enums` | `PATCH .../fields/:id/enums` | `enums` (backend expects `enums[].enumId`) |
| `delete-enums` | `DELETE .../fields/:id/enums` | `enumIds` |

### Search commands (tenant-wide, paged)

| Command | Description | Flags |
|---------|-------------|-------|
| `search-fields` | Search fields across all data sources | `--dataSourceId` (CSV), `--type` (CSV), `--keyword`, `--limit`, `--offset` |
| `search-enums` | Search enums across data sources, fields, and enum sets | `--dataSourceId`, `--enumSetId`, `--fieldId`, `--limit`, `--offset` |
| `search-enum-sets` | Search enum sets | `--limit`, `--offset` |

### Data view commands

Route through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/views`. Pass either `--appId`/`--appSlug` and either `--dataSourceId`/`--dataSourceSlug`; the CLI resolves the missing side.

| Command | Route | Notes |
|---------|-------|-------|
| `list-data-views` | `GET .../views` | `--tenantAppId` optional |
| `get-data-view` | `GET .../views/:viewId` | `--viewId` required |
| `create-data-view` | `POST .../views` | `--name`, `--description`, `--tenantAppId`, `--columns` (JSON), `--filters` (JSON), `--sort` (JSON), `--color`, `--icon`, `--colorRules` (JSON), `--quickFilterFields` (JSON), `--isDefault`, `--sortOrder` |
| `update-data-view` | `PUT .../views/:viewId` | create flags + `--viewId`, `--archived` |
| `delete-data-view` | `DELETE .../views/:viewId` | `--viewId` required |

### Form commands

Route through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/forms`.

| Command | Route | Notes |
|---------|-------|-------|
| `list-forms` | `GET .../forms` | — |
| `get-form` | `GET .../forms/:formId` | `--formId` required |
| `create-form` | `POST .../forms` | `--name`, `--description`, `--title`, `--subtopic`, `--color`, `--icon`, `--layout` (JSON), `--isDefault`, `--status` |
| `update-form` | `PUT .../forms/:formId` | create flags + `--formId`, `--archived` |
| `delete-form` | `DELETE .../forms/:formId` | `--formId` required |

### Webform commands

Route through `/v1/dev/webforms`. CRUD by `--webformId`. List/create accept `--dataSourceId`, or `--dataSourceSlug` with `--appId`/`--appSlug` to resolve. When `dataSourceId` is omitted on create, submissions land in the tenant-schema `webform_record` table.

| Command | Route | Notes |
|---------|-------|-------|
| `list-webforms` | `GET /dev/webforms` | optional data source filter |
| `get-webform` | `GET /dev/webforms/:id` | `--webformId` |
| `create-webform` | `POST /dev/webforms` | `--name`, `--schema` (JSON), `--status` (1 active, 2 inactive), `--webformOptions` (JSON), `--sandbox`, `--css` |
| `update-webform` | `PATCH /dev/webforms/:id` | `--webformId` + same fields |
| `delete-webform` | `DELETE /dev/webforms/:id` | `--webformId` |

### HTML template commands (HTML / PDF / DOCX export)

Route through `/v1/dev/html-templates`. CRUD by `--templateId`. Data source binding by `--dataSourceId` or `--dataSourceSlug` (slug requires `--appId`/`--appSlug`).

| Command | Route | Notes |
|---------|-------|-------|
| `list-html-templates` | `GET /dev/html-templates` | `--isDefault`, `--limit`, `--offset` |
| `get-html-template` | `GET /dev/html-templates/:id` | `--templateId` |
| `create-html-template` | `POST /dev/html-templates` | `--name`, `--filenameTmpl`, `--pageOrientation`, `--sourceType` (`html`/`pdf`/`docx`), `--marginLeft/Right/Top/Bottom`, `--pageFormat` (`A4`/`Letter`), `--body`, `--isDefault`, `--headerTmpl`, `--footerTmpl`, `--styles` |
| `update-html-template` | `PUT /dev/html-templates/:id` | create flags + `--templateId` |
| `delete-html-template` | `DELETE /dev/html-templates/:id` | `--templateId` |

### Email template commands

Route through `/v1/dev/email-templates`. CRUD by `--templateId`. Data source binding optional.

| Command | Route | Notes |
|---------|-------|-------|
| `list-email-templates` | `GET /dev/email-templates` | `--limit`, `--offset` |
| `get-email-template` | `GET /dev/email-templates/:id` | `--templateId` |
| `create-email-template` | `POST /dev/email-templates` | `--name`, `--subject`, `--body`, `--ownership` (`system`/`user`) |
| `update-email-template` | `PUT /dev/email-templates/:id` | create flags + `--templateId` |
| `delete-email-template` | `DELETE /dev/email-templates/:id` | `--templateId` |

---

## docyrus automation

Automation, trigger, and action node CRUD. Routes through `/v1/dev/apps/:appId/automations`.

Selector rules:
- app selector: exactly one of `--appId` or `--appSlug`
- automation selector: `--automationId` only (no slug)
- trigger `--type` (kebab-case): `record-created`, `record-modified`, `record-deleted`, `recurrence`, `app-event`, `webhook`, `emailhook`, `webform`, `button-activation`, `manual-activation`
- node `--type` (kebab-case): `external-action`, `send-email`, `send-notification`, `create-record`, `update-records`, `request-approval`, `request-input`, `http-request`, `data-source-query`, `custom-query`, `generate-document`, `ai-prompt`, `ai-agent`, `execute-script`, `wait-for`

Write payload rules:
- write commands accept `--data '<json>'` or `--from-file ./payload.json` (JSON only)
- convenience flags are camelCase and converted to `snake_case`
- nested objects (`data`, `field_mapping`, `dynamic_field_mapping`, `condition`, `input_template`, `input_transformer`, `custom_headers`, `pre_action_request`, `post_action_request`, `target_data_source_condition`) must be supplied via `--data`/`--from-file`
- `delete`, `delete-trigger`, `delete-node` return `{ deleted: true, id }` (API returns 204)

### Automation CRUD

| Command | Route | Notes |
|---------|-------|-------|
| `list` | `GET .../automations` | app selector |
| `get` | `GET .../automations/:id` | `--automationId` |
| `create` | `POST .../automations` | `--name`, `--triggerType` (camelCase: `recordCreated`, `recordModified`, `recordDeleted`, `recurrence`, `appEvent`, `webhook`, `emailhook`, `webform`, `buttonActivation`, `manualActivation`), `--status`, `--sourceDataSourceId`, `--triggerDataSourceId`, `--triggerDataProviderId`, `--triggerWebhookId` |
| `update` | `PATCH .../automations/:id` | `--automationId`, `--name`, `--status`, `--sourceDataSourceId` |
| `delete` | `DELETE .../automations/:id` | `--automationId` |

### Trigger commands

`list-triggers` / `get-trigger` are derived from the automation GET response. `create-trigger`/`update-trigger` route through `POST|PATCH .../triggers/:type[/ :triggerId]`; `delete-trigger` routes through `DELETE .../triggers/:triggerId` (type-independent).

`create-trigger` flags (`--automationId`, `--type` required):

| Flag | Type | Description |
|------|------|-------------|
| `--active` | boolean | Whether the trigger is active |
| `--sourceDataSourceId` | string | Source data source (record-*, recurrence, button/manual) |
| `--maxRunPerRecord` | number | Max runs per record (record-*, recurrence) |
| `--modifiedColumns` | string | Comma-separated modified columns (record-modified) |
| `--modifiedColumnsCondition` | string | `all` or `any` (record-modified) |
| `--recurrenceFrequency` | string | `hour`, `day`, `week`, `month`, `year` (recurrence) |
| `--recurrenceInterval` | number | Recurrence interval (recurrence) |
| `--recurrenceMinutes` | number | Minutes `0\|15\|30\|45` (recurrence) |
| `--recurrenceWeekDays` | string | Comma-separated `MON,TUE,...` (recurrence) |
| `--recurrenceMonthDays` | string | `DAY_OF_MONTH` or `DAY_OF_WEEK` (recurrence) |
| `--recurrenceStartDate` / `--recurrenceEndDate` | string | ISO date (recurrence) |
| `--recurrenceRunAt` | string | `HH:mm` (recurrence) |
| `--dataProviderId` | string | Data provider id, a.k.a. connector (app-event); obtain via `docyrus connect list-connectors` |
| `--dataProviderWebhookId` | string | Data provider webhook id (app-event); obtain via `docyrus connect get-connector <slug>` |
| `--webhookId` | string | Webhook id (webhook, emailhook) |
| `--webhookName` | string | Name for auto-created webhook (webhook, emailhook) |
| `--webformId` | string | Webform id (webform) |
| `--data` / `--from-file` | string | JSON payload / file |

`update-trigger` adds `--triggerId` (required). `delete-trigger` takes `--automationId` and `--triggerId`.

### Action node commands

`list-nodes` (`GET .../nodes`), `get-node` (`GET .../nodes/:nodeId`). `create-node`/`update-node` route through `POST|PATCH .../nodes/:type[/ :nodeId]`; `delete-node` routes through `DELETE .../nodes/:nodeId` (type-independent).

`create-node` flags (`--automationId`, `--type` required):

| Flag | Type | Description |
|------|------|-------------|
| `--name` / `--description` | string | Node name / description |
| `--subType` | string | Sub type discriminator |
| `--parent` | string | Parent node id |
| `--active` | boolean | Whether the node is active |
| `--actionTypeId` | string | Action type id (required for `external-action`; maps to `core_action.id`) |
| `--sourceDataSourceId` | string | Source data source id (external-action) |
| `--targetDataSourceId` | string | Target data source id (create-record, update-records, http-request, data-source-query, external-action) |
| `--targetDataSourceFieldId` | string | Target field id (update-records) |
| `--connectionId` / `--connectionAccountId` | string | Connection (http-request, external-action) |
| `--webhookId` | string | Webhook id (external-action) |
| `--inputDataSourceId` | string | Input data source id (request-approval, request-input) |
| `--requestMethod` | string | HTTP method (http-request) |
| `--contentType` | string | HTTP content type (http-request) |
| `--customEndpoint` | string | HTTP endpoint (http-request) |
| `--relativeEndpoint` | boolean | Endpoint relative to connection base url (http-request) |
| `--batch` / `--batchSize` | boolean / number | HTTP batching 1..10000 (http-request) |
| `--outputTransformer` / `--batchTransformer` / `--errorTransformer` | string | Transformer expressions (http-request) |
| `--data` / `--from-file` | string | JSON payload / file |

`update-node` adds `--nodeId` (required). `delete-node` takes `--automationId` and `--nodeId`.

Notes:
- `external-action` create validates `data` against the linked `core_action.input_json_schema` and creates the `tenant_action` row in the same transaction.
- `wait-for` takes no flat flags beyond the common base. Set the delay inside `data`: `delaySeconds` (integer ≤ 30 days / 2_592_000) or `delayValue` + `delayUnit` (`seconds`/`minutes`/`hours`/`days`). It forwards input unchanged and queues next step(s) with a deferred `tenant_job_queue.process_after`.

```sh
docyrus automation create-node \
  --appSlug crm --automationId 9c4f… \
  --type wait-for --name "Wait 2 hours" --parent <previous-node-id> \
  --data '{"data":{"delayValue":2,"delayUnit":"hours"}}'
```

---

## docyrus agent

Custom-agent CRUD for the dev controller (`/v1/dev/apps/:appId/agents...`). Distinct from the pi-agent launchers (`opsy`/`cody`/`coder`).

Selector and payload rules:
- app selector: exactly one of `--appId` or `--appSlug`
- the parent agent is `--agentId`; an individual sub-resource row is `--id`
- `create`/`update` accept `--data`/`--from-file` (JSON) plus camelCase convenience flags that map 1:1 onto the matching `Create*Dto`/`Update*Dto` `snake_case` keys
- JSON flags are parsed from JSON; list flags (e.g. `--standardSuggestions`, `--supportedFileFormats`, `--ownerProductId`) are comma-separated arrays
- `createOnly` fields appear only on `create`; `updateOnly` fields (e.g. `--archived`) only on `update`; incur rejects unknown flags
- `delete` returns `{ deleted: true, id }`

### Agent resource

| Command | Notes |
|---------|-------|
| `list` / `get` / `delete` | `--agentId` for get/delete |
| `create` | requires `--skillName` |
| `update` | `--agentId` + changed fields; supports `--archived` |
| `upload` | multipart image; `--column` (`avatar`/`gallery_image`), `--file`, `--contentType` |

`create`/`update` convenience flags: `--name`, `--agentName`, `--skillName`, `--description`, `--category`, `--mode`, `--instructions`, `--welcomeMessage`, `--defaultAiModelId`, `--backupAiModelId`, `--cotAiModelId`, `--defaultReasoningLevel`, `--parent`, `--ownership` (`SYSTEM`/`APP`/`TENANT`), `--tenantAppId`, `--workDataSourceId`, `--status`, `--developmentStatus`, `--sortOrder`, `--temperature`, `--maxTokens`, `--isAgent`, `--isAssistant`, `--isSkill`, `--jsonOutput`, `--hasWorkflow`, `--multipleDeployment`, `--supportTools`, `--supportDataSources`, `--supportFiles`, `--supportKnowledgeBase`, `--supportWebSearch`, `--standardSuggestions` (CSV), `--supportedFileFormats` (CSV), `--ownerProductId` (CSV), `--instructionSchema` (JSON), `--inputFormSchema` (JSON), `--outputRenderSchema` (JSON), `--memoryOptions` (JSON), `--helpDocs` (JSON), plus `--data`/`--from-file`. The long tail (`compaction_*`, `context_clear_*`, `output_*`, `prompt_*`, …) goes through `--data`/`--from-file`.

### Sub-resource groups

Each group supports `list`/`get`/`create`/`update`/`delete` (unless noted). All take `--appId`/`--appSlug` and `--agentId`; row commands take `--id`. Key create flags shown.

| Group | Key create flags / notes |
|-------|--------------------------|
| `models` | DTO fields via flags or `--data`/`--from-file` |
| `tools` | `--coreAiToolId` (required), `--defaultParams` (JSON), `--tenantConnectionId`, `--tenantConnectionAccountId` |
| `data-sources` | `--tenantDataSourceId` (required), `--privilege` |
| `docs` | DTO fields via flags or `--data`/`--from-file` |
| `mcps` | DTO fields via flags or `--data`/`--from-file` |
| `connections` | `--connectedAiAgentId` (required), `--connectionType` (required); routes via `agent-connections` segment |
| `dynamic-contexts` | DTO fields via flags or `--data`/`--from-file` |
| `tasks` | `--parent_message_id` (createOnly); `--status`/`--error_message`/`--retry_count` (updateOnly) |
| `recurring-tasks` | backend requires `cronExpression`/`nextRunAt` |
| `workflow-steps` | backend requires `inputSchema`/`outputSchema` |
| `deployments` | nested arrays (`tools`) via `--tools` JSON or `--data`/`--from-file` |
| `deployment-tools` | nested under a deployment; `list`/`create`/`update`/`delete` (no `get`); requires `--deploymentId` |
| `deployment-data-sources` | same as `deployment-tools` |
| `workflow-jobs` | read-only: `list`/`get`/`traces`/`delete` |

---

## docyrus messaging

Tenant email account discovery and transactional email send. Routes through `/v1/messaging/email/*`. Requires the `Messaging.Email.Send` OAuth2 scope. Credentials are never returned.

### docyrus messaging accounts

`GET /messaging/email/accounts`. Each account exposes `id`, `name`, `provider`, `senderEmail`, `senderName`, `isUserAccessible`, `allowOverrideName`, `allowOverrideEmail`, `createdOn`.

### docyrus messaging email send

`POST /messaging/email/accounts/:accountId/send`

| Flag | Type | Description |
|------|------|-------------|
| `--accountId` | string | Tenant email account UUID (required) |
| `--to` / `--cc` / `--bcc` / `--replyTo` | string | Comma-separated addresses |
| `--subject` | string | Subject (max 998 chars) |
| `--body` | string | HTML or text body (max 1 000 000 chars) |
| `--sendAsUser` | boolean | Send using the authenticated user's identity when allowed |
| `--data` / `--from-file` | string | Full JSON payload; overrides individual flags |

Limits: up to 50 addresses per list, up to 10 attachments. Attachments are `{ filePath, fileName?, mimeType? }` referencing tenant-scoped storage paths. Response: `{ messageId, provider, accepted, rejected }`.

---

## docyrus connect

Connector and action commands.

### docyrus connect list-connectors

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--q` | string | — | Keyword search on name, slug, or description |
| `--limit` | number | 100 | Max results |
| `--offset` | number | 0 | Result offset |

### docyrus connect get-connector / get-action / list-connections

```
docyrus connect get-connector <slug>
docyrus connect get-action <slug> <actionKey>
docyrus connect list-connections <slug>
```

`get-connector` returns data sources + actions; `get-action` returns input/output JSON schemas; `list-connections` returns tenant and user connections.

### docyrus connect curl

Send an HTTP request through a connector's provider auth.

```
docyrus connect curl <slug> <endpoint> [options]
```

| Flag | Alias | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--method` | `-X` | string | GET | HTTP method |
| `--data` | `-d` | string | — | JSON payload (body for POST/PUT/PATCH, query for GET) |
| `--contentType` | | string | application/json | Content-Type header |
| `--headers` | | string | — | JSON object of additional headers (can override Authorization) |
| `--connectionId` | `-c` | string | — | Tenant connection ID override |
| `--connectionAccountId` | | string | — | Connection account ID |

### docyrus connect run-action

Run a connector action directly by provider slug + action key via `POST /v1/connectors/:slug/actions/:actionKey/run`. (To run a persisted app action, use `apps actions run`.)

```
docyrus connect run-action <slug> <actionKey> [options]
```

| Flag | Alias | Type | Default | Description |
|------|-------|------|---------|-------------|
| `--params` | `-p` | string | — | JSON object with action input parameters |
| `--connectionId` | `-c` | string | — | Tenant connection ID override |
| `--connectionAccountId` | | string | — | Tenant connection account ID |
| `--dryRun` | `-n` | boolean | false | Preview the request without executing |

---

## docyrus discover

Discovery commands for the tenant OpenAPI spec. All require an active login session. Commands other than `discover api` auto-download the spec if missing locally (public bucket -> authenticated `GET /v1/api/openapi.json` -> public retry).

| Command | Description |
|---------|-------------|
| `discover api` | Download tenant OpenAPI spec for the active tenant |
| `discover namespaces` | List deduplicated namespace prefixes (e.g. `/v1/users`) |
| `discover path <prefix>` | List endpoints (method + description) for a path prefix (`/v1` optional) |
| `discover endpoint <selector>` | Full endpoint object; selector `/path` (GET) or `[METHOD]/path` |
| `discover entity <name>` | Full entity schema by name (case-sensitive, e.g. `UserEntity`) |
| `discover search <query>` | Search endpoint paths and entity names (comma-separated terms) |

---

## docyrus curl

Send arbitrary requests to the Docyrus API.

```
docyrus curl <path> [options]
```

| Flag | Alias | Type | Description |
|------|-------|------|-------------|
| `--request` | `-X` | string | HTTP method |
| `--header` | `-H` | array | Request header (repeatable) |
| `--data` | `-d` | string | Request payload |
| `--get` | `-G` | boolean | Send data as query string |
| `--include` | `-i` | boolean | Include status and response headers |
| `--noAuth` | | boolean | Skip Authorization header |

Notes: path-only (no absolute URLs); `/v1` prefix auto-normalized; JSON payloads auto-detect `Content-Type: application/json`; default method GET (POST if `-d` provided).

---

## docyrus docy

Chat with the platform's main AI agent.

```
docyrus docy "<prompt>"
```

| Flag | Type | Description |
|------|------|-------------|
| `--agentId` | string | Agent ID; defaults to the Docyrus CLI agent |
| `--deploymentId` | string | Optional agent deployment ID |

Renders markdown in a TTY; preserves structured output with `--json`/`--verbose`/`--format`.

---

## docyrus opsy / cody / coder (pi agents)

Launch the pi agent runtime. `opsy` = Cowork Agent; `cody` = Coding Agent (`coder` is an alias of `cody`). Each takes an optional `prompt` argument (omit to open the interactive TUI).

| Flag | Type | Description |
|------|------|-------------|
| `--print` | boolean | One-shot print mode instead of the TUI |
| `--mode` | string | Print mode output: `text` or `json` |
| `--continue` | boolean | Continue the previous pi session |
| `--resume` | boolean | Open the pi session picker |
| `--provider` | string | Model provider |
| `--model` | string | Model pattern or full `provider/model` id |
| `--thinking` | string | Thinking level |
| `--session` / `--sessionDir` | string | Specific session file / storage dir override |
| `--apiKey` | string | Temporary provider API key override for this run |
| `--verbose` | boolean | Verbose pi startup output |

---

## docyrus server

Start an HTTP server bridging a pi agent to the AI SDK `useChat` protocol (SSE chat stream with reconnect/resume).

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--profile` | string | `coder` | Agent profile to use |
| `--port` | number | 3111 | Server port |
| `--provider` / `--model` / `--thinking` | string | — | Model selection |
| `--sessionDir` | string | — | Session storage directory override |
| `--apiKey` | string | — | Temporary provider API key override |
| `--auth` | string | — | Require this bearer token for all HTTP requests |
| `--verbose` | boolean | — | Verbose pi startup output |
| `--sandbox` | boolean | — | Enable sandbox browser mode (remote Cloudflare Browser Rendering) |
| `--desktop` | boolean | — | Enable desktop browser automation tools (`docyrus_browser_*`) |

---

## docyrus browser

Browser automation (local Chrome on `:9222` or remote Cloudflare). Commands return JSON with a `mode` field.

| Command | Description | Key flags / args |
|---------|-------------|------------------|
| `start` | Start a session | `--profile` (copy default Chrome profile, local only) |
| `close` | Close the session | `--kill` (kill local Chrome) |
| `nav <url>` | Navigate / open URL | `--new`, `--reload` |
| `tabs` | List/switch tabs | `--switch <index>` |
| `info` | Page URL, title, viewport, scroll | — |
| `snapshot` | Compact element refs (`@e1`) for interaction | `--all`, `--selector` |
| `click <target> [y]` | Click ref `@e1`, CSS selector, or `x y` coords | `--timeout` |
| `fill <target> <value>` | Type into input/textarea | `--timeout` |
| `select <target> <value>` | Select a dropdown option | `--timeout` |
| `eval <code>` | Evaluate JS in the active tab | `--timeout` |
| `wait [ms]` | Wait for delay/condition | `--idle`, `--selector`, `--url`, `--timeout` |
| `screenshot` | Capture the tab | `--full`, `--base64` |
| `content <url>` | Extract readable markdown | — |
| `cookies` | Show cookies | `--name`, `--domain` |
| `console` | Capture console messages | `--level`, `--listen <ms>` |
| `network` | Inspect captured requests | `--method`, `--status`, `--url`, `--listen <ms>` |
| `devtools <subcommand>` | Read `@docyrus/devtools` state/errors/issues/console | `--level` |
| `run-script <script>` | Run a CDP script file | `--appSlug`, `--appId`, `--keepAlive` |

---

## docyrus knowledge

Repo knowledge-graph commands operating on `docyrus/knowledge`. Local dev-workflow tooling.

| Command | Description |
|---------|-------------|
| `init` | Initialize `docyrus/knowledge` + external-agent integration files (`--brief`, `--installGitHook`) |
| `generate-initial` | Generate/refresh starter structure from repo facts (`brief` arg) |
| `refresh` | Refresh managed starter files (`--staged`, `--sections`) |
| `search <query>` | Semantic search (`--limit`, `--reindex`) |
| `section <query>` | Show a section with refs and backlinks |
| `locate <query>` | Find sections by exact/short/fuzzy match |
| `refs <query>` | Find markdown/code references to a section or symbol (`--scope`) |
| `expand [text]` | Expand `[[refs]]` into resolved context (`--stdin`) |
| `check` | Validate the graph (links/backlinks) |
| `doctor` | Explain drift, quality issues, hot spots |
| `config` | Show resolved graph/cache/provider config |
| `list-impacted` | List sections likely impacted by changes (`--staged`) |
| `audit-staged` | Run a staged-diff knowledge audit |
| `pre-commit` | Hook-oriented staged knowledge gate |
| `hook <agent> <event>` | Internal hook command for external-agent integrations |

---

## docyrus project-plan

Repo-tracked project plan graph (phases -> features -> tasks). Token-efficient list/find/upsert/status commands. Local dev-workflow tooling.

| Command | Description |
|---------|-------------|
| `ensure` / `check` / `config` | Ensure graph exists / validate / show resolved paths |
| `show` / `summary` | Show hierarchy / summary statistics |
| `list-phases` / `list-features` / `list-tasks` | Slim listings (`list-tasks`: `--phaseId`, `--featureId`, `--status`, `--limit`, `--includeSummary`) |
| `find-tasks` | Filter tasks (`--title`, `--titleContains`, `--summaryContains`, `--status`, `--type`, `--assignee`, `--featureId`, `--phaseId`, `--taskId`, `--limit`) |
| `get-task` | Get a task with linked local subtasks (`--taskId`) |
| `upsert-phase` / `upsert-feature` / `upsert-task` | Create/update graph nodes |
| `set-order` | Set display order of a phase/feature/task (`--order`) |
| `set-task-status` | Update a task status (`--taskId`, `--status`) |
| `create-linked-todo` | Create a local `.pi/todos` subtask linked to a task |
| `upsert-from-architect` / `upsert-from-plan` | Internal sync from `/architect` and `/plan` artifacts |

---

## docyrus release

Release and versioning commands.

### docyrus release status

Show current release status and unreleased changes.

### docyrus release new-version

Create a new release with version bump, changelog generation, and optional GitHub release.

| Flag | Type | Description |
|------|------|-------------|
| `--bump` | string | SemVer bump type (auto-detect if omitted) |
| `--version` | string | Explicit target version (e.g. `1.0.0`) |
| `--appId` | string | App ID for the release record; defaults to `DOCYRUS_SANDBOX_APP_ID` |
| `--dryRun` | boolean | Preview changes without committing |
| `--skipChangelog` / `--skipTag` / `--skipGithubRelease` / `--skipDbRelease` | boolean | Skip individual steps |

---

## docyrus tui

Launch the OpenTUI terminal UI.

Notes:
- requires Bun installed locally
- reuses the existing CLI command graph
- intended for interactive terminal usage rather than browser embedding
