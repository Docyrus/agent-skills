---
name: docyrus-integrations-and-connectors
description: Discover and use Docyrus integration connectors from the terminal with the `docyrus connect` CLI commands — call third-party/external APIs (Microsoft Graph/Outlook, Twilio, Meta, etc.) through a connector's managed provider auth. Use when the user wants to find available connectors, inspect a connector's actions and input/output schemas, list a tenant's connections for a provider, run a connector action by provider slug + action key (e.g. send an SMS, send an email, fetch from an external API), or make a raw authenticated HTTP request through a connector's provider auth. Triggers on "list connectors", "what integrations are available", "run a connector action", "call the X API through Docyrus", "send SMS/email via a connector", "connect curl", "provider auth", `docyrus connect`, `docyrus connect run-action`, `docyrus connect curl`, or any connector/integration usage task. NOTE — connections (credentials) are created via the Docyrus UI / OAuth flow, not this CLI — this skill discovers and *uses* connectors. For app-internal AI tools use docyrus-app-ai-tools; for automations that call connectors use docyrus-automation-design.
---

# Docyrus Integrations & Connectors

Use third-party integrations from the terminal with `docyrus connect`. A **connector** (a.k.a. data provider) is a platform-defined external integration (identified by a **slug**, e.g. `msgraph`, `twilio`, `meta`) that bundles managed auth + a set of callable **actions** + data sources. You **discover** connectors and their actions, then **run actions** or make **raw authenticated requests** through the connector's provider auth — without handling tokens yourself.

## Concepts (read first)

- **Connector / data provider** — the integration definition (`core_data_provider`), addressed by **`slug`**. Holds the auth type, base URL, and the actions it exposes.
- **Action** — a callable operation a connector exposes (`core_action`), addressed by **provider slug + action `key`** (e.g. `msgraph` + `sendEmailWithOutlook`). Has `inputJsonSchema` / `outputJsonSchema`.
- **Connection** — a tenant's stored credentials for a connector (`tenant_connection`), or a per-user OAuth2 connection (`tenant_connection_user`). **Connections hold the tokens/keys.**
- **Connection account** — an optional sub-account *within* a connection (e.g. one of several ad accounts / mailboxes), addressed by `--connectionAccountId`.

> ⚠️ **Connections are NOT created by this CLI.** There is no `create-connection` command. Credentials/OAuth connections are set up in the **Docyrus UI** (OAuth flow) or via the raw API. This skill **discovers and uses** connectors; if no connection exists for a provider, the user must connect it first.

## Workflow

1. **Confirm auth.** Every command needs an active session.
   ```bash
   docyrus auth who --json        # or: docyrus auth login
   ```

2. **Discover the connector** you need and its slug:
   ```bash
   docyrus connect list-connectors --q "twilio" --json
   docyrus connect get-connector twilio --json    # → its actions[] and dataSources[]
   ```

3. **Confirm a connection exists** for that provider (you need credentials to actually run):
   ```bash
   docyrus connect list-connections twilio --json
   # → { tenantScope: [{id,name,...}], userScope: { connected, connectionId } }
   ```
   If `tenantScope` is empty and `userScope.connected` is false, **stop and ask the user to connect the provider in the Docyrus UI** first.

4. **Inspect the action's input schema** before calling:
   ```bash
   docyrus connect get-action twilio sendSms --json    # → inputJsonSchema / outputJsonSchema / requestMethod
   ```

5. **Run the action** (build `--params` to match the input schema). **Dry-run first** to preview:
   ```bash
   docyrus connect run-action twilio sendSms -p '{"to":"+1555...","body":"Hi"}' --dryRun --json
   docyrus connect run-action twilio sendSms -p '{"to":"+1555...","body":"Hi"}' --json
   ```
   Or make a **raw authenticated request** when there's no action for what you need:
   ```bash
   docyrus connect curl msgraph "/me/messages" -X GET --json
   ```

A full reference of every command, the data model, auth types, connection resolution, and gotchas is in [references/connector-model-and-actions.md](references/connector-model-and-actions.md).

## Command cheat-sheet

All commands need an active session; append `--json`. Connectors are addressed by **slug**, actions by **slug + actionKey**.

```bash
docyrus connect list-connectors [--q <kw>] [--limit 100] [--offset 0]   # find connectors
docyrus connect get-connector  <slug>                                    # detail: actions[] + dataSources[]
docyrus connect list-connections <slug>                                  # tenantScope[] + userScope{connected,connectionId}
docyrus connect get-action     <slug> <actionKey>                        # input/output JSON schemas + requestMethod
docyrus connect run-action     <slug> <actionKey> -p '<json>' [-c <connId>] [--connectionAccountId <id>] [-n]
docyrus connect curl           <slug> <endpoint> [-X <method>] [-d '<json>'] [--headers '<json>'] [-c <connId>] [--connectionAccountId <id>]
```

- **`run-action`**: `-p`/`--params` is a **JSON object** matching the action's `inputJsonSchema` (server-validated — AJV, 400 on mismatch). `-n`/`--dryRun` previews the request **client-side without sending**. Connection selectors (`-c`/`--connectionId`, `--connectionAccountId`) are sent as headers.
- **`curl`**: `<endpoint>` is a **relative path** (appended to the connection/provider base URL) **or an absolute `http…` URL** (used verbatim). `-X` sets the HTTP method (default `GET`); `-d`/`--headers` are JSON. The provider auth header is injected automatically.

## Critical rules

- **Connectors = slug; actions = slug + key.** There is no connector id or action id in these commands. Get the slug from `list-connectors`, the action `key` from `get-connector`/`get-action`.
- **Connections are not created here.** `list-connections` only *reads*. If none exists, the provider must be connected via the Docyrus UI / OAuth flow (or the raw API) before `run-action`/`curl` can authenticate. Treat an empty `tenantScope` + `userScope.connected:false` as "not connected yet."
- **Connection auto-selection:** with no `--connectionId`, the tenant's **first** connection for that provider is used. For OAuth2 `authorization_code` providers, `--connectionId` is **ignored** and the **current user's** connection (or a shared one) is used. Pass `--connectionId` only to disambiguate among multiple tenant connections.
- **`run-action --params` must be a JSON object** (not an array/primitive) — the CLI rejects otherwise — and is validated server-side against the action's `inputJsonSchema` (400 `"Action input validation failed"` with AJV errors).
- **`run-action` needs the `Automations.Run` scope; read commands need `Connectors.Read.All`.** A session that can list connectors may not be allowed to run actions.
- **`curl` is an RPC passthrough** (`PUT /connectors/{slug}` with the endpoint in the body) — you pass the *external* method via `-X`, not the API method. An absolute endpoint bypasses the connector's base URL.
- **Always `--dryRun` a `run-action` first** when the side effect is real (sending email/SMS, posting data) to confirm the resolved request before executing.
- **The action must exist and be active** (provider `slug` + action `key`, `status=1`) — otherwise 404.

## Test / validate

Read commands are safe to run anytime; action runs have real side effects (gate with `--dryRun`).

1. **Discover round-trip:** `list-connectors` → pick a slug → `get-connector <slug>` shows its `actions[]` → `get-action <slug> <key>` shows the input schema. Confirms the connector + action exist and what params they need.
2. **Connection check:** `list-connections <slug>` — confirm a usable connection (`tenantScope` non-empty or `userScope.connected:true`) before attempting a run.
3. **Dry-run:** `run-action <slug> <key> -p '<params>' --dryRun` returns `{ dryRun:true, method, path, headers, body }` and sends nothing — verify the params/connection resolve as expected.
4. **Execute** only when the dry-run looks right and the side effect is intended; inspect the returned `{ data, status }`.

## References

- **[references/connector-model-and-actions.md](references/connector-model-and-actions.md)** — Full command reference (args/flags/paths), the connector/connection/account/action data model, the supported auth types, how `run-action` resolves a connection (tenant vs per-user OAuth2), `run-action` vs `curl` mechanics, and the gotchas (header-vs-body selectors, scopes, connection creation outside the CLI).
- **docyrus-automation-design** — the `external-action`/`http-request` automation nodes that run connectors inside a workflow. **docyrus-app-ai-tools** — app-scoped AI tools. **docyrus-cli-app** — the full CLI command index. **docyrus-platform** → `references/integrations-and-events.md` — the integrations/events concept overview.
