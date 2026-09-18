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
| Per-user / reads cookies, headers, session | [`'use cache: private'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-private) | Cached in the browser only, doesn't persist across reloads; never stored on the server. Still give it a `cacheTag` — sign-in and sign-out must `updateTag` it, or the session read stays stale until reload. |
| Remote service, safe across users, worth durable storage | [`'use cache: remote'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-remote) | Protects against rate-limited third-party APIs. |
| Genuinely dynamic per request | none | Must be justified. Read inside `<Suspense>`; mutations use `refresh()` because no tag exists. |

When a flow needs the same data twice — once to *decide* (should this section show, is this option still available) and once to *render* it — make it one cached read and let both callers hit it. Don't add a lighter "exists" query next to the full one; the second request is free from the cache, and the decision and the render can never disagree.

Cache the **query** when its result should be reused across requests. Cache the **component** when rendering is expensive and props are stable (a nav, a trending sidebar). Don't `'use cache'` a component that already calls a `'use cache'` query — double-caching, no benefit.

## Keep a synchronous value out of the shell

You usually don't need this. A query that reads `cookies()`/`headers()` or awaits a DB/`fetch` inside `<Suspense>` already stays out of the shell on its own. Only a *synchronous* request-time read (`new Date()`, `Math.random()`, a sync sqlite read) needs help: `await` [`io()`](https://preview.nextjs.org/docs/app/api-reference/functions/io) before it, with the caller inside `<Suspense>`.

Prefer `io()` over [`connection()`](https://preview.nextjs.org/docs/app/api-reference/functions/connection): both exclude what follows from the shell, but `connection()` blocks prefetches while `io()` stays prefetchable. Reach for `connection()` only when rendering must wait for a real user request.

### A live layer needs no marker, only its own boundary

A read that must be fresh on every request (stock, presence, a live count) is simply left uncached and given its own `<Suspense>` inside the cached component. Under Cache Components an uncached async read is dynamic by itself: it is left out of the static shell and of every prefetch, and streams in on the request. Don't add `connection()` or `io()` to say so. The one trap is a synchronous unstable value such as `new Date()`: the prerender flags it if it runs before the first awaited I/O, so evaluate it after the first `await` (run the dated query second), or move it inside a cache scope where it is allowed.

## Decide how to invalidate

- [`updateTag(tag)`](https://preview.nextjs.org/docs/app/api-reference/functions/updateTag) — in **server actions**, when the user should see the result immediately (read-your-own-writes). Requires the query to carry a matching `cacheTag`.
- [`revalidateTag(tag, 'max')`](https://preview.nextjs.org/docs/app/api-reference/functions/revalidateTag) — in **route handlers** (webhooks, cron) for stale-while-revalidate. The single-arg `revalidateTag(tag)` form is deprecated.
- [`refresh()`](https://preview.nextjs.org/docs/app/api-reference/functions/refresh) — re-render the current route for the current user. Use it for deliberately dynamic reads with no tag; don't use it instead of `updateTag()` for cached reads.

Tag, cache, invalidate: the `cacheTag` in the query and the `updateTag` in the action use the same string and live in the same feature folder.

### Plain `use cache` is per instance in serverless

`'use cache'` stores entries in the memory of the process that computed them. On a serverless host every request can land on a different instance, so shared reads recompute on nearly every request even though they are "cached", and the per-link prefetch pays the same cost. Symptom: the same page takes the same time on every reload in production while it is instant on the second reload locally. Give shared reads (catalogs, computed offers, search results) `'use cache: remote'` so the platform's shared cache handler holds them; Vercel wires the Runtime Cache to it automatically, and without a configured handler `remote` falls back to the in-memory LRU, so local behavior is unchanged. Per-user reads keyed by the user's id stay on plain `'use cache'`, and only the request-bound read (the session) is `'use cache: private'`. Verify in production by requesting the same URL a few times with a fixed session cookie and comparing streamed time, not TTFB.

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
