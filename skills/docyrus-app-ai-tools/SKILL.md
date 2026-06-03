---
name: docyrus-app-ai-tools
description: Create and manage app-scoped AI tools for Docyrus AI agents using the `docyrus apps ai-tools` CLI commands, and set app-level agent guidance with `docyrus apps set-agent-context`. Use when building custom tools an AI agent can call — covering all four executable tool types — `data_source_query` (read one data source with author-fixed shaping and parameter-bound filters), `custom_query` (hand-written read-only SELECT SQL with Handlebars templating), `secure_exec` (sandboxed JavaScript that calls the Docyrus REST API), and `client_side` (tool executed in the user's browser/app). Triggers on tasks like "create an AI tool for an app", "add a data source query tool", "build a custom query tool", "write a secure_exec tool", "make a client-side tool", "let the agent query/look up X", `docyrus apps ai-tools create/update/list/get/delete`, `docyrus apps set-agent-context`, or writing the agent context that tells the system base assistant (Docy) when to use which app-scoped tool.
---

# Docyrus App-Scoped AI Tools

Build custom tools that the Docyrus system base assistant ("Docy") can call during a conversation. Tools are created per app (`tenant_ai_tool`, `ownership=CUSTOM`, `tenant_app_id=<app>`) with the `docyrus apps ai-tools` CLI. Once the app is installed in the tenant, its tools are **automatically available to the base assistant** — there is no per-agent wiring.

## End-to-end workflow

1. **Create the tool** — `docyrus apps ai-tools create` with the right `--type` and that type's config.
2. **Guide the base assistant** (optional but recommended) — `docyrus apps set-agent-context` to tell "Docy" *when to use which tool*.

**App-scoped tools are owned by the app and exclusive to the system base assistant ("Docy").** When the app is installed in the tenant, its tools are attached to Docy automatically — you do **not** (and cannot) wire them to a specific agent, and there is no `agent tools` attach step. If the app is **not** installed in the tenant, its tools are not loaded.

> App-scoped tools cannot be attached to custom AI agents. Giving a *custom* agent its own tools is a different, agent-owned flow (`docyrus agent tools …`) outside this skill's scope.

All commands need an authenticated CLI session (`docyrus auth who` to verify). The app is selected with **exactly one** of `--appId` or `--appSlug` on every command.

## The four tool types — pick one

| `--type` | Use when | Executes | Author config |
| --- | --- | --- | --- |
| `data_source_query` | Read/list records from **one** data source; the LLM only supplies filter values | Server, RLS-enforced | data source id + fixed columns/limit/formulas + a filter template with `{{param}}` bindings → [data-source-query-tool.md](references/data-source-query-tool.md) |
| `custom_query` | Read across joins/aggregations needing hand-written **read-only** SELECT SQL | Server, read-only txn, RLS | a Handlebars-templated SQL string → [custom-query-tool.md](references/custom-query-tool.md) |
| `secure_exec` | Multi-step logic, calling the Docyrus REST API (incl. writes), transforming/compacting results | Server sandbox (10s, no FS/env, Docyrus-API-only network) | a JavaScript body → [secure-exec-tool.md](references/secure-exec-tool.md) |
| `client_side` | The action must run in the **user's browser/app** (UI navigation, local selection, host APIs) | Client/frontend | input/output schema only; the host app implements the handler → [client-side-tool.md](references/client-side-tool.md) |

Read the matching reference file before authoring that type — each documents the exact config fields, templating/binding rules, runtime behavior, and worked examples.

## `docyrus apps ai-tools` command surface

Routes to `/v1/dev/apps/:appId/ai-tools`. The CLI resolves `--appSlug` to an app id.

```bash
docyrus apps ai-tools list   --appSlug <slug>
docyrus apps ai-tools get    --appSlug <slug> --toolId <id>
docyrus apps ai-tools create --appSlug <slug> --type <type> [config flags | --from-file payload.json]
docyrus apps ai-tools update --appSlug <slug> --toolId <id> [flags | --from-file payload.json]
docyrus apps ai-tools delete --appSlug <slug> --toolId <id>
```

### Convenience flags (camelCase flag → snake_case payload key)

Common to every type:

| Flag | Key | Notes |
| --- | --- | --- |
| `--name` | `name` | **Required on create.** Display name. |
| `--key` | `key` | **Required on create.** The function name the LLM sees. `snake_case`, stable, and **globally unique** (a DB `UNIQUE` constraint across all tenants) — namespace it (e.g. `crm_get_customer_balance`) to avoid collisions. |
| `--description` | `description` | **Drives LLM tool selection — always write a clear, specific one.** |
| `--type` | `type` | **Must be set** to `data_source_query` \| `custom_query` \| `secure_exec` \| `client_side`. Omitting it defaults to `system`, which will not register as any of these. |
| `--inputJsonSchema` | `input_json_schema` | **Required at runtime for all four types** (the LLM's argument schema). A no-argument tool still needs `{"type":"object","properties":{}}`. |
| `--outputJsonSchema` | `output_json_schema` | Optional result schema (mainly `client_side`). |
| `--icon` | `icon` | Optional. |
| `--environments` | `environments` | Comma list of `web,desktop,ios`. Restricts where the tool is offered. |
| `--needsApproval` | `needs_approval` | Require user approval before the call runs. Use for tools that mutate data. |
| `--dynamicApprovalFormula` | `dynamic_approval_formula` | JSONata that decides approval per-call. |

Type-specific flags (`--secureExecCode`, `--customQuerySqlQuery`, `--customQueryFilters`, `--dataSourceQueryDataSourceId`, `--dataSourceQueryColumns`, `--dataSourceQueryFilters`, `--dataSourceQueryFormulas`, `--dataSourceQueryChildQueries`, `--dataSourceQueryLimit`, `--clientSideExecution`) are documented in each type's reference file.

`--from-file <path>` / `--data '<json>'` send a raw JSON payload (snake_case keys); convenience flags are merged **over** it. **Prefer `--from-file` for anything with JSON schemas, SQL, or code** — it sidesteps shell quoting. JSON-typed flags (`--inputJsonSchema`, `--dataSourceQueryFilters`, …) expect a JSON string when passed inline.

> The endpoint forces `ownership=CUSTOM` and `tenant_app_id`. Platform-managed fields (`group`, `avatar`, `restricted`, `cost`, `development_status`, `core_action_id`, `core_data_provider_id`, `owner_product_id`) are not settable here.

## Set the app's agent context

`agent_context` is app-level guidance text injected into the base assistant's prompt. Use it to orchestrate: name each tool's `key` and say **when** to reach for it, what each returns, and any ordering ("look up the customer with `get_customer` before calling `get_customer_balance`").

```bash
docyrus apps set-agent-context --appSlug <slug> --from-file agent-context.md   # recommended for prose
docyrus apps set-agent-context --appSlug <slug> --value "Use get_customer_balance when the user asks about balances or overdue amounts."
docyrus apps set-agent-context --appSlug <slug> --clear                        # remove it
```

Provide **exactly one** of `--value`, `--from-file`, or `--clear`. (This writes `agent_context` via `PATCH /v1/dev/apps/:appId`; `docyrus apps update --agentContext` does the same.)

## Authoring checklist

- `--type` set, and the matching config provided (see the type's reference file).
- `input_json_schema` present (even if empty) and describing **only** what the LLM should supply.
- `description` is specific enough for the model to choose the tool correctly; `key` is `snake_case` and stable.
- The app is **installed in the tenant** — that's what surfaces its tools to the base assistant (Docy). App-scoped tools auto-attach to Docy only; they're never wired to custom agents.
- Mutating tools (`secure_exec` writes, etc.) consider `--needsApproval true`.
- Agent context mentions the new tool's `key` and trigger conditions.
- Verify with `docyrus apps ai-tools get --toolId <id>`, then exercise it by chatting with the base assistant (`docyrus docy "..."`).
