# Fullscreen Studio Layout + Flow Builder Gaps

**Date:** 2026-06-03
**Status:** Approved
**Scope:** Frontend only (no backend changes)

## Context

The studio roadmap presentation (slide 6) shows a fullscreen app with no global header/footer — just the Studio title, mode toggle, and flow builder filling the viewport. The current implementation renders Studio inside `RootElement`, which unconditionally shows Header, Footer, and PlayerTray. This eats vertical space and breaks the immersive creation experience.

Additionally, two flow builder features remain stubbed:
- "Save to Playlist" still shows a "Coming soon" toast
- "Preview All" was fixed in PR #632 (now on stage) — no work needed

The "Results strip" from the design (status cards below settings) is deferred to a future phase — the transition gap indicators are sufficient for now.

## 1. Fullscreen Studio Layout

### Approach

Move the `/studio` route out of `RootElement`'s children into a new top-level route entry with its own `StudioLayout` wrapper. All other routes stay untouched.

```
router = [
  { path: "/",       element: <RootElement/>,   children: [...everything except studio...] },
  { path: "/studio", element: <StudioLayout/> }
]
```

### StudioLayout

A new component at `src/components/pages/studio/studio.layout.tsx`:

- Wraps `StudioPage` with the same `withProviders(…Providers)` pattern as `RootElement`
- Full viewport: `height: 100vh; overflow: hidden` (studio manages its own scroll)
- Fires a single `ReactGA.send({ hitType: "pageview" })` on mount (no location listener needed — single route)
- Does NOT render: Header, Footer, PlayerTray, ScrollToHashElement

### Studio Header Changes

Replace the current `StudioHeader` in `studio.page.tsx`:

- **Left:** Back arrow icon button — uses guarded navigation: if entered from within the app, `navigate(-1)`; otherwise falls back to `navigate(ROUTES.REMOTE_CONTROL)` (the app's default home route). Use `window.history.length > 1` as the heuristic.
- **Center:** Flow / Batch (Advanced) mode toggle (unchanged)
- **Right:** New Session button (batch mode only, same as current). Flow mode has no reset button in the header — `FlowReset` already exists as a standalone component inside the flow builder.

### What We Lose (Intentionally)

| Feature | Decision |
|---------|----------|
| Header (nav bar) | Not needed — back arrow provides escape hatch |
| Footer | Not needed in a creation tool |
| PlayerTray | Irrelevant during creation |
| ScrollToHashElement | Studio doesn't use hash navigation |

### Protected Route

The studio route currently sits inside a `ProtectedRoute` with `CREATOR_GROUP` + `ADMIN_GROUP` roles. The new top-level route wraps `StudioLayout` inside the same `ProtectedRoute`:

```tsx
{
  path: ROUTES.STUDIO,
  element: (
    <ProtectedRoute allowedRoles={[ROLES.CREATOR_GROUP, ROLES.ADMIN_GROUP]}>
      <StudioLayoutWithProviders />
    </ProtectedRoute>
  ),
}
```

`ProtectedRoute` handles its own redirect (to login), so no additional auth logic is needed in `StudioLayout`.

### Error Boundary

The new top-level route must include `errorElement: <NotFoundPageWithProviders />` to match the existing router pattern. Without this, a rendering error inside Studio shows a blank page.

### Files Changed

| File | Change |
|------|--------|
| `src/routes/router.tsx` | Move studio route to top-level, add `StudioLayout` entry |
| `src/components/pages/studio/studio.layout.tsx` | **New** — chromeless layout wrapper |
| `src/components/pages/studio/studio.page.tsx` | Update header: add back arrow, reposition controls |
| `src/components/pages/studio/studio.page.styled.tsx` | Update `StudioContainer` for full viewport, update `StudioHeader` for back arrow |

## 2. Save to Playlist

### Trigger

The "Save to Playlist" button in `FlowActionBar`. Enabled when at least one transition has `status === "processed"` and a `dreamUuid`.

### Modal UI

New component: `SaveToPlaylistModal`

- **Title:** "Save to Playlist"
- **Mode toggle** (top): "New Playlist" (default) / "Existing Playlist"
- **New Playlist mode:** Text input pre-filled with `"Studio Flow — {YYYY-MM-DD}"`
- **Existing Playlist mode:** Scrollable list from `useMyPlaylists`, click to select, highlight selected
- **Summary:** "Adding {N} transitions"
- **Buttons:** "Save" (primary) + "Cancel"
- **Loading state:** Spinner on Save button during API calls, modal stays open
- **Success:** Auto-close, toast with playlist name + link to `/playlist/{uuid}`
- **Error:** Toast with error message, modal stays open for retry

### Behavior

1. If "New Playlist" — create via `useCreatePlaylist().mutateAsync({ name: playlistName })`
2. For each completed transition (in order), add via `useAddPlaylistItem().mutateAsync({ playlistUUID, values: { type: "dream", uuid: transition.dreamUuid } })`
3. Sequential adds to preserve transition order. Show per-item progress in the modal ("Adding 3 of 8...")
4. On success: close modal, show toast with link

**Use mutation hooks exclusively** — not raw axios. This keeps the pattern consistent and gets React Query cache invalidation for free.

### Reuse

- Playlist API hooks: `useCreatePlaylist`, `useAddPlaylistItem`, `useMyPlaylists` (from `src/api/playlist/`) — all exist
- Modal styling: follow the pattern from `AddKeyframesFromPlaylistModal` (already has playlist list + selection UI)

### Files Changed

| File | Change |
|------|--------|
| `src/components/pages/studio/components/save-to-playlist-modal.tsx` | **New** — the modal component |
| `src/components/pages/studio/components/save-to-playlist-modal.styled.tsx` | **New** — modal styles |
| `src/components/pages/studio/components/flow-action-bar.tsx` | Replace toast with modal open state, render modal inline (self-contained — no state threading to flow-builder) |

## Summary of All Changes

| # | Change | New/Edit |
|---|--------|----------|
| 1 | `studio.layout.tsx` — chromeless layout | New |
| 2 | `router.tsx` — move studio to top-level route | Edit |
| 3 | `studio.page.tsx` — back arrow header | Edit |
| 4 | `studio.page.styled.tsx` — full viewport + header update | Edit |
| 5 | `save-to-playlist-modal.tsx` — modal component | New |
| 6 | `save-to-playlist-modal.styled.tsx` — modal styles | New |
| 7 | `flow-action-bar.tsx` — wire up modal (self-contained) | Edit |

## Out of Scope

- Results strip (transition status cards below settings) — deferred
- Preview All — already fixed on stage (PR #632)
- Server-side video concatenation
- Client-side ffmpeg concat
