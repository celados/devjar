---
type: Proposal
title: Agent-Friendly Feedback and Framework Adapters
status: draft
version: 0.2
timestamp: 2026-06-28T00:00:00+08:00
description: >
  Proposes a structured feedback buffer and framework adapter boundary for
  Devjar's browser live preview runtime, with explicit execution-domain and
  source-mapping constraints.
tags: [agent-feedback, runtime, frameworks]
---

# Agent-Friendly Feedback and Framework Adapters

## Context

Devjar's current runtime is optimized for a React browser preview: files are
transformed in a worker, local imports are rewritten into browser-loadable
module URLs, and the iframe renders the default export from `index.js`.

The current source establishes these constraints:

- `src/transform-worker.ts` imports `oxc-transform` dynamically and calls
  `transformSync` for each file the host hands it (the host pre-filters to
  changed non-CSS files in `core.ts`'s `load`).
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

The worker compresses OXC diagnostics into one thrown `Error`
(`transform-worker.ts:48`): it keeps only the first `severity === 'Error'` and
drops filename, phase, location, warnings, and every secondary diagnostic before
the host can decide what to send to a model.

### React Boundary Errors Stay Inside The Iframe

The React error boundary renders `error.message` into the iframe
(`core.ts:145`). It does not post a structured event to the parent, so the host
may miss the error even when the preview visibly failed.

### Runtime Channels Are Missing

There is no capture for:

- the `error` event on `window`
- the `unhandledrejection` event on `window`
- iframe console errors and warnings
- failed dynamic imports
- module graph failures such as missing local modules or circular local imports

Some of those failures currently become thrown errors during `load()`, but they
arrive as plain exceptions without a stable phase or source identity.

### Framework Code Is Entangled With Generic Runtime Code

React assumptions are spread across the transform worker (`jsx.runtime` and
`refresh` in `transform-worker.ts:40`), import rewriting (auto-injected
`import React`, `core.ts:99`), iframe renderer (`core.ts:111`), module creation
(`react-refresh/runtime`, `module.ts:111`), and public API. That makes React
support work, but it gives no clean place to add Vue, Svelte, Solid, or Preact
without duplicating the whole runtime or adding broad conditionals.

## Proposed Feedback Contract

Introduce a first-class diagnostic event. There is no synthetic `id`: a
diagnostic's identity is its `(phase, filename, location, message)` tuple
(see Feedback Buffer Rules), and `severity` is only `error` or `warning`
because the capture points only produce those two.

```ts
export type DevJarDiagnostic = {
  revision: number
  phase:
    | 'transform'
    | 'module-graph'
    | 'import'
    | 'render'
    | 'runtime'
    | 'console'
  severity: 'error' | 'warning'
  message: string
  filename?: string
  line?: number
  column?: number
  codeframe?: string
  stack?: string
  source: 'worker' | 'iframe' | 'host'
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

## Revision Identity

Three counters already exist with overlapping names but different scopes. The
feedback contract reuses the first and must not collapse them into one:

| Counter                     | Location      | Scope                    | Role                                      |
| --------------------------- | ------------- | ------------------------ | ----------------------------------------- |
| `loadIdRef`                 | `core.ts:331` | one per `load()` call    | the public preview revision; supersession |
| `createRenderer`'s `revision` | `core.ts:115` | one per React (re)mount  | resets the error boundary on remount      |
| `runtime.revision`          | `module.ts:38`| one per `createModule`   | module-graph bookkeeping                  |

`DevJarDiagnostic.revision` is `loadIdRef`. The other two are internal and must
not leak into the feedback contract. The implementation should promote
`loadIdRef` to a public revision rather than introduce a fourth counter.

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
for historical diagnostics. This avoids stale feedback without relying on edit
debouncing.

Three subtleties the implementation must respect:

**Supersession already exists.** `load()` increments `loadIdRef`
(`core.ts:331`), and the `loadId !== loadIdRef.current` guards (`core.ts:350`,
`core.ts:396`) already drop superseded work. Build the revision lifecycle on top
of these, not beside them.

**`ok` is not "`render()` returned".** `reactRoot.render()` (`core.ts:155`,
`core.ts:170`) schedules a concurrent render and returns before React commits,
so the current `devjar:render` dispatch (`core.ts:397`) fires pre-commit. A
trustworthy `ok` requires the renderer to signal a real commit — a mount-time
`report`, or `onRecoverableError` for the failure path — not the resolution of
`render()`. This belongs to the adapter's renderer contract; the generic runtime
cannot infer it.

**Runtime diagnostics outlive settlement.** `transform`, `module-graph`, and
`render` reach a terminal state deterministically. `runtime` and `console` do
not: an error thrown from an effect or event handler can arrive long after a
revision settled `ok`. The attribution rule must be explicit — a runtime/console
diagnostic attaches to whichever revision is current when it occurs and updates
that revision's already-emitted snapshot, so a late failure can flip a settled
`ok` to `failed`. The host must treat a snapshot as revisable until the revision
is evicted, not frozen at first settle.

## Feedback Buffer Rules

- Keep only the last N revisions. The agent loop only needs the latest settled
  revision, so default N to 1; raise it only when a host wants to inspect
  superseded revisions.
- Deduplicate identical diagnostics within one revision by
  `(phase, filename, location, message)`. That tuple is the identity; there is
  no synthetic `id`.
- Preserve ordering by occurrence time.
- Prefer source diagnostics over wrapper errors. For example, an OXC codeframe
  is more useful than `Error: transform failed`.
- Mark warnings as warnings. Do not collapse every diagnostic into failure.
- On successful render, clear active failure state but keep the previous
  revision in history until the ring buffer evicts it.

## Source Mapping

`transform` and `module-graph` diagnostics carry locations for free: OXC emits
codeframes, and `createModule` knows the offending `moduleKey`. `runtime` and
`console` diagnostics do not, and two facts in the current runtime erase their
source identity:

- `transform-worker.ts:45` sets `sourcemap: false`.
- Modules execute as `data:text/javascript,…` URLs (`module.ts:16`), so stack
  frames point at opaque data URLs with post-transform line numbers.

To populate `filename` / `line` / `column` / `stack` for the `runtime` and
`console` phases, the runtime must:

1. Enable sourcemaps in the transform worker, carry them through `createModule`,
   and map captured stack frames back to source positions in the bridge before
   posting.
2. Maintain a reverse index from generated `data:` URL to `moduleKey`.
   `runtime.urls` (`module.ts:37`) is `moduleKey → url`; invert it so a stack
   frame's URL resolves to a source filename.

Without both, the `runtime`-phase fields are structurally present but empty,
which defeats Goal 1 for exactly the errors agents most need to act on. Treat
runtime-location support as its own milestone, separate from transform
diagnostics, which need neither step.

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
local modules (`module.ts:74`) and circular local imports (`module.ts:88`).
These are currently thrown as generic errors, but they are source-contract
errors that an agent can fix directly.

### Iframe Runtime

Install an iframe-local feedback bridge before executing user code:

- capture the `error` event
- capture the `unhandledrejection` event
- wrap `console.error` and `console.warn`
- allow framework adapters to report render boundary errors

The bridge should post sanitized diagnostics to the parent runtime. Do not pass
raw objects across the boundary; normalize to message, stack, and primitive
metadata.

### Render Completion

Dispatch a successful render event only after framework rendering has actually
committed — not when `render()` returns (see Feedback Timing). The renderer must
report commit explicitly; the existing `devjar:render` event can remain as an
internal signal, but the public contract is the feedback snapshot reaching
`ok`.

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

- transform options (or a named compiler) for framework syntax
- implicit imports, if any
- framework runtime imports
- root mount behavior
- refresh behavior
- render error capture
- cleanup behavior

### Execution Domains

This is the constraint the boundary lives or dies by. Devjar already runs code
across three domains, and an adapter member must declare which domain it runs in
because **functions cannot be `postMessage`'d into the worker, and code that
runs in the iframe is `.toString()`'d, so it cannot close over outer state.**

| Adapter member          | Domain            | Crosses as                                   | Constraint                                                           |
| ----------------------- | ----------------- | -------------------------------------------- | ------------------------------------------------------------------- |
| `name`                  | —                 | string                                       | —                                                                   |
| `transform.oxcOptions`  | transform worker  | structured clone                             | must be serializable; no functions                                  |
| `transform.compiler`    | transform worker  | module specifier, `import()`'d in the worker | the compiler must be importable in worker scope                     |
| `rewriteImports`        | main thread       | live function                                | runs alongside `es-module-lexer` (`core.ts:46`); may close over state |
| `createRenderer`        | iframe            | `.toString()` → data: script (`core.ts:182`) | self-contained: no outer closure; deps only via `resolveModule`     |
| `DevJarRenderer.report` | iframe → parent   | structured clone via `window.parent.__devjar__` (`core.ts:184`) | primitives only                                  |

The consequence for the public API: a custom adapter may supply a live
`rewriteImports`, serializable `transform` options, and a self-contained
`createRenderer`. It may **not** supply a live `transform` function that expects
to run inside the worker — frameworks OXC cannot handle (Vue SFC, Svelte) must
name a `compiler` module that the worker imports, not pass a closure.

Suggested adapter shape:

```ts
export type DevJarAdapter = {
  name: string

  // Declarative + structured-cloneable → crosses into the transform worker by
  // value. A live transform() function cannot be postMessage'd into the worker.
  transform: {
    oxcOptions?: Record<string, unknown>
    compiler?: string // module specifier, import()'d inside the worker
  }

  // Runs on the main thread next to es-module-lexer (core.ts:46).
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

  // .toString()'d into the iframe bootstrap, like the current createRenderer
  // (core.ts:182). MUST be self-contained: no outer closure; reach dependencies
  // only through resolveModule; reach the parent only through report.
  createRenderer:
    | string
    | ((input: {
        root: HTMLElement
        resolveModule: (specifier: string) => string
        report: (diagnostic: DevJarDiagnostic) => void
      }) => DevJarRenderer)
}

export type DevJarRenderer = {
  // Resolve only after the framework has committed — see Feedback Timing.
  render(entryModule: unknown, revision: number): Promise<void>
  refresh?(changedModules: Set<string>, revision: number): Promise<boolean>
  dispose?(): void
}
```

The exact API can be tightened during implementation, but the boundary should
keep React's renderer out of the generic module graph and honor the execution
domains above.

## Adapter Expectations

### React Adapter

React becomes the first adapter and preserves today's behavior:

- JSX automatic runtime
- React Refresh
- default export from `index.js` as the component
- `react-dom/client` root
- error boundary that reports diagnostics to the parent bridge

One open detail to carry over deliberately: the runtime currently auto-injects
`import React` when user code does not (`core.ts:99`). Under the automatic JSX
runtime that injection is not needed for JSX itself; the React adapter should
decide whether to keep it (for user code that references `React` directly) and
say so, rather than inheriting it implicitly.

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
- If `.vue` SFC support is required, Devjar needs a Vue compiler path declared
  through `transform.compiler` (loaded inside the worker). OXC alone is not the
  right abstraction for SFC compilation.

### Svelte Adapter

Svelte requires a Svelte compiler path, again via `transform.compiler`. It
should not be added through generic JSX transform options.

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

Allow custom adapters for advanced users, subject to the execution-domain
constraints in Proposed Framework Boundary:

```tsx
<DevJar
  adapter={customAdapter}
  files={files}
  resolveModule={resolveModule}
/>
```

The simplest host primitive, however, is an awaitable `load`. `load` is already
`async` (`core.ts:330`); make it resolve to the settled snapshot of the revision
it created — this is the literal shape of "notify when the latest revision
settles":

```ts
const snapshot = await load(files) // resolves when this revision is ok | failed
sendLatestSettledRevisionToAgent(snapshot)
```

`onFeedback` stays as a push subscription for hosts that also want late
runtime diagnostics after settlement (see Feedback Timing), but the pull form
matches the loop in Review Notes directly.

Avoid adding framework-specific booleans such as `react`, `vue`, `svelte`, or
`enableRefresh`. They create a weak API surface and push framework behavior back
into generic code.

## Implementation Order

1. Promote `loadIdRef` to a public `revision`, add the feedback buffer to
   `useLiveCode`, and make `load` resolve to the settled snapshot.
2. Change transform worker responses to return structured diagnostics (all OXC
   errors and warnings), not a single throw.
3. Add the iframe diagnostic bridge for the `error` and `unhandledrejection`
   events, `console.error`/`console.warn`, and failed dynamic imports.
4. Add real commit signalling so `ok` means committed, and define the
   runtime-diagnostic attribution window.
5. Move current React behavior into a React adapter without changing behavior;
   expose `adapter="react"` as the default.
6. Add sourcemap plus the `data:` URL reverse map so runtime diagnostics carry
   source locations.
7. Add one non-React adapter only after the adapter boundary survives React.

## Open Questions

- Should console warnings be included in the default LLM feedback packet, or
  retained only in the buffer for inspection? (Wrapping `console.error`/`warn`
  also captures React's own dev warnings and third-party logs.)
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
The two things most likely to break it in practice are settling `ok` before the
framework actually commits, and runtime errors that arrive after settlement —
both are addressed in Feedback Timing and must not be deferred.
