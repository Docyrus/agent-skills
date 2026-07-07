# Developer Tools

## OpenAPI Specification

- Auto-generated, tenant-specific OpenAPI specs reflecting all configured data sources and fields
- Regeneratable on-demand after schema changes
- Powers client SDK generation and API discovery

## MCP Server (Model Context Protocol)

- Built-in MCP server exposing platform capabilities as AI-consumable tools
- Discovery, CRUD, querying, enum management, custom query execution, and JSONata evaluation — all accessible via MCP transport

## CLI

- Full-featured CLI (`@docyrus/docyrus`) for terminal and AI agent use
- Data & schema: data operations (`ds`, full query engine + comments + file attachments), read-only logical SQL with schema discovery (`dsql`: `query` + `schema app`/`data-source`/`data-sources`), schema management (`studio`: data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX export templates, email templates, plus tenant-wide field/enum search), automation management (`automation`: automation, trigger, and action node CRUD)
- App & AI configuration: app management including web-app scaffolding (`apps create` — techstack starter / empty repo / template remix, with `--local-development` clone + `.env`; `apps clone`; `apps templates`), AI agent context, and app-scoped AI tools (`apps`), custom AI agent builder with full sub-resource CRUD — models, tools, data sources, docs, MCPs, connections, tasks, recurring tasks, workflow steps, deployments, and workflow jobs (`agent`)
- Messaging, connectors & discovery: tenant email account discovery and transactional send (`messaging`), connector discovery, provider-auth requests, and action runs (`connect`), OpenAPI discovery (`discover`), direct API requests (`curl`)
- Account management (`account`): tenant brand CRUD (`account brands`) — visual identity, typography, voice, and content guidelines (`tenant_brand`), including website-scrape import; current-user profile get/update (`account user-profile`, `/users/me`); and the tenant's regional/formatting preferences get/update (`account tenant-preferences`, `tenant_pref`, tenant-admin only)
- Agent runtime & dev tooling: platform AI chat (`docy`), pi Cowork/Coding agents (`opsy`/`cody`/`coder`), agent bridge server (`server`), browser automation (`browser`), repo knowledge graph (`knowledge`), project plan graph (`project-plan`), and release management (`release`)
- Multi-account, multi-tenant session management; named environments (not `API_BASE_URL`); OpenAPI discovery with caching and fallback generation; interactive TUI mode

For the full CLI command index, see the **docyrus-cli-app** skill (and `docyrus <cmd> --help` for flags).

## Client Libraries
  
- REST API client (`@docyrus/api-client`) with OAuth2 support, interceptors, streaming, and file operations
- React authentication provider (`@docyrus/signin`) with standalone OAuth2 PKCE and iframe postMessage modes, automatic current-user fetch from `/v1/users/me`, and `hasRole` / `hasPermission` / `refreshUser` helpers
- Framework-agnostic authorization helpers are also available from `@docyrus/signin/core`
- Auto-generated collection hooks from OpenAPI specs for data fetching integration
