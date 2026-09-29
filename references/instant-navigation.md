# Instant navigation

Decisions for apps with `cacheComponents` and `partialPrefetching`: which links resolve their data before the click, which reads move to a later render stage, and how to check what a navigation actually shows. Mechanics live in the linked docs; `references/pages-suspense.md` owns where the boundaries go, and `references/cache-components.md` owns what gets cached.

## Optimizing prefetching for high-value routes

With `cacheComponents` + [`partialPrefetching`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/partialPrefetching) enabled, a visible `<Link>` prefetches the destination's shared [App Shell](https://preview.nextjs.org/docs/app/glossary#app-shell) — enough to commit navigation instantly, with link-specific content streaming after. The default (`'auto'`) already does this; don't write `prefetch = 'auto'`.

Use `<Link prefetch={true}>` on high-value links to also resolve the destination's per-link data (`params`, `searchParams`, the full URL) at prefetch time. Each such link can wake the server for a prerender, so reserve it for routes users predictably visit next. See [Optimizing prefetching](https://preview.nextjs.org/docs/app/guides/optimizing-prefetching).

That cost is a choice, not a given. When a list's rows should all prefetch, make the per-link payload small first: gate the heavy part of the destination (message bodies, full records) behind `unstable_navigation()` and keep the prefetched part to what the first paint needs (a header, a summary). Then `prefetch={true}` on every row in view is affordable. Use hover prefetch for long-tail links that are costly or rarely followed.

Can't enable `partialPrefetching` app-wide yet? Opt in per route with `export const prefetch = 'partial'` on the destination, then drop the per-route exports once the global flag is on — see [Adopting Partial Prefetching](https://preview.nextjs.org/docs/app/guides/adopting-partial-prefetching) for the incremental path and [prefetch config](https://preview.nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/prefetch) for the options. To check that navigation actually feels instant, see [Validating instant navigation](#validating-instant-navigation).

### Three render stages, two gates

The [Optimizing prefetching guide](https://preview.nextjs.org/docs/app/guides/optimizing-prefetching) distinguishes the shared App Shell, an optional per-link prefetch, and the navigation. Two canary APIs move cacheable work to a later stage without making it request-dependent:

| Gate | Kept out of | Rendered by |
| --- | --- | --- |
| [`await unstable_prefetch()`](https://preview.nextjs.org/docs/app/api-reference/functions/prefetch) | the App Shell | a per-link prefetch, or the navigation |
| [`await unstable_navigation()`](https://preview.nextjs.org/docs/app/api-reference/functions/navigation) | the App Shell and per-link prefetches | the navigation only |

Use `unstable_prefetch()` for cacheable content that should stay out of the shared App Shell but may be resolved by `prefetch={true}`; use `unstable_navigation()` for cacheable content that should be produced only after navigation. Both are canary APIs, so read their `preview.nextjs.org` references before using them.

Neither may be awaited inside a cache scope. Await the gate in the async server component, right before it calls its query, and leave the query a plain function:

```tsx
export async function Comments({ postId }: { postId: string }) {
  await unstable_navigation();
  const comments = await getComments(postId); // plain query, or 'use cache' called below the gate
  ...
}
```

Don't add a second query whose only job is to hold the gate (`getComments` calling `getCommentsCached`). The gate is a render-stage decision, so it belongs where the render happens; the query stays reusable from other stages and other callers. A `'use cache'` query called below the gate keeps its lifetime and still serves the next visitor, and a query that needs session or cookie values still uses the extract-and-pass wrapper from `references/cache-components.md` for those, just without the gate in it.

### Keep a live layer out of the prefetch without blocking it

A per-link prerender advances through static or cached work and stops at uncached reads, showing the nearest `<Suspense>` fallback. Keep a live read (presence, live availability) in its own sibling boundary so it does not reduce the useful cached content available before the click. Use `unstable_navigation()` only when the work is cacheable but intentionally excluded from prefetches; ordinary uncached async work already stops the prerender. See [Exclude content from a prefetch](https://preview.nextjs.org/docs/app/guides/optimizing-prefetching#exclude-content-from-a-prefetch).

If the live value is handed to a Client Component as a promise, resolve it with `use()` inside a small `<Suspense>` whose fallback preserves the same layout. Persistent navigation chrome belongs in a layout that survives the navigation.

## Validating instant navigation

With `cacheComponents` on, Next.js [validates every Page and Default segment in development by default](https://preview.nextjs.org/docs/app/guides/instant-navigation#validate-instant-navigation). You don't add `export const instant` or `experimental.instantInsights` to switch that on — read what it reports and fix it with the rules above: cache the read, or narrow the boundary.

Reach for [`export const instant = false`](https://preview.nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant#disabling-instant) only as an escape hatch — to exempt a blocking ancestor layout while still asserting the pages beneath it, or to opt a route out of static-shell validation. It can't be used in a Client Component.

To see what actually lands in the initial UI, use the [Navigation Inspector](https://preview.nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant#inspecting-loading-states). To keep it from regressing, lock it in with an end-to-end test (→ `references/testing.md`).

## Hidden routes stay mounted

Next.js keeps recently visited segments mounted inside a hidden [`<Activity>`](https://preview.nextjs.org/docs/app/guides/preserving-ui-state) so their state and scroll survive back and forward navigation. An uncontrolled input in a shared shell can therefore keep a DOM value that no longer matches the URL; sync it from the URL in a layout effect on mount. For what this means when locating elements in tests, see `references/testing.md`.
