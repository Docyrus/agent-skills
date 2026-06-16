# `secure_exec` tool

Runs a sandboxed JavaScript body when the agent calls the tool. Use for multi-step logic, calling the Docyrus REST API (including writes), and shaping/compacting results before they reach the model. The most flexible tool type — and the only one that can mutate data.

Runtime: the `secure-exec` Node sandbox (`NodeRuntime`). Your body runs as an **ES module** (`/script.mjs`), so **top-level `await` works** — no wrapper function needed. On success the LLM receives:

```json
{ "type": "secure_exec_result", "data": <your exports>, "durationMs": <n>, "stdout": "<captured>" }
```

If the script throws (or exits non-zero), the LLM instead receives an error envelope — not a `secure_exec_result`:

```json
{ "type": "error", "error": "Script execution failed: <message>" }
```

## Config fields

| Flag | Key | Required | Meaning |
| --- | --- | --- | --- |
| `--secureExecCode` | `secure_exec_code` | **Yes** | The JS body. Prefer `--from-file code.js` then send the file's text, or include it in a `--from-file payload.json`. |
| `--inputJsonSchema` | `input_json_schema` | **Yes** | Schema for the `data` object the LLM produces. Required even if empty. |

Consider `--needsApproval true` for tools that create/update/delete records.

## Sandbox limits & policy

| Limit | Value (AI tool) |
| --- | --- |
| Wall-clock timeout | **10 s** (hardcoded for AI tools) |
| Memory | 128 MB (max 256) |
| Code size | 512 KB |
| `record` + `data` | 1 MB serialized |
| stdout / stderr | 1 MB each |

Denied by default: filesystem, child processes, environment variables, and **all outbound network except the tenant's Docyrus API host** (`fetch`/`http`/DNS are allowlisted to that host only). There is no opt-in to external network for AI tools.

Limit notes (from `SCRIPT_LIMITS` / `secureExecTool.ts`):

- The 10 s timeout is hardcoded for AI tools; the underlying engine default is 30 s (max 60 s), but AI tools always pass 10 s.
- Memory and the 256 MB cap are engine defaults; AI tools don't override memory, so they run at 128 MB.
- `record` + `data` are size-checked **combined** (1 MB total serialized).
- stdout/stderr are capped at 1 MB **each**, then silently truncated. `stderr` never reaches the LLM (operator logs only).

## Globals available to the script

| Name | Value |
| --- | --- |
| `data` | The LLM's arguments (validated against `input_json_schema`). Injected as a `const` — mutate its properties, but don't reassign `data` itself. |
| `record` | Always `{}` for AI tools. Also a `const`. |
| `api.ds` | Pre-authenticated Docyrus data API (bearer token is closure-captured — unreadable from user code). The **only** authenticated path. |
| `console.log` / `console.error` | Captured into stdout / stderr. `stderr` is logged for operators but **not** sent to the LLM. |
| `fetch` | The real global, but permission-gated to the API host only **and unauthenticated** — the token lives in `api.ds`'s closure, so a raw `fetch` to the API gets 401. Prefer `api.ds`; reach for `fetch` only for unauthenticated endpoints on the same host. |
| Standard JS & web APIs | Full ES built-ins (`Date`, `Math`, `JSON`, `Promise`, `async`/`await`) plus the web-style globals the Node-compatible isolate provides — `TextEncoder`/`TextDecoder`, `URL`/`URLSearchParams`, `structuredClone`, `atob`/`btoa`, `queueMicrotask`, timers, etc. Anything that touches the filesystem, child processes, env vars, or non-API network is gated by the permission layer and **throws** if attempted, regardless of how it's reached. |

`api.ds` methods (all `async`; return parsed JSON when the response is `application/json`, otherwise the raw text body; throw on non-2xx as `API request failed: <status> <statusText> - <body>`):

| Call | REST endpoint |
| --- | --- |
| `api.ds.list(appSlug, dsSlug, params?)` | `GET /apps/:appSlug/data-sources/:dsSlug/items?…` |
| `api.ds.get(appSlug, dsSlug, id, params?)` | `GET …/items/:id` |
| `api.ds.create(appSlug, dsSlug, payload)` | `POST …/items` |
| `api.ds.update(appSlug, dsSlug, id, payload)` | `PATCH …/items/:id` |
| `api.ds.delete(appSlug, dsSlug, id)` | `DELETE …/items/:id` |

`params` is a plain object; object values (e.g. `filters`, `sort`) are JSON-stringified into the query string, matching the REST query payload.

## Returning a result

Assign to **`exports`** — that object is the tool result `data`. If you set nothing you get `{}`. Keep returns small and structured: **the LLM pays tokens for everything you return**, so map raw API rows down to the fields it needs.

```js
exports.count = items.length;
exports.items = items.map(i => ({ id: i.id, name: i.name }));
```

## Example: `find_overdue_invoices`

`input_json_schema`:

```json
{ "type": "object",
  "properties": {
    "daysOverdue": { "type": "number", "minimum": 1 },
    "customerId":  { "type": "string" }
  },
  "required": ["daysOverdue"] }
```

`secure_exec_code`:

```js
const cutoff = new Date(Date.now() - data.daysOverdue * 86400000).toISOString();
const filters = { status: { eq: "open" }, dueDate: { lt: cutoff } };
if (data.customerId) filters.customerId = { eq: data.customerId };

const invoices = await api.ds.list("finance", "invoices", {
  filters, limit: 50, sort: [{ field: "dueDate", direction: "asc" }],
});

const rows = invoices.data || [];
exports.count = rows.length;
exports.totalDue = rows.reduce((s, i) => s + (i.amount || 0), 0);
exports.invoices = rows.map(i => ({ id: i.id, customer: i.customerName, amount: i.amount, dueDate: i.dueDate }));
```

Create it (code from a file keeps quoting sane):

```bash
docyrus apps ai-tools create --appSlug finance \
  --name "Find overdue invoices" --key find_overdue_invoices --type secure_exec \
  --needsApproval false \
  --inputJsonSchema "$(cat schema.json)" \
  --secureExecCode "$(cat find-overdue.js)"
```

Or put everything (including `secure_exec_code` as a JSON string) in one `--from-file payload.json`.
