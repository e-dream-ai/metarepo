# autogen-engine Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** A nightly Python service that ranks still-image dreams by engagement (via the public API), draws image pairs with a connectivity-biased weighted lottery, and creates LTX i2v videos between them as the `autogen` account — adding each to one output playlist that also serves as its dedup state.

**Architecture:** Two deliverables. (1) **Backend:** a per-env provisioning script that creates the `autogen` creator user + API key + output playlist. (2) **`autogen-engine`** (new Python repo): pure functions (`popularity`, `draw_pairs`, `build_ltx_i2v_prompt`) + a thin API client (reusing `edream_sdk`) + a `run()` orchestrator, run nightly by Heroku Scheduler. No DB, no Redis, no admin — public API only.

**Tech Stack:** Python 3, `edream_sdk` (python-api), `httpx`/`requests`, `pytest`. Backend task is TypeScript (existing backend repo).

**Spec:** `docs/superpowers/specs/2026-09-30-auto-prioritized-keyframe-generation-design.md` (v5.1 — read it; the plan argues from it).

## Global Constraints

- **`autogen-engine` is a NEW standalone repo** (Python), modeled on `engines`/`electric-sheep-engine`. It depends on `edream_sdk` and talks **only to the backend HTTP API** — never the DB or Redis.
- **Auth:** header `Authorization: Api-Key <key>` (prefix `"Api-Key "`), as the `autogen` account. `edream_sdk` already sends this.
- **The backend provisioning task (Phase A) is in the `backend` repo** — base on fresh `origin/stage`, land on `stage` (deploy branch). Do not use the stale local working tree.
- **Model is pinned to `ltx-i2v`** on every generated prompt.
- **Pure functions take an injected `random.Random`** (seeded) and an injected `now: datetime` — never call `random`/`datetime.now()` inside them.
- **`DRY_RUN` defaults to `true`**: compute + log, create nothing.
- Package/test: `pytest`; follow `engines` repo layout conventions where they exist.

## Review Focus

- **Empty / tiny candidate pool** — zero eligible image dreams must not throw; the run logs a summary and creates nothing. (Task B6 test.)
- **Everything already generated** — if every drawn pair is already in the output playlist, draws redraw until exhausted and the run stops cleanly rather than looping forever. (Task B3 + B6 tests: bounded redraw attempts.)
- **`DRY_RUN` safety** — in dry-run no `POST /v1/dream` and no playlist add happen, under any input. (Task B6 test.)
- **In-flight guard** — when the `autogen` account already has `> MAX_INFLIGHT` queue/processing dreams, the run skips without creating. (Task B6 test.)
- **Mid-run API failure** — a failed create for one pair is logged and the run continues to the next; a failed *template* fetch aborts before any creates. (Task B5 + B6 tests.)

---

### Task A1: Backend — `autogen` account provisioning script (per env)

**Repo:** `backend` (branch off `origin/stage`).
**Files:**
- Create: `src/script/provision-autogen-user.ts`
- Modify: `package.json` (add a `provision:autogen` script, mirroring `cron:default-playlist`)
- Test: `src/__tests__/provision-autogen-user.test.ts` (follow the existing controller/script test pattern)

**Interfaces:**
- Produces: an idempotent script that, for the connected env, ensures a user `autogen` exists with the **creator** role (so `POST /v1/dream` passes its `CREATOR_GROUP` check), ensures it has an **API key**, ensures an **output playlist** exists owned by it, and prints `{ apiKey, playlistUuid, userUuid }`. Re-running must not duplicate the user, key, or playlist.

- [ ] **Step 1: Read the existing patterns** — how users are created, how API keys are issued (grep for the API-key entity/issue path used by `passport.client.ts`), the role enum (`CREATOR_GROUP`), playlist creation util, and the `src/script/default-playlist.ts` DataSource init/teardown. Note exact symbols.

- [ ] **Step 2: Write the failing idempotency test**

```ts
// src/__tests__/provision-autogen-user.test.ts
// Pattern: use the repo's existing test DataSource/setup (see other *.test.ts in src/__tests__).
import { ensureAutogenUser } from "script/provision-autogen-user";

it("creates the autogen creator user once and is idempotent", async () => {
  const first = await ensureAutogenUser();
  const second = await ensureAutogenUser();
  expect(second.userUuid).toBe(first.userUuid);     // no duplicate user
  expect(second.playlistUuid).toBe(first.playlistUuid); // no duplicate playlist
  expect(first.apiKey).toBeTruthy();
  // role is creator so POST /v1/dream is permitted
  expect(first.roles).toContain("creator");
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd backend && pnpm jest src/__tests__/provision-autogen-user.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 4: Implement `ensureAutogenUser()` + a thin `main()`** — look up the `autogen` user by a stable handle/email (e.g. `autogen@infinidream.ai`); create with the creator role if absent; ensure an API key (create if none); ensure the output playlist (create if none); return `{ userUuid, apiKey, playlistUuid, roles }`. `main()` runs it via the `default-playlist.ts` DataSource pattern and prints the result as JSON. Use the real role enum + key-issuing + playlist-creation utilities found in Step 1.

- [ ] **Step 5: Add the npm script** to `backend/package.json`:

```json
"provision:autogen": "NODE_PATH=dist node dist/script/provision-autogen-user.js"
```

- [ ] **Step 6: Run test + build**

Run: `cd backend && pnpm jest src/__tests__/provision-autogen-user.test.ts && pnpm run build`
Expected: PASS; build succeeds.

- [ ] **Step 7: Commit**

```bash
git add src/script/provision-autogen-user.ts src/__tests__/provision-autogen-user.test.ts package.json
git commit -m "feat(autogen): per-env provisioning script for the autogen creator account"
```

> After merge, run `pnpm run provision:autogen` once per env (dev/stage/prod); record the printed `apiKey` and `playlistUuid` for `autogen-engine`'s config.

---

### Task B1: `autogen-engine` repo scaffold + config

**Repo:** new `autogen-engine` (Python).
**Files:**
- Create: `pyproject.toml` (or `requirements.txt`) — deps: `edream_sdk` (git+ssh, as other repos install it), `httpx`, `pytest`.
- Create: `autogen_engine/__init__.py`, `autogen_engine/config.py`
- Create: `tests/test_config.py`
- Create: `README.md` (env table + rollout stages, from the spec)

**Interfaces:**
- Produces: `Config` dataclass + `load_config(env: Mapping[str, str]) -> Config` with the spec's defaults; raises `ValueError` if any of `BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID` is missing.

- [ ] **Step 1: Write the failing config test**

```python
# tests/test_config.py
import pytest
from autogen_engine.config import load_config

BASE = {
    "BACKEND_API": "https://api-stage.infinidream.ai/api",
    "API_KEY": "k", "TEMPLATE_DREAM_UUID": "t", "PLAYLIST_UUID": "p",
}

def test_defaults():
    c = load_config(BASE)
    assert c.dry_run is True
    assert c.jobs_per_run == 10
    assert c.connectivity_k == 1.0
    assert c.max_inflight == 100
    assert c.half_life_days == 30
    # politeness defaults
    assert c.startup_jitter_seconds == 300
    assert c.max_rps == 3.0
    assert c.shortlist_size == 200
    assert c.max_feed_pages == 20
    assert c.retry_max == 4
    # featureRank weight kept low by default (thin coverage)
    assert c.weights["feature"] <= c.weights["votes"]

def test_requires_core_env():
    with pytest.raises(ValueError):
        load_config({"BACKEND_API": "x"})  # missing API_KEY/TEMPLATE/PLAYLIST
```

- [ ] **Step 2: Run → fail.** `pytest tests/test_config.py` → FAIL (import error).

- [ ] **Step 3: Implement `config.py`**

```python
# autogen_engine/config.py
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Mapping

REQUIRED = ("BACKEND_API", "API_KEY", "TEMPLATE_DREAM_UUID", "PLAYLIST_UUID")

def _num(env, key, default):
    v = env.get(key)
    try:
        return type(default)(v) if v not in (None, "") else default
    except (TypeError, ValueError):
        return default

@dataclass
class Config:
    backend_api: str
    api_key: str
    template_dream_uuid: str
    playlist_uuid: str
    dry_run: bool = True
    jobs_per_run: int = 10
    connectivity_k: float = 1.0
    half_life_days: float = 30.0
    max_inflight: int = 100
    # politeness / load management
    startup_jitter_seconds: int = 300
    max_rps: float = 3.0
    shortlist_size: int = 200
    max_feed_pages: int = 20
    retry_max: int = 4
    weights: dict = field(default_factory=lambda: {
        "votes": 1.0, "playlist": 2.0, "recency": 3.0, "feature": 1.0,
    })

def load_config(env: Mapping[str, str]) -> Config:
    missing = [k for k in REQUIRED if not env.get(k)]
    if missing:
        raise ValueError(f"missing required env: {', '.join(missing)}")
    return Config(
        backend_api=env["BACKEND_API"], api_key=env["API_KEY"],
        template_dream_uuid=env["TEMPLATE_DREAM_UUID"], playlist_uuid=env["PLAYLIST_UUID"],
        dry_run=(env.get("DRY_RUN", "true").lower() != "false"),
        jobs_per_run=_num(env, "JOBS_PER_RUN", 10),
        connectivity_k=_num(env, "CONNECTIVITY_K", 1.0),
        half_life_days=_num(env, "HALF_LIFE_DAYS", 30.0),
        max_inflight=_num(env, "MAX_INFLIGHT", 100),
        startup_jitter_seconds=_num(env, "STARTUP_JITTER_SECONDS", 300),
        max_rps=_num(env, "MAX_RPS", 3.0),
        shortlist_size=_num(env, "SHORTLIST_SIZE", 200),
        max_feed_pages=_num(env, "MAX_FEED_PAGES", 20),
        retry_max=_num(env, "RETRY_MAX", 4),
        weights={
            "votes": _num(env, "W_VOTES", 1.0), "playlist": _num(env, "W_PLAYLIST", 2.0),
            "recency": _num(env, "W_RECENCY", 3.0), "feature": _num(env, "W_FEATURE", 1.0),
        },
    )
```

- [ ] **Step 4: Run → pass.** `pytest tests/test_config.py` → PASS.

- [ ] **Step 5: Commit** `feat: autogen-engine scaffold + config`.

---

### Task B2: `popularity()` (pure)

**Files:** Create `autogen_engine/scoring.py`; Test `tests/test_scoring.py`.

**Interfaces:**
- `popularity(images: list[ImageSignals], weights: dict, half_life_days: float, now: datetime) -> dict[str, float]` — returns `{uuid: score}`; each of votes/playlist/recency/feature min-max scaled to [0,1] across the set then weighted, `net_votes` clamped ≥ 0, recency `exp(-age_days/half_life)`, plus a small floor so no score is 0.
- `ImageSignals` = dataclass `{uuid, net_votes, playlist_count, feature_score, updated_at}` (`feature_score` already computed from ranked-feed order by the caller).

- [ ] **Step 1: Write failing tests** — empty→{}; higher votes ranks higher; negative net_votes clamped (ties unvoted); fresher recency scores higher; floor makes all-zero inputs non-zero and equal; deterministic given fixed `now`.

```python
# tests/test_scoring.py
from datetime import datetime, timezone
from autogen_engine.scoring import popularity, ImageSignals

NOW = datetime(2026, 10, 8, tzinfo=timezone.utc)
W = {"votes": 1, "playlist": 1, "recency": 1, "feature": 1}

def sig(uuid, **kw):
    base = dict(net_votes=0, playlist_count=0, feature_score=0.0, updated_at=NOW)
    base.update(kw)
    return ImageSignals(uuid=uuid, **base)

def test_empty():
    assert popularity([], W, 30, NOW) == {}

def test_votes_rank():
    s = popularity([sig("lo", net_votes=0), sig("hi", net_votes=10)], W, 30, NOW)
    assert s["hi"] > s["lo"]

def test_clamp_negative_votes():
    s = popularity([sig("a", net_votes=-50), sig("b", net_votes=0)],
                   {"votes": 1, "playlist": 0, "recency": 0, "feature": 0}, 30, NOW)
    assert s["a"] == s["b"]  # both floor only

def test_floor_nonzero():
    s = popularity([sig("a"), sig("b")], W, 30, NOW)
    assert s["a"] > 0 and s["a"] == s["b"]
```

- [ ] **Step 2: Run → fail.**
- [ ] **Step 3: Implement** `scoring.py` (min-max helper, clamp, recency decay, weighted sum + `FLOOR = 0.01`).
- [ ] **Step 4: Run → pass.**
- [ ] **Step 5: Commit** `feat: popularity scoring`.

---

### Task B3: `draw_pairs()` (pure weighted lottery)

**Files:** Create `autogen_engine/selection.py`; Test `tests/test_selection.py`.

**Interfaces:**
- `draw_pairs(popularity: dict[str,float], connected: dict[str,int], already: set[tuple[str,str]], n: int, k: float, rng: random.Random, max_attempts: int = 1000) -> list[tuple[str,str]]` — weight `w(d)=popularity[d]*(1+k*connected.get(d,0))`; draw `start` then `end` ∝ `w`; **skip** if `start==end` or `(start,end) in already` or already drawn this run; stop at `n` pairs or `max_attempts` exhausted (bounded — no infinite loop); deterministic given `rng`.

- [ ] **Step 1: Write failing tests**

```python
# tests/test_selection.py
import random
from autogen_engine.selection import draw_pairs

def pops(*uuids): return {u: 1.0 for u in uuids}

def test_no_self_pairs():
    out = draw_pairs(pops("a","b","c"), {}, set(), n=5, k=0, rng=random.Random(1))
    assert all(s != e for s, e in out)

def test_skips_already_made():
    out = draw_pairs(pops("a","b"), {}, already={("a","b"),("b","a")}, n=3, k=0, rng=random.Random(1))
    assert out == []  # only possible pairs are excluded

def test_honors_n_and_is_deterministic():
    a = draw_pairs(pops("a","b","c","d"), {}, set(), n=2, k=0, rng=random.Random(42))
    b = draw_pairs(pops("a","b","c","d"), {}, set(), n=2, k=0, rng=random.Random(42))
    assert a == b and len(a) == 2

def test_connectivity_bias_favored():
    # 'a' highly connected -> appears far more often across draws
    out = draw_pairs(pops("a","b","c"), {"a": 10}, set(), n=20, k=5, rng=random.Random(7))
    starts_ends = [u for pair in out for u in pair]
    assert starts_ends.count("a") > starts_ends.count("b")

def test_bounded_when_exhausted():
    # one usable pair; asking for many must terminate, not loop
    out = draw_pairs(pops("a","b"), {}, set(), n=100, k=0, rng=random.Random(1), max_attempts=200)
    assert set(map(frozenset, out)) <= {frozenset(("a","b"))}
```

- [ ] **Step 2: Run → fail.**
- [ ] **Step 3: Implement** weighted draw with `rng.choices(population, weights)`, dedupe within run, `already` skip, `max_attempts` guard.
- [ ] **Step 4: Run → pass.**
- [ ] **Step 5: Commit** `feat: connectivity-biased weighted-lottery pairing`.

---

### Task B4: `build_ltx_i2v_prompt()` (pure)

**Files:** Create `autogen_engine/prompt.py`; Test `tests/test_prompt.py`.

**Interfaces:**
- `build_ltx_i2v_prompt(template_prompt: dict, start_uuid: str, end_uuid: str) -> dict` — deep-copy `template_prompt`, force `infinidream_algorithm="ltx-i2v"`, set `source_dream_uuid=start_uuid` / `end_source_uuid=end_uuid`, leave other keys (prompt text, duration, guidance, loras, seed) intact. Accepts the template prompt whether it arrives as a dict or a JSON string (parse if str).

- [ ] **Step 1: Write failing tests** — forces ltx-i2v even if template was kling; sets both source uuids; preserves `duration`/`prompt`/`high_noise_loras`; does not mutate the input; parses a JSON-string prompt.
- [ ] **Step 2: Run → fail.**
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run → pass.**
- [ ] **Step 5: Commit** `feat: ltx-i2v prompt builder from a template dream`.

---

### Task B4b: Rate limiter + retry/backoff (injected clock + sleep)

**Files:** Create `autogen_engine/limiter.py`; Test `tests/test_limiter.py`.

**Interfaces:**
- `RateLimiter(max_rps, now_fn, sleep_fn)` with `.acquire()` — a token-bucket / min-interval
  pacer that `sleep_fn`s enough to keep calls ≤ `max_rps` and sequential. `now_fn`/`sleep_fn`
  are injected so tests are deterministic (no real wall clock).
- `retry(fn, retry_max, sleep_fn, rng, is_retryable, retry_after_fn)` — calls `fn`; on a
  retryable error (HTTP 429/5xx) sleeps `min(base*2**attempt, cap) + rng.random()*jitter`
  (honoring `Retry-After` when present) up to `retry_max`, then re-raises. Never hot-loops.

- [ ] **Step 1: Write failing tests** — limiter: N rapid `.acquire()` calls accumulate the
  expected total `sleep_fn` time for a given `max_rps`; retry: retryable error retries up to
  `retry_max` then raises, non-retryable raises immediately, `Retry-After` is honored, and
  backoff grows (assert the recorded sleep sequence with a seeded `rng` + fake `sleep_fn`).
- [ ] **Step 2: Run → fail.**
- [ ] **Step 3: Implement** `limiter.py` (pure logic; all time/sleep/rng injected).
- [ ] **Step 4: Run → pass.**
- [ ] **Step 5: Commit** `feat: injected-clock rate limiter + retry/backoff`.

---

### Task B5: API client (reuse `edream_sdk` + wrap missing endpoints)

**Files:** Create `autogen_engine/client.py`; Test `tests/test_client.py`.

**Interfaces:**
- `ApiClient(backend_api, api_key, limiter)` wrapping an `edream_sdk` client for what it already supports and issuing raw authed requests (`Authorization: Api-Key <key>`) for the rest. **Every** request goes through `limiter.acquire()` + `retry(...)` (Task B4b) so all traffic is paced and backs off on 429/5xx. Methods:
  - `get_dream(uuid) -> dict` (the template; also the **per-candidate votes** source — the feed omits votes, so fetch `upvotes`/`downvotes` here)
  - `list_image_dreams(take, skip) -> list[dict]` → `GET /v1/feed?mediaType=image` (each item has `uuid`,`status`,`playlistItems`,`user`,`created_at`/`updated_at` — **no** votes, **no** prompt; the feed applies no server-side status filter, so the caller keeps only `status == "processed"`)
  - `ranked_playlists() -> list[dict]` → `GET /v1/feed/ranked` (featured playlists in order; read their items)
  - `inflight_count() -> int` → `GET /v1/dream?status=queue,processing` (own dreams)
  - `playlist_items(uuid) -> list[dict]` → `GET /v1/playlist/:uuid/items` (items include dream `prompt`)
  - `create_dream(name, prompt) -> str` → `POST /v1/dream`, returns uuid
  - `add_to_playlist(playlist_uuid, dream_uuids: list[str])` → `POST /v1/playlist/:uuid/items` (atomic batch)

> First verify which of these `edream_sdk` already provides (check `python-api/src/edream_sdk`); reuse those, wrap only the gaps. Do not edit the `python-api` repo — keep wrappers local to `autogen-engine` (promotion to the SDK is a future item).

- [ ] **Step 1: Write failing tests** with a mocked HTTP transport — assert each method hits the right path/verb, sends `Authorization: Api-Key <key>`, and parses the documented response shape (e.g. `create_dream` returns `data.dream.uuid`; `list_image_dreams` maps the feed envelope; `add_to_playlist` posts `{items:[{type:"dream",uuid}]}`).
- [ ] **Step 2: Run → fail.**
- [ ] **Step 3: Implement** `client.py`.
- [ ] **Step 4: Run → pass.**
- [ ] **Step 5: Commit** `feat: autogen-engine API client over edream_sdk`.

---

### Task B6: `run()` orchestration (injected client; unit-tested)

**Files:** Create `autogen_engine/run.py`; Test `tests/test_run.py`.

**Interfaces:**
- `run(client, config, now, rng) -> RunSummary` — the data flow from the spec: in-flight guard → template fetch (fail-fast) → read playlist pairs (dedup `already` + `connected` counts) → page candidate image dreams (≤ `max_feed_pages`) and **keep only `status == "processed"`** (the feed has no server-side status filter) → **narrow to a shortlist**: rank by the cheap feed signals (playlist-adds + recency), take top `shortlist_size`, and **fetch votes via `get_dream` for the shortlist only** (bounds vote calls regardless of catalog size) → compute `feature_score` from `ranked_playlists()` order → assemble `ImageSignals` → `popularity()` → `draw_pairs()` → for each pair, unless `config.dry_run`: `build_ltx_i2v_prompt` → `create_dream` → collect uuid; batch `add_to_playlist` at the end. Returns `{pool, shortlisted, drawn, created, skipped_inflight: bool, dry_run: bool}`.
- A `create_dream` failure for one pair is caught, logged, counted; the run continues. A template-fetch failure raises (fail-fast, before any creates).

- [ ] **Step 1: Write failing tests** with a fake client:
  - in-flight over cap → `skipped_inflight=True`, no creates;
  - empty candidate pool → `drawn=0`, no throw;
  - dry-run → `create_dream`/`add_to_playlist` never called; `drawn>0`;
  - all pairs already in playlist → `created=0`;
  - non-`processed` images are dropped from the pool (feed returns `queue`/`failed` too);
  - shortlist caps vote fetches: with a pool > `shortlist_size`, `get_dream` is called at most `shortlist_size` times;
  - happy path (dry_run false, cap 2) → exactly 2 creates, all added to the playlist once;
  - template fetch raises → `run` raises, no creates;
  - one create raises → run continues, `created` counts the successes.
- [ ] **Step 2: Run → fail.**
- [ ] **Step 3: Implement** `run.py` (pure fns do the logic; this wires I/O + logging).
- [ ] **Step 4: Run → pass.**
- [ ] **Step 5: Commit** `feat: autogen-engine run orchestration`.

---

### Task B7: Entrypoint + Heroku wiring

**Files:** Create `autogen_engine/__main__.py`; `Procfile`; update `README.md`.

**Interfaces:**
- `python -m autogen_engine`: `load_config(os.environ)` (exits non-zero with the clear error if required env missing) → **`sleep(rng.uniform(0, config.startup_jitter_seconds))`** so the run lands off the :00 boundary → build `RateLimiter(config.max_rps, ...)` + `ApiClient(..., limiter)` → `run(client, config, now=datetime.now(tz=utc), rng=random.Random())` → log the summary → exit 0; exit 1 on fatal (e.g. template fetch). (The jitter sleep is in the entrypoint, not `run()`, so `run()` stays fast/deterministic in tests.)

- [ ] **Step 1: Implement `__main__.py`** (thin; mirrors the spec entrypoint, incl. startup jitter).
- [ ] **Step 2: Add `Procfile`** — no `web`; document the Heroku Scheduler command `python -m autogen_engine` (nightly, UTC).
- [ ] **Step 3: Smoke-run the config guard** — `python -m autogen_engine` with required env unset exits non-zero with the missing-env error (no network).

Run: `BACKEND_API= python -m autogen_engine; echo $?` → non-zero.

- [ ] **Step 4: Commit** `feat: autogen-engine entrypoint + Heroku Scheduler wiring`.

---

### Task B8: Deploy + staged rollout (docs/ops)

**Files:** `README.md` (ops section).

- [ ] **Step 1: Document deploy** — create the `autogen-engine` Heroku app (no web dyno), add the Scheduler add-on with `python -m autogen_engine` nightly (UTC). Set env from Task A1's output (`API_KEY`, `PLAYLIST_UUID`) + `BACKEND_API`, `TEMPLATE_DREAM_UUID`, and tunables.
- [ ] **Step 2: Document the rollout gates** — (1) `DRY_RUN=true` (default): score/draw/log, create nothing; inspect the summary + the `feature_score` coverage (drop/keep `W_FEATURE`). (2) `DRY_RUN=false`, `JOBS_PER_RUN=2`: eyeball the `autogen` account's output + the playlist. (3) raise `JOBS_PER_RUN`, leave scheduled.
- [ ] **Step 3: Commit** `docs: autogen-engine deploy + rollout`.

---

## Phase C (follow-on — NOT part of v1 execution; gets its own spec + plan)

> Rationale: v1 hammers the public API — paging the whole stills feed and one
> `GET /v1/dream/:uuid` per candidate for votes (which the feed omits, likely on
> purpose). Moving the scoring into the API server is far more efficient (one SQL pass,
> no pagination, no per-image calls) and avoids ever shipping raw vote counts to a
> client. Do this once v1 has proven the selection logic in practice. These are
> outlines, to be expanded into a proper spec/plan when scheduled — not executed from
> this document.

**Backend tasks (api server):**
- **C1 — Candidate endpoint.** `GET /v1/autogen/candidates` (regular-user) returning
  processed, non-hidden image dreams **scored/ordered server-side** — votes +
  playlist-adds + recency + featureRank folded in one query, `status = processed`
  enforced server-side, votes never leaving the server as raw counts. Reuses the pure
  scoring logic (port `popularity` to TS, or compute in SQL). Resolves spec API-improvements
  #1–#3.
- **C2 — (optional interim) batch votes** `POST /v1/dream/votes` (list of uuids → counts)
  if C1 is deferred — a cheaper stopgap than per-image fetches.
- **C3 — `FeedItem` coverage check/backfill** (or have C1 query dreams directly), so no
  processed public image is invisible to the pool.

**`autogen-engine` change:**
- **C4 — Thin client.** Replace `list_image_dreams` + per-candidate `get_dream` votes +
  client-side `popularity` with a single call to C1; keep `draw_pairs`,
  `build_ltx_i2v_prompt`, dedup-via-playlist, and create+add unchanged. (Pure
  `draw_pairs`/`build_ltx_i2v_prompt` and their tests carry over verbatim.) Alternatively,
  if the backend owns the whole loop, `autogen-engine` is retired.

**Decision at that time:** does the backend just *rank* (autogen-engine still draws +
creates) or own the *entire* loop (draw + create + playlist add)? Ranking-only keeps the
selection logic testable in Python and the create path as the `autogen` user; full-backend
removes the service but needs the pairing/lottery + playlist writes server-side.

---

## Self-Review

**Spec coverage:** provisioning (A1), config (B1), popularity (B2), weighted-lottery draw (B3), ltx-i2v prompt from a template dream (B4), API client over `edream_sdk` incl. feed/ranked/playlist/dream-status/create/batch-add (B5), orchestration with in-flight guard + dry-run + dedup-via-playlist + fail-fast template + per-pair failure tolerance + batch add (B6), entrypoint + Scheduler (B7), deploy + staged rollout (B8). Deferred items (watch-through #691, ranked-signals endpoint, cooldown re-gen, SDK promotion) correctly omitted. No new backend query endpoints — the API survey confirmed they exist for a creator account.

**Placeholders:** pure-function tasks (B2–B4) and config (B1) carry full code; B5/B6 give exact method signatures + enumerated test cases; A1 and B5 defer to named existing files/SDK to copy patterns from (grounded, not TBD) because those symbols must be read from the live `backend`/`python-api` repos.

**Type consistency:** `ImageSignals`, `Config`, `RunSummary`, and the `popularity`/`draw_pairs`/`build_ltx_i2v_prompt`/`ApiClient` signatures are defined once and consumed unchanged by `run()` (B6).

**Review Focus:** empty pool (B6), all-already-made + bounded redraw (B3+B6), dry-run safety (B6), in-flight guard (B6), mid-run create failure vs fail-fast template (B5+B6) — each pinned to the owning task's tests.
