# Product Requirements Document
## [AppName] — MVP Planning Document
**Version:** 0.2 (Post-founder sync update)
**Date:** 2026-07-09
**Authors:** Sheryl + Diana
**Status:** Draft — for founder alignment review

---

## Purpose of This Document

This PRD exists to drive alignment between co-founders before implementation begins. It captures the product vision, agreed decisions, and explicitly flags open decisions that require a joint call before development starts. Decisions marked **🔴 OPEN** are unresolved and should be discussed at the next founder meeting.

---

## 1. Product Vision

[AppName] is a universal fiber arts companion app for crocheters, knitters, and all craft enthusiasts. It gives fiber artists a single, beautifully designed place to manage their creative life — tracking their material stash, organizing projects, saving inspiration, following their patterns row by row, and syncing everything seamlessly across every device they use.

Where existing tools split into disconnected silos — a stash tracker here, a counter there, a pattern library somewhere else — [AppName] unifies the full creative workflow into one coherent, reliable experience. It is not a Ravelry replacement; it is the app that Ravelry users wish they had alongside it.

**Tone & voice:** Warm, encouraging, inclusive. Language that welcomes every skill level without condescension. The app speaks like a knowledgeable friend, not a database.

---

## 2. Problem Statement

Fiber artists today manage their creative lives across too many disconnected tools: spreadsheets for stash inventory, social media bookmarks for inspiration, row counter apps mid-project, and community platforms like Ravelry for pattern discovery. No single tool does all of this well — and the tools that try to do most of it fail at the fundamentals:

- **Sync is broken industry-wide.** Nearly every competitor relies on manual push/pull backups or paywalled, unreliable sync. Data loss across devices is one of the most-cited complaints in the category.
- **The stash-to-project workflow is fragmented.** Users can't easily go from "what yarn do I have" to "where am I in this pattern right now" without switching apps.
- **Existing apps are craft-specific.** Most tools speak exclusively to knitters or crocheters, ignoring the large audience that does both — or other fiber arts entirely.
- **Mobile is an afterthought.** The category leader (Ravelry) explicitly outsources its mobile experience. Modern alternatives are better but still rough at the edges.
- **Subscription fatigue is real.** Users across the category repeatedly call out wanting a fair, honest pricing model — not features locked behind paywalls that feel arbitrary.

---

## 3. Target Personas

### Primary: The Serious Hobbyist
- Crochets and/or knits consistently — multiple active projects at any time
- Has some kind of tracking system today (spreadsheet, notes app, memory) but it's not working
- Wants depth and reliability over flashy features
- Mobile-native but also uses a tablet or laptop regularly
- Values an app that respects their time and their craft

### Secondary: The Ravelry Refugee
- Has invested years into building a Ravelry stash and project history
- Frustrated by Ravelry's clunky UX, poor mobile experience, and lack of real sync
- Wants to migrate their data without losing years of work
- Will advocate loudly for an app that gets import right

### Tertiary: The New-to-Apps Crafter
- Recently discovered their craft, often via social media
- No existing stash tracker or project history
- Needs zero-friction onboarding — gets turned off by complexity immediately
- Will grow into advanced features over time if early experience is positive

---

## 4. Platform & Technical Direction

### 4.1 Platform Approach
**Decision:** React Native with Expo, **mobile and tablet first, native-first**. Desktop is deferred to Phase 2.

**Rationale:**
- Phone and tablet are where the target personas spend the most time with this type of app
- Native-first with React Native/Expo enables OS-level integrations (lock-screen counters, Live Activity, Apple Watch, home screen widgets) that are critical differentiators in later phases — a responsive web app forecloses these permanently
- One shared codebase means desktop and additional platforms follow without a full rewrite
- Ravelry's desktop-first approach is a documented weakness; mobile-first is a direct opportunity

### 4.2 Device Priority
1. **Phase 1 (MVP):** Phone + Tablet (iOS-first; Android as fast-follow)
2. **Phase 2:** Desktop web
3. **Phase 3:** Apple Watch, lock-screen widgets, Live Activity; Android (if not already shipped as fast-follow)

### 4.3 Backend Stack
Supabase (PostgreSQL + Auth + Storage + Realtime) is the recommended backend, providing:
- Relational database for the linked entity model
- Row-level security for multi-user data isolation
- Built-in auth with social provider support
- Realtime subscriptions for live cross-device sync
- File storage for images and pattern attachments

### 4.4 Infrastructure Cost Principle
**All tooling and services must run on free tiers. The full CI/CD pipeline target is $0.**

This is a named constraint, not an aspiration. Every infrastructure decision — hosting, database, testing, CI/CD, error tracking, logging — must have a free tier that covers our usage at MVP scale. Paid services are not introduced without an explicit decision to do so. Where a service requires a pay-as-you-go plan (e.g., Firebase App Hosting on the Blaze plan), usage must stay within the free quota and a billing alert must be configured as a safeguard.

---

## 5. MVP Scope

### 5.1 In Scope for MVP

| Feature Area | What Ships in MVP |
|---|---|
| Inventory — Yarn stash | Full yarn stash management (brand, colorway, weight, yardage, fiber, images, notes) |
| Inventory — Tools | Full tools/hooks/needles inventory (type, brand, size, material, images, notes) |
| Projects | Project tracking with status pipeline, linked stash items, images, files, notes |
| Pins | Inspiration board (masonry gallery) with push-to-project conversion; eventually builds into pattern discovery/search and Pinterest-style inspiration accumulation |
| Yarn stash suggestions | Smart suggestions for what to make based on yarns in the user's stash; craft-adaptive dynamic forms based on selection |
| Counters | Multiple named counters per project (manual tap-to-increment/decrement) |
| Sync | Offline-first, real-time cross-device sync — free for all users |
| Ravelry Import | Import stash, tools, and projects from a connected Ravelry account |
| Authentication | Sign in with Apple, Sign in with Google, email + password |
| Craft-adaptive forms | Fields, labels, and terminology adapt per craft type per item/project |

### 5.2 Out of Scope for MVP (named future phases)

| Feature | Phase |
|---|---|
| Pattern parsing / PDF import with editing, highlighting, and in-pattern notes | Phase 2 |
| OCR / vision-based scanned pattern support | Phase 2+ |
| Lock-screen / Live Activity counters | Phase 3 |
| Apple Watch companion | Phase 3 |
| Home screen widgets | Phase 3 |
| Community / social features | Phase 4 |
| Social sharing (project stat cards, public profiles) | Phase 4 |
| Designer marketplace with payment management | Phase 4 |
| AI features and yarn stash suggestions | Phase 3 |
| Ravelry queue/library → Pins import | Phase 2 |
| Android | Phase 1 fast-follow / Phase 2 |
| Desktop web | Phase 2 |
| In-app craft glossary with terminology links to tutorials | Phase 2 |

---

## 6. Core Feature Specifications

### 6.1 Universal Craft-Adaptive Forms

A foundational UX pattern that differentiates [AppName] from every competitor.

**How it works:**
- Every project, stash item, and tool entry begins with a **craft type selection** (Crochet, Knitting)
- Based on the selection, field labels, terminology, and available options adapt dynamically:
  - Crochet → hooks, hook sizes (US letter/mm), stitch counts
  - Knitting → needles (US number/mm, type: circular/straight/DPN), row counts
- A user can have projects of different craft types simultaneously — craft type is per-item, not per-account
- The architecture must be designed to add new craft types without schema rewrites

**Craft types at launch:** Crochet, Knitting
**Future craft types:** Architecture must support adding new craft types without schema rewrites; specific crafts TBD based on user feedback

---

### 6.2 Yarn Stash

**Core fields:**
- Brand (searchable combobox, create-new)
- Yarn name (searchable combobox, create-new)
- Color name (auto-populated from brand + yarn name when known in the database)
- Dye lot
- Fiber content (multi-select with percentages: Acrylic, Wool, Cotton, Alpaca, Bamboo, Polyester, Nylon, Rayon + create-new; e.g., "80% Wool, 20% Nylon")
- Weight (standard: Lace #0 → Jumbo #7)
- Yards per skein (auto-populated when brand + yarn name are known)
- Grams (weight per skein)
- Gauge (numeric selector for stitch count, then unit toggle: per 4 in or per 10 cm; e.g., "18 stitches in 4 in")
- Needle/hook size (adapts per craft type)
- Number of skeins (defaults to 1)
- Care method (multi-select: Machine wash, Hand wash, Dry flat, Tumble dry low, Dry clean, etc.)
- Country of origin
- Source URL
- Images (up to 3)
- Notes (textarea)

**Smart auto-population:** When a user selects a known brand + yarn name combination, the app pre-populates color name options, yards per skein, fiber type, and weight from the shared yarn database. User can override any auto-filled value.

**QR scanner:** Users can scan a yarn label QR code or barcode to auto-fill the entry form. Where QR data is available from the manufacturer, relevant fields are populated automatically. **Research required:** Evaluate 3rd party tools and open barcode databases for reliability and coverage before committing to an implementation approach — see §10 Open Decisions.

**Stash intelligence:**
- Track number of skeins used per project and remaining grams when a skein is partially used
- Auto-prompt to update remaining skeins/grams when a linked project is marked Finished
- Partially used skeins show remaining grams alongside original skein weight

**Stats view:** Total skeins, unique yarn count, estimated total yardage, top brands

---

### 6.3 Tools Inventory

**Core fields:**
- Tool name
- Type (Crochet hook, Knitting needle — circular/straight/DPN, Tapestry needle, Stitch markers, Row counter, Scissors, Blocking mat, Other)
- Brand (searchable combobox, create-new)
- Size (adapts per craft type — hook sizes for crochet; US + metric for knitting needles)
- Material (Aluminum, Steel, Bamboo, Wood, Plastic, Ergonomic resin, Rubber, Silicone + create-new)
- Images
- Notes (textarea)

**QR scanner:** Users can scan a tool's QR code or barcode to auto-fill the entry form where manufacturer data is available.

---

### 6.4 Projects

**Core fields:**
- Project name
- Craft type (drives field adaptation)
- Pattern designer (searchable combobox, create-new)
- Pattern URL
- Status (To do / In progress / Finished)
- Difficulty (Beginner / Advanced Beginner / Intermediate / Advanced / Expert)
- Start date
- Finish date
- Yarn from stash (multi-select, with inline create-new)
- Hook/needle from kit (multi-select, filtered to tools with a size value)
- Images (up to 5)
- Files & patterns (any file type — PDFs, docs)
- **Project notes** (textarea — persistent notes about the project overall; distinct from session notes)
- **Session notes** (timestamped updates/log entries for a working session — "what did I do today on this project")
- **Counters** (multiple named counters — see §6.6)

**Status pipeline:** A clear kanban-style status flow (To do → In progress → Finished). Custom statuses are deferred to a later phase.

**Notes vs. session notes distinction:** Project notes are a persistent scratchpad for the project as a whole. Session notes are timestamped log entries — think of them as a work journal for that project. Both are accessible from the project detail view.

**Linked item behavior:**
- Linked yarn renders as a tappable chip — tap opens a quick-view of the yarn entry
- Linked tools render as size — tap opens a quick-view of the tool entry
- If a stash item is deleted, any project/pin references to it are also removed

---

### 6.5 Pins (Inspiration Board)

A saved inspiration collection — patterns, ideas, and projects-to-make-someday.

**Core fields:**
- Project/pattern name
- Pattern designer (searchable combobox, create-new)
- Source / pattern URL
- Craft type
- Difficulty
- Yarn ideas from stash (multi-select, inline create-new)
- Hook/needle from kit (multi-select, inline create-new)
- Images (up to 5)
- Notes (textarea)

**Layout:** Masonry gallery — visual, image-first browsing

**Push to Projects:** Primary CTA on every Pin detail view — creates a new Project pre-filled with the pin's name, pattern, URL, yarn/tool links, images, and notes. Status set to "To do." The original Pin is retained and marked "Pushed to projects" (copy, not move).

**Long-term vision (post-MVP):** As a user's pin collection grows, Pins evolve toward a Pinterest-style inspiration layer — rich with saved ideas, discoverable patterns, and eventually in-app pattern search and discovery. The stash-to-pin relationship enables yarn suggestions ("you could make X with the yarn you have"). This is the foundation for the intelligence layer in Phase 3+.

---

### 6.6 Counters

Multiple named counters per project, accessible directly from the project view.

**Each counter includes:**
- Custom name (e.g., "Rows," "Repeats," "Pattern Section A," "Increases")
- Current count (tap + to increment, tap − to decrement)
- Reset to zero option (with confirmation)
- Optional target count (shows progress bar/percentage when set)

**Counter behavior:**
- Counters persist per project — never lost between sessions
- Multiple counters can be active simultaneously
- Counter state syncs across devices in real time

**Phase 3 additions (not MVP):** Lock-screen / Live Activity counter, Apple Watch tap, home-screen widget

---

### 6.7 Ravelry Import

A guided import flow for users with existing Ravelry accounts, serving the Ravelry refugee persona.

**Import scope (MVP):**
- Yarn stash → maps to [AppName] stash entities
- Tools/needles → maps to [AppName] tools entities
- Projects → maps to [AppName] project entities

**Import flow:**
1. User connects their Ravelry account via OAuth
2. Import preview screen shows mapped data before committing — user can review and deselect items
3. Field mismatch warnings surface where Ravelry data doesn't map cleanly (user decides how to handle)
4. On confirm, data imports and is immediately synced to cloud

**API constraint:** Ravelry API rate limit is 1 request/second per API key. Import flow must be designed with debouncing and queued requests to stay within limits.

> **🔴 Research required:** Before finalizing the import flow, we need to fully understand what the Ravelry API supports — available endpoints, data scope, OAuth flow, rate limit details, and any known gaps in what can actually be exported. Additionally: **if [AppName] generates revenue, Ravelry charges a commercial API licensing fee.** This cost must be accounted for in the monetization model before committing to this feature as free. See §10 Open Decisions.

**Phase 2:** Ravelry queue/library import → [AppName] Pins

---

### 6.8 Search, Filtering & Sort

All four entity tabs include: search bar + filters + sort + "clear filters" + active-filter-count badge.

**Filter UX approach:** Modern, progressive, and touch-native. The specific interaction pattern (bottom sheet, inline chips, drawer, etc.) is a design decision — what matters at the PRD level is that filters are discoverable without being overwhelming, state is never lost unexpectedly, and the experience feels native to the platform. The implementation should not mimic desktop web filter patterns.

**Filter & sort state persistence:** All active filter and sort state must be persisted and restorable — via URL query params on web, or deep-link state on native (React Native / Flutter). This ensures back-navigation works correctly, states are shareable via deep links, and filter state is never lost on navigation.

**Combined search + filter:** Filtered results must match both the active search query and any applied filters simultaneously — search text and active filters narrow results together, not independently.

**Recent search pre-population:** The most recent search query and last-applied filter state are pre-populated when a user returns to a tab, so nothing is lost on navigation. A "clear" control is always accessible.

**Saved filters:** Users can name and save filter combinations for one-tap reuse (see subscription tier in §7 for access level).

| Tab | Search fields | Free tier filters | Subscription filters | Sort options |
|---|---|---|---|---|
| Stash | Brand, name, color, fiber, notes | Weight, Fiber type | Custom filters | Newest, Oldest, Brand A→Z, Weight, Most yards, Most skeins |
| Tools | Name, brand, size, type, notes | Type, Brand, Size | Custom filters | Newest, Oldest, Type, Brand A→Z, Size |
| Projects | Name, pattern, notes | Status, Difficulty, Designer | Custom filters, saved filter sets | Newest, Oldest, Name A→Z, Difficulty, Start date, Finish date |
| Pins | Name, pattern, notes | Difficulty, Designer | Custom filters | Newest, Oldest |

---

### 6.9 Offline-First Sync

Sync is a **free, core feature** — not a paid add-on.

**Architecture principles:**
- Data-first local storage: all writes go to local DB immediately, never blocked by network state
- Background sync queue: changes made offline are queued and pushed when connectivity returns
- Real-time sync: when online, changes propagate to other signed-in devices in real time via Supabase Realtime
- **Conflict resolution:** Must be better than last-write-wins. Recommended approach: per-field timestamped merge (last-write-wins at the field level, not the document level) with a conflict notification when simultaneous edits are detected on the same field
- Image sync: images sync as part of the same pipeline as data — not a separate manual trigger

**Sync status:** Sync is an implicit, trust-building background experience — no persistent indicator in the UI. The only time sync state surfaces is on error (e.g., "Unable to sync — check your connection") so users are never surprised by data loss, but are never nagged by a status they didn't ask for.

**Multi-device live update:** When the user has the app open simultaneously on multiple devices, edits made on one device should become visible on other open devices without requiring a manual refresh. The mechanism (Supabase Realtime subscriptions, polling, or hybrid) and its behavior under simultaneous active edits require research — particularly around reconnection handling, conflict resolution when both devices are actively writing, and battery/bandwidth impact. *Research required before finalizing this behavior.*

**Pull-to-refresh:** Standard swipe-down gesture triggers a manual refetch of the current view or section. Provides a familiar escape hatch for users who suspect the UI may be stale.

**Backup & restore:** Deferred — not in MVP scope. Will be considered for a future phase based on user feedback.

---

### 6.10 Authentication

**Methods available at launch:**
- Sign in with Apple (required for iOS App Store compliance when any social login is offered)
- Sign in with Google
- Email + password

**Account behavior:**
- Single-user accounts (no sharing, collaboration, or multi-user concepts in MVP)
- Signing in on a new device automatically pulls existing cloud data
- Sign out preserves local data on device until explicitly cleared

> **🔴 Open question:** Do we want to offer a verification code (email/SMS OTP) or full 2-step authentication (TOTP/authenticator app)? Options: (1) no 2FA at launch, (2) optional verification code, (3) optional TOTP. See §10 Open Decisions.

---

## 7. Monetization

### Free Tier (forever free, no time limit)
- Full stash inventory — no cap
- Full tools inventory — no cap
- Projects — capped at **15** (encourages upgrade for active makers)
- Full pins — no cap
- Real-time cross-device sync
- Basic search and filtering
- Multiple named counters per project
- Ravelry import
- Limited image storage *(specific limit TBD — see §10 Open Decisions)*
- Apple Watch counter (Phase 3)
- Home screen widgets (Phase 3)
- Lock-screen / Live Activity counters (Phase 3)

### Subscription Tier
- Unlimited projects (no cap)
- Expanded / unlimited image storage
- Advanced filtering + custom saved filter sets
- AI features and pattern parsing (Phase 3+)

> **Pricing TBD.** Recommended reference points: Loopsy (~$3.99/mo), LooseLoop (~$4.99/mo). Subscription fatigue is a documented pain point in this market — price should feel fair and features behind the paywall should feel genuinely premium, not artificially withheld.

> **🔴 Cost open questions (resolve before finalizing monetization):**
> - **Ravelry import:** Does the Ravelry commercial API license have a per-user or revenue-share cost? What does it cost at scale?
> - **Image storage:** At projected free tier volume, what does Supabase Storage actually cost? What's the break-even limit per user?
> - **AI features & pattern parsing:** What would per-user LLM/OCR API costs look like at 1K / 10K / 100K users? Are these viable within a subscription at $3.99–4.99/mo?
> See §10 Open Decisions.

---

## 8. Navigation & Information Architecture

### Bottom navigation (phone) / Sidebar (tablet):
1. **Stash** — yarn inventory + stats
2. **Tools** — hook/needle/tool inventory
3. **Projects** — project tracker with status pipeline
4. **Pins** — inspiration board (masonry gallery)
5. **Profile / Settings** — account, subscription management

### Persistent elements:
- Quick-add FAB (floating action button) on all main tabs
- Active tab persists across sessions

---

## 9. UX Principles

These are non-negotiable design constraints, not aspirational goals:

1. **Craft-adaptive** — the interface adapts to the user's craft, not the other way around
2. **Offline-first** — the app is fully functional without a network connection, always
3. **Zero silent failures** — sync errors surface clearly when they occur; nothing breaks without telling the user
4. **Accessibility is a requirement, not a feature** — [AppName] targets **WCAG 2.2 AA** compliance at launch. This means: sufficient color contrast ratios, touch targets that meet minimum size guidelines, full support for `prefers-reduced-motion`, dark mode, scalable font sizes, and screen reader compatibility. Accessibility is never deferred to a later phase.
5. **System appearance by default** — light and dark mode automatically match the user's device preferences (`prefers-color-scheme`). Users may override manually in settings, but the default is always system-synced.
6. **Low friction onboarding** — a new user with zero existing data should reach their first meaningful action (adding a stash item or starting a project) in under 60 seconds
7. **No penny-pinching** — the free tier is genuinely useful; subscription unlocks feel premium, not like features that were artificially removed

---

## 10. Open Decisions

These items must be resolved before development begins. Flagged for founder alignment meeting.

| # | Decision | Status | Owner |
|---|---|---|---|
| 1 | **App name** | 🔴 OPEN — naming exercise needed | Both |
| 2 | **Platform: React Native/Expo, mobile/tablet first, native-first** | ✅ RESOLVED — React Native/Expo, mobile + tablet first | — |
| 3 | **Desktop timing** | ✅ RESOLVED — Phase 2 | — |
| 4 | **Phase 3 platform additions** | ✅ RESOLVED — Apple Watch, lock screen widgets, Live Activity in Phase 3; Android as fast-follow to MVP | — |
| 5 | **v1 scope: inventory-first** | ✅ RESOLVED — inventory-first (stash/tools/projects/pins + sync) | — |
| 6 | **Free tier image storage limit** | 🔴 OPEN — suggest 100 images or ~500MB | Both |
| 7 | **Subscription pricing** | 🔴 OPEN — ~$3.99–4.99/mo; consider annual option | Both |
| 8 | **Unauthenticated browsing** | ✅ RESOLVED — account required even for free tier; no sign-in-free usage supported | — |
| 9 | **QR / barcode scanner for yarn & tools** | 🔴 OPEN — research required: (1) reliable open yarn UPC/QR database options; (2) Ravelry API barcode lookup capability; (3) 3rd party scanning tools. If no solid solution, defer to Phase 2 | Diana (technical research) |
| 10 | **Adaptive language / dynamic forms research** | 🔴 OPEN — research how craft-adaptive terminology and dynamic form patterns work in practice across yarn suggestion flows (e.g., how do fields change based on stash selection vs. craft type selection?) | Both |
| 11 | **Custom project statuses** | ✅ RESOLVED — deferred to a later phase; MVP ships with To do / In progress / Finished only | — |
| 12 | **Ravelry API research** | 🔴 OPEN — research what endpoints and data the API supports, OAuth flow, rate limit details, and commercial licensing fee if app generates revenue | Diana (technical research) |
| 13 | **2FA / account verification** | 🔴 OPEN — decide: no 2FA at launch, optional email/SMS verification code, or optional TOTP authenticator app | Both |
| 14 | **Infrastructure & feature costs** | 🔴 OPEN — quantify: (1) Ravelry commercial API cost at scale, (2) image storage cost per user at free tier volume, (3) AI/OCR API cost per user at $3.99–4.99/mo price point | Both |

---

## 11. Post-MVP Roadmap (Phases)

### Phase 2 — Desktop & Pattern Depth
- Desktop web app (React Native Web / Expo for Web)
- Pattern PDF import with text editing, highlighting, and in-pattern notes (text-selectable PDFs first)
- Ravelry queue/library → Pins import
- Android (fast-follow, if not already shipped)
- In-app craft glossary with tutorial links and terminology examples
- Additional craft types (based on user feedback)
- Custom project statuses
- Backup & restore (export/import JSON) — if user feedback warrants it

### Phase 3 — OS Integrations & Intelligence
- Lock-screen / Live Activity counters
- Apple Watch companion
- Home screen widgets
- Yarn stash suggestions (AI-assisted recommendations based on stash)
- Pattern search and discovery from Pins
- AI features: "Do I have enough yarn?" stash check, pattern assistant
- OCR / vision-based scanned/photographed pattern parsing

### Phase 4 — Community & Marketplace
- Public project profiles
- Social sharing (project stat cards exportable to social media)
- Follow other crafters
- Browse public projects and inspiration
- Community feed
- Designer marketplace with payment management

---

## 12. Competitive Positioning Summary

| What the market does poorly | What [AppName] does instead |
|---|---|
| Sync is manual, paid, or broken | Real-time sync free for everyone |
| Apps are crochet-only or knitting-only | Craft-adaptive — built for all fiber artists |
| Stash and pattern-following are separate apps | Unified in one coherent experience |
| Subscription locks core functionality | Free tier is genuinely useful; subscription unlocks real extras |
| Mobile is an afterthought | Phone + tablet first, always |
| Importing from Ravelry is unreliable or absent | Guided, preview-first Ravelry import as a first-class feature |
| Accessibility is a blind spot | Inclusive design ships at launch |

---

*Document status: Pre-alignment draft. Do not treat confirmed decisions as final until both founders have reviewed and signed off.*
