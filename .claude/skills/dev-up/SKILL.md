---
name: dev-up
description: Start the backend and frontend dev servers, first fast-forwarding each to origin only when it is sitting on stage/main and clean. Use for "start the servers", "restart the servers", "bring up dev", or after pulling work that touches both repos.
---

# Bring up backend + frontend

Both repos point at **staging** infrastructure (AWS RDS, Upstash Redis) — see
AGENTS.md. Nothing here touches a database.

## 1. Decide per repo whether to update

Run this for `backend` and `frontend` independently. They are separate
checkouts and are often on different branches.

```bash
for r in backend frontend; do
  d=/Users/spot/e-dream-ai/$r
  b=$(git -C "$d" rev-parse --abbrev-ref HEAD)
  dirty=$(git -C "$d" status --porcelain --untracked-files=no)
  printf '%-9s %-28s %s\n' "$r" "$b" "${dirty:+DIRTY}"
done
```

Then, for each:

| State | Action |
| --- | --- |
| on `stage` or `main`, clean | `git -C "$d" pull --ff-only` |
| on `stage` or `main`, dirty | **do not pull.** Report the modified files and carry on with what is there |
| on any other branch | **leave it alone.** Report the branch name and carry on |
| `pull --ff-only` fails | the branch has diverged — report it, do not merge or rebase |

Only ever fast-forward. A merge or rebase here is a decision the user makes,
not something a startup routine does on their behalf.

## 2. Install only if the lockfile moved

A pull that changed `pnpm-lock.yaml` needs deps before the server will boot:

```bash
git -C "$d" diff --name-only HEAD@{1} HEAD 2>/dev/null | grep -q pnpm-lock.yaml && (cd "$d" && pnpm install)
```

`Cannot find module 'bullmq'` (or similar) on startup means the same thing
after the fact — run `pnpm install` and accept the prompt to wipe
`node_modules` if offered.

## 3. Do not run migrations

`backend/.env` points at **shared staging**, which the Heroku deploy has
almost certainly already migrated. Never run `migration:run` as a reflex here.
See AGENTS.md for the full reasoning and the read-only `migration:show` check.

## 4. Start whatever is not already up

Check first — restarting a healthy server costs the user their warm state:

```bash
lsof -nP -iTCP:8080 -iTCP:5173 -sTCP:LISTEN
```

Start each missing one in the background:

```bash
cd /Users/spot/e-dream-ai/backend  && pnpm run dev   # tsx watch, :8080
cd /Users/spot/e-dream-ai/frontend && pnpm run dev   # vite, :5173
```

Both watch their sources, so an already-running pair picks up a pull on its
own — no restart needed for code changes alone.

## 5. Confirm they are actually up

Wait on the log markers rather than guessing. Note the backend prints no
"listening"/"ready" string — match `started on port`:

- **backend** — `Worker NNNN: Connected with postgres`, then
  `e-dream.ai api 0.0.1 started on port 8080`
- **frontend** — `VITE vX.Y.Z  ready in NNN ms` and `Local:   http://localhost:5173/`

Then check both respond:

```bash
curl -o /dev/null -w 'backend  %{http_code}\n' http://localhost:8080/api/v1
curl -o /dev/null -w 'frontend %{http_code}\n' http://localhost:5173
```

Both should be `200`.

## 6. Report

One line per repo: branch, whether it was updated (and to what) or why it was
skipped, and whether the server was started or was already running.
