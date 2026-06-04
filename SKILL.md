---
name: nextjs-app-architecture
description: Architecture patterns for Next.js 16 App Router apps. Use when scaffolding a new app, adding a feature, refactoring code into feature folders, deciding where queries/actions/components live, placing Suspense boundaries, choosing the client/server boundary, designing skeletons, preventing CLS, or enabling Cache Components. Also use when the user asks about RSC composition, `params.then()`, `'use cache'`, `cacheTag`, `updateTag`, or static-shell prerendering.
license: MIT
metadata:
  author: aurorascharff
  version: '1.0.1'
---

# Next.js App Architecture

Use when building or refactoring Next.js 16+ App Router apps. The skill is organized into focused references — load only the ones relevant to the current task.

## When to read which reference

| Task                                                                      | Read                             |
| ------------------------------------------------------------------------- | -------------------------------- |
| Creating a new feature, deciding folder structure, naming files           | `references/feature-folders.md`  |
| Writing a query or server action, invalidating cached data                | `references/queries-actions.md`  |
| Building a server/client component, designing a skeleton, using `use()`   | `references/components.md`       |
| Composing a page, placing `<Suspense>`, preventing CLS                    | `references/pages-suspense.md`   |
| Turning on `cacheComponents`, adding `'use cache'`, static shell          | `references/cache-components.md` |
| Toasts, pending state, destructive flows, action-prop pattern, pagination | `references/ux-patterns.md`      |

Read references **in addition to**, not instead of, this overview. Each one assumes the rules below already apply.

## The model

Next.js 16+ App Router with React Server Components. Three high-level patterns shape everything else:

1. **Feature-sliced layout** — domain folders under `features/`, each owning its queries, actions, and components. Pages in `app/` compose features; they never contain domain logic.
2. **Async server components by default** — components `await` their own queries directly. Client components (`'use client'`) are leaves, not parents, of the tree.
3. **Suspense at the page** — the feature exports content + a sibling skeleton. The page imports both and places the `<Suspense>` boundary.

## Decision rules (always apply)

These rules apply to every change. If you violate one, you'll fight the framework later.

- **Pages never fetch data directly.** They compose feature components.
- **Pages stay synchronous.** Use `params.then()` instead of `await params` so page chrome paints into the static shell. See `references/pages-suspense.md`.
- **Queries live in `<domain>-queries.ts`** with `import 'server-only'` and `cache()` wrapping every export. See `references/queries-actions.md`.
- **Actions live in `<domain>-actions.ts`** with `'use server'` at the top. The file name matches the folder, even when the mutation targets a sub-concept. See `references/feature-folders.md` and `references/queries-actions.md`.
- **Async server component is the default.** Add `'use client'` only when you need hooks, event handlers, or browser APIs. See `references/components.md`.
- **The page owns the Suspense boundary; the feature owns the skeleton.** Don't pre-wrap components in `<Suspense>` inside the feature. See `references/pages-suspense.md`.
- **Skeletons live in the same file as the component.** `Feed` and `FeedSkeleton` are sibling exports. See `references/components.md`.
- **Single-use sub-components stay inlined as non-exported functions** in the same file. Exports are for things other files import. See `references/components.md`.
- **Cache Components is opt-in.** If `cacheComponents: true` is set, every async operation must be inside `<Suspense>` or marked `'use cache'`. See `references/cache-components.md`.

## Decision flow for a new feature

```
1. Does a feature folder for this domain already exist?
   → Yes: use it. Don't make a new one.
   → No: is this a real domain noun, or a sub-concept of an existing one?
        → Sub-concept (favorite, like, vote, bookmark): fold into the parent feature.
        → Real domain: create features/<domain>/.

2. Add the query
   → features/<domain>/<domain>-queries.ts
   → Wrap in cache(), add 'use cache' + cacheTag if using Cache Components.
   → See references/queries-actions.md.

3. Add the action (if there's a mutation)
   → features/<domain>/<domain>-actions.ts
   → 'use server' at the top, validate input, updateTag() to invalidate.

4. Build the component
   → features/<domain>/components/<name>.tsx
   → async server component that awaits its query
   → Export <Name> and <NameSkeleton> from the same file.

5. Compose the page
   → app/<route>/page.tsx
   → Keep the page synchronous, use params.then().
   → Place <Suspense fallback={<NameSkeleton />}><Name /></Suspense>.
```

## Pitfalls

- **Passing server actions as props to call them.** Client components import actions directly.
- **Refetching what the parent already has.** Server components take plain values, not promises. If `<Feed>` queried the list, pass each `post` to `<Post post={post} />`, don't refetch by id.
- **Inlining route-specific components in the page file.** Extract them into the feature folder. The page should not grow past composition.
- **Splitting a card and its grid into separate files.** They're always used together. One file, multiple exports.
- **Making a feature folder for one query, one action, one button.** Fold it into the parent feature.
- **Using `'use cache'` without `cacheComponents: true`.** They go together. See `references/cache-components.md`.
- **Wrapping the entire page in a Suspense fallback.** Page chrome paints instantly, only data-dependent sections suspend. See `references/pages-suspense.md`.

## Reference index

- **`references/feature-folders.md`** — Folder layout, naming, merging sub-concepts, when a new folder is justified, action/query file naming.
- **`references/queries-actions.md`** — Server-only queries, `cache()` for dedup, `'use cache'` + `cacheTag` + `cacheLife`, server actions, `updateTag()` / `revalidateTag()` / `refresh()`.
- **`references/components.md`** — Async server components, skeletons, client boundary, promise + `use()` pattern, single-use helpers, polling for live data.
- **`references/pages-suspense.md`** — Page composition, `params.then()`, Suspense placement rules, CLS prevention, error boundaries, layout-level Suspense.
- **`references/cache-components.md`** — `cacheComponents: true` model, the static shell, build constraints, `'use cache'` on components vs queries.
- **`references/ux-patterns.md`** — Toasts, pending state via `data-pending`, destructive action flows, the action-prop pattern, URL pagination, escape hatches for `usePathname`/`useSearchParams`, `useFormStatus`.
