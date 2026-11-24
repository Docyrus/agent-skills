# Webform model and submission

A generic webform is a `tenant_webform` row with a paired `tenant_webhook`. The Webforms Studio uses `/v1/dev/webforms` for administration. The `docyrus studio` webform commands call the same API.

## Create and update

| CLI flag | JSON body key | Behavior |
|---|---|---|
| `--name` | `name` | Required on create |
| `--schema` | `schema` | Required on create. Use the [native builder JSON](webform-design-schema.md#2-canonical-persisted-design) with `version: 1` and `children[].field.slug` |
| `--status` | `status` | Required on create. `0` = draft, `1` = active, `2` = inactive. The CLI help lists 1 and 2, and the API also accepts 0 |
| `--dataSourceId` / `--dataSourceSlug` | `dataSourceId` | Optional, create-only. Resolve slugs using the app selector |
| `--webformOptions` | `options` | Optional layout, theme, and copy |
| `--css` | `css` | Stored custom CSS. The current hosted page and embed load the bundle stylesheet and use the form theme options |
| `--sandbox` | `sandbox` | Legacy flag, defaults to false. The current Preview tab uses the hosted page on the embed origin |

`update-webform` makes a partial PATCH. `get-webform`, `update-webform`, and `delete-webform` take the administrative UUID via `--webformId`. Deleting the form also removes its paired webhook.

Use `--data` or `--from-file` for the settings below. The dedicated CLI flags override matching fields in the JSON.

- Set `tenantBrandId` and `slug` for a branded address. The brand needs a `workspace_slug`. The form slug is unique within its brand and matches `^[a-z0-9]+(?:[-_][a-z0-9]+)*$`. Check availability with `GET /v1/dev/webforms/slug/availability` using `tenantBrandId`, `slug`, and the optional current `webformId`.
- `allowedOrigins` lists the exact website origins allowed to submit, such as `https://www.example.com`, with no page path. Studio requires at least one on create. The admin API and CLI allow it to be omitted. Use the relevant Studio origin for the live hosted page, the embed origin for hosted preview, and the website origin for an external embed when using an allowlist. An empty list clears the origin restriction. When using custom CAPTCHA keys, configure the provider for the hostnames on which the widget runs.
- Generic public forms use the platform Turnstile by default. Set `captchaProvider` to `turnstile` or `recaptcha_v2` with `captchaSiteKey` and write-only `captchaSecretKey` to use the form's own keys. Set `captchaProvider: null` to restore the platform configuration.

## Publishing and embed

| Purpose | Where to get it | Address |
|---|---|---|
| Hosted link for respondents | Studio **Publish** | `/w/<workspaceSlug>/<formSlug>` when both slugs exist, otherwise `/s/<formId>/<formToken>`, on the Studio host |
| Hosted preview and website embed | Studio **Preview** (`#preview`, also reached by old `#embed` links) | Hosted preview on the embed host. The snippet includes versioned CSS and JS, the mount div, and `data-webform-id`, `data-webform-target`, and `data-webform-api` |
| Public API | Admin detail `form_id`, `form_token`, and `form_submit_url` | GET for the form JSON and POST for submission at `/v1/webforms/<formId>/<formToken>`. The workspace equivalent is `/v1/webforms/workspaces/<workspaceSlug>/<formSlug>` |
| API-generated embed | Admin detail `embed_code` | Versioned CSS and JS from the API-configured embed host, with an `s/…` address |

`formId` in a public address is the webhook's short ID. The API also returns environment-configured `form_url` and `form_sandbox_url` values. Use the Studio link when distributing the current hosted page.

The readable workspace address requires an active form. The generic id/token route can still serve and accept submissions for drafts and inactive forms, subject to access settings and submission checks. Use a test form for end-to-end preview submissions. Invite-only forms use recipient links and reject the ordinary public endpoints.

## Browser submission

1. The public GET returns `captcha` with the provider and site key. The hosted and embedded form render the challenge from this config.
2. Submit JSON with the answer object under `data` and the solved token as top-level `captchaToken`. The API accepts legacy `turnstileToken` as an alias. A non-empty generic public submission requires a valid token. If the webhook has allowed origins, the request must also come from an allowed origin.
3. The API queues the submission in `tenant_webhook_data` and returns `201` with `{ id, created_at }`. Missing, null, or empty-object `data` is a verification ping returning `{ ok: true }` without queuing a record.
4. The queue writes the answers to the bound data source by slug or to the tenant's `webform_record` table. Verify the result separately from the POST response. Read unbound responses in the Studio Submissions view or with `GET /v1/webforms/<webformId>/items`.

Calendar booking forms follow the separate booking contract and are outside this generic intake workflow.

## Server-to-server submission

For an authorized server integration, enable the server submission URL in Studio or with `POST /v1/dev/webforms/<webformId>/server-submit`. Save the returned `server_submit_url` as a secret. The full URL is returned only on creation or rotation. Submit `{ "data": { … } }` to it from the server. This endpoint authenticates with its own URL secret and rejects requests carrying a browser `Origin` header. Rotation invalidates the previous URL. Revoke it with `DELETE /v1/dev/webforms/<webformId>/server-submit`.

## Automation

An active automation `webform` trigger can run after a queued submission for a bound or unbound form.

```bash
docyrus automation create-trigger --appSlug crm --automationId <automationId> \
  --type webform --webformId <webformId> --json
```

The trigger binds to the admin webform UUID. Its input is the submitted data, optionally transformed with JSONata. See **docyrus-automation-design** for trigger and action configuration.

## Implementation sources

- Core CLI and admin API: `apps/cli/src/commands/studioCommands.ts`, `apps/api/src/dev/dto/webform.ts`, and `apps/api/src/dev/app/webform.{controller,service}.ts`
- Core public API and queue: `apps/api/src/webform/`, `apps/api/src/auth/strategy/webform-slug.strategy.ts`, and `libs/shared/src/edge/queue-webhook/index.ts`
- Webforms Studio and Worker: `apps/webforms/src/lib/embed-config.ts`, `apps/webforms/src/hooks/use-webform-public-urls.ts`, and `apps/webforms/src/worker.ts`
