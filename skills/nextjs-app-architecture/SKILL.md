---
name: nextjs-app-architecture
description: Build or audit Next.js 16 App Router apps using a next-beats-style React Server Components architecture. Use when scaffolding a new app, adding a feature, reviewing an existing app, refactoring route-loader-shaped pages into feature-owned async server components, deciding where queries/actions/components live, keeping pages synchronous with `params.then()`, placing Suspense boundaries, choosing the client/server boundary, designing skeletons, preventing CLS, or enabling Cache Components. Also use when the user asks about RSC composition, components receiving IDs instead of route params, `'use cache'`, `cacheTag`, `updateTag`, static-shell prerendering, or making an app easier for AI agents to modify.
license: MIT
metadata:
  author: aurorascharff
  version: "1.3.6"
---

# Next.js App Architecture

A workflow for building and auditing Next.js 16+ App Router apps so they follow one consistent, feature-sliced RSC architecture like `next-beats`.

**Follow the workflow below step by step** — it produces the invariants by construction. Load the reference a step names for the decision it depends on. Get framework _mechanics_ (API signatures, config options, hook contracts) from the linked docs — don't restate or improvise them.

## Prerequisite

Before changing a Next.js app, make sure the project is set up for AI agents to read version-matched docs. Follow the [AI Coding Agents guide](https://preview.nextjs.org/docs/app/guides/ai-agents): prefer the project's `AGENTS.md` / bundled docs, and create or refresh them when missing. Then use this skill for architecture decisions.

## Architecture target

Build pages that describe the loading experience, not pages that act like route loaders:

- `app/**/page.tsx` and `layout.tsx` are synchronous composition surfaces: static chrome, section headings, `<Suspense>` boundaries, error boundaries, and transition wrappers.
- Feature components own their reads on the server. They receive minimal stable inputs (`id`, `slug`, `handle`, parsed filter values) or already-fetched records, never raw `params` / `searchParams`.
- Queries and actions live in the feature folder. Components import queries; client leaves import actions directly.
- Skeletons mirror the component tree and live beside the component they represent.

## Invariants (what every change must satisfy)

The non-negotiables. The workflow produces them; the final check verifies them.

1. **Pages compose, they never fetch.** A page/layout imports feature components and places `<Suspense>`. No queries, no domain logic, no route-specific components defined inline.
2. **Pages stay synchronous.** Use `params.then()` / `searchParams.then()`, never `await params` at the top — so chrome paints into the static shell and only data-dependent sections suspend.
3. **Feature components receive IDs, not route props.** Resolve `params` / `searchParams` at the page boundary and pass plain values (`id`, `slug`, `query`) into features.
4. **Async server component is the default.** `'use client'` only for hooks, event handlers, or browser APIs — and only on leaves, never on parents of server content.
5. **The page owns the Suspense boundary; the feature owns the skeleton.** Features never pre-wrap themselves in `<Suspense>`.
6. **Skeletons live in the same file as the component**, exported alongside it, defined at the end. `Feed` and `FeedSkeleton` are siblings.
7. **Queries live in `<domain>-queries.ts`** (`import 'server-only'`); **actions live in `<domain>-actions.ts`** (`'use server'`). The file name matches the folder, even for sub-concepts.
8. **One feature folder per real domain noun.** Sub-concepts (favorite, like, vote, bookmark, search) fold into the parent feature, never their own folder.
9. **Client components import actions directly** — never receive a server action as a prop just to call it.

## Workflow

Run these in order for build-from-scratch, feature work, or audits. Each step names the reference to consult and the check it must pass.

1. **Choose mode.**
   - **Build from scratch:** sketch routes, real domain nouns, static shell, and expected loading groups before writing code.
   - **Audit/refactor:** scan current `app/` pages first; list every async page, page-level query import, route prop leak, missing Suspense boundary, and feature folder mismatch.
     → `references/example.md` for the target shape; `references/feature-folders.md` for placement.
     ✓ You know whether you are creating the architecture or converting loader-shaped code into it.
2. **Place the work.** Decide the feature folder before writing anything.
   → `references/feature-folders.md` (decision tree + merge rules).
   ✓ A real domain, or folded into the right parent.
3. **Write the query.** `features/<domain>/<domain>-queries.ts`, `import 'server-only'`.
   → `references/queries-actions.md`; with `cacheComponents: true`, also → `references/cache-components.md`.
   ✓ Server-only; reusable reads cached/tagged/lifetimed under Cache Components; React `cache()` only for proven same-request dedup; returns domain types, not ORM rows.
4. **Write the action** (if there's a mutation). `features/<domain>/<domain>-actions.ts`, `'use server'` at the top.
   → `references/queries-actions.md`.
   ✓ Re-checks auth, validates input, invalidates matching cache tags under Cache Components (`refresh()` only for justified dynamic reads), returns a discriminated union.
5. **Build the component + skeleton.** `features/<domain>/components/<name>.tsx`: an async server component that awaits its own query from minimal props; `'use client'` only on interactive leaves.
   → `references/components.md`.
   ✓ Component receives IDs/handles/parsed filters or already-resolved records, not `params`; skeleton is a sibling export at the end; no alias skeleton wrappers.
6. **Compose the page.** `app/<route>/page.tsx`: synchronous, `params.then()`, place `<Suspense fallback={<NameSkeleton />}><Name /></Suspense>`, and wrap fallible sections in an error boundary.
   → `references/pages-suspense.md`.
   ✓ The page only composes; the boundary lives here, not in the feature; route props are resolved to plain values before reaching feature components.
7. **Add interaction** (if any): optimistic updates, pending state, toasts, confirmation.
   → `references/ux-patterns.md`.
   ✓ Feedback isn't doubled; destructive actions confirm.
8. **Verify** against the checklist below before declaring done.

## Verify before done

Inspect the diff against every invariant — each is checkable by reading the changed files:

- [ ] No page/layout imports a `*-queries` file or defines a route-specific component inline.
- [ ] Every page with params is synchronous and uses `params.then()` / `searchParams.then()`.
- [ ] Feature components receive plain IDs/handles/parsed filters or resolved records; no feature prop is named `params` or `searchParams`.
- [ ] Every `<Suspense>` for page data sits in the page; no feature pre-wraps itself.
- [ ] Every component has its real `*Skeleton` in the same file, at the end; no tiny skeleton aliases just to pass props.
- [ ] Every `*-queries.ts` starts with `import 'server-only'`; every `*-actions.ts` with `'use server'`.
- [ ] With `cacheComponents: true`, reusable reads use `'use cache'` / `cacheTag` / `cacheLife`, or `'use cache: private'` / `'use cache: remote'` when appropriate; any dynamic read is intentional and justified.
- [ ] Mutations touching cached reads call `updateTag()` / `revalidateTag(..., 'max')` for the matching tags; `refresh()` is not a substitute for tag invalidation.
- [ ] Action files are named `<folder>-actions.ts`; no sub-concept spawned its own folder.
- [ ] `'use client'` components are leaves — they import actions/hooks/providers, not async server components.
- [ ] Mutations validate their input and invalidate the affected data.

## Reference index

- **`references/feature-folders.md`** — where code goes: folder layout, naming, merging sub-concepts.
- **`references/queries-actions.md`** — query/action rules: server-only, dedup, validation, invalidation, return shape.
- **`references/components.md`** — server/client boundary, skeletons, `use()`, single-use helpers, live data.
- **`references/pages-suspense.md`** — page composition, `params.then()`, Suspense placement, CLS, error boundaries, prefetch.
- **`references/cache-components.md`** — the `cacheComponents` decisions: which reads to cache, which directive to use, how to invalidate.
- **`references/ux-patterns.md`** — interaction decisions: optimistic vs pending vs inline error, toasts, action-prop, confirmations.
- **`references/example.md`** — the next-beats reference app: invariant → file map, for seeing any rule in real code.
