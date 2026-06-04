# Next.js App Architecture Skill

An agent skill for structuring Next.js 16+ App Router apps with feature-sliced design, Suspense streaming, and optional Cache Components.

## What This Skill Covers

- **Feature folders** — when to create one, when to merge sub-concepts into a parent, file naming
- **Queries** — `cache()` for dedup, `'use cache'` + `cacheTag` + `cacheLife` for Cache Components
- **Actions** — `'use server'`, validation, `updateTag()` invalidation, calling from client components
- **Components** — async server components, skeletons, single-use helpers, the client boundary, `use()` + promise props, live data via polling
- **Pages** — `params.then()` for static-shell preservation, Suspense boundary placement, CLS prevention
- **Cache Components** — when to opt in, the static shell model, build constraints

## Skill Structure

```
nextjs-app-architecture-skill/
├── SKILL.md                        # Decision rules + when to read each reference (always loaded)
└── references/
    ├── feature-folders.md          # Folder layout, naming, merging sub-concepts
    ├── queries-actions.md          # Server-only queries + server actions + invalidation
    ├── components.md               # Async server components, skeletons, client boundary, promises
    ├── pages-suspense.md           # Page composition, params.then(), Suspense placement, CLS rules
    ├── cache-components.md         # cacheComponents: true, 'use cache', static shell model
    └── ux-patterns.md              # Toasts, pending state, destructive flows, action-prop, pagination
```

## Installation

Install via [skills.sh](https://skills.sh):

```bash
npx skills install https://github.com/aurorascharff/nextjs-app-architecture-skill
```

## Resources

- [Next.js App Router docs](https://nextjs.org/docs/app)
- [`cacheComponents` config](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)
- [`'use cache'` directive](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [`updateTag`](https://nextjs.org/docs/app/api-reference/functions/update-tag), [`cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cache-tag), [`cacheLife`](https://nextjs.org/docs/app/api-reference/functions/cache-life)
- [Interactive Apps guide (PR)](https://github.com/vercel/next.js/pull/94020) — covers `useTransition`, `useOptimistic`, `useActionState`, `data-pending`, Suspense streaming, and caching across server and client. The guide is not merged yet; the PR is the canonical reference until it lands at `nextjs.org/docs/app/guides/interactive-apps`.
