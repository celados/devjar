---
type: Proposal
title: Agent-Friendly Feedback and Framework Adapters
status: draft
version: 0.1
timestamp: 2026-06-28T00:00:00+08:00
description: >
  Proposes a structured feedback buffer and framework adapter boundary for
  Devjar's browser live preview runtime.
tags: [agent-feedback, runtime, frameworks]
---

# Agent-Friendly Feedback and Framework Adapters

## Context

Devjar's current runtime is optimized for a React browser preview: files are
transformed in a worker, local imports are rewritten into browser-loadable
module URLs, and the iframe renders the default export from `index.js`.

The current source establishes these constraints:

- `src/transform-worker.ts` imports `oxc-transform` dynamically and calls
  `transformSync` for every changed non-CSS file.
- `src/core.ts` requests that worker through `resolveModule('oxc-transform')`.
- `src/core.ts` rewrites imports, auto-adds React when needed, and executes an
  iframe-local `__render__`.
- `src/module.ts` imports `react-refresh/runtime` and imports the generated
  `index` data URL as the entry module.
- `src/render.tsx` exposes only the latest `error` through `useLiveCode` and
  `<DevJar onError>`.

This is enough for human-visible preview failures. It is not enough for the
intended agent loop: an agent writes a coherent batch of files, Devjar previews
that revision, and the host needs a compact, structured feedback packet to hand
back to the LLM.

## Goals

- Represent preview failures as structured events, not a single latest `Error`.
- Keep a bounded feedback buffer keyed by preview revision.
- Emit feedback when a file batch settles, not on every text edit.
- Separate framework-specific render and refresh behavior from the generic file
  graph and transform pipeline.
- Preserve Devjar's small-tool shape: no dev server, no bundler daemon, no
  compatibility shim for the current React-only contract unless a real consumer
  requires it.

## Non-Goals

- Do not build a full Vite replacement.
- Do not add server-side compilation.
- Do not support arbitrary framework file formats by pretending JSX transform
  options are enough. Vue SFC and Svelte require their own compilers.
- Do not stream noisy intermediate diagnostics for human keystrokes. The target
  writer is an agent that usually applies atomic or near-atomic file batches.

## Current Gaps

### Feedback Is Not Structured

The worker compresses OXC diagnostics into one thrown `Error`. Filename, phase,
severity, location, and all secondary diagnostics are lost before the host can
decide what to send to a model.

### React Boundary Errors Stay Inside The Iframe

The React error boundary renders `error.message` into the iframe. It does not
post a structured event to the parent, so the host may miss the error even when
the preview visibly failed.

### Runtime Channels Are Missing

There is no capture for:

- `window.error`
- `window.unhandledrejection`
- iframe console errors and warnings
- failed dynamic imports
- module graph failures such as missing local modules or circular local imports

Some of those failures currently become thrown errors during `load()`, but they
arrive as plain exceptions without a stable phase or source identity.

### Framework Code Is Entangled With Generic Runtime Code

React assumptions are spread across the transform worker, import rewriting,
iframe renderer, module creation, and public API. That makes React support work,
but it gives no clean place to add Vue, Svelte, Solid, or Preact without
duplicating the whole runtime or adding broad conditionals.

## Proposed Feedback Contract

Introduce a first-class diagnostic event:

```ts
export type DevJarDiagnostic = {
  id: string
  revision: number
  phase:
    | 'transform'
    | 'module-graph'
    | 'import'
    | 'render'
    | 'runtime'
    | 'console'
  severity: 'error' | 'warning' | 'info'
  message: string
  filename?: string
  line?: number
  column?: number
  codeframe?: string
  stack?: string
  source?: 'worker' | 'iframe' | 'host'
  timestamp: number
}
```

Expose a bounded buffer:

```ts
export type DevJarFeedbackSnapshot = {
  revision: number
  status: 'pending' | 'ok' | 'failed'
  diagnostics: DevJarDiagnostic[]
}
```

`useLiveCode` should return this snapshot alongside the current `error` during
the transition:

```ts
const { ref, feedback, load } = useLiveCode(options)
```

For a clean break, `error` can be removed in the next breaking release and
replaced by `feedback.status` plus `feedback.diagnostics`.

## Feedback Timing

The timing model should follow agent writes, not human typing.

Each `load(files)` call creates one monotonic preview `revision`. The runtime
collects diagnostics for that revision until the preview reaches a terminal
state:

- `ok`: transform, module graph, dynamic imports, and render completed.
- `failed`: a terminal error prevented a valid render.
- `pending`: work is still in flight, or this revision was superseded.

The host should notify the LLM only when the latest revision settles. If a new
`load(files)` starts before the previous one settles, the older revision becomes
superseded and should not be sent as feedback unless the host explicitly asks
for historical diagnostics.

This avoids stale feedback without relying on edit debouncing.

## Feedback Buffer Rules

- Keep only the last N revisions; default N should be small.
- Deduplicate identical diagnostics within one revision by phase, filename,
  location, and message.
- Preserve ordering by occurrence time.
- Prefer source diagnostics over wrapper errors. For example, an OXC codeframe
  is more useful than `Error: transform failed`.
- Mark warnings as warnings. Do not collapse every diagnostic into failure.
- On successful render, clear active failure state but keep the previous
  revision in history until the ring buffer evicts it.

## Runtime Capture Points

### Transform Worker

Return structured diagnostics instead of throwing after the first OXC error.
The worker should include all OXC errors and warnings it can extract safely:

```ts
type TransformResult = {
  transformed: Record<string, string>
  diagnostics: DevJarDiagnostic[]
}
```

If any transform diagnostic is severity `error`, the revision fails before
module graph execution.

### Module Graph

`createModule` should report structured `module-graph` diagnostics for missing
local modules and circular local imports. These are currently thrown as generic
errors, but they are source-contract errors that an agent can fix directly.

### Iframe Runtime

Install an iframe-local feedback bridge before executing user code:

- capture `error`
- capture `unhandledrejection`
- wrap `console.error` and `console.warn`
- allow framework adapters to report render boundary errors

The bridge should post sanitized diagnostics to the parent runtime. Do not pass
raw objects across the boundary; normalize to message, stack, and primitive
metadata.

### Render Completion

Dispatch a successful render event only after framework rendering has either
committed or declared itself complete. The existing `devjar:render` event can
remain as an internal signal, but the public contract should be the feedback
snapshot.

## Proposed Framework Boundary

Split the runtime into two layers:

1. Generic Devjar runtime
2. Framework adapter

The generic runtime owns:

- file map normalization
- changed-file detection
- transform request orchestration
- local import resolution and dependency graph invalidation
- CSS module handling
- iframe setup
- feedback buffer
- diagnostic bridge

The adapter owns:

- transform options for framework syntax
- implicit imports, if any
- framework runtime imports
- root mount behavior
- refresh behavior
- render error capture
- cleanup behavior

Suggested adapter shape:

```ts
export type DevJarAdapter = {
  name: string
  transform?(input: {
    filename: string
    source: string
    oxc: unknown
  }): Promise<{
    code: string
    diagnostics: DevJarDiagnostic[]
  }>
  rewriteImports?(input: {
    filename: string
    moduleKey: string
    code: string
    resolveModule: (specifier: string) => string
    localModules: Set<string>
  }): {
    code: string
    dependencies: string[]
  }
  createRenderer(input: {
    root: HTMLElement
    resolveModule: (specifier: string) => string
    report: (diagnostic: DevJarDiagnostic) => void
  }): Promise<DevJarRenderer>
}

export type DevJarRenderer = {
  render(entryModule: unknown, revision: number): Promise<void>
  refresh?(changedModules: Set<string>, revision: number): Promise<boolean>
  dispose?(): void
}
```

The exact API can be tightened during implementation, but the boundary should
keep React's renderer out of the generic module graph.

## Adapter Expectations

### React Adapter

React becomes the first adapter and preserves today's behavior:

- JSX automatic runtime
- React Refresh
- default export from `index.js` as the component
- `react-dom/client` root
- error boundary that reports diagnostics to the parent bridge

This is not a compatibility layer; it is moving the current behavior behind an
explicit adapter.

### Preact Adapter

Preact is the lowest-risk second adapter because it can share the browser module
graph shape and JSX mental model. It still needs explicit JSX import source and
render semantics.

### Solid Adapter

Solid needs its own JSX transform behavior. It should not be treated as React
with different imports unless the selected compiler path proves that contract.

### Vue Adapter

Vue support should be scoped around whether `.vue` files are required.

- If only plain `.js/.ts` modules using Vue runtime APIs are supported, the
  adapter can mount an exported component.
- If `.vue` SFC support is required, Devjar needs a Vue compiler path. OXC
  alone is not the right abstraction for SFC compilation.

### Svelte Adapter

Svelte requires a Svelte compiler path. It should not be added through generic
JSX transform options.

## Public API Direction

Prefer explicit adapter selection:

```tsx
<DevJar
  adapter="react"
  files={files}
  resolveModule={resolveModule}
  onFeedback={(snapshot) => {
    sendLatestSettledRevisionToAgent(snapshot)
  }}
/>
```

Allow custom adapters for advanced users:

```tsx
<DevJar
  adapter={customAdapter}
  files={files}
  resolveModule={resolveModule}
/>
```

Avoid adding framework-specific booleans such as `react`, `vue`, `svelte`, or
`enableRefresh`. They create a weak API surface and push framework behavior back
into generic code.

## Implementation Order

1. Add `revision` and feedback buffer to `useLiveCode`.
2. Change transform worker responses to return structured diagnostics.
3. Add iframe diagnostic bridge for runtime, console, and unhandled rejection
   events.
4. Move current React behavior into a React adapter without changing behavior.
5. Expose `adapter="react"` as the default.
6. Add one non-React adapter only after the adapter boundary survives React.

## Open Questions

- Should console warnings be included in the default LLM feedback packet, or
  retained only in the buffer for inspection?
- Should successful revisions emit a compact positive signal to the agent, or
  should silence mean success?
- Should transform diagnostics preserve OXC's raw diagnostic code when
  available?
- Should the public API keep `onError` for one release, or make the feedback
  contract a clean breaking change?

## Review Notes

The most important design choice is the feedback timing. Devjar should not
optimize for human keystrokes here. The primary loop is:

1. Agent writes a file batch.
2. Host calls `load(files)`.
3. Devjar settles the latest revision.
4. Host sends one compact feedback snapshot to the LLM.

That loop needs deterministic revision semantics more than it needs debouncing.
