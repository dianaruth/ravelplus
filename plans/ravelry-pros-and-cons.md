# Ravelry.com — Pros & Cons Analysis

**Purpose:** Reference document for RavelPlus design decisions. Captures what Ravelry does well (to preserve or exceed) and where it falls short (our primary opportunities to differentiate).

---

## Pros — What Ravelry Does Well

### Data & Content Depth
- Enormous pattern library with rich metadata: craft type, yarn weight, gauge, needle size, yardage, difficulty, language availability (15+ languages on popular patterns)
- Community-sourced project photos give real-world context beyond the designer's photos
- Ratings on two dimensions: overall quality + pattern clarity — genuinely useful signal
- Project counts (37k+ on popular patterns) communicate popularity and viability at a glance
- Tag system (bias, reversible, triangle-shaped, etc.) enables nuanced discovery beyond category dropdowns

### Community Depth
- Forum posts, blog posts, and comments all linked directly from pattern pages — the pattern is a hub, not just a document
- Yarn substitution ideas (crowdsourced) on each pattern's "Yarn ideas" tab
- Designer profiles with cross-linked patterns and project galleries

### Feature Completeness (for power users)
- Queue, library, favorites — the three core collection types we're also building
- Notebook (projects with yarn/needle tracking) — our v1.1 roadmap item
- Yarn stash tracking
- Group and forum integration

### Information Architecture on Pattern Detail
- Breadcrumb navigation (Patterns > Designer > Pattern Name)
- Tabbed layout keeps the page from being overwhelming — details, projects, discussion are segmented
- "Yarn ideas" tab with specifications is genuinely useful for planning

### Pricing Transparency
- Clear price display with currency and approximate USD conversion
- Bundle purchase options shown alongside individual purchase

---

## Cons — Where Ravelry Falls Short (Our Opportunities)

### Mobile Experience
- The site is not mobile-first; it was designed for desktop and adapted
- Navigation, filter panels, and dense sidebar layouts don't reflow well on small screens
- Image thumbnails require tapping through rather than a native swipe gallery
- No dedicated mobile app

### Search & Filtering UX
- Filter interface is overwhelming — dozens of options with no progressive disclosure
- No saved searches or filter presets
- Filter state is not always URL-synced, breaking back-button behavior and shareability
- Results layout is a basic grid with limited sorting options
- No real-time filter updates — requires explicit form submission in some flows

### Visual Design
- Design is visually dated — dense, information-heavy, low contrast in places
- Typography is small and compressed, especially in metadata sections
- Color palette and layout feel like early-2010s web design
- The infamous 2020 redesign (high-contrast black/white) caused widespread accessibility complaints from users with visual sensitivities (migraines, photosensitivity) — Ravelry has not fully recovered community trust on this front
- No dark mode

### Accessibility
- The 2020 redesign was specifically criticized for causing harm to users with visual impairments and photosensitivity — a major, ongoing community trust issue
- Small default font sizes throughout
- Tab duplication on pattern pages (nav appears twice) — redundant and confusing

### Onboarding & Discovery
- Homepage is just a login form — zero pattern discovery before signing in
- No browsable "trending" or "featured" content for logged-out users
- Sign-up requires an invitation (waitlist) — creates friction for new users

### Pattern Detail Page Clutter
- "Viewing as guest" nag prompt appears without enough context
- Seven tabs on pattern detail page can feel overwhelming
- Related content (forum posts, blog posts) is surfaced on the pattern page but with no curation — volume without quality signal

### Performance
- Page loads are slow, especially pattern search with large result sets
- Heavy JavaScript payloads from an aging codebase

### No Native Collections UX
- Adding to queue/library/favorites requires knowing the feature exists — no guided "what do you want to do with this pattern?" flow
- No visual differentiation between queue/library/favorites on the pattern card in search results

---

## Specific Patterns to Replicate in RavelPlus

| Ravelry Pattern | Our Implementation |
|---|---|
| Two-axis ratings (quality + clarity) | Show both on pattern cards and detail pages |
| Project count as social proof | Display prominently in search results |
| Tabbed pattern detail (details / projects / discussion) | Adapt with mobile-first tab pattern |
| Tag-based discovery | Include top tags as filter chips in search |
| Yarn requirements clearly displayed | Yardage + weight + needle size above the fold |

## Specific Anti-Patterns to Avoid

| Ravelry Failure | Our Fix |
|---|---|
| Login wall on homepage | Allow pattern browsing without auth (auth required for collections only) |
| Overwhelming filter sidebar | Progressive disclosure — core filters visible, advanced filters behind "More filters" |
| No URL-synced filters | All filter state in URL query params from day one |
| Dense desktop-first layout | Mobile-first layout, test on real iPhone before each phase ships |
| No guided add-to-collection flow | Contextual bottom sheet: "Save this pattern" with queue/library/favorites options |
| Accessibility complaints from redesign | Follow WCAG 2.1 AA from the start; offer light/dark mode; respect `prefers-reduced-motion` |

---

## Open Questions for Sheryl

- Should pattern browsing (without auth) be a v1 scope item, or auth-required from day one given the friends-only intent?
- How much of Ravelry's tab structure (projects, discussion, yarn ideas) do we replicate in v1 vs. later phases?
- Should we show community project photos pulled from the Ravelry API, or only designer photos in v1?
