# RavelPlus (working title) — v1 Implementation Plan

> **Revision note (2026-07-18):** This plan supersedes the earlier "RavelPlus as a Ravelry web front-end" plan (see git history for the original). The product direction pivoted at the July 12 founder sync — see `docs/Product Requirements.md` (PRD v0.2). The app is now a standalone, craft-adaptive fiber-arts companion (stash / tools / projects / pins / counters) built **React Native + Expo, mobile and tablet first, iOS first**, with offline-first sync on Supabase. The Ravelry API's role shrank from live data source to one-time import. The app name is still open (PRD §10 #1); "RavelPlus" remains the working title and repo name.

## Context

Per the PRD: a universal fiber-arts companion app for crocheters and knitters — one place to manage stash, tools, projects, inspiration (pins), and in-progress counters, with free offline-first cross-device sync. Not a Ravelry replacement; the app Ravelry users wish they had alongside it. MVP scope is PRD §5.1; personas are PRD §3.

**Collaborators:**
- **Diana** — lead software engineer, owns all technical decisions
- **Sheryl** — senior UX designer, owns product/UX decisions and the PRD

---

## Cost Discipline

**Rule (PRD §4.4):** All tooling and services run on free tiers; the CI/CD pipeline target is $0. Paid services require an explicit joint decision.

### Free-tier audit (as of 2026-07 — re-verify each limit at adoption time)

| Service | Free tier | Notes |
|---|---|---|
| Supabase (DB + Auth + Storage + Realtime + Edge Functions) | 500 MB DB · 1 GB storage · 5 GB egress/mo · 50K MAU auth · 500K edge invocations/mo · 200 concurrent realtime connections | Covers MVP scale. Pauses after 1 week idle (fine during active dev). **1 GB storage is the first real ceiling** — see image storage note below. |
| Expo EAS (Build / Submit / Update) | ~30 cloud builds/mo, EAS Submit included, EAS Update ~1K MAU | Enough for MVP cadence if we don't build on every commit. **Verify current limits before Phase 0.** |
| GitHub (repo + Actions) | 2,000 Actions min/mo on private repos | ⚠️ macOS runners bill 10×, Windows 2× — stay on `ubuntu-latest` only. |
| Sentry (React Native SDK) | 1 user, 5K errors/mo | Only Diana needs dashboard access. |
| RevenueCat (subscriptions) | Free up to $2.5K/mo tracked revenue | Standard choice for RN in-app subscriptions; free tier far exceeds MVP needs. |
| Ravelry API | Free (Pro account, no charge); no published rate limit — we self-impose 1 req/sec | ✅ R2 resolved (2026-07-18): **no commercial fee exists** in the current license agreement — see `plans/ravelry-api-research.md`. Real constraint is discretionary revocation risk, not cost; import stays a non-load-bearing feature. |
| Jest, RN Testing Library, Maestro, Drizzle, Tailwind-equivalent (NativeWind) | Open source | Safe. |

### ⚠️ The two unavoidable costs (flagged, not free)

1. **Apple Developer Program — $99/year.** ✅ **RESOLVED (2026-07-18): approved — Diana pays out of pocket.** Required for TestFlight and App Store distribution. Early development still runs $0 via Expo Go on physical iPhones; enrollment becomes necessary **before the first TestFlight build goes to Sheryl** (no rush to enroll earlier).
2. **Google Play — $25 one-time** when the Android fast-follow ships.

Everything else stays $0. **GCP/Firebase is dropped entirely** — no Next.js server means no Cloud Run, no Blaze plan, no GCP credit card. Server-side logic lives in Supabase Edge Functions.

### Image storage pressure point (feeds PRD §10 #6/#14 — ✅ researched 2026-07-21, see `plans/image-storage-cost-research.md`)

**Per-GB cost is a non-issue.** At ~200 KB/compressed image, even a 10,000-image extreme case costs ~$0.05/month; fleet-wide at 10K users averaging 30 MB each it's ~$7/month. The proposed >5-images-per-project paywall gate can't be justified as cost recovery on this basis — if kept, it's a product/upsell decision, not a cost-control one.

**The real cost step is a platform tier ceiling, not per-GB pricing:** Supabase free tier = 1 GB total storage (~5,000 compressed images across all users) — fine for the friends-scale beta. The next step is **Supabase Pro at $25/month flat** (includes 100 GB), which is the actual first infrastructure-spend trigger, arriving around ~50 heavy users. Alternative if/when that's reached: move images specifically to **Cloudflare R2** (10 GB free forever, zero egress fees, $0.015/GB after) — a contained migration since images are addressed by storage path.

Implications:
- Client-side compression/resizing before upload remains **mandatory from day one** (e.g., 1600px long edge, ~75% JPEG) — this is what keeps the per-GB math negligible in the first place.
- The free-tier per-user image cap (open decision #6) should be set generously for product-shape/UX reasons — cost does not meaningfully constrain the number.

---

## Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| App framework | React Native + **Expo** (latest SDK), TypeScript strict | PRD-resolved decision. Expo gives managed builds (EAS), OTA updates, and the OS-integration path (widgets, Live Activity, Watch) for Phase 3. |
| Web target | **Enabled from day one** (react-native-web) | Two payoffs: (1) `expo start` → `w` gives a browser dev loop with React DevTools — Diana's primary daily surface for UI/CRUD work (she's web-experienced and on Windows with no iOS Simulator); (2) Phase 2 desktop web becomes "polish and host what already runs," not a new build-out. |
| Navigation | expo-router | File-based routing; deep links for free (needed for PRD §6.8 filter-state deep-linking). |
| Styling | NativeWind (Tailwind for RN) + custom design tokens | Keeps the Tailwind mental model; Sheryl's tokens map to a theme config. Component styling stays fully owned (no heavy UI kit). |
| Local database | **SQLite via expo-sqlite, accessed through Drizzle** | Offline-first requires a real local DB. Drizzle's expo-sqlite driver gives typed queries + `useLiveQuery` reactivity. |
| ORM / migrations | **Drizzle, double duty** | One set of TypeScript schema definitions; drizzle-kit manages Postgres migrations (remote) and SQLite migrations (on-device). Maximizes schema symmetry between local and cloud. |
| Backend | Supabase (Postgres + Auth + Storage + Realtime + Edge Functions) | PRD-recommended. RLS for per-user isolation; Realtime for live cross-device updates; Storage for images; Edge Functions for the Ravelry import worker and RevenueCat webhook. |
| Auth | Supabase Auth — Sign in with Apple, Google, email+password | PRD §6.10. Apple sign-in is mandatory on iOS when other social logins are offered. |
| Sync engine | 🔬 **Research spike R1 — not yet locked** | Candidates and criteria below. The PRD requires better-than-document-LWW conflict handling; this is the highest-risk technical decision in the plan. |
| Subscriptions | RevenueCat + StoreKit | Receipt validation, entitlements, and cross-platform readiness without building billing infrastructure. |
| Error tracking | Sentry (`@sentry/react-native`) | Free tier; wired in Phase 1. |
| E2E testing | Maestro | Free, YAML-driven, first-class Expo support. (Playwright returns in Phase 2 for desktop web.) |

**Why this stack:** The domain is relational (stash ↔ projects ↔ pins ↔ tools are many-to-many; auto-population joins against a shared yarn catalog), so Postgres + a mirrored local SQLite models it natively. Expo is the only realistic path to iOS-first development from a Windows machine (below). Everything server-side fits in Supabase, collapsing the old two-cloud (GCP + Supabase) footprint into one.

### Phase 2 web/desktop readiness — day-one rules

The Phase 2 web app is intended to be **feature-rich and desktop-native in feel, not a scaled-up phone UI**. One codebase constrains shared *logic*, not shared *layouts* — Metro resolves `component.web.tsx` per platform, web files may use any DOM/web library, and expo-router allows web-only routes. Candidates for web-divergent presentation: multi-pane stash browse (list + detail side by side), drag-and-drop project kanban, always-visible filter facets, bulk edit / CSV import-export, hover previews on the pins masonry, keyboard shortcuts, printable views.

Rules that start in Phase 1 so this stays cheap:

1. **Web target enabled in the scaffold** and kept building (CI runs the web bundle) — web never bit-rots into a Phase 2 rescue project.
2. **All logic lives in shared hooks/lib** — platform-divergent files are thin presentation over the same queries/sync/registry. Enforced in code review.
3. **Every library adoption gets a web-compat check** before it lands (e.g., FlashList's web support is newer — verify at adoption). A native-only library is acceptable only behind a platform file with a web fallback.
4. The MVP's phone (bottom tabs) vs tablet (sidebar) split — PRD §8 — establishes the "same data, different shell" architecture early; web becomes a third shell, not a new concept.

### Development environment reality (Diana is on Windows)

- **No iOS Simulator on Windows.** The daily write-code-see-result loop runs **in the browser** (web target, hot reload, React DevTools) — a familiar web DX — with **Expo Go on a physical iPhone** as the continuous native check (and an Android emulator for layout spot-checks). The browser never substitutes for on-device verification; it complements it.
- **iOS binaries are built in the cloud by EAS Build** — no Mac required for building or submitting to TestFlight.
- Expo Go covers the MVP feature set (expo-sqlite, supabase-js, expo-camera for the barcode spike, expo-image-picker). The switch to a **custom dev build** happens only when a config-plugin/native module forces it — at that point device installs need the paid Apple account.
- Consequence: **iOS-specific bugs surface on-device, late.** Mitigation: test on the physical iPhone continuously, not at phase-end.

---

## Repository Structure

```
ravelplus/
├── plans/                      # Plans and PRDs
├── docs/                       # Tech-stack reference (resources.md — needs matching overhaul)
├── app/                        # expo-router routes
│   ├── (auth)/                 # Sign-in screens
│   ├── (tabs)/                 # Stash / Tools / Projects / Pins / Profile
│   │   ├── stash/
│   │   ├── tools/
│   │   ├── projects/           # includes [id] detail w/ counters, notes, session log
│   │   └── pins/
│   └── import/                 # Ravelry import flow
├── components/
│   ├── ui/                     # Owned primitives (buttons, sheets, combobox, etc.)
│   └── forms/                  # Craft-adaptive form system
├── lib/
│   ├── supabase.ts             # Supabase client (auth-aware)
│   ├── sync/                   # Sync engine (outcome of spike R1)
│   ├── crafts/                 # Craft-adaptive field registry (see below)
│   └── purchases.ts            # RevenueCat wrapper
├── db/
│   ├── schema.ts               # Drizzle schema — single source of truth
│   ├── local.ts                # expo-sqlite Drizzle client
│   └── migrations/             # drizzle-kit output (postgres + sqlite)
├── supabase/
│   ├── functions/              # Edge Functions: ravelry-import, revenuecat-webhook
│   └── migrations/             # RLS policies, triggers (SQL applied via supabase CLI)
├── e2e/                        # Maestro flows (.yaml)
├── eas.json                    # EAS Build profiles (dev / preview / production)
└── package.json
```

---

## Data Model (Postgres, mirrored to SQLite)

**Supabase-managed:** `auth.users`. All user tables reference it via `user_id uuid references auth.users(id) on delete cascade` and carry RLS `using (auth.uid() = user_id)`.

**User data tables (per-user, synced):**

- `yarn_stash` — brand, name, color_name, dye_lot, weight (enum lace0–jumbo7), grams_per_skein, yards_per_skein, skeins (default 1), remaining_grams, gauge_stitches / gauge_unit, hook_needle_size_mm, care_methods (join table), country_of_origin, source_url, notes, ravelry_yarn_id (nullable — provenance from Ravelry lookup or import)
- `yarn_stash_fibers` — (stash_id, fiber, percent) — the "80% wool / 20% nylon" multi-select
- `tools` — name, type (enum incl. needle subtype), brand, size_mm, material, notes
- `projects` — name, craft_type, designer, pattern_url, status (todo/in_progress/finished), difficulty, start_date, finish_date, notes
- `project_yarns`, `project_tools` — join tables, `on delete cascade` from the stash/tool side (PRD: deleting a stash item removes references)
- `session_notes` — (project_id, body, created_at) — the timestamped work journal
- `counters` — (project_id, name, count, target_count) — synced in real time
- `pins` — name, designer, source_url, craft_type, difficulty, notes, pushed_to_project_id (nullable — the "pushed to projects" marker)
- `pin_yarns`, `pin_tools` — join tables
- `entity_images` — (owner table + id, storage_path, position) — one image pipeline for all entities; caps (3 stash / 5 project / 5 pin) enforced in app + DB trigger
- `attachments` — project files/PDFs (storage_path, filename, mime)
- `profiles` — display name, settings, theme override
- `subscriptions` — entitlement state, written **only** by the RevenueCat webhook Edge Function (service role)

**Shared/global tables (read-only to users, no user_id):**

- `yarn_catalog` — brand + yarn name → default weight, fiber makeup, yards/grams per skein, known colorways. Powers the auto-population combobox (PRD §6.2) alongside **live Ravelry lookup** (decision 2026-07-18, see `plans/ravelry-api-research.md` §Yarn lookup design): the combobox merges our catalog with user-initiated `yarns/search` results; picking a Ravelry match writes attributes into the *user's own stash row* (+ `ravelry_yarn_id` provenance) — **never into the shared catalog**. Catalog rows come only from user-created entries and manual/open-data seeding (community contribution later). ⚠️ Bulk-seeding or caching Ravelry data in the catalog is prohibited (license clause 1i) unless Ravelry blesses it via api@ravelry.com. Ravelry lookup is an online-only enhancement — offline stash-add degrades to manual fields (flag the degraded state to Sheryl for design).

**Sizes are stored normalized (mm)** and formatted per craft at the display layer — US letter hooks vs US-number needles is presentation, not storage. This is what lets a future craft type ship without schema changes (PRD §6.1).

**Free-tier project cap (15):** enforced by a Postgres trigger checking `subscriptions` entitlement on insert — never client-only. Client mirrors the check for friendly UX.

**Sync metadata:** every synced table carries `updated_at`, `deleted_at` (soft delete for offline tombstones), and whatever change-tracking spike R1 dictates (per-field timestamps live in a companion changes table if we build custom).

---

## Offline-First Sync — Research Spike R1 (highest-risk decision)

PRD §6.9 requirements: local-first writes, background queue, realtime propagation, **per-field LWW merge** (not document-level), conflict notification on same-field collisions, images in the same pipeline, error-only sync surfacing.

**Candidates to evaluate (in order):**

1. **PowerSync** — purpose-built Postgres↔SQLite sync with Supabase integration; handles the queue, checkpoints, and reconnection. Check: free-tier/self-host terms, whether its conflict model can express per-field merge, Expo compatibility.
2. **Custom sync layer** on expo-sqlite + Drizzle: outbox table of field-level changes, push via supabase-js (RLS-enforced), pull via `updated_at` cursors, realtime channel for live nudges. Full control of the merge policy; most engineering effort; most honest fit to the PRD's conflict spec.
3. **WatermelonDB / Legend-State** — evaluated mainly to reject or confirm quickly; both need meaningful backend glue to reach the PRD's conflict spec.

**Decision criteria:** meets per-field merge · $0 at MVP scale · works in Expo (Go or dev build?) · **works on web** (expo-sqlite's WASM/OPFS support or a vendor web SDK — Phase 2 desktop web is committed, so a native-only engine is disqualifying) · RLS enforced on the sync path · handles images/attachments or coexists with a separate upload queue · battery/bandwidth sanity for the multi-device live-edit case (PRD flagged this research too).

**Spike deliverable:** a throwaway two-device demo syncing `counters` (smallest, highest-frequency entity) offline→online, plus a one-page decision writeup. **Timeboxed; blocks Phase 3, not Phases 0–2.**

**→ Full architecture explanation and step-by-step spike plan (candidate order, 9-scenario pass/fail test matrix, timebox): `plans/sync-architecture-r1.md`.**

---

## Server-Side Logic (Supabase Edge Functions)

No app server. Three functions:

| Function | Purpose |
|---|---|
| `ravelry-import` | OAuth handshake with Ravelry + the rate-limited (1 req/sec) import worker. Fetches stash/tools/projects, maps to our entities, returns a preview payload; commits on user confirm. Long imports run chunked with progress stored in an `import_jobs` table. |
| `revenuecat-webhook` | Receives entitlement events, writes `subscriptions` with the service role. |
| `delete-account` | App Store requirement — full account + data deletion. |

The Ravelry API key/secret live in Edge Function secrets — never in the app bundle.

---

## Craft-Adaptive Forms (PRD §6.1 — the differentiator)

Architecture: a **TypeScript field registry** in `lib/crafts/`, not per-craft DB schemas.

- `CraftType` union (`'crochet' | 'knitting'`) drives a registry: for each entity form, a typed config of which fields show, their labels ("hook size" vs "needle size"), unit formatters (mm → US letter vs US number), and option lists.
- Storage stays craft-neutral (normalized mm, generic counters); adaptation is entirely presentational + validation-level.
- Adding a craft type = adding a registry entry (+ any new option lists). Zero migrations. This satisfies the PRD's "new craft types without schema rewrites" constraint and is cheap to unit test.
- The dynamic-forms research item (PRD §10 #10) is about UX behavior (what changes when, stash-selection vs craft-selection interplay) — Sheryl's design spike; the registry architecture accommodates whatever she lands on.

---

## Monetization Infrastructure (MVP scaffolding)

- RevenueCat SDK + one subscription product (price TBD — PRD §10 #7). Entitlement: `plus`.
- Gates in MVP: project count > 15, image storage above free cap, saved filter sets. All enforced server-side (trigger/RLS) with client-side friendly messaging.
- Paywall screen is Sheryl's design; a placeholder ships behind a feature flag until pricing is decided.
- **Note:** in-app purchases don't work in Expo Go — the RevenueCat integration lands only once we've moved to a dev build, and real purchase testing needs TestFlight sandbox. Sequence this late (Phase 6).

---

## Testing Standards

### Unit — Jest + React Native Testing Library

| Target | Examples |
|---|---|
| Craft registry | field visibility/labels/formatters per craft; size conversions (mm↔US letter↔US number) round-trip |
| Sync logic (if custom) | outbox ordering, per-field merge, tombstone handling — pure functions, heavily tested |
| Stash intelligence | remaining-grams math, finished-project prompts |
| Hooks | filter/sort state + deep-link sync, counter logic |
| Ravelry import mapping | Ravelry JSON fixtures → our entities, mismatch warnings |
| Drizzle queries | against pglite / local SQLite |

Co-located `*.test.ts(x)`. No coverage %; logic-heavy modules get thorough coverage, snapshots are skipped.

### E2E — Maestro

Small critical-flow suite: sign in → add stash item → create project → link yarn → tap counter → force-offline edit → verify sync on reconnect (once R1 lands). Runs locally against a dev build + local Supabase (`supabase start`). In CI: Android emulator on `ubuntu-latest` only (KVM), **never macOS runners**. Keep under ~5 min; defer CI E2E until flows stabilize (Phase 4+).

### RLS verification

A dedicated test (SQL or supabase-js with two test users) asserting user A cannot read/write user B's rows on every user table. Runs in CI against `supabase start`.

---

## CI/CD

**GitHub Actions, `ubuntu-latest` only, $0.**

- **Every PR:** ESLint → `tsc --noEmit` → Jest → RLS test vs `supabase start`. ~4–6 min. Required status checks; `concurrency: cancel-in-progress`; cache npm + Supabase docker layers.
- **Builds:** EAS Build (Expo's cloud) — not Actions — on demand, not per-commit, to stay inside ~30 builds/mo. Profiles: `development`, `preview` (internal/TestFlight), `production`.
- **Distribution to Sheryl:** TestFlight via EAS Submit (post-$99 signoff). Before that, she previews via Expo Go + EAS Update channels ($0).
- **OTA:** EAS Update for JS-only fixes between store releases; store builds for anything native.

---

## Build Sequence

### Phase 0 — Design spec + decisions (Sheryl ∥ Diana)
- Sheryl: design spec — tokens (color/type/spacing incl. dark mode, WCAG 2.2 AA), key screens (tab bar, stash list+form, project detail w/ counters, pins masonry, filter interaction, empty states), app name direction.
- Diana: research spike **R1 (sync)**. ~~R2 (Ravelry API audit)~~ ✅ done — `plans/ravelry-api-research.md`. R3 (barcode/QR) can slip to later without blocking.
- Joint: resolve image cap (#6), pricing ballpark (#7), 2FA stance (#13 — recommend: launch with none; Supabase makes email OTP a config flip later). (Apple Developer $99/yr: already approved.)
- **Gate:** Phases 2+ UI needs the token set + key screens; Phase 1 proceeds regardless.

### Phase 1 — Scaffold
1. `create-expo-app` (TypeScript strict), expo-router, NativeWind, ESLint/Prettier — **web target enabled** (`expo start` → `w` is the daily dev loop)
2. Supabase project + local dev (`supabase start`); Drizzle schema v0 (`profiles`, `yarn_stash` + fibers, `entity_images`) migrated to both Postgres and SQLite
3. RLS policies + the RLS CI test; Sentry; GitHub Actions CI green
4. Expo Go running on the physical iPhone against local Supabase

### Phase 2 — Auth
1. Supabase Auth: Apple + Google + email/password (native flows via `expo-apple-authentication` / Google sign-in; verify Expo Go support — may trigger the dev-build switch)
2. Session persistence, protected routes, sign-out-preserves-local-data behavior (PRD §6.10)

### Phase 3 — Stash + Tools (first real vertical slice)
1. Craft-adaptive form system + field registry
2. Stash CRUD: comboboxes (brand/name create-new), fiber multi-select, `yarn_catalog` auto-population, images (compress → upload queue → Storage)
3. Tools CRUD
4. Search/filter/sort with deep-link state persistence (PRD §6.8); stash stats view
5. **Sync engine v1 (from R1)** wired under stash/tools

### Phase 4 — Projects + Counters
1. Project CRUD, status pipeline, yarn/tool linking with quick-view chips, cascade-on-stash-delete
2. Counters (multi, named, targets) — realtime sync across devices
3. Project notes + session notes; attachments
4. Stash intelligence: skeins-used prompts on Finish

### Phase 5 — Pins + Push-to-Project
1. Pins CRUD + masonry gallery
2. Push to Projects (copy, mark pushed)

### Phase 6 — Ravelry import + monetization + hardening
1. `ravelry-import` Edge Function per R2: OAuth connect → preview/deselect → mismatch warnings → commit (1 req/sec queue)
2. RevenueCat + project-cap trigger + placeholder paywall (dev build + TestFlight sandbox)
3. Offline/conflict hardening; accessibility audit (WCAG 2.2 AA, Dynamic Type, VoiceOver, reduced motion); onboarding (<60s to first action)

### Phase 7 — Beta
1. TestFlight to Sheryl + friends; Maestro suite into CI; store listing prep

---

## Versioning & Releases

Native apps change the picture from the old web plan:

- **App versions are real** — `version` (marketing, e.g. 1.0.0) + build number, managed in `app.json`/EAS (`autoIncrement`). Tag releases `v1.0.0` etc. with GitHub Releases (`--generate-notes`); Conventional Commits continue.
- **EAS Update `runtimeVersion`** must be managed deliberately — OTA updates only apply to builds with a matching runtime version; any native change bumps it.
- What's-live traceability = store build number + EAS Update id; Sentry releases tied to both.
- Changesets remain premature — revisit if/when web (Phase 2 roadmap) shares workspace packages.

---

## Research Spikes (mapped to PRD §10)

| Spike | PRD item | Owner | Blocks | Deliverable |
|---|---|---|---|---|
| **R1 — Sync engine** | §6.9 research flag | Diana | Phase 3 step 5 | 🔬 Architecture notes + spike plan ready: `plans/sync-architecture-r1.md` (PowerSync first, custom fallback; 9-scenario pass/fail matrix). Deliverable: two-device counter demo + decision writeup |
| **R2 — Ravelry API audit** | #12, #14 | Diana | Phase 6 | ✅ **Done 2026-07-18** — see `plans/ravelry-api-research.md`. All import endpoints exist; OAuth 2.0; no commercial fee. Remaining: confirmation email to api@ravelry.com before Phase 6 ships |
| **R3 — Barcode/QR for yarn labels** | #9 | Diana | Nothing | ✅ **Done 2026-07-18** — see `plans/ravelry-api-research.md` §Barcode scanning. Scanning is free (expo-camera); Ravelry API has no barcode lookup; no yarn-specific UPC DB exists. **Recommendation: defer to Phase 2**, built as scan → own mapping table → UPC-API fallback → pre-fill the lookup combobox |
| **R4 — Dynamic forms UX** | #10 | Sheryl (+Diana) | Phase 3 step 1 polish | Interaction spec; registry architecture already accommodates it |
| **Cost quantification** | #6, #7, #14 | Both | Monetization finalization | Storage math (above), Ravelry licensing (R2), AI/OCR costs (Phase 3 features — can wait) |

---

## What Changed From the Previous Plan

| Area | Old plan | This plan |
|---|---|---|
| Product | Ravelry web front-end (search/queue/library/favorites) | Standalone stash/projects/pins companion (PRD §5) |
| Platform | Next.js 14 on Firebase App Hosting/Cloud Run | React Native + Expo, iOS-first; **GCP dropped entirely** |
| Ravelry API | Live proxied data source | One-time OAuth import (Edge Function) |
| Data model | One `saved_patterns` table | Full relational stash/tools/projects/pins/counters model + shared `yarn_catalog` |
| Sync | N/A (server-rendered web) | Offline-first local SQLite + sync engine (spike R1) |
| E2E | Playwright | Maestro (Playwright returns with Phase-2 web) |
| Costs | $0 flat | $0 services + **$99/yr Apple** (approved, Diana pays) (+$25 Google later) |
| Kept | — | Supabase + RLS, Drizzle, $0 discipline, Actions CI shape, Conventional Commits + milestone tags, Sentry |

**Follow-up:** `docs/resources.md` still reflects the old stack (Next.js, Firebase, Playwright-for-web) and needs a matching overhaul. `plans/ravelry-pros-and-cons.md` remains useful as competitive/UX reference; its "replicate in RavelPlus" table now applies mostly to the Phase 3+ pattern-discovery vision rather than MVP.

---

## Open Questions for Sheryl

Superseding the old plan's list (filter panels, add-to-collection flows for the web app are moot):

- Design spec per Phase 0 — especially the filter interaction (PRD deliberately leaves bottom-sheet vs chips vs drawer to design) and pins masonry density
- Dynamic forms UX spike (R4) — what adapts on craft selection vs stash selection?
- Paywall presentation and free-tier messaging tone ("no penny-pinching" principle)
- Onboarding flow ordering: empty-state-first vs guided add-first-yarn vs offer-Ravelry-import-first for the refugee persona
- App name shortlist timing — blocks App Store metadata, icon, and TestFlight naming by Phase 7
