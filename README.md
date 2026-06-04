# Next.js App Architecture Skill

An agent skill that teaches AI coding agents the **component architecture for React Server Components** patterns from [this blog post](https://aurorascharff.no/posts/component-architecture-for-react-server-components/).

## The core

Five principles, straight from the post. Everything else is additional.

- **Pages are synchronous compositors.** They don't fetch, they compose.
- **Async components fetch their own data.** Co-locate the read with the JSX.
- **Skeletons live next to their component.** Same file, exported alongside it.
- **Suspense boundaries go at the page level.** The page designs the loading sequence.
- **Client boundaries are leaf nodes.** Push `'use client'` as deep as it can go.

That's the foundation. Without these, the rest of the skill doesn't matter. With these, the rest is optional polish — Cache Components, optimistic UI, toasts, action-prop components, etc.

For the full reasoning (why loaders couple components to routes, how RSCs decouple them, how `Suspense` lets pages design the loading sequence), read the [blog post](https://aurorascharff.no/posts/component-architecture-for-react-server-components/).

## Installation

Install via [skills.sh](https://skills.sh):

```bash
npx skills install https://github.com/aurorascharff/nextjs-app-architecture-skill
```

## Skill structure

The skill is organized into two zones so an agent only loads what the task needs.

### Core (the five principles, fleshed out)

Required reading for any RSC Next.js app.

- **`SKILL.md`** — Decision rules + reference index (always loaded)
- **`references/feature-folders.md`** — Folder layout, naming, merging sub-concepts
- **`references/queries-actions.md`** — Server-only queries with `cache()`, server actions, validation, `refresh()`
- **`references/components.md`** — Async server components, skeletons, client boundary, `use()` + promise props
- **`references/pages-suspense.md`** — Page composition, `params.then()`, Suspense placement, CLS prevention

### Instant Apps (opt-in)

Additional patterns for making the app feel instant. Reach for these only when the task calls for them.

- **`references/cache-components.md`** — `cacheComponents: true`, `'use cache'` / `private` / `remote`, `cacheTag` / `cacheLife`, `updateTag`, `connection()`
- **`references/ux-patterns.md`** — `useOptimistic`, toasts, `data-pending`, destructive flows, action-prop pattern, URL pagination, `useFormStatus`

## Companion skills

- [React View Transitions](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-view-transitions) — `<ViewTransition>`, `addTransitionType`, directional navigation, shared elements, Suspense reveals, CSS recipes. Pairs naturally with this skill's page-level boundary placement.

## Further reading

- [Component Architecture for React Server Components](https://aurorascharff.no/posts/component-architecture-for-react-server-components/) — the blog post this skill is based on
- [Server and Client Component Composition in Practice](https://aurorascharff.no/posts/server-client-component-composition-in-practice/)
- [Building Design Components with Action Props using Async React](https://aurorascharff.no/posts/building-design-components-with-action-props-using-async-react/)
- [Error Handling in Next.js with catchError](https://aurorascharff.no/posts/error-handling-in-nextjs-with-catch-error/)
- [Avoiding Server Component Waterfall Fetching with React 19 cache()](https://aurorascharff.no/posts/avoiding-server-component-waterfall-fetching-with-react-19-cache/)
- [next16-social-media](https://github.com/aurorascharff/next16-social-media) — the demo app from the blog post

## Next.js / React docs

- [Next.js App Router](https://nextjs.org/docs/app)
- [`cacheComponents`](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents)
- [`'use cache'`](https://nextjs.org/docs/app/api-reference/directives/use-cache)
- [`updateTag`](https://nextjs.org/docs/app/api-reference/functions/update-tag), [`cacheTag`](https://nextjs.org/docs/app/api-reference/functions/cache-tag), [`cacheLife`](https://nextjs.org/docs/app/api-reference/functions/cache-life)
- [Interactive Apps guide (PR)](https://github.com/vercel/next.js/pull/94020) — `useTransition`, `useOptimistic`, `useActionState`, `data-pending`, Suspense streaming, and caching across server and client. Not merged yet; the PR is the canonical reference until it lands.
