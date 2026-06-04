# Pages and Suspense

How to compose pages, place Suspense boundaries, and prevent layout shift.

## Pages are composition only

Pages in `app/` import feature components and place `<Suspense>` boundaries. They never:

- Fetch data directly (queries live in feature folders)
- Define new components except wrappers (e.g. `<NavForward>`)
- Inline route-specific UI (extract it into the feature folder)

## Page function signatures

Type page and layout functions with the generated `PageProps<'/route'>` and `LayoutProps<'/route'>` helpers — they're emitted by [`typedRoutes`](https://nextjs.org/docs/app/api-reference/config/next-config-js/typedRoutes) and give you the correct `params` / `searchParams` / `children` shape for that exact route.

```tsx
export default function PostPage({ params }: PageProps<'/post/[id]'>) { /* ... */ }
export default function PostLayout({ children, params }: LayoutProps<'/post/[id]'>) { /* ... */ }
```

Don't hand-write `{ params: Promise<{ id: string }> }` — the generated types stay in sync with the route, including catch-all and optional segments. Route handlers have their own `RouteContext<'/api/...'>`. Turn on `typedRoutes: true` in `next.config.ts`.

## Keep pages synchronous

Use `params.then()` instead of `await params`. Content above the `.then()` pre-renders into the static shell; content inside it suspends.

```tsx
import { Suspense } from 'react';
import { PostDetail, PostDetailSkeleton } from '@/features/post/components/post-detail';

export default function PostPage({ params }: PageProps<'/post/[id]'>) {
  return (
    <div>
      <PageHeader back title="Post" />
      <Suspense fallback={<PostDetailSkeleton />}>
        {params.then(({ id }) => (
          <PostDetail id={id} />
        ))}
      </Suspense>
    </div>
  );
}
```

Page chrome (`<PageHeader>`) sits **above** the `params.then()` so it paints instantly. The `Suspense` fallback covers only the dynamic section.

### `searchParams` and combined params

```tsx
// searchParams only
export default function SearchPage({ searchParams }: PageProps<'/search'>) {
  return searchParams.then(sp => {
    const q = typeof sp.q === 'string' ? sp.q : '';
    return q ? <SearchResults query={q} /> : <EmptyState />;
  });
}

// Both params and searchParams
export default function ProfilePage({ params, searchParams }: PageProps<'/u/[handle]'>) {
  return Promise.all([params, searchParams]).then(([{ handle }, sp]) => (
    <ProfileFeed handle={handle} tab={parseTab(sp.tab)} />
  ));
}
```

### `generateMetadata` can await directly

[`generateMetadata`](https://nextjs.org/docs/app/api-reference/functions/generate-metadata) runs before the page renders, so `await params` is fine there:

```tsx
export async function generateMetadata({ params }: PageProps<'/post/[id]'>): Promise<Metadata> {
  const { id } = await params;
  const post = await getPost(id);
  return { title: post.title, description: post.body.slice(0, 160) };
}
```

### `generateStaticParams` for dynamic routes

For `[slug]`-style routes, export [`generateStaticParams`](https://nextjs.org/docs/app/api-reference/functions/generate-static-params) from the page or layout to pre-build a known set of pages. Combined with `cacheComponents` and `'use cache'` on the queries, every listed slug ends up in the static shell:

```tsx
import { getEvents } from '@/features/event/event-queries';

export async function generateStaticParams() {
  const events = await getEvents();
  return events.map(event => ({ slug: event.slug }));
}
```

The gotcha: `generateStaticParams` does **not** change the page function signature. `params` is still a Promise; the page still uses `params.then(...)`. The pre-build only affects which slugs prerender at build time.

### `notFound()` from queries

When a query can't find the resource, call [`notFound()`](https://nextjs.org/docs/app/api-reference/functions/not-found):

```ts
import { notFound } from 'next/navigation';

export const getEventBySlug = cache(async (slug: string) => {
  const event = await db.event.findUnique({ where: { slug } });
  if (!event) notFound();
  return event;
});
```

It bubbles to the nearest [`not-found.tsx`](https://nextjs.org/docs/app/api-reference/file-conventions/not-found). Don't wrap it in a try/catch — use [`unstable_rethrow`](https://nextjs.org/docs/app/api-reference/functions/unstable_rethrow) from `next/navigation` if you need to.

## The page owns the Suspense boundary

The feature exports the async component **and** its skeleton. The page imports both and places the boundary. Don't pre-wrap inside the feature — that hides the boundary and prevents grouping siblings.

```tsx
// features/post/components/post-detail.tsx
export async function PostDetail({ id }: { id: string }) { ... }
export function PostDetailSkeleton() { ... }
```

```tsx
// app/post/[id]/page.tsx
<Suspense fallback={<PostDetailSkeleton />}>
  {params.then(({ id }) => (
    <>
      <PostDetail id={id} />
      <ErrorBoundary title="Replies didn't load">
        <Suspense fallback={<RepliesSkeleton />}>
          <Replies postId={id} />
        </Suspense>
      </ErrorBoundary>
    </>
  ))}
</Suspense>
```

If a page uses a transition wrapper (e.g. `<NavForward>`), place it in the page next to the `<Suspense>` boundary. Feature components render content and skeletons, not transition wrappers.

## Don't create page-local wrapper components

Avoid components whose only job is to group boundary content, like `HomeLists` or `HomeListsSkeleton`. Keep the resolved JSX and fallback JSX **inline in the page** so the loading shape, headings, and grouped reveal behavior are visible at the boundary.

```tsx
// Wrong — hides the structure behind a wrapper
<Suspense fallback={<HomeListsSkeleton />}>
  <HomeLists searchParams={searchParams} />
</Suspense>
```

```tsx
// Right — structure visible at the page level
<Suspense
  fallback={
    <>
      <FeaturedSkeleton />
      <RecentSkeleton />
    </>
  }
>
  {searchParams.then(sp => (
    <>
      <Featured filter={sp.filter} />
      <Recent filter={sp.filter} />
    </>
  ))}
</Suspense>
```

## Suspense boundary placement rules

1. **First section gets its own Suspense** with a known-height skeleton fallback.
2. **Section headings stay outside Suspense** when their final position is stable.
3. **Variable-height sections: group everything below them** in the same Suspense, including any headings that would otherwise paint in the wrong vertical position.
4. **Fixed-height sections: own boundary is safe.**
5. **Variable-length lists: show 2–5 skeleton items**, not the real count.
6. **Inner Suspense content stays out of the outer skeleton.** Each boundary owns its own.
7. **Never `fallback={null}` for visible UI.** If a boundary covers UI, give it a real shaped fallback, or group it with a sibling boundary that already has the correct fallback.
8. **If the top section's final height is unknown, group the following sections** in the same boundary so they reveal together and don't jump underneath.

## Error boundaries

Wrap fallible sections in a Next.js-aware error boundary so one failure doesn't take down the page. Use `unstable_catchError` from `next/error` (Next.js 16.2+) — it knows about Next's control-flow throws (`notFound()`, `redirect()`, `unauthorized()`, `forbidden()`) and re-fetches server data on retry:

```tsx
// components/ui/error-boundary.tsx
'use client';

import { unstable_catchError as catchError, type ErrorInfo } from 'next/error';

function ErrorFallback(props: { title?: string }, { unstable_retry: retry }: ErrorInfo) {
  return (
    <div>
      <p>{props.title ?? 'Something went wrong'}</p>
      <button onClick={() => retry()}>Try again</button>
    </div>
  );
}

export default catchError(ErrorFallback);
```

Use it around suspending sections:

```tsx
import ErrorBoundary from '@/components/ui/error-boundary';

<ErrorBoundary title="Replies didn't load">
  <Suspense fallback={<RepliesSkeleton />}>
    <Replies postId={id} />
  </Suspense>
</ErrorBoundary>
```

Why not plain `react-error-boundary`? It catches Next's framework throws (so `notFound()` never reaches `not-found.tsx`), and `resetErrorBoundary` doesn't re-fetch server data. For background, see [Error Handling in Next.js with catchError](https://aurorascharff.no/posts/error-handling-in-nextjs-with-catch-error/).

Pair component-level boundaries with [`error.tsx`](https://nextjs.org/docs/app/api-reference/file-conventions/error) at the route segment for unrecoverable errors. `error.tsx` also receives an `unstable_retry` callback you can wire to a button.

## Layout-level Suspense

Layouts compose feature components the same way pages do. Use `<Suspense>` for slots that fetch data (auth badge, sidebar):

```tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <Suspense>
          <AuthGate userPromise={getCurrentUser()} />
        </Suspense>
        <main>{children}</main>
      </body>
    </html>
  );
}
```

`AuthGate` is a client component that resolves the promise with `use()` so the dialog can render conditionally without server-side branching.

## CLS prevention

Layout shift happens when:

- A skeleton is shorter than the real content
- A heading sits inside a Suspense boundary whose final height is unknown — it paints in the wrong place, then jumps
- A variable-length list streams in without a fallback that reserves space

Fixes:

- Match skeleton height to the typical real content height.
- Move headings **outside** boundaries when their position depends on data above them.
- For unknown-height top sections, group everything below in one boundary so siblings stream together.

To audit CLS, use React DevTools' Suspense panel to pin each boundary in its loading state and check vertical positions.

## Optional: opt-in prefetch for high-value routes

For routes where instant-feeling navigation matters most (feed, detail pages), opt in to runtime prefetching at the route level:

```tsx
export const unstable_prefetch = 'force-runtime';
```

This is **optional and unstable**. The `unstable_` prefix means semantics may change between releases — test on the canary you're targeting. It's not part of the core architecture; reach for it when you've measured the navigation feels slow on a high-value route and want to trade server CPU for perceived speed.

Don't put this on every route. Each opt-in page runs a full render in the background for every `<Link>` that enters the viewport — that costs server CPU and database load. Reserve it for high-value navigation targets.

## Never wrap the entire page in a Suspense fallback

Page chrome (header, nav, surrounding layout) should paint instantly. Only data-dependent sections suspend. If you find yourself wrapping `<div>` and everything in it with `<Suspense fallback={<FullPageSkeleton />}>`, restructure: pull static elements out, narrow the boundary to just the dynamic part.
