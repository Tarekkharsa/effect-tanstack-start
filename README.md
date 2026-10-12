# effect-tanstack-start

Write [TanStack Start](https://tanstack.com/start) server functions as [Effect](https://effect.website) 4 programs. The app keeps one runtime, and every call:
- gets typed errors that reach the client as the same class;
- validates its input with Effect Schema;
- stops its work when the request is aborted.

> **Status: planning.** Nothing is published yet. Read [PLAN.md](PLAN.md) for the API and milestones, and [RESEARCH.md](RESEARCH.md) for the findings behind them.

```ts
export const getProfile = createServerFn({ method: "GET" })
  .validator(EffectStart.schema(ProfileInput))
  .handler(app.handler("member.profile", ({ data }) => Profiles.get(data.id)))
```

## Why

Every app that runs Effect behind TanStack Start writes the same glue by hand:
- a runtime that survives hot reload;
- `runPromise` inside each handler;
- errors that lose their class on the way to the client;
- requests that keep running after the user leaves.

The only earlier package for this is archived, and it doesn't work on Effect 4.0. This project fills that gap with a small surface built on stable Effect APIs.

## License

MIT
