# Research

What exists, what TanStack Start and Effect 4 offer, and the traps a library has to handle. Researched on 2026-10-12.

Versions read:
- **Effect:** `effect` 4.0.3, at Effect-TS/effect `8aee3b9c`. `latest` on npm is 4.0.3; 4.0.0 shipped on 2026-10-01.
- **TanStack:** `@tanstack/react-start` 1.168.61, `start-client-core` 1.170.35, `start-server-core` 1.169.40 and `router-core` 1.171.35, at TanStack/router `57bd1a92`. Also seroval 1.6.8 and srvx 1.0.5.

Line references point into those two repositories. "Measured" means a small probe was run. "Inferred" means it was read from source and not run.

## 1. The gap

No package on Effect 4.0.x lets you write a TanStack Start server function as an Effect program that:
- runs on one long-lived `ManagedRuntime` that survives hot reload;
- returns typed failures that reach the client as the same error class;
- validates input with Effect Schema;
- turns Start middleware context into Effect services;
- interrupts the fiber when the request is aborted.

### Prior art

| Package | Latest | Scope | Works on Effect 4.0.x |
|---|---|---|---|
| [`effect-tanstack-start`](https://github.com/EthanShoeDev/effect-tanstack-start) | 0.2.0 (2026-04-13); 1.0.0-beta.3 (2026-07-24); repository archived 2026-09-22 | Mounts an Effect `HttpApi` on a splat route, with an in-process client during SSR. Does not wrap `createServerFn` | No. 0.2.0 targets Effect 3. The 4.x beta imports `effect/unstable/httpapi`, which 4.0 removed (inferred from the exports map). Ships raw `.ts` |
| [`effect-start`](https://github.com/nounder/effect-start) | 0.51.0 (2026-10-06) | A separate Bun-first full-stack framework | Yes, but it is unrelated to TanStack Start |
| [`@mcrovero/effect-nextjs`](https://www.npmjs.com/package/@mcrovero/effect-nextjs) | 0.32.1 (2026-09-15) | Next.js pages and actions; `Next.make(name, layer).middleware(Tag).build(fn)` | No (peer `<4`). It throws `Cause.squash`, so typed failures do not cross the wire, and it has no cancellation |
| `effect-query`, `@tiesen/effect-tanstack-query` | 1.0.0; 0.0.26 | TanStack Query adapters | Yes, but they cover the client query layer only |
| `lucas-barake/effect-tanstack-start` (example repo) | 2025-12 | Rpc and HttpApi mounted on `api/$.ts` | No (Effect 3) |

The archived package's README lists this library's idea under "Future goals": define a `createServerFn` with an Effect generator, without wrapping it in `Effect.runPromise`.

### Maintainer stance

- **Effect** has no official framework adapter. Requests for new packages are sent to community packages:
  - [#7976](https://github.com/Effect-TS/effect/issues/7976), Tim Smart: "Feel free to make this a community package :) We don't want to take on the maintenance burden at this time."
  - [#7521](https://github.com/Effect-TS/effect/issues/7521) went the same way.
- **TanStack** has no Effect integration in its issues or discussions.

## 2. TanStack Start: what a library can hook into

### Builder and compiler

- **Builder methods.** `createServerFn({ method })` takes `.middleware([...])`, `.validator(...)` and `.handler(...)` (`start-client-core/src/createServerFn.ts:84-126`).
- **`.inputValidator` is deprecated** in favour of `.validator` ("TODO remove upon stable").
- **Handler context.** The handler receives `{ data, serverFnMeta, context, method }` (`:477-486`). `context` is the typed accumulation of the middleware contexts.
- **Constraint: the compiler only splits `.handler()` on builders traced to `createServerFn`** from `@tanstack/{react,solid,vue}-start` or `start-client-core` (`start-plugin-core/src/start-compiler/config.ts:54-74`). A library therefore hands out a **handler adapter**, not its own builder: `createServerFn().handler(app.handler(...))`. A builder factory traced through `node_modules` might work, but that is unproven.

### Validation

- **What Start accepts.** `execValidator` accepts Standard Schema (`'~standard'`), then `.parse`, then a plain function (`createServerFn.ts:893-916`).
- **Effect Schema works directly (measured).** `Schema.toStandardSchemaV1(S)` plugs straight in; `NumberFromString` decoded `{ id: "7" }` to `{ id: 7 }`. Note that it mutates and returns the same schema object.
- **Trap (measured).** A raw Effect 4 schema is a function with no `~standard` key. Start treats it as a validator function, which silently returns `undefined`. The library must always wrap schemas.
- **How validation fails.** Start throws `new Error(JSON.stringify(issues))` (`:903`). The client gets a plain `Error` with HTTP 200 (see below).

### Errors over the wire

- **Status code.** A throw in a handler or middleware is caught into `{ error }` and serialized with seroval. The status is `getResponse().status`, so it is **HTTP 200 unless the code calls `setResponseStatus`** (`start-server-core/src/server-functions-handler.ts:175-198`; inferred).
- **Class identity is lost by default (measured).** `ShallowErrorPlugin` rebuilds every `Error` as `new Error(message)` (`router-core/src/ssr/serializer/ShallowErrorPlugin.ts:12-43`). A tagged error reaches the client with `name: "Error"`, and its `_tag` and fields are gone.
- **The extension point is `createSerializationAdapter`** (`router-core/src/ssr/serializer/transformer.ts:42`), registered in `createStart(() => ({ serializationAdapters }))`. Adapters run before the router's plugins (`start-client-core/src/getSerovalPlugins.ts:10-17`). Measured: with an adapter, `instanceof` holds on the client and the fields survive. The upstream example is `e2e/react-start/serialization-adapters`.
- **`redirect()` and `notFound()`.** `redirect()` is serialized and rethrown on the client. `notFound()` answers 404.
- **No round trip during SSR.** `createSsrRpc` calls the function directly, so the original instances survive server-side.

### The request, its signal and headers

- **Helpers.** `getRequest`, `setResponseStatus`, `setResponseHeader(s)` and the cookie helpers come from `@tanstack/react-start/server` (`start-server-core/src/request-response.ts`).
- **They depend on async context.** They are backed by an `AsyncLocalStorage` stored on `globalThis`.
- **No signal in the handler.** The handler context has no `signal`, even though the middleware option type declares one (`createMiddleware.ts:531`; it is `undefined` on the server, inferred). The official pattern is `getRequest().signal` (`e2e/react-start/server-functions/src/routes/abort-signal`).
- **When the signal aborts.**
  - On Node, srvx aborts it when the response closes early (`srvx/dist/adapters/node.mjs:394-415`).
  - On Cloudflare Workers it needs the `enable_request_signal` compatibility flag.

### Lifecycle and hot reload

- **When the entry loads.** `src/start.ts` loads once and is memoized, but `getOptions` runs on every request (`createStartHandler.ts:118-137,568-575`). Never build a runtime inside `getOptions`.
- **Dev full reloads** clear the module runner and re-import the entry. `import.meta.hot.dispose` callbacks **do not run** on a full reload; only `vite:beforeFullReload` listeners fire. So a `ManagedRuntime` created at module scope leaks on every edit. Keep it on `globalThis[Symbol.for(...)]`, which is what Start does for its own storage, and dispose it on `vite:beforeFullReload`.

### Other facts

- **Payloads and methods.** GET carries the payload in `?payload=`, capped at 1 MB. FormData is POST-only.
- **Raw responses.** A handler may return a `Response`, which is passed through raw. Streams use a framed protocol.
- **Shared core across frameworks.** React, Solid and Vue Start share `start-client-core` and `start-server-core`, so one library can serve all three.
- **Churn.** `.validator` and `.inputValidator` swapped places within the 1.16x line, and the archived package's last fix was a cast against `@tanstack/react-start` 1.163+. Pin peer ranges and run the tests against the latest Start in CI.

## 3. Effect 4: building blocks to reuse

All of these are `@stability stable` unless the table says otherwise.

| API | Module | Use in the library |
|---|---|---|
| `ManagedRuntime.make(layer, { memoMap })` (`ManagedRuntime.ts:291`) | `effect/ManagedRuntime` | One app runtime. Built lazily on the first run, then cached; a call costs a fiber. `dispose()` interrupts its fibers and closes the layer scope |
| `runtime.runPromiseExit(effect, { signal })` (`Effect.ts:9033`, `RunOptions`) | `effect/ManagedRuntime` | Run each call. An aborted signal interrupts the fiber, at once if it was already aborted, and the listener is removed on exit (`internal/effect.ts:5686-5693`). Use `runPromiseExit`, because `runPromise` rejects with a squashed cause |
| `Layer.makeMemoMapUnsafe()` | `effect/Layer` | One global memo map shared across runtimes, as Effect's own docs recommend (`ai-docs/src/04_integration/10_managed-runtime.ts:60-69`); the HMR-safe pattern |
| `Effect.provideService`, `Effect.provide(layer, { local })`, `Effect.scoped` | `effect/Effect` | Per-call services and finalizers |
| `Effect.withSpan(name, { kind: "server" })`, `Effect.fn(name)` | `effect/Effect` | One span per server function call |
| `Metric.counter`, `Metric.timer` | `effect/Metric` | Call count and latency |
| `Schema.TaggedError<Self>()("Tag", fields)`, `Schema.Error` (`Schema.ts:16196-16257`) | `effect/Schema` | User error classes. In 4.0, `ErrorClass` and `TaggedErrorClass` were renamed |
| `Schema.Exit(success, error, Schema.Defect())`, `Schema.toCodecJson` (`:14268`, `:16617`) | `effect/Schema` | Encode a typed Exit as JSON: `{ _tag: "Success", value } \| { _tag: "Failure", cause: [Fail, Die, Interrupt] }` |
| `Schema.toStandardSchemaV1(schema, { parseOptions })` (`:1385`) | `effect/Schema` | The input for `.validator`. `parseOptions` takes `errors: "all"` and `onExcessProperty` |
| `Cause.hasInterruptsOnly`, `findErrorOption`, `hasDies`, `findDefect` | `effect/Cause` | Tell a failure, a defect and an interruption apart |
| `Rpc.exitSchema`, `RpcSerialization` | `effect/rpc` (unstable) | Reference only: the wire pattern to copy with the stable `Schema.Exit` |
| `HttpEffect.toWebHandlerWith` (`http/HttpEffect.ts:310-347`) | `effect/http` (unstable) | Reference only. Effect's own bridge merges a per-request Context and interrupts on abort with a `ClientAbort` annotation, mapped to 499 |
| `it.effect`, `layer()` | `@effect/vitest` (unstable) | Document these for users' handler tests |

Gotcha (inferred): when an `Exit` is encoded, an `Interrupt` keeps only its `fiberId`, so annotations such as `ClientAbort` are lost.

## 4. Traps the design must handle

1. **Async context across fibers.** Start's request helpers read an `AsyncLocalStorage`. Effect's scheduler may resume a fiber from a task queued by another request, so calling `getRequest()` or `setResponseHeader()` inside a fiber may read the wrong request (inferred, not reproduced). The fix:
   - read the request once, at the start of the handler, and provide it as a service;
   - buffer response headers and the status in a service, and apply them after the fiber ends, back in the handler's own context.
   This needs a test with concurrent requests.
2. **A raw schema passed to `.validator` silently yields `undefined`.** Always wrap it.
3. **Errors lose their class without an adapter.** The library must generate serialization adapters for the app's error classes.
4. **Status 200 on failures.** The library sets the status from the `Exit`:
   - a typed failure gets a status mapped per error, defaulting to 500;
   - a defect gets 500 with a generic body, and the details are logged, never sent;
   - an interruption gets 499.
5. **Hot reload leaks runtimes.** Keep the runtime on a `globalThis` symbol and dispose it on `vite:beforeFullReload`.
6. **Cloudflare** needs `enable_request_signal` before cancellation works there.
7. **Upkeep.** Both Effect 4.0 and Start's server-function API are still moving. The previous package was archived within six months. Keep the surface small, ship compiled JS, and test against both latest releases in CI.

## 5. Naming

- **npm 404s (2026-10-12) for all of these:**
  - `@tarekkharsa/effect-tanstack-start`
  - `effect-tanstack`
  - `tanstack-start-effect`
  - `effect-server-fn`
  - `effect-start-fn`
- **Avoid names close to `effect-start`**, which is an unrelated framework.
- **The unscoped `effect-tanstack-start` is taken** by the archived package.
