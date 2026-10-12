# Plan

Write TanStack Start server functions as Effect 4 programs. The app keeps one runtime, and every call:
- gets typed errors that reach the client as the same class;
- validates its input with Schema;
- stops its work when the request is aborted.

The findings behind each decision are in [RESEARCH.md](RESEARCH.md).

## API for version 0.1

### One app, created once

```ts
// src/effect.ts
import { EffectStart } from "@tarekkharsa/effect-tanstack-start"

export const app = EffectStart.make({
  id: "app",               // the key for the HMR-safe runtime on globalThis
  layer: AppLive,          // your services
  errors: [NotFound, UpstreamUnavailable], // classes that cross the wire
})
```

### A server function, as a handler adapter

Start's compiler only splits `.handler()` on its own builder, so the builder stays Start's:

```ts
import { createServerFn } from "@tanstack/react-start"
import { EffectStart } from "@tarekkharsa/effect-tanstack-start"

export const getProfile = createServerFn({ method: "GET" })
  .validator(EffectStart.schema(ProfileInput)) // Standard Schema, errors: "all"
  .handler(
    app.handler("member.profile", ({ data }) =>
      Effect.gen(function* () {
        const profile = yield* Profiles.get(data.id) // can fail with NotFound
        yield* ResponseHeaders.set("cache-control", "private, max-age=60")
        return profile
      }),
    ),
  )
```

### Typed errors on the client

```ts
// src/start.ts
export const startInstance = createStart(() => ({
  serializationAdapters: app.serializationAdapters, // one adapter per error class
}))

// anywhere on the client
try {
  await getProfile({ data: { id } })
} catch (error) {
  if (error instanceof NotFound) showNotFound() // the class survives
}
```

### What `app.handler` does on every call

1. Reads the request once (`getRequest()`) and provides it as the `ServerRequest` service.
2. Runs the program with `runtime.runPromiseExit(program, { signal: request.signal })`, wrapped in `Effect.withSpan(name, { kind: "server" })` plus a call counter and a timer.
3. Applies the `ResponseHeaders` buffer and the status after the fiber ends, back in the handler's own async context.
4. Maps the `Exit`:

| Exit | Response |
|---|---|
| Success | Returns the value |
| A failure in `errors` | Sets the status (500 by default; overridable per class) and throws the instance, which the adapter carries to the client |
| A failure not in `errors`, or a defect | Logs the full cause, sets 500, and throws a generic `InternalError` with no details |
| An interruption | Sets 499 and throws `RequestAborted` |

## Out of scope for 0.1

- Bridging Start middleware context to Effect services. That's 0.2.
- A `Result` or `Exit` form of the client call, so a caller can skip try/catch. That's 0.2.
- Trace propagation from incoming headers, through `HttpTraceContext`. That's 0.2.
- Streaming responses, and `Response` passthrough beyond what Start already does.
- Solid and Vue examples. The core is shared, so this is examples only, in 0.3.

## Milestones

Each milestone ends in a check that proves it.

### M0. Repository and toolchain

TypeScript, tsdown for compiled ESM, Vitest with `@effect/vitest`, and an `examples/react` TanStack Start app.

- Peer dependencies:
  - `effect: ^4.0.3`
  - `@tanstack/react-start: >=1.168 <2`, tightened once tests run against it

**Proof:** `pnpm build` produces `.js` and `.d.ts`, and CI runs on Node 24.

### M1. The runtime registry

`EffectStart.make` creates its runtime lazily and stores it under `globalThis[Symbol.for("effect-tanstack-start/<id>")]`, with a shared `Layer.makeMemoMapUnsafe()`. It disposes the runtime on `import.meta.hot.on("vite:beforeFullReload")`.

**Proof:** a dev test edits a server module 20 times and counts live runtimes. The count stays at 1. A finalizer log shows each old runtime disposed.

### M2. The handler: run and cancel

`app.handler(name, fn)`: the request service, `runPromiseExit` with the signal, the span and the metrics.

**Proof:**
- A test where the client aborts a call that sleeps for 5 s. The fiber's finalizer runs within 100 ms, and the server logs no "success".
- The same test run on Node through srvx.
- For Cloudflare: documented, and tested with the flag set.

### M3. Typed errors

The `errors` list, `app.serializationAdapters` built from each class's Schema with `Schema.toCodecJson`, the status mapping, `InternalError` and `RequestAborted`.

**Proof:** in the example app, a client `catch` sees `instanceof NotFound === true`, with its fields intact, and HTTP status 404. A defect reaches the client as `InternalError`, with no message or stack leaked.

### M4. Input validation

`EffectStart.schema(S, options?)`: Standard Schema, `errors: "all"`, and a typed `InputError` carrying the issues.

**Proof:**
- A bad payload reaches the client as `instanceof InputError`, with every issue listed and status 400.
- A raw schema passed by mistake is caught by the type checker or a dev warning, not a silent `undefined`.

### M5. Concurrency and async context

**Proof:** 200 concurrent calls with different request headers each read their own `ServerRequest` and set their own response header. Zero cross-talk.

### M6. Release 0.1.0

README with the API, a migration example from a hand-written `runPromise` handler, and Changesets. Publish to npm under `@tarekkharsa/effect-tanstack-start`.

**Proof:** the published package installs into a fresh `create-start` app and its example call works.

## Decisions taken

- **A handler adapter, not a builder.** The Start compiler only recognises its own `createServerFn` builder.
- **Stable Effect APIs only on the core path.** `effect/http` and `effect/rpc` are unstable in 4.0, so they serve as reference patterns, not dependencies.
- **The request is read once.** Effect fibers may run outside the request's async context, so the library never calls Start's request helpers from inside a fiber.
- **Errors travel as class instances through serialization adapters,** not as an encoded `Exit` value. That keeps the client code idiomatic: `try`/`catch` and `instanceof`. The `Exit` form arrives as an opt-in in 0.2.
- **Small surface, compiled output.** The archived predecessor shows the upkeep cost.

## Open questions

1. Does a call from a client middleware's `sendContext` need to reach Effect as a service in 0.1, or can it wait for 0.2?
2. Should status codes come from a Schema annotation on the error class, or from an `errors` map in `make`?
3. Should the package also export `@effect/vitest` helpers for handler tests, or only document the pattern?
4. Does a builder factory traced through `node_modules` actually pass Start's compiler? If yes, a 0.2 `app.serverFn(...)` builder becomes possible.

## Ownership

This is an independent open-source project, written from scratch against the public Effect and TanStack APIs and their source. Its design comes from experience running Effect behind TanStack Start in production. No code is copied from any private codebase.
