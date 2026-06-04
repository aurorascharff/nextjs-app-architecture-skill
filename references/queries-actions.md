# Queries and actions

The data layer. Every feature has both: queries to read, actions to write.

This page covers the universal data layer that applies to every Next.js App Router app. For the opt-in Cache Components model (`'use cache'`, `cacheTag`, `cacheLife`, `updateTag`), see `references/cache-components.md`.

## Queries

Create `features/<domain>/<domain>-queries.ts`. Mark it `import 'server-only'`. Wrap every export in [`cache()`](https://react.dev/reference/react/cache) from React for **request-level deduplication** — same query called from multiple components in the same render hits the database once.

```ts
import 'server-only';
import { cache } from 'react';

export const getFeed = cache(async (userId: string) => {
  return db.post.findMany({ where: { userId } });
});
```

That's the floor: every query is server-only and request-deduplicated. If you turn on Cache Components, you'll also add `'use cache'` + `cacheTag` to share results across requests — see `references/cache-components.md`.

## Actions

Create `features/<domain>/<domain>-actions.ts`. Mark with `'use server'` at the top. Always:

1. Verify auth.
2. Validate input with your schema validator.
3. Run the mutation.
4. Invalidate cached data so the next render sees the new state.
5. Return a result (`{ ok }` or `{ error }`).

```tsx
'use server';

import { refresh } from 'next/cache';

export async function createPost(formData: FormData) {
  const user = await verifyUser();
  const parsed = schema.safeParse({ body: formData.get('body') });
  if (!parsed.success) {
    return { ok: false as const, error: parsed.error.issues[0].message };
  }

  await db.post.create({ data: { body: parsed.data.body, userId: user.id } });
  refresh();
  return { ok: true as const };
}
```

[`refresh()`](https://nextjs.org/docs/app/api-reference/functions/refresh) re-renders the current route for the current user. With Cache Components, you'd swap this for `updateTag('feed')` to invalidate everything tagged `feed` and get read-your-own-writes across users — see `references/cache-components.md`.

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

Design components (`<BottomNav>`, `<ToggleGroup>`, `<SubmitButton>`) take this further with the **action-prop pattern** — `action` is a callback wrapped in `useTransition` / `useOptimistic` internally. See `references/ux-patterns.md`.

## Form actions vs onClick handlers

Prefer [`<form action={serverAction}>`](https://react.dev/reference/react-dom/components/form#action) for form mutations — React wraps the call in a transition and surfaces pending state automatically.

For one-off buttons, `onClick={() => action(args)}` is fine. Wrap in [`startTransition`](https://react.dev/reference/react/startTransition) if you need pending state.

## Return shape

Return a discriminated union from actions that can fail:

```tsx
export type ActionResult<T = void> = { ok: true; data: T } | { ok: false; error: string };
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
