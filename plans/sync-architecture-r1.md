# Offline-First Sync — Architecture Notes & Spike Plan (R1)

**Status:** Architecture understood; engine decision pending the spike below.
**Owner:** Diana · **Blocks:** Phase 3 step 5 (sync under stash/tools) — not the scaffold, auth, or UI work.
**Requirements (PRD §6.9):** local-first writes · background queue · real-time propagation · per-field LWW merge (not document-level) · conflict notification on same-field collisions · images in the same pipeline · error-only sync surfacing.

---

## How cross-device sync actually works

### The core mental model: the UI never talks to the network

Every screen reads from and writes to the **local SQLite database only**. A phone doesn't send an update "to the tablet," and the tablet doesn't fetch from the server to render. Two independent mechanisms meet in the middle:

```
Phone UI ──write──▶ phone SQLite ──sync engine──▶ Postgres (Supabase)
                                                      │
Tablet UI ◀──reactive query── tablet SQLite ◀──sync engine──┘
```

1. **Sync engine (DB ↔ DB):** moves changes between each device's SQLite and Postgres — uploads from the editing device, downloads to every other device.
2. **Reactive queries (DB → UI):** screens *watch* queries (Drizzle `useLiveQuery` / PowerSync watched queries) instead of doing one-time reads. When the sync engine writes rows into local SQLite, watching screens re-render automatically. The UI never knows sync happened; it just sees its local DB change.

Consequence: **no refresh or pull-to-refresh is needed for liveness.** The PRD's pull-to-refresh (§6.9) is a trust-building escape hatch only.

### The websocket: what makes it "immediate"

The download half needs to know *when* to pull. Polling on a timer is laggy and battery-hungry; the live feel comes from a **persistent websocket the server pushes over** — the same mechanism as any chat app, and cheap when idle (keepalive pings). Both candidates use one:

- **Supabase Realtime** (custom-build path): the device subscribes to a per-user channel; a committed Postgres change is pushed as a `postgres_changes` event within a few hundred ms.
- **PowerSync**: each device holds a streaming connection to the PowerSync service, which tails Postgres's replication log and streams committed changes straight into the device's SQLite.

**Latency, both devices foregrounded:** local write (instant UI on the editing device) → upload (~100–500 ms) → push → other device's SQLite → re-render. **Typically < 2 s, often sub-second.** Counters — our highest-frequency entity — feel close to live.

### The honest caveats

- **"Immediate" applies while foregrounded.** iOS suspends backgrounded apps and their sockets. A sleeping tablet receives nothing; on wake, the engine reconnects and does a **catch-up pull from a cursor/checkpoint** ("everything since sequence N"). Checkpoint logic matters as much as the websocket.
- **Push is a hint; pull is the truth.** Supabase Realtime delivery is best-effort — a 10-second socket drop means missed events. A correct design treats the push only as *pull now*; cursor-based catch-up is what guarantees consistency. PowerSync bakes this checkpointing in (a big reason it's candidate #1).
- **Offline is the same path, delayed.** Offline edits queue in a local outbox; reconnect uploads them; catch-up covers the download side. No special cases.
- **Simultaneous edits** are where per-field merge earns its keep: phone edits a project's notes while tablet renames it → each field resolves on its own timestamp → both devices converge to name-from-tablet + notes-from-phone. Document-level LWW would silently discard one side — exactly what the PRD forbids. Same-field collisions get a deterministic winner plus a conflict notification.

### What the decision actually is

Not "websocket or not" (both use one) — it's **who writes the checkpointing, outbox, and merge logic**:

| | PowerSync | Custom on Supabase Realtime |
|---|---|---|
| Connection mgmt, cursors, catch-up, local persistence | Built in | All ours |
| Upload path | Our hook via supabase-js (RLS applies) | Ours via supabase-js (RLS applies) |
| Merge policy | Ours, in the upload/apply hooks — expressiveness to verify | Fully ours |
| Cost | Free tier / self-host — **verify against $0 rule** | $0 (within Supabase free tier) |
| Risk profile | Vendor dependency | Most intricate code in the app; easiest place to ship subtle data-loss bugs |
| Web support (disqualifying criterion) | Dedicated web SDK | expo-sqlite WASM/OPFS — verify |

---

## Spike plan

**Goal:** pick the engine with evidence, via a throwaway two-device demo syncing `counters` (smallest schema, highest edit frequency, exercises every hard path).

**Timebox:** ~3–4 days for Phase A; only if it fails, ~3–4 days for Phase B. Output is a decision, not production code.

### Phase A — PowerSync

- **A0 (half-day, desk check — bail early if it fails):** current pricing/self-host terms vs the $0 rule; Expo Go vs dev-build requirement; per-field merge expressiveness in upload/apply hooks; web SDK status.
- **A1:** local Supabase (`supabase start`) + `counters` table (id, user_id, name, count, updated_at) + RLS policies + two test users.
- **A2:** PowerSync service (free tier or self-host docker) connected to the local Postgres; sync rules scoping rows to the authenticated user.
- **A3:** throwaway Expo app: sign-in, counter list via watched queries, tap-to-increment writing locally; upload hook posting through supabase-js.
- **A4:** run the test matrix below on two devices (physical iPhone + browser via the web target — doubles as the web-compat check).

### Phase B — custom outbox (only if A fails)

Outbox table of **field-level** change rows `(table, row_id, field, value, changed_at)` → push via supabase-js on connectivity → Supabase Realtime per-user channel as the pull-hint → cursor-based catch-up pull → pure, unit-tested merge function applying per-field LWW + tombstones.

### Test matrix (pass/fail — same matrix for either phase)

| # | Scenario | Pass condition |
|---|---|---|
| 1 | Both devices online, tap on A | B updates **< 2 s**, no user action |
| 2 | A offline, 10 taps, reconnect | B converges; zero lost taps |
| 3 | Both offline, edit **different fields** of the same row, reconnect | Both fields survive (per-field merge) — **hard requirement** |
| 4 | Both offline, edit the **same field**, reconnect | Deterministic winner; conflict detectable for notification |
| 5 | Kill app mid-sync, relaunch | No duplicates, no loss (checkpoint integrity) |
| 6 | Drop socket silently (airplane-mode blip), edit elsewhere meanwhile | Catch-up pull recovers the missed change (push-is-hint verified) |
| 7 | Same demo in the browser (web target) | Works — web compat is disqualifying if absent |
| 8 | User B's device with user A's data flowing | B never receives A's rows (RLS on the sync path) |
| 9 | Cost check after a week of dev usage | $0 across Supabase + engine |

### Deliverable

One-page decision writeup: chosen engine, matrix results, accepted constraints, and the schema consequences (exact sync-metadata columns / change-log table the production schema needs — feeds Phase 1's Drizzle schema v0). Then the demo gets thrown away.

### Open questions the spike will answer

- Does PowerSync's model express per-field merge, or does it force row-level apply?
- Expo Go or dev build? (Affects how early the $99 Apple enrollment matters for on-device testing — web + Android emulator may suffice during the spike.)
- Real reconnect behavior on iOS backgrounding (matrix #5/#6 on the physical iPhone).
- Whether images/attachments ride the same pipeline or need a separate upload queue (both candidates likely need the separate queue — confirm).
