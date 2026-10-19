# SmartSchool — Setup & Architecture

A school management portal with four role-based dashboards (Administrator,
Teacher, Parent, Student), built with React + TypeScript + Tailwind, backed by
Supabase (Postgres, Auth, Row Level Security).

---

## 1. Two modes — and which one you are in right now

The app detects its backend at startup. **No configuration is required to
explore it.**

| | Demo mode *(current default)* | Live mode |
|---|---|---|
| Triggered by | No env vars set | `VITE_SUPABASE_URL` + `VITE_SUPABASE_ANON_KEY` set |
| Data | Bundled demo dataset in `localStorage` | Your real Supabase tables |
| Writes | Local only | Postgres, then re-read |
| Auth | Local credential check against demo users | Supabase Auth |
| Indicator | Login page badge reads **“Demo Mode — Local Data”** | Badge reads **“Connected to Supabase”** |

**Being explicit about this:** with no credentials, the portal is *not* talking
to a database. It is a fully interactive UI running on seeded local data so you
can inspect every screen and interaction. Nothing pretends otherwise — the
login page says so, and password reset reports that it needs a live connection
instead of faking success.

The Supabase layer is complete and written (`src/lib/supabaseService.ts`,
`src/lib/dataSource.ts`); the SQL is complete and validated
(`supabase/schema.sql`). Follow section 2 to switch on.

---

## 2. Connect Supabase (≈10 minutes)

### 2.1 Create the project
1. https://supabase.com/dashboard → **New project**.
2. Note the **Project URL** and **anon public** key from
   *Project Settings → API*.

### 2.2 Create the schema
3. **SQL Editor** → paste the entire contents of
   [`supabase/schema.sql`](supabase/schema.sql) → **Run**.

   This creates 16 tables, all constraints and indexes, 20 functions,
   41 RLS policies, and the trigger set. It is idempotent — safe to re-run.

4. *(Optional, for demo data)* paste [`supabase/seed.sql`](supabase/seed.sql) →
   **Run**. Creates the demo accounts, 8 students, 6 classes, results,
   attendance, announcements and a timetable. Skip this for production.

### 2.3 Configure auth
5. *Authentication → Providers → Email* → **enabled**.
   - **Confirm email**: ON for production, OFF for a quick local test.
6. *Authentication → URL Configuration*
   - **Site URL**: `http://localhost:5173` (your deployed origin in production)
   - **Redirect URLs**: add `http://localhost:5173/**`

### 2.4 Wire the frontend
```bash
cp .env.example .env
# fill in the two values
npm install
npm run dev
```
The login page badge should now read **“Connected to Supabase”**.

---

## 3. Demo accounts

| Role | Email | Password |
|---|---|---|
| Administrator | `admin@smartschool.edu` | `Admin123!` |
| Teacher | `teacher@smartschool.edu` | `Teacher123!` |
| Teacher | `emily.carter@smartschool.edu` | `Teacher123!` |
| Parent | `parent@smartschool.edu` | `Parent123!` |
| Parent | `david.okafor@gmail.com` | `Parent123!` |
| Student | `student@smartschool.edu` | `Student123!` |

Change these immediately in production. In demo mode the login page has
one-tap buttons that fill each credential set.

---

## 4. How role security actually works

Frontend routing is *not* the security boundary — it is convenience. Four
independent layers enforce access:

**a) The role is not client-controlled.**
`profiles.role` is written by the `handle_new_user()` trigger, which
**always** assigns `'student'`. Public signup cannot request a role; the
parameter does not exist.

**b) Self-escalation is blocked by trigger.**
```sql
create trigger trg_block_role_change before update on public.profiles
  -- raises SQLSTATE 42501 if a user updates their own row's `role`
```
Even a user calling the API directly with their own valid JWT gets
`Privilege escalation blocked`.

**c) RLS filters every row, per role.**
41 policies across all 16 tables. Examples:

- A **parent** reading `students` receives only rows where
  `is_my_child(profile_id)` is true — their own linked children.
- A **teacher** reading `results` receives only students enrolled in classes
  where `teacher_manages_class()` is true for *them*.
- A **student** reading `attendance` receives only their own rows.
- `activity_logs` has **no** insert/update/delete policy at all, so audit
  entries cannot be forged or erased by any client.

**d) Privileged actions go through `SECURITY DEFINER` RPCs.**
Creating teacher/parent/admin accounts, linking families, enrolling students
and assigning teachers all call `admin_*` functions that begin with
`if not public.is_admin() then raise exception ... using errcode='42501'`.

Two more guards live in triggers:

- `guard_teacher_authorisation` rejects a result or attendance write for a
  class the teacher does not manage, returning a clear error.
- `enforce_class_capacity` rejects enrolment past the class capacity.

And at the constraint level:

- `results_unique_record (student_id, subject_id, session_id, term)` — a
  duplicate result is rejected by Postgres itself.
- `attendance_unique_record (student_id, record_date)` — a student cannot be
  marked twice on the same day; the RPC upserts instead.

---

## 5. What the frontend does with each mode

`src/lib/dataSource.ts` is the single switch:

- **Reads** stay synchronous against an in-memory cache so the UI stays fast.
  In live mode that cache is *replaced* by real rows on boot, and re-pulled
  after every write. Because RLS scopes the queries, the cache can never hold
  a row the signed-in user was not permitted to read.
- **Writes** go to Supabase **first**. If Postgres rejects the operation, the
  error propagates and the cache is left untouched — so the UI can never show
  a success state for something the database refused.
- `subscribeTables()` wires Supabase Realtime so, for example, a teacher's
  uploaded result appears on the parent's screen without a refresh.

---

## 6. Scripts

```bash
npm run dev        # dev server on :5173
npm run build      # typecheck (tsc -b) + production bundle
npm run preview    # serve the production build

node validate_sql.cjs supabase/schema.sql   # parse schema + all function bodies
node check_policies.cjs supabase/schema.sql # RLS coverage + drop/create pairing
```

The two SQL checkers use `libpg-query`, the real PostgreSQL parser compiled to
WASM. `validate_sql.cjs` parses the file *and* re-parses each of the 20
function bodies with its true signature. `check_policies.cjs` asserts every
`create policy` has a matching `drop policy` (so the file stays re-runnable)
and that no RLS-enabled table was left without a policy — which would
silently block all access.

> Note: these validate syntax and policy wiring. They cannot check semantics
> that require a running database, such as whether a policy's subquery returns
> what you intend. Test the policies in the Supabase SQL Editor before trusting
> them in production.

---

## 7. Known limits of this first version

Stated plainly so nothing is mistaken for finished work:

1. **Live mode has not been exercised against a real Supabase project** — no
   project credentials were available here. The SQL parses under PostgreSQL
   18's parser and the TypeScript typechecks, but the queries have not been
   run end-to-end. Expect small fixes on first connection (column names,
   relation embeds).
2. **Admin screens still write through the demo store** in demo mode. The
   write-through paths exist in `dataSource.writes`, but not every admin page
   calls them yet — students, classes, results, attendance and announcements
   are wired; teacher creation and class reassignment are not.
3. **No file upload, payments, messaging or reporting** — deliberately out of
   scope for this build.
4. **The bundle is ~597 kB un-split.** Fine for a first version; route-level
   code splitting is the obvious next step.
5. **`terms` and `activity_logs` tables exist** but have no UI yet.

---

## 8. Layout

```
smartschool/
├── supabase/
│   ├── schema.sql          16 tables · RLS · triggers · RPCs  (run first)
│   └── seed.sql            optional demo dataset              (run second)
├── src/
│   ├── lib/
│   │   ├── supabase.ts            client + shared types
│   │   ├── store.ts               demo dataset + localStorage cache
│   │   ├── supabaseService.ts     real auth / queries / mutations
│   │   └── dataSource.ts          mode switch + write-through
│   ├── context/AuthContext.tsx    session, role, mode
│   ├── App.tsx                    landing, login, 4 dashboards, routing
│   └── index.css                  Tailwind + design tokens
├── validate_sql.cjs        PostgreSQL parser check
├── check_policies.cjs      RLS coverage check
└── .env.example
```

Design tokens: primary `#1E40AF`, accent `#38BDF8`, background `#F8FAFC`,
white surfaces, Inter body / Plus Jakarta Sans headings.
