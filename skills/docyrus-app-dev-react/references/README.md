# Docyrus App Dev React Reference Map

This directory contains the full reference set for `docyrus-app-dev-react`.

## App Development References

Read these files for data access, auth, query design, and collection patterns:

- `api-client-and-auth.md`
- `collections-and-patterns.md`
- `host-communication-integration.md` — embedded-app ↔ host shell `postMessage` integration (all directions **except** Guidy): host→app navigation & notifications, app→host route sync (`syncRouteToHost` / `notifyRouteChange`) and navigation requests (`requestHostNavigation`)
- `guidy-integration.md` — make an embedded app Guidy-compatible: `enableGuidyBridge`, targetable controls (stable ids + labels), and declared routes (`guidyRoutes` / `useDocyrusGuidyRoutes`)
- the **docyrus-acl-design** skill — ACL roles, permissions, and role-queries (`docyrus acl`); plus the ACL REST endpoints in **docyrus-api-dev**'s SKILL.md
- `../../docyrus-api-dev/references/data-source-query-guide.md`
- `../../docyrus-api-dev/references/formula-design-guide-llm.md`
- `../../docyrus-api-dev/references/query-guide.md`

## Package READMEs

Full, mirrored READMEs of the core Docyrus frontend packages. Each is published on npm and lives in `docyrus-devkit/packages/*`:

- `signin-readme.md` — `@docyrus/signin`: auth provider, `useDocyrusAuth`/`useDocyrusClient`, standalone/iframe/Electron/React Native/Next.js SSR modes, host bridge, roles & permissions. Note: signin auto-fetches the signed-in user from `/v1/users/me`.
- `app-utils-readme.md` — `@docyrus/app-utils`: tenant preferences, date/number formatting, template/formula engine, app/user config, data views/forms, data-source metadata, and the **Inventory Cache** (`createInventoryClient` + `load()` warm-up after sign-in).
- `devtools-readme.md` — `@docyrus/devtools`: in-app developer panel, request/console/iframe-message instrumentation, OpenAPI request explorer, DOM element picker, and the CDP/agent in-page API.

## UI Design References

Read these files for component selection, design patterns, and UI library guidance:

- `preferred-components-catalog.md`
- `component-selection-guide.md`
- `icon-usage-guide.md`

## How to Use This Skill

1. Start with `../SKILL.md` for the merged operating guidance.
2. Use the app-development references when working on auth, queries, collections, routing, and mutations.
3. Use the UI-design references when selecting components, designing layouts, or implementing polished feature flows.
4. Combine both when building end-to-end Docyrus React features.
