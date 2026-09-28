# Clothing 3D Fitting Viewer (working name)

## What This Is

A web app for online clothing shoppers. The user provides two photos of a garment (front + back) from any clothing store — by upload or by pasting a product URL — plus the product description. The app generates a real 3D model of the garment, dresses it on an adjustable mannequin (slim → plus-size), and lets the user orbit it in 360°. AI extracts sizing (category, region, S–XXL, measurements) from the description so the garment displays at true-to-size scale on the chosen body type.

## Core Value

A shopper can see a real garment they're considering, in 3D on a body like theirs, and judge how it would look and fit — from just two photos and a description.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] User can create an account with email/password and verify their email (received via Gmail)
- [ ] User can log in, log out, and stay logged in across sessions
- [ ] User can upload guided front + back photos of a garment
- [ ] User can paste a store product URL and the app fetches the product images + description automatically
- [ ] System turns the two photos into a 3D garment model via a 3D-generation API (background job with status)
- [ ] User can view the generated garment on a mannequin with full 360° orbit controls
- [ ] User can adjust the mannequin body type (slim → plus-size) via presets/sliders, and the garment follows the body (pseudo-fit)
- [ ] System extracts size category, region, available sizes (S–XXL) and measurements from the product description via LLM, with standard size chart fallback
- [ ] Garment is scaled true-to-size on the mannequin based on extracted measurements
- [ ] User can select garment size (S–XXL) and see the display update
- [ ] User has a saved garment library tied to their account
- [ ] Free tier quota is enforced (e.g. N garments/month per account)
- [ ] User receives an in-app notification when their garment finishes processing
- [ ] User can share any of their garments via a public read-only 360° view link
- [ ] App is deployed publicly and usable by real people

### Out of Scope

- Real cloth physics simulation (drape) — research-grade (Neural Tailor / Sewformer territory); v3 destination, not v1
- Stripe subscription / payments — v2; v1 builds the quota rails it will plug into
- Native mobile apps — web first, mobile later
- 2D virtual try-on (diffusion image of user wearing garment) — possible bolt-on later; not the core 360° vision
- Custom body scans / measuring the user from their own photos — blend-shape mannequin presets only in v1
- Self-hosted 3D generation (TRELLIS/Hunyuan3D) — revisit after v1 validates demand; Meshy/Tripo API first

## Context

- Origin: brainstorm session (2026-09-25) converged on the "Option C" MVP — guided front + back photos for high-fidelity 3D without AI guessing what the back looks like
- Primary users: online clothing shoppers who want fit/look confidence before buying from any store
- **No API keys exist yet** — Meshy/Tripo (3D generation, ~$0.20–0.40/garment) and an LLM provider (size extraction) accounts must be created during development. Phases requiring keys must flag this clearly.
- The 3D generation pipeline is asynchronous (minutes per garment) — queue, job status UI, and notifications are core, not extras
- Prior agreed build order: viewer skeleton → upload + 3D-gen worker → LLM size extraction → body morphs + binding → polish
- Sizing gotcha already identified: sizing is regional (Asian "L" ≈ US "M") — extract region and normalize to centimeters internally

## Constraints

- **Tech stack**: Next.js + react-three-fiber web app — locked from brainstorm for fastest test/iteration
- **3D generation**: third-party API first (Meshy or Tripo) — self-hosting deferred; API cost ~$0.30/garment
- **Cost**: no paid keys owned yet; free tiers must carry development; per-garment cost makes quota enforcement necessary before public launch
- **Async processing**: garment generation takes minutes — must be a background job with status polling, never a blocking request
- **Fit realism**: pseudo-fit only in v1 (garment vertices bound to mannequin surface + collision push-out); extreme body sizes will hug like spandex — acceptable, communicate honestly

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Option C: guided front + back photos | Highest fidelity without AI imagining the back; both sides are real | — Pending |
| Web platform (not mobile) | Fastest to test and iterate | — Pending |
| Meshy/Tripo API now, self-hosted TRELLIS later | Fast MVP; own infra once demand is validated | — Pending |
| Blend-shape mannequin (slim → plus) + pseudo-fit binding | Body morphs without research-grade body models or cloth sim | — Pending |
| LLM structured size extraction + standard chart fallback + regional normalization | Most tractable AI feature; text problem, not 3D problem | — Pending |
| Accounts required from day one | User data, garment libraries, quota enforcement, freemium future | — Pending |
| Quota enforced in v1, payments in v2 | Free tier rails exist early; Stripe scope deferred | — Pending |
| In-app notifications (not email) for garment-ready | User preference; email only for verification | — Pending |
| Public share links in v1 | Organic growth, low extra effort | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-28 after initialization*
