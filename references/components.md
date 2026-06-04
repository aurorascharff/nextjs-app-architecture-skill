# Components

How to build server and client components inside a feature folder.

## Default: async server component

Server components await their own queries directly — no `useEffect`, no client-side fetching, no manual loading state. See the [Server Components docs](https://nextjs.org/docs/app/getting-started/server-and-client-components) for the model.

```tsx
// features/notifications/components/notifications-badge.tsx
import { getUnreadNotificationCount } from '@/features/notifications/notifications-queries';

export async function NotificationsBadge() {
  const count = await getUnreadNotificationCount();
  if (count === 0) return null;
  return <span aria-label={`${count} unread`}>{count}</span>;
}
```

Wrapped at the page or layout level with Suspense:

```tsx
<Suspense fallback={<NotificationsBadgeSkeleton />}>
  <NotificationsBadge />
</Suspense>
```

## Skeletons live in the same file

Export the main component and its skeleton from the same file. Pages import both.

```tsx
export async function Feed({ userId }: { userId: string }) {
  const posts = await getFeed(userId);
  return (
    <ul>
      {posts.map(p => (
        <Post key={p.id} post={p} />
      ))}
    </ul>
  );
}

export function FeedSkeleton() {
  return (
    <ul>
      {Array.from({ length: 3 }).map((_, i) => (
        <li key={i}>
          <Skeleton className="h-24" />
        </li>
      ))}
    </ul>
  );
}
```

### Skeleton design checklist

1. Match the real component's layout: flex direction, gaps, padding, breakpoints.
2. Include all structural elements: avatar circles, action button placeholders, image squares.
3. Responsive visibility must match (`hidden sm:block` in the real component → same in the skeleton).
4. Show 2–5 placeholders for variable-length lists, not the real count.
5. Don't include skeletons for inner Suspense content — those have their own boundaries.
6. Reserve the right height. CLS comes from skeletons that are shorter than the real content.

## Group related components in one file

A card and its grid live in the same file. For example, `genre-card.tsx` exports `GenrePill`, `GenreCard`, `GenreGrid`, `GenreGridSkeleton`. Don't split shared UI primitives prematurely — wait until three call sites need the same shape before extracting.

Two sidebar widgets that happen to look similar but render different data shapes are **not** the same component. The visuals diverge as soon as one needs an extra slot.

### Single-use sub-components stay inlined

For a metadata strip inside one card, a header used only by one detail view, a list item only rendered by its list — inline them as **non-exported** functions in the same file:

```tsx
export async function EventDetails({ slug }: { slug: string }) {
  const event = await getEventBySlug(slug);
  return (
    <article>
      <MetaStrip event={event} />
      <Speaker speaker={event.speaker} />
      <p>{event.description}</p>
    </article>
  );
}

function MetaStrip({ event }: { event: Event }) { ... }
function Speaker({ speaker }: { speaker: string }) { ... }
```

Exports are for things other files will import. Internal structure is for readability inside one file.

## The server/client boundary

`'use client'` only when you need:

- Hooks (`useState`, `useReducer`, `useOptimistic`, `useTransition`, `useEffect`)
- Event handlers (`onClick`, `onChange`, `onSubmit`)
- Browser APIs (`window`, `localStorage`, refs to DOM)

If the component needs interactive pieces, keep the server component as the parent and render client leaves:

```tsx
async function PostDetail({ id }: { id: string }) {
  const [post, userState] = await Promise.all([getPost(id), getPostUserState(id)]);
  return (
    <article>
      <PostBody body={post.body} />
      <PostActions userState={userState} /> {/* 'use client' leaf */}
    </article>
  );
}
```

### Server content as children of client components

Composition crosses the boundary. A client component can accept server-rendered JSX as children or props:

```tsx
<ComposerForm
  avatar={
    <Suspense fallback={<AvatarSkeleton />}>
      <CurrentUserAvatar />
    </Suspense>
  }
/>
```

`ComposerForm` is `'use client'`. It doesn't know where the avatar JSX came from. The Suspense boundary streams the avatar in without the form re-rendering.

### Server components don't take promises as props

Pass plain values (strings, IDs, resolved data). When a parent already has the data from its own query, pass it as a prop instead of having the child refetch.

```tsx
// Right — parent fetches the list, passes each item
async function Feed({ userId }: { userId: string }) {
  const posts = await getFeed(userId);
  return posts.map(post => <Post key={post.id} post={post} />);
}

async function Post({ post }: { post: Post }) {
  return <article>{post.body}</article>;
}
```

```tsx
// Wrong — child refetches what the parent already had
async function Post({ id }: { id: string }) {
  const post = await getPost(id);
  return <article>{post.body}</article>;
}
```

## Client components that own their loading state

When a client component needs server data but should manage its own loading (a sidebar badge, a popover that opens on hover), pass an **unresolved promise** from the server and resolve it with [`use()`](https://react.dev/reference/react/use) on the client. Wrap the consumer in `<Suspense>`.

```tsx
// Server side — don't await the query
<Suspense fallback={<TagListSkeleton />}>
  <TagPicker itemsPromise={getTags()} />
</Suspense>
```

```tsx
'use client';
import { use } from 'react';

export function TagPicker({ itemsPromise }: { itemsPromise: Promise<Tag[]> }) {
  const items = use(itemsPromise);
  // ...
}
```

The opinionated bit: name promise props with a `Promise` suffix (`itemsPromise`, `userPromise`) so the contract is obvious at the call site.

## Caching a whole component

If the entire component output can be cached, put `'use cache'` on the component directly instead of on the query:

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

This caches the rendered HTML, not just the query result. Useful for components with expensive rendering or shared content across users. See `references/cache-components.md` for when this is safe.

## Live data via polling

For features that reflect server-side updates without user action (other users posting, new notifications, vote counts changing), drop a `<Poller>` client component into the page that calls [`router.refresh()`](https://nextjs.org/docs/app/api-reference/functions/use-router) on an interval. Combined with `'use cache'` + `cacheLife('seconds')`, queries return cached data until they expire, then the next refresh picks up new data. No WebSockets, no SSE — just the existing cache cycle.

## `useOptimistic` for mutations

For instant feedback on mutations that are unlikely to fail (favorite, vote, follow), use [`useOptimistic`](https://react.dev/reference/react/useOptimistic):

```tsx
'use client';
import { useOptimistic, useTransition } from 'react';

export function FavoriteButton({ slug, favorited }: { slug: string; favorited: boolean }) {
  const [optimisticFavorited, setOptimisticFavorited] = useOptimistic(favorited);
  const [, startTransition] = useTransition();

  function handleClick() {
    startTransition(async () => {
      setOptimisticFavorited(!favorited);
      await toggleFavorite(slug);
    });
  }

  return <button onClick={handleClick}>{optimisticFavorited ? '★' : '☆'}</button>;
}
```

Key rules (the rest is in the React docs):

- Call `setOptimistic` inside a [transition](https://react.dev/reference/react/useTransition), not as a free update.
- Inside `<form action={…}>`, React opens the transition for you — you can call `setOptimistic` directly in the action body.
- For counter-style updates, pass a reducer as the second argument so the next value is derived from the current one (vote counts, like counts).

For non-optimistic pending UI (filters, sort changes, deferred work), use `useTransition` with the `data-pending` pattern in `references/ux-patterns.md`. For success/error feedback rules, see the same reference.

For the deeper picture (coordinating `useTransition`, `useOptimistic`, `useActionState`, `data-pending`, Suspense streaming, and caching across server and client), see the [Interactive Apps guide PR](https://github.com/vercel/next.js/pull/94020) — not merged yet, but the canonical reference until it lands at `nextjs.org/docs/app/guides/interactive-apps`.
