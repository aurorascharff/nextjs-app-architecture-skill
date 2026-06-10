# Cache Components

The model and constraints when [`cacheComponents: true`](https://nextjs.org/docs/canary/app/api-reference/config/next-config-js/cacheComponents) is set in `next.config.ts`.

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
};

export default nextConfig;
```

With this flag set:

- **Static shell** — synchronous content, `'use cache'` components, and Suspense fallbacks prerender **at build time**.
- **Dynamic holes** — async components without `'use cache'` stream in behind `<Suspense>` at request time.
- **Build constraint** — any async work without `'use cache'` must sit inside `<Suspense>`, or the build fails with "this component performs dynamic work outside a Suspense boundary."

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

- `'use cache'` makes the query result cacheable across requests.
- `cacheTag` adds invalidation tags. Use both a global tag and a scoped one for flexible invalidation.
- `cacheLife` sets the staleness profile (`'seconds'`, `'minutes'`, `'hours'`, custom).

### `'use cache: private'`

If the query reads cookies, headers, or session data, use `'use cache: private'` instead. The cache is scoped per user.

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

For data fetched from a remote service (third-party API, public endpoint) that's safe to cache across users and durable beyond the local edge, use `'use cache: remote'`. Useful for protecting yourself against rate-limited APIs (GitHub, payment providers, geocoders).

```ts
export const getRepo = cache(async (owner: string, name: string) => {
  'use cache: remote';
  cacheTag(`repo-${owner}-${name}`);
  cacheLife('hours');
  return fetch(`https://api.github.com/repos/${owner}/${name}`).then(r => r.json());
});
```

### Opting back into dynamic with `connection()`

For a single query that must always run per request, call [`connection()`](https://nextjs.org/docs/app/api-reference/functions/connection) from `next/server` first. The surrounding render becomes dynamic, so the calling component must sit inside a `<Suspense>` boundary:

```ts
import 'server-only';

import { connection } from 'next/server';
import { cache } from 'react';

export const getUserFavorites = cache(async (userName: string) => {
  await connection();
  return db.favorite.findMany({ where: { userName } });
});
```

Use it when the data is genuinely per-request and per-user (favorites, draft state, real-time counts) and `'use cache: private'` isn't a fit.

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

- **`updateTag(tag)`** — invalidates immediately and re-renders for read-your-own-writes. Use inside **server actions** when the user expects to see the result.
- **`revalidateTag(tag)`** — stale-while-revalidate. Use in **route handlers** (webhooks, cron) where you don't need an immediate UI update.

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

When you run `next build` with `cacheComponents: true`:

1. Next runs every page through prerendering.
2. Synchronous content + `'use cache'` results + Suspense fallbacks are baked into the static output.
3. Anything async without `'use cache'` and without an ancestor `<Suspense>` causes a build error.
4. Per request, dynamic holes are rendered and streamed in.

Watch for these errors:

- **"Async work outside Suspense boundary"** — wrap the offending component in `<Suspense>` or add `'use cache'`.
- **"Cannot read cookies/headers inside `'use cache'`"** — switch to `'use cache: private'`.
- **"params accessed at the top level of the page"** — convert `await params` to `params.then()`.

## Without Cache Components

If `cacheComponents` is not enabled:

- Don't use `'use cache'`, [`cacheTag`](https://nextjs.org/docs/canary/app/api-reference/functions/cacheTag), or [`cacheLife`](https://nextjs.org/docs/canary/app/api-reference/config/next-config-js/cacheLife) — they require the flag.
- Keep `cache()` from React for per-request dedup.
- Invalidate via [`refresh()`](https://nextjs.org/docs/canary/app/api-reference/functions/refresh) from server actions instead of `updateTag()`. `refresh()` re-renders the route for the current user.
- Pages can use `await params` or `params.then()` — either works. `params.then()` still helps unrelated chrome paint faster, but there's no build-time prerender to preserve.

The rest of the architecture (feature folders, async server components, Suspense at the page) works identically.
