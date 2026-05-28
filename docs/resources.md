# RavelPlus — Tech Stack Reference

A living document. Add links as you find useful resources. Organized by priority — GCP/Firebase and Playwright are the areas with the steepest ramp-up.

---

## Hosting, Database & Auth

> Priority reading. Diana hasn't used GCP in years and is new to Supabase/Drizzle. Start here before Phase 1.

### Firebase App Hosting (Cloud Run) — deployment
- [Get started with App Hosting](https://firebase.google.com/docs/app-hosting/get-started) — **current path for Next.js on Firebase** (runs on Cloud Run; the older "Hosting frameworks experiment" is permanently closed)
- [App Hosting overview](https://firebase.google.com/docs/app-hosting) — GitHub integration, automatic deploys on push to `main`, `apphosting.yaml` config
- [Cloud Run overview](https://cloud.google.com/run/docs/overview/what-is-cloud-run) — what App Hosting runs on underneath; useful for understanding scaling and `maxInstances`
- ⚠️ App Hosting requires the **Blaze (pay-as-you-go)** plan. Free quota covers our usage — set a $1 billing alert as a safeguard.

### Supabase (Postgres)
- [Supabase quickstart](https://supabase.com/docs/guides/getting-started) — create a project, get the connection string
- [Local Development & CLI](https://supabase.com/docs/guides/local-development) — `supabase start` runs the full stack locally (used for tests + offline dev)
- [Database overview](https://supabase.com/docs/guides/database/overview) — table editor, SQL editor, connection pooling
- [Connecting to your database](https://supabase.com/docs/guides/database/connecting-to-postgres) — direct vs. pooled connection strings (use the pooled/`6543` URL for serverless)
- [Supabase pricing](https://supabase.com/pricing) — free tier limits (500 MB DB, pauses after 1 week idle)

### Drizzle ORM
- [Get Started with Drizzle and Supabase](https://orm.drizzle.team/docs/get-started/supabase-new) — **our exact setup**: schema, env, connection, CRUD
- [Schema declaration](https://orm.drizzle.team/docs/sql-schema-declaration) — defining tables in `db/schema.ts`
- [Migrations with drizzle-kit](https://orm.drizzle.team/docs/migrations) — `drizzle-kit generate` + `migrate`
- [Select / insert / update / delete queries](https://orm.drizzle.team/docs/select) — query builder reference
- [Drizzle relations](https://orm.drizzle.team/docs/relations) — modeling the many-to-many joins (relevant for v1.1 yarn/projects)

### Supabase Auth
- [Auth overview](https://supabase.com/docs/guides/auth) — concepts: users, sessions, JWTs, providers
- [Next.js Server-Side Auth (`@supabase/ssr`)](https://supabase.com/docs/guides/auth/server-side/nextjs) — **our exact setup**: browser/server/middleware clients, cookie-based sessions in the App Router
- [Login with Google](https://supabase.com/docs/guides/auth/social-login/auth-google) — OAuth setup + Google Cloud Console config
- [Password-based auth](https://supabase.com/docs/guides/auth/passwords) — email/password sign-up and sign-in
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security) — **core to our security model**: `auth.uid()` policies enforce per-user data isolation in the DB
- [RLS + `auth.uid()` helpers](https://supabase.com/docs/guides/database/postgres/row-level-security#helper-functions) — writing `using (auth.uid() = user_id)` policies
- [Redirect URLs & middleware](https://supabase.com/docs/guides/auth/server-side/nextjs#create-a-middleware-file) — session refresh + protecting `/me/**`

### GCP Billing, Logging & Errors
- [Create and manage budgets & alerts](https://docs.cloud.google.com/billing/docs/how-to/budgets) — set the $1 alert here
- [Firebase pricing overview](https://firebase.google.com/pricing) — Spark vs. Blaze, free tier table
- [Cloud Run pricing](https://cloud.google.com/run/pricing) — request + compute free tier details
- [Cloud Logging overview](https://cloud.google.com/logging/docs/overview) — where `console.log` from the Next.js server (on Cloud Run) goes
- [Sentry for Next.js](https://docs.sentry.io/platforms/javascript/guides/nextjs/) — client + server error tracking (free, 1-user plan)

---

## Playwright

> Second priority. Playwright is powerful but has a learning curve around auth state and async patterns.

### Getting started
- [Installation](https://playwright.dev/docs/intro) — `npm init playwright@latest`, test runner basics
- [Writing your first test](https://playwright.dev/docs/writing-tests) — `test()`, `expect()`, `page.goto()`
- [Running tests](https://playwright.dev/docs/running-tests) — CLI flags, `--headed`, `--debug`, `--ui`
- [VS Code extension](https://playwright.dev/docs/getting-started-vscode) — run/debug tests directly in VS Code, test picker

### Core concepts
- [Selectors](https://playwright.dev/docs/locators) — prefer `getByRole`, `getByLabel`, `getByText` over CSS selectors
- [Assertions](https://playwright.dev/docs/test-assertions) — `expect(locator).toBeVisible()`, `toHaveText()`, etc.
- [Actions](https://playwright.dev/docs/input) — `click()`, `fill()`, `press()`, navigation
- [Network interception](https://playwright.dev/docs/network) — `page.route()` for mocking API responses in tests

### Auth strategy
- [Authentication guide](https://playwright.dev/docs/auth) — **most important page for our setup**. The `storageState` pattern lets you sign in once and reuse the session across tests — critical since we use Supabase Auth sessions.
- Our pattern: sign in once in a `global-setup.ts` file, save to `playwright/.auth/user.json`, all tests that need auth load that state. Tests run against a local Supabase (`supabase start`) test database.

### Debugging
- [Trace viewer](https://playwright.dev/docs/trace-viewer-intro) — step-by-step trace of a test run, screenshots at each action
- [Inspector / pause](https://playwright.dev/docs/debug) — `page.pause()` drops you into the Playwright Inspector to debug interactively
- [Codegen](https://playwright.dev/docs/codegen) — record browser actions and generate test code automatically (good for getting started fast)

### Configuration
- [Config file reference](https://playwright.dev/docs/test-configuration) — `playwright.config.ts`, `baseURL`, `projects` (multi-browser)
- [Using environment variables](https://playwright.dev/docs/test-parameterize) — pass Supabase connection strings etc. into tests

---

## CI/CD — GitHub Actions

> Free within budget. The whole pipeline stays at $0 — see the CI/CD section of the implementation plan for the minute budget and guardrails.

### GitHub Actions basics
- [Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) — `on:`, `jobs:`, `steps:`, the YAML reference
- [Events that trigger workflows](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) — `pull_request` (CI gates) and `workflow_dispatch` (manual preview deploy)
- [Caching dependencies](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows) — cache npm + Playwright browsers to cut minutes
- [`concurrency` / cancel-in-progress](https://docs.github.com/en/actions/using-jobs/using-concurrency) — kill stale PR runs so they don't burn minutes
- [Billing & free-tier minutes](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions) — **2,000 min/month** on private repos; ⚠️ macOS bills 10×, Windows 2×, Linux 1× — stay on `ubuntu-latest`
- [Required status checks / branch protection](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — block merges until CI passes

### Playwright in CI
- [Playwright on CI](https://playwright.dev/docs/ci) — official guidance: containers, browser install, headless config
- [Playwright GitHub Actions guide](https://playwright.dev/docs/ci-intro) — **our exact path**: the generated workflow + trace artifact upload
- ⚠️ Microsoft's hosted "Playwright Testing" Azure service is **paid** — we do NOT use it. Plain `npx playwright test` on an Actions runner is the free path.

### Cloud Run preview deploys (manual)
- [Deploy to Cloud Run from a workflow](https://github.com/google-github-actions/deploy-cloudrun) — the `google-github-actions/deploy-cloudrun` action used by the manual preview job
- [Cloud Run revision tags & traffic](https://cloud.google.com/run/docs/managing/revisions) — tagged revisions give a stable preview URL without taking production traffic

---

## Next.js App Router

> Diana is experienced here — this is quick-reference only.

- [App Router docs](https://nextjs.org/docs/app) — layouts, pages, server/client components, routing
- [Server vs. client components](https://nextjs.org/docs/app/building-your-application/rendering/server-components) — decision guide for when to use each
- [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) — `app/api/` endpoints (our Cloud Functions proxy calls go through here)
- [Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) — where the Supabase middleware client refreshes the session and protects `/me/**`
- [useSearchParams](https://nextjs.org/docs/app/api-reference/functions/use-search-params) — URL-synced filter state for the search page
- [`next/image`](https://nextjs.org/docs/app/api-reference/components/image) — image optimization, important for pattern thumbnails
- [`next/font`](https://nextjs.org/docs/app/api-reference/components/font) — Google Fonts with zero layout shift

---

## shadcn/ui + Tailwind

- [shadcn/ui — Next.js installation](https://ui.shadcn.com/docs/installation/next) — CLI setup, `components.json`, import alias
- [shadcn/ui component catalog](https://ui.shadcn.com/docs/components) — browse available components (Button, Card, Dialog, Drawer, etc.)
- [Theming](https://ui.shadcn.com/docs/theming) — how CSS custom properties map to component styles; where Sheryl's palette goes
- [Tailwind CSS v4 docs](https://tailwindcss.com/docs) — utility reference
- [Tailwind — Responsive design](https://tailwindcss.com/docs/responsive-design) — `sm:`, `md:`, `lg:` breakpoints, mobile-first approach
- [Radix UI primitives](https://www.radix-ui.com/primitives) — the unstyled components shadcn/ui builds on; useful when customizing behavior

---

## Testing

### Jest + React Testing Library
- [Jest with Next.js](https://nextjs.org/docs/app/building-your-application/testing/jest) — official Next.js setup guide
- [React Testing Library docs](https://testing-library.com/docs/react-testing-library/intro/) — `render`, `screen`, `userEvent`
- [Common queries cheatsheet](https://testing-library.com/docs/queries/about) — `getByRole` vs `getByText` vs `getByTestId`
- [Testing custom hooks](https://react-hooks-testing-library.com/) — `renderHook` for testing `usePatternSearch`, `useCollection`

### Mocking
- [Jest mock functions](https://jestjs.io/docs/mock-functions) — `jest.fn()`, `jest.mock()`
- [Mock Service Worker (MSW)](https://mswjs.io/docs/) — intercept fetch calls in tests; useful for mocking the Ravelry API in Route Handler unit tests
- [pglite](https://github.com/electric-sql/pglite) — in-memory Postgres for testing Drizzle queries without a running database

---

## Ravelry API

- [Ravelry API docs](https://www.ravelry.com/api) — full endpoint reference, auth setup
- [Pattern search endpoint](https://www.ravelry.com/api#patterns_search) — query params, filters, response shape
- [Pattern detail endpoint](https://www.ravelry.com/api#patterns_show) — full pattern object
- [API authentication](https://www.ravelry.com/api#introduction_authentication) — Basic Auth with personal access key (what we use in Cloud Functions)
- ⚠️ **Rate limit:** 1 request/second per API key. Our debounced search (`300ms`) stays well within this.

---

## Recommended Tutorials

> End-to-end walkthroughs — not just docs. Good for building mental models.

- **Drizzle + Supabase end-to-end** — [Get Started with Drizzle and Supabase](https://orm.drizzle.team/docs/get-started/supabase-new) — our exact DB setup, start to finish
- **Supabase Auth with Next.js App Router** — [Server-Side Auth tutorial](https://supabase.com/docs/guides/auth/server-side/nextjs) — `@supabase/ssr` clients, middleware session refresh, RLS
- **Next.js deploy to Firebase App Hosting** — [App Hosting get started](https://firebase.google.com/docs/app-hosting/get-started) — GitHub-connected deploy pipeline
- **Playwright end-to-end** — [Playwright official tutorial](https://playwright.dev/docs/writing-tests) + [auth setup walkthrough](https://playwright.dev/docs/auth) — do these in order
