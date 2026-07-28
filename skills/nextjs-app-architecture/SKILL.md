---
name: nextjs-app-architecture
description: Architecture patterns for Next.js 16 App Router apps. Use when scaffolding a new app, adding a feature, refactoring code into feature folders, deciding where queries/actions/components live, placing Suspense boundaries, choosing the client/server boundary, designing skeletons, preventing CLS, or enabling Cache Components. Also use when the user asks about RSC composition, `params.then()`, `'use cache'`, `cacheTag`, `updateTag`, or static-shell prerendering.
license: MIT
metadata:
  author: aurorascharff
  version: '1.3.0'
---

# Next.js App Architecture

A workflow for building and refactoring Next.js 16+ App Router apps so they follow one consistent, feature-sliced RSC architecture.

**Follow the workflow below step by step** — it produces the invariants by construction. Load the reference a step names for the decision it depends on. Get framework *mechanics* (API signatures, config options, hook contracts) from the linked docs — don't restate or improvise them.

## Invariants (what every change must satisfy)

The non-negotiables. The workflow produces them; the final check verifies them.

1. **Pages compose, they never fetch.** A page/layout imports feature components and places `<Suspense>`. No queries, no domain logic, no route-specific components defined inline.
2. **Pages stay synchronous.** Use `params.then()` / `searchParams.then()`, never `await params` at the top — so chrome paints into the static shell and only data-dependent sections suspend.
3. **Async server component is the default.** `'use client'` only for hooks, event handlers, or browser APIs — and only on leaves, never on parents of server content.
4. **The page owns the Suspense boundary; the feature owns the skeleton.** Features never pre-wrap themselves in `<Suspense>`.
5. **Skeletons live in the same file as the component**, exported alongside it, defined at the end. `Feed` and `FeedSkeleton` are siblings.
6. **Queries live in `<domain>-queries.ts`** (`import 'server-only'`); **actions live in `<domain>-actions.ts`** (`'use server'`). The file name matches the folder, even for sub-concepts.
7. **One feature folder per real domain noun.** Sub-concepts (favorite, like, vote, bookmark, search) fold into the parent feature, never their own folder.
8. **Client components import actions directly** — never receive a server action as a prop just to call it.

## Workflow

Run these in order for any feature work. Each step names the reference to consult and the check it must pass.

1. **Place the work.** Decide the feature folder before writing anything.
   → `references/feature-folders.md` (decision tree + merge rules).
   ✓ A real domain, or folded into the right parent.
2. **Write the query.** `features/<domain>/<domain>-queries.ts`, `import 'server-only'`.
   → `references/queries-actions.md`; if the data is cacheable, → `references/cache-components.md`.
   ✓ Server-only; shared reads deduped; returns domain types, not ORM rows.
3. **Write the action** (if there's a mutation). `features/<domain>/<domain>-actions.ts`, `'use server'` at the top.
   → `references/queries-actions.md`.
   ✓ Re-checks auth, validates input, invalidates (`refresh()`, or `updateTag()` once the query is tagged), returns a discriminated union.
4. **Build the component + skeleton.** `features/<domain>/components/<name>.tsx`: an async server component that awaits its own query; `'use client'` only on interactive leaves.
   → `references/components.md`.
   ✓ Skeleton is a sibling export at the end of the file.
5. **Compose the page.** `app/<route>/page.tsx`: synchronous, `params.then()`, place `<Suspense fallback={<NameSkeleton />}><Name /></Suspense>`, and wrap fallible sections in an error boundary.
   → `references/pages-suspense.md`.
   ✓ The page only composes; the boundary lives here, not in the feature.
6. **Add interaction** (if any): optimistic updates, pending state, toasts, confirmation.
   → `references/ux-patterns.md`.
   ✓ Feedback isn't doubled; destructive actions confirm.
7. **Verify** against the checklist below before declaring done.

## Verify before done

Inspect the diff against every invariant — each is checkable by reading the changed files:

- [ ] No page/layout imports a `*-queries` file or defines a route-specific component inline.
- [ ] Every page with params is synchronous and uses `params.then()` / `searchParams.then()`.
- [ ] Every `<Suspense>` for page data sits in the page; no feature pre-wraps itself.
- [ ] Every component has its `*Skeleton` in the same file, at the end.
- [ ] Every `*-queries.ts` starts with `import 'server-only'`; every `*-actions.ts` with `'use server'`.
- [ ] Action files are named `<folder>-actions.ts`; no sub-concept spawned its own folder.
- [ ] `'use client'` components are leaves — they import actions/hooks/providers, not async server components.
- [ ] Mutations validate their input and invalidate (`refresh()` or a tag).

## Reference index

- **`references/feature-folders.md`** — where code goes: folder layout, naming, merging sub-concepts.
- **`references/queries-actions.md`** — query/action rules: server-only, dedup, validation, invalidation, return shape.
- **`references/components.md`** — server/client boundary, skeletons, `use()`, single-use helpers, live data.
- **`references/pages-suspense.md`** — page composition, `params.then()`, Suspense placement, CLS, error boundaries, prefetch.
- **`references/cache-components.md`** — the `cacheComponents` decisions: whether to cache, which directive, how to invalidate.
- **`references/ux-patterns.md`** — interaction decisions: optimistic vs pending vs inline error, toasts, action-prop, confirmations.
- **`references/example.md`** — the next-beats reference app: invariant → file map, for seeing any rule in real code.
