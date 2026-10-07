# Auto-Prioritized Keyframe Video Generation — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A nightly backend job that ranks still image Dreams by engagement, pairs the most relevant ones (hub-and-spoke), and auto-enqueues LTX i2v generation between them — unattended, system-owned, visible immediately.

**Architecture:** Four pure functions (scoring, pairing, dedup, prompt-building) hold all the logic and are DB-free and fully unit-tested. A thin orchestration function (`runAutoGeneration`) wires them with an injected `deps` gateway, so it is also unit-testable with fakes. A real DB gateway implements the gateway interface against TypeORM. A standalone script (`cron:auto-generate`) runs it once, invoked nightly by Heroku Scheduler. Generation reuses the existing `processDreamRequest` → `ltxi2v` queue path; no new queue or worker.

**Tech Stack:** TypeScript, Express, TypeORM (PostgreSQL), BullMQ, Jest, pnpm. Backend repo only.

**Spec:** `docs/superpowers/specs/2026-09-30-auto-prioritized-keyframe-generation-design.md` (read it — the plan argues from it).

## Global Constraints

- **Base on fresh `origin/stage`, and land on `stage`.** The local `backend` working tree is ~58 commits behind with uncommitted changes. Execute in an isolated worktree branched from `origin/stage` (`superpowers:using-git-worktrees`); the PR targets `stage` (deploy branch). Do NOT build on the stale working tree. Verified against `origin/stage` @ `48bb383`.
- **Package manager:** `pnpm` (never npm/yarn). Tests: `pnpm run test:unit` or `pnpm jest <path>`.
- **Imports use the src base-url aliases** already in the codebase (e.g. `import { X } from "utils/foo"`, `"entities/Dream.entity"`, `"types/..."`) — not relative `../../`.
- **Do NOT run `pnpm run migration:run`** against shared staging as part of implementation (it writes to the shared DB — see AGENTS.md). Hand-write the migration file; it is applied at deploy time. Unit tests must not need it.
- **Model is pinned to `ltx-i2v`** on every generated prompt, overriding the template's stored model.
- **Generated Dreams are owned by the system/curation account and visible immediately** (no hidden flag).
- Pure functions take an injected `now: Date` — never call `Date.now()`/`new Date()` inside them.

## Review Focus

- **Empty / tiny candidate set** — zero or `< hubsPerRun` eligible image Dreams must not throw; the run produces fewer/zero candidates and logs a summary. (Pinned in Task 3 + Task 7 tests.)
- **Duplicate-triple race / re-run** — two rows for the same `(start,end,template)` must be impossible; the unique constraint rejects the second, and the run treats the conflict as "already done," not a crash. (Pinned in Task 6 + Task 8 tests.)
- **Enqueue failure after commit** — if `processDreamRequest` throws after the Dream+log are committed, the Dream is marked failed and the run continues to the next candidate. (Pinned in Task 8 tests.)
- **Missing / malformed template transition** — if the designated `EditorProject` or transition key is absent, the run fails fast with a clear error before creating any Dreams. (Pinned in Task 7 tests.)
- **All candidates share one owner** — diversity picks must degrade gracefully (fall back to quality picks) rather than produce fewer-than-requested or duplicate spokes. (Pinned in Task 3 tests.)

---

### Task 1: Shared types + config constants

**Files:**
- Create: `src/types/auto-generation.types.ts`
- Create: `src/constants/auto-generation.constants.ts`
- Test: `src/constants/__tests__/auto-generation.constants.test.ts`

**Interfaces:**
- Produces: types `ImageDreamSignals`, `ScoredImageDream`, `Candidate`, `TemplateSettings`, `AutoGenLogRow`, `ScoringWeights`, `AutoGenConfig`; and `loadAutoGenConfig(env): AutoGenConfig` with documented defaults.

- [ ] **Step 1: Write the types file**

```ts
// src/types/auto-generation.types.ts
export interface ImageDreamSignals {
  uuid: string;
  ownerUuid: string;
  netVotes: number;       // raw upvotes - downvotes (clamped in scoring)
  playlistCount: number;
  featureRank: number;
  updatedAt: Date;
}

export interface ScoredImageDream {
  uuid: string;
  ownerUuid: string;
  score: number;          // 0..~sum(weights)
}

export interface Candidate {
  startDreamUuid: string; // hub  -> source_dream_uuid
  endDreamUuid: string;   // spoke -> end_source_uuid
  pairScore: number;      // score(start) + score(end)
}

// The subset of a flow transition's settings the generator uses.
// model is intentionally omitted: it is always pinned to "ltx-i2v".
export interface TemplateSettings {
  prompt: string;
  negativePrompt: string;
  duration: number;
  guidance: number;
  seed: number;
  highNoiseLoras: Array<{ path: string; scale: number }>;
}

export interface AutoGenLogRow {
  startDreamUuid: string;
  endDreamUuid: string;
  templateRef: string;    // `${projectUuid}#${transitionKey}`
  pairScore: number;
  createdAt: Date;
}

export interface ScoringWeights {
  votes: number;
  playlist: number;
  feature: number;
  recency: number;
}

export interface AutoGenConfig {
  weights: ScoringWeights;
  halfLifeDays: number;
  hubsPerRun: number;
  spokesPerHub: number;
  diversityRatio: number;
  bothDirections: boolean;
  maxJobsPerRun: number;
  cooldownDays: number;
  regenScoreRisePct: number;
  maxInflightAutoJobs: number;
  templateProjectUuid: string;
  templateTransitionKey: string | null; // null => first transition in frame order
  curationUserId: number;
}
```

- [ ] **Step 2: Write the failing test for config defaults + env parsing**

```ts
// src/constants/__tests__/auto-generation.constants.test.ts
import { loadAutoGenConfig } from "constants/auto-generation.constants";

describe("loadAutoGenConfig", () => {
  it("applies documented defaults when env is empty", () => {
    const c = loadAutoGenConfig({});
    expect(c.hubsPerRun).toBe(10);
    expect(c.spokesPerHub).toBe(3);
    expect(c.maxJobsPerRun).toBe(50);
    expect(c.cooldownDays).toBe(30);
    expect(c.regenScoreRisePct).toBe(50);
    expect(c.maxInflightAutoJobs).toBe(100);
    expect(c.halfLifeDays).toBe(30);
    expect(c.diversityRatio).toBeCloseTo(1 / 3);
    // v1 weighting biases to featureRank + recency (stills accrue few votes)
    expect(c.weights.feature).toBeGreaterThan(c.weights.votes);
    expect(c.weights.recency).toBeGreaterThan(c.weights.votes);
    expect(c.templateTransitionKey).toBeNull();
  });

  it("parses overrides from env strings", () => {
    const c = loadAutoGenConfig({
      AUTO_GEN_MAX_JOBS_PER_RUN: "5",
      AUTO_GEN_TEMPLATE_PROJECT_UUID: "proj-1",
      AUTO_GEN_TEMPLATE_TRANSITION_KEY: "a::b",
      CURATION_USER_ID: "42",
    });
    expect(c.maxJobsPerRun).toBe(5);
    expect(c.templateProjectUuid).toBe("proj-1");
    expect(c.templateTransitionKey).toBe("a::b");
    expect(c.curationUserId).toBe(42);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `pnpm jest src/constants/__tests__/auto-generation.constants.test.ts`
Expected: FAIL — cannot find module `constants/auto-generation.constants`.

- [ ] **Step 4: Write the constants file**

```ts
// src/constants/auto-generation.constants.ts
import type { AutoGenConfig } from "types/auto-generation.types";

const num = (v: string | undefined, d: number): number =>
  v === undefined || v === "" || Number.isNaN(Number(v)) ? d : Number(v);

export const loadAutoGenConfig = (
  env: Record<string, string | undefined>,
): AutoGenConfig => ({
  weights: {
    votes: num(env.AUTO_GEN_W_VOTES, 1),
    playlist: num(env.AUTO_GEN_W_PLAYLIST, 2),
    feature: num(env.AUTO_GEN_W_FEATURE, 4),
    recency: num(env.AUTO_GEN_W_RECENCY, 3),
  },
  halfLifeDays: num(env.AUTO_GEN_HALF_LIFE_DAYS, 30),
  hubsPerRun: num(env.AUTO_GEN_HUBS_PER_RUN, 10),
  spokesPerHub: num(env.AUTO_GEN_SPOKES_PER_HUB, 3),
  diversityRatio: num(env.AUTO_GEN_DIVERSITY_RATIO, 1 / 3),
  bothDirections: env.AUTO_GEN_BOTH_DIRECTIONS === "true",
  maxJobsPerRun: num(env.AUTO_GEN_MAX_JOBS_PER_RUN, 50),
  cooldownDays: num(env.AUTO_GEN_COOLDOWN_DAYS, 30),
  regenScoreRisePct: num(env.AUTO_GEN_REGEN_SCORE_RISE_PCT, 50),
  maxInflightAutoJobs: num(env.AUTO_GEN_MAX_INFLIGHT_AUTO_JOBS, 100),
  templateProjectUuid: env.AUTO_GEN_TEMPLATE_PROJECT_UUID ?? "",
  templateTransitionKey: env.AUTO_GEN_TEMPLATE_TRANSITION_KEY ?? null,
  curationUserId: num(env.CURATION_USER_ID, 0),
});
```

- [ ] **Step 5: Run test to verify it passes**

Run: `pnpm jest src/constants/__tests__/auto-generation.constants.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add src/types/auto-generation.types.ts src/constants/auto-generation.constants.ts src/constants/__tests__/auto-generation.constants.test.ts
git commit -m "feat(auto-gen): shared types and config constants"
```

---

### Task 2: Scoring (pure)

**Files:**
- Create: `src/utils/keyframe-scoring.ts`
- Test: `src/utils/__tests__/keyframe-scoring.test.ts`

**Interfaces:**
- Consumes: `ImageDreamSignals`, `ScoredImageDream`, `ScoringWeights` (Task 1).
- Produces: `scoreImageDreams(signals: ImageDreamSignals[], weights: ScoringWeights, halfLifeDays: number, now: Date): ScoredImageDream[]` — returns all inputs scored, sorted by `score` desc (tie-break `uuid` asc). Each term min-max normalized to [0,1] across the set before weighting; `netVotes` clamped at 0; recency = `exp(-ageDays/halfLifeDays)`.

- [ ] **Step 1: Write the failing tests**

```ts
// src/utils/__tests__/keyframe-scoring.test.ts
import { scoreImageDreams } from "utils/keyframe-scoring";
import type { ImageDreamSignals, ScoringWeights } from "types/auto-generation.types";

const W: ScoringWeights = { votes: 1, playlist: 1, feature: 1, recency: 1 };
const NOW = new Date("2026-10-05T00:00:00Z");

const sig = (o: Partial<ImageDreamSignals> & { uuid: string }): ImageDreamSignals => ({
  ownerUuid: "owner-" + o.uuid,
  netVotes: 0,
  playlistCount: 0,
  featureRank: 0,
  updatedAt: NOW,
  ...o,
});

describe("scoreImageDreams", () => {
  it("returns empty for empty input", () => {
    expect(scoreImageDreams([], W, 30, NOW)).toEqual([]);
  });

  it("ranks higher engagement above lower, sorted desc", () => {
    const out = scoreImageDreams(
      [sig({ uuid: "low", netVotes: 0 }), sig({ uuid: "high", netVotes: 10 })],
      W, 30, NOW,
    );
    expect(out.map((s) => s.uuid)).toEqual(["high", "low"]);
    expect(out[0].score).toBeGreaterThan(out[1].score);
  });

  it("clamps negative netVotes to zero (downvoted cannot go below unvoted)", () => {
    const out = scoreImageDreams(
      [sig({ uuid: "a", netVotes: -100 }), sig({ uuid: "b", netVotes: 0 })],
      { votes: 1, playlist: 0, feature: 0, recency: 0 }, 30, NOW,
    );
    // both normalize to 0 on the votes term -> equal score, uuid tie-break
    expect(out.map((s) => s.uuid)).toEqual(["a", "b"]);
    expect(out[0].score).toBe(out[1].score);
  });

  it("decays recency by age", () => {
    const old = new Date("2026-09-05T00:00:00Z"); // 30 days => ~0.368
    const out = scoreImageDreams(
      [sig({ uuid: "fresh", updatedAt: NOW }), sig({ uuid: "old", updatedAt: old })],
      { votes: 0, playlist: 0, feature: 0, recency: 1 }, 30, NOW,
    );
    const fresh = out.find((s) => s.uuid === "fresh")!;
    const stale = out.find((s) => s.uuid === "old")!;
    expect(fresh.score).toBeGreaterThan(stale.score);
  });

  it("is deterministic given a fixed now", () => {
    const input = [sig({ uuid: "a", netVotes: 3 }), sig({ uuid: "b", playlistCount: 2 })];
    expect(scoreImageDreams(input, W, 30, NOW)).toEqual(scoreImageDreams(input, W, 30, NOW));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm jest src/utils/__tests__/keyframe-scoring.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the implementation**

```ts
// src/utils/keyframe-scoring.ts
import type {
  ImageDreamSignals,
  ScoredImageDream,
  ScoringWeights,
} from "types/auto-generation.types";

const DAY_MS = 24 * 60 * 60 * 1000;

const minMax = (values: number[]): ((v: number) => number) => {
  const min = Math.min(...values);
  const max = Math.max(...values);
  const span = max - min;
  return (v: number) => (span === 0 ? 0 : (v - min) / span);
};

export const scoreImageDreams = (
  signals: ImageDreamSignals[],
  weights: ScoringWeights,
  halfLifeDays: number,
  now: Date,
): ScoredImageDream[] => {
  if (signals.length === 0) return [];

  const votes = signals.map((s) => Math.max(0, s.netVotes));
  const playlists = signals.map((s) => s.playlistCount);
  const features = signals.map((s) => s.featureRank);

  const normVotes = minMax(votes);
  const normPlaylist = minMax(playlists);
  const normFeature = minMax(features);

  const recency = (updatedAt: Date): number => {
    const ageDays = Math.max(0, (now.getTime() - updatedAt.getTime()) / DAY_MS);
    return Math.exp(-ageDays / halfLifeDays);
  };

  return signals
    .map((s) => ({
      uuid: s.uuid,
      ownerUuid: s.ownerUuid,
      score:
        weights.votes * normVotes(Math.max(0, s.netVotes)) +
        weights.playlist * normPlaylist(s.playlistCount) +
        weights.feature * normFeature(s.featureRank) +
        weights.recency * recency(s.updatedAt),
    }))
    .sort((a, b) => b.score - a.score || a.uuid.localeCompare(b.uuid));
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm jest src/utils/__tests__/keyframe-scoring.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/utils/keyframe-scoring.ts src/utils/__tests__/keyframe-scoring.test.ts
git commit -m "feat(auto-gen): per-run normalized image-dream scoring"
```

---

### Task 3: Pairing — hub-and-spoke (pure)

**Files:**
- Create: `src/utils/candidate-pairing.ts`
- Test: `src/utils/__tests__/candidate-pairing.test.ts`

**Interfaces:**
- Consumes: `ScoredImageDream`, `Candidate` (Task 1).
- Produces: `buildCandidates(scored: ScoredImageDream[], cfg: { hubsPerRun; spokesPerHub; diversityRatio; bothDirections }): Candidate[]` — hubs = top `hubsPerRun`; each hub gets `spokesPerHub` partners (quality + deterministic diversity, see spec); skips `start===end`; dedupes ordered pairs and (when `!bothDirections`) their reverse; sorted by `pairScore` desc.

- [ ] **Step 1: Write the failing tests**

```ts
// src/utils/__tests__/candidate-pairing.test.ts
import { buildCandidates } from "utils/candidate-pairing";
import type { ScoredImageDream } from "types/auto-generation.types";

const d = (uuid: string, score: number, ownerUuid = "o-" + uuid): ScoredImageDream =>
  ({ uuid, score, ownerUuid });

const CFG = { hubsPerRun: 2, spokesPerHub: 2, diversityRatio: 0.5, bothDirections: false };

describe("buildCandidates", () => {
  it("returns empty for empty input", () => {
    expect(buildCandidates([], CFG)).toEqual([]);
  });

  it("never pairs a dream with itself", () => {
    const out = buildCandidates([d("a", 5), d("b", 4), d("c", 3)], CFG);
    expect(out.every((c) => c.startDreamUuid !== c.endDreamUuid)).toBe(true);
  });

  it("uses top-scored dreams as hubs", () => {
    const out = buildCandidates([d("a", 9), d("b", 8), d("c", 1), d("d", 1)], CFG);
    const hubs = new Set(out.map((c) => c.startDreamUuid));
    expect(hubs).toEqual(new Set(["a", "b"]));
  });

  it("caps spokes per hub", () => {
    const out = buildCandidates(
      [d("a", 9), d("b", 8), d("c", 7), d("d", 6), d("e", 5)],
      { hubsPerRun: 1, spokesPerHub: 2, diversityRatio: 0, bothDirections: false },
    );
    expect(out.filter((c) => c.startDreamUuid === "a").length).toBe(2);
  });

  it("sets pairScore = start score + end score", () => {
    const out = buildCandidates(
      [d("a", 9), d("b", 8)],
      { hubsPerRun: 1, spokesPerHub: 1, diversityRatio: 0, bothDirections: false },
    );
    expect(out[0]).toMatchObject({ startDreamUuid: "a", endDreamUuid: "b", pairScore: 17 });
  });

  it("falls back gracefully when all dreams share one owner (diversity impossible)", () => {
    const sameOwner = [d("a", 9, "o"), d("b", 8, "o"), d("c", 7, "o")];
    const out = buildCandidates(sameOwner, { hubsPerRun: 1, spokesPerHub: 2, diversityRatio: 1, bothDirections: false });
    // still yields 2 distinct spokes, no duplicates, no self-pair
    const spokes = out.filter((c) => c.startDreamUuid === "a").map((c) => c.endDreamUuid);
    expect(new Set(spokes).size).toBe(spokes.length);
    expect(spokes.length).toBe(2);
    expect(spokes).not.toContain("a");
  });

  it("does not emit both a->b and b->a when bothDirections is false", () => {
    const out = buildCandidates(
      [d("a", 9), d("b", 8)],
      { hubsPerRun: 2, spokesPerHub: 1, diversityRatio: 0, bothDirections: false },
    );
    const keys = out.map((c) => [c.startDreamUuid, c.endDreamUuid].sort().join("::"));
    expect(new Set(keys).size).toBe(keys.length);
  });

  it("sorts output by pairScore desc", () => {
    const out = buildCandidates([d("a", 9), d("b", 8), d("c", 7)], { hubsPerRun: 3, spokesPerHub: 1, diversityRatio: 0, bothDirections: false });
    const scores = out.map((c) => c.pairScore);
    expect(scores).toEqual([...scores].sort((x, y) => y - x));
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm jest src/utils/__tests__/candidate-pairing.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the implementation**

```ts
// src/utils/candidate-pairing.ts
import type { Candidate, ScoredImageDream } from "types/auto-generation.types";

interface PairingCfg {
  hubsPerRun: number;
  spokesPerHub: number;
  diversityRatio: number;
  bothDirections: boolean;
}

export const buildCandidates = (
  scored: ScoredImageDream[],
  cfg: PairingCfg,
): Candidate[] => {
  const byScore = [...scored].sort(
    (a, b) => b.score - a.score || a.uuid.localeCompare(b.uuid),
  );
  const hubs = byScore.slice(0, cfg.hubsPerRun);
  const candidates: Candidate[] = [];
  const emitted = new Set<string>();

  const canEmit = (start: string, end: string): boolean => {
    const ordered = `${start}::${end}`;
    const reverse = `${end}::${start}`;
    if (emitted.has(ordered)) return false;
    if (!cfg.bothDirections && emitted.has(reverse)) return false;
    return true;
  };

  for (const hub of hubs) {
    const pool = byScore.filter((d) => d.uuid !== hub.uuid);
    const nDiversity = Math.round(cfg.spokesPerHub * cfg.diversityRatio);
    const nQuality = cfg.spokesPerHub - nDiversity;

    const chosen: ScoredImageDream[] = [];
    const take = (d: ScoredImageDream) => {
      if (chosen.some((c) => c.uuid === d.uuid)) return;
      chosen.push(d);
    };

    // Quality picks: highest scorers.
    for (const d of pool) {
      if (chosen.length >= nQuality) break;
      take(d);
    }
    // Diversity picks: lowest-scored of a *different* owner first; tie-break uuid.
    const diversityPool = [...pool]
      .filter((d) => d.ownerUuid !== hub.ownerUuid)
      .sort((a, b) => a.score - b.score || a.uuid.localeCompare(b.uuid));
    for (const d of diversityPool) {
      if (chosen.length >= cfg.spokesPerHub) break;
      take(d);
    }
    // Fallback: if diversity was impossible (same-owner set), fill from pool.
    for (const d of pool) {
      if (chosen.length >= cfg.spokesPerHub) break;
      take(d);
    }

    for (const spoke of chosen) {
      if (spoke.uuid === hub.uuid) continue;
      if (!canEmit(hub.uuid, spoke.uuid)) continue;
      emitted.add(`${hub.uuid}::${spoke.uuid}`);
      candidates.push({
        startDreamUuid: hub.uuid,
        endDreamUuid: spoke.uuid,
        pairScore: hub.score + spoke.score,
      });
    }
  }

  return candidates.sort(
    (a, b) =>
      b.pairScore - a.pairScore ||
      a.startDreamUuid.localeCompare(b.startDreamUuid) ||
      a.endDreamUuid.localeCompare(b.endDreamUuid),
  );
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm jest src/utils/__tests__/candidate-pairing.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/utils/candidate-pairing.ts src/utils/__tests__/candidate-pairing.test.ts
git commit -m "feat(auto-gen): hub-and-spoke candidate pairing"
```

---

### Task 4: Dedup predicate (pure)

**Files:**
- Create: `src/utils/auto-generation-dedup.ts`
- Test: `src/utils/__tests__/auto-generation-dedup.test.ts`

**Interfaces:**
- Consumes: `Candidate`, `AutoGenLogRow` (Task 1).
- Produces: `shouldGenerate(candidate: Candidate, priorRows: AutoGenLogRow[], cfg: { cooldownDays; regenScoreRisePct }, now: Date): boolean` — `priorRows` are the log rows already filtered to this `(start,end,template)` triple. Generate if none; or newest older than cooldown; or `pairScore >= (1 + pct/100) * max(prior pair_score)`.

- [ ] **Step 1: Write the failing tests**

```ts
// src/utils/__tests__/auto-generation-dedup.test.ts
import { shouldGenerate } from "utils/auto-generation-dedup";
import type { AutoGenLogRow, Candidate } from "types/auto-generation.types";

const NOW = new Date("2026-10-05T00:00:00Z");
const cand: Candidate = { startDreamUuid: "a", endDreamUuid: "b", pairScore: 10 };
const CFG = { cooldownDays: 30, regenScoreRisePct: 50 };

const row = (daysAgo: number, pairScore: number): AutoGenLogRow => ({
  startDreamUuid: "a",
  endDreamUuid: "b",
  templateRef: "p#t",
  pairScore,
  createdAt: new Date(NOW.getTime() - daysAgo * 86400000),
});

describe("shouldGenerate", () => {
  it("generates when there is no prior row", () => {
    expect(shouldGenerate(cand, [], CFG, NOW)).toBe(true);
  });
  it("skips within cooldown when score has not risen enough", () => {
    expect(shouldGenerate(cand, [row(5, 10)], CFG, NOW)).toBe(false);
  });
  it("generates after cooldown has elapsed", () => {
    expect(shouldGenerate(cand, [row(31, 10)], CFG, NOW)).toBe(true);
  });
  it("generates within cooldown when score rose >= threshold vs MAX prior", () => {
    // max prior = 6; threshold = 9; candidate 10 >= 9
    expect(shouldGenerate(cand, [row(5, 6), row(2, 5)], CFG, NOW)).toBe(true);
  });
  it("compares against MAX prior, not latest, to avoid repeat re-fires", () => {
    // latest is 5 but max is 9; threshold = 13.5; candidate 10 < 13.5 -> skip
    expect(shouldGenerate(cand, [row(10, 9), row(1, 5)], CFG, NOW)).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm jest src/utils/__tests__/auto-generation-dedup.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the implementation**

```ts
// src/utils/auto-generation-dedup.ts
import type { AutoGenLogRow, Candidate } from "types/auto-generation.types";

const DAY_MS = 86400000;

export const shouldGenerate = (
  candidate: Candidate,
  priorRows: AutoGenLogRow[],
  cfg: { cooldownDays: number; regenScoreRisePct: number },
  now: Date,
): boolean => {
  if (priorRows.length === 0) return true;

  const newest = priorRows.reduce((a, b) => (a.createdAt > b.createdAt ? a : b));
  const ageDays = (now.getTime() - newest.createdAt.getTime()) / DAY_MS;
  if (ageDays >= cfg.cooldownDays) return true;

  const maxPrior = Math.max(...priorRows.map((r) => r.pairScore));
  const threshold = maxPrior * (1 + cfg.regenScoreRisePct / 100);
  return candidate.pairScore >= threshold;
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm jest src/utils/__tests__/auto-generation-dedup.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/utils/auto-generation-dedup.ts src/utils/__tests__/auto-generation-dedup.test.ts
git commit -m "feat(auto-gen): cooldown + score-rise dedup predicate"
```

---

### Task 5: LTX i2v prompt builder (pure)

**Files:**
- Create: `src/utils/auto-gen-prompt.ts`
- Test: `src/utils/__tests__/auto-gen-prompt.test.ts`

**Interfaces:**
- Consumes: `TemplateSettings` (Task 1).
- Produces: `buildLtxI2vPrompt(settings: TemplateSettings, startDreamUuid: string, endDreamUuid: string): Record<string, unknown>` — mirrors the frontend ltx-i2v branch of `build-video-algo-params.ts`: always `infinidream_algorithm:"ltx-i2v"`, `prompt`, `source_dream_uuid`, `end_source_uuid`, `duration`; optional `negative_prompt` (trimmed non-empty), `high_noise_loras` (whole array if present), `seed` (finite), `guidance` (finite). Guidance param name is `"guidance"` for ltx-i2v (`GUIDANCE_PARAM["ltx-i2v"]`).

> **LoRA-fit note:** the frontend's `actionLorasFitModel` depends on the preset
> catalog, which is frontend-only and must not be ported. We pass the template's
> `highNoiseLoras` through as-is. **This makes the template author responsible for
> choosing LTX-appropriate LoRAs:** a flow's stored default model is `kling-25-i2v`,
> so a template transition may carry Kling LoRAs that LTX cannot load. Document in
> the deploy notes (Task 10) that the designated auto-gen template transition must
> use LTX LoRAs (or none). Documented simplification.

- [ ] **Step 1: Write the failing tests**

```ts
// src/utils/__tests__/auto-gen-prompt.test.ts
import { buildLtxI2vPrompt } from "utils/auto-gen-prompt";
import type { TemplateSettings } from "types/auto-generation.types";

const base: TemplateSettings = {
  prompt: "slow zoom in",
  negativePrompt: "",
  duration: 5,
  guidance: 3,
  seed: -1,
  highNoiseLoras: [],
};

describe("buildLtxI2vPrompt", () => {
  it("always pins ltx-i2v and sets both image sources", () => {
    const p = buildLtxI2vPrompt(base, "start-uuid", "end-uuid");
    expect(p).toMatchObject({
      infinidream_algorithm: "ltx-i2v",
      prompt: "slow zoom in",
      source_dream_uuid: "start-uuid",
      end_source_uuid: "end-uuid",
      duration: 5,
      guidance: 3,
    });
  });
  it("omits negative_prompt when blank, includes when set", () => {
    expect(buildLtxI2vPrompt(base, "s", "e")).not.toHaveProperty("negative_prompt");
    const p = buildLtxI2vPrompt({ ...base, negativePrompt: "  blurry  " }, "s", "e");
    expect(p.negative_prompt).toBe("blurry");
  });
  it("includes high_noise_loras as the whole array when present", () => {
    const loras = [{ path: "x.safetensors", scale: 0.4 }];
    const p = buildLtxI2vPrompt({ ...base, highNoiseLoras: loras }, "s", "e");
    expect(p.high_noise_loras).toEqual(loras);
  });
  it("omits seed when not finite, includes when finite (incl. -1)", () => {
    expect(buildLtxI2vPrompt({ ...base, seed: Number.NaN }, "s", "e")).not.toHaveProperty("seed");
    expect(buildLtxI2vPrompt({ ...base, seed: -1 }, "s", "e").seed).toBe(-1);
  });
  it("omits guidance when not finite", () => {
    expect(buildLtxI2vPrompt({ ...base, guidance: Number.NaN }, "s", "e")).not.toHaveProperty("guidance");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm jest src/utils/__tests__/auto-gen-prompt.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the implementation**

```ts
// src/utils/auto-gen-prompt.ts
import type { TemplateSettings } from "types/auto-generation.types";

export const buildLtxI2vPrompt = (
  settings: TemplateSettings,
  startDreamUuid: string,
  endDreamUuid: string,
): Record<string, unknown> => {
  const params: Record<string, unknown> = {
    infinidream_algorithm: "ltx-i2v",
    prompt: settings.prompt,
    source_dream_uuid: startDreamUuid,
    end_source_uuid: endDreamUuid,
    duration: settings.duration,
  };
  const negative = settings.negativePrompt?.trim();
  if (negative) params.negative_prompt = negative;
  if (settings.highNoiseLoras?.length) params.high_noise_loras = settings.highNoiseLoras;
  if (Number.isFinite(settings.seed)) params.seed = settings.seed;
  if (Number.isFinite(settings.guidance)) params.guidance = settings.guidance; // GUIDANCE_PARAM["ltx-i2v"]
  return params;
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm jest src/utils/__tests__/auto-gen-prompt.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/utils/auto-gen-prompt.ts src/utils/__tests__/auto-gen-prompt.test.ts
git commit -m "feat(auto-gen): ltx-i2v prompt builder"
```

---

### Task 6: DB schema — `Dream.autoGenerated` + `AutoGenerationLog` entity + migration

**Files:**
- Modify: `src/entities/Dream.entity.ts` (add `autoGenerated` column)
- Create: `src/entities/AutoGenerationLog.entity.ts`
- Modify: `src/entities/index.ts` (register the new entity, following the existing export pattern)
- Create: `src/migrations/<timestamp>-AddAutoGeneration.ts`
- Test: `src/utils/__tests__/auto-generation-log.mapping.test.ts`

**Interfaces:**
- Produces: `AutoGenerationLog` entity with columns `id`, `startDreamUuid`, `endDreamUuid`, `templateRef`, `pairScore`, `dreamUuid`, `created_at`; a **unique index** on `(startDreamUuid, endDreamUuid, templateRef)`. `Dream.autoGenerated: boolean` (default false, indexed). Plus `toLogRow(entity): AutoGenLogRow` mapper used by the gateway.

- [ ] **Step 1: Add the `autoGenerated` column to Dream**

In `src/entities/Dream.entity.ts`, following the existing boolean-column style (e.g. the `hidden`/`nsfw` columns):

```ts
  @Column({ type: "boolean", default: false })
  @Index()
  autoGenerated: boolean;
```

- [ ] **Step 2: Create the `AutoGenerationLog` entity**

```ts
// src/entities/AutoGenerationLog.entity.ts
import {
  Column,
  CreateDateColumn,
  Entity,
  Index,
  PrimaryGeneratedColumn,
} from "typeorm";

@Entity()
@Index(["startDreamUuid", "endDreamUuid", "templateRef"], { unique: true })
export class AutoGenerationLog {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: "varchar" })
  startDreamUuid: string;

  @Column({ type: "varchar" })
  endDreamUuid: string;

  @Column({ type: "varchar" })
  templateRef: string;

  @Column({ type: "float" })
  pairScore: number;

  @Column({ type: "varchar" })
  dreamUuid: string;

  @CreateDateColumn()
  created_at: Date;
}
```

- [ ] **Step 3: Register the entity.** Runtime registration is automatic: `database/app-data-source.ts` loads entities via the glob `entities/**/*.{ts,js}`, so the new file is picked up without edits. Still add the `export *` line to `src/entities/index.ts` to match the convention (it's a re-export convenience, not the loader).

- [ ] **Step 4: Write the failing mapper test**

```ts
// src/utils/__tests__/auto-generation-log.mapping.test.ts
import { toLogRow } from "utils/auto-generation-log.mapper";
import { AutoGenerationLog } from "entities/AutoGenerationLog.entity";

it("maps an entity row to an AutoGenLogRow", () => {
  const e = new AutoGenerationLog();
  e.startDreamUuid = "a";
  e.endDreamUuid = "b";
  e.templateRef = "p#t";
  e.pairScore = 12.5;
  e.created_at = new Date("2026-10-01T00:00:00Z");
  expect(toLogRow(e)).toEqual({
    startDreamUuid: "a",
    endDreamUuid: "b",
    templateRef: "p#t",
    pairScore: 12.5,
    createdAt: new Date("2026-10-01T00:00:00Z"),
  });
});
```

- [ ] **Step 5: Run test to verify it fails**

Run: `pnpm jest src/utils/__tests__/auto-generation-log.mapping.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 6: Write the mapper**

```ts
// src/utils/auto-generation-log.mapper.ts
import type { AutoGenerationLog } from "entities/AutoGenerationLog.entity";
import type { AutoGenLogRow } from "types/auto-generation.types";

export const toLogRow = (e: AutoGenerationLog): AutoGenLogRow => ({
  startDreamUuid: e.startDreamUuid,
  endDreamUuid: e.endDreamUuid,
  templateRef: e.templateRef,
  pairScore: e.pairScore,
  createdAt: e.created_at,
});
```

- [ ] **Step 7: Hand-write the migration** (do NOT run it against staging). Follow the SQL style of a recent migration in `src/migrations/`. It must: add `dream.autoGenerated boolean not null default false` + its index; create the `auto_generation_log` table with the columns above + the unique index on `(startDreamUuid, endDreamUuid, templateRef)`. Include the matching `down()`.

- [ ] **Step 8: Run the mapper test to verify it passes + typecheck**

Run: `pnpm jest src/utils/__tests__/auto-generation-log.mapping.test.ts`
Run: `pnpm run build` (confirms entity + migration compile)
Expected: PASS; build succeeds.

- [ ] **Step 9: Commit**

```bash
git add src/entities/Dream.entity.ts src/entities/AutoGenerationLog.entity.ts src/entities/index.ts src/migrations/ src/utils/auto-generation-log.mapper.ts src/utils/__tests__/auto-generation-log.mapping.test.ts
git commit -m "feat(auto-gen): autoGenerated column + auto_generation_log table"
```

---

### Task 7: Orchestration `runAutoGeneration(deps, config, now)` with injected gateway (unit-tested)

**Files:**
- Create: `src/services/auto-generation.orchestrator.ts`
- Test: `src/services/__tests__/auto-generation.orchestrator.test.ts`

**Interfaces:**
- Consumes: all four pure fns (Tasks 2–5), `AutoGenConfig`, `TemplateSettings`, `ImageDreamSignals`, `Candidate`, `AutoGenLogRow`.
- Produces:
  - `interface AutoGenDeps { countInflightAutoJobs(): Promise<number>; loadTemplateSettings(projectUuid: string, transitionKey: string | null): Promise<TemplateSettings>; queryCandidateSignals(): Promise<ImageDreamSignals[]>; findPriorLogs(triples: Candidate[], templateRef: string): Promise<Map<string, AutoGenLogRow[]>>; generate(candidate: Candidate, prompt: Record<string, unknown>, templateRef: string): Promise<void>; }`
  - `interface AutoGenSummary { scored: number; paired: number; generated: number; skippedDedup: number; skippedGuard: boolean; templateRef: string; }`
  - `runAutoGeneration(deps: AutoGenDeps, config: AutoGenConfig, now: Date): Promise<AutoGenSummary>`
- Behavior: guard on inflight count; load template (throws propagate — fail fast); score → pair → for each candidate (highest pairScore first) check `shouldGenerate` using `findPriorLogs`, calling `deps.generate` until `maxJobsPerRun` reached; a `generate` rejection is caught, counted, and does not abort the run. `templateRef = \`${projectUuid}#${transitionKey ?? "default"}\``. Key for the prior-logs map: `\`${startDreamUuid}::${endDreamUuid}\``.

- [ ] **Step 1: Write the failing tests**

```ts
// src/services/__tests__/auto-generation.orchestrator.test.ts
import { runAutoGeneration } from "services/auto-generation.orchestrator";
import type { AutoGenConfig } from "types/auto-generation.types";

const NOW = new Date("2026-10-05T00:00:00Z");

const cfg = (over: Partial<AutoGenConfig> = {}): AutoGenConfig => ({
  weights: { votes: 1, playlist: 1, feature: 1, recency: 1 },
  halfLifeDays: 30, hubsPerRun: 2, spokesPerHub: 1, diversityRatio: 0,
  bothDirections: false, maxJobsPerRun: 10, cooldownDays: 30,
  regenScoreRisePct: 50, maxInflightAutoJobs: 100,
  templateProjectUuid: "proj", templateTransitionKey: "a::b", curationUserId: 1,
  ...over,
});

const template = { prompt: "p", negativePrompt: "", duration: 5, guidance: 3, seed: -1, highNoiseLoras: [] };
const sig = (uuid: string, netVotes: number) => ({
  uuid, ownerUuid: "o-" + uuid, netVotes, playlistCount: 0, featureRank: 0, updatedAt: NOW,
});

const makeDeps = (over: any = {}) => ({
  countInflightAutoJobs: async () => 0,
  loadTemplateSettings: async () => template,
  queryCandidateSignals: async () => [sig("a", 9), sig("b", 5), sig("c", 1)],
  findPriorLogs: async () => new Map(),
  generate: jest.fn(async () => {}),
  ...over,
});

describe("runAutoGeneration", () => {
  it("skips the run when too many auto jobs are in flight", async () => {
    const deps = makeDeps({ countInflightAutoJobs: async () => 999 });
    const s = await runAutoGeneration(deps, cfg(), NOW);
    expect(s.skippedGuard).toBe(true);
    expect(s.generated).toBe(0);
    expect(deps.generate).not.toHaveBeenCalled();
  });

  it("generates for fresh candidates up to maxJobsPerRun", async () => {
    const deps = makeDeps();
    const s = await runAutoGeneration(deps, cfg({ maxJobsPerRun: 1 }), NOW);
    expect(s.generated).toBe(1);
    expect(deps.generate).toHaveBeenCalledTimes(1);
  });

  it("handles an empty candidate set without throwing", async () => {
    const deps = makeDeps({ queryCandidateSignals: async () => [] });
    const s = await runAutoGeneration(deps, cfg(), NOW);
    expect(s).toMatchObject({ scored: 0, paired: 0, generated: 0 });
  });

  it("skips candidates the dedup predicate rejects", async () => {
    const recent = [{ startDreamUuid: "a", endDreamUuid: "b", templateRef: "proj#a::b", pairScore: 999, createdAt: NOW }];
    const deps = makeDeps({
      findPriorLogs: async () => new Map([["a::b", recent]]),
    });
    const s = await runAutoGeneration(deps, cfg(), NOW);
    // a->b is suppressed; other pairs still allowed
    expect(deps.generate).not.toHaveBeenCalledWith(
      expect.objectContaining({ startDreamUuid: "a", endDreamUuid: "b" }),
      expect.anything(), expect.anything(),
    );
    expect(s.skippedDedup).toBeGreaterThanOrEqual(1);
  });

  it("continues after a generate() failure and counts it", async () => {
    const deps = makeDeps({ generate: jest.fn(async () => { throw new Error("enqueue boom"); }) });
    const s = await runAutoGeneration(deps, cfg(), NOW);
    expect(s.generated).toBe(0);
    expect(deps.generate).toHaveBeenCalled(); // attempted, did not abort
  });

  it("fails fast if the template cannot be loaded", async () => {
    const deps = makeDeps({ loadTemplateSettings: async () => { throw new Error("no project"); } });
    await expect(runAutoGeneration(deps, cfg(), NOW)).rejects.toThrow("no project");
    expect(deps.generate).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm jest src/services/__tests__/auto-generation.orchestrator.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Write the implementation**

```ts
// src/services/auto-generation.orchestrator.ts
import { scoreImageDreams } from "utils/keyframe-scoring";
import { buildCandidates } from "utils/candidate-pairing";
import { shouldGenerate } from "utils/auto-generation-dedup";
import { buildLtxI2vPrompt } from "utils/auto-gen-prompt";
import type {
  AutoGenConfig,
  AutoGenLogRow,
  Candidate,
  ImageDreamSignals,
  TemplateSettings,
} from "types/auto-generation.types";

export interface AutoGenDeps {
  countInflightAutoJobs(): Promise<number>;
  loadTemplateSettings(projectUuid: string, transitionKey: string | null): Promise<TemplateSettings>;
  queryCandidateSignals(): Promise<ImageDreamSignals[]>;
  findPriorLogs(triples: Candidate[], templateRef: string): Promise<Map<string, AutoGenLogRow[]>>;
  generate(candidate: Candidate, prompt: Record<string, unknown>, templateRef: string): Promise<void>;
}

export interface AutoGenSummary {
  scored: number;
  paired: number;
  generated: number;
  skippedDedup: number;
  skippedGuard: boolean;
  templateRef: string;
}

const pairKey = (c: Candidate) => `${c.startDreamUuid}::${c.endDreamUuid}`;

export const runAutoGeneration = async (
  deps: AutoGenDeps,
  config: AutoGenConfig,
  now: Date,
): Promise<AutoGenSummary> => {
  const templateRef = `${config.templateProjectUuid}#${config.templateTransitionKey ?? "default"}`;
  const base: AutoGenSummary = {
    scored: 0, paired: 0, generated: 0, skippedDedup: 0, skippedGuard: false, templateRef,
  };

  const inflight = await deps.countInflightAutoJobs();
  if (inflight > config.maxInflightAutoJobs) {
    return { ...base, skippedGuard: true };
  }

  // Fail fast before any writes if the template is unavailable.
  const settings = await deps.loadTemplateSettings(
    config.templateProjectUuid,
    config.templateTransitionKey,
  );

  const signals = await deps.queryCandidateSignals();
  const scored = scoreImageDreams(signals, config.weights, config.halfLifeDays, now);
  const candidates = buildCandidates(scored, config);

  const priorByPair = await deps.findPriorLogs(candidates, templateRef);

  let generated = 0;
  let skippedDedup = 0;
  for (const candidate of candidates) {
    if (generated >= config.maxJobsPerRun) break;
    const prior = priorByPair.get(pairKey(candidate)) ?? [];
    if (!shouldGenerate(candidate, prior, config, now)) {
      skippedDedup++;
      continue;
    }
    const prompt = buildLtxI2vPrompt(settings, candidate.startDreamUuid, candidate.endDreamUuid);
    try {
      await deps.generate(candidate, prompt, templateRef);
      generated++;
    } catch {
      // Enqueue/create failed for this candidate; the gateway marks the Dream
      // failed. Continue with the rest of the run.
    }
  }

  return {
    ...base,
    scored: scored.length,
    paired: candidates.length,
    generated,
    skippedDedup,
  };
};
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `pnpm jest src/services/__tests__/auto-generation.orchestrator.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/services/auto-generation.orchestrator.ts src/services/__tests__/auto-generation.orchestrator.test.ts
git commit -m "feat(auto-gen): orchestration with injected gateway"
```

---

### Task 8: Real DB gateway + `createAutoGeneratedDream` (queue-after-commit)

**Files:**
- Create: `src/services/auto-generation.gateway.ts`
- Test: `src/services/__tests__/auto-generation.gateway.test.ts`

**Interfaces:**
- Consumes: `AutoGenDeps` (Task 7), entities, `toLogRow` (Task 6), `processDreamRequest` (`utils/dream.util`), the app `DataSource`.
- Produces: `createDbGateway(dataSource: DataSource, config: AutoGenConfig): AutoGenDeps` implementing every method against TypeORM.
- **`generate` ordering (critical):** within one transaction, create the `Dream` (`user`=curation user — and set `displayedOwner` to the curation user too, so the feed attributes it to the system account; `status`=QUEUE, `mediaType`=VIDEO, generated `name`, `autoGenerated=true`, `prompt`=JSON) and insert the `AutoGenerationLog` row (catch unique-violation → treat as already-done, abort this candidate quietly); **commit**; only then call `processDreamRequest(dream)`. If `processDreamRequest` throws, mark the Dream failed and rethrow (the orchestrator counts it).
- **`loadTemplateSettings`:** load `EditorProject` by uuid; throw if missing. Resolve the transition key — if `null`, build the default key from `state.referenceFrames[0].id` + `[1].id`; read `state.transitions[key].settings`; throw if missing/malformed. Map to `TemplateSettings` (model is ignored — always ltx-i2v downstream). `highNoiseLoras` defaults to `[]`.
- **`queryCandidateSignals`:** query image Dreams (`mediaType = DreamMediaType.IMAGE`, status `PROCESSED`) excluding `nsfw`, `hidden`, and any that have a `Report`. **Owner-active filter:** there is **no** `disabled`/`banned` column on `User`; the only inactive-account concept is soft-deletion (`deleted_at`). **Do NOT rely on TypeORM auto-hiding soft-deleted rows** — in this codebase's `createQueryBuilder` queries the `deleted_at IS NULL` predicate is written **explicitly**, even for root entities (see `dream.util.ts` ~`Dream.deleted_at IS NULL`, `playlist.util.ts` ~`item.deleted_at IS NULL`), and it is **never auto-applied to a joined relation**. So you MUST add an explicit `.andWhere("<userAlias>.deleted_at IS NULL")` on the joined `User` alias. (The semantics to mirror: `deleted_at IS NOT NULL` ⇒ inactive — the same rule `utils/account-status.util.ts: isAccountDeleted` encodes. Do **not** import/call `isAccountDeleted`; it takes `(email, workOSId)` and runs its own per-user query, unusable in a set query.) Project to `ImageDreamSignals`: `netVotes = upvotes - downvotes`, `playlistCount` = joined count of `playlistItems`, `featureRank`, `updatedAt`, and `ownerUuid`. The **one** `User` join serves both purposes — selecting `user.uuid` for `ownerUuid` (the FK only carries `userId`) **and** the `deleted_at IS NULL` filter above. Use a **`createQueryBuilder`** with a `leftJoin` + `COUNT`/`loadRelationCountAndMap` for the playlist count and a `Report` anti-join — copy the query-builder style from `utils/dream.util.ts` or `utils/playlist.util.ts` (both use `createQueryBuilder`; **note `utils/feed.util.ts` does NOT — do not copy it**, it only builds `FindOptionsWhere` objects and cannot express the aggregate count).
- **`countInflightAutoJobs`:** `count` of Dreams with `autoGenerated = true` and status in `{QUEUE, PROCESSING}` (use the real `DreamStatusType` enum values on stage).
- **`findPriorLogs`:** one query over `AutoGenerationLog` filtered to `templateRef` and the candidate pairs; group into the `Map<pairKey, AutoGenLogRow[]>` via `toLogRow`.

> **Testing note:** this gateway is the thin I/O layer. Test `generate`'s ordering guarantee and `loadTemplateSettings`'s parsing with **mocked** TypeORM repositories / a mocked `QueryRunner` and a stubbed `processDreamRequest` (jest mock) — do not stand up a real DB. The behavioral logic is already covered by Tasks 2–7; here we pin the two hazards the orchestrator can't see.

- [ ] **Step 1: Write the failing tests** (mock `DataSource.createQueryRunner`, repositories, and `processDreamRequest`)

```ts
// src/services/__tests__/auto-generation.gateway.test.ts
import { createDbGateway } from "services/auto-generation.gateway";
import * as dreamUtil from "utils/dream.util";

jest.mock("utils/dream.util");

const makeQueryRunner = () => {
  const saved: any[] = [];
  return {
    saved,
    connect: jest.fn(async () => {}),
    startTransaction: jest.fn(async () => {}),
    commitTransaction: jest.fn(async () => {}),
    rollbackTransaction: jest.fn(async () => {}),
    release: jest.fn(async () => {}),
    manager: { save: jest.fn(async (e: any) => { saved.push(e); return e; }) },
  };
};

const baseConfig: any = {
  templateProjectUuid: "proj", templateTransitionKey: "a::b", curationUserId: 7,
};

describe("createDbGateway.generate", () => {
  it("commits the dream+log BEFORE enqueuing (no enqueue inside the txn)", async () => {
    const qr = makeQueryRunner();
    const order: string[] = [];
    qr.commitTransaction = jest.fn(async () => { order.push("commit"); });
    (dreamUtil.processDreamRequest as jest.Mock).mockImplementation(async () => { order.push("enqueue"); });

    const ds: any = {
      createQueryRunner: () => qr,
      getRepository: () => ({ findOne: async () => ({ id: 7 }) }), // curation user
    };
    const gw = createDbGateway(ds, baseConfig);
    await gw.generate(
      { startDreamUuid: "a", endDreamUuid: "b", pairScore: 10 },
      { infinidream_algorithm: "ltx-i2v" },
      "proj#a::b",
    );
    expect(order).toEqual(["commit", "enqueue"]);
  });

  it("marks the dream failed and rethrows if enqueue fails after commit", async () => {
    const qr = makeQueryRunner();
    (dreamUtil.processDreamRequest as jest.Mock).mockRejectedValue(new Error("boom"));
    const markFailed = jest.fn(async () => {});
    const ds: any = {
      createQueryRunner: () => qr,
      getRepository: () => ({ findOne: async () => ({ id: 7 }), update: markFailed }),
    };
    const gw = createDbGateway(ds, baseConfig);
    await expect(
      gw.generate({ startDreamUuid: "a", endDreamUuid: "b", pairScore: 10 }, { infinidream_algorithm: "ltx-i2v" }, "proj#a::b"),
    ).rejects.toThrow("boom");
    expect(markFailed).toHaveBeenCalled();
  });

  it("treats a unique-violation on the log insert as already-done (no enqueue, no throw)", async () => {
    const qr = makeQueryRunner();
    qr.manager.save = jest.fn(async (e: any) => {
      if (e?.constructor?.name === "AutoGenerationLog" || e?.templateRef) {
        const err: any = new Error("dup"); err.code = "23505"; throw err;
      }
      return e;
    });
    (dreamUtil.processDreamRequest as jest.Mock).mockClear();
    const ds: any = {
      createQueryRunner: () => qr,
      getRepository: () => ({ findOne: async () => ({ id: 7 }) }),
    };
    const gw = createDbGateway(ds, baseConfig);
    await gw.generate({ startDreamUuid: "a", endDreamUuid: "b", pairScore: 10 }, { infinidream_algorithm: "ltx-i2v" }, "proj#a::b");
    expect(dreamUtil.processDreamRequest as jest.Mock).not.toHaveBeenCalled();
    expect(qr.rollbackTransaction).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `pnpm jest src/services/__tests__/auto-generation.gateway.test.ts`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement the gateway.** Write `createDbGateway` so each method matches the Interfaces block above. For `generate`: use `dataSource.createQueryRunner()`, `startTransaction`, `manager.save(dream)` then `manager.save(logRow)` (catch Postgres `23505` → `rollbackTransaction`, return without enqueue), `commitTransaction`, `release`; then `await processDreamRequest(dream)` in a try/catch that `update`s the Dream to a failed status and rethrows. For `queryCandidateSignals`/`countInflightAutoJobs`/`findPriorLogs`/`loadTemplateSettings`, use `dataSource.getRepository(...)` query builders following `utils/feed.util.ts` conventions. Use the real `DreamStatusType`, `DreamMediaType`, and entity names from stage.

- [ ] **Step 4: Run tests to verify they pass + typecheck**

Run: `pnpm jest src/services/__tests__/auto-generation.gateway.test.ts`
Run: `pnpm run build`
Expected: PASS; build succeeds.

- [ ] **Step 5: Commit**

```bash
git add src/services/auto-generation.gateway.ts src/services/__tests__/auto-generation.gateway.test.ts
git commit -m "feat(auto-gen): DB gateway with queue-after-commit ordering"
```

---

### Task 9: Script entrypoint + `cron:auto-generate` + run-summary logging

**Files:**
- Create: `src/script/auto-generate.ts`
- Modify: `package.json` (add `"cron:auto-generate"` script, mirroring `cron:default-playlist`)
- Modify: `AGENTS.md`/deploy docs — note the Heroku Scheduler entry (brief)

**Interfaces:**
- Consumes: `loadAutoGenConfig` (Task 1), `createDbGateway` (Task 8), `runAutoGeneration` (Task 7), the app `DataSource` initializer used by `src/script/default-playlist.ts`.
- Produces: a runnable script that initializes the DataSource, builds config from `process.env`, runs one pass with `now = new Date()`, logs the summary, and exits non-zero on fatal error (e.g. template load failure).

- [ ] **Step 1: Write the script** (mirror `src/script/default-playlist.ts` for DataSource init/teardown)

```ts
// src/script/auto-generate.ts
import appDataSource from "database/app-data-source"; // default export, lowercase — same as default-playlist.ts
import { APP_LOGGER } from "shared/logger";
import { loadAutoGenConfig } from "constants/auto-generation.constants";
import { createDbGateway } from "services/auto-generation.gateway";
import { runAutoGeneration } from "services/auto-generation.orchestrator";

const main = async (): Promise<void> => {
  await appDataSource.initialize();
  const config = loadAutoGenConfig(process.env);
  if (!config.templateProjectUuid || !config.curationUserId) {
    throw new Error("AUTO_GEN_TEMPLATE_PROJECT_UUID and CURATION_USER_ID are required");
  }
  const deps = createDbGateway(appDataSource, config);
  const summary = await runAutoGeneration(deps, config, new Date());
  APP_LOGGER.info({ summary }, "auto-generate run complete");
};

main()
  .then(() => process.exit(0))
  .catch((err) => {
    APP_LOGGER.error({ err }, "auto-generate failed");
    process.exit(1);
  });
```

> Verify the exact `appDataSource` default-export path and `APP_LOGGER` path against `src/script/default-playlist.ts` on stage before relying on them. `default-playlist.ts` does **not** call `.destroy()`; mirror that (process exit tears the pool down). Pass `appDataSource` (the initialized DataSource) into `createDbGateway`.

- [ ] **Step 2: Add the npm script** to `package.json`, next to `cron:default-playlist`:

```json
"cron:auto-generate": "NODE_PATH=dist node dist/script/auto-generate.js"
```

- [ ] **Step 3: Build and smoke-run against an empty/guarded config**

Run: `pnpm run build`
Run: `AUTO_GEN_TEMPLATE_PROJECT_UUID= CURATION_USER_ID= pnpm run cron:auto-generate` (expect it to exit non-zero with the "required" error — proves wiring without touching data)
Expected: build succeeds; script exits 1 with the clear config error.

- [ ] **Step 4: Document the Heroku Scheduler entry** — add one line to the deploy notes: run `pnpm run cron:auto-generate` nightly (UTC). Note the first run with an empty log generates up to `MAX_JOBS_PER_RUN`. **Also set `LOGGING=true` in the scheduler env** — `APP_LOGGER` (pino) is disabled unless `LOGGING` is truthy (`shared/logger.ts`), so without it the run summary is silently dropped and an operator sees nothing.

- [ ] **Step 5: Commit**

```bash
git add src/script/auto-generate.ts package.json AGENTS.md
git commit -m "feat(auto-gen): cron:auto-generate script + scheduler docs"
```

---

### Task 10: System/curation account prerequisite + config wiring

**Files:**
- Create: `src/script/ensure-curation-user.ts` (idempotent seed) OR a migration — follow whichever pattern stage uses for seeding system data (check `src/script/` and `src/migrations/`)
- Modify: deploy docs — document `CURATION_USER_ID` and account provisioning
- Test: `src/script/__tests__/ensure-curation-user.test.ts` (idempotency of the pure selection, if logic warrants)

**Interfaces:**
- Produces: a documented, repeatable way to create/identify the system account whose id goes in `CURATION_USER_ID`, with its WorkOS identity and license flags decided.

- [ ] **Step 1: Decide + document the account + template** — in the deploy notes, record: how the curation user is created (seed script vs manual admin), its WorkOS identity (or that it is a local-only non-login account), and the license flags to set on its Dreams (e.g. `ccbyLicense`). Credit accounting is moot for ltx-i2v (RunPod, no pricing) — note this. **Also document that the designated auto-gen template transition (`AUTO_GEN_TEMPLATE_PROJECT_UUID` / `AUTO_GEN_TEMPLATE_TRANSITION_KEY`) must use LTX-appropriate LoRAs or none** (its stored model defaults to `kling-25-i2v`; its LoRAs pass through unchecked — see Task 5 LoRA-fit note).

- [ ] **Step 2: Write the idempotent ensure-script** (if a seed pattern exists): look up the curation user by a stable email/handle; create if absent; print its id. Keep the lookup logic pure where possible and unit-test idempotency (same input → same selected id, no duplicate create).

- [ ] **Step 3: Run the test**

Run: `pnpm jest src/script/__tests__/ensure-curation-user.test.ts`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/script/ensure-curation-user.ts src/script/__tests__/ensure-curation-user.test.ts AGENTS.md
git commit -m "feat(auto-gen): curation-account provisioning + docs"
```

---

## Self-Review

**Spec coverage:** scoring (T2), pairing/hub-and-spoke (T3), dedup cooldown+score-rise (T4), ltx-i2v template prompt from a flow EditorProject transition (T5+T8), `autoGenerated` + `auto_generation_log` + unique constraint (T6), orchestration with caps + queue-pressure guard + run summary (T7), queue-after-commit + candidate query + safety gate (T8), nightly script + Heroku Scheduler (T9), system account (T10), config/env (T1). Deferred items (plays, usage-as-source, curated template set, retry) remain out of scope per spec.

**Placeholders:** pure-function tasks carry full test + impl code. Two tasks legitimately defer to stage-specific symbols (T8 query-builder details, T9 DataSource/logger imports) because those names must be read from fresh `origin/stage` — each says exactly which existing file to copy the pattern from, which is a grounded instruction, not a TBD.

**Type consistency:** `ImageDreamSignals`, `ScoredImageDream`, `Candidate`, `TemplateSettings`, `AutoGenLogRow`, `AutoGenConfig`, `AutoGenDeps`, `AutoGenSummary` are defined once (T1/T7) and consumed with the same shapes throughout; `scoreImageDreams`/`buildCandidates`/`shouldGenerate`/`buildLtxI2vPrompt` signatures match their call sites in T7.

**Review Focus:** empty/tiny set (T3, T7), duplicate-triple race (T6 unique index, T8 23505 handling), enqueue-failure-after-commit (T7, T8), missing/malformed template (T7 fail-fast test, T8 loader), all-same-owner diversity fallback (T3) — each pinned to an owning task's tests.
