---
name: docyrus-platform
description: Comprehensive reference for Docyrus platform capabilities, building blocks, features, and API coverage. Use when the user asks about what Docyrus can do, what features are available, how the platform works, or needs context about platform capabilities to design solutions.
---

# Docyrus Platform

Docyrus is an AI-native Backend Platform as a Service (BPaaS) that enables businesses to build B2B web apps, mobile apps, client portals, internal tools, AI agents, chatbots, and integrations — without building backend infrastructure from scratch.

## Core Building Blocks

The platform is composed of six core building blocks that work together:

- **Apps** — Top-level containers that group data sources, automations, and custom queries into a deployable unit.
- **Data Sources** — Structured collections of records with defined schemas. Support simple, advanced, external (connected databases/APIs), and system (pre-built) types. Every data source — internal or external — is exposed through a single, unified CRUD endpoint.
- **Fields** — 45+ field types that define the schema of each data source, covering text, numbers, dates, selections, relations, files, formulas, nested data, and more.
- **Enumerations** — Reusable option sets for selection fields, with color, icon, and ordering support.
- **Custom Queries** — SQL-based analytics templates with dynamic variable interpolation, runtime filters, and multi-database targeting.
- **Automations** — Event-driven workflows with triggers (record changes, time-based, webhooks, buttons) and action chains (email, notifications, HTTP requests, AI prompts, record operations).

For detailed specifications of each building block, see [references/core-building-blocks.md](references/core-building-blocks.md).

## Key Feature Areas

### Querying & Data Operations

Unified query engine with column selection, 50+ filter operators, aggregations, formulas, pivots, child queries, and full-text search. Full CRUD with bulk operations, record comments, and file attachments.

See [references/querying-and-data-operations.md](references/querying-and-data-operations.md).

For complete query payload reference with examples, see [references/data-source-query-guide.md](references/data-source-query-guide.md).

For block formula design (inline expressions and subqueries), see [references/formula-design-guide-llm.md](references/formula-design-guide-llm.md).

### AI Capabilities

AI agent builder with tool binding, knowledge bases, MCP servers, 18+ model providers, agent chaining, task scheduling, persistent memory, chat integrations, and evaluation metrics.

See [references/ai-capabilities.md](references/ai-capabilities.md).

### Automation & Workflows

Event-driven automation engine with six trigger types and eleven action types. Supports conditional flows, action chains, and archiving.

See [references/automation-and-workflows.md](references/automation-and-workflows.md).

### Authentication & Multi-Tenancy

OAuth2 flows (PKCE, Client Credentials, Device Code), scope-based permissions, tenant isolation, role-based and record-level ACL, React auth-provider authorization helpers, and client portal system.

See [references/auth-and-multi-tenancy.md](references/auth-and-multi-tenancy.md).

### Integrations & Events

Connector framework for HTTP and SQL providers, webhook management, collaborative document editing, in-app messaging, notifications, and email.

See [references/integrations-and-events.md](references/integrations-and-events.md).

### Platform Services

App templates, data import/export, webforms, reporting & analytics, deployment, localization, audit logging, and billing.

See [references/platform-services.md](references/platform-services.md).

### Developer Tools

Auto-generated OpenAPI specs, MCP server, full-featured CLI, REST API client libraries, React auth provider with current-user and authorization helpers, and auto-generated collection hooks.

See [references/developer-tools.md](references/developer-tools.md).

For full CLI command reference (all commands, options, and flags), see [references/docyrus-cli-usage.md](references/docyrus-cli-usage.md).
