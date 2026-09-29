<!-- GSD:project-start source:PROJECT.md -->

## Project

**Clothing 3D Fitting Viewer (working name)**

A web app for online clothing shoppers. The user provides two photos of a garment (front + back) from any clothing store — by upload or by pasting a product URL — plus the product description. The app generates a real 3D model of the garment, dresses it on an adjustable mannequin (slim → plus-size), and lets the user orbit it in 360°. AI extracts sizing (category, region, S–XXL, measurements) from the description so the garment displays at true-to-size scale on the chosen body type.

**Core Value:** A shopper can see a real garment they're considering, in 3D on a body like theirs, and judge how it would look and fit — from just two photos and a description.

### Constraints

- **Tech stack**: Next.js + react-three-fiber web app — locked from brainstorm for fastest test/iteration
- **3D generation**: third-party API first (Meshy or Tripo) — self-hosting deferred; API cost ~$0.30/garment
- **Cost**: no paid keys owned yet; free tiers must carry development; per-garment cost makes quota enforcement necessary before public launch
- **Async processing**: garment generation takes minutes — must be a background job with status polling, never a blocking request
- **Fit realism**: pseudo-fit only in v1 (garment vertices bound to mannequin surface + collision push-out); extreme body sizes will hug like spandex — acceptable, communicate honestly

<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->

## Technology Stack

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Next.js | 16.3.x (locked) | App framework | Already locked. App Router + Server Actions cover API surface; no separate backend needed. |
| React / react-three-fiber | 19.3 / 9.8.x (locked) | 3D viewer | R3F v9 is React 19-native; drei 10.x provides OrbitControls, useGLTF, loaders out of the box. |
| PostgreSQL (Neon) | 17 | Primary database | Free tier (autosuspend, ~0.5 GB) carries development; branch-per-env for staging; Postgres is required by auth + quota counting anyway. |
| Drizzle ORM | 0.45.x | ORM / schema / migrations | TypeScript-native, no codegen step, tiny edge footprint, first-class Neon driver. Better Auth ships an official Drizzle adapter, so auth tables and app tables live in one schema. |
| Better Auth | 1.7.x | Email/password auth + email verification | Built-in `emailAndPassword.requireEmailVerification`, pluggable `sendVerificationEmail` hook (wire to Resend directly), framework-native (no external auth API call per request). Fastest path to the exact requirement set: signup, verify via Gmail, sessions. |
| Resend | 6.x + React Email 6.x | Transactional email | Verification emails only (per PROJECT decision). React Email components + `resend` SDK; free tier ~3,000 emails/mo is far above verification-only volume. (Confidence: MEDIUM on exact free-tier numbers — verify at signup.) |
| Inngest | 4.21.x | Durable background jobs | The 3D-generation pipeline (fetch photos → optional BG removal → call 3D API → poll minutes → process GLB → upload → notify) maps 1:1 to Inngest steps: `step.run()` for each stage, `step.sleep()` between polls, automatic retries, and resumption after serverless cold starts. No Redis, no worker process, no always-on cost. Free tier: 50k executions/mo, 5 concurrent steps — one garment job ≈ 7–9 executions, so thousands of garments/mo fit free. (Verified from inngest.com/pricing.) |
| Tripo API | — (pay-as-you-go) | Image-to-3D generation (primary) | Official pricing: multiview/image-to-3D = **20 credits (no texture) / 30 credits (standard texture)**, 1 credit = $0.01 → **~$0.20–0.30 per garment**, pure pay-as-you-go (no subscription needed for dev). Multiview endpoint natively fits the front+back photo model. Community consensus rates Tripo quality high on complex assets (garments qualify). (Verified from developers.tripo3d.ai/en/pricing.) |
| Meshy API | — (credit packs) | Image-to-3D generation (fallback / A-B) | Official pricing: **meshy-7.1 / meshy-6 = 20 credits mesh-only, 30 credits with 2K textures, 35 with 8K**; meshy-6-lite at 5/15 credits for cheap experiments. Same ~$0.20–0.35/garment band. Create-task → poll (or SSE stream) → download GLB; returns GLB/FBX/OBJ/USDZ plus `pre_remeshed_glb`. (Verified from docs.meshy.ai.) Wrap both behind one provider interface; A-B a few garments early and commit to the better garment output. |
| Gemini Flash-class via @ai-sdk/google | current Flash tier | LLM size extraction | The task (extract category/region/sizes/measurements from a product description → JSON) is trivial for any frontier model; use Vercel AI SDK `generateObject` with a Zod schema and the cheapest current Flash-tier model. Per-call cost is fractions of a cent (a few KB of text). Swap models freely — the AI SDK abstracts the provider. (Model-name confidence: MEDIUM — pick the newest Flash-tier model at build time.) |
| Vercel | — | Hosting (Next.js app) | Free Hobby tier covers dev/launch. Critically: the ONLY long-running work (3D gen) lives in Inngest, not in Vercel functions, so Vercel's 300s Hobby function cap is irrelevant. Hobby cron limits (1/day) also don't matter — Inngest replaces cron. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| @gltf-transform/core + functions + cli | 4.5.x | Server-side GLB post-processing | After generation: normalize orientation/scale, vertex-count simplification, Draco/meshopt compression, texture resize. Pure npm — no Python runtime needed. Run inside an Inngest step. |
| @imgly/background-removal-node | 1.4.x | Optional background removal pre-generation | ONNX-based, runs in Node inside an Inngest step, $0 marginal cost. Use it to clean garment photos before sending to Tripo/Meshy (cleaner segmentation = better multiview results). **Flag:** output quality on garments must be validated in phase — if it underperforms, alternatives are a tiny rembg (Python) microservice or skipping removal entirely (both APIs segment foreground internally). |
| zod | 4.6.x | Schema validation + LLM structured output | One schema language for API validation, env validation, and `generateObject` — the size-extraction schema is defined once and typed end-to-end. |
| @t3-oss/env-nextjs | 0.13.x | Env var validation | Validate the (many) API keys — Tripo/Meshy, LLM, Resend, R2, Inngest — at boot instead of at runtime failure. |
| @tanstack/react-query | 5.104.x | Client data + job-status polling | Poll the garment job status endpoint (`refetchInterval` while pending) and invalidate on completion; pairs with the in-app "garment ready" notification. |
| zustand | 5.x | Viewer state | Mannequin body params (slim→plus), selected size, orbit state — small store co-located with the R3F canvas without re-rendering React on every frame. |
| @aws-sdk/client-s3 + s3-request-presigner | 3.x | Cloudflare R2 access (S3-compatible) | Presigned uploads for user photos; server-side put for generated .glb files. |
| cheerio | 1.2.x | Product-URL scraping (v1) | Fetch store page HTML, extract OG/product images + description text, then hand to the LLM. Works for most stores; JS-rendered stores need the phase-researched fallback (managed scraping API). |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| inngest CLI (`npx inngest-cli dev`) | Local job dev server | Runs Inngest locally with the Dev UI; step-by-step replay of failed garment jobs. |
| drizzle-kit | Migrations | `drizzle-kit generate` + `migrate` in CI. |
| gltf-transform CLI | Manual GLB inspection/optimization | `npx @gltf-transform/cli inspect model.glb` during development of the mesh pipeline. |
| React Email dev (`email/dev`) | Preview verification emails | Local preview before wiring Resend. |

## Installation

# Core app

# 3D pipeline + AI

# Dev dependencies

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| Drizzle ORM | Prisma 7/8 | If the team strongly prefers Prisma's DSL; Better Auth supports it too. Prisma 7 dropped the Rust engine (faster), but codegen + heavier runtime buys nothing here. |
| Better Auth | Clerk | If you want zero auth code and accept per-MTU pricing + external API latency on every session check; revisit if email deliverability or auth edge cases eat dev time. |
| Better Auth | Auth.js (NextAuth v5) | Only if you need its specific OAuth provider matrix. v5 sat in beta for years; Better Auth is where Next.js auth ecosystem energy currently is. (A claim circulating that Auth.js was merged into Better Auth appeared only in low-quality search results — **unverified, do not rely on it**.) |
| Inngest | BullMQ + Redis on a Railway worker | If you later need true long-running compute (e.g., self-hosted TRELLIS per the PROJECT evolution path) or Python in the hot path. BullMQ cannot run on Vercel (no persistent worker). |
| Inngest | Trigger.dev v4 | Very close second — also durable steps, TS-native, free tier. Inngest edges it on the free tier's execution allowance and no-container workflow. Either works; pick one and don't mix. |
| Neon Postgres | Supabase (Postgres + its auth/storage) | If you'd rather adopt Supabase Auth and Storage wholesale — but that replaces Better Auth + R2 decisions and couples the stack to one vendor. |
| Cloudflare R2 | UploadThing | Only for the photo uploads DX; its per-file caps and 2 GB free tier make it wrong for .glb serving. |
| Tripo (primary) | Meshy (primary) | If Meshy output quality wins the early A-B on garments, or if Meshy's multiview endpoint (unverified credit cost — confirm at integration) beats Tripo's. Both are ~same price; the interface wrapper makes this a one-file swap. |
| @imgly/background-removal-node | rembg (Python microservice) | If imgly quality fails validation on garments — worth the second runtime only then. Both APIs' internal segmentation may make removal skippable entirely. |
| cheerio scraping | Firecrawl / ScrapingBee API | When target stores are JS-rendered SPAs. Phase-research which stores the user base actually pastes URLs from. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| A Python worker (Celery/FastAPI + Redis) for v1 | The entire generation pipeline is HTTP orchestration (call API, poll, download, upload) — no Python-native compute is required. A second runtime doubles deploy/ops surface for zero benefit. | Inngest steps in TypeScript; @gltf-transform for GLB work |
| BullMQ / Redis queues on Vercel | Serverless functions can't host a persistent worker; BullMQ consumers need a long-lived process. | Inngest (or BullMQ only if you also move to a Railway worker) |
| Vercel Cron + chained serverless invocations to "simulate" the job loop | Hobby cron runs once per day and functions cap at 300s; polling a minutes-long 3D job this way is fragile and burns invocations. | Inngest `step.sleep()` between status polls |
| AWS S3 for garment .glb serving | $0.09/GB egress dominates cost; shareable public 360° links mean heavy download traffic by design. | R2 (zero egress) |
| UploadThing for .glb storage | Per-file size caps (~16–50 MB classes) risk blocking real garment meshes; free tier is 2 GB. | R2 with presigned uploads |
| Self-hosted TRELLIS / Hunyuan3D in v1 | Already an explicit out-of-scope decision; GPU hosting cost/ops is unjustified pre-validation. | Tripo/Meshy API (~$0.20–0.30/garment) |
| SMPL/SMPLify body models for the mannequin | Research-only license, incompatible with a public commercial web app. | CC0/purchasable base mannequin mesh + client-side morph targets |
| pymeshlab / trimesh (Python) in v1 | Nothing in the v1 pipeline needs Python mesh ops; the APIs return retopologized GLB and @gltf-transform handles scale/simplify/compress in npm. | @gltf-transform; revisit only if server-side remeshing proves necessary |
| Per-user quota via Redis counters | Extra infra + consistency risk for a count that Postgres does atomically. | `SELECT count(*)` on the garments table with a `(user_id, created_at)` index, checked inside the creation transaction |
| Supabase Auth / Clerk when the rest is self-owned auth | Mixing hosted auth with Better Auth-style DB sessions splits identity data across systems. | Better Auth with Drizzle adapter — one database, one source of truth |

## Stack Patterns by Variant

- Keep the provider interface; swap Tripo for Meshy `meshy-7.1` (30 credits textured) or `meshy-6-lite` (15) for dev iteration to stretch credits.
- Note Meshy task objects expose SSE streaming (`/stream`) — nicer than polling if you want live progress in the UI.
- Route dev/test generations through `meshy-6-lite` / Tripo no-texture (5–20 credits, ~$0.05–0.20) and reserve textured quality for real users.
- Add per-user concurrent-job limit (1) via Inngest throttling keys so one user can't queue 50 garments.
- First lever: raise quality bar / queue position (jobs are minutes-long anyway — users expect a queue).
- Real lever: Pro at $99/mo, or migrate the worker to a Railway container with BullMQ — the provider/job logic ports because it's already step-shaped.
- Add a managed scraping API (Firecrawl class) behind the same `fetchProduct(url)` interface; the LLM extraction step is unchanged.
- Simplest fix first: skip removal (both 3D APIs segment internally) and require clean photos via upload UI guidance.
- Only if output quality demonstrably improves with clean inputs: add a rembg HTTP microservice on Fly.io (one small always-on box, ~$3–5/mo).

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|-----------------|-------|
| @react-three/fiber 9.x | React 19 + Next.js 16 | R3F v9 dropped React 18; do not downgrade React to satisfy old tutorials. |
| @react-three/drei 10.x | three 0.18x + R3F 9 | Pin three and drei together; drei releases track three's minor cadence. |
| ai (AI SDK) 7.x | zod 4 | AI SDK uses Standard Schema — zod 4 works natively; older tutorials showing `zod-to-json-schema` shims are obsolete. |
| better-auth 1.7.x | drizzle-orm 0.4x | Use the official Drizzle adapter and generate auth tables into the same schema as app tables. |
| @gltf-transform 4.x | three 0.18x output | v4 changed several function signatures vs v3 tutorials — follow v4 docs, not older blog posts. |
| inngest 4.x | Next.js 16 App Router | Serve via `serve()` in a route handler (`/api/inngest`); use `step.sleep()` for the minutes-long poll loops. |
| @imgly/background-removal-node | Node 20+ | Downloads ONNX model weights at first run — pre-warm in the job step or bundle/serve weights yourself for deterministic cold starts. |

## Per-Garment Cost Model (quota design input)

| Item | Cost | Source confidence |
|------|------|-------------------|
| 3D generation (textured, front+back multiview) | ~$0.20–0.30 (20–30 credits × $0.01) | HIGH (both official pricing pages) |
| LLM size extraction (Flash-class, ~2 KB in / small JSON out) | <$0.001 per call | MEDIUM (pricing class, not exact figure) |
| Background removal (imgly, self-run) | ~$0 compute (seconds of Inngest step time) | HIGH |
| GLB storage (5–20 MB) + serving | ~$0 on R2 free tier (10 GB ≈ 500–2,000 garments stored; zero egress) | HIGH |
| Email verification | ~$0 (Resend free tier) | MEDIUM |

## Sources

- docs.meshy.ai/en/api/pricing — per-model credit costs (meshy-7.1/6/6-lite: 20/30/35 credits) — fetched directly, HIGH
- docs.meshy.ai/en/api/image-to-3d — task create/poll/stream flow, GLB outputs, `pre_remeshed_glb` — fetched directly, HIGH
- developers.tripo3d.ai/en/pricing — image-to-3D 20/30 credits, $0.01/credit, pay-as-you-go, multiview pricing — fetched directly, HIGH
- inngest.com/pricing — free tier 50k executions, 5 concurrent steps, Pro $99/mo — fetched directly, HIGH
- npm registry (`npm view`) — all package versions in this document — HIGH
- vercel.com/docs/functions (duration/cron limits) — via search snippets of official docs, MEDIUM
- Cloudflare R2 free tier (10 GB, zero egress) — multiple secondary sources agreeing, MEDIUM
- Tripo ~2,000 signup API credits — third-party claim only, LOW — verify at account creation
- Meshy free/new-account API credits — not confirmed on fetched pages, LOW — verify at account creation
- Better Auth email-verification API surface — npm version HIGH; API details from training data + docs, MEDIUM
- Gemini/OpenAI current model names and prices — MEDIUM/LOW — select newest Flash-tier model at implementation time

<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->

## Conventions

- **Push to GitHub after every completed phase or major area of work** — `git push origin master` (repo: github.com/gideonpurnomo/clothingmodel, public). User-requested progress publishing; applies at phase completion and milestone boundaries.
- All TypeScript, no Python in v1 (per stack research).
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->

## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->

## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->

## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:

- `/gsd:quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd:debug` for investigation and bug fixing
- `/gsd:execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->

<!-- GSD:profile-start -->

## Developer Profile

> Profile not yet configured. Run `/gsd:profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
