# autogen-engine Implementation Plan (v6)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax.

**Goal:** A Python CLI (run by hand in v1) that ranks still-image dreams by engagement via the public API, draws image pairs with a connectivity-biased weighted lottery, and creates **Kling** (`kling-25-i2v`) videos between them as the hand-created `autogen` account — adding each to one output playlist that also serves as dedup/connectivity state, under a per-run spending cap.

**Architecture:** Two deliverables. (1) **`edream_sdk` (python-api) extensions** — general-purpose methods for the endpoints the CLI needs that aren't wrapped yet. (2) **`autogen-engine`** (new Python repo): pure functions (`popularity`, `draw_pairs`, `build_kling_prompt`) + a thin service over `edream_sdk` + a `run()` orchestrator + a manual CLI entrypoint. No DB, no Redis, no provisioning script, no rate limiter, no Heroku (v1).

**Tech Stack:** Python 3, `edream_sdk` (python-api), `httpx`/`requests`, `pytest`.

**Spec:** `docs/superpowers/specs/2026-09-30-auto-prioritized-keyframe-generation-design.md` (v6 — read it).

## Global Constraints

- **`autogen-engine` is a NEW Python repo**; talks **only to the backend HTTP API** via `edream_sdk` (header `Authorization: Api-Key <key>`). No DB/Redis/admin.
- **Account created by hand** (Spot, per env): the `autogen` user, its API key, and the output playlist. The CLI reads `API_KEY`/`PLAYLIST_UUID`/… from config and looks up its own user via `get_logged_user()` — **no provisioning script, no `AUTOGEN_USER_UUID`**.
- **New API methods go in `edream_sdk` (python-api), general-purpose** (not autogen-specific). The SDK is pre-1.0; additive changes are fine, keep existing consumers (engines/video/electric-sheep) working.
- **Generation pinned to `kling-25-i2v`**; `TEMPLATE_DREAM_UUID` is a Kling dream; Kling bills per video → honor `MAX_SPEND_PER_RUN`.
- **Inputs are SFW** — enforced by the `autogen` account's **profile `nsfw=false`** flag (the feed keys off `user.nsfw`, not a request option; it's a hand-setup item) — and `status==processed`.
- **No rate limiter / jitter / retry** — a failed call fails the run; re-run. **v1 is a manual CLI**; no Heroku/Procfile/Scheduler.
- Pure functions take an injected `random.Random` (seeded) and `now: datetime`.
- `pytest`; follow `engines` repo conventions where they exist.

## Review Focus

- **Empty / tiny pool** — zero eligible images → run logs a summary, creates nothing, no throw. (B6)
- **Everything already made / pool of 1** — `draw_pairs` hits its attempt cap and returns < N (possibly 0); never loops. (B3, B6)
- **`DRY_RUN` safety** — no create / no playlist write under any input; still logs the cost estimate. (B6)
- **Spending cap** — `n` never exceeds `floor(MAX_SPEND_PER_RUN / COST_PER_VIDEO)` nor `JOBS_PER_RUN`. (B6)
- **In-flight guard + fail-fast** — skip when `autogen` has `> MAX_INFLIGHT` queue/processing dreams; raise before any create on template/playlist 404, a non-Kling template, or a template `duration ∉ {5,10}`. (B5, B6)
- **Dedup doesn't decay as the playlist grows** — `existing_pairs` pages past 500 items; `get_dreams` is never called without a `userUUID`. (B5)

---

### Task A1: Extend `edream_sdk` (python-api) with general-purpose methods

**Repo:** `python-api` (branch off its default; keep existing consumers working).
**Files:** add methods to the SDK client (e.g. `src/edream_sdk/client/*`); tests alongside the repo's existing SDK tests.

**Interfaces (add only what's missing — verify current coverage first):**
- `get_logged_user()` → `GET /auth/user` (the key's own user) — if not already present.
- `get_dream(uuid)`, `create_dream_from_prompt({name, prompt})` — likely already present; confirm.
- `get_feed(media_type=None, type=None, take, skip)` → `GET /v1/feed` (supports `type=dream`, `mediaType=image`).
- `get_ranked_feed(take, skip)` → `GET /v1/feed/ranked`.
- `get_dreams(user_uuid=None, status=None, take, skip)` → `GET /v1/dream` (single `status` enum; `userUUID` param).
- `get_playlist_items(uuid, take, skip)` → `GET /v1/playlist/:uuid/items` (items include each dream's `prompt`).
- `add_playlist_items(uuid, items)` → `POST /v1/playlist/:uuid/items` (atomic batch ≤500).

- [ ] **Step 1: Inventory** `edream_sdk` — list which of the above already exist (grep the client). Only add the gaps. Keep signatures general-purpose (no autogen naming).
- [ ] **Step 2: Write failing tests** (mocked HTTP) for each new method — path/verb/query, `Authorization: Api-Key` header, response parsing.
- [ ] **Step 3: Run → fail.**
- [ ] **Step 4: Implement** the new methods.
- [ ] **Step 5: Run → pass**; run the SDK's existing test suite to confirm no regressions.
- [ ] **Step 6: Commit** in python-api: `feat(sdk): general-purpose feed/dream/playlist/user methods`.

> `autogen-engine` depends on the updated `edream_sdk` (git ref). Pin that ref once merged.

---

### Task B1: `autogen-engine` scaffold + config

**Files:** `pyproject.toml`/`requirements.txt` (dep: `edream_sdk`, `pytest`); `autogen_engine/__init__.py`, `autogen_engine/config.py`; `tests/test_config.py`; `README.md`.

**Interfaces:** `load_config(env) -> Config`; raises `ValueError` if any of `BACKEND_API`, `API_KEY`, `TEMPLATE_DREAM_UUID`, `PLAYLIST_UUID` missing.

- [ ] **Step 1: Write failing test**

```python
# tests/test_config.py
import pytest
from autogen_engine.config import load_config
BASE = {"BACKEND_API": "https://api-stage.infinidream.ai/api",
        "API_KEY": "k", "TEMPLATE_DREAM_UUID": "t", "PLAYLIST_UUID": "p"}

def test_defaults():
    c = load_config(BASE)
    assert c.dry_run is True
    assert c.jobs_per_run == 10
    assert c.connectivity_k == 1.0
    assert c.max_inflight == 100
    assert c.max_feed_pages == 20
    assert c.cost_per_video == 0.0     # must be set in real envs
    assert c.max_spend_per_run == 0.0  # 0 => create nothing until configured
    assert c.weights["feature"] <= c.weights["playlist"]

def test_requires_core_env():
    with pytest.raises(ValueError):
        load_config({"BACKEND_API": "x"})
```

- [ ] **Step 2: Run → fail.**
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
    max_feed_pages: int = 20
    cost_per_video: float = 0.0
    max_spend_per_run: float = 0.0
    weights: dict = field(default_factory=lambda: {
        "playlist": 2.0, "recency": 3.0, "feature": 1.0,
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
        max_feed_pages=_num(env, "MAX_FEED_PAGES", 20),
        cost_per_video=_num(env, "COST_PER_VIDEO", 0.0),
        max_spend_per_run=_num(env, "MAX_SPEND_PER_RUN", 0.0),
        weights={"playlist": _num(env, "W_PLAYLIST", 2.0),
                 "recency": _num(env, "W_RECENCY", 3.0),
                 "feature": _num(env, "W_FEATURE", 1.0)},
    )
```

- [ ] **Step 4: Run → pass.**  **Step 5: Commit** `feat: scaffold + config`.

---

### Task B2: `popularity()` (pure) — v1: playlist-adds + recency (+ feature)

**Files:** `autogen_engine/scoring.py`; `tests/test_scoring.py`.

**Interfaces:** `popularity(images: list[ImageSignals], weights, half_life_days, now) -> dict[str,float]`; `ImageSignals = {uuid, playlist_count, feature_score, updated_at}` (no votes in v1 — images have none; derived-video votes are a later term). Each term min-max→[0,1], weighted, + a floor so no score is 0.

- [ ] **Step 1: Write failing tests** — empty→{}; more playlist-adds ranks higher; fresher recency higher; floor makes all-zero equal + non-zero; deterministic given fixed `now`.
- [ ] **Step 2: Run → fail.** **Step 3: Implement** (min-max helper, recency `exp(-age/half)`, `FLOOR=0.01`). **Step 4: Run → pass.** **Step 5: Commit** `feat: popularity (adds+recency+feature)`.

---

### Task B3: `draw_pairs()` (pure weighted lottery)

**Files:** `autogen_engine/selection.py`; `tests/test_selection.py`.

**Interfaces:** `draw_pairs(popularity, connected, already, n, k, rng, max_attempts=1000) -> list[tuple[str,str]]` — `w(d)=popularity[d]*(1+k*connected.get(d,0))`; draw start/end ∝ w; skip `start==end` / pair in `already` / dup this run; stop at `n` or `max_attempts` (bounded). Deterministic given `rng`.

- [ ] **Step 1: Write failing tests** — no self-pairs; skips already-made; honors `n` + deterministic; connectivity bias favors connected; bounded when exhausted (pool of 1 / all-made → ≤ available, no loop).
- [ ] **Step 2: Run → fail.** **Step 3: Implement** (`rng.choices`, dedupe, attempt cap). **Step 4: Run → pass.** **Step 5: Commit** `feat: connectivity-biased weighted-lottery pairing`.

---

### Task B4: `build_kling_prompt()` (pure)

**Files:** `autogen_engine/prompt.py`; `tests/test_prompt.py`.

**Interfaces:** `build_kling_prompt(template_prompt, start_uuid, end_uuid) -> dict` — deep-copy template, force `infinidream_algorithm="kling-25-i2v"`, set `source_dream_uuid=start`/`end_source_uuid=end`, preserve other keys (`prompt`, `duration`, `cfg_scale`, …). Accepts dict or JSON string. Does not mutate input.

- [ ] **Step 1: Write failing tests** — forces `kling-25-i2v` even if template was something else; sets both source uuids; preserves `duration`/`cfg_scale`/prompt text; no input mutation; parses JSON-string prompt.
- [ ] **Step 2: Run → fail.** **Step 3: Implement.** **Step 4: Run → pass.** **Step 5: Commit** `feat: kling-25-i2v prompt builder`.

---

### Task B5: Service layer over `edream_sdk`

**Files:** `autogen_engine/service.py`; `tests/test_service.py`.

**Interfaces:** `Service(sdk)` wrapping the (extended) `edream_sdk` client. Methods:
- `whoami() -> uuid` → `sdk.get_logged_user()`
- `inflight_count(user_uuid) -> int` → two `get_dreams(user_uuid, status="queue"/"processing")` calls, summed
- `get_template(uuid) -> dict` → `sdk.get_dream`; **validate** prompt is `kling-25-i2v` **and `duration ∈ {5,10}`** (raise if not — the worker rejects other durations). Also derive `cost_per_video = 0.35 + 0.07 × duration` for the cost cap (config `COST_PER_VIDEO` is an optional override).
- `candidate_images(max_pages) -> list[dict]` → page `get_feed(type="dream", media_type="image")`; keep `status=="processed"`; return uuid, playlist_count (len `playlistItems`), created_at, owner uuid
- `feature_scores() -> dict[uuid,float]` → from `get_ranked_feed` + walking ranked playlists' items (only if `W_FEATURE>0`)
- `existing_pairs(playlist_uuid) -> set[(start,end)]` → `get_playlist_items` **paged to exhaustion** (items cap at 500/page — a single page silently misses older pairs as the playlist grows, re-generating them); read `(source_dream_uuid,end_source_uuid)` from each item's prompt; **skip** items missing either
- `create(name, prompt) -> uuid` → `sdk.create_dream_from_prompt`
- `add_to_playlist(playlist_uuid, uuids)` → `sdk.add_playlist_items` (atomic batch); **on batch failure (e.g. 409 duplicate → whole batch dropped), fall back to adding items one at a time** so one bad item doesn't drop all

- [ ] **Step 1: Write failing tests** with a fake `sdk` — inflight sums two calls **and never calls `get_dreams` without a `userUUID`** (guard: a bare call matches all users); template validation raises on non-Kling **or non-{5,10} duration**, and derives cost from duration; candidate filtering (processed-only); `existing_pairs` pages to exhaustion (fake returns 2 pages) and skips prompts missing source/end; `add_to_playlist` falls back to per-item on batch failure. **Step 2: Run → fail.** **Step 3: Implement.** **Step 4: Run → pass.** **Step 5: Commit** `feat: edream_sdk service layer`.

---

### Task B6: `run()` orchestration (injected service; unit-tested)

**Files:** `autogen_engine/run.py`; `tests/test_run.py`.

**Interfaces:** `run(service, config, now, rng) -> RunSummary` — `whoami()` → in-flight guard (skip if `> max_inflight`) → `get_template` (fail-fast; validates Kling) → `existing_pairs` (dedup `already` + `connected`) → `candidate_images` (drop owner==self) → optional `feature_scores` → assemble `ImageSignals` → `popularity()` → `draw_pairs()` → **cost cap** using the template-derived `cost_per_video` (`0.35 + 0.07×duration`, or `COST_PER_VIDEO` override): `n = min(jobs_per_run, floor(max_spend_per_run / cost_per_video))` (0 if `cost_per_video<=0` or `max_spend_per_run<=0`) → for each of `n` pairs unless `dry_run`: `build_kling_prompt` → `service.create` (name `autogen <s8>→<e8>`) → collect; `add_to_playlist` at end. Returns `{pool, drawn, est_cost, created, skipped_inflight, dry_run}`. A `create` failure for one pair is caught/counted; template/playlist 404 or non-Kling template raises before any create.

- [ ] **Step 1: Write failing tests** with a fake service:
  - in-flight over cap → `skipped_inflight`, no creates;
  - empty pool → `drawn=0`, no throw;
  - self-owned images dropped from pool;
  - dry-run → no create/add, but `est_cost` logged/returned;
  - cost cap bounds `n` (e.g. `max_spend=5`, `cost_per_video=2` → `n≤2`); `cost_per_video<=0` → create nothing;
  - all pairs already in playlist → `created=0`;
  - happy path (dry_run false) → creates `n`, each added once, name `autogen <s8>→<e8>`;
  - non-Kling template / template 404 / playlist 404 → raises, no creates;
  - one create raises → run continues, counts successes.
- [ ] **Step 2: Run → fail.** **Step 3: Implement.** **Step 4: Run → pass.** **Step 5: Commit** `feat: run orchestration with cost cap`.

---

### Task B7: Manual CLI entrypoint

**Files:** `autogen_engine/__main__.py`; `README.md`.

**Interfaces:** `python -m autogen_engine`: `load_config(os.environ)` (supports a `.env` file; exits non-zero on missing required env) → build `edream_sdk` client + `Service` → `run(service, config, now=datetime.now(tz=utc), rng=random.Random())` → log the summary (incl. est cost) → exit 0; exit 1 on fatal (template/playlist 404, non-Kling template). **No startup jitter, no Heroku/Procfile/Scheduler.**

- [ ] **Step 1: Implement `__main__.py`** (thin; `.env` load via env; dry-run default).
- [ ] **Step 2: Smoke-run the config guard** — `BACKEND_API= python -m autogen_engine; echo $?` → non-zero, no network.
- [ ] **Step 3: Commit** `feat: manual CLI entrypoint`.

---

### Task B8: README — run by hand + rollout

**Files:** `README.md`.

- [ ] **Step 1: Document running it** — install (`pip install -e .` + `edream_sdk` ref), config via env / `.env` (table from the spec; set `MAX_SPEND_PER_RUN`; `COST_PER_VIDEO` optional — derived from the template otherwise), run `python -m autogen_engine`. **Hand-setup checklist:** create the `autogen` account as a **creator** with **profile `nsfw=false`** (this is what enforces SFW inputs), issue its API key, create the output playlist, and pick a Kling (`kling-25-i2v`, duration 5/10) `TEMPLATE_DREAM_UUID`. **Loud note:** with `MAX_SPEND_PER_RUN` unset/0, a `DRY_RUN=false` run creates **zero** videos by design (fail-safe) and only logs it — set the cap to actually generate.
- [ ] **Step 2: Document rollout** — (1) `DRY_RUN=true` (default): score/draw/log est cost, create nothing; (2) `DRY_RUN=false`, small `JOBS_PER_RUN` + `MAX_SPEND_PER_RUN`: generate a few, eyeball the `autogen` account + playlist; (3) raise caps as trust grows. Cloud/scheduling is a later, separate decision.
- [ ] **Step 3: Commit** `docs: run-by-hand + rollout`.

---

## Phase C (follow-on — NOT v1; its own spec + plan when scheduled)

General-purpose backend work, added only if/when needed (Spot: fine as long as it's general, not autogen-specific):
- **Derived-dreams endpoint + `edream_sdk` method** — "dreams derived from uuid X" (by `source_dream_uuid`/`end_source_uuid`, and/or `startKeyframe`/`endKeyframe`), so image `popularity` can include a `votes`/adds term from the videos made from it (no client-side prompt scraping). Confirm which apps write `source_dream_uuid` first.
- **Ranked candidate/signals endpoint** — returns the scored pool in one call if feed paging + per-item feature walking gets expensive.
- **Any dream (incl. video) as a start frame** via frame extraction (thumbnail/filmstrip) — widens the pool + featureRank coverage; a general platform feature.
- **Cloud runtime + scheduling** — decided after v1 proves the output.

(No discovery-exclusion task — per Spot, `autogen` is treated like any regular user.)

---

## Self-Review

**Spec coverage:** SDK extensions (A1), config incl. cost cap (B1), popularity adds+recency+feature (B2), weighted-lottery draw (B3), Kling prompt (B4), service over edream_sdk incl. whoami/inflight/template-validate/candidates/feature/existing-pairs/create/add (B5), run with in-flight guard + dry-run + dedup + self-exclusion + cost cap + fail-fast (B6), manual CLI (B7), run-by-hand + rollout (B8). Dropped vs v5: provisioning script, rate limiter/jitter, Heroku, discovery-exclusion C0 — per Spot. Deferred: derived-video endpoint, video-frame inputs, cloud scheduling.

**Placeholders:** pure tasks (B2–B4) + config (B1) carry full code; A1/B5/B6 give exact signatures + enumerated tests; A1 says "verify current `edream_sdk` coverage first" (grounded, not TBD).

**Type consistency:** `ImageSignals` (no votes), `Config` (cost fields, no AUTOGEN_USER_UUID/W_VOTES/politeness knobs), `RunSummary`, and the `popularity`/`draw_pairs`/`build_kling_prompt`/`Service` signatures are defined once and consumed unchanged by `run()`.

**Review Focus:** empty/tiny pool (B6), exhaustion bounded (B3/B6), dry-run safety + cost log (B6), spending cap (B6), in-flight guard + fail-fast (B5/B6) — each pinned to the owning task's tests.
