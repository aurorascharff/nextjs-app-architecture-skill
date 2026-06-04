# Queries and actions

The data layer. Every feature has both: queries to read, actions to write.

## Queries

Create `features/<domain>/<domain>-queries.ts`. Mark it `import 'server-only'`. Wrap every export in `cache()` from React for **request-level deduplication**.

```ts
import 'server-only';
import { cache } from 'react';

export const getFeed = cache(async (userId: string) => {
  return db.post.findMany({ where: { userId } });
});
```

Without `cache()`, the same query called from multiple components in the same render hits the database multiple times.

### With Cache Components

If `cacheComponents: true` is enabled, add `'use cache'` + `cacheTag` + `cacheLife` to mark the query as cacheable across requests:

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

- `cacheTag` lets you invalidate the cache by tag from a mutation. Use both a global tag (`'feed'`) and a scoped one (`` `feed-${userId}` ``) so you can invalidate at either granularity.
- `cacheLife` sets the staleness profile. `'seconds'`, `'minutes'`, `'hours'`, or a custom profile.
- `'use cache: private'` scopes the cache per user when the query reads cookies or session data.

See `references/cache-components.md` for the full Cache Components model.

### Without Cache Components

If `cacheComponents` is not enabled, skip `'use cache'`. Keep `cache()` for request-level dedup. Invalidate via `refresh()` from actions instead of `updateTag()`.

## Actions

Create `features/<domain>/<domain>-actions.ts`. Mark with `'use server'` at the top. Always:

1. Verify auth.
2. Validate input with your schema validator.
3. Run the mutation.
4. Invalidate cached data.
5. Return a result (`{ ok }` or `{ error }`).

```tsx
'use server';

import { updateTag } from 'next/cache';

export async function createPost(formData: FormData) {
  const user = await verifyUser();
  const parsed = schema.safeParse({ body: formData.get('body') });
  if (!parsed.success) {
    return { ok: false as const, error: parsed.error.issues[0].message };
  }

  await db.post.create({ data: { body: parsed.data.body, userId: user.id } });
  updateTag('feed');
  return { ok: true as const };
}
```

The `cacheTag` in the query and the `updateTag` in the action live in the same feature folder. This is the full cycle: **tag, cache, invalidate**.

### `updateTag` vs `revalidateTag` vs `refresh`

- **`updateTag(tag)`** — invalidates immediately and re-renders for read-your-own-writes. Use inside server actions when the user expects to see the result.
- **`revalidateTag(tag)`** — stale-while-revalidate. Use in route handlers (webhooks, cron) where you don't need an immediate UI update.
- **`refresh()`** — re-renders the current route for the current user. Use when there's no `cacheTag` to invalidate (e.g. Cache Components is off, or the data is uncached).

### Action file naming

Actions for a feature always go in `<folder>-actions.ts`, matching the folder name — even when the mutation operates on a sub-concept. `toggleFavorite` in `features/event/` lives in `event-actions.ts`, not `favorite-actions.ts`. The folder is the source of truth for the name.

## Calling actions from client components

Client components import server actions directly. **Don't** pass an action as a prop just to call it:

```tsx
// Right
'use client';
import { likePost } from '@/features/post/post-actions';

export function LikeButton({ postId }: { postId: string }) {
  return <button onClick={() => likePost(postId)}>Like</button>;
}
```

```tsx
// Wrong — adds indirection with no benefit
async function Post({ id }: { id: string }) {
  return <LikeButton postId={id} onLike={likePost} />;
}
```

### Exception: parameter-bound actions

When the child shouldn't know about the parent's id, bind it at the call site:

```tsx
<DeleteButton onAction={deletePost.bind(null, post.id)} />
```

This is fine — the action prop now takes no arguments, the child just calls it.

Design components (`<BottomNav>`, `<ToggleGroup>`, `<SubmitButton>`) take this further with the **action-prop pattern** — `action` is a callback wrapped in `useTransition` / `useOptimistic` internally. See `references/ux-patterns.md`.

## Form actions vs onClick handlers

For mutations from a form, prefer `<form action={serverAction}>`. React wraps the call in a transition automatically and handles pending state.

For one-off buttons, `onClick={() => action(args)}` is fine. Wrap in `startTransition` if you need pending state.

## Return shape

Return a discriminated union from actions that can fail:

```tsx
export type ActionResult<T = void> =
  | { ok: true; data: T }
  | { ok: false; error: string };
```

Toast on `ok: false` from the client. Skip success toasts when an optimistic UI already shows the result.

## Mappers and domain types

If your DB rows have shapes you don't want to leak to components (extra columns, ORM-specific types), write a mapper inside the query:

```ts
export const getPost = cache(async (id: string) => {
  const row = await db.post.findUnique({ where: { id }, include: { author: true } });
  if (!row) notFound();
  return toPost(row);
});

function toPost(row: PostRow & { author: UserRow }): Post {
  return { id: row.id, body: row.body, author: row.author.handle };
}
```

Components see `Post`, not the ORM row. Where the `Post` type lives (a `types/` folder, co-located with the query, etc.) is a project convention — set it in `AGENTS.md`.
