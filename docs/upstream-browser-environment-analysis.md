# Analysis: How this fork adapts upstream Playwright to run in a browser/extension environment

## Scope and framing

This repository keeps upstream Playwright as a git subtree under `playwright/` and builds a browser-compatible runtime on top of it (`src/` + Vite build). The browser-targeted runtime is not a small patch: it reuses Playwright protocol/client/server pieces, but swaps transport, environment assumptions, and Node-only dependencies.

The key enabling idea is:

1. keep using Playwright's dispatcher/channel architecture, and
2. replace the process/transport/runtime substrate so it can execute inside a Chrome Extension service worker.

## 1) Transport layer replacement (the biggest runtime change)

Upstream Playwright normally talks to browser backends over Node-side transports (pipe/WebSocket/CDP server assumptions). This fork introduces a custom `ConnectionTransport` implementation, `CrxTransport`, backed by `chrome.debugger`.

- `CrxTransport` implements Playwright's transport interface and wires debugger lifecycle events (`onEvent`, `onDetach`) plus tab lifecycle (`tabs.onRemoved`, `tabs.onCreated`).
- It maps Playwright sessions/targets to Chrome tab IDs and synthesizes the protocol messages Playwright expects (`Target.attachedToTarget` / `Target.detachedFromTarget` events).
- It special-cases CDP commands that do not work or are incomplete in extension context (e.g. `Target.setAutoAttach`, `Target.createTarget`, `Browser.getVersion`, `Storage.getCookies`, etc.) and emulates/fallbacks behavior where needed.
- It explicitly excludes service workers in auto-attach flow and has version-gated behavior for Chrome 126+ fallback target filtering.

In short: rather than launching/managing a browser process, Playwright is attached to already-running tabs via extension debugger APIs.

## 2) Browser startup model rewritten around existing tabs/contexts

`src/server/crx.ts` adapts Playwright server abstractions (`CRBrowser`, `CRBrowserContext`) to extension semantics:

- `Crx.start()` builds a `CRBrowser.connect(...)` session using the custom transport instead of a launched child process.
- Browser process hooks are mocked (`close` delegates to transport close, `kill` is a no-op) because there is no owned OS process to terminate.
- For incognito, a tab is found/created, attached first, then a `CRBrowserContext` is created from discovered `browserContextId` before target events are emitted.
- Recorder factory is overridden so recorder UI is opened by extension windows/sidepanel logic, not by upstream desktop app launch flow.

This preserves upstream Playwright internals where possible while replacing process orchestration with tab/context orchestration.

## 3) Channel/protocol surface extension (`_crx`)

This fork extends Playwright initializer/channel schema to expose a CRX-specific API:

- Adds `_crx` to `PlaywrightInitializer` and defines `Crx` / `CrxApplication` channels.
- Adds methods like `start`, `attach`, `attachAll`, `newPage`, `showRecorder`, `run`, `load`, etc.
- Adds schema validation entries so custom channels participate in normal protocol validation.

A dedicated `CrxPlaywrightDispatcher` composes regular Playwright dispatchers plus `CrxDispatcher`, allowing standard client/server object creation while adding extension features.

## 4) Runtime bootstrapping: keep Playwright dispatcher architecture, swap environment

`src/index.ts` keeps upstream-style dispatcher/client bootstrap:

- creates `DispatcherConnection` + `Connection`,
- creates root dispatcher and Playwright channel object,
- then switches to async dispatch once initialized.

But before doing so it imports browser shims and validator overrides, and injects `CrxPlaywright` instead of upstream `Playwright`. This is the compatibility bridge that lets most of upstream protocol machinery still run.

## 5) Node builtins/polyfills strategy (critical for browser execution)

The Vite config contains the large compatibility surface:

- aliases Playwright source imports directly to `playwright/packages/.../src`,
- maps Node builtins (`fs`, `path`, `net`, `tls`, `child_process`, `process`, etc.) to browser packages or local shims,
- configures bundle-specific aliasing for Playwright bundle implementations,
- rewrites `__dirname` for Playwright source subtrees,
- avoids resolving known Node-only entrypoints in CommonJS conversion.

This is effectively a custom “browserification” profile for Playwright internals.

## 6) Concrete shim behaviors enabling browser runtime

Shims are not generic placeholders; they encode Playwright-specific assumptions:

- `global` shim injects `global`, `__dirname`, `Buffer`, `setImmediate`, and `fs` equivalents.
- `process` shim patches `hrtime`, sets `platform`, `versions.node`, `stdout.isTTY`, and `PLAYWRIGHT_BROWSERS_PATH` expected by upstream codepaths.
- `fs` shim uses `memfs` and pre-creates `/tmp`, enabling traces/downloads/artifacts flows that expect writable filesystem APIs.
- `child_process`, `net`, `tls`, `readline`, `chokidar`, etc. export no-op/minimal stubs to satisfy imports for codepaths that are not meaningful inside extension runtime.

This combination keeps dependency graph resolvable while allowing subset behavior actually needed by CRX usage.

## 7) What changed conceptually vs upstream

From an architectural perspective, the upstream-to-browser adaptation is made by replacing three assumptions:

1. **Process ownership assumption** (Node launches browser) → replaced by **attachment assumption** (extension controls existing tabs).
2. **Node runtime assumption** (full builtins + filesystem + child process) → replaced by **polyfilled browser runtime** (memfs + stubs + browser modules).
3. **Standard Playwright public API surface only** → extended with **`_crx` channel/API** for extension-specific lifecycle and recorder operations.

## 8) Why this approach works well with upstream updates

The fork keeps upstream Playwright code vendored under subtree and consumes it through source aliases instead of rewriting broad upstream files in place. This isolates adaptation into:

- `src/server/**` transport/runtime glue,
- `src/client/**` object wiring,
- `src/protocol/**` channel/schema extension,
- `src/shims/**` runtime compatibility,
- `vite.config.mts` bundling/alias policy.

That separation reduces merge pain when upstream updates internal behavior, but still requires periodic adjustments whenever upstream changes transport expectations, protocol schema, or builtin usage.
