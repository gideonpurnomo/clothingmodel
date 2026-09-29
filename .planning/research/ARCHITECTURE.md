# Architecture Research

**Domain:** Photo-to-3D garment visualization / virtual fitting (Next.js + react-three-fiber)
**Researched:** 2026-09-28
**Confidence:** MEDIUM-HIGH (3D-gen API contracts verified from official docs; binding technique verified across academic + practitioner sources; queue patterns from current ecosystem guides)

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         CLIENT (Next.js App)                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌────────────────┐    │
│  │ Upload / │  │ Job      │  │ 3D Viewer    │  │ Body Adjust +  │    │
│  │ URL Poll │  │ Status UI│  │ (r3f + GLTF) │  │ Size Selector  │    │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘  └───────┬────────┘    │
│       │  polling /  │               │                  │             │
│       │  SSE        │  SWR/poll     │                  │             │
├───────┴─────────────┴───────────────┴──────────────────┴─────────────┤
│                      NEXT.JS API ROUTES (BFF)                        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌───────────┐  │
│  │ Auth     │ │ Garment  │ │ Job      │ │ Share     │ │ Notify    │  │
│  │ (email)  │ │ CRUD     │ │ create/  │ │ link      │ │ poll      │  │
│  │          │ │          │ │ status   │ │ (public)  │ │           │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────┬─────┘ └─────┬─────┘  │
├───────┴────────────┴────────────┴──────────────┴─────────────┴────────┤
│                     WORKER / PIPELINE (job-driven)                    │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │ Stage 1: image cleanup + validate + store to object storage    │   │
│  │ Stage 2: call 3D-gen API (Meshy/Tripo) → store provider task id│   │
│  │ Stage 3: await completion (webhook OR poll) → download GLB     │   │
│  │ Stage 4: mesh post-process (decimate, orient, strip junk)      │   │
│  │ Stage 5: LLM size extraction + regional normalization → cm     │   │
│  │ Stage 6: scale calibration (measurements → world scale)        │   │
│  │ Stage 7: garment↔mannequin binding precompute → finalized      │   │
│  └────────────────────────────────────────────────────────────────┘   │
├───────────────────────────────────────────────────────────────────────┤
│                           EXTERNAL SERVICES                           │
│  ┌───────────┐  ┌───────────┐  ┌──────────┐  ┌─────────────────────┐  │
│  │ Meshy /   │  │ LLM API   │  │ Object   │  │ Queue/durable-exec  │  │
│  │ Tripo API │  │ (sizing)  │  │ storage  │  │ (Inngest/QStash/    │  │
│  │           │  │           │  │ (S3/R2)  │  │  pg-boss)           │  │
│  └───────────┘  └───────────┘  └──────────┘  └─────────────────────┘  │
├───────────────────────────────────────────────────────────────────────┤
│                        DATA (Postgres)                                │
│  ┌───────┐ ┌──────────┐ ┌──────┐ ┌───────────────┐ ┌─────────────┐    │
│  │ users │ │ garments │ │ jobs │ │ share_links   │ │ notifications│   │
│  └───────┘ └──────────┘ └──────┘ └───────────────┘ └─────────────┘    │
└───────────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|------------------------|
| Client viewer | Render GLB garment + parametric mannequin; orbit; apply pseudo-fit per body preset; size selector | react-three-fiber + drei `OrbitControls`, `useGLTF`; binding solved client-side or via precomputed bind data |
| API routes (BFF) | Auth, CRUD, quota check, job submission, share-link resolution, notification feed | Next.js route handlers, thin — validate/authorize then delegate to services |
| Pipeline worker | The 7-stage async chain per garment (see Data Flow) | Durable-execution job (Inngest step function / QStash message chain / pg-boss job). One job = one garment, all stages |
| 3D-gen adapter | Wrap Meshy/Tripo behind one interface: `submit(images) → providerTaskId`, `fetchStatus(id)`, `downloadResult(id)` | Single module so swapping/adding providers (or self-hosted TRELLIS later) touches one file |
| LLM sizing extractor | Product description → structured JSON {category, region, sizes[], measurements{chest, waist, length...} in cm} | One LLM call with JSON schema / tool-use; validate + clamp; fall back to standard size chart on failure |
| Object storage | Source photos, provider GLB downloads, post-processed GLBs, thumbnails | S3-compatible (R2/S3/Supabase Storage). Never store provider's signed URLs long-term — they expire |
| Postgres | System of record: users, garments, jobs, share links, notifications, quota usage | Prisma or Drizzle ORM; jobs table is also the queue's source of truth |
| Notifications | In-app "garment ready/failed" bell | Rows in `notifications` table; client polls (v1) — no WebSocket infra needed |

## Recommended Project Structure

```
src/
├── app/
│   ├── (auth)/               # login, signup, verify-email routes
│   ├── (app)/                # authenticated shell: library, garment detail
│   ├── share/[token]/        # PUBLIC share-link viewer (no auth layout)
│   └── api/
│       ├── auth/[...]/       # auth handlers
│       ├── garments/         # CRUD, POST /garments (creates job)
│       ├── jobs/[id]/        # status polling endpoint
│       ├── notifications/    # unread count + list + mark-read
│       ├── share/[token]/    # public garment data (read-only)
│       └── webhooks/         # 3D-gen provider callback receiver (if used)
├── components/
│   ├── viewer/               # r3f scene, garment loader, mannequin, orbit
│   ├── upload/               # guided front/back photo capture + URL paste
│   └── ui/                   # generic UI (status badges, size selector...)
├── lib/
│   ├── providers/
│   │   ├── meshy.ts          # 3D-gen adapter: Meshy
│   │   └── tripo.ts          # 3D-gen adapter: Tripo
│   ├── sizing/
│   │   ├── extract.ts        # LLM structured extraction
│   │   ├── size-charts.ts    # standard fallback charts (per region)
│   │   └── normalize.ts      # regional units → cm
│   ├── fit/
│   │   ├── bind.ts           # closest-point binding + push-out (shared logic)
│   │   └── morphs.ts         # mannequin blend-shape presets/sliders
│   ├── pipeline/             # the 7 stages, one function each
│   └── db/                   # schema, migrations, queries
└── ingest/ or worker/        # queue entrypoint (Inngest functions / pg-boss worker)
```

### Structure Rationale

- **`lib/providers/`:** Project explicitly defers self-hosting; a single adapter interface means the Meshy→Tripo comparison is cheap and TRELLIS later is a new file, not a rewrite.
- **`lib/pipeline/` stages as separate functions:** Each stage is independently retryable; durable-execution tools (Inngest `step.run`) map 1:1 onto stage functions.
- **`lib/fit/`:** Binding math must be testable in Node (unit tests with tiny meshes) before it touches the browser viewer.
- **`share/[token]/` as its own route group:** Public pages must not leak auth layout or fetch patterns; explicit isolation prevents accidental auth dependencies.

## Architectural Patterns

### Pattern 1: Pipeline-as-Recorded-Job (jobs table = source of truth)

**What:** One DB row per garment generation carrying `stage`, `provider_task_id`, `error`, timestamps. The queue mechanism (Inngest/QStash/pg-boss) drives execution, but the DB row is what the UI reads — the queue is an implementation detail.
**When to use:** Always for this project. Multi-minute, multi-stage, paid-API jobs demand durable state that survives deploys and restarts.
**Trade-offs:** Slight duplication (queue state + DB state); resolved by making DB the read model and queue the executor only.

```typescript
// Inngest example — each step is retried independently
export const generateGarment = inngest.createFunction(
  { id: "generate-garment", retries: 2 },
  { event: "garment/created" },
  async ({ event, step }) => {
    const images = await step.run("cleanup", () => cleanAndStore(event.garmentId));
    const taskId = await step.run("submit-3dgen", () => meshy.submit(images));
    const glb = await step.run("await+download", () =>
      pollUntilDone(taskId).then(() => meshy.downloadGlb(taskId)));
    const processed = await step.run("postprocess", () => processMesh(glb));
    const sizes = await step.run("extract-sizes", () => extractSizing(event.description));
    const scaled = await step.run("calibrate", () => calibrateScale(processed, sizes));
    await step.run("bind+finalize", () => precomputeBinding(scaled));
  }
);
```

**Queue choice for this stack (opinionated):**

| Option | Verdict | Why |
|--------|---------|-----|
| **Inngest** | Recommended default | Durable steps map 1:1 to pipeline stages; free tier; works on Vercel and self-hosted; local dev UI |
| QStash (Upstash) | Good alternative if pipeline stays a simple chain | HTTP-native, dead simple, but multi-stage state must be hand-rolled |
| pg-boss | Good if self-hosted with a persistent worker process | ACID + no third party; painful on serverless (needs long-running worker) |
| BullMQ + Redis | Avoid for v1 | Extra infra (Redis) for zero benefit at this scale |
| Vercel cron + jobs table | Acceptable floor | Poll DB every minute, advance any stale job. Crude but zero-dependency; fine while validating |

### Pattern 2: Poll-first, webhook-later (3D-gen completion)

**What:** The worker polls the provider's task endpoint on an interval until terminal status; webhooks are an optional later optimization to reduce latency/cost.

**Verified API facts:**

- **Meshy** (official docs, HIGH confidence): `POST /openapi/v1/image-to-3d` → task id; `GET .../:id` returns `status` ∈ {`PENDING`, `IN_PROGRESS`, `SUCCEEDED`, `FAILED`, `CANCELED`} with `progress` 0–100; result `model_urls` (glb/fbx/obj/usdz/stl) are **signed and expiring** — download promptly. Also offers an SSE `/stream` endpoint per task. A webhook option exists (help.meshy.ai documents webhooks vs polling) but `webhook_url` was not present in the current endpoint parameter list I fetched — verify at build time; do not design around it.
- **Tripo** (developer docs, MEDIUM confidence): task-based `POST /task` → `GET /task/{id}` with status/progress/output URLs, plus a documented `webhook_url` callback and batch status query (up to 100 task ids per call).

**Recommendation:** Poll every 5–10s with exponential backoff inside the queue step (durable sleep). Add webhooks in v2 only if polling costs or completion latency actually hurt. Rationale: webhooks require a publicly reachable endpoint, signature verification per provider, and a reconciliation poller anyway (webhooks get lost). Since the worker is already alive waiting on this one job, polling is simpler and provider-agnostic.

```typescript
// provider-agnostic wait inside a durable step
async function pollUntilDone(taskId: string): Promise<TaskResult> {
  for (let i = 0; i < MAX_POLLS; i++) {
    const t = await meshy.fetch(taskId);
    if (t.status === "SUCCEEDED") return t;
    if (t.status === "FAILED" || t.status === "CANCELED") throw new GenError(t.task_error);
    await sleep(Math.min(5000 * 1.5 ** i, 30000)); // durable sleep, not setTimeout
  }
  throw new TimeoutError(taskId);
}
```

### Pattern 3: Pseudo-fit — closest-point binding + collision push-out

**What:** Per garment vertex, find the closest point on the mannequin surface; store (closest-point reference, surface normal, rest offset). When the body morphs, transform the vertex toward the new surface and push it out along the normal so it never penetrates.

**Technique (verified across academic + practitioner sources, MEDIUM-HIGH confidence):**

1. **Precompute bind (once per garment, can be server-side or on first load):**
   - Normalize garment into mannequin space (position + scale from size calibration).
   - For each garment vertex, query closest point on the **base** mannequin mesh using a BVH (`three-mesh-bvh` `closestPointToPoint` — the standard acceleration structure in the three.js ecosystem; brute force is O(n·m) and too slow for 30k+ vertex meshes).
   - Store per vertex: `barycentric coords / triangle index` of the closest point + `offset = vertexPos − closestPoint` (the garment's "looseness" at rest).
2. **On body morph (runtime, per slider change):**
   - Apply blend-shape morphs to mannequin; the BVH must be **refit** (`bvh.refit()` is cheap for deformations).
   - Target position = interpolate the stored closest point on the morphed surface + rotated offset along the (new) normal.
   - **Push-out:** if the resulting vertex is inside the morphed body (sign test via normal direction or SDF), project it back to the surface along the normal. This is the canonical formulation used in SMPL-based garment fitting research (the "outside term": push vertex along vector from vertex to closest surface point).
3. **Smoothing:** run one Laplacian smoothing pass over affected vertices (or blend per-vertex displacement with neighborhood average) to avoid single-vertex spikes where push-out is aggressive.

**Known limitation (already accepted in PROJECT.md):** offset-preservation means extreme morphs hug like spandex. Mitigations that stay cheap: scale the rest offset proportionally with body surface growth at that region; clamp per-vertex displacement; communicate honestly in UI.

**Alternative rejected for v1:** skinning-weight transfer (transfer body bones to garment via proximity) — works well for posed skeletons but our mannequin is blend-shape morphs, not bone-driven, so closest-point binding is the direct fit. Full cloth sim (cannon-es/ammo soft bodies, Blender-baked drape) is the v3 destination.

### Pattern 4: Opaque-token share links

**What:** `share_links` table: `id`, `garment_id`, `token` (random 22+ chars, base64url, indexed unique), `revoked_at`. Public route `/share/[token]` renders the read-only viewer fetching from `/api/share/[token]` — no auth, no user data, only garment + display state. Never use sequential/garment ids in URLs (enumeration).

## Data Flow

### Garment creation flow (the core flow)

```
[Upload 2 photos or paste URL]
    ↓ POST /api/garments (multipart or URL)
[API route] → auth check → quota check (count jobs this month)
    → store raw images → INSERT garment(status=queued) + job row
    → emit queue event → return {garmentId} immediately
    ↓ (worker, async, minutes)
[Stage 1 cleanup]  strip metadata, validate image, check min dimensions
[Stage 2 submit]   meshy.submit(front, back) → provider_task_id saved
[Stage 3 wait]     poll → SUCCEEDED → download GLB → object storage
[Stage 4 process]  decimate to viewer budget (~30-50k tris), fix orientation
[Stage 5 sizes]    LLM(description) → {region, sizes, cm measurements} → validated
[Stage 6 calibrate] measurement (e.g. chest cm) → distance between mesh loops
                   → uniform scale factor → baked transform
[Stage 7 bind]     precompute bind data (closest points + offsets) → JSON/binary
    → UPDATE garment(status=ready, assetUrls...) + INSERT notification
    ↓
[Client] polls /api/jobs/[id] (SWR, 3-5s interval) → shows progress →
on ready: fetch notification list, load viewer, apply binding
```

### Share-link flow

```
[Visitor opens /share/TOKEN] → public page → GET /api/share/TOKEN
→ lookup token (not revoked) → return garment + default display state only
→ viewer renders read-only (no body adjust controls in v1, or frozen at owner's preset)
```

### State management (client)

- **Server state:** SWR/React Query for garments list, job status (poll), notifications (poll every ~30s, fast-poll while a job is active).
- **Viewer state:** plain zustand store — `{bodyPreset, sliders, selectedSize, garmentAssets}`. Body morph + binding recompute is a pure function of that store, memoized on slider values.

## Scaling Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0–1k users | Single Next.js deploy + Inngest/QStash free tier + one Postgres. Binding precompute in the worker (server), viewer only consumes. Nothing else needed |
| 1k–100k users | First costs are API costs, not infra: enforce quota strictly, cache provider results. Add CDN caching on share-link pages (immutable GLBs). Move binding precompute off the request path entirely |
| 100k+ users | Split worker from web deploy; consider self-hosted TRELLIS for cost (per PROJECT.md, revisit); WebGL viewer perf becomes the constraint (DRACO/meshopt compression on GLBs) |

### Scaling Priorities

1. **First bottleneck: 3D-gen API cost & latency, not servers.** At $0.20–0.40/garment, quota enforcement and idempotent job submission (never double-submit on retry — check for existing `provider_task_id` before Stage 2) matter before any infrastructure scaling.
2. **Second bottleneck: viewer payload.** Raw Meshy GLBs are heavy; meshopt/DRACO compression and a ~30–50k triangle budget in Stage 4 keep mobile browsers usable.

## Anti-Patterns

### Anti-Pattern 1: Trusting provider asset URLs

**What people do:** Store Meshy/Tripo `model_urls` in the DB and serve them directly to the viewer.
**Why it's wrong:** They are signed, **expiring** URLs — share links and old library entries break days later.
**Do this instead:** Download the GLB to own object storage in Stage 3; DB stores your URLs only.

### Anti-Pattern 2: Blocking the request on generation / running pipeline in a route handler

**What people do:** `await generateGarment()` inside the POST route, or `after()`/fire-and-forget promises in serverless.
**Why it's wrong:** Minutes-long requests time out; serverless invocations are killed after response, silently dropping multi-stage work mid-pipeline (and mid-payment).
**Do this instead:** Durable queue job that writes stage progress to the jobs table; route returns in <1s.

### Anti-Pattern 3: Binding computed against the base mannequin only

**What people do:** Precompute closest points once, then at runtime just apply offsets on the morphed skeleton without refitting/testing penetration.
**Why it's wrong:** Morphs move the surface; vertices end up inside the body (visible clipping at plus-size presets) — the exact artifact this feature exists to prevent.
**Do this instead:** Refit the BVH after morph application and always run the push-out pass; precompute only the *correspondences*, not final positions.

### Anti-Pattern 4: Free-form LLM output for sizing

**What people do:** Ask the LLM for sizing and parse its prose, or accept its numbers unvalidated.
**Why it's wrong:** Garments render at wrong scale — the core value prop fails — and regional sizes (Asian L ≈ US M) silently mis-map.
**Do this instead:** JSON-schema-constrained output (tool use), validate every field against plausible ranges, normalize region+units to cm immediately, fall back to standard size charts per category when extraction fails or returns garbage.

### Anti-Pattern 5: Sequential share tokens / leaking user data on share pages

**What people do:** `/share/123`, returning the garment row with owner info.
**Why it's wrong:** Enumeration + PII leak.
**Do this instead:** Opaque random token, revoked-flag support, dedicated public API returning only display data.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Meshy image-to-3d | `POST /openapi/v1/image-to-3d` (Bearer key) → poll `GET /:id` → download GLB | Statuses PENDING/IN_PROGRESS/SUCCEEDED/FAILED/CANCELED; supports multi-image input (front+back) — this is why "Option C" works; signed expiring result URLs; credits refunded only if task deleted while PENDING |
| Tripo | `POST /task` → poll `GET /task/{id}`, optional `webhook_url` | Batch status query (100 ids/call) if many jobs; adapter keeps swap cheap |
| LLM provider | Single structured-output call per garment | Cheap relative to 3D-gen (~$0.01 vs ~$0.30); retry with fallback chart |
| Object storage (S3/R2) | Presigned upload for photos (direct client→storage), server-side write for GLBs | Don't proxy image bytes through Next.js routes |
| Queue (Inngest/QStash) | HTTP endpoint in Next.js receives trigger + step callbacks | Local dev story matters — Inngest Dev Server is the strongest |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| API routes ↔ pipeline | Queue events only (never direct function calls) | Keeps web tier stateless; jobs survive deploys |
| Pipeline ↔ 3D-gen | Adapter module (`lib/providers`) | All provider knowledge in one place |
| Worker ↔ viewer (binding) | Precomputed bind data artifact (JSON/binary in storage) | Bind math runs once server-side; viewer just applies — keeps client cheap and logic testable in Node |
| Viewer ↔ body/size state | Zustand store → pure `applyFit()` function | Deterministic, memoizable, unit-testable |

## Build Order Implications (for roadmap)

Dependency-driven order — each phase unlocks the next:

1. **Viewer skeleton** — mannequin + GLB load + orbit controls. No backend. Enables all later visual work and de-risks r3f early.
2. **Accounts + garment CRUD + upload** — auth, storage, data model. Jobs table exists but pipeline can stub out (fake GLB).
3. **3D-gen pipeline** — queue + Meshy adapter + poll + download + post-process + job status UI. First paid-key phase; needs key flagging.
4. **LLM size extraction + scale calibration** — text problem; independent of pipeline internals except final scale bake. Also needs a key.
5. **Body morphs + pseudo-fit binding** — mannequin blend shapes, BVH bind, push-out. Depends on calibrated garments (Phase 4) to look right.
6. **Notifications + share links + quota + polish** — notifications depend on pipeline completion events (Phase 3); share links depend on viewer + assets (Phase 3); quota needed before public launch.

Critical path: **2 → 3 → 5**. Phases 4 and the 6-trio are mostly parallelizable off that spine.

## Sources

- Meshy image-to-3d API reference (endpoints, statuses, model_urls, parameters) — https://docs.meshy.ai/en/api/image-to-3d (fetched 2026-09-28, HIGH confidence)
- Meshy webhooks vs polling guidance — https://help.meshy.ai (Sep 2026 article surfaced via search; webhook_url not confirmed in current endpoint params — verify at build time, MEDIUM)
- Tripo developer platform (task API, webhook, batch query) — https://developers.tripo3d.ai (MEDIUM)
- Collision push-out canonical formulation ("outside term", closest-point projection) — Detailed human shape estimation, arXiv 1703.04454; HOOD/garment-line work, arXiv 2103.06871 (HIGH confidence in technique, academic)
- three-mesh-bvh closest-point acceleration — https://github.com/gkjohnson/three-mesh-bvh (practitioner standard, MEDIUM-HIGH)
- Blender-bake / playback pseudo-fit production pattern — Codrops, Aug 2025 (LOW relevance to v1, noted as alternative)
- Next.js background jobs landscape (Inngest/QStash/pg-boss/Trigger.dev trade-offs) — vercel.com guides, inngest.com docs, upstash.com comparison, pkgpulse.com 2026 (MEDIUM — ecosystem consensus, no single authority)

---
*Architecture research for: photo-to-3D garment visualization / virtual fitting*
*Researched: 2026-09-28*
