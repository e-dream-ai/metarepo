# Deforum LoRAs: ingest to R2, drop the rebuild

Plan for e-dream-ai/backend#479, answering the 2026-09-02 comment: *"forget about
interpolation for now, let's just get loading working... best to ingest and host
ourselves, like a video asset in R2. can that simplify this?"*

## Short answer: yes, and it collapses the two "next steps" into one

The comment framed two possible next steps — (a) fix the container rebuild, and
(b) make LoRAs user-settable/uploadable. R2 hosting is not a third option; it is
the one change that does both, and it is *less* new machinery than the
RunPod-volume-plus-S3-API sync plan from 2026-08-26.

Why it is simpler than syncing a manifest onto the RunPod volume:

- **The container already speaks R2.** `gpu-container-deforum/src/handler.py`
  builds a boto3 client from `R2_BUCKET_NAME` / `R2_ENDPOINT_URL` /
  `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` to upload every finished render.
  Reading a LoRA back out of the same bucket is a `get_object` call, not a new
  integration.
- **No sync service, no volume at all.** The volume plan needs an admin CLI or UI
  that writes to RunPod's S3 endpoint, a manifest kept coherent with what is
  actually on disk, and a volume that pins the endpoint to one datacenter. With
  R2 as the source of truth the worker just downloads what the job asks for into
  its own container disk. Nothing is lost, nothing to reconcile, no RunPod-side
  state to administer. See "Why not cache on the RunPod volume" below.
- **It matches an existing pattern.** `wan-i2v-lora` already ships LoRAs to the
  GPU as `loras: [{path, scale}]` (`worker/src/workers/job-handlers.ts:822`).
  The difference is that those paths point at
  `huggingface.co/ostris/wan22_i2v_14b_*` — hardcoded in
  `frontend/src/components/pages/studio/constants/action-presets.ts:5`. That is
  exactly the "what if the links go down" exposure, already live in the Studio
  camera presets. The same registry fixes it.
- **Reproducibility falls out of content addressing.** Store each file at
  `loras/<sha256>.safetensors`. The civitai URL becomes provenance metadata, not
  a runtime dependency, and a dream's render record can pin the exact digest it
  used.

**Nothing in the prompt syntax changes.** `<lora:alias:strength>` keeps working
as it does today. `LoraRegistry` (`deforum-studio/src/deforum/utils/lora.py:31`)
re-scans `LORA_PATH` on *every* render — `prepare_lora_prompts` constructs a
fresh registry per call — so a file dropped into that directory at job start is
picked up with no restart and no deforum-studio change.

## Current state (verified against `origin/main`, 2026-09-04)

| Piece | Where | Behavior |
| --- | --- | --- |
| LoRA files | `gpu-container-deforum/Dockerfile-build:50` | Downloaded at **image build** by `scripts/download_loras.py` from `loras/manifest.json` into `/deforum_storage/models/loras/<alias>.safetensors` |
| Manifest | `gpu-container-deforum/loras/manifest.json` | 7 entries: alias, civitai URL, sha256 |
| Lookup dir | `ENV LORA_PATH=/deforum_storage/models/loras` | Read once at import by `deforum/utils/constants.py:61`; contents re-scanned per render |
| Tag parsing | `deforum/utils/lora.py:parse_lora_prompt` | Strips `<lora:alias:strength>` from each keyframe prompt, resolves alias → file by filename stem |
| Job payload | `worker/src/workers/job-handlers.ts:313` (`handleDeforumVideoJob`) | Splits numeric keys into `prompts`, passes everything else through as `settings` verbatim |
| Queue dispatch | `backend/src/utils/dream.util.ts:136` | `jobData = {...promptJson, dream_uuid, ...}` → `deforumvideo` queue |
| Input schema | `gpu-container-deforum/src/rp_schema.py` | `{settings: dict}` only |

So the *only* thing standing between a user and a LoRA is the file's presence in
`LORA_PATH` — which today requires a Docker build.

## Phase 0 — verify and unblock (half a day)

Three things to check before writing much code: two measurements, and one
hypothesis to confirm or discard.

1. **Check ephemeral disk headroom on a deforum worker.** `predict.py:15` already
   logs total/used/free at init; read it off a recent job. The LoRA cache is
   container-local (see below), so we need room for a handful of ~218 MB files on
   top of an already-large image. This sets the per-job LoRA cap and the cache
   high-water mark.

2. **Measure a cold LoRA download.** The bundled SDXL LoRAs are 218 MiB each
   (`content-length: 228453340` for `ral-frctlgmtry-sdxl.safetensors`). R2 →
   RunPod should be a second or two, with no egress cost, but time it once
   against the render — a 2400-frame job at 1344x768 runs for minutes, and the
   cold start already loads a ~6.5 GB checkpoint, so even 30 s is noise.

3. **Assess the warm-worker LoRA-set question — do not fix it yet.** This one is
   a code-reading hypothesis, not an observed failure, and it needs confirming
   before anyone spends time on it. `configure_loras`
   (`comfy_sd_generator.py:92`) raises `RuntimeError("LoRAs cannot change after
   generation has started")` when `model_loaded` is `True` and the LoRA set
   differs. On the normal path that looks harmless: `run_post_fn_list` →
   `cleanup()` sets `model_loaded=False` between jobs, so the next job
   reconfigures freely. The open question is whether a job that errors or is
   interrupted mid-render can leave a warm worker with `model_loaded=True` —
   `REFRESH_WORKER` defaults to `false` (`handler.py:16`) and
   `models["deforum_pipe"]` is a module-level cache — and whether RunPod recycles
   a worker after a failed job anyway, which would make the whole thing moot.
   Confirm empirically: on one warm worker, run a job using LoRA A, fail it
   mid-render, then submit a job using LoRA B, and look for the RuntimeError in
   the worker log. If it does not reproduce, drop this item. If it does, the fix
   is cheap — reset generator state at the start of `Predictor.predict`, or wrap
   the pipeline call in `try/finally: self.pipe.cleanup()`.

   Either way this does not gate Phase 1, which changes only where the files come
   from.


## Phase 1 — kill the rebuild (container + worker)

Goal: adding a LoRA stops requiring an image build. No user-facing change yet.

**Container** (`gpu-container-deforum`):

1. `src/rp_schema.py` — add an optional `loras` field:
   `{"type": list, "required": False, "default": []}`, entries
   `{alias, key | url, sha256}`. (Extra top-level keys are rejected by
   `runpod.serverless.utils.rp_validator.validate`, so this must be declared.)
2. New `src/lora_fetch.py`:
   - Cache dir: `/deforum_storage/models/lora-cache`, i.e. container-local,
     overridable with a `LORA_CACHE_DIR` env var (the escape hatch if a shared
     volume ever turns out to be worth it). It persists for the life of the
     *worker*, not the job, so a warm worker downloads a given LoRA once across
     every render it serves.
   - For each requested entry: if `<cache>/<sha256>.safetensors` is absent,
     `get_object` it from R2 (or GET the URL) to a `.part` file, verify SHA-256,
     `os.replace` into place. Reuse the download/verify logic already written in
     `scripts/download_loras.py` — same shape, different source.
   - Materialize `LORA_PATH/<alias>.safetensors` as a symlink to the cached file.
     `LoraRegistry` keys on the filename stem and **raises on duplicate aliases in
     one directory**, so clear previously-materialized symlinks (not the baked
     files) at the start of every job.
   - Download missing files concurrently; skip everything already cached.
   - Bound the cache: cap LoRAs per job, and sweep least-recently-used entries
     once the directory exceeds a size ceiling, so a long-lived worker serving
     many users cannot fill its disk.
3. `src/handler.py` — call the fetcher after validation, before
   `generate_video.predict(...)`. Surface a clear error if a file is missing or
   its digest mismatches, so the job fails fast with a legible message.
4. Keep `loras/manifest.json` and the build-time install as the **built-in set**
   (fallback plus offline dev). The manifest stops being the mechanism for adding
   LoRAs; it becomes the default catalog.

**Worker** (`worker/src/workers/job-handlers.ts`, `handleDeforumVideoJob`):

Pull `loras` out of `promptData` alongside `dream_uuid` / `auto_upload` so it
does not leak into `settings`, and pass it as a sibling of `settings` in the
RunPod input:

```ts
const { dream_uuid, auto_upload = true, loras, ...promptData } = job.data;
// ...
{ input: { settings: {...}, ...(loras?.length ? { loras } : {}) } }
```

**Seeding**: upload the 7 current files to `loras/<sha256>.safetensors` in R2
(they are already on disk in any built image, and `manifest.json` carries the
digests, so this is a one-off script). Verify a render against a
manifest-declared alias with the file *removed* from the image.

**Exit criterion**: a new LoRA is usable by uploading one file to R2 and adding
one registry row. No Docker build, no RunPod release.

### Why not cache on the RunPod volume

An earlier draft cached to `/runpod-volume/loras`. Container-local is better:

- **Container disk already persists across jobs.** It lives as long as the
  worker, not the job. A warm worker handling ten renders downloads a LoRA once
  either way; the volume only saves the *first* job on each *new* worker.
- **A network volume pins the endpoint to a single datacenter**, shrinking the
  GPU pool it can schedule on. Free only if one is already attached for the
  checkpoint; a real cost if we would attach one for this.
- **Shared storage means concurrent writers on one path.** Two workers pulling
  the same file need per-process temp names to avoid clobbering each other's
  partial download, and they duplicate the transfer regardless.
- **User uploads grow without bound.** On a fixed-size volume that eventually
  needs an eviction policy and an owner; container disk evaporates by itself.
- It removes a blocking unknown from Phase 0 — the design no longer cares whether
  a volume exists.

At 218 MiB per file over zero-egress R2, the download is small next to a cold
start that loads a ~6.5 GB checkpoint. `LORA_CACHE_DIR` keeps the volume
available as a one-line change if measurement ever contradicts this.


## Phase 2 — LoRAs as a third media type (backend)

An uploaded LoRA is a **Dream**, not a new top-level entity. `DreamMediaType` is
`video | image` today (`backend/src/types/dream.types.ts:124`); add
`LORA = "lora"`. Everything a dream already has then applies for free: an owner,
`nsfw` / `hidden` / `ccbyLicense`, votes, a thumbnail, feed presence — the feed
already filters on `mediaType` (`backend/src/utils/feed.util.ts:142`), so a LoRA
tab is a query parameter rather than a new surface — plus the multipart R2 upload
flow and the `queue → processing → processed` status machine.

It also matches how generation already refers to assets: `wan-i2v` and `ltx-i2v`
take **Dream UUIDs** for their source images. A LoRA that is a Dream is a thing
the worker can already resolve, which is exactly the lesson in this repo's
"Before choosing an upload strategy" table — upload must produce the entity type
the downstream consumer understands.

**Fields.** Most of what a LoRA needs is already on `Dream` (`name`,
`description`, `sourceUrl`, `user`, `thumbnail`). What is missing:

| Field | Notes |
| --- | --- |
| `alias` | prompt-facing handle, `^[A-Za-z0-9._-]+$`. `name` is free text truncated to 1000 chars (`dream.controller.ts:173`) and cannot serve as the `<lora:...>` key. Admin-minted — see "The alias namespace" below |
| `sha256` | content address; the R2 object is `loras/<sha256>.safetensors`, so identical uploads dedupe |
| `baseModel` | `sdxl` / `sd15` — deforum runs ProtovisionXL, and an SD1.5 LoRA silently produces garbage |
| `triggerWords` | nobody guesses `ral-frctlgmtry` unaided |

Four columns is small enough to hang off `Dream` as nullables, or to put in a
`lora_metadata` side table keyed by dream id — the side table keeps an already
wide entity from growing four mostly-null columns.

### The alias namespace: one field, claim on upload

`<lora:name:strength>` has to resolve to exactly one LoRA, globally. That is a
nullable `loraAlias` column on `Dream` with a partial unique index
(`WHERE lora_alias IS NOT NULL`) — no registry table, no lifecycle states.

- **Uploading claims the name, if it is free.** Slugify the dream's name, or take
  an explicit alias from the uploader. Free → theirs immediately, no review, no
  waiting on anyone.
- **A taken name is never reassigned by another upload.** The second uploader is
  told it is taken and picks another. Nothing is blocked meanwhile (below).
- **Admins can write the column on any dream.** That covers every case: clear a
  bad name, correct a mistaken claim, or block a name by pointing it at a safe
  LoRA of their own. Squatting on a name with a known-good asset is simpler than
  retiring it, and leaves the alias resolving to something real.
- **A LoRA is usable before it has a name at all.** No syntax change needed:
  `<lora:3f2a9c14-8b7e-4d51-9a02-6c1e5f8b7d33:0.8>` already parses against
  `_LORA_TAG` (`lora.py:10`, verified), and a UUID is a legal filename stem, so
  the container materializes it like any other alias. That is the fallback
  whenever a name is taken or withheld.

**Resolution order** at submit time: exact match on `loraAlias` → else parse the
token as a UUID and look up that dream (the requester must own it, or it must be
public) → else fail with the list of available aliases. Forbid claiming an alias
that parses as a UUID, so the two namespaces cannot cross.

**No audit table.** The column is self-describing: an alias belongs to a dream,
and the dream already carries its owner and timestamps, so "who claimed this
name" is answerable without any extra bookkeeping.

**Claiming is a database race, not an application check.** Two uploads can request
the same free name at the same instant, so let the unique index arbitrate:
attempt the write, catch the violation, treat it as taken. A read-then-write
"is it free?" check is a TOCTOU bug that will hand one alias to two dreams.

**Seed the built-ins as already-claimed.** The seven aliases in
`loras/manifest.json` (`ral-frctlgmtry`, `trichom-style`, `chrome-style`, …) get
claimed in the same migration that adds the column, so no creator can take a name
the existing dreams already reference.

**Re-pointing an alias changes what an old stored prompt means.** Worth knowing,
not worth building for: deleting any asset already does this to videos and
images, so LoRAs are not a new class of problem and can be solved with them
later. Recording the resolved digest on the render is still worth doing because
it is free — we compute the SHA-256 during ingest anyway — but nothing here
depends on it.

**The alias never reaches the model.** `parse_lora_prompt` strips the whole tag
before the text reaches the encoder, so an alias is a lookup key, not a trigger
word — renaming one is semantically free, and a trigger word like
`ral-frctlgmtry` still has to appear in the prompt text on its own to do
anything. The picker in Phase 3 should insert both.

**One optional guard rail**, needing no admin in the loop: cap alias claims per
user per day, so a script cannot grab the good names in an afternoon.


**The exclusion-filter trap — read this before adding the enum value.**
Native-client and playlist queries are written as *exclusions*, not inclusions:
`mediaType: Not(DreamMediaType.IMAGE)` at `client.controller.ts:229`, `:278`,
`:334`, and `client.util.ts:97`. A third enum value therefore **joins native
client playback silently** — the macOS screensaver would queue up a 218 MB
safetensors file as if it were a dream to play. Every one of those must become an
explicit `In([VIDEO])` (or `Not(In([IMAGE, LORA]))`) in the same PR that adds the
value. Audit `playlist.util.ts` (`:251`, `:370`, `:622`) and the frontend feed and
grid for anything else that treats "not image" as "video".

**Ingest.** Two paths, both landing at `loras/<sha256>.safetensors`:

- *Direct upload* — reuse the dream multipart flow unchanged
  (`dream.routes.ts:153-434`, `utils/r2.util.ts`); the browser PUTs parts
  straight to R2. Not `multerSingleFileMiddleware` — memory storage, 25 MB cap,
  image extensions only (`multer.middleware.ts:11`).
- *Ingest from URL* — a queued job downloads server-side, hashes, and stores.
  This is the civitai answer: paste the link once, we hold the bytes, and where
  it came from stops being a runtime dependency.

Add `safetensors` to the extension allowlists, and note that
`detectMediaTypeFromExtension` (`utils/media.util.ts`) **defaults unknown
extensions to VIDEO** — without a mapping, a `.safetensors` upload lands as a
broken video dream.

**Validation before flipping to `processed`:**

- Extension `.safetensors` **and** a header check: safetensors is an 8-byte
  little-endian header length followed by that many bytes of UTF-8 JSON. Parse
  it; reject anything else. This is what makes "they're just safetensors, so
  they're safe" actually true — a pickle renamed to `.safetensors` is not.
- Size cap (1 GB is generous; the bundled ones are 218 MiB).
- Compute SHA-256 server-side; never trust a client-supplied digest.
- Optionally read tensor key prefixes from that JSON header to infer SDXL vs
  SD1.5 rather than asking the uploader.

**No video ingest for LoRA dreams.** There are no frames, so the `videoingest`
jobs (thumbnail / filmstrip / md5, via `queueVideoIngestJob`) must be skipped.
For the feed thumbnail the good answer is to **queue a short deforum render that
uses the LoRA** and use that as its thumbnail — the catalog then shows what each
LoRA actually does, which is precisely what a picker needs. A user-supplied
preview image is the interim.

**Resolution at submit time** — `backend/src/utils/dream.util.ts:136`, where
`jobData` is built for the `deforumvideo` queue:

1. Scan the prompt values for `<lora:alias:strength>` (same regex as
   `lora.py:10`).
2. Resolve each token in the order above — registry alias first, then UUID
   against the submitter's own LoRA dreams and public ones. Unresolved → reject
   via the existing `failDreamWithError` path, listing the available aliases.
   Today that failure costs a GPU cold start before it surfaces.
3. Attach `loras: [{alias, r2Key, sha256, dream_uuid}]` to `jobData`, and record
   what was resolved on the render — free, since the digest already exists.

Passing `r2Key` keeps the container simple, since it already holds R2
credentials; presigned GETs are the fallback if the LoRA bucket ever diverges
from the render bucket.

**Also needed**: `worker/src/services/cli.service.ts` accepts prompts directly
for CLI submissions, bypassing the backend. Either resolve aliases there too, or
document that CLI submissions must pass `loras` explicitly.


## Phase 3 — surfacing (frontend, optional/later)

- A `lora` filter on the feed and on a user's profile — the query support is
  already there, so this is mostly grid/card rendering for a media type that has
  a thumbnail but nothing to play.
- A picker in the deforum settings that inserts `<lora:alias:1>` into the prompt
  along with the LoRA's trigger words, since the tag alone never reaches the
  encoder.
- An admin view of claimed aliases, with the ability to clear or re-point one.
- Migrate the Studio camera presets off hardcoded HuggingFace URLs
  (`action-presets.ts:5`) onto LoRA dreams — same records, `wan-i2v-lora` path.


## Risks and open questions

| Item | Handling |
| --- | --- |
| Warm worker retains a stale LoRA set after a failed job | **Unconfirmed** — reproduce per Phase 0 item 3 before fixing; does not gate Phase 1 |
| Cold-start download latency | Digest-keyed container-local cache that outlives the job; parallel fetch; only missing files. Measure in Phase 0 |
| Ephemeral disk exhaustion on a long-lived worker | Per-job LoRA cap plus an LRU sweep of the cache dir above a size ceiling |
| Alias takeover | A claim is never overwritten by another upload; only an admin can write over an existing one |
| Alias land-grabs | Optional per-user claim rate limit; admins can re-point a name at a safe LoRA. UUID fallback means a taken name never blocks anyone |
| Alias collisions on the worker | Resolution happens **server-side**; the container only ever sees the resolved set for one job, and stale symlinks are cleared per job |
| A third `mediaType` silently joining `Not(IMAGE)` queries | Convert native-client and playlist filters to explicit `In([VIDEO])` in the same PR as the enum value |
| Malicious upload | safetensors header validation + size cap + server-side hashing |
| SD1.5 LoRA on an SDXL checkpoint | Record `baseModel`, warn or reject at submit |
| Licensing / civitai ToS / NSFW | Store `sourceUrl` and `license`; keep user uploads private by default, publishing is an explicit admin action |
| Storage cost | Negligible — content-addressed, deduped, a few hundred MB each |

**Open questions for the issue:**

1. On a name collision at upload: leave the LoRA unaliased and tell the uploader,
   or auto-suffix (`fractal-geometry-2`)? Unaliased-and-told is honest; the
   auto-suffix never interrupts but litters the namespace.
2. Do LoRA dreams belong in the main feed alongside videos, or only behind the
   `lora` filter? They are the only media type you cannot watch, so mixing them
   into the default feed changes what the feed is.

## Sequencing

Phase 0 and Phase 1 are independently shippable and deliver the thing that
actually hurts today (the rebuild). Phase 2 is the larger chunk and can land
behind an admin-only flag. Phase 3 is polish.

Per-frame LoRA strength interpolation (the 2026-08-29 comment) stays out of
scope and is unaffected by any of this — it changes how strengths are *parsed and
applied*, not where the files come from.
