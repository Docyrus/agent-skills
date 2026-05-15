# Automation & Workflows

## Automation Engine

Define event-driven workflows with triggers and action chains. Each automation belongs to a tenant app and combines one or more triggers with a graph of typed action nodes.

## Trigger Types

The runtime supports ten trigger types. The dev API exposes each as a typed endpoint, and the CLI exposes the same set under `docyrus automation create-trigger --type <kebab-case>`.

- `record-created` — fires when a record is inserted into the source data source
- `record-modified` — fires when specific columns change (with `all` / `any` match mode)
- `record-deleted` — fires when a record is removed from the source data source
- `recurrence` — fires on a cron-like schedule (hour / day / week / month / year, with run-at time and weekday/month-day options)
- `app-event` — fires from a connector or core data-provider event (with optional webhook binding)
- `webhook` — fires when an external HTTP webhook is received
- `emailhook` — fires when a tenant-bound inbound email is received
- `webform` — fires when a public webform is submitted
- `button-activation` — fires when a user activates an in-UI button on a record
- `manual-activation` — fires when a record is manually activated via the UI

The platform also tracks `max_run_per_record` for record-* and recurrence triggers to prevent re-firing on the same record.

## Action Node Types

Action nodes are the building blocks of an automation's execution graph. The dev API exposes each as a typed endpoint, and the CLI exposes the same set under `docyrus automation create-node --type <kebab-case>`.

- `external-action` — invokes a registered `core_action` against a connector (requires `action_type_id`; backend validates payload against `core_action.input_json_schema` and provisions the `tenant_action` row)
- `send-email` — sends an email through a tenant email account or template
- `send-notification` — pushes an in-app or device notification
- `create-record` — inserts a record into a target data source
- `update-records` — bulk-updates records in a target data source (optionally pivoted on a target field)
- `request-approval` — opens an approval cycle against an input data source
- `request-input` — collects ad-hoc input from a user via the input data source
- `http-request` — fires an HTTP request (supports batch mode, transformers, and connection auth)
- `data-source-query` — runs a data source query and emits the result into the chain
- `custom-query` — runs a saved custom SQL query
- `generate-document` — renders an HTML/PDF/DOCX template against a record
- `ai-prompt` — runs a stored prompt against an AI provider
- `ai-agent` — invokes a Docyrus AI agent
- `execute-script` — runs a sandboxed JavaScript snippet
- `wait-for` — bridge action that delays the next step. Does no work itself; forwards input data unchanged and queues the next step(s) with `tenant_job_queue.process_after = clock_timestamp() + delaySeconds` so the worker defers execution. Configure via `data.delaySeconds` (integer, ≤ 30 days) or the `data.delayValue` + `data.delayUnit` (`seconds`/`minutes`/`hours`/`days`) pair.

## Composition Features

- Conditional branches via per-node `condition` payloads
- Action chains with parent/child wiring (`parent` node id)
- Field mappings (`field_mapping`, `dynamic_field_mapping`) for record-shaped nodes
- Request lifecycle hooks (`pre_action_request`, `post_action_request`) on external-action nodes
- Input/output transformers (`input_template`, `input_transformer`, `output_transformer`, `batch_transformer`, `error_transformer`) on http-request nodes
- Custom headers (`custom_headers`) for http-request nodes
- Target data source conditions (`target_data_source_condition`) on update-records and external-action nodes
- Soft-delete (archiving) for both automations and individual nodes
