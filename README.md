# KIVA Agent Skills

A comprehensive library of instructions and reference materials designed for AI agents (and developers) working with the KIVACORE platform.

## Overview

This repository centralizes the knowledge and patterns required to manage the KIVACORE platform, integrate with its APIs, and develop React-based applications using KIVA as a backend.

## Installation

To add these skills to your environment, run:

```bash
npx skills add kivacore/agent-skills
```

## Available Skills

| Skill                     | Description                                                                                       | Location                                            |
| :------------------------ | :------------------------------------------------------------------------------------------------ | :-------------------------------------------------- |
| **KIVA API Dev**       | Integrate with KIVA API using `@kivacore/api-client` and `@kivacore/signin`.                     | [SKILL.md](./skills/kiva-api-dev/SKILL.md)       |
| **KIVA App Dev React** | Build KIVA React TypeScript applications end-to-end, combining auth, collections, queries, TanStack patterns, and production-grade UI implementation. | [SKILL.md](./skills/kiva-app-dev-react/SKILL.md) |
| **KIVA API Doctor**    | Post-implementation checklist to catch bugs, performance issues, and code quality problems in KIVA API code. | [SKILL.md](./skills/kiva-api-doctor/SKILL.md)    |
| **KIVA CLI App**       | Interact with the KIVACORE platform from the terminal using the `kiva` CLI for auth, data records, schema management, and API requests. | [SKILL.md](./skills/kiva-cli-app/SKILL.md)       |
| **KIVA Data Source Design** | Design, validate, and test a data source (schema, fields, relations, enums, access rules) end-to-end with `kiva studio`. | [SKILL.md](./skills/kiva-data-source-design/SKILL.md) |
| **KIVA Automation Design** | Build, validate, and test automations — triggers plus action-node graphs (conditions, field mappings, sequencing) — with `kiva automation`. | [SKILL.md](./skills/kiva-automation-design/SKILL.md) |
| **KIVA Agent Design** | Design custom AI agents and their sub-resources (tools, data sources, docs, MCPs, connections, workflow steps, deployments) with `kiva agent`. | [SKILL.md](./skills/kiva-agent-design/SKILL.md) |
| **KIVA Email Template Design** | Design, validate, and test data-bound email templates (`{{field}}` Handlebars) with `kiva studio`. | [SKILL.md](./skills/kiva-email-template-design/SKILL.md) |
| **KIVA Print PDF Template Design** | Design, validate, and test HTML/PDF export/report templates (rendered per record) with `kiva studio`. | [SKILL.md](./skills/kiva-print-pdf-template-design/SKILL.md) |
| **KIVA Webform Design** | Design public webforms that collect submissions into records, bound to a data source, with `kiva studio`. | [SKILL.md](./skills/kiva-webform-design/SKILL.md) |
| **KIVA Dynamic Form Design** | Design, save, and validate a data source's dynamic form — the record create/edit/view layout (grid, sections, required rules, conditional actions, submit validation) — with `kiva studio` form commands. | [SKILL.md](./skills/kiva-dynamic-form-design/SKILL.md) |
| **KIVA ACL Design** | Design, configure, and validate tenant ACL — roles, permissions, role queries, hierarchy units — with `kiva acl`. | [SKILL.md](./skills/kiva-acl-design/SKILL.md) |
| **KIVA DSQL Query Design** | Write, discover, and run DSQL queries against logical data sources (incl. `kiva dsql ask`). | [SKILL.md](./skills/kiva-dsql-query-design/SKILL.md) |
| **KIVA Custom Query** | CRUD, run, and consume saved custom query records — reusable, parameterized DSQL query templates — via `kiva dsql *-custom-query` and `RestApiClient.runCustomQuery`. | [SKILL.md](./skills/kiva-custom-query/SKILL.md) |
| **KIVA Integrations & Connectors** | Discover and use integration connectors — call external APIs/actions through managed provider auth — with `kiva connect`. | [SKILL.md](./skills/kiva-integrations-and-connectors/SKILL.md) |
| **KIVA App AI Tools** | Create app-scoped AI tools for the system base agent ("Docy") — `data_source_query`, `custom_query`, `secure_exec`, `client_side` — with `kiva apps ai-tools`. | [SKILL.md](./skills/kiva-app-ai-tools/SKILL.md) |
| **KIVACORE Platform** | Architecture and building-blocks reference for the KIVACORE platform, and a router to the dedicated design/CLI/API skills. | [SKILL.md](./skills/kiva-platform/SKILL.md) |
| **KIVA Tenant Brand Management** | Manage tenant brand identity records (`tenant_brand`) — colors, typography, voice, content guidelines — with `kiva account brands`, incl. the full field catalog. | [SKILL.md](./skills/kiva-tenant-brand-management/SKILL.md) |
| **KIVA Account Settings** | Read/update the current user's profile and the tenant's regional & formatting preferences with `kiva account user-profile` / `account tenant-preferences`, incl. the full field catalog. | [SKILL.md](./skills/kiva-account-settings/SKILL.md) |
| **KIVA E2E Browser Testing** | Self-test a KIVA-backed app in a real browser — sign in headlessly with `kiva auth sso-session`, drive the UI with the kiva browser tools, and read runtime errors/issues/console from `@kivacore/devtools`. | [SKILL.md](./skills/kiva-e2e-browser-testing/SKILL.md) |
| **KIVA Browser CLI** | Browser automation via the `kiva browser` CLI (persistent CDP daemon) — navigate, snapshot, click/fill, screenshot, eval, console/network inspection, in local Chrome or remote Cloudflare Browser Rendering. | [SKILL.md](./skills/kiva-browser-cli/SKILL.md) |
| **KIVA Project Planning** | Plan, structure, and track a KIVA-backed app with the repo-local project plan system (`kiva project-plan`) — phases, features, tasks, diagrams. | [SKILL.md](./skills/kiva-project-planning/SKILL.md) |

## Repository Structure

Each skill directory is organized as follows:

- `SKILL.md`: The primary instruction file and entry point for the AI agent.
- `references/`: Detailed guides for specific topics like query syntax, authentication flows, and component patterns.

## Usage for AI Agents

When tasked with work related to KIVA, AI agents should:

1. Identify the relevant skill directory within the `skills/` folder.
2. Read the `SKILL.md` file using the `view_file` tool.
3. Consult the files in the `references/` directory as needed for deep-dives into specific functionality.

---

_Generated by KIVA Agentic Team._
