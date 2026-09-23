---
paths:
  - "frontend/**"
---

## Anti-Rationalization Tables

These tables pre-rebut excuses for skipping steps that have caused real bugs in this project. If you catch yourself thinking one of these, stop.

### Before writing store mutations

| Rationalization | Reality |
|----------------|---------|
| "I'll wire up the derived state recomputation later." | Every mutation that changes source data (keyframes) must re-derive dependent state (transitions) in the same `set()` call. "Later" means shipping a store where adding a keyframe creates zero transitions. |
| "The existing ID type should work for the downstream consumer." | Trace the actual data path: frontend field -> API payload -> worker handler -> model input. Keyframe UUIDs are not Dream UUIDs. The worker can't resolve entity types it wasn't built to handle. |
| "I don't need an `updateKeyframe` action, add/remove is enough." | Any upload flow needs to patch records in place (progress updates, swap placeholder URL for real one, set final UUID). If you can't update, you can't upload. |

### Before persisting state to localStorage

| Rationalization | Reality |
|----------------|---------|
| "I'll persist everything and filter on load." | A half-uploaded keyframe with a dead objectURL is a broken ghost card on reload. `partialize` must filter to settled records only. If it doesn't have a backend UUID, don't save it. |
| "Persisting UI state (selected index, expanded panels) improves UX." | Persisted indices go stale when the underlying array changes. A `selectedTransitionIndex` of 3 after keyframes were deleted points at nothing. Persist data, not UI state. |
| "I only need to reconcile generation status on rehydration." | If the store has parallel state machines (generation + uprez), reconciliation must cover all of them. Stale uprez jobs stuck in "processing" are just as broken as stale generation jobs. |

### Before implementing API-calling hooks

| Rationalization | Reality |
|----------------|---------|
| "A sequential for-loop is fine for Generate All." | With 5+ transitions, each API roundtrip blocks the next. Users see nothing happening for 10+ seconds. Use a worker-pool pattern with a concurrency cap (4). |
| "I'll subscribe to store values and put them in useCallback deps." | Every settings keystroke re-creates the callback, which re-renders every consumer. Read volatile store data via `getState()` inside the callback body. Only subscribe to stable action refs. |
| "console.error is fine for production error handling." | Use Bugsnag. Console errors are invisible to users and to us. Bugsnag alerts surface issues before users report them. |

### Before choosing an upload strategy

| Rationalization | Reality |
|----------------|---------|
| "The Keyframe CRUD API is the natural place to upload keyframe images." | The generation pipeline consumes Dream UUIDs, not Keyframe UUIDs. Upload must produce the entity type that the downstream consumer (worker) can resolve. Use `useUploadImageDream` to create image-type Dreams. |
| "I can pass the image URL and the worker will figure it out." | Check which field the worker reads per model. Wan uses `image` (accepts URLs). LTX uses `source_dream_uuid` (expects a UUID, not a URL). Each model has different field semantics. |

### Before introducing design tokens

| Rationalization | Reality |
|----------------|---------|
| "DM Sans / Instrument Serif matches the design deck." | The deck is aspirational. The shipping app uses Comfortaa. New components must match the running product, not the mockup. Check the actual font-family in the app's global styles. |
| "I only need success/processing/queued colors." | You also need error, errorDim, successDim, processingDim. Every status has a foreground and a dim/background variant. Shipping an incomplete palette means components hardcode colors. |
