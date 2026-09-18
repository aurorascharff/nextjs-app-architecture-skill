# UX patterns

Interaction decisions on top of the architecture: which feedback mechanism to reach for, and the boundary edge-cases that trip agents up. Hook mechanics live in the React / Next docs — linked, not restated. The deeper end-to-end picture is the [Building interactive apps guide](https://preview.nextjs.org/docs/app/guides/interactive-apps).

## Choose the feedback mechanism

| Situation | Reach for | Key rule |
| --------- | --------- | -------- |
| Mutation unlikely to fail (favorite, vote, follow) | [`useOptimistic`](https://react.dev/reference/react/useOptimistic) | Update immediately, roll back on throw. Set it inside a transition; inside `<form action>` React opens the transition for you. Use a reducer for counters. |
| No optimistic fit (filters, sort, navigation) | [`useTransition`](https://react.dev/reference/react/useTransition) + `data-pending` | Put `data-pending` on the pending node; let ancestors react with CSS (`has-data-pending:` for a direct parent, `group-has-data-pending:` further up) so it bubbles without prop drilling. |
| Form field validation ("fix this field") | [`useActionState`](https://react.dev/reference/react/useActionState) | Action returns `{ error }`; render inline with `aria-invalid` + `role="alert"`. |
| Submit disable + spinner | [`useFormStatus`](https://react.dev/reference/react-dom/hooks/useFormStatus) | Call it from a child of `<form>`, not the form component itself. |
| Selections that live in the URL (filters, wizard steps) | `useOptimistic(draft)` + [`router.replace`](https://preview.nextjs.org/docs/app/api-reference/functions/use-router) inside `startTransition` | The URL is the source of truth; the optimistic draft keeps the control responsive until the server confirms the new search params. Submit the final draft to the action through hidden inputs and re-validate it there. |
| One-shot result with no visible change | toast | See below. |

### Name the signal when a subtree has more than one pending source

`data-pending` bubbles through CSS, so every descendant that sets it participates. `useLinkStatus` sets it on any pending link, which means `group-has-data-pending:` on a page wrapper also fires for ordinary navigation and dims content that is not being refetched. When a subtree can be pending for more than one reason, give the reason you are reacting to its own attribute (`data-filtering`, `data-saving`) and key the ancestor off that.

### Two things to get right with `useOptimistic`

- The value reverts as soon as its transition settles, so the work it predicts has to run **inside the same** `startTransition`. Setting the optimistic value in one transition and navigating in another snaps it back before the URL changes.
- Derive the controls from the optimistic value and the surrounding chrome from the committed one. A slider thumb should follow the drag immediately, but a "Clear all filters" button keyed off that same optimistic value appears while the filter is still in flight.

`useOptimistic(false)` also works as a transition-scoped **pending flag** that resets automatically when the transition settles — handy when you don't need the `data-pending` bubbling.

## Optimistic mutations for interactive apps

For create/update/delete flows where the changed item is visible, prefer one feature-owned optimistic reducer over hand-rolled pending state spread across the board, modal, and list.

```tsx
type OptimisticListAction<TItem> =
  | { type: 'create'; item: TItem }
  | { type: 'update'; item: TItem }
  | { type: 'delete'; id: string }
  | { type: 'rollback'; id: string };

const [items, dispatchOptimisticItem] = useOptimistic(
  initialItems,
  optimisticListReducer,
);
```

Use names that describe the app's domain (`dispatchOptimisticPost`, `messageReducer`, `groupReducer`, `eventReducer`) rather than implementation mechanics like `applyOptimisticAction`. The reducer owns the optimistic list shape; components dispatch intent.

For modal creates, apply the optimistic item and close the modal immediately when the local form is valid. If the action fails, roll back and show an error toast. Keeping the modal open with a slow primary button makes the optimistic result feel broken.

For deletes, remove optimistically, then roll back on failure. Do not wait for the server before the item disappears when the user's intent is clear.

## Pending states

Every click that starts a write or a navigation shows its pending state on the thing that was clicked, without changing the layout around it.

- **Item pickers** (a seat, a slot, a time). While the chosen item's write is pending, disable the whole group, fade the other items to about 50%, and keep the chosen one solid with `aria-busy`. No spinner inside the item and no cursor change. A status line under the group says what is happening, then what was chosen.
- **Nudges are inline.** A Continue pressed before a required pick turns that status line into the instruction and resets when the pick is made. The button stays enabled; a disabled button hides the reason.
- **Expected failures are inline.** When the write fails for a reason the user can act on (the item was just taken), show it on the same status line and clear the optimistic value. Toasts are for failures with no place in the layout.
- **`useActionState` outside a form.** Its dispatch is a plain call, so wrap it: `startTransition(() => dispatch(value))`. Otherwise `isPending` and the optimistic value appear only when the action settles, which reads as a frozen click.
- **Optimistic values inside an action.** A `setState` inside an async action is held until the action finishes. Use `useOptimistic` for anything that must show while the action runs.
- **Indicators keep their size.** A spinner that appears inside a button must not widen it; see CLS prevention in `references/pages-suspense.md`.

## Forms

Validate locally before submitting when the missing field is already on the client, and show field errors inline without changing the card's geometry. Reserve `useActionState` for errors that need the action result. When a completed action changes the user's task (a reservation, a checkout), replace the form with the confirmation instead of toasting.

### Search and filter forms

A plain `<form method="get">` submits as a document request, so the page reloads and every boundary shows its skeleton again. Make these forms client navigations:

- **Defaults come from the URL.** Read `useSearchParams()` in the client form and key the `<form>` on `params.toString()`, so the fields follow the URL and remount with fresh `defaultValue`s. The form then lives in the route shell and renders at once on navigation. A form that takes its defaults from a `searchParams.then()` sits inside the URL boundary and re-skeletons on every search.
- **Prefetch while the user is still choosing.** On `onChange`, build the destination URL and call `router.prefetch(href)`.
- **Submit in a transition.** `onSubmit` prevents the default and runs `startTransition(() => router.push(href))`. The submit button's `useFormStatus` spinner already reflects that transition; don't add a second one.
- **Fade the current results.** Render the results as children of the form's client shell under `data-pending={isPending ? '' : undefined}` with `transition-opacity data-pending:opacity-60`. The old results stay on screen, dimmed, until the new ones commit.

`next/form` gives the client navigation and the button spinner on its own; use the client shell when the results should dim.

## Toasts

- **Toast only on error** when an optimistic UI already shows the result. A success toast next to an optimistic checkmark is double feedback.
- **Toast on success** only for side effects with no visible result (email sent, link copied).
- **Don't toast for navigation.** The page change is the feedback.
- **Don't toast inside a server action.** Return a result and toast at the call site.
- **Don't toast a nudge or an expected in-page failure.** Use the component's status line (see Pending states).

## Destructive actions (delete / leave / unsubscribe)

Gate behind a confirmation dialog whose confirm button owns the pending state, toast `error` from the result, and navigate away only after `{ ok: true }`. Mind two edge cases:

- **Don't `redirect()` inside an action called from a click or dialog.** It throws, which stops the client from toasting or closing the dialog, and a `try/catch` around the call mistakes it for a failure. Return `{ ok: true }` and navigate with `router.push()`. (Form actions are different — see `references/queries-actions.md`.)
- **Don't wrap the whole action call in `useTransition`** inside the dialog — with view transitions on, that animates the background UI behind the dialog. Track pending with `useState` / `useOptimistic(false)` and reserve `startTransition` for the post-success navigation only.

## Blocking a write in flight

A full-screen overlay on link navigation is never worth it: the App Shell commits the destination at once, with a fallback at worst, so the overlay flashes for a frame. Reserve a blocker for the one write that must not be double-submitted or abandoned (confirm, pay). While it is pending, replace the action buttons with the progress indicator so nothing is left to click.

- **Render it from a layout that survives the redirect.** Anything inside the finishing page's subtree, portaled or not, is deleted with that page, and React does not animate a nested boundary's exit inside a deleted subtree.
- **Show it with `useOptimistic` from inside the action.** The optimistic value appears at once and reverts in the same transition that commits the redirect, which is what lets the overlay crossfade out and a shared element (`<ViewTransition name share>`) morph into the result page.
- **Keep transitions short and snapshot-safe.** Use the shared timing tokens and avoid `backdrop-filter` on the captured element, or the result page's Suspense reveal interrupts the running transition and the settle flickers.
- **Don't add a `beforeunload` guard.** The server completes the write whether the tab stays open or not, and a deploy or code reload makes the framework hard-navigate mid-action, which turns the guard into a spurious "Leave site?" dialog.

## Leaving a finished flow

Redirect from the completing action with `RedirectType.replace`, so the final step is replaced by the result page instead of sitting one swipe behind it. Don't add an "already done, redirect to the result" check to the steps: it breaks doing the same flow again on purpose, and a `redirect()` in a destination is never followed by a prefetch.

## View transitions: portaled / floating UI

Portaled elements (toasts, dialogs, popovers, dropdowns, tooltips) flicker during route transitions unless excluded. Apply `viewTransitionName: 'none'` to the portal root. When the portal also needs stacking control (z-index) or has translucent layers (backdrop-blur), give it a *named* transition and neutralize it in CSS instead — `::view-transition-group(name) { animation: none; z-index: … }` can do things `'none'` can't. See the [React View Transitions skill](https://github.com/vercel-labs/agent-skills/tree/main/skills/react-view-transitions).

## The action-prop pattern

A reusable design component (`<ToggleGroup>`, `<BottomNav>`, `<SubmitButton>`) can take an action-style prop and own the async coordination (optimistic update, pending, dimming) so consumers pass a plain callback. Convention: an `action` / `*Action` prop signals "triggers a mutation this component coordinates," versus a plain `onChange` / `onClick` — renaming between them is a contract change. Not every such prop is transition-wrapped: a destructive `confirmAction` is awaited *without* a transition (see above). Transition-wrapping is the default for optimistic/navigation actions, not a rule tied to the name.

## URL-based pagination

Drive the page number through `searchParams`, render each page as its own `<Suspense>` boundary so pages stream independently, and add "load more" with `<Link scroll={false}>`. See [linking and navigating](https://preview.nextjs.org/docs/app/getting-started/linking-and-navigating).

```tsx
import { Suspense } from 'react';

export function Feed({ page = 1 }: { page?: number }) {
  return (
    <ul>
      {Array.from({ length: page }).map((_, i) => {
        const p = i + 1;
        return p === 1 ? (
          <FeedPage key={p} page={p} />
        ) : (
          <Suspense key={p} fallback={<FeedPageSkeleton />}>
            <FeedPage page={p} />
          </Suspense>
        );
      })}
    </ul>
  );
}
```

The first page can render in the parent boundary; later pages get their own fallbacks so "load more" streams only the newly requested page. If the URL update should preserve scroll, use [`<Link scroll={false}>`](https://preview.nextjs.org/docs/app/api-reference/components/link).

## Global client state

For truly global client state (audio player, cart, system-reactive theme), wrap a [context provider](https://react.dev/reference/react/createContext) at the root; the provider is `'use client'` but `children` stays server-rendered, and only leaf components read the context. **Don't push server data into it** — server data stays in queries; client state is for ephemeral UI (open menus, playback position, optimistic drafts). See the live-data decision in `references/components.md`.
