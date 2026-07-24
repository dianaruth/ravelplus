# Ravelry API — Research Findings (Spike R2)

**Date:** 2026-07-18 · **Researcher:** Diana (with Claude) · **Status:** Substantially complete — one confirmation email recommended
**Sources:** [API documentation](https://www.ravelry.com/api) (last updated 2026-06-18, login required), [Developer portal](https://www.ravelry.com/pro/developer), [Application Developer and API License Agreement](https://www.ravelry.com/content/legal/api-agreement) (v1.0, 2020-11-06)

---

## TL;DR

1. **The API is free. There is no commercial licensing fee.** The PRD's assumption that "if [AppName] generates revenue, Ravelry charges a commercial API licensing fee" is **not supported by the current license agreement** — no fee appears anywhere in it. The real commercial constraint is an *audience restriction*, not money (see Licensing below).
2. **Everything the MVP import needs exists**: full CRUD-grade read access to a user's stash, needles/hooks, and projects (including photos, yarn linkage, notes, statuses), via OAuth.
3. **No official global rate limit is published.** The docs define a 429 response with "per-method limits" but publish no numbers. The PRD's "1 req/sec" is community lore, not documentation — fine to keep as our self-imposed cap.
4. **One real risk to respect:** Ravelry reserves broad discretion (non-compete clause, destroy-on-request clause, may discontinue the API at any time). The import feature must stay a *nice-to-have accelerant*, never a load-bearing dependency.
5. **Do not seed our shared `yarn_catalog` from the Ravelry API** — redistribution/syndication of Ravelry data is explicitly prohibited (clause 1i). Catalog seeding needs a different source (or Ravelry's blessing — they invite exactly this conversation).

---

## Access & Authentication

- API keys come from a **free Pro account** at [ravelry.com/pro/developer](https://www.ravelry.com/pro/developer). ✅ Already done — a "RavelPlus" Pro account exists under dianaruth40, no apps created yet.
- Four credential types:

| Credential | Access | Use for us |
|---|---|---|
| Basic Auth, read-only | Only non-"authenticated" methods (pattern/yarn search, etc.) | Dev experimentation |
| Basic Auth, personal key | **Full access to your own account** | Diana's local testing against her own stash |
| **OAuth 2.0** | Per-user, scoped, user grants via standard flow | **The import flow** — this is what ships |
| OAuth 1.0a | Legacy equivalent | Ignore |

- OAuth 2.0 specifics: endpoints `www.ravelry.com/oauth2/auth` + `/oauth2/token`; **tokens expire in 24 h**; request the `offline` scope for refresh tokens; client credentials must go in the Authorization header (body auth unsupported); handle 401 by re-authenticating. Default scopes cover notebook reads — no special scope needed for stash/projects/needles.
- SSL required everywhere. Keys/secrets live server-side only (our Edge Function).

## Endpoint Inventory (relevant subset)

The API is large (~40 method groups — it even covers forums, carts, pattern stores). What matters for us:

### MVP import (PRD §6.7) — all present ✅

| Ravelry endpoint | Maps to |
|---|---|
| `stash/list`, `stash/show`, `stash/unified/list` | → `yarn_stash` (brand, colorway, dye lot, photos; unified list includes handspun/fiber stash) |
| `needles/list`, `needles/sizes`, `needles/types` | → `tools` (covers knitting needles **and** crochet hooks) |
| `projects/list`, `projects/show`, `projects/crafts`, `projects/project_statuses` | → `projects` (status, craft, notes, dates, photos) |
| `packs` (project↔yarn linkage) | → `project_yarns` join rows |
| `patterns/show` | pattern name/designer/URL enrichment for imported projects |
| `people/show`, `/current_user` | account linkage + import preview header |

### Phase 2 (queue/library → Pins) — all present ✅
`queue/list`, `queue/show`, `favorites/list`, `library/search`

### Phase 3+ (pattern/yarn discovery) — present, but see licensing
`patterns/search`, `yarns/search`, `yarns/show`, `yarn_companies` — full search with pagination. Usable for *live, user-initiated* queries; **not** for bulk-copying into our own database.

### Useful mechanics
- Pagination on all list methods (`page`, `page_size`; keep ≤100 — responses taking >10 s are killed with a 504)
- ETag support for avoiding redundant fetches
- Photos come as hosted URLs in responses — the importer downloads the user's own photos and re-uploads to our storage

## Rate Limiting — what's actually documented

- **No published global requests/sec limit.** The docs define `429 Too Many Requests` with "refer to the documentation for per-method limits," but no numbers are published on the main docs page.
- Specific mentions only: voting endpoints are rate-limited; `library-pdf`-scoped tokens expire faster and can expire on rate-limit breach.
- **Our approach stands:** self-imposed 1 req/sec queue in the import worker (conservative, matches community convention), exponential backoff on 429, ETags where applicable. An average import (say 200 stash items + 50 projects, mostly via paginated list calls) is a few dozen requests — minutes, not hours, even at 1 req/sec.

## Licensing & Cost — the correction

The [API License Agreement](https://www.ravelry.com/content/legal/api-agreement) (v1.0, Nov 2020) contains **no fees**. What it actually requires:

1. **Commercial use is permitted** — but "only … for commercial applications whose primary audience is Ravelry Users unless Ravelry expressly allows otherwise," with Ravelry as sole arbiter (clause 1c). Our *import feature's* audience is by definition Ravelry users; the *app's* audience is broader. This is the gray area worth a confirmation email.
2. **Non-compete clause (1f):** may not "compete with or diminish the need for any of Ravelry's own commercial applications" — Ravelry sole arbiter. Ravelry's commercial operations are pattern sales and advertising; a stash/project tracker doesn't obviously touch either, and the PRD's "not a Ravelry replacement" positioning helps. Still discretionary.
3. **Data handling:** disclose how we store users' data (2a); use data consistent with the owner's wishes (2b — the owner here is the importing user, acting on their own data); store it "only as long as necessary to provide the service" (2c — our service *is* long-term storage of the user's own records, a defensible read); **destroy on request by Ravelry** (1h) or by the data owner (2b).
4. **No redistribution/syndication (1i):** we may not resell or redistribute Ravelry data — this is what rules out bulk-seeding `yarn_catalog` from `yarns/search`.
5. **No stability guarantee:** Ravelry may change or discontinue the API at any time (3a–b).
6. Notably, the agreement opens with an invitation: apps wanting to build on Ravelry's yarn/pattern data — even apps "unrelated to Ravelry" — are asked to email **api@ravelry.com**; "we're very interested in assisting."

### Cost impact on the monetization model (PRD §7, §10 #12/#14)

- **Direct API cost at scale: $0.** No per-user, per-call, or revenue-share fee exists in the current terms. The PRD's Ravelry-fee line item in the monetization open questions can be closed as "no fee under current terms (v1.0, 2020), confirmed 2026-07-18."
- The residual exposure is **not a cost but a revocation risk**: clauses 1c/1f/1h are discretionary, and the API can be discontinued. Priced accordingly: Ravelry import is a growth feature we can lose without breaking the product — never gate paid functionality on it, never make onboarding depend on it.
- **Terms are from 2020 and could change.** Because we're commercial-ish (freemium), send the confirmation email before shipping the import feature publicly.

## Feature → API call map

Verified against the live docs (2026-07-18). All notebook (`/people/...`, `/projects/...`) calls require OAuth user context; `yarns/*` and `patterns/*` reads work with read-only Basic Auth too. Everything routes through our Edge Function.

### MVP — Ravelry Import (stash + tools + projects)

Call sequence for the `ravelry-import` worker:

| Step | Call | Returns / notes |
|---|---|---|
| 0 | `POST www.ravelry.com/oauth2/token` (after `/oauth2/auth` redirect, scope `offline`) | Access + refresh token |
| 1 | `GET /current_user.json` | Username (needed in every notebook URL) + identity for the preview header |
| 2 | `GET /people/{username}/stash/list.json` | Paginated (default 50/page), `Stash (small)` array — small variant already includes `colorway_name`, `dye_lot`, `first_photo`, `personal_yarn_weight` |
| 3 | `GET /people/{username}/stash/{id}.json` per entry | `Stash (full)`: adds full photo set, notes, `packs`, nested yarn (→ our `ravelry_yarn_id`). **Optimization to verify in dev:** if the list variant covers our field map, skip per-item shows and cut import calls by ~2/3 |
| 4 | `GET /people/{username}/needles/list.json` | Single call, `NeedleRecord (full)` array — **covers crochet hooks and knitting needles** |
| 5 | `GET /needles/types.json` + `GET /needles/sizes.json` | Reference data (once, not per user) for size→mm normalization; `craft=crochet\|knitting` param confirms hook/needle split |
| 6 | `GET /projects/{username}/list.json` | `Project (small)` array; `page_size` defaults to the entire set in one call |
| 7 | `GET /projects/{username}/{id}.json` per project | `Project (full)`: `status_name`, `started`, `completed`, `craft_name`, `progress`, `made_for`, `notes`, `pattern_id`/`pattern_name`, `needle_sizes` (→ `project_tools` matching), `packs` (→ `project_yarns` rows — packs tie a project to stash entries incl. allocated yardage/grams) |
| 8 | `GET /patterns/{id}.json` per *unique* pattern_id | Optional enrichment: designer, pattern URL for imported projects |
| 9 | Photo downloads (plain HTTPS GETs on returned URLs) → compress → Supabase Storage | Not API calls; still throttled politely |

**Call budget:** a heavy notebook (200 stash, 50 projects, ~30 unique patterns) ≈ 1 + 4 + 200 + 1 + 1 + 50 + 30 ≈ **~290 calls ≈ 5 minutes at our 1 req/sec cap** (down to ~2 min if step 3 proves skippable). Chunked with progress in `import_jobs` either way, so UX doesn't hinge on it.

**Status mapping note:** Ravelry statuses (In progress / Finished / Hibernating / Frogged / Queued…) exceed our To do / In progress / Finished pipeline — Hibernating/Frogged need a mapping decision in the import preview's mismatch warnings (PRD §6.7 flow handles this).

### MVP — Yarn lookup on stash-add (pointer model)

| Step | Call | Returns / notes |
|---|---|---|
| 1 | `GET /yarns/search.json?query=…` (debounced ~300 ms) | `Yarn (list)` + paginator; supports all on-site yarn search filters if we want them |
| 2 | `GET /yarns/{id}.json?include=colorways` on selection | `Yarn (full)` — weight, yardage/grams, fibers — plus `Colorway (full)` array for the color-name dropdown (PRD §6.2's colorway auto-population) |
| — | `GET /yarn_weights.json`, `GET /color_families.json` | Static reference lists, fetched once and stored as app constants |

### Phase 2 — Queue / Favorites / Library → Pins

| Source | Call | Notes |
|---|---|---|
| Queue | `GET /people/{username}/queue/list.json` (+ `queue/{id}.json`) | `QueuedProject` includes `pattern_id`, queue position |
| Favorites | `GET /people/{username}/favorites/list.json` | Favorites span types (patterns, yarns, projects) — filter to patterns for Pins |
| Library | `GET /people/{username}/library/search.json` | Supports `query` + `type` params |
| Enrichment | `GET /patterns/{id}.json` per unique pattern | Name, designer, photos, difficulty, URL → pin fields |

### Phase 3+ — Pattern discovery (live, user-initiated only)

- `GET /patterns/search.json` — full search with the on-site filter set (craft, weight, free/paid, etc.)
- `GET /patterns/{id}.json` — detail views
- `https://api.ravelry.com/feeds/v1/patterns` — ETag-friendly feed of recently added/updated patterns (only if Ravelry blesses any caching; otherwise skip)

### Not needed / explicitly skipped

- `stash/unified/list` — includes fiber/handspun stash; out of scope until we support spinning as a craft
- All write endpoints (`stash/create`, `projects/update`, `packs/create`, …) — we never write back to Ravelry in any planned phase
- Forums, messages, carts, shops, stores, deliveries — out of scope entirely

## Barcode scanning (Spike R3 — quick findings, 2026-07-18)

- **Scanning is free and built in:** expo-camera reads UPC/EAN/QR in Expo Go — no third-party program needed.
- **Ravelry API has no barcode lookup** — confirmed by search of the full API docs; no UPC/EAN/barcode field exists on any endpoint. Closes PRD #9 sub-question (2).
- **No yarn-specific UPC database exists.** Generic ones (UPCitemdb — 100 free lookups/day, Barcode Spider, UPC Database, Go-UPC) likely cover big-box brands only, return unstructured product-name strings (no weight/fiber/yardage), and indie/hand-dyed yarn is largely absent or unbarcoded. Dye lot is never in a barcode.
- **Recommended design (Phase 2):** scan → check our own `barcode → yarn` mapping table → on miss, free-tier UPC API for a product name → pre-fill the existing yarn search/combobox → user confirms → save the mapping. The barcode is a shortcut into the lookup flow, not a data source; our mapping table grows from real scans and is entirely our data.
- **Recommendation:** defer to Phase 2 (the PRD's documented fallback) — the feature layers cleanly on the combobox flow once it exists, so deferring loses nothing.

## Yarn lookup design (decided 2026-07-18)

How auto-population (PRD §6.2) uses Ravelry data without violating clause 1i — the **pointer model**:

- The stash-add combobox merges two labeled sources: our own `yarn_catalog` (user-created + manually seeded rows) and **live, user-initiated** `yarns/search` results.
- Selecting a Ravelry match writes the attributes (weight, fiber, yardage, colorways) into the **user's own `yarn_stash` row only**, and stores `ravelry_yarn_id` for provenance — the same ownership logic that makes import safe (the user acting on data for their own records).
- **Ravelry-sourced data never lands in the shared `yarn_catalog`** — no caching, no lazy accumulation. Caching was considered and rejected: demand-driven caching converges on a copy of their database over time, which is exactly what 1i prohibits. It becomes an option only if Ravelry blesses it (asked as a nice-to-have in the email below).
- **Scale check:** lookups happen only when a user adds newly bought yarn — a few times per month per user. Even 1K active users ≈ well under 0.01 req/sec average against our self-imposed 1 req/sec cap. Rate limits are a non-issue at any realistic scale for this feature.
- **Offline:** Ravelry lookup is an online-only enhancement. Offline stash-add still works with manual field entry + our own catalog. This is a soft carve-out from the PRD's "fully functional offline" principle — flag to Sheryl so the degraded combobox state gets a designed treatment.
- **Failure mode:** if Ravelry access is ever revoked, the feature degrades to catalog + manual entry; nothing breaks. Keeps import/lookup non-load-bearing as required.

## Recommended actions

1. **Email api@ravelry.com** (before Phase 6 ships, not urgent now): describe RavelPlus — freemium fiber-arts tracker; (a) import lets Ravelry users copy *their own* stash/projects/needles via OAuth — confirm fit with clauses 1c/1f; (b) describe the pointer-model yarn lookup (live user-initiated search, results saved only to the user's own stash row with `ravelry_yarn_id` provenance) as a courtesy; (c) nice-to-have: ask whether they'd bless caching looked-up yarn records with attribution, which would let auto-population work offline.
2. **Create the OAuth 2.0 app** in the existing RavelPlus Pro account when Phase 6 starts (redirect URI will be the Edge Function callback).
3. **PRD updates for Sheryl** (needs her sign-off since it's the shared doc): §6.7 and §10 #12/#14 — replace the "commercial API licensing fee" assumption with the audience-restriction + revocation-risk framing above.
4. **Plan updates** (done alongside this doc): rate-limit claim corrected to "self-imposed"; `yarn_catalog` seeding constraint noted.
