# Cache Components

Decisions for when [`cacheComponents: true`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents) is set in `next.config.ts`. This file is about *which reads to cache, which directive to use, and how to invalidate* — for the mechanics of each directive, follow the doc links.

## When this reference applies

Use this reference when an app already has `cacheComponents: true`, the user wants this architecture while enabling it, or you are reviewing/refactoring an app that targets Cache Components.

If the project has not adopted Cache Components yet and the user asks to enable, migrate, or work through adoption blockers, use the `next-cache-components-adoption` skill first. It owns the route-by-route migration loop, opt-out strategy, and build/dev overlay workflow. Then return here for steady-state query/action/component architecture.

If `cacheComponents` is not enabled and the task is ordinary feature work, follow the core references without adding cache directives. Do not recommend skipping Cache Components based on app category alone; adoption is a migration/project decision, not a per-feature shortcut.

Adopting these in an existing app: follow [Migrating to Cache Components](https://preview.nextjs.org/docs/app/guides/migrating-to-cache-components) and [Adopting Partial Prefetching](https://preview.nextjs.org/docs/app/guides/adopting-partial-prefetching) — they cover the incremental path (per-route `prefetch = 'partial'`, fixing dynamic-usage build errors) rather than a big-bang switch.

## The model

```ts
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true, // prefetch the static shell of linked routes
};
```

- **Static shell** — synchronous content, `'use cache'` output, and Suspense fallbacks prerender at build time.
- **Dynamic holes** — async work without `'use cache'` streams in behind `<Suspense>` at request time.
- **Build constraint** — any async work without `'use cache'` must sit inside `<Suspense>`, or the build fails (wrap it, or add `'use cache'`).

`cacheComponents` implies Partial Prerendering — it replaced `experimental.ppr` / `dynamicIO` / `useCache`, so don't set those. See [caching](https://preview.nextjs.org/docs/app/getting-started/caching).

With `cacheComponents: true`, the skill practice is **cache reusable reads**. Do not leave a database/API read dynamic just because Suspense makes the build pass. If a read has a stable key and a mutation can name what changed, give it a cache directive, tags, and a lifetime.

Dynamic reads are the exception: use them for values that must be recomputed for every request or cannot be invalidated coherently. When you leave a read dynamic, note the reason in the surrounding code/review and invalidate its mutations with `refresh()` because there is no tag to update.

## Decide what to cache

| Data | Directive | Notes |
| ---- | --------- | ----- |
| Cacheable across users (public listings, computed pages) | [`'use cache'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache) | Add [`cacheTag`](https://preview.nextjs.org/docs/app/api-reference/functions/cacheTag) (a global + a scoped tag) and a [`cacheLife`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/cacheLife) profile. |
| Per-user / reads cookies, headers, session | [`'use cache: private'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-private) | May reuse matching calls within one request and keep rendered output in browser memory for its `stale` time; it is not stored in the server cache across production requests or across reloads. |
| Shared result that needs durable storage across server instances | [`'use cache: remote'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-remote) | Use when the hit rate and upstream cost justify a shared cache-handler lookup, including rate-limited services. |
| Genuinely dynamic per request | none | Must be justified. Read inside `<Suspense>`; mutations use `refresh()` because no tag exists. |

When a flow needs the same data twice — once to *decide* and once to *render* — prefer one cached read so both callers share an identity when that entry is available. Add a lighter "exists" query only when it is measurably cheaper or has a different freshness requirement.

Cache the **query** when its result should be reused independently of UI. Cache the **component** when the rendered output is the reusable unit and its props are stable. Usually cache one layer; add a second cache scope only when it has a distinct key, lifetime, or measured rendering benefit.

## Keep a synchronous value out of the shell

You usually don't need this. A query that reads `cookies()`/`headers()` or awaits a DB/`fetch` inside `<Suspense>` already stays out of the shell on its own. Only a *synchronous* request-time read (`new Date()`, `Math.random()`, a sync sqlite read) needs help: `await` [`io()`](https://preview.nextjs.org/docs/app/api-reference/functions/io) before it, with the caller inside `<Suspense>`.

Prefer `io()` over [`connection()`](https://preview.nextjs.org/docs/app/api-reference/functions/connection): both exclude what follows from the shell, but `connection()` blocks prefetches while `io()` stays prefetchable. Reach for `connection()` only when rendering must wait for a real user request.

### A live async layer needs its own boundary

For an async read that must be fresh on every request (stock, presence, a live count), leave it uncached and give it its own `<Suspense>` boundary. The official [Caching guide](https://preview.nextjs.org/docs/app/getting-started/caching#streaming-uncached-data) covers how its fallback joins the shell while the read streams at request time. Use `io()` or `connection()` only for the synchronous request-time cases described in their API references.

## Decide how to invalidate

- [`updateTag(tag)`](https://preview.nextjs.org/docs/app/api-reference/functions/updateTag) — in **server actions**, when the user should see the result immediately (read-your-own-writes). Requires the query to carry a matching `cacheTag`.
- [`revalidateTag(tag, 'max')`](https://preview.nextjs.org/docs/app/api-reference/functions/revalidateTag) — when stale-while-revalidate is appropriate, including route handlers for webhooks or cron. The single-arg `revalidateTag(tag)` form is deprecated.
- [`refresh()`](https://preview.nextjs.org/docs/app/api-reference/functions/refresh) — re-render the current route for the current user. Use it for deliberately dynamic reads with no tag; don't use it instead of `updateTag()` for cached reads.

Tag, cache, invalidate: the `cacheTag` in the query and the `updateTag` in the action use the same string and live in the same feature folder.

### Choose remote caching from observed hit rate

The default shared store for `'use cache'` is per-instance memory and may be ephemeral on serverless. [`'use cache: remote'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-remote) moves entries to a durable cache handler shared across instances, with infrastructure cost and network latency. Use it for shared reads whose upstream cost and expected hit rate justify that trade-off; keep high-cardinality or per-user keys out of the remote cache unless measurement supports them. The [Caching guide](https://preview.nextjs.org/docs/app/getting-started/caching#where-cached-content-is-stored) describes each store.

### Tag by write frequency, not by screen

When one view merges a slow, rarely changing read (a computed offer, a catalog) with a cheap, chatty one (reservations, presence, counters), give each its own cache function and tag. A write to the chatty layer then invalidates only that layer. If both share one tag, every small write recomputes the expensive read and the mutation feels as slow as the page. Measure the mutation from click to settled UI; a time that matches a cached read's cost means the tag is too broad.

## Coordinate hydrated client data

When cached server data seeds SWR, TanStack Query, or another browser cache, follow `references/single-page-applications.md`. Server and client freshness policies are independent; hydration adds library-specific constraints.

## Build failure map

When `next build` fails under Cache Components, map the error back to an architecture rule instead of patching locally:

- Async work without `'use cache'` and without an ancestor `<Suspense>` → cache the reusable read, or wrap a justified dynamic read in a page-owned `<Suspense>`.
- Request data inside `'use cache'` → switch to `'use cache: private'` when it is per-user cacheable, or keep it dynamic with a documented reason.
- `await params` / `await searchParams` at the top of a page → keep the page synchronous and move the read into `params.then()` / `searchParams.then()`.
- Sync request-time values (`new Date()`, `Math.random()`, sync storage reads) captured in the shell → cache stable values, or use [`io()`](https://preview.nextjs.org/docs/app/api-reference/functions/io) for per-request values.

For adoption-wide blocker triage, use `next-cache-components-adoption`. For API-specific recipes, follow the [Migrating to Cache Components guide](https://preview.nextjs.org/docs/app/guides/migrating-to-cache-components).

## Without Cache Components

- Don't use `'use cache'` / `cacheTag` / `cacheLife` — they require the flag.
- Use React `cache()` only for proven same-request dedup needs; plain `server-only` async queries are the default.
- Invalidate with `refresh()` from server actions.
- Pages still use `params.then()` in this architecture. Without Cache Components there is no build-time static shell to preserve, but keeping pages synchronous still lets chrome paint before route-specific data resolves and keeps the app consistent.
