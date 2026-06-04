# Next.js App Architecture Skill

An agent skill for Next.js 16+ App Router apps. Packages the patterns from [Component Architecture for React Server Components](https://aurorascharff.no/posts/component-architecture-for-react-server-components/) for AI coding agents.

The five principles from the post:

- Pages are synchronous compositors. They don't fetch, they compose.
- Async components fetch their own data. Co-locate the read with the JSX.
- Skeletons live next to their component. Same file, exported alongside it.
- Suspense boundaries go at the page level. The page designs the loading sequence.
- Client boundaries are leaf nodes. Push `'use client'` as deep as it can go.

Everything else in the skill is additional.

## Install

```bash
npx skills install https://github.com/aurorascharff/nextjs-app-architecture-skill
```

## Layout

`SKILL.md` is always loaded. References load on demand.

**Core** — required for any RSC app

- `references/feature-folders.md`
- `references/queries-actions.md`
- `references/components.md`
- `references/pages-suspense.md`

**Instant Apps** — opt-in patterns for making the app feel instant

- `references/cache-components.md`
- `references/ux-patterns.md`

## Companion

- [React View Transitions](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-view-transitions)

## Further reading

- [Server and Client Component Composition in Practice](https://aurorascharff.no/posts/server-client-component-composition-in-practice/)
- [Building Design Components with Action Props using Async React](https://aurorascharff.no/posts/building-design-components-with-action-props-using-async-react/)
- [Error Handling in Next.js with catchError](https://aurorascharff.no/posts/error-handling-in-nextjs-with-catch-error/)
- [Avoiding Server Component Waterfall Fetching with React 19 cache()](https://aurorascharff.no/posts/avoiding-server-component-waterfall-fetching-with-react-19-cache/)
- [next16-social-media](https://github.com/aurorascharff/next16-social-media) — demo app
