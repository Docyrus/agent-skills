---
name: docyrus-platform
description: Architecture and building-blocks reference for the Docyrus platform — what the platform is made of (apps, data sources, fields, enumerations, custom queries, automations) and how the pieces fit together. Use when you need conceptual context about how Docyrus works, what its building blocks and capabilities are, or to decide which dedicated skill to use for a task. This is the conceptual map and router; hands-on how-to lives in the dedicated design / CLI / API skills it points to (data sources, automations, agents, AI tools, templates, webforms, ACL, querying, the CLI, and the REST API).
---

# Docyrus Platform

Docyrus is an AI-native Backend Platform as a Service (BPaaS) for building B2B web/mobile apps, client portals, internal tools, AI agents, chatbots, and integrations — without building backend infrastructure from scratch. This skill is the **architecture map**: the building blocks, how they fit, and which dedicated skill owns each how-to.

## Core Building Blocks

The platform is composed of building blocks that compose into apps:

- **Apps** — Top-level containers that group data sources, automations, agents, and custom queries into a deployable unit.
- **Data Sources** — Structured collections of records with typed schemas. Four types: simple, advanced, external (connected DBs/APIs), and system (pre-built). Every data source — internal or external — is exposed through one unified CRUD endpoint.
- **Fields** — 45+ field types defining each data source's schema: text, numbers, dates, selections, relations, files, formulas, nested data, and more.
- **Enumerations** — Reusable option sets for selection fields (color, icon, ordering).
- **Custom Queries** — SQL-based analytics templates with variable interpolation, runtime filters, and multi-database targeting.
- **Automations** — Event-driven workflows: triggers (record changes, schedule, webhook, email, webform, button) → action-node chains (email, notifications, record ops, HTTP, AI, scripts).

Detailed building-block specifications: [references/core-building-blocks.md](references/core-building-blocks.md).

## Which skill for which task

This skill holds the *concepts*. For doing the work, use the dedicated skill:

| To… | Use skill |
|---|---|
| Design a data source (schema, fields, relations, enums, access) | **docyrus-data-source-design** |
| Build an automation (triggers + action-node graph) | **docyrus-automation-design** |
| Build a custom AI agent + its sub-resources | **docyrus-agent-design** |
| Author AI tools the system base agent ("Docy") calls | **docyrus-app-ai-tools** |
| Design an email template | **docyrus-email-template-design** |
| Design an HTML/PDF print/report template | **docyrus-print-pdf-template-design** |
| Build a public webform that collects submissions into records | **docyrus-webform-design** |
| Design a data source's record create/edit/view form (grid, sections, conditional rules) | **docyrus-dynamic-form-design** |
| Configure tenant ACL (roles, hierarchy units, role rules) | **docyrus-acl-design** |
| Discover & call third-party integrations/connectors (`docyrus connect`) | **docyrus-integrations-and-connectors** |
| Query/CRUD data, write filters/formulas, build query payloads | **docyrus-api-dev** |
| Write DSQL (logical SQL) queries, or natural-language `dsql ask` | **docyrus-dsql-query-design** |
| Drive everything from the terminal (auth, ds, studio, automation, agent, …) | **docyrus-cli-app** |
| Build a React app on Docyrus (collections, grids, forms) | **docyrus-app-dev-react**, **docyrus-data-grid-page-design**, **docyrus-record-detail-form-design** |

## Capability areas (concepts)

### Querying & Data Operations

Unified query engine: column selection, 50+ filter operators, aggregations, formulas, pivots, child queries, full-text search; full CRUD with bulk ops, comments, and attachments; plus a read-only logical SQL endpoint (DSQL) over `appSlug.dataSourceSlug` tables. Concept overview: [references/querying-and-data-operations.md](references/querying-and-data-operations.md).
→ Query payloads & formulas: **docyrus-api-dev**. DSQL (logical SQL, `dsql ask`): **docyrus-dsql-query-design**. Terminal querying: **docyrus-cli-app**.

### AI Capabilities

AI agent builder with tool binding, knowledge bases, MCP servers, 18+ model providers, agent chaining, task scheduling, persistent memory, chat integrations, and evaluation metrics. Concept overview: [references/ai-capabilities.md](references/ai-capabilities.md).
→ Build agents: **docyrus-agent-design**. Author the tools Docy calls: **docyrus-app-ai-tools**.

### Automation & Workflows

Event-driven automation engine: trigger types and action-node types, conditional flows, action chains, archiving. Concept overview: [references/automation-and-workflows.md](references/automation-and-workflows.md).
→ Build automations: **docyrus-automation-design**.

### Authentication & Multi-Tenancy

OAuth2 flows (PKCE, Client Credentials, Device Code), scope-based permissions, tenant isolation, role-based and record-level ACL, React auth helpers, and the client portal. Concept overview: [references/auth-and-multi-tenancy.md](references/auth-and-multi-tenancy.md).
→ Configure ACL: **docyrus-acl-design**. OAuth in app code: **docyrus-api-dev**.

### Integrations & Events

Connector framework for HTTP and SQL providers, webhook/event management, connector discovery, collaborative document editing, in-app messaging, notifications, and email. Concept overview: [references/integrations-and-events.md](references/integrations-and-events.md).
→ Discover & use connectors (`docyrus connect`): **docyrus-integrations-and-connectors**. Webhook/messaging CLI: **docyrus-cli-app**.

### Platform Services

App templates, data import/export, reporting & analytics, deployment & versioning, localization & navigation, audit logging, and billing. Concept overview: [references/platform-services.md](references/platform-services.md).
→ Public data-collection webforms are their own skill: **docyrus-webform-design**.

### Developer Tools

Auto-generated OpenAPI specs, MCP server, a full-featured CLI, REST API client libraries, a React auth provider, and auto-generated collection hooks. Concept overview: [references/developer-tools.md](references/developer-tools.md).
→ Full CLI command index: **docyrus-cli-app** (`docyrus <cmd> --help` for flags). REST client + query payloads + formulas: **docyrus-api-dev**.
