# RavelPlus — v1 Implementation Plan

## Context

Ravelry is the dominant knitting/crochet pattern database but has significant UX debt: poor search filtering, very poor mobile experience, and a cluttered UI. RavelPlus is a personal-use reimagining of Ravelry that uses the official Ravelry API as its data source, adds a polished mobile-first interface, and stores user-specific data (queue, library, favorites) in the user's own account. Starting as a friends-only app with a path to grow.

**Collaborators:**
- **Diana** — lead software engineer, owns all technical decisions
- **Sheryl** — senior UX designer (also new to crocheting), owns product/UX decisions and will write the PRD

---

## Cost Discipline

**Rule:** Everything in v1 must be free. Any service or feature that risks paid usage must be flagged before adoption.

### Free-tier audit (as of 2026)

| Service | Free tier | Notes |
|---|---|---|
| Supabase (Postgres DB) | 500 MB database, 2 projects, 5 GB egress/month | Safe for v1 — relational data is tiny. Pauses after 1 week inactivity (fine for active dev). |
| Cloud Run (Next.js hosting) | 2M requests/month, 360k vCPU-seconds | Far under — friend group traffic is negligible |
| Supabase Auth | 50,000 monthly active users | Bundled with the Supabase project; no credit card. Far under for a friend group. |
| Google Cloud Logging | 50 GiB/month of logs | Safe |
| Sentry | 1 user, 5k errors/month | Safe — only Diana needs access |
| Ravelry API | Free for personal/dev keys | Safe — 1 req/sec rate limit |
| GitHub | Free private repos | Safe |
| Playwright, Jest, RTL, Tailwind, Drizzle | Open source | Safe |

### ⚠️ The one gotcha: Cloud Run / App Hosting requires the Blaze plan

Firebase App Hosting (which runs Next.js on Cloud Run under the hood) requires the project to be on the **Blaze (pay-as-you-go)** plan, which requires a credit card. Blaze includes a generous free quota; staying under it means a $0 bill.

| Service (Blaze) | Free quota | Realistic v1 usage |
|---|---|---|
| Cloud Run (Next.js SSR) | 2M requests/month, 360k vCPU-seconds | Far under |
| Cloud Build (deploys) | 120 build-minutes/day | Far under |
| Cloud Storage (build artifacts) | 5 GB | Safe |

**Supabase** is separate from GCP billing — its free tier requires no credit card at all.

**Mitigation:** Set a hard GCP billing alert at **$1** during setup. Set Cloud Run `maxInstances` low to prevent runaway scaling.

### Decision (locked)

- **Database/Auth:** Supabase free tier (no credit card).
- **Hosting:** Firebase App Hosting on Blaze with a $1 GCP billing alert (email at 50/90/100%). Realistic monthly cost $0.
- **Cloud Run `maxInstances`** capped low as a safeguard.

### Future enhancements with cost implications (flagged in advance)

| Future feature | Cost concern |
|---|---|
| **v2 — Algolia search** | Free tier is 10k records / 10k ops/month. Ravelry's pattern catalog is far larger; we'd be using Algolia for the *subset we sync*, but this needs careful design to stay free. Will re-evaluate when v2 is planned. |
| **v3 — iOS App Store** | Apple Developer account is **$99/year**. Not free. Flagged for the v3 decision. |
| **Sentry team access** | Adding Sheryl as a Sentry user requires paid plan ($26/user/month). Will use GCP Error Reporting if she ever needs error access. |
| **Firebase Test Lab** | Free tier limited; we're using Playwright instead so this isn't a concern. |

---

## Tech Stack

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | Next.js 14 (App Router, TypeScript) | SSR/SSG, SEO, shares ecosystem with React Native for future iOS |
| Styling | Tailwind CSS | Mobile-first utility classes, fast to iterate |
| Auth | Supabase Auth | Native to our Postgres; enables **Row Level Security** (DB-enforced per-user access, not app-enforced). Free for 50k MAU, no credit card. **Shares an identity model with theStashBook** (also Supabase Auth), so future integration is a JOIN rather than an identity bridge. |
| Database | Supabase (Postgres) | Relational — models the pattern/yarn/collection many-to-many domain natively. Free tier needs no credit card. **theStashBook already runs on Supabase**, so future integration is dramatically simpler. |
| ORM | Drizzle | TypeScript-native, lightweight, serverless-friendly (HTTP driver, no connection-pool issues). Type-safe schema + migrations. |
| API proxy | Next.js Route Handlers | Ravelry API key stays server-side in the Next.js server env. No separate Cloud Functions project needed — Next.js on Cloud Run is a real server. |
| Hosting | Firebase App Hosting (Cloud Run) | GCP-native Next.js deployment via GitHub integration. Runs on Cloud Run under the hood. **Note:** the older "Firebase Hosting frameworks experiment" is closed — App Hosting is the current path. |
| Future iOS | React Native / Expo | Shares types and API client code with Next.js |

**Why this stack:** The domain is relational (patterns ↔ yarn ↔ projects ↔ collections are many-to-many; the yarn-suggestion feature is a join). Postgres models it natively where Firestore would force denormalization and app-side join logic. Using Next.js Route Handlers instead of Cloud Functions removes an entire service. Compute stays on GCP (Cloud Run); the database is a managed Postgres URL that can move to Cloud SQL later if ever desired.

**Portability:** Compute and data are cloud-agnostic — the app is a standard container; the DB is standard Postgres. The one deliberate coupling is auth: Supabase Auth ties login/session to Supabase's GoTrue and RLS policies to Postgres. This is an accepted trade: since the database is already committed to Supabase, auth portability protects a cheap-to-migrate layer, while Supabase Auth buys RLS, less code, and the theStashBook join today. If we ever leave Supabase we're moving the DB regardless, at which point re-doing auth is a rounding error.

---

## Repository Structure

```
ravelplus/
├── plans/                     # Plans and PRDs live here
├── docs/                      # Tech-stack reference (resources.md)
├── apps/
│   └── web/                   # Next.js app (the whole app — no separate backend)
│       ├── app/
│       │   ├── (auth)/        # Login page
│       │   ├── api/
│       │   │   └── ravelry/   # Ravelry proxy route handlers (search, [id])
│       │   ├── patterns/      # Search + detail pages
│       │   └── me/            # Queue, Library, Favorites
│       ├── components/
│       │   └── ui/            # shadcn/ui components
│       ├── lib/
│       │   ├── supabase/      # Supabase clients (browser, server, middleware)
│       │   ├── ravelry.ts     # Server-side Ravelry API client
│       │   └── collections.ts # Drizzle queries for queue/library/favorites
│       ├── db/
│       │   ├── schema.ts      # Drizzle schema (saved_patterns; auth.users is managed by Supabase)
│       │   └── index.ts       # Drizzle client (Supabase connection)
│       ├── drizzle/           # Generated migrations
│       └── types/             # Shared TypeScript types
├── e2e/                       # Playwright tests
├── apphosting.yaml            # Firebase App Hosting config
└── package.json               # Root workspace (npm workspaces)
```

**Note:** `functions/`, `firebase.json`, and `firestore.rules` from the earlier scaffold are removed — no longer needed.

---

## v1 Feature Scope

> **Note:** UX details (filter layouts, empty states, collection flows, navigation patterns) are Sheryl's call and will be informed by her PRD. Implementation below describes the technical shape; visual/interaction design defers to Sheryl.

### 1. Auth (Supabase Auth)
- **Providers for v1:** Google OAuth + email/password. Additional providers (GitHub, Apple, magic links, etc.) are config-only changes in the Supabase dashboard — no data model changes.
- **Identity storage:** Supabase's managed `auth.users` table. Our own tables reference users via `user_id uuid references auth.users(id)`.
- **Access control:** **Row Level Security (RLS)** — every user-data table gets a policy `using (auth.uid() = user_id)`. The database itself refuses to return rows that aren't the caller's, regardless of what a query says. A forgotten filter returns nothing instead of leaking everything. This moves the security boundary from "every developer remembers to filter, forever" to "enforced once, centrally."
- **Clients:** `@supabase/ssr` for server components / route handlers / middleware (cookie-based sessions). Drizzle is used for schema, migrations, and privileged/complex queries; user-scoped reads go through the RLS-aware Supabase client so `auth.uid()` is set.
- Protected routes via Next.js middleware (refreshes the Supabase session, redirects unauthenticated users from `/me/**`).
- `/login` page — visual design TBD by Sheryl.

**Why Supabase Auth (not Auth.js):** The DB is already committed to Supabase, so "auth portability" protects a layer that's cheap to migrate anyway. Supabase Auth instead buys three things now: (1) RLS — DB-enforced access control rather than app-side `where user_id = ...` discipline on every query; (2) less code — hosted OAuth/MFA/magic links vs. wiring providers ourselves; (3) **theStashBook integration becomes a cross-schema JOIN** under one identity instead of an identity-bridge between two auth systems. The v1.1 yarn-suggestion feature is literally a join against Sheryl's stash data — shared Supabase Auth erases its hardest part before we write a line.

**Accepted trade-off:** Supabase Auth couples login/session UX to Supabase's hosted GoTrue, and RLS policies are Postgres/`auth.uid()`-specific. For a friends-only app this is a remote risk and the policies are ~5 lines per table.

### 2. Pattern Search & Browse (Core v1 priority)
**Improvements over Ravelry:**
- Filters visible on mobile (interaction pattern TBD by Sheryl)
- Instant URL-synced filter state (shareable search URLs)
- Cleaner results grid
- Filter by: craft (knitting/crochet), yarn weight, category, difficulty, free/paid, language

**Technical implementation:**
- `/patterns` — search page with filter panel + results grid
- Next.js Route Handler `GET /api/ravelry/search` proxies `api.ravelry.com/patterns/search.json` (server-side, API key from env)
- Debounced search-as-you-type with loading skeletons
- `usePatternSearch` hook handles query state, pagination, and URL sync

### 3. Pattern Detail Page
- `/patterns/[id]` — photo carousel, description, yarn requirements, needle sizes, difficulty, free/paid badge
- Next.js Route Handler `GET /api/ravelry/[id]` proxies `api.ravelry.com/patterns/:id.json`
- Add to Queue / Library / Favorites (authenticated only)
- Links to purchase/download on Ravelry

### 4. Queue / Library / Favorites
**Replicate Ravelry's 3-list model:**
- **Queue** — patterns the user plans to make
- **Library** — patterns the user owns (purchased/downloaded)
- **Favorites** — patterns the user loves but hasn't committed to

**Technical implementation:**
- Stored in Postgres `saved_patterns` table, one row per (user, pattern, list)
- `/me/queue`, `/me/library`, `/me/favorites` pages
- `useCollection(listType)` hook → Server Actions backed by Drizzle queries in `lib/collections.ts`
- Optimistic UI updates
- Move-between-lists = update the `list_type` column — interaction pattern TBD by Sheryl

---

## Data Model (Postgres / Drizzle)

**Supabase-managed:** `auth.users` (and related auth schema) — owned by Supabase Auth, not declared in our Drizzle schema. Our tables reference it via `auth.users(id)`.

**App tables:**

```sql
-- saved_patterns: one row per pattern saved to one of a user's lists
saved_patterns (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references auth.users(id) on delete cascade,
  ravelry_id    text not null,              -- Ravelry pattern id
  list_type     text not null,              -- 'queue' | 'library' | 'favorites'
  name          text not null,              -- denormalized for fast list rendering
  thumbnail_url text,
  notes         text,
  added_at      timestamptz not null default now(),
  unique (user_id, ravelry_id, list_type)
)
```

**Access control via RLS** — enable on every user-data table:

```sql
alter table saved_patterns enable row level security;
create policy "own rows" on saved_patterns
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

The database enforces per-user isolation, so a missing `where user_id = ...` returns nothing rather than leaking other users' rows. User-scoped reads go through the RLS-aware Supabase client (carries the JWT so `auth.uid()` resolves); Drizzle is used for migrations and any privileged/service-role queries.

**Why relational pays off later:** v1.1 stash/yarn tables (`yarn_stash`, `projects`, `project_yarns` join table) and the yarn-suggestion join slot in naturally here — no denormalization gymnastics.

---

## Ravelry Proxy (Next.js Route Handlers)

All Ravelry API calls go through server-side Route Handlers — the API key (env var) never reaches the browser. Each handler checks the Supabase session before proxying. No separate backend service.

| Route Handler | Proxies |
|---|---|
| `GET /api/ravelry/search` | `GET /patterns/search.json` |
| `GET /api/ravelry/[id]` | `GET /patterns/:id.json` |
| `GET /api/ravelry/yarn/search` | `GET /yarns/search.json` *(v1.1)* |

Rate limiting: Ravelry allows 1 req/sec per key. The 300ms search debounce keeps us well within it.

---

## Design System

**Approach:** Design-first. Phase 2+ UI work is gated on Sheryl's design spec being delivered. Phase 1 scaffold proceeds independently.

**Component library:** [shadcn/ui](https://ui.shadcn.com/) — unstyled Radix UI primitives copied directly into the project (`components/ui/`). Fully owned, fully restyable, free. Pairs with Tailwind CSS. Sheryl can restyle any component without fighting a third-party library.

### What Sheryl needs to deliver before Phase 2

A design spec (Figma, a doc, or equivalent) covering:

| Item | Examples |
|---|---|
| Color palette | Primary, secondary, background, surface, error, success — light mode; dark mode optional for v1 |
| Typography | Font family (Google Fonts preferred — free), size scale (heading levels, body, small), weights |
| Spacing & border radius | Card radius, button radius, general spacing rhythm |
| Key screens | Login, pattern search (desktop + mobile), pattern detail, one collection page |
| Mobile navigation | Bottom tab bar vs. other — her call |
| Filter panel | Mobile interaction (drawer? inline?) — her call |
| Add-to-collection flow | Bottom sheet? inline toggle? modal? — her call |
| Empty states | At least queue, library, favorites empty states |
| Brand/name | Is "RavelPlus" the name? Does it need a logo or wordmark for v1? |

### How design tokens get implemented

Once Sheryl's palette and typography are decided:
1. Define CSS custom properties in `apps/web/app/globals.css` (colors, radius, fonts)
2. Extend `tailwind.config.ts` with the project's named tokens (e.g. `bg-surface`, `text-primary`)
3. Install shadcn/ui components on top of those tokens — they'll automatically inherit the design
4. Any component that needs restyling gets updated in `components/ui/`

This means Sheryl can hand off a palette and we wire it in one afternoon — no component-by-component reskin.

---

## Build Sequence

### Phase 0 — Design spec (Sheryl, parallel to Phase 1)
- Sheryl delivers design spec covering the items above
- **Gate:** Phase 2+ UI work does not start until spec is received
- Diana can proceed with Phase 1 scaffold in parallel

### Phase 1 — Project scaffold *(needs rework — see note)*
1. Root `package.json` with npm workspaces ✓ (keep)
2. `npx create-next-app` in `apps/web` (TypeScript + Tailwind + App Router) ✓ (keep)
3. **Remove** earlier Firebase artifacts: `functions/`, `firebase.json`, `firestore.rules`
4. **TODO:** Create Supabase project; add `DATABASE_URL` + `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY` to `apps/web/.env.local`
5. **TODO:** Install Drizzle + `drizzle-kit` and `@supabase/ssr`; create `db/schema.ts` + `db/index.ts`; run first migration
6. **TODO:** Create Firebase project, upgrade to Blaze, set $1 billing alert, connect repo via App Hosting
7. **TODO:** Deploy a Hello World to App Hosting to confirm the pipeline

### Phase 2 — Auth
1. Install `@supabase/ssr`; create browser/server/middleware Supabase clients in `lib/supabase/`
2. Enable Google OAuth + email/password in the Supabase dashboard; configure redirect URLs
3. `/login` page (Google + email/password) + auth callback route handler
4. `middleware.ts` to refresh the Supabase session and protect `/me/**` routes
5. Enable RLS + `auth.uid() = user_id` policies on `saved_patterns`

### Phase 3 — Pattern Search
1. Route Handler `app/api/ravelry/search/route.ts` (session check + proxy)
2. `lib/ravelry.ts` server-side API client
3. `/patterns` page: search input, filter panel, results grid
4. `PatternCard` component
5. URL-synced filter state

### Phase 4 — Pattern Detail
1. Route Handler `app/api/ravelry/[id]/route.ts`
2. `/patterns/[id]` page
3. Add-to-collection buttons

### Phase 5 — Collections
1. `lib/collections.ts` Drizzle queries + Server Actions
2. `/me/queue`, `/me/library`, `/me/favorites` pages
3. Optimistic add/remove on pattern detail page
4. Move-between-lists (update `list_type`)

### Phase 6 — Mobile polish + deploy
1. Responsive audit (Sheryl's design direction)
2. Navigation pattern (Sheryl's call — bottom tabs vs. top nav)
3. Final App Hosting deploy

---

## Future Roadmap (Post-v1)

- **v1.1** — Yarn database (search + stash tracking)
- **v1.2** — Pattern notes / project tracking (WIPs, finished objects)
- **v2** — Sync Ravelry data into Algolia for faster, more flexible search
- **v2.5** — Social: follow friends, see their queues
- **v3** — React Native / Expo iOS app (shares `lib/` types and API client)

### theStashBook Integration (Future Enhancement)

Sheryl's [theStashBook](https://github.com/sheryl-madethis/theStashBook) is a yarn stash + project tracker (vanilla JS, Supabase backend) that is a natural complement to RavelPlus. Potential integration points:

- **Stash-aware pattern search** — filter results by yarn weight/yardage the user already owns
- **"Start a project" flow** — when queuing a pattern, link yarn from your stash; check if you have enough yardage
- **Pattern → project bridge** — theStashBook's Pins tab already saves Ravelry pattern URLs manually; this could become a one-click flow from RavelPlus
- **Yarn suggestions on pattern pages** — given a pattern's yarn requirements (weight, yardage, fiber), surface stash matches ranked by: (1) exact brand/colorway match, (2) same weight + enough yardage, (3) same weight but insufficient yardage (shows how many skeins short). Sheryl's call on the UX — could be a "You might already have this" shelf on the pattern detail page.

**Key data model (theStashBook yarn):**
```
{ brand, name, colorName, colorHex, fibers[], weight, yards, skeins, images[], notes }
```
Projects already store `patternUrl` (links to Ravelry) and `yarnIds[]` (many-to-many to stash).

**Integration outlook (much improved by the Supabase decision):**
- **Same database platform.** Both apps now run on Supabase Postgres. Integration could be as deep as a shared Supabase project / shared schema, or as light as cross-querying — no cross-platform data migration needed.
- **Auth: now unified.** Both apps use Supabase Auth, so a user is one identity across both. The yarn-suggestion feature becomes a cross-schema JOIN (`saved_patterns` ↔ stash under the same `auth.uid()`) rather than an identity-bridge between two auth systems — the biggest integration blocker is gone.
- theStashBook is a single monolithic HTML file with no API surface — would need refactoring before deep integration.
- **Recommended path:** When v1.1 stash tracking is built in RavelPlus, model the `yarn_stash` table to mirror theStashBook's yarn fields so the two schemas line up directly.

---

## Logging & Error Tracking

| Layer | Tool | Notes |
|---|---|---|
| Cloud Run (Next.js server + Route Handlers) | Google Cloud Logging | Automatic — `console.log/warn/error` from the server routes to the GCP console for the Cloud Run service. No setup needed. |
| Next.js client errors | Sentry | Catches unhandled JS errors, React error boundaries, network failures in the browser. |
| Next.js server errors | Sentry | Catches App Router server component, Server Action, and Route Handler failures via the Next.js Sentry SDK. |

**Setup:** Add `@sentry/nextjs` to `apps/web` in Phase 2. Sentry's wizard (`npx @sentry/wizard -i nextjs`) wires up source maps and the error boundary automatically. Free tier (1 user, 5k errors/month) is sufficient — only Diana needs access.

**What to log in the Ravelry proxy handlers:** All Ravelry API errors should log `{ status, ravelryId/query, message }` so failures are traceable in Cloud Logging without exposing the API key.

---

## Testing Standards

### Unit Tests — Jest + React Testing Library

**Setup:** `jest` + `@testing-library/react` + `@testing-library/user-event` in `apps/web`.

**What to unit test:**

| Target | Examples |
|---|---|
| Custom hooks | `usePatternSearch` (query/filter state, URL sync), `useCollection` (add/remove/move logic, optimistic updates) |
| Collection queries | `lib/collections.ts` Drizzle queries / Server Actions — run against a local test Postgres (or `pglite` in-memory); assert rows scoped to `user_id` |
| Route Handlers | `app/api/ravelry/*` — mock `fetch` and the Supabase session; assert correct proxying, auth rejection, error responses |
| Utility functions | Yarn suggestion matching logic (weight match, yardage calculation) when built |
| UI components | `PatternCard` renders correctly with given props; filter drawer opens/closes |

**What not to unit test:** Supabase Auth internals, RLS policies (cover with an E2E test that asserts user A can't read user B's rows), real OAuth flows, third-party UI library components.

**File convention:** Co-located with source — `component.test.tsx` next to `component.tsx`; `handler.test.ts` next to `handler.ts`.

**Coverage target:** No hard % threshold — focus on testing logic-heavy hooks, helpers, and Cloud Function handlers thoroughly. UI snapshots are low value; skip them.

---

### E2E Tests — Playwright

**Setup:** `@playwright/test` at the repo root, running against the Next.js dev server (`localhost:3000`) backed by a local/test Supabase Postgres.

**What to E2E test (critical user flows only):**

| Flow | What it covers |
|---|---|
| Auth | Sign up, log in, redirect to `/patterns`, log out |
| Pattern search | Type a query, apply a filter, verify results update, verify URL reflects filters |
| Pattern detail | Navigate to a pattern, verify key details render, add to Queue |
| Collections | Add pattern to Queue from detail page, visit `/me/queue`, verify it appears, remove it |
| Protected routes | Visit `/me/queue` while logged out, verify redirect to `/login` |

**What not to E2E test:** Every UI state, every filter combination, visual regression. Keep the E2E suite small and fast — it should run in under 2 minutes.

**Test database:** E2E tests run against a dedicated test Postgres (local Supabase via `supabase start`, or a separate free Supabase project) with seeded data, kept isolated from production. Auth state reused across tests via Playwright's `storageState`.

**File location:** `e2e/` at repo root, e.g. `e2e/auth.spec.ts`, `e2e/search.spec.ts`, `e2e/collections.spec.ts`.

---

## CI/CD

**Platform: GitHub Actions.** Native to the repo, best-in-class Playwright support, and free within budget. Everything below stays at **$0**.

### CI — checks on every PR

A single workflow on `pull_request` runs four gates, all on `ubuntu-latest`:

| Gate | Command | ~Time |
|---|---|---|
| Lint | `eslint` | ~1 min |
| Typecheck | `tsc --noEmit` | ~1 min |
| Unit tests | Jest + RTL | ~1 min |
| E2E | Playwright vs. `supabase start` (local Postgres + RLS) + Next.js dev server | ~2–5 min |

All four are **required status checks** in branch protection — nothing merges red.

**Why E2E in CI is free:** Playwright runs headless browsers *on the runner itself* — it calls no paid service, so it only consumes GitHub Actions **minutes** (same budget as the other jobs). (Note: Microsoft's hosted "Playwright Testing" service on Azure *is* paid — we are **not** using it. Plain `npx playwright test` on a runner is the free path.)

**Minute budget:** Private repos get **2,000 Actions min/month** free. A full PR run is ~6–11 min, so ~180–330 runs/month — far beyond a two-contributor project's needs.

**Guardrails to stay free (these matter):**
1. **`ubuntu-latest` only.** macOS bills 10×, Windows 2× against minutes; Linux is 1×. This is the #1 way people accidentally blow the free tier.
2. **Cache** npm deps, the Playwright browser binaries (`~/.cache/ms-playwright`), and Docker layers for the Supabase images.
3. **`concurrency: cancel-in-progress`** — a new push to a PR cancels the older run so stale builds don't burn minutes.

### CD — production

App Hosting **already auto-deploys `main`** on push via its GitHub integration (free). No additional workflow needed for production deploys.

### Preview deploys for Sheryl (manual)

App Hosting does not provide per-PR preview URLs (the old Firebase Hosting preview-channel feature doesn't carry over). To give Sheryl a live preview without a paid tier:

- A **`workflow_dispatch`** (manually triggered) Action builds the PR branch and deploys it to a **tagged Cloud Run revision** — a stable preview URL with no production traffic.
- Cloud Run's free tier (2M req/month, scales to zero) covers a preview hit a handful of times; the build runs on Actions minutes. Free.
- Manual trigger (not every PR) keeps minute usage and clutter down — run it when Sheryl needs to look.

**Day-one compatibility:** the test setup is structured so these workflows drop in without refactoring (E2E already runs against `supabase start`; tests are runnable headless). Workflow files are written when CI is actually stood up — not part of initial scaffold.

---

## Documentation Reference (`docs/resources.md`)

A separate file at `docs/resources.md` (in the repo) will collect official docs, tutorials, and reference material for the tech stack — focused on areas where Diana wants ramp-up support (GCP, Playwright) and quick-reference for the rest.

### Planned structure

The doc will be organized by tech area, with each section listing: official docs, a recommended starter tutorial, and any "gotcha" callouts specific to this project.

**Sections:**

1. **GCP / Supabase / Drizzle** (highest priority — Diana hasn't used GCP in years)
   - Firebase App Hosting + Cloud Run (Next.js deployment, GitHub integration)
   - GCP billing budgets & alerts; Cloud Logging
   - Supabase (project setup, Postgres connection, table editor, local dev via `supabase start`)
   - Drizzle ORM (schema, migrations with `drizzle-kit`, queries, Supabase connection)
   - Supabase Auth (providers, `@supabase/ssr`, App Router middleware, Row Level Security policies)

2. **Playwright** (second priority — minimal experience)
   - Getting started + first test
   - Auth strategies (storageState pattern — sign in once, reuse session)
   - Selectors and assertions
   - Configuring against the test Postgres
   - Trace viewer & debugging

3. **Next.js + React quick reference** (Diana is experienced — light section)
   - App Router docs (server vs. client components, routing patterns)
   - Server Actions / Route Handlers
   - `next/image`, `next/font`
   - Middleware (for auth-protected routes)

4. **shadcn/ui + Tailwind**
   - shadcn/ui installation for Next.js
   - Component catalog
   - Tailwind v4 docs

5. **Testing libraries**
   - Jest (Next.js setup)
   - React Testing Library
   - Mock Service Worker (to mock the Ravelry proxy in unit tests)
   - pglite / local Postgres for testing Drizzle queries

6. **Ravelry API**
   - API docs index
   - OAuth/basic auth setup
   - Pattern search endpoint reference
   - Rate limits and best practices

7. **Sentry**
   - Sentry for Next.js setup
   - Source map upload
   - Free tier limits reminder

8. **Recommended deep-dive tutorials** (not docs — actual learning resources)
   - One end-to-end Firebase + Next.js tutorial
   - One Playwright + Next.js E2E walkthrough
   - One Cloud Functions TypeScript walkthrough

Each link will be verified at the time of writing. The file is a living document — Diana adds links as she finds useful resources.

---

## Verification Plan

- After Phase 3: search results load, filters update URL params, tested on Chrome mobile viewport
- After Phase 5: add a pattern to queue while logged in, verify the row appears in the Supabase table editor
- Final: deploy to App Hosting, test on a real iPhone
- Access control: verify one user cannot read another user's saved patterns (Server Action scoped by session `user_id`)

---

## Open Questions for Sheryl (PRD)

- Filter panel interaction on mobile (slide-up drawer? collapsible sidebar? always-visible?)
- Add-to-collection flow (bottom sheet? inline toggle? explicit modal?)
- Navigation structure (bottom tab bar on mobile? hamburger? top tabs?)
- Empty state copy and illustration direction
- Onboarding flow for new users (connect existing Ravelry account? start fresh?)
