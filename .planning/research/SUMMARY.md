# Project Research Summary

**Project:** clothingmodel — web-based 3D garment visualization / virtual fitting
**Domain:** Consumer web app: photo-to-3D garment generation + true-to-size viewer (Next.js + react-three-fiber)
**Researched:** 2026-09-29
**Confidence:** MEDIUM-HIGH overall (pricing, API contracts, and package versions verified from official sources; garment-quality of 3D-gen output and binding technique quality are the residual unknowns)

## Executive Summary

This product turns two garment photos (front + back) — or a pasted product URL — into a textured 3D garment rendered on an adjustable mannequin at true-to-size scale, with LLM-extracted sizing. Research shows no consumer competitor does exactly this: the 2D try-on space (Zyler, Zeekit/Walmart) is crowded, and professional 3D garment tools (CLO3D, Browzwear) are pattern-based design software. The "2 photos → real 3D garment on adjustable body" niche is empty, which means the primary risk is 3D generation *quality on garments* (thin shells, straps, patterns are the hardest inputs for photo-to-3D APIs), not competition.

The recommended approach is already well-supported by research: keep the locked Next.js 16 + react-three-fiber stack, add Inngest for the minutes-long 7-stage generation pipeline (cleanup → submit to Tripo/Meshy → poll → download → gltf-transform post-process → LLM size extraction → scale calibration → bind precompute), Postgres/Drizzle/Better Auth for data and accounts, and Cloudflare R2 for assets. Use Tripo as primary 3D-gen provider with Meshy as A/B fallback behind a single adapter interface (~$0.20–0.30/garment textured, verified pricing). Never run the pipeline in request handlers, never trust provider asset URLs (they expire), never trust mesh scale (arbitrary units — calibrate against LLM-extracted cm).

Key risks and mitigations: (1) **garbage meshes** — treat API output as untrusted; build an automated validation gate and retry path the moment the first real generation is wired; (2) **cost exposure** — quota enforcement is server-side, atomic, and built into job creation from day one; a global daily spend circuit breaker must exist before public launch; (3) **mobile performance** — enforce an asset budget (≤30–50k tris, ≤8MB GLB, meshopt + KTX2) in the worker pipeline, and test on a real mid-range Android from the first viewer phase; (4) **job reliability** — persist before submitting, idempotent state transitions, reconciliation sweeper for stuck jobs; (5) **pseudo-fit artifacts** — garment normal-inflation (1–3mm), BVH refit after morphs, collision push-out, and honest UX copy about the spandex-hug limit.

## Key Findings

### Recommended Stack

See [STACK.md](./STACK.md). Everything is TypeScript/npm-only — no Python runtime, no Redis, no worker process.

**Core technologies:**
- **Next.js 16.3 + React 19 + react-three-fiber 9.8 / drei 10** — locked; App Router + Server Actions cover the API surface; R3F v9 is React 19-native (do not downgrade React for old tutorials)
- **Postgres (Neon) + Drizzle ORM** — auth tables and app tables in one schema; free tier carries development
- **Better Auth 1.7 + Resend** — email/password with built-in email verification hook; fastest path to the exact requirement set
- **Inngest 4.21** — durable steps map 1:1 to pipeline stages (`step.run` per stage, `step.sleep` between polls); free tier (50k executions/mo) fits thousands of garments; replaces cron and worker infra
- **Tripo API (primary) / Meshy API (fallback)** — image-to-3D at 20–30 credits ($0.20–0.30/garment), pay-as-you-go; wrap both behind one provider interface and A/B on real garments early
- **Gemini Flash-class via Vercel AI SDK `generateObject`** — size extraction (description → JSON schema), fractions of a cent per call; model picked at build time
- **Cloudflare R2** — zero-egress storage for photos and GLBs (S3 egress at $0.09/GB would dominate cost for shareable 360° links)
- **@gltf-transform 4.5** — server-side GLB post-processing (simplify, orient, meshopt/KTX2 compression) inside an Inngest step
- **Supporting:** zod 4 (one schema language everywhere), @t3-oss/env-nextjs, React Query (job polling), zustand (viewer state), cheerio (v1 scraping)

**Critical economic fact:** monthly COGS per active free user ≈ **$2–3, dominated entirely by 3D-gen credits** at a 10-garment quota. Quota enforcement is a cost-survival feature, not a nicety.

### Expected Features

See [FEATURES.md](./FEATURES.md). Full v1 scope is decided in PROJECT.md; research validates it and adds nuance.

**Must have (table stakes):**
- Guided front+back photo upload with example thumbnails and validation
- Async generation with staged status UI (queued → generating → finalizing) + in-app notification — never a frozen spinner
- 360° orbit/zoom viewer with load progress (Sketchfab sets the UX floor)
- 4–6 body presets including plus-size (Zeekit precedent: representation is expected)
- LLM size extraction with regional normalization + size selector S–XXL with visible change
- Garment library with delete/rename, visible free-tier quota, retry-on-failure without quota charge, public read-only share links, mobile-responsive viewer

**Should have (differentiators):**
- **Paste-a-product-URL import** — found in NO surveyed competitor; the primary differentiator, but ship as best-effort with upload fallback (upload must never be dependent on it)
- True-to-size scale on adjustable mannequin from just 2 photos — the core "wow"
- Measurement overlay, size comparison view, body sliders, turntable video export (v1.x triggers defined)

**Defer (v2+):** Stripe payments (rails in v1), AR Quick Look/USDZ export, 2D photorealistic try-on bolt-on, multi-garment outfits, self-hosted generation, in-viewer material editing, social features, email-ready notifications (in-app only, per PROJECT.md).

### Architecture Approach

See [ARCHITECTURE.md](./ARCHITECTURE.md). One Next.js deploy + Inngest pipeline + one Postgres + R2. The core pattern is a **pipeline-as-recorded-job**: a `jobs` table is the UI's read model; Inngest is only the executor. A single provider adapter isolates all 3D-gen knowledge. The fit problem uses **pseudo-fit**: precompute per-vertex closest-point correspondences against the base mannequin (via three-mesh-bvh), then at runtime refit the BVH after morphing, interpolate along the morphed surface, and run a collision push-out pass. Binding is computed server-side once per garment; the viewer only applies it.

**Major components:**
1. **Client viewer** — r3f scene: garment GLB + blend-shape mannequin, orbit, `applyFit()` as a pure function of a zustand store
2. **API routes (BFF)** — thin: auth, CRUD, quota check, job submit, share-token resolution; return in <1s
3. **7-stage pipeline worker** — cleanup → submit → wait/download → mesh post-process → size extraction → scale calibration → bind precompute; each stage independently retryable
4. **Provider adapter** — `submit() / fetchStatus() / downloadResult()`; Meshy and Tripo both fit; makes A/B and future TRELLIS a one-file change
5. **Data** — users, garments, jobs, share_links (opaque 22+ char tokens, never sequential IDs), notifications

### Critical Pitfalls

See [PITFALLS.md](./PITFALLS.md). Top 5:

1. **Photo-to-3D APIs return garbage meshes for garments** (~70% of raw AI outputs have non-manifold edges/inverted normals; straps vanish, buttons blob) — build an automated validation gate (`gltf-transform inspect` heuristics) and a retry/"try another photo" path *with* the first real integration, plus a 10–20 garment eval set tracked across API changes.
2. **Generated meshes are not true-to-size** — API output is arbitrary units; calibrate in the pipeline against LLM-extracted cm (canonical dimension policy per category), build everything in 1-unit=1cm space, add a dev-mode ruler.
3. **Job loss and double-billing** — persist task IDs before/upon submission, idempotent webhook/poll state transitions, reconciliation sweeper for stuck jobs, dedupe on client request IDs, log credit spend per job and reconcile against the provider bill.
4. **Cost exposure without quota enforcement** — server-side atomic quota at job creation, per-IP/device rate limits + Turnstile, and a global daily spend circuit breaker verified before public launch.
5. **Mobile GPU melt + pseudo-fit clipping** — asset budget enforced in the worker (≤8MB, meshopt, ≤1024px textures, KTX2); garment normal-inflation + BVH refit + push-out + polygonOffset + honest `camera.near` tuning; test on real mid-range Android from the first viewer phase.

Also significant: **URL scraping fragility** (fashion retail runs aggressive anti-bot; design upload-first with graceful degradation, JSON-LD-first extraction, per-domain success telemetry) and **background removal failure on real user photos** (mandatory matting with a pre-flight quality gate + user confirmation of the cutout *before* spending credits — the single highest-leverage prevention in the pipeline).

## Implications for Roadmap

Based on research, suggested phase structure (aligned with the dependency-driven build order in ARCHITECTURE.md; critical path is **2 → 3 → 5**):

### Phase 1: Viewer Skeleton
**Rationale:** De-risks r3f/React 19/drei compatibility and establishes the asset budget contract (tri count, GLB size, decoder setup, dpr, frameloop) before any pipeline exists. No backend needed.
**Delivers:** Mannequin mesh + sample GLB load, orbit/zoom with touch support, load progress, HDRI lighting, performance budget documented and tested on a real mid-range Android.
**Addresses:** 360° viewer table stake; sets contract for Pitfall 3 (mobile perf).
**Avoids:** Mobile GPU melt — budget is a Phase 1 decision, not a polish-phase retrofit.

### Phase 2: Accounts, Upload, and Garment CRUD
**Rationale:** Everything user-facing hangs off auth; the data model (users, garments, jobs, share_links) unblocks the pipeline phase. Upload includes the matting + quality gate + cutout confirmation, which must exist before any credits are spent.
**Delivers:** Better Auth + email verification, guided front+back upload with background removal and pre-submission cutout confirmation, image validation/caps, garment records in Postgres, R2 presigned photo uploads.
**Uses:** Better Auth/Drizzle/Resend, @imgly/background-removal-node, R2.
**Avoids:** Pitfall 8 (background removal on bad photos) and the SSRF/upload-abuse security mistakes; Pitfall 6 partially (server-side input caps).

### Phase 3: 3D-Gen Pipeline (the paid-API phase)
**Rationale:** The product's engine. The job state machine, validation gate, fixture mode, and provider abstraction are first-version requirements, not hardening. Server-side quota enforcement lands here because this is where cost begins.
**Delivers:** Inngest 7-stage pipeline with Meshy + Tripo adapters (A/B harness), poll-first completion with reconciliation sweeper, mesh validation gate + gltf-transform post-processing to the Phase 1 budget, job status UI with staged progress + in-app notification, server-side atomic quota + retry-without-charge, per-job credit spend logging, recorded-fixture mode so tests never hit the live API.
**Uses:** Inngest, provider adapter pattern, gltf-transform, React Query polling.
**Implements:** Pipeline-as-recorded-job; avoids Pitfalls 1, 5, 9 and the cost side of Pitfall 6.

### Phase 4: LLM Size Extraction + Scale Calibration
**Rationale:** A text problem independent of pipeline internals except for the final scale bake. Must land before body morphs/binding can mean anything — scaling to "size M" requires knowing what M measures.
**Delivers:** `generateObject` extraction (category, region, sizes, cm measurements) with validation ranges and regional normalization, standard size-chart fallback, deterministic scale calibration step (canonical dimension policy), dev-mode measurement ruler.
**Uses:** AI SDK + Gemini Flash, zod schema shared end-to-end.
**Avoids:** Pitfall 2 (arbitrary scale) and the free-form-LLM-output anti-pattern.

### Phase 5: Body Morphs + Pseudo-Fit Binding
**Rationale:** The hardest technical phase and the core "wow." Depends on calibrated garments (Phase 4) and the asset budget (Phase 1). Binding strategy (co-morph + push-out + inflation) must be decided up front — reworking it later is the most expensive recovery in the pitfalls table.
**Delivers:** 4–6 blend-shape mannequin presets (interpolable from the start to enable later sliders), server-side BVH closest-point bind precompute, runtime refit + push-out + Laplacian smoothing, size selector driving garment swap/scale, honest fit-approximation UX on large deltas.
**Uses:** three-mesh-bvh, blend-shape morph targets (not SMPL — license-incompatible).
**Avoids:** Pitfall 4 (clipping/z-fighting) and the bind-against-base-mannequin-only anti-pattern.

### Phase 6: Library, Share Links, Quota UI, URL Import, Polish
**Rationale:** Retention/growth loop plus the differentiator; all depend on earlier phases (notifications ← pipeline, share links ← viewer + assets) and are parallelizable. URL import ships as best-effort with upload as the guaranteed path. Global spend circuit breaker verified here, before any public launch.
**Delivers:** Garment library grid with delete/rename, visible remaining-quota UI, opaque-token public share links (read-only viewer, OG preview, revoke + abuse report), paste-URL import (JSON-LD/OG extraction, graceful degradation, per-domain success telemetry), mobile polish, spend dashboard/alerting.
**Addresses:** Table-stake library/quota/share features + the URL-import differentiator.
**Avoids:** Pitfall 7 (scraping fragility), share-link enumeration/PII anti-patterns, quota bypass.

### Phase Ordering Rationale

- **Dependencies:** Viewer budget → pipeline output contract; auth/upload → jobs → pipeline; extraction → calibration → meaningful fit; pipeline events → notifications; viewer + assets → share links.
- **Critical path 2 → 3 → 5** (accounts → generation → fit); Phases 4 and 6 are mostly parallelizable off that spine.
- **Pitfall alignment:** each pitfall's "phase to address" from PITFALLS.md maps onto exactly one of these phases, with validation/verification criteria specified there.
- **v1.x backlog (post-roadmap):** body sliders, size comparison view, measurement overlay, turntable video export, scraping breadth — each has a defined trigger from FEATURES.md.

### Research Flags

Phases likely needing deeper research during planning (`/gsd:plan-phase --research-phase`):
- **Phase 3 (3D-gen pipeline):** Meshy `webhook_url` unconfirmed in current endpoint params; signup credit allowances unverified for both providers; A/B evaluation design; exact credit/refund policies at account creation.
- **Phase 5 (pseudo-fit binding):** Highest technical risk — blend-shape rig design, bind-data artifact format, z-fighting tuning on real generated meshes. Sparse practitioner documentation; academic sources only.
- **Phase 2 (background removal):** @imgly quality on real garment photos is flagged for validation; fallback (rembg microservice or skip) decided by in-phase experiment.
- **Phase 6 (URL import):** Which stores dominate the target user base; unblocker-vs-DIY posture; managed scraping API selection.

Phases with standard patterns (skip research-phase):
- **Phase 1 (viewer):** r3f/drei orbit/lighting/loading patterns are extremely well documented; only the budget numbers carry over from pitfalls research.
- **Phase 4 (size extraction):** Single structured-output LLM call with a zod schema — established AI SDK pattern; model choice made at build time.
- **Phase 6 (library/share):** Standard CRUD + opaque-token public routes; no novel patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | All versions from npm registry; Meshy/Tripo/Inngest pricing from official pages fetched directly; a few MEDIUM items (R2 free tier, Better Auth API surface, exact Flash model) |
| Features | MEDIUM | Competitor landscape well-documented from public materials; closed B2B tools inferred; "URL import is unclaimed" is a negative claim (repeated null results) — flagged for ongoing validation |
| Architecture | MEDIUM-HIGH | API contracts verified from official docs; binding technique verified across academic + practitioner sources; queue patterns from ecosystem consensus |
| Pitfalls | MEDIUM | API behavior multi-source verified; some figures (70% non-manifold rate, ~15 credits/generation) single-source or community-reported; cost/queue figures must be re-verified when API keys are created |

**Overall confidence:** MEDIUM-HIGH. Decisions are well-supported; the dominant unknown is empirical — how Tripo/Meshy actually perform on *garments* — which no amount of desk research resolves. The roadmap must front-load an A/B quality evaluation with a real eval set.

### Gaps to Address

- **Garment-quality of 3D-gen output:** unresolvable without live testing. Handle via: eval set of 10–20 garment photos built in Phase 3, Tripo-vs-Meshy A/B, tracked pass rate.
- **Provider signup credits and webhook parameters:** verify at account creation (Phase 3 kickoff checklist); don't design around Meshy webhooks.
- **Background-removal quality on garments:** in-phase experiment in Phase 2; fallback decision tree already documented (skip removal → rembg microservice).
- **Current Gemini Flash model name/pricing:** select newest Flash-tier model at implementation; the AI SDK makes this a one-line swap.
- **Tripo multiview endpoint credit cost (and Meshy's):** confirm at integration; both fall in the same ~$0.20–0.35 band regardless.
- **Target store landscape for URL import:** gather failed-scrape/per-domain telemetry from Phase 6 onward before investing in breadth.
- **JS-rendered stores:** if scraping fails broadly, add a managed scraping API behind the existing `fetchProduct(url)` interface — LLM step unchanged.

## Sources

Full source lists with confidence ratings are in each research file. Aggregated highlights:

### Primary (HIGH confidence)
- docs.meshy.ai — image-to-3D API contract, statuses, expiring signed URLs, credit pricing (fetched directly)
- developers.tripo3d.ai — task API, 20/30-credit pricing, $0.01/credit, webhook + batch query
- inngest.com/pricing — free tier (50k executions, 5 concurrent steps), Pro $99/mo
- npm registry — all package versions
- style3d.ai, arXiv 1703.04454 / 2103.06871 — garment fitting "outside term" collision formulation

### Secondary (MEDIUM confidence)
- three-mesh-bvh (GitHub) — closest-point acceleration standard
- Vercel/Upstash/inngest ecosystem guides — background-job platform trade-offs
- Proxyway/Proxycove — scraping ban rates (70–90%) and API success testing
- Sketchfab viewer/sharing standards; VNTANA garment format standards
- three.js discourse/GitHub — polygonOffset, z-fighting, SkinnedMesh binding

### Tertiary (LOW confidence — validate at build time)
- ~70% non-manifold rate in raw AI outputs (single blog source)
- Meshy ~200 free credits/mo, ~15 credits/generation (community-reported)
- Tripo ~2,000 signup credits (third-party claim)
- "Auth.js merged into Better Auth" claim — circulating in low-quality results; do not rely on it

---
*Research completed: 2026-09-29*
*Ready for roadmap: yes*
