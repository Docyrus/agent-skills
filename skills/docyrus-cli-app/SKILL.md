---
name: docyrus-cli-app
description: Use the Docyrus CLI (`docyrus`) to interact with the Docyrus platform from the terminal. Use when the user asks to authenticate, list apps, query or manage data records (`ds`), manage dev app data source schema objects (`studio`) including data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX export templates, and email templates, manage automations and their triggers/action nodes (`automation`), build or edit custom AI agents and their sub-resources (`agent`), set an app's AI agent context or manage app-scoped AI tools (`apps`), list tenant email accounts and send emails (`messaging`), discover and call connectors (`connect`), send API requests, switch environments, tenants, or accounts, discover tenant OpenAPI specs, drive browser automation (`browser`), chat with platform agents (`docy`), launch the pi cowork/coding agents (`opsy`/`cody`/`coder`), run the agent bridge server (`server`), manage the repo knowledge graph (`knowledge`) and project plan (`project-plan`), cut releases (`release`), or use the Bun-powered terminal UI via `docyrus tui`. Triggers on tasks involving docyrus CLI commands, terminal-based Docyrus operations, `docyrus ds list`, `docyrus ds comments`, `docyrus ds files upload`, `docyrus studio`, `docyrus studio search-fields`, `docyrus automation`, `docyrus automation create-node`, `docyrus agent`, `docyrus apps set-agent-context`, `docyrus apps ai-tools`, `docyrus messaging email send`, `docyrus connect`, `docyrus discover`, `docyrus auth`, `docyrus env`, `docyrus browser`, `docyrus knowledge`, `docyrus project-plan`, `docyrus release`, `docyrus server`, `docyrus tui`, or shell-based Docyrus workflows.
---

# Docyrus CLI

Guide for using the `docyrus` CLI (`@docyrus/docyrus`) to interact with the Docyrus platform from the terminal.

## Command Overview

| Command | Description |
|---------|-------------|
| `docyrus` | Show active environment, current auth context, and help summary |
| `docyrus env list` / `env use` / `env which` | Manage named environments and inspect resolved settings scope |
| `docyrus auth login` / `set-tokens` | Authenticate via OAuth2 device flow or manual tokens |
| `docyrus auth logout` / `who` / `tenant` | Logout the active account / show active user / show the current tenant record |
| `docyrus auth accounts list` / `use` | Manage saved user accounts |
| `docyrus auth tenants list` / `use` | Manage saved tenants for a user |
| `docyrus auth github` / `sandbox` / `sso-session` / `git-credential` | Sandbox/CI token helpers |
| `docyrus apps list` | List apps from `/v1/apps` |
| `docyrus apps update` / `delete` / `restore` / `permanent-delete` | Mutate apps via `/v1/dev/apps/:appId` |
| `docyrus apps set-agent-context` | Set an app's AI agent context |
| `docyrus apps ai-tools ...` | CRUD app-scoped AI tools |
| `docyrus ds get` / `list` | Read data source metadata and query records |
| `docyrus ds create` / `update` / `delete` | Mutate records, including bulk create/update |
| `docyrus ds comments create` / `files upload` | Add a record comment / upload a record file attachment |
| `docyrus studio ...` | CRUD dev app data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX templates, email templates; search fields/enums/enum-sets |
| `docyrus automation ...` | CRUD automations, triggers, and action nodes for an app |
| `docyrus agent ...` | CRUD custom AI agents and their sub-resources (models, tools, data-sources, docs, mcps, connections, dynamic-contexts, tasks, recurring-tasks, workflow-steps, deployments, workflow-jobs) |
| `docyrus messaging accounts` / `email send` | List tenant email accounts / send transactional email |
| `docyrus connect ...` | Discover connectors, inspect actions, send provider-auth requests, run actions |
| `docyrus discover ...` | Download and explore the tenant OpenAPI spec |
| `docyrus curl` | Send arbitrary API requests |
| `docyrus docy "<prompt>"` | Chat with the platform's main AI agent |
| `docyrus opsy` / `cody` (`coder`) | Launch the pi Cowork / Coding agents (interactive TUI or one-shot) |
| `docyrus server` | Start the HTTP server bridging a pi agent to AI SDK `useChat` |
| `docyrus browser ...` | Browser automation (local Chrome or remote Cloudflare) |
| `docyrus knowledge ...` | Repo knowledge-graph search, audit, and maintenance |
| `docyrus project-plan ...` | Repo-tracked project plan graph (phases, features, tasks) |
| `docyrus release ...` | Version bump, changelog, and release record creation |
| `docyrus tui` | Launch the OpenTUI terminal UI (requires Bun) |

**See [references/cli-manifest.md](references/cli-manifest.md) for the complete command reference with flags and arguments.**

> **Flag forms:** `--help` prints flags in kebab-case (`--app-slug`, `--from-file`), but the parser also accepts the camelCase schema keys (`--appSlug`, `--fromFile`). This guide uses the camelCase form; both work.

## Common Workflows

### Settings Scope

By default, `docyrus` stores settings in a project-local `.docyrus/` folder in the current working directory.

- Local default: `./.docyrus/`
- Global override: `~/.docyrus/` via `-g` or `--global`
- Tenant OpenAPI cache: `<settings-root>/tenans/<tenantId>/openapi.json`

```bash
# Local project settings (default)
docyrus auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631"

# Force global settings for this run
docyrus -g auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631"

# Inspect which scope/environment is active for this folder
docyrus env which --json
```

`env which` reports `local`/`global` scope, the resolved `settingsRoot`, `configFilePath`, `authFilePath`, `cwd`, and whether a local `.docyrus/` directory exists.

### Environments

The CLI does not use `API_BASE_URL`. It uses saved named environments:

- `live` (`prod` alias) -> `https://api.docyrus.com`
- `beta` -> `https://beta-api.docyrus.com`
- `alpha` -> `https://alpha-api.docyrus.com`
- `dev` (`local-development` alias) -> `https://localhost:3366`

```bash
docyrus
docyrus env list --json
docyrus env use beta --json
```

Running `docyrus` without a subcommand returns the active environment, help summary, and current auth `context`.

### Authentication

Device flow login:

```bash
docyrus auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631" --json
```

Manual token login (`auth login` or `auth set-tokens`):

```bash
docyrus auth login \
  --accessToken "<access-token>" \
  --refreshToken "<optional-refresh-token>" \
  --clientId "<optional-client-id>" \
  --json

docyrus auth set-tokens --accessToken "<access-token>" --refreshToken "<refresh-token>" --json
```

Rules:

- `--refreshToken` requires `--accessToken`
- if local login omits `--clientId`, the CLI falls back to the saved global client ID when available
- client ID resolution order: `--clientId` -> `DOCYRUS_API_CLIENT_ID` -> saved local config -> saved global config -> `manual-token` (manual auth only)
- default scopes are hardcoded: `openid email profile offline_access ReadWrite.All Architect.ReadWrite.All Automations.Run Reports.Run.CustomQuery Messaging.Email.Send Messaging.Sms.Send Messaging.Whatsapp.Send MCP.Connect`

Multi-account and multi-tenant workflows:

```bash
docyrus auth accounts list --json
docyrus auth accounts use --userId "<user-id>" --json
docyrus auth tenants list --userId "<user-id>" --json
docyrus auth tenants use 1002 --json
docyrus auth tenants use "8d130f7a-4bc4-4be6-a05b-0f8f1b2d93e9" --userId "<user-id>" --json
docyrus auth who --json
docyrus auth tenant --json
```

`auth tenants use` takes a positional tenant selector. Numeric -> `tenantNo`; otherwise it must be a UUID tenant ID.

`auth tenant` (singular) returns the active tenant record from `GET /v1/tenant/current` (id, no, name, account status, product/subscription refs, seats, billing, trial/subscription dates, onboarding status). It is a read-only passthrough like `auth who`; don't confuse it with the `auth tenants` (plural) account-management group.

Sandbox / CI token helpers (`auth sandbox`, `auth github`, `auth sso-session`, `auth git-credential`) inject fresh tokens into a running sandbox app, mint repo-scoped GitHub tokens, or create short-lived SSO sessions for headless browsers. They default `--appId` to `DOCYRUS_SANDBOX_APP_ID` and are mostly used by the sandbox runtime rather than by hand.

### Successful Result Shape

Every successful command injects a top-level `context` field:

```json
{
  "data": {},
  "context": {
    "email": "user@example.com",
    "tenantName": "Acme",
    "tenantNo": 1002,
    "tenantDisplay": "Acme (1002)"
  }
}
```

If there is no active session, `context` is `null`.

### Discover API and Entities

Discover commands require an active session. Commands other than `discover api` auto-download the OpenAPI spec if it is missing locally.

```bash
docyrus discover api --json
docyrus discover namespaces --json
docyrus discover path /v1/users --json
docyrus discover endpoint /v1/users/me --json
docyrus discover endpoint [PUT]/v1/users/me/photo --json
docyrus discover entity UserEntity --json
docyrus discover search users,UserEntity --json
```

### Discover Data Sources

```bash
docyrus apps list --json
docyrus ds get crm contacts --json
```

### Query Records (`ds list`)

Basic listing:

```bash
docyrus ds list crm contacts --columns "name, email, phone" --limit 20
```

With filters:

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

Keyword filter and date shortcut:

```bash
docyrus ds list crm tasks --filterKeyword "renewal"
docyrus ds list crm tasks --filters '{"rules":[{"field":"created_on","operator":"this_month"}]}'
```

`ds list` also supports the full query engine: `--formulas`, `--calculations`, `--groupSummaries`, `--pivot`, `--childQueries`, `--expand`, `--distinctColumns`, and `--collapseRows`.

**See [references/list-query-examples.md](references/list-query-examples.md) for columns, filters, sorting, pagination, and advanced (formulas/calculations/pivot/child-query) examples.**

### Record Mutations

Create / update / delete:

```bash
docyrus ds create crm contacts --data '{"name":"Jane Doe","email":"jane@example.com"}'
docyrus ds update crm contacts <recordId> --data '{"phone":"+1234567890"}'
docyrus ds delete crm contacts <recordId>
```

Batch and file input:

```bash
docyrus ds create crm contacts --data '[{"name":"A"},{"name":"B"}]' --json
docyrus ds update crm contacts --data '[{"id":"1","phone":"+111"},{"id":"2","phone":"+222"}]' --json
docyrus ds create crm contacts --from-file ./contacts-create.csv --json
docyrus ds update crm contacts <recordId> --from-file ./contact-update.json --json
```

Array payloads route to bulk endpoints and are limited to 50 items per request.

### Record Comments and File Attachments

```bash
# Add a comment to a record
docyrus ds comments create crm contacts <recordId> --message "Followed up by phone" --json
docyrus ds comments create crm contacts <recordId> --from-file ./comment.json --json

# Upload a file attachment to a record (multipart/form-data)
docyrus ds files upload crm contacts <recordId> --file ./contract.pdf --json
docyrus ds files upload crm contacts <recordId> --file ./logo.png --publicFile --json
```

Comment accepts either `--message` or a full DTO via `--data`/`--from-file`. Uploads infer content type from the file extension unless `--contentType` is set; `--publicFile` stores in the public tenant bucket.

### Studio Schema CRUD (`studio`)

Use `studio` for developer-facing schema operations: data sources, fields, enums, saved views, forms, webforms, and export templates. Most data-source-scoped commands accept either `--appId`/`--appSlug` and either `--dataSourceId`/`--dataSourceSlug`, and the CLI resolves whichever side you did not pass.

```bash
# Data sources
docyrus studio list-data-sources --appSlug crm --expand fields --json
docyrus studio get-data-source --appSlug crm --dataSourceSlug contacts --json
docyrus studio create-data-source --appSlug crm --title "Contacts" --name "contacts" --slug "contacts" --json
docyrus studio update-data-source --appId <appId> --dataSourceId <dataSourceId> --data '{"title":"Contacts v2"}' --json
docyrus studio delete-data-source --appId <appId> --dataSourceSlug contacts --json
docyrus studio restore-data-source --appId <appId> --dataSourceId <dataSourceId> --json
docyrus studio permanent-delete-data-source --appId <appId> --dataSourceId <dataSourceId> --json
docyrus studio bulk-create-data-sources --appId <appId> --from-file ./data-sources.json --json

# Fields
docyrus studio list-fields --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-field --appSlug crm --dataSourceSlug contacts --fieldSlug email --json
docyrus studio create-field --appId <appId> --dataSourceId <dataSourceId> --name "Email" --slug "email" --type "text" --json
docyrus studio update-field --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '{"name":"Primary Email"}' --json
docyrus studio delete-field --appId <appId> --dataSourceId <dataSourceId> --fieldSlug email --json
docyrus studio create-fields-batch --appId <appId> --dataSourceId <dataSourceId> --data '[{"name":"Status","slug":"status","type":"text"}]' --json
docyrus studio update-fields-batch --appId <appId> --dataSourceId <dataSourceId> --from-file ./fields-update.json --json
docyrus studio delete-fields-batch --appId <appId> --dataSourceId <dataSourceId> --data '["field-1","field-2"]' --json

# Enums
docyrus studio list-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --json
docyrus studio create-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '[{"name":"Open","sortOrder":1}]' --json
docyrus studio update-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --from-file ./enums-update.json --json
docyrus studio delete-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '["enum-1","enum-2"]' --json

# Cross-data-source search (tenant-wide, paged) — useful for discovery/refactors
docyrus studio search-fields --keyword "email" --type "text,email" --json
docyrus studio search-enums --dataSourceId <dataSourceId> --json
docyrus studio search-enum-sets --limit 50 --json

# Data views (/v1/apps/:appSlug/data-sources/:dataSourceSlug/views)
docyrus studio list-data-views --appSlug crm --dataSourceSlug contacts --json
docyrus studio create-data-view --appSlug crm --dataSourceSlug contacts --name "Active customers" \
  --filters '{"rules":[{"field":"status","operator":"=","value":"active"}]}' --isDefault --json
docyrus studio update-data-view --appSlug crm --dataSourceSlug contacts --viewId <viewId> --data '{"name":"Renamed view"}' --json
docyrus studio delete-data-view --appSlug crm --dataSourceSlug contacts --viewId <viewId> --json

# Forms (/v1/apps/:appSlug/data-sources/:dataSourceSlug/forms)
docyrus studio create-form --appSlug crm --dataSourceSlug contacts --name "Lead intake" --title "New lead" --json
docyrus studio update-form --appSlug crm --dataSourceSlug contacts --formId <formId> --data '{"title":"Renamed"}' --json

# Webforms (/v1/dev/webforms)
docyrus studio create-webform --appSlug crm --dataSourceSlug contacts --from-file ./contact-webform.json --json
docyrus studio update-webform --webformId <webformId> --data '{"name":"Renamed"}' --json

# HTML / PDF / DOCX export templates (/v1/dev/html-templates)
docyrus studio create-html-template --appSlug crm --dataSourceSlug contacts \
  --name "Invoice" --sourceType pdf --pageFormat A4 --pageOrientation portrait \
  --body "<h1>{{ company.name }}</h1>" --isDefault --json

# Email templates (/v1/dev/email-templates)
docyrus studio create-email-template --name "Welcome" --subject "Hello {{ user.name }}" --body "<p>Hi</p>" --json
```

### App Management (`apps`)

`apps list` uses `/v1/apps`; mutations route through `/v1/dev/apps/:appId`.

```bash
docyrus apps list --json
docyrus apps update --appSlug crm --description "Customer CRM" --color "#2563eb" --json
docyrus apps delete --appId <appId> --json
docyrus apps restore --appId <appId> --json
docyrus apps permanent-delete --appId <appId> --json
```

`apps update` accepts camelCase convenience flags (`--name`, `--slug`, `--description`, `--icon`, `--color`, `--status`, `--betaUrl`, `--chromeExtensionPath`, `--mobileVersionPath`, `--agentContext`, `--routePath`) merged over `--data`/`--from-file`. Store fields and array/object values must go through `--data`/`--from-file`. `--status` is one of `active`, `design`, `development`, `draft`, `inactive`.

#### App AI Agent Context

Set the freeform AI agent context an app injects into its agents (PATCH `/v1/dev/apps/:appId` `agent_context`):

```bash
docyrus apps set-agent-context --appSlug crm --value "This app manages B2B sales pipelines." --json
docyrus apps set-agent-context --appSlug crm --from-file ./agent-context.md --json
docyrus apps set-agent-context --appSlug crm --clear --json
```

Provide exactly one of `--value` (inline), `--from-file` (raw text/markdown), or `--clear`.

#### App-Scoped AI Tools (`apps ai-tools`)

Full CRUD over `tenant_ai_tool` rows scoped to an app (`/v1/dev/apps/:appId/ai-tools`):

```bash
docyrus apps ai-tools list --appSlug crm --json
docyrus apps ai-tools get --appSlug crm --toolId <toolId> --json
docyrus apps ai-tools create --appSlug crm --name "Lookup contact" --key lookup_contact \
  --type data-source-query --dataSourceQueryDataSourceId <dataSourceId> --json
docyrus apps ai-tools update --appSlug crm --toolId <toolId> --description "Updated" --json
docyrus apps ai-tools delete --appSlug crm --toolId <toolId> --json
```

`--name` and `--key` are required on create. Convenience flags cover the common columns; complex/JSON fields (`--inputJsonSchema`, `--outputJsonSchema`, `--customQueryFilters`, `--dataSourceQueryColumns`, `--avatar`, etc.) are parsed from JSON, and the long tail can go through `--data`/`--from-file`.

### Automations (`automation`)

Manage automations, their triggers, and action nodes for an app. All commands route through `/v1/dev/apps/:appId/automations`. App is resolved with `--appId` or `--appSlug`. Automations and nodes have IDs only (no slugs).

```bash
# Automation CRUD
docyrus automation list --appSlug crm --json
docyrus automation get --appSlug crm --automationId <automationId> --json
docyrus automation create --appSlug crm \
  --name "Notify on new deal" --triggerType recordCreated \
  --sourceDataSourceId <dataSourceId> --status 1 --json
docyrus automation update --appSlug crm --automationId <automationId> \
  --data '{"name":"Renamed automation","status":1}' --json
docyrus automation delete --appSlug crm --automationId <automationId> --json

# Triggers (typed create/update, type-independent delete)
docyrus automation list-triggers --appSlug crm --automationId <automationId> --json
docyrus automation create-trigger --appSlug crm --automationId <automationId> \
  --type record-modified --sourceDataSourceId <dataSourceId> \
  --modifiedColumns "status,stage" --modifiedColumnsCondition any --json
docyrus automation create-trigger --appSlug crm --automationId <automationId> \
  --type recurrence --recurrenceFrequency day --recurrenceInterval 1 --recurrenceRunAt "09:00" --json
docyrus automation create-trigger --appSlug crm --automationId <automationId> \
  --type app-event --dataProviderId <coreDataProviderId> --dataProviderWebhookId <webhookId> --json
docyrus automation update-trigger --appSlug crm --automationId <automationId> \
  --type webhook --triggerId <triggerId> --data '{"webhook_name":"renamed"}' --json
docyrus automation delete-trigger --appSlug crm --automationId <automationId> --triggerId <triggerId> --json

# Action nodes (typed create/update, type-independent delete)
docyrus automation list-nodes --appSlug crm --automationId <automationId> --json
docyrus automation create-node --appSlug crm --automationId <automationId> \
  --type http-request --requestMethod POST --customEndpoint "https://example.com/webhook" \
  --contentType "application/json" --from-file ./http-node.json --json
docyrus automation create-node --appSlug crm --automationId <automationId> \
  --type external-action --actionTypeId <coreActionId> --from-file ./external-action.json --json
docyrus automation update-node --appSlug crm --automationId <automationId> \
  --type send-email --nodeId <nodeId> --data '{"data":{"to":"user@example.com","subject":"Hi"}}' --json
docyrus automation create-node --appSlug crm --automationId <automationId> \
  --type wait-for --name "Wait 2 hours" --parent <previousNodeId> \
  --data '{"data":{"delayValue":2,"delayUnit":"hours"}}' --json
docyrus automation delete-node --appSlug crm --automationId <automationId> --nodeId <nodeId> --json
```

Trigger `--type` values (kebab-case URL form): `record-created`, `record-modified`, `record-deleted`, `recurrence`, `app-event`, `webhook`, `emailhook`, `webform`, `button-activation`, `manual-activation`.

Node `--type` values (kebab-case URL form): `external-action`, `send-email`, `send-notification`, `create-record`, `update-records`, `request-approval`, `request-input`, `http-request`, `data-source-query`, `custom-query`, `generate-document`, `ai-prompt`, `ai-agent`, `execute-script`, `wait-for`.

Notes:

- `automation create --triggerType` uses the camelCase form (`recordCreated`, `recordModified`, …) to match `CreateAutomationDto.trigger_type`; trigger CRUD commands use kebab-case for `--type`.
- Convenience flags are camelCase on the CLI but converted to `snake_case` in the request body. Complex nested objects (trigger `data`, node `data`, `field_mapping`, `dynamic_field_mapping`, `condition`, `input_template`, `input_transformer`, `custom_headers`, `pre_action_request`, `post_action_request`, `target_data_source_condition`, …) must be supplied via `--data`/`--from-file`. `--recurrenceWeekDays` and `--modifiedColumns` accept comma-separated values sent as arrays.
- `app-event` triggers use `--dataProviderId` and `--dataProviderWebhookId` (obtain via `docyrus connect list-connectors` / `connect get-connector <slug>`); `webform` triggers use `--webformId`.
- `create-node --type external-action` requires `--actionTypeId`; the backend validates the supplied `data` against the linked `core_action.input_json_schema`.
- `create-node --type wait-for` takes no flat flags beyond the common base. Set the delay inside `data`: either `delaySeconds` (integer, capped at 30 days / 2_592_000) or the `delayValue` + `delayUnit` (`seconds`/`minutes`/`hours`/`days`) pair.

### Custom AI Agents (`agent`)

CRUD for dev-app custom agents and their sub-resources, routed through `/v1/dev/apps/:appId/agents...`. App is resolved with `--appId`/`--appSlug`; the parent agent is `--agentId`; an individual sub-resource row is `--id`.

```bash
# Agent resource
docyrus agent list --appSlug crm --json
docyrus agent get --appSlug crm --agentId <agentId> --json
docyrus agent create --appSlug crm --skillName "sales_copilot" --name "Sales Copilot" \
  --description "Assists with deal updates" --defaultAiModelId <modelId> --supportTools --json
docyrus agent update --appSlug crm --agentId <agentId> --welcomeMessage "Hi!" --json
docyrus agent delete --appSlug crm --agentId <agentId> --json
docyrus agent upload --appSlug crm --agentId <agentId> --column avatar --file ./avatar.png --json

# Sub-resources (each: list / get / create / update / delete, unless noted)
docyrus agent models   create --appSlug crm --agentId <agentId> --from-file ./model.json --json
docyrus agent tools    create --appSlug crm --agentId <agentId> --coreAiToolId <toolId> --json
docyrus agent data-sources create --appSlug crm --agentId <agentId> --tenantDataSourceId <dataSourceId> --json
docyrus agent docs     create --appSlug crm --agentId <agentId> --from-file ./doc.json --json
docyrus agent mcps     create --appSlug crm --agentId <agentId> --from-file ./mcp.json --json
docyrus agent connections create --appSlug crm --agentId <agentId> \
  --connectedAiAgentId <agentId> --connectionType handoff --json
docyrus agent deployments create --appSlug crm --agentId <agentId> --from-file ./deployment.json --json
docyrus agent workflow-jobs list --appSlug crm --agentId <agentId> --json   # read-only (+ get / traces / delete)
```

Sub-resource groups: `models`, `tools`, `data-sources`, `docs`, `mcps`, `connections`, `dynamic-contexts`, `tasks`, `recurring-tasks`, `workflow-steps`, `deployments`, `deployment-tools` (nested under a deployment via `--deploymentId`; no `get`), `deployment-data-sources` (same), and read-only `workflow-jobs` (`list`/`get`/`traces`/`delete`).

Notes:

- `agent create` requires `--skillName` (the only field required by `CreateAgentDto`); other sub-resource required fields are enforced by the backend.
- Convenience flags map 1:1 onto the matching `Create*Dto`/`Update*Dto` `snake_case` keys; the agent DTO is large, so its flags are a curated subset — the long tail (`compaction_*`, `output_*`, `prompt_*`, …) and any nested arrays go through `--data`/`--from-file`.
- `createOnly` fields (e.g. `--tenantDataSourceId`, agent-tool `--coreAiToolId`) appear only on `create`; `updateOnly` fields (e.g. `--archived`) only on `update`. incur rejects unknown flags, so passing the wrong one fails before any request.
- `delete` returns a `{ deleted: true, id }` envelope.

### Messaging (`messaging`)

List tenant email accounts and send transactional emails. Routes through `/v1/messaging/email/*`. Requires the `Messaging.Email.Send` OAuth2 scope.

```bash
docyrus messaging accounts --json
docyrus messaging email send \
  --accountId <accountUuid> \
  --to "ops@example.com,sales@example.com" \
  --subject "Daily summary" \
  --body "<p>Hello</p>" --json

docyrus messaging email send --accountId <accountUuid> \
  --to "user@example.com" --cc "manager@example.com" --replyTo "support@example.com" --sendAsUser \
  --subject "Update" --body "<p>...</p>" --json

docyrus messaging email send --accountId <accountUuid> --from-file ./send.json --json
```

Limits: up to 50 recipients per `to`/`cc`/`bcc`/`replyTo`, subject max 998 chars, body max 1 000 000 chars, up to 10 attachments. Attachment `filePath` references a tenant-scoped storage path. The response contains `messageId`, `provider`, `accepted`, and `rejected`.

### Connectors and Actions (`connect`)

Connectors are external integration providers (e.g. Meta WhatsApp, Microsoft Graph, Salesforce). Use `connect` to find connectors, inspect data sources/actions, check connection status, send provider-auth requests, and run actions.

```bash
# Discovery
docyrus connect list-connectors --q whatsapp --json
docyrus connect get-connector meta-whatsapp --json
docyrus connect get-action meta-whatsapp sendWhatsappMessage --json
docyrus connect list-connections meta-whatsapp --json

# Send requests through connector auth (OAuth tokens, base URL, API keys)
docyrus connect curl meta-whatsapp "433457363182570/phone_numbers" \
  -d '{"fields":"id,display_phone_number,verified_name"}' --json
docyrus connect curl meta-whatsapp "418088118057836/messages" -X POST \
  -d '{"messaging_product":"whatsapp","to":"905551234567","type":"template","template":{"name":"sample_template","language":{"code":"en_US"}}}' \
  --contentType "application/json" --json

# Run a predefined action (POST /v1/apps/:appSlug/actions/:actionKey/run)
docyrus connect run-action base sendWhatsappMessage --params '{"to":"905551234567","templateName":"hello_world"}' --json
docyrus connect run-action base sendWhatsappMessage --params '{"to":"905551234567"}' --dryRun --json
```

Aliases: `connect curl` — `-X` (method), `-d` (data), `-c` (connectionId). `connect run-action` — `-p` (params), `-c` (connectionId), `-n` (dryRun). `connect curl` data is sent as body for POST/PUT/PATCH and as query params for GET; `--headers` can override the Authorization header.

### Arbitrary API Calls

```bash
docyrus curl /v1/users/me
docyrus curl /v1/apps -X GET --format json
docyrus curl /v1/some/endpoint -X POST -d '{"key":"value"}'
```

## AI Agents and Dev Tooling

Beyond data operations, the CLI bundles the pi agent runtime and repo dev tooling. These are largely interactive or sandbox-runtime commands.

### Chat and pi Agents

```bash
docyrus docy "Summarize open deals this month"       # one prompt to the platform's main AI agent
docyrus opsy                                          # launch the Cowork Agent (interactive TUI)
docyrus cody "fix the failing test in ds list"       # launch the Coding Agent (coder is an alias)
docyrus cody --print --mode json "list TODOs"        # one-shot, machine output
```

`docy` sends a single prompt (`--agentId`/`--deploymentId` optional); it renders markdown in a TTY and structured output with `--json`/`--verbose`/`--format`. `opsy`/`cody`/`coder` open the pi TUI by default, or run one-shot with `--print`; they accept `--provider`, `--model`, `--thinking`, `--continue`, `--resume`, `--session`, `--apiKey`, etc.

### Agent Server

```bash
docyrus server --profile coder --port 3111
```

Starts an HTTP server bridging a pi agent to the AI SDK `useChat` protocol (SSE chat stream with reconnect/resume). Flags include `--auth` (require a bearer token), `--sandbox` (remote Cloudflare browser mode), and `--desktop` (expose `docyrus_browser_*` tools).

### Browser Automation

```bash
docyrus browser start --profile
docyrus browser nav https://example.com
docyrus browser snapshot                 # compact element refs (@e1 …) for interaction
docyrus browser click @e3
docyrus browser fill @e5 "hello"
docyrus browser screenshot --full
docyrus browser content https://example.com   # extract readable markdown
docyrus browser close
```

Runs against local Chrome (`:9222`) or a remote Cloudflare session. Other subcommands: `eval`, `select`, `wait`, `tabs`, `cookies`, `console`, `network`, `info`, `devtools`, `run-script`.

### Repo Knowledge Graph and Project Plan

```bash
docyrus knowledge search "how does auth refresh work"
docyrus knowledge section <sectionId>
docyrus knowledge check                      # validate links/backlinks
docyrus project-plan show
docyrus project-plan list-tasks --status in_progress --limit 5
docyrus project-plan set-task-status --taskId <id> --status done
```

`knowledge` manages the repo's `docyrus/knowledge` graph (search, audit, refresh, pre-commit gates). `project-plan` manages a repo-tracked plan graph of phases, features, and tasks (token-efficient list/find/upsert/status commands). Both are local dev-workflow tools used by the pi agents.

### Releases

```bash
docyrus release status
docyrus release new-version --bump minor --json
docyrus release new-version --version 1.2.0 --dryRun --json
```

Bumps the version, generates a changelog, optionally tags, creates a GitHub release, and records a DB release row (skip with `--skip*` flags).

### Terminal UI

```bash
docyrus tui
```

Launches the OpenTUI interface (requires Bun). It reuses the existing CLI command graph.

## Key Rules

- Settings are project-local by default in `./.docyrus/`; use `-g`/`--global` for `~/.docyrus/`. `env which` shows the resolved scope.
- The CLI uses named environments, not `API_BASE_URL`.
- Flags accept both kebab-case (`--app-slug`, as `--help` prints) and camelCase (`--appSlug`, the schema key). `--from-file` reads JSON (and CSV for `ds`).
- Default login scopes are `openid email profile offline_access ReadWrite.All Architect.ReadWrite.All Automations.Run Reports.Run.CustomQuery Messaging.Email.Send Messaging.Sms.Send Messaging.Whatsapp.Send MCP.Connect`.
- Successful responses inject `context` (`email`, `tenantName`, `tenantNo`, `tenantDisplay`); it is `null` with no active session.
- `apps list` uses `/v1/apps`; `apps` mutations and `set-agent-context`/`ai-tools` use `/v1/dev/apps/:appId`.
- `ds` commands use `appSlug` and `dataSourceSlug`. `ds create`/`update` accept `--data` JSON or `--from-file` (`.json`/`.csv`), not both. Array payloads use bulk endpoints (max 50); bulk update requires `id` in each item and no positional `<recordId>`.
- `ds list` supports `--columns` (relation `()`, spread `...`, alias `:`, function `@`), `--filters` (JSON filter group), `--filterKeyword`, `--orderBy`, `--formulas`, `--calculations`, `--groupSummaries`, `--pivot`, `--childQueries`, `--expand`, `--distinctColumns`, `--collapseRows`, `--limit`, `--offset`, `--fullCount`. Related-field filters use `rel_<relation_slug>/<field_slug>`.
- `--format` supports `toon`, `json`, `yaml`, `md`, and `jsonl`.
- Studio selectors are exclusive pairs: exactly one of `--appId|--appSlug`, `--dataSourceId|--dataSourceSlug`, and `--fieldId|--fieldSlug` as required. Studio write commands accept `--data` or `--from-file` (JSON only); explicit flags override overlapping JSON keys. `restore-data-source`/`permanent-delete-data-source` require `--dataSourceId`.
- Studio data-view/form commands route through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/...`; webform/html-template/email-template commands route through `/v1/dev/webforms`, `/v1/dev/html-templates`, `/v1/dev/email-templates` (CRUD by `--webformId`/`--templateId`).
- `automation` routes through `/v1/dev/apps/:appId/automations`; trigger/node `create`/`update` use typed URLs (`/triggers/<type>`, `/nodes/<type>`), but `delete-trigger`/`delete-node` use the type-independent route. CLI flags are camelCase, converted to `snake_case`; nested objects go through `--data`/`--from-file`. `automation create --triggerType` is camelCase; trigger/node `--type` is kebab-case.
- `agent` commands route through `/v1/dev/apps/:appId/agents...`; the parent agent is `--agentId` and a sub-resource row is `--id`. `agent create` requires `--skillName`. `createOnly`/`updateOnly` flags are filtered per command.
- `messaging` routes through `/v1/messaging/email/*` and needs `Messaging.Email.Send`; `accounts` never returns credentials.
- `connect` uses the `/v1/connectors` API; `connect curl` sends through provider auth; `connect run-action` posts to `/v1/apps/:appSlug/actions/:actionKey/run`.
- `tui` requires Bun; `docy`/`opsy`/`cody`/`coder`/`server` run the pi agent runtime; `knowledge`/`project-plan`/`release` are repo dev-workflow tools.

## References

- **[CLI Manifest](references/cli-manifest.md)** — Complete command reference with flags, arguments, and command notes.
- **[List Query Examples](references/list-query-examples.md)** — Practical `ds list` examples covering columns, filters, sorting, pagination, and advanced queries.
