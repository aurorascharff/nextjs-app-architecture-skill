# UX patterns

UX patterns that work alongside the architecture in the rest of this skill. These are mostly about pending state, feedback, and edge cases of the client/server boundary.

## Feedback via toasts

Wire your toast library through a single import so the call sites stay clean. Use it for user-facing feedback from mutations.

```tsx
'use client';
import { toast } from '@/lib/toast'; // your toast library
import { createPost } from '@/features/post/post-actions';

async function submitAction(formData: FormData) {
  const result = await createPost(formData);
  if (!result.ok) {
    toast.error(result.error);
  }
}
```

Rules:

- **Toast only on error** when an optimistic UI already shows the result of the action. Double feedback is noise.
- **Toast on success** for non-visible side effects (email sent, link copied, file uploaded).
- **Don't toast for routine navigation.** The page change is the feedback.
- **Don't `toast.promise()` inside a server action.** Toasts are client-side; let the action return a result and toast at the call site.

## Floating UI and view transitions

If the app uses view transitions, portaled/floating elements flicker during route transitions unless excluded. Apply `viewTransitionName: 'none'` to:

- Toast containers
- Dialog backdrops and panels
- Popover panels
- Dropdown menus
- Tooltips

```tsx
<Dialog style={{ viewTransitionName: 'none' }}>...</Dialog>
```

Any UI library that renders portaled content needs this treatment. The fix is the same regardless of which library: pass an inline style with `viewTransitionName: 'none'` on the portal's root element.

When a portal also needs stacking control (z-index) or has translucent layers (a backdrop-blur), give it a *named* transition instead and neutralize it in CSS — this can do things `'none'` can't:

```css
::view-transition-group(modal-backdrop) { animation: none; z-index: 200; }
::view-transition-old(modal-backdrop) { display: none; } /* avoid double-blur */
```

For more on view transitions, see the [React View Transitions skill](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-view-transitions).

## Destructive actions with confirmation

For delete / leave / unsubscribe flows:

```tsx
'use client';
import { useState, useTransition } from 'react';
import { useRouter } from 'next/navigation';
import { toast } from '@/lib/toast'; // your toast library
import { deletePost } from '@/features/post/post-actions';

export function DeletePostDialog({ postId }: { postId: string }) {
  const router = useRouter();
  const [pending, setPending] = useState(false);
  const [, startTransition] = useTransition();

  async function handleDelete() {
    setPending(true);
    const result = await deletePost(postId);
    setPending(false);
    if (!result.ok) {
      toast.error(result.error);
      return;
    }
    toast.success('Deleted');
    startTransition(() => {
      router.push('/');
    });
  }

  // ... render dialog with handleDelete
}
```

### Don't `redirect()` inside a destructive server action

`redirect()` throws. Inside an action called from a dialog, it prevents the client from showing a toast or closing the dialog. Return `{ ok: true }` and navigate client-side with `router.push()`.

### Don't wrap the whole action call in `useTransition` inside a confirm dialog

If view transitions are enabled, wrapping the server action call in `startTransition` will animate the background UI behind the dialog. Use `useState` for the action's pending state and reserve `startTransition` for the post-success navigation only.

## `useOptimistic` for instant feedback

For mutations that are unlikely to fail (favorite, vote, follow), use [`useOptimistic`](https://react.dev/reference/react/useOptimistic) so the UI updates immediately and rolls back if the action throws:

```tsx
'use client';
import { useOptimistic, useTransition } from 'react';

export function FavoriteButton({ slug, favorited }: { slug: string; favorited: boolean }) {
  const [optimisticFavorited, setOptimisticFavorited] = useOptimistic(favorited);
  const [, startTransition] = useTransition();

  function handleClick() {
    startTransition(async () => {
      setOptimisticFavorited(!favorited);
      await toggleFavorite(slug);
    });
  }

  return <button onClick={handleClick}>{optimisticFavorited ? '★' : '☆'}</button>;
}
```

Key rules (the rest is in the React docs):

- Call `setOptimistic` inside a [transition](https://react.dev/reference/react/useTransition), not as a free update.
- Inside `<form action={…}>`, React opens the transition for you — call `setOptimistic` directly in the action body.
- For counter-style updates, pass a reducer as the second argument so the next value is derived from the current one (vote counts, like counts).
- `useOptimistic(false)` is a handy transition-scoped **pending flag** — `const [isPending, setPending] = useOptimistic(false)`, set it `true` in the action body, and it resets automatically when the transition settles (no manual `finally`).

For the deeper picture (coordinating `useTransition`, `useOptimistic`, `useActionState`, Suspense streaming, and caching across server and client), see the [Building interactive apps guide](https://preview.nextjs.org/docs/app/guides/interactive-apps).

## Pending state without `useOptimistic`

For interactions where optimistic UI doesn't fit (filters, sort changes, navigation), use `useTransition` and surface pending state with `data-pending`:

```tsx
'use client';
import { useTransition } from 'react';

export function LabelFilter() {
  const [isPending, startTransition] = useTransition();
  // ...

  function handleChange(value: string) {
    startTransition(() => {
      router.push(`?label=${value}`);
    });
  }

  return (
    <div data-pending={isPending ? '' : undefined}>
      <ToggleGroup ... />
    </div>
  );
}
```

Style ancestors with CSS that responds to a descendant's `data-pending`:

```css
.group:has([data-pending]) {
  opacity: 0.5;
}
```

Or with Tailwind:

```tsx
<div className="group">
  <LabelFilter />
  <div className="group-has-data-pending:opacity-50">{/* content that fades while filter is pending */}</div>
</div>
```

The pending state bubbles up through CSS without prop drilling. When the fading element is a *direct* parent of the `data-pending` node, skip the `group` marker and use the plain variant — `has-data-pending:opacity-50` — since `:has()` matches its own descendants.

## The action-prop pattern

When you have a design component (`<BottomNav>`, `<ToggleGroup>`, `<SubmitButton>`) that wraps user interaction, expose an `action` prop and handle the transition internally:

```tsx
// Inside BottomNav
'use client';
import { useOptimistic, useTransition } from 'react';

export function BottomNav({ tabs, activeIndex, action }: Props) {
  const [optimisticIndex, setOptimistic] = useOptimistic(activeIndex);
  const [isPending, startTransition] = useTransition();

  function handleClick(href: string, index: number) {
    startTransition(async () => {
      setOptimistic(index);
      await action(href);
    });
  }

  return (
    <nav data-pending={isPending ? '' : undefined}>
      {tabs.map((tab, i) => (
        <Link
          key={tab.href}
          href={tab.href}
          className={i === optimisticIndex ? 'active' : ''}
          onClick={e => {
            e.preventDefault();
            handleClick(tab.href, i);
          }}
        >
          {tab.label}
        </Link>
      ))}
    </nav>
  );
}
```

Consumer:

```tsx
<BottomNav tabs={tabs} activeIndex={activeIndex} action={href => router.push(href)} />
```

**Convention**: an action-style prop signals "this triggers a mutation the component coordinates," versus a plain `onChange`/`onClick` callback. Name it `action` or with an `*Action` suffix (`toggleAction`, `confirmAction`) — renaming `onChange` → `action` is a contract change, not just a rename.

This pattern lets the design component handle the async coordination (optimistic updates, pending state, dimming) once, so consumers pass a simple callback. One caveat: **not every action-prop should be wrapped in a transition.** A destructive `confirmAction` behind a dialog is awaited *without* a transition (see below), so the dialog can show its own pending state and block until the mutation resolves. Transition-wrapping is the default for optimistic/navigation actions, not a rule for every prop named `action`.

## URL-based pagination

For "load more" or paginated feeds, drive page number through `searchParams`. Render each page as a separate Suspense boundary so pages stream independently:

```tsx
export async function Feed({ page = 1 }: { page?: number }) {
  return (
    <ul>
      {Array.from({ length: page }).map((_, i) => {
        const p = i + 1;
        const isLast = p === page;
        return p === 1 ? (
          <FeedPage key={p} page={p} isLast={isLast} />
        ) : (
          <Suspense key={p} fallback={<FeedPageSkeleton />}>
            <FeedPage page={p} isLast={isLast} />
          </Suspense>
        );
      })}
    </ul>
  );
}
```

The "Load more" button uses [`<Link scroll={false}>`](https://preview.nextjs.org/docs/app/api-reference/components/link) (or `router.push(url, { scroll: false })` inside `startTransition`). Each page is a separate request; the cache layer handles dedup.

## Global client state

If the app has truly global client state (audio player, shopping cart, theme that needs to react to system changes), wrap a [provider](https://react.dev/reference/react/createContext#provider) with [`useReducer`](https://react.dev/reference/react/useReducer) at the root:

```tsx
// providers.tsx
'use client';
import { createContext, useContext, useReducer } from 'react';

const Ctx = createContext(null);

export function AppProviders({ children }: { children: React.ReactNode }) {
  const [state, dispatch] = useReducer(reducer, initial);
  return <Ctx.Provider value={{ state, dispatch }}>{children}</Ctx.Provider>;
}

export function useAppState() {
  return useContext(Ctx);
}
```

The provider is `'use client'`, but `children` stays server-rendered. Place the provider in the root layout. Only leaf components call `useContext`.

**Don't** push server data into global client state. Keep server data in queries; client state is for ephemeral UI (open menus, audio playback position, optimistic drafts).

## Form pending state from `useFormStatus`

[`useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus) reads the pending state of the **nearest parent form**. It must be called from a component rendered inside `<form>`, not from the form component itself.

```tsx
'use client';
import { useFormStatus } from 'react-dom';

function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus();
  return (
    <button type="submit" disabled={pending}>
      {pending ? 'Submitting…' : children}
    </button>
  );
}

// usage
<form action={submitAction}>
  <Input name="content" />
  <SubmitButton>Post</SubmitButton>
</form>;
```

This is the cleanest way to handle "disable + spinner" on submit without lifting state.

## Inline field errors from `useActionState`

Toasts suit transient, whole-action feedback. For form **validation** errors that belong next to the field, use [`useActionState`](https://react.dev/reference/react/useActionState): the action returns `{ error }`, and you render it inline with `aria-invalid` + `role="alert"` for accessibility.

```tsx
'use client';
import { useActionState } from 'react';

export function SignInForm() {
  const [state, formAction] = useActionState(signIn, null);
  return (
    <form action={formAction}>
      <input name="email" aria-invalid={!!state?.error} aria-describedby="email-error" />
      {state?.error && <p id="email-error" role="alert">{state.error}</p>}
      <SubmitButton>Sign in</SubmitButton>
    </form>
  );
}
```

Use inline errors for "fix this field" feedback; use toasts for "that didn't work" feedback on one-shot mutations.

## Error states in components

When a query throws, `error.tsx` at the route segment catches it. Wrap sub-sections that may fail independently in an error boundary built on `catchError` from `next/error`. For `notFound()` use `not-found.tsx`. See `references/pages-suspense.md` for the boundary primitive and placement.
