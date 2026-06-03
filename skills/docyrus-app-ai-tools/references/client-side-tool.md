# `client_side` tool

A tool with **no server-side execution**. When the agent calls it, the AI SDK surfaces the call to the **client** (the web/desktop/mobile host app), which runs its own handler and returns the result into the conversation. Use for actions that only make sense in the user's environment: navigate the UI, open a modal/record, read the current selection, render a custom component, request input, or call a host/device API.

There is no sandbox and no Docyrus-side execution — **the host app must implement a handler** for the tool's `key`. If the app has no handler registered, the call will not resolve.

## Config fields

| Flag | Key | Required | Meaning |
| --- | --- | --- | --- |
| `--type` | `type` | **Yes** | Set to `client_side`. |
| `--clientSideExecution` | `client_side_execution` | **Yes** | `true`. This boolean is what actually registers the tool as client-handled. |
| `--inputJsonSchema` | `input_json_schema` | **Yes** | The arguments the LLM produces. Without it the tool is skipped at runtime. |
| `--outputJsonSchema` | `output_json_schema` | No | Shape the client returns — declare it so the model knows what it gets back. |
| `--description` | `description` | Strongly recommended | Drives tool selection; a missing one is logged as degrading quality. |
| `--environments` | `environments` | No | Limit to `web` / `desktop` / `ios` where a handler exists. |

> Set both `type=client_side` **and** `client_side_execution=true`. Do not also give it an action — a tool with both an action and `client_side_execution` skips client registration (the action path wins).

## How it runs

1. The agent decides to call the tool and emits arguments matching `input_json_schema`.
2. The AI SDK yields the tool call to the client instead of executing server-side.
3. The host app's handler (keyed by the tool `key`) performs the action and returns a result (ideally matching `output_json_schema`).
4. The result is fed back to the agent to continue the turn.

Authoring the skill only creates the contract (key + schemas + description). The handler implementation lives in the front-end app and is out of scope for the CLI.

## Example: `open_record`

```json
{
  "name": "Open record",
  "key": "open_record",
  "type": "client_side",
  "client_side_execution": true,
  "description": "Opens a record's detail page in the current app UI so the user can see or edit it. Call after you have identified the exact record the user wants to view.",
  "environments": ["web", "desktop"],
  "input_json_schema": {
    "type": "object",
    "properties": {
      "dataSourceSlug": { "type": "string", "description": "Slug of the data source" },
      "recordId": { "type": "string", "description": "Record id to open" }
    },
    "required": ["dataSourceSlug", "recordId"]
  },
  "output_json_schema": {
    "type": "object",
    "properties": { "opened": { "type": "boolean" } }
  }
}
```

```bash
docyrus apps ai-tools create --appSlug crm --from-file open-record.json
```

Once the app is installed in the tenant, the tool is available to the base assistant automatically — make sure the web app registers an `open_record` handler for it. Mention in the agent context when to call it (e.g. "use `open_record` only after confirming the record with the user").
