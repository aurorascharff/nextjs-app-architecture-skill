# Cache Components

The model and constraints when [`cacheComponents: true`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents) is set in `next.config.ts`.

## When to enable it

Turn on Cache Components when:

- You want a true static shell prerendered at build time, with dynamic data streaming in per request.
- You're shipping to a CDN-served target (Vercel, Cloudflare) and want HTML served before the data layer runs.
- The app has clear cacheable boundaries (per-user feeds, public listings, computed pages).

Skip it when:

- The app is dynamic by nature (admin tool, dashboard with per-user-per-row data).
- You're still iterating on the data model and don't want to think about tags yet.

Without `cacheComponents`, you still get RSC streaming and Suspense — you just don't get the prebuilt static shell. The architecture in `references/feature-folders.md` and `references/components.md` works either way.

## The model

```ts
// next.config.ts
import type { NextConfig } from 'next';

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
};

export default nextConfig;
```

Pair `cacheComponents` with [`partialPrefetching: true`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/partialPrefetching) so links prefetch the static shell — it's also what makes the `io()`-vs-`connection()` prefetch tradeoff below matter. See `references/pages-suspense.md`.

With this flag set:

- **Static shell** — synchronous content, `'use cache'` components, and Suspense fallbacks prerender **at build time**.
- **Dynamic holes** — async components without `'use cache'` stream in behind `<Suspense>` at request time.
- **Build constraint** — any async work without `'use cache'` must sit inside `<Suspense>`, or the build fails. Fix it by wrapping the component in `<Suspense>` or adding `'use cache'`.

You don't have to cache anything. A perfectly valid strategy is to enable `cacheComponents` purely for the static shell + streaming, keep every query dynamic (read per request inside `<Suspense>`), and invalidate with `refresh()` — no `'use cache'`, `cacheTag`, or `updateTag` at all. Reach for the caching primitives below only when a specific query is worth caching; `updateTag` becomes relevant only once a query carries a `cacheTag`.

Two things follow from the flag in Next 16. `cacheComponents` **implies Partial Prerendering** — it's the single switch that replaced the old `experimental.ppr`, `experimental.dynamicIO`, and `experimental.useCache` flags, so don't set those. And navigation preserves state via React [`<Activity>`](https://react.dev/reference/react/Activity): a route you leave is hidden rather than unmounted, so its state survives when you return. See [caching](https://preview.nextjs.org/docs/app/getting-started/caching) and [preserving UI state](https://preview.nextjs.org/docs/app/guides/preserving-ui-state).

## Pages stay synchronous

This is the rule that makes Cache Components actually work for you. If you `await params` at the top of a page, the entire page is dynamic — nothing prerenders into the shell.

Use `params.then()` instead, so the page function returns synchronously and chrome above the `.then()` ends up in the static shell. See `references/pages-suspense.md` for the pattern.

## `'use cache'` on queries

The most common spot:

```ts
import 'server-only';
import { cacheLife, cacheTag } from 'next/cache';
import { cache } from 'react';

export const getFeed = cache(async (userId: string) => {
  'use cache';
  cacheTag('feed', `feed-${userId}`);
  cacheLife('seconds');
  return db.post.findMany({ where: { userId } });
});
```

`'use cache'` makes the result cacheable across requests. [`cacheTag`](https://preview.nextjs.org/docs/app/api-reference/functions/cacheTag) adds invalidation tags — use both a global tag and a scoped one for flexible invalidation. [`cacheLife`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/cacheLife) sets the staleness profile (`'seconds'`, `'minutes'`, `'hours'`, or custom).

### `'use cache: private'`

If the query reads cookies, headers, or session data, use [`'use cache: private'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-private) instead. Results are cached only in the user's browser and don't persist across page reloads — never stored on the server.

```ts
export const getNotifications = cache(async () => {
  'use cache: private';
  cacheTag('notifications');
  cacheLife('seconds');
  const userId = await getCurrentUserId();
  return db.notification.findMany({ where: { userId } });
});
```

### `'use cache: remote'`

For data from a remote service (third-party API, public endpoint) that's safe to cache across users and worth storing durably beyond the local edge, use [`'use cache: remote'`](https://preview.nextjs.org/docs/app/api-reference/directives/use-cache-remote). Useful for protecting against rate-limited APIs (GitHub, payment providers, geocoders).

```ts
export const getRepo = cache(async (owner: string, name: string) => {
  'use cache: remote';
  cacheTag(`repo-${owner}-${name}`);
  cacheLife('hours');
  return fetch(`https://api.github.com/repos/${owner}/${name}`).then(r => r.json());
});
```

### Keeping a synchronous value out of the static shell: `io()`

**You usually don't need this.** A query that reads `cookies()`/`headers()` or `await`s a DB/`fetch` inside a `<Suspense>` boundary already suspends per request — it stays out of the shell on its own. `io()` is only for reading a *synchronous* request-time source (`new Date()`, `Math.random()`, a sync `sqlite` read) that would otherwise be captured into the shell at build time.

`await` [`io()`](https://preview.nextjs.org/docs/app/api-reference/functions/io) from `next/cache` before that read; it suspends during prerender, so the caller must sit inside `<Suspense>`:

```ts
import { io } from 'next/cache';

async function CurrentTime() {
  await io();
  const now = new Date(); // without io(), this would bake into the static shell
  return <time>{now.toLocaleTimeString()}</time>;
}
```

Prefer `io()` over [`connection()`](https://preview.nextjs.org/docs/app/api-reference/functions/connection) (from `next/server`): both exclude what follows from the shell, but `connection()` blocks prefetches until a real user request, while `io()` stays prefetchable. Reach for `connection()` only when rendering should genuinely wait for a live request.

## `'use cache'` on components

If the entire component output can be cached (e.g. it doesn't change per user, or you want to cache the rendered HTML), put `'use cache'` on the component:

```tsx
async function TrendingTags() {
  'use cache';
  cacheTag('trending');
  cacheLife('minutes');

  const tags = await db.tag.findMany({ orderBy: { count: 'desc' }, take: 6 });
  return (
    <ul>
      {tags.map(t => (
        <li key={t.name}>#{t.name}</li>
      ))}
    </ul>
  );
}
```

This caches the rendered HTML, not just the query result. Faster on cache hit (no rendering work), but the cache key includes the props — pass minimal props.

### When to cache the query vs the component

- **Cache the query** when multiple components consume the same data (you want to dedupe + reuse).
- **Cache the component** when rendering is expensive and the props are stable (a category nav, a sidebar of trending items).

Don't `'use cache'` on a component that internally calls a `'use cache'`d query — that's double-caching with no benefit and harder invalidation semantics.

## Invalidation: `updateTag` vs `revalidateTag`

- **[`updateTag(tag)`](https://preview.nextjs.org/docs/app/api-reference/functions/updateTag)** — invalidates immediately and re-renders for read-your-own-writes. Use inside **server actions** when the user expects to see the result.
- **[`revalidateTag(tag, 'max')`](https://preview.nextjs.org/docs/app/api-reference/functions/revalidateTag)** — stale-while-revalidate. Use in **route handlers** (webhooks, cron) where you don't need an immediate UI update. The `'max'` profile is required for SWR; the single-arg `revalidateTag(tag)` form is deprecated (it expired immediately — use `updateTag` if that's what you want).

```ts
'use server';
import { updateTag } from 'next/cache';

export async function createPost(formData: FormData) {
  // ... insert ...
  updateTag('feed');
}
```

The `cacheTag` in the query and the `updateTag` in the action live in the same feature folder. Same string. **Tag, cache, invalidate.**

## Build behavior

`next build` prerenders every page: synchronous content, `'use cache'` results, and Suspense fallbacks bake into the static output, while dynamic holes render and stream per request. Three common build failures each map to a skill rule:

- Async work without `'use cache'` and without an ancestor `<Suspense>` → wrap it in `<Suspense>` or add `'use cache'`.
- Reading cookies/headers inside `'use cache'` → switch to `'use cache: private'`.
- `params` accessed at the top level of the page → convert `await params` to `params.then()`.

See [caching](https://preview.nextjs.org/docs/app/getting-started/caching) for the full prerendering model.

## Without Cache Components

If `cacheComponents` is not enabled:

- Don't use `'use cache'`, [`cacheTag`](https://preview.nextjs.org/docs/app/api-reference/functions/cacheTag), or [`cacheLife`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/cacheLife) — they require the flag.
- Keep `cache()` from React for per-request dedup.
- Invalidate via [`refresh()`](https://preview.nextjs.org/docs/app/api-reference/functions/refresh) from server actions instead of `updateTag()`. `refresh()` re-renders the route for the current user.
- Pages can use `await params` or `params.then()` — either works. `params.then()` still helps unrelated chrome paint faster, but there's no build-time prerender to preserve.

The rest of the architecture (feature folders, async server components, Suspense at the page) works identically.
