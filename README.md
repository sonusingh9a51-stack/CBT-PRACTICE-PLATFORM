# CBT Practice Platform

A secure, section-aware, negatively-marked CBT (Computer Based Test) practice
platform. Teachers log in, build a test from a PDF, and generate single-use
links for students. Results land in a live dashboard automatically — no
manual entry, no localStorage tricks, no copy-paste result codes.

Built with **React + TypeScript + Vite** on the frontend and **Supabase**
(Postgres + Auth + Row Level Security + Storage) as the backend.

---

## 1. Why this version is different from a "just localStorage" app

The earlier prototype of this app stored everything (including correct
answers) inside the page/link itself, and enforced "one attempt only" using
the student's own browser storage — both bypassable by anyone who opened dev
tools. This version fixes both:

- **Correct answers never reach the browser.** The student's client only
  ever receives `{id, prompt, options}` — never `correct_index`. Scoring
  happens inside a Postgres function (`submit_attempt`), server-side.
- **One-time links are enforced by the database**, not by a flag in
  localStorage. `start_attempt` locks the invite token inside a single
  transaction (`SELECT ... FOR UPDATE`), so even two simultaneous requests
  with the same link can't both succeed.
- **Teachers only ever see their own data.** Row Level Security policies
  scope every table to `owner_id = auth.uid()`. Students have **no**
  direct table access at all — the only way in is through two narrowly-
  scoped functions.

**What's still a soft limit, honestly:** the fullscreen/tab-switch
enforcement in the browser deters casual cheating (leaving the tab, opening
another app) but can't stop a genuinely determined student with a second
device. That's true of any browser-based exam tool without dedicated
proctoring hardware/software, and worth setting expectations around before
you present this.

---

## 2. Architecture

```
Teacher (authenticated, Supabase Auth)
  → PDF upload, parsed client-side (pdf.js) into questions
  → INSERT into tests / questions (RLS: owner_id = auth.uid())
  → generates invite links (INSERT into invites)

Student (no account, opens /take/:token)
  → start_attempt(token, name)   [SECURITY DEFINER, bypasses RLS internally]
      - locks + validates the invite row
      - creates an attempts row (status='running')
      - returns questions WITHOUT correct_index
  → answers stored client-side during the attempt (autosaved to
    localStorage so a refresh doesn't lose progress — the token is already
    marked used, so a refresh must resume, not restart)
  → submit_attempt(attemptId, answers, exitReason)
      - scores server-side, section-by-section
      - marks the attempt 'submitted' (idempotent on retry)

Teacher dashboard
  → SELECT from attempts (RLS: only rows for tests you own)
  → results appear as students submit — no manual step
```

## 3. Database schema

| Table | Purpose |
|---|---|
| `profiles` | One row per teacher, auto-created by a trigger on signup |
| `tests` | Title, duration, marking scheme (`marks_correct`/`marks_incorrect`), `sections` (JSON array of `{name, c}` contiguous question-count blocks) |
| `questions` | `prompt`, `options` (4-item JSON array), `correct_index` — never selected by students directly |
| `invites` | A `token` that admits up to `max_uses` attempts (default 1 = single-use); `use_count` increments atomically in `start_attempt`, checked with a row lock so it's exact even under concurrent requests |
| `attempts` | One row per completed/in-progress attempt; scoring fields filled in by `submit_attempt` |

Full schema, RLS policies, and both RPC functions are in
[`supabase/migrations/`](./supabase/migrations), heavily commented — read
them, don't just run them blind, since you're presenting this.

**These migrations were tested locally** against a real Postgres instance
before being written here (see [`test/`](./test) — not part of the shipped
app, but worth keeping as evidence of correctness if asked). Verified:
question data reaching the client never includes answers, a reused token is
rejected, scoring math matches hand-calculation, resubmission is idempotent,
and anonymous requests get zero rows back from direct table access. The
master-link capacity check was additionally stress-tested with 40 genuinely
concurrent database connections hitting one capacity-23 link at the same
instant — exactly 23 were admitted, exactly 17 rejected, every time.

---

## 4. Setup

### 4.1 Run the migrations on your Supabase project

Open your project's **SQL Editor** (Supabase dashboard → SQL Editor) and run
these five files **in order**, pasting each one in full:

1. `supabase/migrations/0001_init_schema.sql`
2. `supabase/migrations/0002_rls_policies.sql`
3. `supabase/migrations/0003_functions.sql`
4. `supabase/migrations/0004_storage.sql`
5. `supabase/migrations/0005_master_links.sql`

(If you prefer the CLI: `supabase link --project-ref cfwvcliqzysonfcuxady`
then `supabase db push` — but the SQL Editor is simpler if you're not
already set up with the CLI and a database password.)

### 4.2 Confirm auth settings

Dashboard → **Authentication → Providers** → Email should be enabled by
default. If you don't want email confirmation required for teacher signups
(fine for a demo/college project), turn off "Confirm email" under
Authentication → Settings.

### 4.3 Configure the frontend

```bash
cp .env.example .env
```

Edit `.env`:

```
VITE_SUPABASE_URL=https://cfwvcliqzysonfcuxady.supabase.co
VITE_SUPABASE_PUBLISHABLE_KEY=sb_publishable__Pb8YQSiMrvcD22lGUcr0w_y2rPJYi6
```

Double-check the URL against **Settings → API** in your dashboard — the one
above is derived from the project ref you gave me, confirm it matches
exactly. The publishable key is safe to commit to a public repo *in principle*
(it's designed for client-side use), but `.env` is still gitignored here —
keep secrets and config separate as a habit, and **never** put a
`service_role` key or database password anywhere in this project.

### 4.4 Install and run

```bash
npm install
npm run dev        # http://localhost:5173
```

```bash
npm run build       # production build to dist/
npm run preview      # serve the production build locally
```

---

## 5. Using it

1. Go to `/login`, sign up as a teacher.
2. `+ New test` → upload a PDF formatted like this (required, one field per line):
   ```
   Q1. What is the capital of France?
   A) Paris
   B) Rome
   C) Berlin
   D) Madrid
   ANSWER: A
   ```
3. Parse → review the preview → optionally split into sections (e.g.
   Physics/Chemistry/Biology as contiguous blocks) → set duration and
   marking scheme (defaults to NEET: +4/−1) → Create test.
4. On the test's page, choose **Master link** to generate one link that
   admits up to a class-size number of students (e.g. 30) — each on their
   own device gets their own attempt, and the link stops admitting anyone
   once full, enforced by the database even if the whole class opens it in
   the same second. Or choose **Individual links** to generate a separate
   single-use link per student, same as before.
5. Results land in the same page's **Results** table automatically as
   students finish — no code-pasting, no manual entry.

---

## 6. Deployment

Any static host works since this is a Vite SPA talking directly to
Supabase — no separate backend server to deploy.

**Vercel / Netlify (recommended):**
1. Push this repo to GitHub (see below).
2. Import it in Vercel or Netlify.
3. Set the same two env vars (`VITE_SUPABASE_URL`,
   `VITE_SUPABASE_PUBLISHABLE_KEY`) in the host's project settings.
4. Build command `npm run build`, output directory `dist`.
5. Add a rewrite rule so client-side routes (`/take/:token` etc.) don't
   404 on refresh:
   - Vercel: add a `vercel.json` with
     `{"rewrites":[{"source":"/(.*)","destination":"/index.html"}]}`
   - Netlify: add `public/_redirects` with `/*  /index.html  200`

## 7. Push to GitHub

This folder is already a git repository with the initial commit made. From
here:

```bash
git remote add origin https://github.com/<your-username>/<repo-name>.git
git branch -M main
git push -u origin main
```

`.env` is gitignored, so your local config won't be pushed — good, it
shouldn't be. Anyone cloning the repo follows section 4 above with their own
`.env`.

---

## 8. Project structure

```
supabase/migrations/     4 SQL files — schema, RLS, functions, storage
src/
  lib/
    supabaseClient.ts     Supabase client, reads config from env vars only
    types.ts               Shared TS types matching the DB schema + RPC shapes
    pdfParser.ts            PDF → line-delimited MCQ parser (pdf.js)
  hooks/
    useAuth.ts              Current Supabase session
    useCountdown.ts         Drift-safe exam timer
    useExamGuard.ts         Fullscreen + tab-switch enforcement, back-button trap
  components/
    ProtectedRoute.tsx, Timer.tsx, QuestionCard.tsx, QuestionPalette.tsx
  pages/
    LoginPage, DashboardPage, NewTestPage, TestDetailPage   (teacher, protected)
    TakeTestPage                                             (student, public)
  App.tsx, main.tsx, styles.css
```

## 9. Known limitations to mention if asked

- No password reset flow wired up yet (Supabase Auth supports it — would be
  a `resetPasswordForEmail` call plus a reset page, not currently built).
- No CSV export of results (straightforward to add: query `attempts`,
  convert to CSV client-side).
- Question banks are capped by practical PDF-parsing reliability, not by
  the database — Postgres handles thousands of rows easily; the PDF
  format needs to match the required line-by-line pattern.
- Fullscreen/tab-switch enforcement is a deterrent, not proctoring-grade
  security — see section 1.
