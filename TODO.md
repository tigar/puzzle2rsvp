# Puzzle RSVP — Implementation TODO

Derived from [PLAN.md](PLAN.md). Check items off as they are completed.

---

## 1. Project Scaffolding

- [x] Initialize repo with `.gitignore`
- [x] Create `PLAN.md`
- [x] Create `TODO.md` (this file)
- [x] Create `package.json` with dependencies and scripts
- [x] Create Vite + React boilerplate ("hello puzzler" dummy app)
- [x] Create minimal Express server with health-check route
- [ ] Create `.env.example` with required env vars (`ADMIN_PASSWORD`, `JWT_SECRET`, `PORT`)

---

## 2. Database Layer

- [ ] Create `server/db.js` — initialize `better-sqlite3`, open/create `data.db`
- [ ] Create `events` table (slug, title, event_date, is_active)
- [ ] Create `invites` table (token, event_slug, guest_name, puzzle_solved, rsvp_status, rsvp_data, timestamps)
- [ ] Implement DB helper functions:
  - [ ] `getInvite(token)`
  - [ ] `markSolved(token)`
  - [ ] `upsertRsvp(token, data)`
  - [ ] `getEventInvites(slug)`
  - [ ] `createInvite(eventSlug, guestName)`
  - [ ] `getEvents()`
- [ ] Create `server/seed.js` — reads event seed files and inserts/updates event rows

---

## 3. Server — API Routes

- [ ] `GET /api/health` — health check (already in scaffold)
- [ ] `GET /api/invite/:token` — return guest_name, event_slug, puzzle_solved, has_rsvp (no PII)
- [ ] `POST /api/puzzle/:slug/verify` — rate-limited (10/min per token), delegates to verifier
- [ ] `POST /api/rsvp/:token` — accepts RSVP JSON only if puzzle_solved is true, upsert semantics
- [ ] `GET /api/admin/events` — list all events (protected)
- [ ] `GET /api/admin/events/:slug/invites` — list invites for an event (protected)
- [ ] `POST /api/admin/events/:slug/invites` — generate new invite link (protected)
- [ ] `POST /api/admin/login` — validate password, issue JWT

---

## 4. Server — Middleware & Auth

- [ ] Rate limiter on puzzle verify endpoint (`express-rate-limit`, 10 req/min per token)
- [ ] Admin auth middleware — verify JWT on all `/api/admin/*` routes
- [ ] CORS configuration (allow Vite dev server origin in development)

---

## 5. Puzzle Verifier System

- [ ] Create `server/verifiers/index.js` — barrel file mapping event slug to verifier function
- [ ] Create example verifier: `server/verifiers/dinner-party-march-2026.js`
- [ ] Document the verifier interface: `verify(submission) → boolean`

---

## 6. Event System

- [ ] Create example event folder: `events/dinner-party-march-2026/`
  - [ ] `Puzzle.jsx` — client-side interactive puzzle component
  - [ ] `verifier.js` — server-side answer check
  - [ ] `RsvpPage.jsx` — custom RSVP layout for this event
  - [ ] `seed.js` — event metadata + rsvpFields config
- [ ] Wire event seed files into `server/seed.js`

---

## 7. Client — Routing & Pages

- [ ] Install and configure React Router in `src/main.jsx`
- [ ] Routes:
  - [ ] `/invite/:token` — InvitePage
  - [ ] `/admin` — AdminLogin
  - [ ] `/admin/events/:slug` — EventDashboard
- [ ] 404 / catch-all route

---

## 8. Client — Invite Flow

- [ ] `src/pages/InvitePage.jsx` — fetches `GET /api/invite/:token`, decides puzzle vs RSVP
- [ ] `src/components/PuzzleShell.jsx` — loads event-specific puzzle component (code-split)
- [ ] `src/components/RsvpForm.jsx` — renders fields from event rsvpFields config
- [ ] `src/components/RsvpBanner.jsx` — "You've already submitted..." banner when has_rsvp is true
- [ ] Submit puzzle answer via `POST /api/puzzle/:slug/verify`
- [ ] Submit RSVP via `POST /api/rsvp/:token`

---

## 9. Client — Admin Dashboard

- [ ] `src/pages/AdminLogin.jsx` — password form, store JWT in memory/sessionStorage
- [ ] `src/layouts/AdminLayout.jsx` — sidebar with event list
- [ ] `src/pages/EventDashboard.jsx` — invite table (name, solved?, rsvp status)
- [ ] `src/components/GenerateInvite.jsx` — form to create new invite, display generated link
- [ ] Auth context/hook for admin JWT management

---

## 10. Styling & UX

- [ ] Base styles / CSS reset
- [ ] Responsive layout for invite pages (mobile-first — guests will use phones)
- [ ] Admin dashboard layout (sidebar + content area)
- [ ] Loading and error states for all async operations
- [ ] Friendly error page for invalid/expired tokens

---

## 11. Deployment

- [ ] Create `fly.toml` for Fly.io
- [ ] Configure persistent volume for `data.db`
- [ ] Build step: `vite build` outputs to `dist/`, Express serves it in production
- [ ] Seed script runs on deploy (insert new events, skip existing)
- [ ] Set environment variables on Fly.io (`ADMIN_PASSWORD`, `JWT_SECRET`)

---

## 12. Future Work (post-MVP)

- [ ] Email/SMS notifications on RSVP (webhook to Resend/Twilio)
- [ ] Link expiration / TTL per event
- [ ] RSVP field registry (shared field catalog across events)
- [ ] Multi-admin with role-based access
- [ ] Analytics (puzzle completion rates, time-to-solve, RSVP conversion)
