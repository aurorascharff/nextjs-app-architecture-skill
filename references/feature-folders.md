# Feature folders

How to organize code under `features/` and `app/`.

## Folder layout

```
features/<domain>/
  <domain>-queries.ts   # Server-only queries
  <domain>-actions.ts   # Server actions
  components/           # Server + client components, each with its skeleton
```

The folder name **is** the domain. The query and action filenames match the folder.

## How many features?

Keep the feature list short. One folder per **domain noun a user would recognize**, not per database table or technical concern.

A new folder is justified when **all three** are true:

1. The concept has its own queries.
2. The concept has its own pages or routes.
3. The concept is referenced from at least two other features.

If you find yourself making a feature folder with one query, one action, and one button, fold it into the parent feature instead.

### Merge aggressively

Concepts that exist only in service of a parent entity belong inside the parent's feature folder:

- A `favorite` or `bookmark` concept that only attaches to one parent entity (events, posts) → inside that parent's folder.
- A `like`, `repost`, `vote`, or `reaction` concept on a piece of content → with that content's feature.
- `auth` / `session` / `current user` → a single `user` folder, not split.

Concrete example: `toggleFavorite` is a mutation about events ("I favorite an event"), not its own domain. It lives in `features/event/event-actions.ts`, not `features/favorite/favorite-actions.ts`.

## File naming

Filenames inside the folder always start with the folder name:

```
features/event/
  event-queries.ts
  event-actions.ts
  components/
    event-grid.tsx
    event-details.tsx
    favorite-button.tsx     ← OK: a component, not a "favorite" feature
```

- `<folder>-queries.ts` — even if the file has only one query.
- `<folder>-actions.ts` — even if a mutation is about a sub-concept.
- Component files use any descriptive name. The component (not the feature) is the unit here.

## What goes in `components/`

Each component file exports the main component **plus its skeleton**:

```tsx
// features/event/components/event-grid.tsx
export async function EventGrid(...) { ... }
export function EventGridSkeleton() { ... }
```

Group related components in one file when they're always used together or one is a natural building block for another. A card and its grid live together. For example, `genre-card.tsx` exports `GenrePill`, `GenreCard`, `GenreGrid`, `GenreGridSkeleton`.

Split into separate files only when:

- A component is consumed by multiple sibling components (one shared use is not enough — wait until three call sites need it).
- A component is `'use client'` and a sibling is a server component (the server/client boundary forbids sharing a file).

See `references/components.md` for inlining rules and the skeleton design checklist.

## What pages do

Pages in `app/` compose feature components with Suspense and transition wrappers. They never:

- Contain domain logic
- Define new components except wrappers like `<NavForward>`
- Fetch data directly
- Inline route-specific components — extract them into the feature folder

See `references/pages-suspense.md` for page composition details.

## Top-level layout

```
app/                  # Pages and layouts
features/             # Domain folders
components/           # UI primitives, theme, and app-shell singletons
types/                # Domain types (optional — co-located is also fine)
lib/                  # Utility functions
```

`components/` holds:

- **`components/ui/`** — primitives. Low-level building blocks and action-prop components.
- **`components/theme/`** — theme provider and toggle, paired.
- **Top-level files** (`site-header.tsx`, `auth-gate.tsx`, `poller.tsx`) — app-shell singletons used once each. No `common/` folder — "common" is not a category. If a component is used everywhere it's a primitive (→ `ui/`); if it's used once it lives at the top level.

Conventions for filenames and casing live in the project's `AGENTS.md`. This skill doesn't impose one.
