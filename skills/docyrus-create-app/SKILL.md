---
name: docyrus-create-app
description: Create a new Docyrus app from the terminal with `docyrus apps create` and bootstrap a local dev workspace. Use when the user wants to scaffold a new app (web app / `external-app`, React techstack), create an app with an empty git repo, remix an existing template app, list remixable template apps, clone an existing app's repo locally, or generate the `.env` for local development against Docyrus. Triggers on "create an app", "new web app", "scaffold a Docyrus app", "start a new Docyrus frontend/project", "empty app repo", "remix a template app", "clone my app locally", "set up local development for an app", `docyrus apps create`, `docyrus apps clone`, `docyrus apps templates`, or any Docyrus app-creation + local-dev-bootstrap task via the CLI. For the full CLI command index see docyrus-cli-app; for building the app's code after cloning see docyrus-app-dev-react.
---

# Docyrus App Creation (CLI)

Scaffold a new Docyrus app and, optionally, pull its code down for local development — all from `docyrus apps`. Today this covers **web apps** (`type: external-app`) on a React techstack; mobile/portal types land later.

Every command prints its authoritative, always-current flags with `docyrus apps <cmd> --help`. Add `--json` for machine-readable output.

## Pick a creation mode (read first)

`docyrus apps create` creates the app record and provisions its git repo one of three ways — choose with a flag:

| Mode | Flag | Repo you get |
|---|---|---|
| **Techstack starter** (default) | *(none)* | A fork of the techstack's starter template — a ready-to-run app |
| **Empty repo** | `--empty-repo` | A fresh repo with only an initial commit — bring your own code |
| **Remix a template** | `--from-template <appId\|slug>` | A copy of an existing template app's repo |

Repo provisioning runs **asynchronously** after the app record is created — `git_repo` is populated a few seconds later. The `--local-development` and `apps clone` flows poll for it, so you don't have to wait manually.

## Create an app

The CLI **requires** `--slug` (unlike the UI, which auto-generates one). Check a slug is free first with `docyrus apps check-slug` (see below).

```bash
# React web app from the techstack starter (defaults: type=external-app, techstack=react-spa)
docyrus apps create --name "My CRM" --slug my-crm --json

# Pick a techstack (react-spa | vue-spa | svelte-spa | nextjs)
docyrus apps create --name "My Site" --slug my-site --techstack nextjs

# Empty repo (no starter code)
docyrus apps create --name "Blank" --slug blank --empty-repo

# Remix an existing template app (discover one with `apps templates`)
docyrus apps create --name "From Template" --slug my-copy --from-template crm-starter
```

Key flags (`docyrus apps create --help` for the full list):

- `--name <name>` — required.
- `--slug <slug>` — **required** (lowercase letters/numbers/`-`/`_`, must start with a letter). Verify it's free first with `docyrus apps check-slug --slug <slug>`.
- `--type <appType>` — defaults to `external-app` (the only creatable type today).
- `--techstack <id>` — defaults to `react-spa`. **The id is `react-spa`, not `react-ts`.**
- `--description`, `--route-path </crm>`.
- `--no-api-app` — skip provisioning an OAuth client (by default one is created so local sign-in works).
- `--local-development` — after create, clone the repo and write a dev `.env` (see below); pair with `--dir <path>` and `--client-id <id>`.

`--empty-repo` and `--from-template` are mutually exclusive.

## Check a slug is available

Slugs are unique per tenant, so check before you create:

```bash
docyrus apps check-slug --slug my-crm     # → { "available": true }
```

If a slug is taken, `apps create` fails with a 409 — pick another or add a suffix (e.g. `my-crm-2`).

## Set up local development (`--local-development` and `apps clone`)

`--local-development` (on `create`) and the standalone `apps clone` run the same routine: **wait for the repo → clone it → write a ready-to-run `.env`**.

```bash
# create + set up local dev in one step
docyrus apps create --name "My CRM" --local-development --dir ./my-crm

# set up local dev for an EXISTING app (selector: --app-slug or --app-id)
docyrus apps clone --app-slug my-crm --dir ./my-crm
```

It clones into `./<repoName>` (or `--dir`) and writes `.env` with everything the app's `@docyrus/api-client` / `@docyrus/signin` need:

```
VITE_DOCYRUS_TENANT_ID, VITE_APP_ID
VITE_DOCYRUS_API_URL, VITE_API_BASE_URL          # the active environment's API base URL
VITE_OAUTH2_CLIENT_ID, VITE_OAUTH2_SCOPES         # from the app's OAuth client
VITE_OAUTH2_REDIRECT_URI=/auth/callback, VITE_OAUTH2_REDIRECT_PATH=/auth/callback
VITE_ALLOWED_HOST_ORIGINS
DOCYRUS_API_CLIENT_ID, DOCYRUS_SANDBOX_APP_ID
```

Then run it:

```bash
cd ./my-crm && pnpm install && pnpm run dev
```

The command writes `.env` and clones for you, but **cannot change your shell's directory** — `cd` into the printed path yourself. It refuses to clone into a non-empty directory.

## Find template apps to remix

```bash
docyrus apps templates                 # external-app templates (default)
docyrus apps templates --app-type all  # every app type
```

Use a listed `id` or `slug` with `apps create --from-template`.

## Critical rules & gotchas

- **Techstack id is `react-spa`** (not `react-ts`). Valid ids: `react-spa`, `vue-spa`, `svelte-spa`, `nextjs`. An unknown techstack is rejected before anything is created.
- **Only `external-app` is creatable today.** Mobile/portal types are not yet supported by the CLI.
- **Repo provisioning is async.** Immediately after `create`, `git_repo` may be empty; the local-dev/clone flows poll (~2 min) until it is ready. On timeout, retry `docyrus apps clone --app-id <id>` later.
- **Private repos clone with a managed token.** You do not need your own GitHub access to the `docyrus-apps` org — the CLI fetches a short-lived, repo-scoped token and embeds it in the clone remote. The token is never printed or stored in output.
- **OAuth client is provisioned by default**, so local sign-in works out of the box; remixed apps inherit one too. If you pass `--no-api-app` (or clone an app that has no client), `VITE_OAUTH2_CLIENT_ID` in `.env` is empty — pass `--client-id <id>` to supply one.
- **There is no "create from an arbitrary git URL" mode** — the three modes above are the supported repo sources.
- **The CLI requires `--slug`** (all modes, including `--from-template`). The platform itself auto-generates a slug when one is omitted (slugified app name, truncated to 30 chars, with a random 4-char suffix if that base is already taken) — but the CLI intentionally makes you pass one explicitly.

## Validate

```bash
# the new app appears (type external-app, a generated slug)
docyrus apps list --json

# once provisioning completes, git_repo (and api_client) are populated
docyrus curl "/v1/dev/apps/<appId>?expand=repo,api_client" --json
```

## Test (end-to-end)

```bash
docyrus apps create --name "Smoke Test" --local-development --dir /tmp/smoke --json
cd /tmp/smoke && pnpm install && pnpm run dev      # app boots; sign-in uses the generated .env
# cleanup:
docyrus apps permanent-delete --app-slug smoke-test
```

## Related skills

- **docyrus-cli-app** — the full `docyrus` CLI command index (auth, env, ds, studio, …).
- **docyrus-app-dev-react** / **docyrus-api-dev** — build the app's code against the Docyrus API after cloning.
- **docyrus-data-source-design** — model the data the app reads and writes.
