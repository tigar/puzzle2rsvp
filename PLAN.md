# Puzzle RSVP — Project Plan

## Problem Statement

Build a website where each guest receives a unique invite link to an event. The link presents a puzzle that, when solved, unlocks an RSVP page. Admins manage invite links and view RSVP status per event. New events (and their puzzles) are deployed via code push — the admin dashboard is solely a link generator and status viewer.

---

## Backend Recommendation

Three viable options, ranked by fit:

| Option | Why It Fits | Tradeoff |
|---|---|---|
| **Supabase (Postgres + Edge Functions)** | Auth, DB, and serverless functions in one box. Row-level security maps cleanly to invite-scoped access. Generous free tier. | Vendor coupling to Supabase SDK patterns. |
| **Express + SQLite (deployed on Fly.io or Railway)** | Maximum control, single-process simplicity. SQLite is more than enough for this scale. Trivial to reason about. | You own the ops. No managed auth — roll your own JWT signing (straightforward given your stack skills). |
| **Cloudflare Workers + D1 (SQLite at edge)** | Near-zero cold starts, D1 is SQLite-based so migrations are simple. Workers are cheap. | D1 is still maturing. Less ecosystem tooling than Postgres. |

**Recommendation: Express + SQLite on Fly.io** — it aligns with doing one thing well at each layer, gives you full control over the puzzle verification logic, and avoids introducing managed-service abstractions for a system this size. A single `data.db` file holds everything. If you later outgrow it, migrating to Postgres is a well-worn path.

If you'd rather minimize ops entirely: **Supabase** is the pragmatic second choice — Edge Functions handle puzzle verification, and the dashboard practically builds itself on top of their client SDK.

---

## System Architecture

```
┌─────────────────────────────────────────────────┐
│                   Client (React)                │
│                                                 │
│  ┌─────────────┐  ┌────────────┐  ┌──────────┐ │
│  │ Puzzle View  │  │ RSVP View  │  │  Admin   │ │
│  │ (per-event)  │  │            │  │Dashboard │ │
│  └──────┬───────┘  └─────┬──────┘  └────┬─────┘ │
└─────────┼────────────────┼───────────────┼───────┘
          │                │               │
          ▼                ▼               ▼
┌─────────────────────────────────────────────────┐
│              Express API Server                 │
│                                                 │
│  GET  /api/invite/:token                        │
│  POST /api/puzzle/:eventSlug/verify             │
│  POST /api/rsvp/:inviteToken                    │
│  GET  /api/admin/events                         │
│  GET  /api/admin/events/:slug/invites           │
│  POST /api/admin/events/:slug/invites           │
│                                                 │
│  ┌────────────────────────────────────────────┐ │
│  │  Puzzle Verifiers (one module per event)   │ │
│  │  Each exports: verify(submission) → bool   │ │
│  └────────────────────────────────────────────┘ │
│                                                 │
│  SQLite ─── data.db                             │
└─────────────────────────────────────────────────┘
```

Think of it like a circuit with a relay: the puzzle is a normally-open switch. The server is the only component that can close the relay (mark `puzzle_solved = true`). The client can wiggle the switch all it wants — the relay only closes when the server's verifier confirms a valid solution.

---

## Data Model

```sql
-- Events are defined in code and seeded on deploy
CREATE TABLE events (
  slug        TEXT PRIMARY KEY,       -- e.g. "dinner-party-march-2026"
  title       TEXT NOT NULL,
  event_date  TEXT,                   -- ISO 8601
  is_active   INTEGER DEFAULT 1       -- 1 = active, 0 = archived
);

-- Each row is one invite link (may cover multiple guests)
CREATE TABLE invites (
  token         TEXT PRIMARY KEY,     -- UUID v4, used in the invite URL
  event_slug    TEXT NOT NULL REFERENCES events(slug),
  guest_name    TEXT NOT NULL,        -- display name ("Claude" or "Claude & Dario")
  puzzle_solved INTEGER DEFAULT 0,
  rsvp_status   TEXT DEFAULT NULL,    -- NULL = no response, "accepted", "declined"
  rsvp_data     TEXT DEFAULT NULL,    -- JSON blob, schema varies per event
  created_at    TEXT DEFAULT (datetime('now')),
  solved_at     TEXT DEFAULT NULL,
  rsvp_at       TEXT DEFAULT NULL
);
```

The invite `token` is the only credential a guest needs. It scopes them to exactly one event and one identity. No user accounts, no passwords — the token *is* the session. This keeps the auth model as simple as a key fitting a lock.

### RSVP Field Configuration

Each event defines which RSVP form fields are active. A lightweight approach: each event's `seed.js` exports an `rsvpFields` array that declares the form shape. This could evolve into a shared field registry where events toggle fields on/off rather than defining them from scratch.

```js
// events/dinner-party-march-2026/seed.js
export default {
  slug: "dinner-party-march-2026",
  title: "Dinner Party",
  event_date: "2026-03-15",
  rsvpFields: [
    { key: "attending", type: "select", label: "Will you attend?", options: ["Yes", "No"] },
    { key: "dietary", type: "text", label: "Any dietary restrictions?" },
    { key: "plusOne", type: "select", label: "Bringing a +1?", options: ["Yes", "No"] },
  ],
};
```

The RSVP form component reads this field config and renders accordingly. The server stores whatever JSON the form submits — it doesn't validate the shape beyond requiring `puzzle_solved`. This keeps the server simple (one job: gate on puzzle completion) while the client handles per-event form rendering.

---

## Puzzle Verification — Server-Side Strategy

This is the critical design constraint. The puzzle must feel interactive on the client, but the "answer accepted" decision must live on the server.

### Pattern: Submit-and-Verify

```
Client                          Server
  │                               │
  │  User interacts with puzzle   │
  │  ·························>   │
  │                               │
  │  POST /puzzle/:slug/verify    │
  │  { token, submission }        │
  │  ─────────────────────────>   │
  │                               │
  │        verify(submission)     │
  │        ┌──────────────┐       │
  │        │ Event-specific│       │
  │        │ verifier fn   │       │
  │        └──────┬───────┘       │
  │               │               │
  │  { solved: true/false }       │
  │  <─────────────────────────   │
  │                               │
  │  If solved: show RSVP form    │
  │                               │
```

### How Each Puzzle Verifier Works

Each event ships with a verifier module on the server:

```
server/
  verifiers/
    dinner-party-march-2026.js
    scavenger-hunt-july-2026.js
    index.js          ← maps event slug → verifier function
```

Each verifier exports a single function:

```js
// verifiers/dinner-party-march-2026.js

export function verify(submission) {
  // Example: a word puzzle where the answer is a specific phrase
  const normalized = submission.answer?.trim().toLowerCase();
  return normalized === "the stars align at midnight";
}
```

The `index.js` barrel file maps slugs to verifiers:

```js
import { verify as dinnerPartyMarch } from "./dinner-party-march-2026.js";

export const verifiers = {
  "dinner-party-march-2026": dinnerPartyMarch,
};
```

The API route is then trivial. Rate-limited to 10 requests/minute per token to prevent brute-forcing short answers:

```js
import rateLimit from "express-rate-limit";

const puzzleRateLimit = rateLimit({
  windowMs: 60 * 1000,
  max: 10,
  keyGenerator: (req) => req.body.token,
  message: { error: "Too many attempts. Wait a moment before trying again." },
});

app.post("/api/puzzle/:slug/verify", puzzleRateLimit, (req, res) => {
  const { slug } = req.params;
  const { token, submission } = req.body;

  const invite = db.getInvite(token);
  if (!invite || invite.event_slug !== slug) return res.status(404).json({ error: "Invalid invite" });
  if (invite.puzzle_solved) return res.json({ solved: true }); // idempotent

  const verifier = verifiers[slug];
  if (!verifier) return res.status(404).json({ error: "Unknown event" });

  const solved = verifier(submission);
  if (solved) {
    db.markSolved(token);
  }

  return res.json({ solved });
});
```

### Preventing Client-Side Bypass

The RSVP endpoint independently checks `puzzle_solved` before accepting any submission. It uses upsert semantics — submitting again overwrites the previous response:

```js
app.post("/api/rsvp/:token", (req, res) => {
  const invite = db.getInvite(req.params.token);
  if (!invite) return res.status(404).json({ error: "Invalid invite" });
  if (!invite.puzzle_solved) return res.status(403).json({ error: "Puzzle not yet solved" });

  db.upsertRsvp(invite.token, req.body);
  return res.json({ success: true });
});
```

Even if someone inspects network traffic or manipulates client state, the server won't accept an RSVP unless `puzzle_solved` is already `true` in the database. The client rendering the RSVP form is cosmetic — the enforcement is server-side.

### RSVP View Behavior — PII Protection

Once a puzzle is solved, the invite URL **always** shows the RSVP form — regardless of whether the guest has already submitted. This protects PII: a solved link never displays a read-only summary of someone's responses (dietary restrictions, contact info, etc.) that could be seen by anyone with the URL.

If an RSVP already exists for the token, the form renders with a banner: *"You've already submitted an RSVP. Submit the form again to update your response."* The form fields are **not** pre-filled with previous data (again, PII protection — the link holder may not be the original respondent). The server simply overwrites on re-submission.

The `GET /api/invite/:token` endpoint returns only what the client needs to render:

```js
app.get("/api/invite/:token", (req, res) => {
  const invite = db.getInvite(req.params.token);
  if (!invite) return res.status(404).json({ error: "Invalid invite" });

  return res.json({
    guest_name: invite.guest_name,
    event_slug: invite.event_slug,
    puzzle_solved: !!invite.puzzle_solved,
    has_rsvp: invite.rsvp_status !== null, // boolean only — no PII leaked
  });
});
```

### Puzzle Design Guidelines (for Future Events)

To keep puzzles verifiable server-side, each puzzle should resolve to a **discrete, deterministic answer** — a word, phrase, number, sequence, or set of selections. Avoid puzzles whose "solution" is purely a client-side state (like "drag all pieces into place") unless the final arrangement maps to a verifiable hash or answer string.

Examples that work well:

- **Word/phrase answer** — crossword, riddle, cipher → submit the decoded phrase
- **Sequence answer** — reorder items → submit the ordering as an array
- **Code/combination** — puzzle reveals digits → submit the code
- **Selection set** — choose the correct items from a grid → submit selected IDs

---

## Routing & View Structure

```
/invite/:token              → Puzzle page OR RSVP form (server state decides)
/admin                      → Login (simple shared password or env-based)
/admin/events/:slug         → Invite list + generate new links
```

The `/invite/:token` route is the only guest-facing URL. On load, the client fetches `GET /api/invite/:token` — if `puzzle_solved` is true, it renders the RSVP form directly. No separate RSVP route needed; the token's server-side state is the single source of truth for what the guest sees.

### React Component Tree (Simplified)

```
<App>
  ├── <InvitePage token={token}>
  │     ├── <PuzzleShell>            ← loads the event-specific puzzle component
  │     │     └── <DinnerPuzzle />   ← each event has its own puzzle component
  │     └── <RsvpForm>               ← rendered once server confirms solved
  │           └── <RsvpBanner />     ← "You've already submitted..." if has_rsvp
  │
  └── <AdminLayout>
        ├── <Sidebar>                ← event list (active + past)
        └── <EventDashboard>
              ├── <InviteTable />    ← name, solved?, rsvp status
              └── <GenerateInvite /> ← form: guest name(s) → new link
```

Puzzle components are code-split per event. Each event folder contains both its React puzzle component and its server-side verifier — keeping them co-located makes the "code push = new event" workflow clean:

```
events/
  dinner-party-march-2026/
    Puzzle.jsx           ← client-side interactive puzzle
    verifier.js          ← server-side answer check
    RsvpPage.jsx         ← custom RSVP layout for this event
    seed.js              ← event metadata for DB seeding
```

---

## Admin Authentication

Keep it simple: a single admin password stored as an environment variable. The admin login endpoint issues a short-lived JWT. Middleware protects all `/admin/*` API routes.

This avoids building a user management system for what is likely one or two people. If you later need multi-admin with different roles, that's a clear upgrade path — but not a day-one requirement.

---

## Adding a New Event (Deployment Workflow)

1. Create a new folder under `events/` with the four files (Puzzle, verifier, RsvpPage, seed)
2. Register the verifier in `server/verifiers/index.js`
3. Register the puzzle component in the client-side route/loader
4. Add the event seed data (slug, title, date)
5. Push to main → deploy → seed script inserts the new event row

This is intentionally not a CMS. Each event is a bespoke experience — the puzzle *is* the product. Treating events as code artifacts keeps them version-controlled and reviewable.

---

## Tech Stack Summary

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React (Vite) | Your core strength. Vite for fast dev cycles. |
| Routing | React Router | Standard, handles token-based routes cleanly. |
| Backend | Express.js + `express-rate-limit` | Minimal, does one thing. Rate limiting on puzzle endpoint. |
| Database | SQLite (via `better-sqlite3`) | Single file, zero config, synchronous reads are fine at this scale. |
| Hosting | Fly.io | Single-region is fine. Persistent volume for SQLite. Low cost. |
| Admin Auth | JWT + env password | Simplest thing that works. |
| Guest Auth | Invite token in URL | No accounts needed. Token = identity. |

---

## Resolved Decisions

| Decision | Resolution |
|---|---|
| Rate limiting | 10 requests/minute per invite token on the puzzle verify endpoint |
| RSVP data shape | JSON blob, schema varies per event. Field config lives in each event's seed file. May evolve into a shared field registry with per-event toggles. |
| Multiple guests per link | One link, one RSVP covering all guests on that invite. `guest_name` can be "Claude & Dario". |
| Post-RSVP behavior | Solved links always show the RSVP form (never a read-only summary) to protect PII. Banner indicates if a previous response exists. Form is never pre-filled. Re-submission overwrites. |

---

## Future Work

Ideas to revisit after the core system is stable:

- **Notifications** — email/SMS on RSVP submission via a webhook to Resend or Twilio. Keep it decoupled: a post-RSVP hook that fires asynchronously, no tight coupling to the RSVP write path.
- **Link expiration** — optional per-event TTL or auto-expire after the event date. Enforce in middleware so expired tokens return a friendly "this event has passed" page.
- **RSVP field registry** — shared catalog of common fields (dietary, +1, accessibility needs) that events can toggle on/off rather than redefining. Reduces boilerplate across events.
- **Admin multi-user** — role-based access if more than one or two people need admin. Upgrade path from single env password to proper auth.
- **Analytics** — puzzle completion rates, time-to-solve distributions, RSVP conversion funnels. Useful for calibrating puzzle difficulty.