# Pages and Suspense

How to compose pages, place Suspense boundaries, and prevent layout shift.

## Pages are composition only

Pages in `app/` import feature components and place `<Suspense>` boundaries. They never:

- Fetch data directly (queries live in feature folders)
- Define reusable UI components except thin transition wrappers (e.g. `<ViewTransition>`) or tiny route-local control-flow helpers
- Inline route-specific UI (extract it into the feature folder)
- Pass raw `params` / `searchParams` to features

## Page function signatures

Type page and layout functions with the auto-generated `PageProps<'/route'>` / `LayoutProps<'/route'>` helpers — no import, regenerated on `next dev` / `next build` / `next typegen`. See [route type helpers](https://preview.nextjs.org/docs/app/api-reference/config/typescript#route-type-helpers).

```tsx
export default function PostPage({ params }: PageProps<'/post/[id]'>) { /* ... */ }
```

Don't hand-write `{ params: Promise<{ id: string }> }` — the generated types stay in sync with the route (catch-all, optional segments). Route handlers use `RouteContext<'/api/...'>`. `typedRoutes: true` is a *separate* feature (statically-typed `href`s), not the source of these helpers.

## Keep pages synchronous

Use `params.then()` instead of `await params`. Content above the `.then()` pre-renders into the static shell; content inside it suspends.

```tsx
import { Suspense } from 'react';
import { PostDetail, PostDetailSkeleton } from '@/features/post/components/post-detail';

export default function PostPage({ params }: PageProps<'/post/[id]'>) {
  return (
    <div>
      <h1>Post</h1>
      <Suspense fallback={<PostDetailSkeleton />}>
        {params.then(({ id }) => (
          <PostDetail id={id} />
        ))}
      </Suspense>
    </div>
  );
}
```

The `<h1>` sits **above** the `params.then()` so it paints instantly. The `Suspense` fallback covers only the dynamic section.

Resolve route props to plain values at this boundary. Feature components receive `id`, `slug`, `query`, or parsed filter values — not `params`, `searchParams`, or unresolved server promises.

### Implicit return inside `.then()`

Use an implicit-return arrow function when the callback just renders JSX — e.g. `({ id }) => <PostDetail id={id} />`. Only switch to a block body with `return` when you need to do work first (destructure with defaults, parse a `searchParams` value, branch on a condition). This keeps the JSX-in-page shape readable and matches how the resolved tree will look.

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

Use `Promise.all([params, searchParams])` when both are needed. Avoid nested `.then()` calls that make the resolved tree hard to read and easy to wrap in the wrong boundary.

### Metadata, static params, and `notFound()`

- [`generateMetadata`](https://preview.nextjs.org/docs/app/api-reference/functions/generate-metadata) runs before render, so `await params` is fine there — it's a separate async function, not the page body, so it doesn't make the page dynamic.
- Export [`generateStaticParams`](https://preview.nextjs.org/docs/app/api-reference/functions/generate-static-params) from a `[slug]` page/layout to pre-build a known set of slugs; with `cacheComponents` + `'use cache'` they land in the static shell. It does **not** change the page signature — `params` is still a Promise, still consumed with `params.then()`.
- A query that can't find its resource calls [`notFound()`](https://preview.nextjs.org/docs/app/api-reference/functions/not-found), which bubbles to the nearest [`not-found.tsx`](https://preview.nextjs.org/docs/app/api-reference/file-conventions/not-found). Don't try/catch it — use [`unstable_rethrow`](https://preview.nextjs.org/docs/app/api-reference/functions/unstable_rethrow) if you must catch nearby.

## Prefer a page boundary over `loading.tsx`

Put the boundary in the page next to the `params.then()` / `searchParams.then()` it covers, rather than adding a route-segment `loading.tsx`. A page boundary keeps the fallback beside the JSX it stands in for, lets sibling sections share one boundary or split into several, and can be wrapped in `<ViewTransition>` so the reveal animates. A `loading.tsx` renders outside the page's own tree, so a transition wrapper inside the page cannot animate the swap out of it, and one fallback has to cover the whole route.

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

If a page uses a transition wrapper (e.g. `<ViewTransition>`), place it in the page next to the `<Suspense>` boundary. Feature components render content and skeletons, not transition wrappers.

A transition wrapper morphs the fallback into the content. That only looks right when the skeleton's box equals the content's box — a skeleton that is shorter or narrower shows up as a scaled ghost over the incoming content. Headings and labels that are the same in both states belong **outside** the boundary, otherwise they cross-fade with themselves and flicker. Sibling boundaries that resolve at different moments each start their own transition; that is fine as long as every skeleton is exact.

Give every transition-wrapped boundary a host DOM element (`div`, `section`, `main`) immediately outside it in the page. When a page's root is the wrapper itself, the content-side `<ViewTransition>` becomes the topmost entering subtree on a warm, prefetched navigation and its enter animation runs even though no fallback was ever shown. The host suppresses that while leaving the fallback-to-content reveal intact. Measure it: count `document.startViewTransition` calls on a prefetched navigation; a warm step change should produce zero.

## Stable shell, suspending body

Before designing a fallback, identify what is stable and what is data-dependent.

- Stable: card/panel border, padding, icon, label, section heading, fixed action rail.
- Data-dependent: title text, counts, form body, list rows, loaded choices.
- Rule: stable shell outside Suspense; skeleton/crossfade inside the shell around the data body.

```tsx
<FeaturePanel>
  <Suspense fallback={<FeaturePanelBodySkeleton />}>
    <Crossfade>
      <FeaturePanelBody id={id} />
    </Crossfade>
  </Suspense>
</FeaturePanel>
```

Do not render `<FeaturePanel>` in both `fallback` and final content. That duplicates layout responsibility and makes the skeleton guess the panel size.

Use the app's real domain noun at implementation time (`PostPanel`, `MessageList`, `GroupEditor`, `EventDetails`, etc.); the neutral names here describe the reusable shape, not a component to copy literally.

When the top data section has unknown final height and pushes the sections below, either reserve that height in the stable wrapper or group the affected sections in one boundary. Do not create two independent crossfades if the first one changes the second one's starting position.

## One route, several variants

A dynamic route that renders different UI per param value (a `[step]` wizard, a `[view]` toggle) has one prerendered shell for all variants, so its top-level fallback cannot know which variant is coming. Don't shape that fallback like one of the variants, and don't resolve the variant inside the fallback with pathname hooks or params promises behind nested `Suspense`. Use two levels:

1. The route boundary's fallback is a neutral splash: the variant's frame (card, board) with the animated brand mark centered, `role="status"` and an `aria-label`.
2. Inside `params.then(...)`, once the variant is known, a plain `<Suspense fallback={<VariantSkeleton ... />}>` wraps the data component with the exact skeleton for that variant.

On a hard load the splash shows for a frame or two, then the exact skeleton. On a client navigation the shell is in the router cache and the variant skeleton renders straight away, or the content does when it was prefetched. Splitting the wizard into one route per step to get per-route shells is not worth it: several near-identical pages, and the shared layout still owns the chrome.

## Audit smells

When auditing an existing app, flag and fix these first:

- `export default async function Page(...)` that only awaits `params`, `searchParams`, or page-level queries.
- `import { getSomething } from '@/features/.../*-queries'` inside `app/**/page.tsx` or `layout.tsx`.
- Feature components whose props are `params`, `searchParams`, or a route-shaped object.
- Page-local components like `HomeContent`, `PostShell`, or `ResultsSection` that only exist to fetch data or group a Suspense fallback.
- `<Suspense>` inside feature components that prevents the page from grouping reveal behavior.

## Route-local control-flow helpers

Small inline helpers are fine when their only job is route control flow that must live exactly where a boundary is placed:

```tsx
export default function LoginPage() {
  return (
    <Suspense>
      <LoginRedirect />
    </Suspense>
  );
}

async function LoginRedirect() {
  await connection();
  await redirectIfAuthenticated('/dashboard');
  return <LoginForm />;
}
```

Do not move a helper like this into a feature folder just to satisfy a blanket "no inline components" rule. It is not domain UI and it is not a reusable feature component; extracting it creates noise. Keep the cookie/session read inside the data-access helper (`redirectIfAuthenticated`, `verifyAuth`, etc.) rather than importing a feature query into the page.

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

The same applies to feature-level skeleton aliases. If a variant only passes props to a base skeleton, import the base skeleton and pass those props inline in `fallback={...}`.

## Suspense boundary placement rules

1. **First section gets its own Suspense** with a known-height skeleton fallback.
2. **Section headings stay outside Suspense** when their final position is stable.
3. **Stable wrappers stay outside Suspense.** If fallback and final content both render the same card or panel, lift that wrapper around the boundary.
4. **Variable-height sections: group everything below them** in the same Suspense, including any headings that would otherwise paint in the wrong vertical position.
5. **Fixed-height sections: own boundary is safe.**
6. **Variable-length lists: show 2–5 skeleton items**, not the real count.
7. **Inner Suspense content stays out of the outer skeleton.** Each boundary owns its own.
8. **Never `fallback={null}` for visible UI.** If a boundary covers UI, give it a real shaped fallback, or group it with a sibling boundary that already has the correct fallback.
9. **If the top section's final height is unknown, group the following sections** in the same boundary so they reveal together and don't jump underneath.
10. **Optional sections that may render nothing** (a "your next item" card, a promo slot) sit above other content only if the empty state reserves the same height as the filled state and the skeleton. Otherwise put the section last, or group what follows into its boundary.

## Error boundaries

Wrap fallible sections in a Next.js-aware error boundary so one failure doesn't take down the page. Build it on [`catchError`](https://preview.nextjs.org/docs/app/api-reference/functions/catchError) from `next/error` (its `ErrorInfo` gives you a `retry()` that re-fetches server data) — it understands Next's control-flow throws (`notFound()`, `redirect()`, `unauthorized()`, `forbidden()`) and won't swallow them. Place the boundary around the suspending section, in the page:

```tsx
<ErrorBoundary title="Replies didn't load">
  <Suspense fallback={<RepliesSkeleton />}>
    <Replies postId={id} />
  </Suspense>
</ErrorBoundary>
```

Why not plain `react-error-boundary`? It catches Next's framework throws (so `notFound()` never reaches `not-found.tsx`), and its reset doesn't re-fetch server data. Background: [Error Handling in Next.js with catchError](https://aurorascharff.no/posts/error-handling-in-nextjs-with-catch-error/).

Pair component-level boundaries with route-segment [`error.tsx`](https://preview.nextjs.org/docs/app/api-reference/file-conventions/error) for unrecoverable errors; it also receives a `retry` callback.

## Layout-level Suspense

Layouts compose feature components the same way pages do. Use `<Suspense>` for slots that fetch data (auth badge, sidebar). App-shell slots are the one place the layout, or the shell component it renders (a header, a sidebar), owns the boundary instead of the page — the slot repeats on every route, so its boundary belongs with the shell:

```tsx
export default function RootLayout({ children }: LayoutProps<'/'>) {
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

- Match skeleton height to the real content height — measure both in the browser rather than eyeballing; a 4px difference still jumps.
- Move headings **outside** boundaries when their position depends on data above them.
- For unknown-height top sections, group everything below in one boundary so siblings stream together.

To audit CLS, use React DevTools' Suspense panel to pin each boundary in its loading state and check vertical positions.

Pending indicators are part of CLS too. A spinner inserted into a button widens it and an `auto` grid column moves everything beside it; give such buttons a fixed width that fits the spinner. A chip or badge that appears with data (a count, a countdown) gets a pill-shaped skeleton of the same height.

## Optimizing prefetching for high-value routes

With `cacheComponents` + [`partialPrefetching`](https://preview.nextjs.org/docs/app/api-reference/config/next-config-js/partialPrefetching) enabled, a visible `<Link>` prefetches the destination's shared [App Shell](https://preview.nextjs.org/docs/app/glossary#app-shell) — enough to commit navigation instantly, with link-specific content streaming after. The default (`'auto'`) already does this; don't write `prefetch = 'auto'`.

Use `<Link prefetch={true}>` on high-value links to also resolve the destination's per-link data (`params`, `searchParams`, the full URL) at prefetch time. Each such link can wake the server for a prerender, so reserve it for routes users predictably visit next. See [Optimizing prefetching](https://preview.nextjs.org/docs/app/guides/optimizing-prefetching).

Can't enable `partialPrefetching` app-wide yet? Opt in per route with `export const prefetch = 'partial'` on the destination, then drop the per-route exports once the global flag is on — see [Adopting Partial Prefetching](https://preview.nextjs.org/docs/app/guides/adopting-partial-prefetching) for the incremental path and [prefetch config](https://preview.nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/prefetch) for the options. To check that navigation actually feels instant, see [Validating instant navigation](#validating-instant-navigation).

### Keep a live layer out of the prefetch without blocking it

A `prefetch={true}` prerender advances through everything cached and stops at the first uncached read. One live read (presence, live availability) therefore turns the whole destination back into its fallback. Split it: render the section from cached, URL-keyed data, and read the live layer in a sibling that first `await`s [`unstable_navigation()`](https://preview.nextjs.org/docs/app/guides/optimizing-prefetching) from `next/cache` and then calls a `'use cache: private'` function. Content below the gate is skipped by prefetches but still counts as cacheable, and it runs and streams in on the actual navigation. `await connection()` or `io()` would instead mark the subtree dynamic and drop it from the App Shell too.

Hand the gated value to the client as a promise and `use()` it inside a small `<Suspense>` whose fallback is the same UI without the live layer (items in their cached state, then marked with the live status), so its arrival changes state, not layout. `unstable_navigation()` cannot be called inside a cache scope; keep it in the uncached wrapper, with the cache directive on the function below it.

Two things defeat this prefetch. A server `redirect()` to a canonical URL inside the destination (for example to stamp derived data into the search params) is not followed by the prefetch and re-renders the whole tree on navigation, so write derived URL state from the links the user clicks instead. And chrome that must survive the navigation, such as a progress indicator, belongs in the layout and reads only the URL (`useParams` / `useSearchParams` under its own `Suspense`), never data that would make it re-suspend.

## Validating instant navigation

With `cacheComponents` on, Next.js already validates every Page and Default segment in development. You don't add `export const instant` or `experimental.instantInsights` to switch that on — read what it reports and fix it with the rules above: cache the read, or narrow the boundary.

Reach for [`export const instant = false`](https://preview.nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant#disabling-instant) only as an escape hatch — to exempt a blocking ancestor layout while still asserting the pages beneath it, or to opt a route out of static-shell validation. It can't be used in a Client Component.

To see what actually lands in the initial UI, use the [Navigation Inspector](https://preview.nextjs.org/docs/app/api-reference/file-conventions/route-segment-config/instant#inspecting-loading-states). To keep it from regressing, lock it in with [`instant()` from `@next/playwright`](https://preview.nextjs.org/docs/app/guides/instant-navigation#prevent-regressions-with-e2e-tests).

## Never wrap the entire page in a Suspense fallback

Page chrome (header, nav, surrounding layout) should paint instantly. Only data-dependent sections suspend. If you find yourself wrapping `<div>` and everything in it with `<Suspense fallback={<FullPageSkeleton />}>`, restructure: pull static elements out, narrow the boundary to just the dynamic part.
