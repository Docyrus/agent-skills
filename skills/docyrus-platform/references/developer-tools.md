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
- Commands: environment management, authentication, data operations, record comments, record file attachments, schema management (studio: data sources, fields, enums, data views, forms, webforms, HTML/PDF/DOCX export templates, email templates), automation management (automation, trigger, and action node CRUD), tenant email account discovery and transactional email send (messaging), app management, API discovery, connector discovery and provider-auth requests, AI chat, and direct API requests
- Multi-account, multi-tenant session management
- OpenAPI discovery with caching and fallback generation
- Interactive TUI mode

For full CLI command reference, see [docyrus-cli-usage.md](docyrus-cli-usage.md).

## Client Libraries
  
- REST API client (`@docyrus/api-client`) with OAuth2 support, interceptors, streaming, and file operations
- React authentication provider (`@docyrus/signin`) with standalone OAuth2 PKCE and iframe postMessage modes, automatic current-user fetch from `/v1/users/me`, and `hasRole` / `hasPermission` / `refreshUser` helpers
- Framework-agnostic authorization helpers are also available from `@docyrus/signin/core`
- Auto-generated collection hooks from OpenAPI specs for data fetching integration
