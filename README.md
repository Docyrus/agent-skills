# Docyrus Agent Skills

A comprehensive library of instructions and reference materials designed for AI agents (and developers) working with the Docyrus platform.

## Overview

This repository centralizes the knowledge and patterns required to manage the Docyrus platform, integrate with its APIs, and develop React-based applications using Docyrus as a backend.

## Installation

To add these skills to your environment, run:

```bash
npx skills add docyrus/agent-skills
```

## Available Skills

| Skill                     | Description                                                                                       | Location                                            |
| :------------------------ | :------------------------------------------------------------------------------------------------ | :-------------------------------------------------- |
| **Docyrus API Dev**       | Integrate with Docyrus API using `@docyrus/api-client` and `@docyrus/signin`.                     | [SKILL.md](./skills/docyrus-api-dev/SKILL.md)       |
| **Docyrus App Dev React** | Build Docyrus React TypeScript applications end-to-end, combining auth, collections, queries, TanStack patterns, and production-grade UI implementation. | [SKILL.md](./skills/docyrus-app-dev-react/SKILL.md) |
| **Docyrus Data Grid Page Design** | Build Docyrus web data-grid and list pages with `DataGrid`, `DataGridViewSelect`, `useDataGrid`, `useDocyrusDataViewSelect`, and `useDocyrusDataGrid`. | [SKILL.md](./skills/docyrus-data-grid-page-design/SKILL.md) |
| **Docyrus Record Detail Form Design** | Build Docyrus record forms, detail panels, and inline-edit views with `useDocyrusFormView`, `DynamicFormField`, `DynamicValue`, `EditableRecordDetail`, `EditableValue`, and value renderers. | [SKILL.md](./skills/docyrus-record-detail-form-design/SKILL.md) |
| **Docyrus API Doctor**    | Post-implementation checklist to catch bugs, performance issues, and code quality problems in Docyrus API code. | [SKILL.md](./skills/docyrus-api-doctor/SKILL.md)    |
| **Docyrus CLI App**       | Interact with the Docyrus platform from the terminal using the `docyrus` CLI for auth, data records, schema management, and API requests. | [SKILL.md](./skills/docyrus-cli-app/SKILL.md)       |
| **Docyrus Data Source Design** | Design, validate, and test a data source (schema, fields, relations, enums, access rules) end-to-end with `docyrus studio`. | [SKILL.md](./skills/docyrus-data-source-design/SKILL.md) |
| **Docyrus Automation Design** | Build, validate, and test automations — triggers plus action-node graphs (conditions, field mappings, sequencing) — with `docyrus automation`. | [SKILL.md](./skills/docyrus-automation-design/SKILL.md) |
| **Docyrus Agent Design** | Design custom AI agents and their sub-resources (tools, data sources, docs, MCPs, connections, workflow steps, deployments) with `docyrus agent`. | [SKILL.md](./skills/docyrus-agent-design/SKILL.md) |
| **Docyrus Email Template Design** | Design, validate, and test data-bound email templates (`{{field}}` Handlebars) with `docyrus studio`. | [SKILL.md](./skills/docyrus-email-template-design/SKILL.md) |
| **Docyrus Print PDF Template Design** | Design, validate, and test HTML/PDF export/report templates (rendered per record) with `docyrus studio`. | [SKILL.md](./skills/docyrus-print-pdf-template-design/SKILL.md) |
| **Docyrus Webform Design** | Design public webforms that collect submissions into records, bound to a data source, with `docyrus studio`. | [SKILL.md](./skills/docyrus-webform-design/SKILL.md) |
| **Docyrus ACL Design** | Design, configure, and validate tenant ACL — roles, permissions, role queries, hierarchy units — with `docyrus acl`. | [SKILL.md](./skills/docyrus-acl-design/SKILL.md) |
| **Docyrus DSQL Query Design** | Write, discover, and run DSQL queries against logical data sources (incl. `docyrus dsql ask`). | [SKILL.md](./skills/docyrus-dsql-query-design/SKILL.md) |
| **Docyrus Integrations & Connectors** | Discover and use integration connectors — call external APIs/actions through managed provider auth — with `docyrus connect`. | [SKILL.md](./skills/docyrus-integrations-and-connectors/SKILL.md) |
| **Docyrus App AI Tools** | Create app-scoped AI tools for the system base agent ("Docy") — `data_source_query`, `custom_query`, `secure_exec`, `client_side` — with `docyrus apps ai-tools`. | [SKILL.md](./skills/docyrus-app-ai-tools/SKILL.md) |
| **Docyrus Platform** | Architecture and building-blocks reference for the Docyrus platform, and a router to the dedicated design/CLI/API skills. | [SKILL.md](./skills/docyrus-platform/SKILL.md) |

## Repository Structure

Each skill directory is organized as follows:

- `SKILL.md`: The primary instruction file and entry point for the AI agent.
- `references/`: Detailed guides for specific topics like query syntax, authentication flows, and component patterns.

## Usage for AI Agents

When tasked with work related to Docyrus, AI agents should:

1. Identify the relevant skill directory within the `skills/` folder.
2. Read the `SKILL.md` file using the `view_file` tool.
3. Consult the files in the `references/` directory as needed for deep-dives into specific functionality.

---

_Generated by Docyrus Agentic Team._
