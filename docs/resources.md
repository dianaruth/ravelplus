# RavelPlus — Tech Stack Reference

A living document. Add links as you find useful resources. Organized by priority — Expo/EAS and the offline-sync research are the areas with the steepest ramp-up; Supabase/Drizzle carries over from the previous plan as priority reading.

> **Note:** Spot-check links before deep-diving — doc sites reorganize. Anything broken: find the new path and update it here.

---

## Expo & React Native

> Top priority. Diana is experienced in React web but new to Expo, React Native, and the native app lifecycle. Everything else builds on this. Good news: the daily dev loop runs in the browser (see Web target below), so the familiar web DX carries over.

### Getting started
- [Expo docs — Get started](https://docs.expo.dev/get-started/introduction/) — `create-expo-app`, project structure, running on a device
- [Expo Go vs development builds](https://docs.expo.dev/develop/development-builds/introduction/) — **read early**: Expo Go is the $0 on-device dev loop (critical on Windows, no iOS Simulator); a dev build becomes required the moment we add a native module Expo Go doesn't bundle (e.g., RevenueCat)
- [expo-router](https://docs.expo.dev/router/introduction/) — file-based routing, deliberately modeled on Next.js App Router conventions; layouts, tabs, dynamic routes. Deep linking comes free — this is how PRD §6.8 filter-state persistence works (deep links on native, real URLs on web)
- [React Native docs — Core components](https://reactnative.dev/docs/components-and-apis) — View/Text/FlatList/Pressable; the "div/span don't exist" mental shift
- [React Native — Platform-specific code](https://reactnative.dev/docs/platform-specific-code) — `Platform.select`, `.ios.tsx`/`.web.tsx` file resolution

### Web target (enabled from day one)
- [Expo for Web](https://docs.expo.dev/workflow/web/) — `expo start` → press `w`: browser dev loop with hot reload + React DevTools — **the primary daily dev surface**
- [react-native-web](https://necolas.github.io/react-native-web/) — what RN primitives compile to on web (real DOM/CSS); mature, powers X/Twitter web
- [Platform-specific modules in expo-router](https://docs.expo.dev/router/advanced/platform-specific-modules/) — `.web.tsx` files can use any web/DOM library; how Phase 2's feature-rich web screens diverge without forking logic
- [Publishing Expo web apps](https://docs.expo.dev/distribution/publishing-websites/) — static export + hosting (Phase 2; any free static host works)
- ⚠️ Per the plan's day-one rules: every library adoption gets a **web-compat check**, and CI keeps the web bundle building so Phase 2 never becomes a rescue project.

### SDK modules we use
- [expo-sqlite](https://docs.expo.dev/versions/latest/sdk/sqlite/) — the local database under the offline-first layer (Drizzle drives it); check its web (WASM/OPFS) support status for spike R1
- [expo-image-picker](https://docs.expo.dev/versions/latest/sdk/imagepicker/) — stash/project/pin photos
- [expo-image-manipulator](https://docs.expo.dev/versions/latest/sdk/imagemanipulator/) — **mandatory client-side compression** before upload (see plan: 1 GB Supabase storage ceiling)
- [expo-camera](https://docs.expo.dev/versions/latest/sdk/camera/) — barcode scanning for spike R3
- [expo-apple-authentication](https://docs.expo.dev/versions/latest/sdk/apple-authentication/) — native Sign in with Apple button/flow
- [expo-document-picker](https://docs.expo.dev/versions/latest/sdk/document-picker/) — project file/PDF attachments

### Gotchas for this project
- ⚠️ **Windows dev:** no iOS Simulator. Daily loop = browser (web target) + Expo Go on the physical iPhone as the continuous native check. The browser never substitutes for on-device verification — iOS-specific issues only surface on the phone.
- ⚠️ In-app purchases and some auth native modules don't run in Expo Go — sequence RevenueCat work after the dev-build switch (plan Phase 6).

---

## EAS (Build / Submit / Update)

> How binaries get made and shipped without a Mac.

- [EAS Build — setup](https://docs.expo.dev/build/setup/) — cloud builds for iOS/Android; `eas.json` profiles (`development` / `preview` / `production`)
- [EAS Submit](https://docs.expo.dev/submit/introduction/) — push builds to TestFlight / Play Console from the CLI
- [EAS Update](https://docs.expo.dev/eas-update/introduction/) — OTA updates for JS-only changes between store builds
- [Runtime versions & update compatibility](https://docs.expo.dev/eas-update/runtime-versions/) — **read before first release**: OTA updates only apply to builds with a matching `runtimeVersion`; any native change bumps it
- [Internal distribution](https://docs.expo.dev/build/internal-distribution/) — ad-hoc installs pre-TestFlight (needs the paid Apple account + registered device UDIDs)
- [Expo pricing](https://expo.dev/pricing) — free tier ≈ 30 cloud builds/month — **build on demand, not per-commit**
- [Apple Developer Program enrollment](https://developer.apple.com/programs/enroll/) — $99/yr (approved); enroll before Sheryl's first TestFlight build
- [TestFlight overview](https://developer.apple.com/testflight/) — internal vs external testers, review requirements for external

---

## Supabase (Postgres + Auth + Storage + Realtime + Edge Functions)

> Carried over as priority reading — still new to Diana, and now the *entire* backend (GCP is dropped).

### Core
- [Supabase with Expo/React Native tutorial](https://supabase.com/docs/guides/getting-started/tutorials/with-expo-react-native) — **our exact client setup**: supabase-js in RN, session persistence with AsyncStorage/SecureStore
- [Local development & CLI](https://supabase.com/docs/guides/local-development) — `supabase start` runs the full stack locally (dev + CI tests)
- [Database overview](https://supabase.com/docs/guides/database/overview) — table editor, SQL editor
- [Supabase pricing](https://supabase.com/pricing) — free tier: 500 MB DB, 1 GB storage, 5 GB egress, 500K edge invocations; pauses after 1 week idle

### Auth
- [Auth overview](https://supabase.com/docs/guides/auth) — users, sessions, JWTs
- [Native Sign in with Apple](https://supabase.com/docs/guides/auth/social-login/auth-apple) — pairs with `expo-apple-authentication`; required on iOS since we offer Google login
- [Native Google sign-in](https://supabase.com/docs/guides/auth/social-login/auth-google) — RN flow + Google Cloud Console config; web uses the standard OAuth redirect flow instead
- [Password-based auth](https://supabase.com/docs/guides/auth/passwords) — email/password
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security) — **core security model**: `auth.uid() = user_id` on every user table; the DB refuses to leak rows regardless of query bugs

### Storage
- [Storage overview](https://supabase.com/docs/guides/storage) — buckets, upload from RN
- [Storage access control](https://supabase.com/docs/guides/storage/security/access-control) — RLS-style policies on objects (per-user image isolation)

### Realtime
- [Realtime overview](https://supabase.com/docs/guides/realtime) — channels, broadcast, presence
- [Postgres Changes](https://supabase.com/docs/guides/realtime/postgres-changes) — the live cross-device nudge for counters/entities (exact role depends on spike R1's outcome)

### Edge Functions
- [Edge Functions overview](https://supabase.com/docs/guides/functions) — Deno runtime, deploy via CLI; hosts `ravelry-import`, `revenuecat-webhook`, `delete-account`
- [Managing secrets](https://supabase.com/docs/guides/functions/secrets) — where the Ravelry API key lives (never in the app bundle)
- [Background tasks](https://supabase.com/docs/guides/functions/background-tasks) — relevant for the chunked, rate-limited import worker

---

## Drizzle ORM (double duty: Postgres + expo-sqlite)

- [Get started — Expo SQLite](https://orm.drizzle.team/docs/get-started/expo-new) — **our local DB setup**: schema, driver, `useLiveQuery` reactivity
- [Get started — Supabase](https://orm.drizzle.team/docs/get-started/supabase-new) — the same schema definitions driving the remote Postgres side
- [Schema declaration](https://orm.drizzle.team/docs/sql-schema-declaration) — `db/schema.ts`, single source of truth
- [Migrations with drizzle-kit](https://orm.drizzle.team/docs/migrations) — separate migration outputs for the postgres and sqlite dialects
- [Relations](https://orm.drizzle.team/docs/relations) — the many-to-many joins (`project_yarns`, `pin_tools`, fibers)

---

## Offline-First Sync (Spike R1 reading)

> Second priority. Highest-risk technical decision in the plan — read before building anything in `lib/sync/`.

### Candidates
- [PowerSync docs](https://docs.powersync.com/) — Postgres↔SQLite sync engine; first candidate. Has a dedicated **web SDK** (WASM SQLite) — a point in its favor given the day-one web target
- [PowerSync + Supabase integration guide](https://docs.powersync.com/integration-guides/supabase-+-powersync) — the exact pairing we'd use
- [PowerSync pricing](https://www.powersync.com/pricing) — verify free tier / self-host terms against the $0 rule
- [WatermelonDB](https://watermelondb.dev/docs) — RN-native reactive DB with a sync protocol you implement server-side; **weak web story — likely disqualifying**
- [Legend-State](https://legendapp.com/open-source/state/) — state library with Supabase persistence plugin; evaluate quickly, likely too shallow for per-field merge

### Concepts (for the custom-build option)
- [Local-first software (Ink & Switch)](https://www.inkandswitch.com/local-first/) — the canonical essay; good framing for the conflict model
- [Supabase offline-first discussion](https://github.com/supabase/supabase/discussions/357) — long-running community thread; useful map of approaches and pitfalls

**Spike criteria (from the plan):** per-field LWW merge · $0 at MVP scale · Expo compatibility (Go or dev build?) · **web compatibility** (Phase 2 is committed — native-only engines are disqualified) · RLS enforced on the sync path · coexists with the image upload queue · sane battery/bandwidth under multi-device live edits.

---

## UI — NativeWind, Components, Accessibility

- [NativeWind docs](https://www.nativewind.dev/) — Tailwind syntax for RN; compiles to real CSS on web; where Sheryl's design tokens land (theme config)
- [React Native — FlatList performance](https://reactnative.dev/docs/optimizing-flatlist-configuration) — stash/pins lists at scale
- [FlashList (Shopify)](https://shopify.github.io/flash-list/) — faster list primitive; candidate for the pins masonry gallery — ⚠️ verify web support at adoption (day-one rule)
- [React Native — Accessibility](https://reactnative.dev/docs/accessibility) — accessibility props, VoiceOver/TalkBack; PRD targets WCAG 2.2 AA at launch, not later
- [Apple HIG — Live Activities](https://developer.apple.com/design/human-interface-guidelines/live-activities) — Phase 3 reference only; skim so MVP decisions don't foreclose it

---

## Testing

### Unit — Jest + React Native Testing Library
- [Unit testing with Expo](https://docs.expo.dev/develop/unit-testing/) — `jest-expo` preset setup
- [React Native Testing Library](https://callstack.github.io/react-native-testing-library/) — `render`, `screen`, `userEvent` for RN components
- [Jest mock functions](https://jestjs.io/docs/mock-functions) — `jest.fn()`, `jest.mock()`
- [pglite](https://github.com/electric-sql/pglite) — in-memory Postgres for testing Drizzle queries/Postgres-side logic without a running DB

### E2E — Maestro
- [Maestro docs](https://maestro.mobile.dev/) — install, first flow, YAML syntax
- [E2E tests with Maestro on EAS](https://docs.expo.dev/eas/workflows/reference/e2e-tests/) — running Maestro against Expo builds
- ⚠️ **Maestro Cloud is paid — we don't use it.** Local runs + Android emulator in CI is the free path.
- [Android emulator on GitHub Actions](https://github.com/ReactiveCircus/android-emulator-runner) — the action for CI E2E on `ubuntu-latest` (KVM); deferred until flows stabilize
- (Playwright returns in Phase 2 for the desktop web app — the old plan's Playwright ramp-up notes are in git history.)

### RLS verification
- Two-user test asserting cross-user reads/writes fail on every user table — runs against `supabase start` in CI. (Pattern: supabase-js with two signed-in test clients; no special tooling needed.)

---

## RevenueCat (subscriptions)

- [RevenueCat + Expo installation](https://www.revenuecat.com/docs/getting-started/installation/expo) — SDK setup (**requires a dev build — not Expo Go**)
- [Quickstart](https://www.revenuecat.com/docs/getting-started/quickstart) — products, entitlements, offerings model
- [Webhooks](https://www.revenuecat.com/docs/integrations/webhooks) — feeds the `revenuecat-webhook` Edge Function that writes the `subscriptions` table
- [Sandbox testing](https://www.revenuecat.com/docs/test-and-launch/sandbox) — TestFlight sandbox purchases
- [Pricing](https://www.revenuecat.com/pricing/) — free up to $2.5K/mo tracked revenue
- Phase 2 note: web subscriptions are a different rail (RevenueCat Web Billing / Stripe) — evaluate when web ships, not now

---

## Ravelry API (Spike R2 — ✅ audited 2026-07-18, see `plans/ravelry-api-research.md`)

- [Ravelry API docs](https://www.ravelry.com/api) — full endpoint reference (login required; actively maintained)
- [Developer portal](https://www.ravelry.com/pro/developer) — app/key creation ("RavelPlus" Pro account already exists)
- [API License Agreement](https://www.ravelry.com/content/legal/api-agreement) — the authoritative terms (v1.0, Nov 2020)
- Import endpoints confirmed: `stash/list|show|unified/list`, `needles/list|sizes|types`, `projects/list|show`, `packs`, plus `queue`/`favorites`/`library` for Phase 2
- Auth for import: **OAuth 2.0** (`/oauth2/auth` + `/oauth2/token`, 24 h tokens, `offline` scope for refresh)
- ⚠️ **Rate limit:** none officially published; 429 has undocumented per-method limits — we self-impose 1 req/sec in the import worker with backoff
- ✅ **No commercial fee** in the current agreement; constraints are audience restriction + discretionary revocation — confirmation email to api@ravelry.com before Phase 6 ships
- ⚠️ **No redistribution of Ravelry data** (clause 1i) — never bulk-seed `yarn_catalog` from the API

---

## Sentry

- [Sentry for React Native](https://docs.sentry.io/platforms/react-native/) — SDK install, error boundaries
- [Sentry + Expo setup](https://docs.sentry.io/platforms/react-native/manual-setup/expo/) — config plugin, source maps for EAS builds
- [Releases & source maps](https://docs.sentry.io/platforms/react-native/sourcemaps/) — tie errors to build number + EAS Update id
- Free tier: 1 user, 5K errors/month — only Diana needs dashboard access

---

## CI/CD — GitHub Actions

> The whole pipeline stays at $0 — see the CI/CD section of the implementation plan.

- [Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) — `on:`, `jobs:`, `steps:`
- [Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) — npm + Supabase docker layers
- [`concurrency` / cancel-in-progress](https://docs.github.com/en/actions/using-jobs/using-concurrency) — stale PR runs don't burn minutes
- [Billing & free-tier minutes](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) — 2,000 min/month private; ⚠️ macOS 10×, Windows 2× — **`ubuntu-latest` only**
- [Required status checks](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — nothing merges red
- CI also runs an **Expo web export** so the web bundle never bit-rots (day-one rule)
- Note: EAS Build runs on Expo's infrastructure, not Actions minutes — trigger builds on demand, not per-commit

---

## Recommended Tutorials

> End-to-end walkthroughs — not just docs. Good for building mental models. Do these roughly in order.

1. **Expo fundamentals** — [Expo tutorial (official)](https://docs.expo.dev/tutorial/introduction/) — build a small app with expo-router; covers the phone + web dev loop
2. **Supabase + Expo end-to-end** — [with-Expo tutorial](https://supabase.com/docs/guides/getting-started/tutorials/with-expo-react-native) — auth + CRUD from RN, our exact client setup
3. **Drizzle + Expo SQLite** — [Expo get-started](https://orm.drizzle.team/docs/get-started/expo-new) — local DB with live queries, start to finish
4. **PowerSync + Supabase** — [integration guide](https://docs.powersync.com/integration-guides/supabase-+-powersync) — doubles as spike R1 groundwork
5. **First Maestro flow** — [Maestro getting started](https://maestro.mobile.dev/getting-started/installing-maestro) — install → record → run against a build
