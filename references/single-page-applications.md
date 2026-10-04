# Single-page application patterns

Use this reference when a feature adds SWR, TanStack Query, or another browser data cache. Start with the Next.js [client-side data fetching guide](https://nextjs.org/docs/app/guides/client-side-data-fetching), then use its library-specific [TanStack Query](https://nextjs.org/docs/app/guides/client-side-data-fetching/tanstack-query) or [SWR](https://nextjs.org/docs/app/guides/client-side-data-fetching/swr) guide for runnable examples and framework integration details.

## Decide whether a client cache is needed

Use a client data library when the browser needs revalidation, optimistic mutations, request deduplication, or shared live data. If a Client Component only reads server data once, pass a Promise from its Server Component and unwrap it with `use()` instead.

## Keep ownership with the feature

```text
features/<domain>/
  <domain>-cache.ts          # Pure server tags + client keys
  <domain>-queries.ts        # Server reads and cacheLife
  <domain>-query-options.ts  # Client fetcher/query options
  hooks/use-*.ts             # Client mutations and coordination
  components/                # Async server owner + client leaves
```

The cache contract imports neither Next.js nor the client library. Queries, actions, route handlers, hydration code, query options, and hooks import identities from it. This prevents a key or tag spelling from drifting between a read and its invalidation.

Keep behavior in the layer that owns it:

- Server `cacheLife`, `cacheTag`, and database reads belong in `<domain>-queries.ts`.
- Browser freshness and refetch behavior belong in `<domain>-query-options.ts` or the SWR hook.
- Optimistic mutation behavior belongs in `hooks/use-*.ts`.
- Tiny effect-only or interactive leaves belong in `components/`.

## Seed from the server

The async feature component owns the initial read and the library's hydration provider. The page remains a synchronous composition surface and owns the feature's Suspense boundary.

- With SWR, seed the exact key read by `useSWR`; follow the Next.js [SWR guide](https://nextjs.org/docs/app/guides/client-side-data-fetching/swr) and the library's [Next.js guidance](https://swr.vercel.app/docs/with-nextjs).
- With TanStack Query, seed the same query key read by the client query; follow the Next.js [TanStack Query guide](https://nextjs.org/docs/app/guides/client-side-data-fetching/tanstack-query) and the library's [Advanced Server Rendering guide](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr).

Do not move the initial read to the browser just because the feature also has a client cache.

## Coordinate Cache Components

The server cache and browser cache have independent freshness policies. Do not mirror `cacheLife` into `staleTime`, polling intervals, or SWR revalidation settings. Coordinate identities and invalidation, not durations.

For tag-driven data, a mutation updates the client cache for immediate feedback and invalidates the same server tag used by the seeded read. For a time-driven server read, choose its `cacheLife` from the server data's freshness requirement.

Cache the Server Component that owns `<HydrationBoundary>` or `<SWRConfig>` when the client router should reuse its rendered RSC payload. This component cache is a distinct layer even when the underlying query is already cached: the query cache reuses server data, while the component cache keeps the seeded data and hydration metadata on one lifecycle.

Choose the component directive from the data's sharing boundary:

- Use `'use cache'` with a reusable profile and matching tags when the rendered output is safe to share across requests.
- Use `'use cache'` with `cacheLife({ expire: 0 })` when the output may stay in the current browser's client cache but must not be reused by a later server request.
- Use `'use cache: private'` with an explicit client `stale` time when the component reads cookies, headers, session state, or other request-specific data. It is not stored in the server cache across production requests.

Treat library hydration metadata as part of the seeded snapshot. Create the `QueryClient`, dehydrated state, or SWR fallback inside the cached component; do not cache those objects separately as the server data source. Apply the component's tags to every seeded read whose invalidation must refresh the rendered payload.

For TanStack Query, prefer ordinary `dehydrate()` inside this cached boundary. Build hydration state manually only when the boundary cannot be cached and every seeded query is already resolved; a hand-built resolved-query state cannot represent pending-query dehydration.

## Mutate without drift

Let the client library own the optimistic browser update, rollback, and authoritative response. Let the write invalidate the server tag only after stored data changes. Do not add polling as a cache-coordination mechanism; add focus revalidation, intervals, SSE, or WebSockets only when the product actually needs external updates to appear automatically.
