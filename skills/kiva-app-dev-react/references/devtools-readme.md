<!--
Mirror of the published `@kivacore/devtools` package README (npm: `@kivacore/devtools`,
source: `kiva-devkit/packages/devtools`). Kept here as the authoritative dev-tooling
reference for this skill. If the package README changes, refresh this file.
-->

# @kivacore/devtools

React-first developer tools for KIVA apps.

## Features

- Floating KIVA trigger button with configurable corner placement
- Bottom-docked panel with `Network`, `Errors`, `Issues`, `Explore`, `Console`, and `Messages` tabs
- Request instrumentation for `@kivacore/api-client`, `@kivacore/addin-client`, and `@kivacore/app-client`
- Duplicate request and slow request detection
- Optional TanStack Query issue tracking
- Optional KIVA `fetch` capture
- OpenAPI-driven request explorer for replaying and testing endpoints with the app's existing authenticated API client
- Console capture for `log`, `info`, `warn`, `error`, `debug`, plus `window error` and `unhandledrejection`
- Iframe message capture for both directions (host → app via `window.message`, app → host via the `__KIVA_DEVTOOLS_MESSAGE_TAP__` global hook)
- Iframe/host bridge for sending errors, issues, and console entries to a host assistant panel
- In-page API for external tooling and Chrome DevTools Protocol agents to inspect current errors, issues, and console state
- Inline DOM element picker with per-element feedback collection — works in standalone browser windows without an iframe/host (uses `data-component-name` / `data-component-path` / `data-current-file-path` annotations written by `@kivacore/dom-selector-client`)

## Install

```bash
pnpm add @kivacore/devtools
```

## LLM Install Prompt

Use this prompt with a coding assistant if you want it to install and wire `@kivacore/devtools` for you:

```text
Install and integrate @kivacore/devtools into this project.

Requirements:
- Add @kivacore/devtools as a dependency.
- Wrap the React app with <KivaDevtools /> near the root.
- If the app uses TanStack Query, pass the existing queryClient to KivaDevtools.
- If the app uses @kivacore/signin and useKivaClient(), add a small component that calls useRegisterKivaClient(useKivaClient()) after mount.
- If an OpenAPI spec is already served by the app, pass its path with openApiSpecPath.
- Preserve the existing app structure and styling.
- Do not add unrelated refactors.
- After integration, run the app/package build or typecheck and summarize what changed.
```

## React Usage

```tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { KivaDevtools, useRegisterKivaClient } from '@kivacore/devtools';
import { KivaAuthProvider, useKivaClient } from '@kivacore/signin';

const queryClient = new QueryClient();

function RegisterClient() {
  const client = useKivaClient();

  useRegisterKivaClient(client);

  return null;
}

export function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <KivaDevtools
        queryClient={queryClient}
        openApiSpecPath="/openapi-spec.json">
        <KivaAuthProvider
          apiUrl="https://api.kiva.app"
          clientId="your-client-id">
          <RegisterClient />
          <YourRoutes />
        </KivaAuthProvider>
      </KivaDevtools>
    </QueryClientProvider>
  );
}
```

## API

### `KivaDevtools`

- `enabled?: boolean`
- `activationShortcut?: string | false`
- `clients?: Array<RestApiClient | KivaClient | AppClient | null | undefined>`
- `queryClient?: QueryClient`
- `buttonPosition?: 'bottom-left' | 'bottom-right' | 'top-left' | 'top-right'`
- `slowThresholdMs?: number`
- `maxEntries?: number`
- `captureFetch?: boolean`
- `captureConsole?: boolean`
- `captureIframeMessages?: boolean`
- `iframeMessageIgnoreTypes?: string[]`
- `enableHostBridge?: boolean`
- `enableElementSelector?: boolean`
- `kivaOrigins?: string[]`
- `hostOrigin?: string`
- `openApiSpecPath?: string`
- `onFeedback?(submission: DevtoolsFeedbackSubmission): void`

Defaults:

- `enabled`: `true`
- `activationShortcut`: `false`
- `buttonPosition`: `'bottom-left'`
- `slowThresholdMs`: `1000`
- `maxEntries`: `200`
- `captureFetch`: `false`
- `captureConsole`: `true`
- `captureIframeMessages`: `true`
- `iframeMessageIgnoreTypes`: `['DOCY_DEVTOOLS']`
- `enableHostBridge`: `true`
- `enableElementSelector`: `true`

## Enabling / Disabling

`enabled` gates everything: the floating button, the overlay, and all instrumentation (clients, query client, fetch, console, host bridge). When `false`, `<KivaDevtools>` only renders its children — no listeners are attached and no UI is mounted. Wire it to your build target to keep devtools out of production:

```tsx
<KivaDevtools enabled={import.meta.env.DEV}>
  <App />
</KivaDevtools>
```

`activationShortcut` lets you ship devtools in production but keep them dormant until a user opts in with a keyboard chord. The shortcut listener is attached even when `enabled={false}`; triggering it flips an internal flag that activates devtools alongside the prop. Triggering again deactivates them.

```tsx
<KivaDevtools
  enabled={import.meta.env.DEV}
  activationShortcut="shift+meta+space space">
  <App />
</KivaDevtools>
```

Shortcut syntax:

- Combine modifiers and a key with `+`: `shift+meta+space`, `ctrl+alt+d`.
- Separate sequential chords with whitespace for chord sequences: `shift+meta+space space` requires holding `Shift+⌘` and pressing `Space` twice within ~600ms.
- Modifier aliases: `meta` / `cmd` / `command`, `alt` / `option`, `ctrl` / `control`.

### `useRegisterKivaClient(client)`

Registers a KIVA client after mount so auth-managed clients can be instrumented as soon as they exist.

## DOM Element Picker

A small circular target button is rendered at the bottom-right of the main devtools trigger. Clicking it enables a crosshair overlay that highlights the element under the pointer. Clicking an element opens a popover that reads the `data-component-name`, `data-component-path`, and `data-current-file-path` attributes (emitted by `@kivacore/dom-selector-client/babel-plugin-component-annotate`) plus a generated unique CSS selector. Users can type per-element feedback and either send immediately, collect more elements (`Continue`), or dismiss.

When `onFeedback` is provided, the collected items are passed to that callback. Otherwise the submission is forwarded to the parent window via the host-bridge `sendToCody` action (same schema used by KIVA clients embedding the app in an iframe/webview):

```ts
{
  type: 'DOCY_DEVTOOLS',
  action: 'sendToCody',
  payload: {
    items: DevtoolsFeedbackItem[],
    route: string,
    url: string,
    timestamp: number
  }
}
```

Pass `enableElementSelector={false}` to hide the picker trigger.

## Host Bridge

When running inside an iframe, devtools can send structured events to the parent window with `postMessage`.

Outgoing messages use:

```ts
{
  type: 'DOCY_DEVTOOLS',
  action: 'sendToCody',
  payload: {
    scope: 'errors' | 'issues' | 'console' | 'error' | 'issue' | 'console-entry',
    items: unknown[],
    route: string,
    url: string,
    timestamp: number
  }
}
```

The host can also request data from the iframe with:

```ts
window.postMessage(
  { type: 'DOCY_DEVTOOLS', action: 'getState', requestId: '1' },
  '*'
);
```

Supported host request actions:

- `getState`
- `getErrors`
- `getIssues`
- `getConsoleEntries`

## Iframe Messages

The `Messages` tab logs `postMessage` traffic on both sides of the iframe / WebView boundary so you can debug the host ↔ app protocol used by `@kivacore/signin` (and any other custom messages your app exchanges with its host).

- **Inbound (host → app)** is captured automatically by listening on `window`'s `message` event.
- **Outbound (app → host)** is captured via a global tap. `@kivacore/signin`'s `postToHost` already calls it. If you send your own `postMessage` payloads, call the tap from your sender so devtools can record them too:

```ts
declare global {
  interface Window {
    __KIVA_DEVTOOLS_MESSAGE_TAP__?: (
      direction: 'in' | 'out',
      payload: unknown,
      meta?: {
        transport?: 'iframe' | 'webview' | 'window';
        origin?: string;
        targetOrigin?: string;
      }
    ) => void;
  }
}

window.__KIVA_DEVTOOLS_MESSAGE_TAP__?.(
  'out',
  payload,
  { transport: 'iframe', targetOrigin: hostOrigin }
);
window.parent.postMessage(payload, hostOrigin);
```

Each entry records direction, transport (`iframe` / `webview`), `type` (auto-extracted from `payload.type` when present), origin/targetOrigin, payload, and timestamp.

Devtools' own host-bridge messages (`type: 'DOCY_DEVTOOLS'`) are filtered out by default — override with `iframeMessageIgnoreTypes`.

## Agent / CDP Access

Devtools also exposes an in-page API so external tools can read the current state without UI interaction:

```ts
window.__KIVA_DEVTOOLS__?.getState();
window.__KIVA_DEVTOOLS__?.getErrors();
window.__KIVA_DEVTOOLS__?.getIssues();
window.__KIVA_DEVTOOLS__?.getConsoleEntries('error');
```

This is intended for development tooling, host integrations, and Chrome DevTools Protocol-based assistants.
