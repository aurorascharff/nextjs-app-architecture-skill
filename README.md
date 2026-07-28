# Skills

A collection of [agent skills](https://skills.sh) I maintain. Each lives under [`skills/`](skills/) with its own `SKILL.md` and on-demand references.

| Skill | What it's for |
| ----- | ------------- |
| [`nextjs-app-architecture`](skills/nextjs-app-architecture) | Build or audit Next.js 16+ App Router apps with a next-beats-style RSC architecture, Suspense streaming, and Cache Components practice. |
| [`friction-log`](skills/friction-log) | Records agentic developer-experience friction during a build, and produces a structured, severity-coded markdown log. |

Install any skill with:

```bash
npx skills add aurorascharff/skills/skills/<skill-name>
```

---

## `nextjs-app-architecture`

Build or audit Next.js 16+ App Router apps. Packages the patterns from [Component Architecture for React Server Components](https://aurorascharff.no/posts/component-architecture-for-react-server-components/) and the [next-beats](https://github.com/vercel-labs/next-beats) reference app into a form AI coding agents can load and apply.

```bash
npx skills add aurorascharff/skills/skills/nextjs-app-architecture
```

### The six principles

- **Pages are synchronous compositors.** They don't fetch, they compose.
- **Async components fetch their own data.** Co-locate the read with the JSX.
- **Route props stop at the page.** Components receive IDs, slugs, parsed filters, or records — not raw `params` / `searchParams`.
- **Skeletons live next to their component.** Same file, exported alongside it.
- **Suspense boundaries go at the page level.** The page designs the loading sequence.
- **Client boundaries are leaf nodes.** Push `'use client'` as deep as it can go.

### Prerequisite

Before using the skill on a project, follow the [Next.js AI Coding Agents guide](https://preview.nextjs.org/docs/app/guides/ai-agents) so the agent reads version-matched Next.js docs from `AGENTS.md` / bundled docs instead of stale training data.

### What it covers

- **Feature folders** — when to create one, when to merge a sub-concept into a parent, file naming.
- **Queries** — `import 'server-only'`, plain async reads by default, selective React `cache()` only for proven same-request dedup, and `'use cache'` + `cacheTag` + `cacheLife` for Cache Components.
- **Actions** — `'use server'`, input validation, tag invalidation under Cache Components, calling from client components.
- **Components** — async server components that receive IDs/parsed values, sibling skeletons, single-use helpers, the client boundary, the `use()` + promise-prop pattern, live data via polling.
- **Pages** — sync page composition, `params.then()` for static-shell preservation, Suspense boundary placement, CLS prevention, error boundaries, audit smells.
- **Cache Components** — when to opt in, the static-shell model, `'use cache'` variants, build constraints.
- **UX patterns** — `useOptimistic`, toasts, pending state, destructive-action flows, the action-prop pattern, URL pagination.

### References (loaded on demand)

The `SKILL.md` overview is always loaded; references split into two zones so the agent pulls only what the task needs.

**Core** (any RSC app):

- [`references/feature-folders.md`](skills/nextjs-app-architecture/references/feature-folders.md) — folder layout, naming, merging sub-concepts, action/query file naming.
- [`references/queries-actions.md`](skills/nextjs-app-architecture/references/queries-actions.md) — server-only queries, selective same-request dedup, server actions, validation, and cache/tag invalidation.
- [`references/components.md`](skills/nextjs-app-architecture/references/components.md) — async server components, skeletons without alias wrappers, client boundary, promise + `use()`, single-use helpers, polling.
- [`references/pages-suspense.md`](skills/nextjs-app-architecture/references/pages-suspense.md) — page composition, `PageProps` / `LayoutProps`, `params.then()`, Suspense placement, CLS prevention, error boundaries.
- [`references/example.md`](skills/nextjs-app-architecture/references/example.md) — next-beats invariant map and supporting patterns.

**Instant Apps** (opt-in, load only when optimizing for instant-feeling apps):

- [`references/cache-components.md`](skills/nextjs-app-architecture/references/cache-components.md) — `cacheComponents: true`, the static shell, which reads to cache, `'use cache'` variants, `cacheTag` / `cacheLife`, `updateTag` / `revalidateTag`, and `io()` vs `connection()`.
- [`references/ux-patterns.md`](skills/nextjs-app-architecture/references/ux-patterns.md) — `useOptimistic`, toasts, pending state via `data-pending`, destructive flows, the action-prop pattern, URL pagination, `useFormStatus`.

### Background reading

The patterns in this skill are written up in more depth here:

- [Component Architecture for React Server Components](https://aurorascharff.no/posts/component-architecture-for-react-server-components/) — the source post this skill packages.
- [Server and Client Component Composition in Practice](https://aurorascharff.no/posts/server-client-component-composition-in-practice/)
- [Building Design Components with Action Props using Async React](https://aurorascharff.no/posts/building-design-components-with-action-props-using-async-react/)
- [Error Handling in Next.js with catchError](https://aurorascharff.no/posts/error-handling-in-nextjs-with-catch-error/)
- [Avoiding Server Component Waterfall Fetching with React 19 cache()](https://aurorascharff.no/posts/avoiding-server-component-waterfall-fetching-with-react-19-cache/)
- [next16-social-media](https://github.com/aurorascharff/next16-social-media) — demo app applying these patterns.

**Companion skill:** [React View Transitions](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-view-transitions).

---

## `friction-log`

Documents agentic developer-experience friction encountered during a development flow. Friction that blocks an agent also surfaces DX issues for developers — so the skill treats logging as equally important as finishing the task. Use it when asked to *log friction*, *document a pain point*, *write a friction log*, or when an agent gets stuck on unclear docs, framework behavior, tooling, sandboxing, or stale training-data assumptions.

```bash
npx skills add aurorascharff/skills/skills/friction-log
```

> This collection includes the active `friction-log` skill only. The old passive `friction-report` skill (end-of-session scan) is intentionally not included — it wasn't reliable enough to keep in the collection.

### What it does

1. Attempts the user-specified task.
2. Logs friction **as it's encountered**, in real time — what was expected, what actually happened, what was tried, and how it resolved (or that it didn't).
3. Cites a source tag on every entry, so a reader can judge how much to trust each line.
4. Folds in mid-run user replies (chat-thread messages, queued instructions) at the point in time they arrived.
5. Produces a structured markdown log and prints it to stdout when done.

### Output format

- **Header** — date, model, harness, stack + version, cumulative build time, task, input/output links.
- **Prompt** — the user's initial request, verbatim, plus any clarifications.
- **Tool Timeline** — chronological one-liners of every tool call. Always written, so the log stands alone in a viewer.
- **Summary** — 2–4 sentences: what went well, the biggest pain point, the blast radius.
- **Action Items** — the most important part. Every 🟡/🔴 produces a concrete item with a `Context:` line, split into:
  - **Docs** 🔧 — fixable with better documentation or examples.
  - **Framework** 🔧 — needs a code change (error messages, scaffold defaults, tooling).
  - **DX / Research** 🔍 — open questions or investigations.
- **Log** — chronological, severity-coded narrative.
- **Skill Feedback** 🔁 — *optional, added later* if a review finds the skill itself caused wrong behavior. Never an empty placeholder.

### Severity levels

| Marker | Meaning |
| ------ | ------- |
| 🟢 | Smooth — worked as expected (useful as a regression baseline). |
| 🟡 | Minor friction — extra steps, guesswork, or doc-hunting. |
| 🔴 | Major friction — blocked, broken, missing, or deeply confusing. Highest-value entries; each should have an action item. |

### Source tags

Every log entry ends with a tag showing how the information was obtained — `[docs]`, `[error output]`, `[sandbox]` and `[agents.md]` are high trust; `[web search]` is medium; `[training data]` is lowest and should be treated as an unconfirmed guess.

### Viewing logs

Paste a finished log into **[agent-friction-skill.vercel.app](https://agent-friction-skill.vercel.app/)** for a collapsible, severity-coded rendering. Nothing leaves your browser — the encoded log travels as a URL fragment for sharing.

### References

- [`references/agent-behavior.md`](skills/friction-log/references/agent-behavior.md) — when to log, what counts as friction, the full source-tag list.
- [`references/reading-the-log.md`](skills/friction-log/references/reading-the-log.md) — how a human reads the output (read order, severity legend, trust levels, priority).
- [`references/template.md`](skills/friction-log/references/template.md) — the exact output template.
- [`references/example.md`](skills/friction-log/references/example.md) — a real friction log from an actual run.

---

## License

MIT
