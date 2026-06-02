# `secure_exec` tool

Runs a sandboxed JavaScript body when the agent calls the tool. Use for multi-step logic, calling the Docyrus REST API (including writes), and shaping/compacting results before they reach the model. The most flexible tool type — and the only one that can mutate data.

Runtime: the `secure-exec` Node sandbox. Result returned to the LLM:

```json
{ "type": "secure_exec_result", "data": <your exports>, "durationMs": <n>, "stdout": "<captured>" }
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

Denied by default: filesystem, child processes, environment variables, and **all outbound network except the tenant's Docyrus API host**. There is no opt-in to external network for AI tools.

## Globals available to the script

| Name | Value |
| --- | --- |
| `data` | The LLM's arguments (validated against `input_json_schema`). |
| `record` | Always `{}` for AI tools. |
| `api.ds` | Pre-authenticated Docyrus data API (token is closure-captured — unreadable from user code). |
| `console.log` / `console.error` | Captured into stdout / stderr. `stderr` is logged for operators but **not** sent to the LLM. |
| Standard JS | `Date`, `Math`, `JSON`, `fetch`, async/await, etc. |

`api.ds` methods (all `async`, throw on non-2xx as `API request failed: <status> <statusText> - <body>`):

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
