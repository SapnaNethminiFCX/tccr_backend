# TCCR Backend — Implementation Blueprint

> **Source of truth.** This blueprint is derived from two documents only:
> 1. [`TCCR_System_Specification.md`](../Architecture/TCCR_System_Specification.md) — the **correct** business flow (roles, tasks, rules). This governs *what* the system does.
> 2. [`Version_02__API_Reference.md`](../APIdocument/Version_02__API_Reference.md) — referenced **only** for request/response body shapes (registration, cell creation, cell reports, enrollment, role-requests). We borrow its payload shapes and adapt them to the specification; we do **not** adopt its flow, its data design, or its service layout.
>
> Where the API reference and the specification disagree, **the specification wins.** The key rules: members can self-request **only** the `student` role; leader / g12 / master are assigned by others; the system is a **single monolith** on **PostgreSQL + Supabase**.

---

## 1. What we are building

This blueprint defines the TCCR backend as a single service built from scratch.

- **Architecture:** one **modular monolith** — a single Node.js + Express + TypeScript service. No gateway, no inter-service HTTP. Modules call each other in-process through typed service interfaces.
- **Database:** **PostgreSQL** hosted on **Supabase**, accessed through **Drizzle ORM** + drizzle-kit migrations.
- **Auth:** **Supabase Auth** (email/password + Google + Apple). The backend verifies Supabase JWTs and never stores passwords.
- **Storage:** **Supabase Storage** for images (profile photos, report photos) and PDFs (lesson attachments, qualifications).
- **Design principle:** **roles own tasks.** Code, routes, and this blueprint are organised so each role's capabilities are explicit; a shared task (e.g. "approve enrollment") is authorised for every role that owns it rather than hidden behind a merged bucket.

### Team split (three developers, three workstreams)

| Workstream | Developer | Owns (modules) | Owns (tables) |
|---|---|---|---|
| **A — Auth & Users** | Dev 1 | `auth`, `users` | `profiles`, `user_roles`, `login_attempts`, plus the `auth.users` trigger, the access-token hook, and the report-access / master fields |
| **B — Cells & Reports** | Dev 2 | `cells`, `reports` | `cell_groups`, `cell_members`, `cell_join_requests`, `cell_reports`, `report_photos` |
| **C — Bible School** | Dev 3 | `courses`, `enrollment`, `progress` | `courses`, `semesters`, `subjects`, `lessons`, `attachments`, `batches`, `batch_semesters`, `role_requests`, `enrollments`, `subject_progress`, `lesson_progress`, `video_progress` |
| **Shared kernel** | All (built first, jointly) | `shared`, `db`, `notifications`, `audit`, `jobs` | `notifications`, `audit_log`, `outbox` |

The three workstreams are deliberately low-coupling. Their only contact points are (1) the **shared kernel** and (2) a small number of **published module services** documented in §9. Each developer works on a feature branch off `develop` and never edits another workstream's module internals — only its published `index.ts` contract.

---

## 2. Project structure

```
tccr-backend/
├── src/
│   ├── app.ts                      # Express app: helmet → json → httpLogger → routes → errorHandler
│   ├── server.ts                   # binds port, graceful shutdown (SIGTERM/SIGINT)
│   ├── config.ts                   # reads & validates env once; exports typed `config` (no process.env elsewhere)
│   │
│   ├── shared/                     # ── SHARED KERNEL (built first, owned by all) ──
│   │   ├── http/
│   │   │   ├── authenticate.ts     # Supabase JWT verification → req.principal
│   │   │   ├── authorize.ts        # role union-match (super_admin⊃admin, master⊃g12…)
│   │   │   ├── ownership.ts        # mustBeOwnerOrAdmin()
│   │   │   ├── requestId.ts        # x-request-id propagation
│   │   │   └── errorHandler.ts     # final middleware; sanitises 5xx
│   │   ├── errors.ts               # AppError, createHttpError(), fromZodError()
│   │   ├── response.ts             # sendSuccess(), sendPaginated()
│   │   ├── logger.ts               # Pino + redaction
│   │   ├── principal.ts            # Principal type { uid, email, roles[], status, reportsAccess }
│   │   └── types.ts                # Role, UserStatus, shared enums
│   │
│   ├── db/                         # ── DATABASE (Drizzle) ──
│   │   ├── client.ts               # postgres-js pool (pooler conn, prepare:false)
│   │   ├── schema/                 # one file per table group, re-exported from index.ts
│   │   │   ├── auth.ts             # pgSchema('auth').table('users', …) — read-only handle
│   │   │   ├── users.ts            # profiles, user_roles, login_attempts (Workstream A)
│   │   │   ├── cells.ts            # cell_* tables (Workstream B)
│   │   │   ├── courses.ts          # course/semester/subject/lesson/batch tables (Workstream C)
│   │   │   ├── enrollment.ts       # role_requests, enrollments (Workstream C)
│   │   │   ├── progress.ts         # *_progress tables (Workstream C)
│   │   │   └── shared.ts           # notifications, audit_log, outbox
│   │   ├── migrations/             # drizzle-kit generated SQL + hand-written SQL (triggers, hooks, indexes)
│   │   └── seed.ts                 # seed roles/admin/super_admin for local dev
│   │
│   ├── modules/                    # ── FEATURE MODULES (one per domain) ──
│   │   ├── auth/                   # Workstream A
│   │   ├── users/                  # Workstream A
│   │   ├── cells/                  # Workstream B
│   │   ├── reports/                # Workstream B
│   │   ├── courses/                # Workstream C
│   │   ├── enrollment/             # Workstream C
│   │   └── progress/               # Workstream C
│   │
│   ├── notifications/              # shared kernel: in-app + email dispatch
│   ├── audit/                      # shared kernel: append-only writer + read API
│   └── jobs/                       # shared kernel: batch/semester sweeps, snapshot, outbox dispatcher
│
├── drizzle.config.ts
├── package.json
├── tsconfig.json
├── .env.example
├── Dockerfile                      # multi-stage build → slim production image (see §12)
├── .dockerignore                   # keeps node_modules, .env, .git out of the build context
└── docker-compose.yml              # local dev: app + (optional) local Postgres
```

### Inside each module (the same four layers everywhere)

```
modules/<name>/
├── index.ts          # PUBLIC CONTRACT — exports the module's Service class + its types ONLY.
│                      #   Other modules import from here, never from internal files.
├── routes.ts         # Express router: authenticate → authorize → validate → controller
├── controller.ts     # thin: parse with Zod .safeParse → call use case → sendSuccess; catch → next(err)
├── validators.ts     # Zod schemas; convert failures with fromZodError()
├── service.ts        # the module's published service (use cases callable in-process by other modules)
├── usecases/         # one file per business action; all rules live here
├── repository.ts     # Drizzle queries; the ONLY file that touches this module's tables
└── types.ts          # domain types / DTOs
```

**Dependency rule:** `routes → controller → usecases → repository`. Use cases depend on repositories and on *other modules' published services* (imported from `modules/<other>/index.ts`). A module never imports another module's `repository.ts`. This keeps the three workstreams isolated while still in one process.

---

## 3. Architectural conventions

These are non-negotiable and shared by all three workstreams (mirrors the spec's "API responses" section).

- **Response envelope.**
  - Single resource → return the object **directly**. `201` on create, `200` on read/update.
  - Action with no resource → `200 { "message": "..." }`.
  - List → `{ "items": [...], "nextCursor": <id|null>, "total": <n> }` (keyset pagination on `created_at, id`).
  - Delete → `204` (no body).
  - Error → `{ "error": { "code": "ERROR_CODE", "message": "..." }, "requestId": "..." }` (requestId at root).
- **Errors:** always `createHttpError(status, 'CODE', 'message')`; never throw raw `Error`. `errorHandler` is the last middleware and sanitises 5xx.
- **Validation:** every body/query validated with Zod `.safeParse()` in the controller; failures → `fromZodError()`.
- **Logging:** Pino only (`logger`); redact `authorization`, `password`, `token`. No `console.*`.
- **Config:** all env read in `config.ts` once; nothing else reads `process.env`.
- **IDs:** every primary key is a UUID. The user's id **is** the Supabase `auth.users.id`.
- **Time:** all timestamps `timestamptz`, stored UTC, serialised ISO-8601.

### Authorization model (shared/http/authorize.ts)

```
super_admin  ⊃ admin ⊃ (course/user/enrollment management)
super_admin / admin                → expand to all lower roles for shared tasks
master       ⊃ g12 ⊃ leader        → master inherits every G12 task, org-wide
member       = permanent base, never removed (demotion strips the elevated role only)
```

`authorize(...roles)` union-matches `req.principal.roles`. Effective expansion:
`super_admin → [super_admin, admin, g12, leader, student, member]`,
`master → [master, g12, leader, student, member]` (master is **not** admin/super_admin).

---

## 4. Authentication & database foundations (built first, by Workstream A — everyone depends on it)

These pieces are the contract the other two workstreams build on, so they are delivered in **Sprint 0** before feature work starts.

### 4.1 Supabase ↔ Postgres identity model

`auth.users` (Supabase-managed) holds the credential/identity; `public.profiles` holds the application user, linked 1:1 by UUID. A DB trigger mirrors every new auth user into a profile + base `member` role, so **all four creation paths** (self sign-up, Google, Apple, admin/g12/super-admin direct create) produce a consistent record with no duplicated code.

```sql
-- migrations/0001_auth_trigger.sql
create or replace function public.handle_new_user()
returns trigger language plpgsql security definer set search_path = public as $$
begin
  insert into public.profiles (id, email, first_name, last_name, display_name)
  values (new.id, new.email,
          new.raw_user_meta_data->>'first_name',
          new.raw_user_meta_data->>'last_name',
          trim(coalesce(new.raw_user_meta_data->>'first_name','')||' '||
               coalesce(new.raw_user_meta_data->>'last_name','')));
  insert into public.user_roles (user_id, role) values (new.id, 'member');
  return new;
end; $$;
create trigger on_auth_user_created after insert on auth.users
  for each row execute function public.handle_new_user();
```

### 4.2 Roles in the JWT (custom access-token hook)

A Supabase auth hook bakes `roles[]` into every token so `authorize()` needs no DB read. On any role change, the `users` service dual-writes: Postgres `user_roles` (source of truth) **and** `supabaseAdmin.auth.admin.updateUserById(uid, { app_metadata: { roles } })` so the next token is fresh.

```sql
-- migrations/0002_access_token_hook.sql
create or replace function public.custom_access_token_hook(event jsonb)
returns jsonb language plpgsql stable as $$
declare claims jsonb := event->'claims'; roles text[];
begin
  select array_agg(role) into roles from public.user_roles where user_id=(event->>'user_id')::uuid;
  claims := jsonb_set(claims,'{app_metadata,roles}',coalesce(to_jsonb(roles),'[]'::jsonb));
  return jsonb_set(event,'{claims}',claims);
end; $$;
```

### 4.3 `authenticate()` middleware

Verifies the Supabase JWT via JWKS, loads status, builds the principal. Distinguishes token-expired (client refreshes) from session-expired/revoked (hard logout); enforces an absolute session cap.

```ts
const JWKS = createRemoteJWKSet(new URL(`${config.supabaseUrl}/auth/v1/.well-known/jwks.json`));
export async function authenticate(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '');
  if (!token) return next(createHttpError(401, 'MISSING_TOKEN', 'Missing bearer token'));
  const { payload } = await jwtVerify(token, JWKS, { audience: 'authenticated' });
  const profile = await usersService.getStatus(payload.sub);          // published service
  if (!profile || profile.status === 'suspended')
    return next(createHttpError(403, 'ACCOUNT_DISABLED', 'Account is disabled'));
  req.principal = {
    uid: payload.sub, email: payload.email,
    roles: payload.app_metadata?.roles ?? [],
    status: profile.status, reportsAccess: profile.reportsAccess,
  };
  next();
}
```

> **Email verification:** turn ON Supabase "Confirm email" so no token is minted until verified — this enforces the spec's verification gate at the source. Google/Apple are pre-verified.

### 4.4 Two connection strings

| Use | Connection | Notes |
|---|---|---|
| `drizzle-kit migrate` | **Direct** (5432) | DDL needs a real session |
| App runtime | **Transaction pooler** (6543, pgBouncer) | `postgres-js` with `prepare: false` |

---

## 5. Workstream A — Auth & Users (Developer 1)

**Mission:** identity, sessions, the full `profiles`/`user_roles` lifecycle, and every role-management task the spec gives Admin / Super Admin / G12 (promote, demote, create-direct, suspend, master, report-access). This workstream also delivers the shared `authenticate`/`authorize` kernel.

### 5.1 Tables (`db/schema/users.ts`)

```ts
export const roleEnum   = pgEnum('role', ['member','student','leader','g12','master','admin','super_admin']);
export const userStatus = pgEnum('user_status', ['active','suspended']);

export const profiles = pgTable('profiles', {
  id: uuid('id').primaryKey().references(() => authUsers.id, { onDelete: 'cascade' }),
  email: text('email').notNull(),
  firstName: text('first_name'), lastName: text('last_name'), displayName: text('display_name'),
  phoneNumber: text('phone_number'), address: text('address'),
  dateOfBirth: date('date_of_birth'), gender: text('gender'),
  preferredLanguage: text('preferred_language').notNull().default('en'),
  profilePhotoUrl: text('profile_photo_url'),
  status: userStatus('status').notNull().default('active'),
  reportsAccess: text('reports_access').notNull().default('none'),       // 'none' | 'master'
  reportsAccessExpiresAt: timestamp('reports_access_expires_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const userRoles = pgTable('user_roles', {
  userId: uuid('user_id').notNull().references(() => profiles.id, { onDelete: 'cascade' }),
  role: roleEnum('role').notNull(),
}, t => ({
  pk: primaryKey({ columns: [t.userId, t.role] }),
  oneMaster: uniqueIndex('one_active_master').on(t.role).where(sql`role = 'master'`),  // SINGLETON
}));

export const loginAttempts = pgTable('login_attempts', {
  email: text('email').primaryKey(),
  failedCount: integer('failed_count').notNull().default(0),
  windowStartedAt: timestamp('window_started_at', { withTimezone: true }),
  lockedUntil: timestamp('locked_until', { withTimezone: true }),
});
```

### 5.2 Endpoints

| Method & path | Roles | Purpose |
|---|---|---|
| `POST /auth/register` | public | create member (Supabase signUp) — see body below |
| `POST /auth/federated/google` · `/apple` | public | exchange OAuth → session (Supabase) |
| `POST /auth/track-failure` | public | increment `login_attempts`; returns lock state |
| `POST /auth/resend-verification` · `POST /auth/verify-email` | public/unverified | verification helpers |
| `POST /auth/password-reset` | public | Supabase reset link; always 204 |
| `GET /me` · `PATCH /me` | any auth | read / edit own profile (email read-only) |
| `POST /me/avatar` | any auth | upload profile photo → Supabase Storage |
| `GET /users` · `GET /users/:uid` · `GET /users/summary` | leader, g12, admin | list / view (scoped: leader/g12 cannot see admins) |
| `POST /users` | g12, admin, super_admin | create user directly as **leader or g12** (g12/admin), temp password + email |
| `POST /users/:uid/promote` | leader, g12, admin, super_admin | member→leader/g12 (caller matrix below) |
| `POST /users/:uid/demote` | leader, g12, admin, super_admin | strip elevated role → member |
| `POST /users/:uid/suspend` · `/reactivate` | admin, super_admin | account lifecycle |
| `DELETE /users/:uid` | admin, super_admin | hard-delete (blocks admin/super_admin targets) |
| `POST /super-admin/admins` | super_admin | invite an administrator by email |
| `DELETE /super-admin/admins/:uid` | super_admin | hard-delete an admin |
| `GET /master` · `POST /master/assign` · `/invite` · `POST /master/:uid/suspend` · `/reactivate` · `DELETE /master/:uid` | super_admin | master lifecycle (singleton) |
| `POST /super-admin/g12/:uid/report-access` · `DELETE …` | super_admin | grant/revoke master-scope report access to a **G12 only** |
| `GET /audit-log` · `GET /users/:uid/audit-log` | admin, super_admin | activity logs (read; backed by `audit` kernel) |

**`POST /auth/register` — request** (shape borrowed from API ref §2.1, adapted: created via Supabase, profile via trigger):
```json
{ "firstName": "Viruli", "lastName": "Weerasinghe", "email": "viruli@example.com",
  "password": "SecurePass1@", "preferredLanguage": "si" }
```
**Response `201`:** `{ "uid": "<uuid>", "message": "Registration successful. Check your email to verify your account." }`

**Promote caller-role matrix** (enforced in `PromoteMemberUseCase`):
- g12 / admin / super_admin → may promote to `leader` or `g12`
- leader → may promote to `g12` only
- targeting an existing admin/super_admin → `403` always

**Demote caller-role matrix** (`DemoteUseCase`): super_admin/admin → student/leader/g12; g12 → leader or g12 → member; leader → g12 → member. The spec rule "**demote to member only if they have no cells under them**" is enforced by calling `cellsService.countOwnedCells(uid)` (Workstream B contract) and rejecting with `409 HAS_CELLS` if non-zero.

**Singleton master:** `AssignMasterUseCase` / `InviteMasterUseCase` do a concurrency-safe check (`SELECT … FOR UPDATE` on the master row / rely on the `one_active_master` unique index) and return `409 MASTER_ALREADY_EXISTS` if an active master already exists.

### 5.3 Implementation process (Dev 1)
1. **Sprint 0 (shared):** scaffold `shared/` kernel, `db/client`, `config`, the auth trigger, access-token hook, `authenticate`/`authorize`. Hand this to Devs 2 & 3.
2. Build `users` repository + service (`getStatus`, `getProfile`, `getRoles`, `count... ` consumers need).
3. Profile use cases (`GET/PATCH /me`, avatar).
4. Role mutation use cases with the **dual-write rule** (Postgres + Supabase `app_metadata`).
5. Promote/demote with caller matrices; wire the `cellsService.countOwnedCells` guard.
6. Direct-create (`POST /users`, `POST /super-admin/admins`) via Supabase Admin API + temp password email.
7. Master lifecycle + report-access grant (writes `profiles.reportsAccess`).
8. Lockout (`track-failure`), activity-log read endpoints.
9. Publish `usersService` contract (§9) and freeze it.

---

## 6. Workstream B — Cells & Reports (Developer 2)

**Mission:** the entire cell-group lifecycle and the executive reporting/analytics scope gate. Implements the spec's Leader, G12, and Master cell tasks plus Admin's manage-cells (transfer/bulk/delete).

### 6.1 Tables (`db/schema/cells.ts`)

```ts
export const cellType  = pgEnum('cell_type', ['g12','care','children','outreach']);
export const cellState = pgEnum('cell_state', ['active','archived']);

export const cellGroups = pgTable('cell_groups', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(), area: text('area').notNull(), type: cellType('type').notNull(),
  state: cellState('state').notNull().default('active'),
  leaderUid: uuid('leader_uid').notNull(),              // owning leader/g12 (FK profiles)
  g12SupervisorUid: uuid('g12_supervisor_uid'),         // null when owner is a G12
  createdBy: uuid('created_by').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const cellMembers = pgTable('cell_members', {
  id: uuid('id').primaryKey().defaultRandom(),
  cellId: uuid('cell_id').notNull().references(() => cellGroups.id, { onDelete: 'cascade' }),
  kind: text('kind').notNull(),                         // 'registered' | 'external'
  userUid: uuid('user_uid'),                            // set when registered
  externalName: text('external_name'), externalPhone: text('external_phone'),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, t => ({
  // one-cell-per-member for ordinary members; leaders exempt (enforced in use case via role check)
  oneCell: uniqueIndex('one_cell_per_registered_member').on(t.userUid).where(sql`kind = 'registered'`),
}));

export const cellReports = pgTable('cell_reports', {
  id: uuid('id').primaryKey().defaultRandom(),
  cellId: uuid('cell_id').notNull().references(() => cellGroups.id, { onDelete: 'cascade' }),
  filedBy: uuid('filed_by').notNull(),
  date: date('date').notNull(), didMeet: boolean('did_meet').notNull(),
  noMeetReason: text('no_meet_reason'),
  location: text('location'),                           // NULLABLE — null when no meeting
  data: jsonb('data').notNull(),                        // full report payload (attendance[], counts, etc.)
  photoUrls: text('photo_urls').array(),
  state: text('state').notNull().default('active'),     // 'active' | 'voided'
  clientReqId: text('client_req_id'),                   // idempotency
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, t => ({ idem: uniqueIndex('cell_report_idem').on(t.cellId, t.clientReqId) }));

export const cellJoinRequests = pgTable('cell_join_requests', { /* id, cellId, requesterUid, status, createdAt */ });
```

### 6.2 Endpoints

| Method & path | Roles | Purpose |
|---|---|---|
| `POST /cells` | leader, g12 | create cell (+ members & unregistered at creation) |
| `GET /cells` · `GET /cells/mine` · `GET /cells/:id` | auth (scoped) | list/own/detail — G12 sees ALL cells **view-only except own** |
| `PATCH /cells/:id` | owner (leader/g12) | edit cell **anytime** (no time limit) |
| `POST /cells/:id/members` | owner | add registered `userUids[]` + `externalMembers[]` |
| `DELETE /cells/:id/members/:uid` | owner | remove (accepts profile UUID or external UUID) |
| `POST /cells/:id/archive` · `DELETE /cells/:id` | owner, admin, super_admin | archive / hard-delete |
| `POST /cells/:id/transfer-ownership` | admin, super_admin | leader→leader, g12→g12; bulk via repeated/`cellIds[]` |
| `POST /cells/:id/join-requests` (+ approve/reject) | member / owner | join workflow |
| `POST /cells/:id/report-photos` | leader, g12 | pre-upload photos → URLs |
| `POST /cells/:id/reports` | leader, g12 | file report (idempotent; +unregistered attendees; location optional) |
| `PATCH /cells/:id/reports/:rid` | original filer | edit within **24h** only |
| `POST /cells/:id/reports/:rid/void` | leader, g12, admin, master | void a report |
| `GET /cells/network/reports` · `/members` · `/summary` | g12, master | network/executive reports; `?role=g12|master` scope gate |
| `GET /analytics/*` | g12, master | dashboards; same scope gate + date range |

**`POST /cells` — request** (API ref §13.3; supervisor required for leader, omitted for g12):
```json
{ "name": "Rathmalana West G12", "type": "g12", "area": "Rathmalana",
  "g12LeaderUid": "<g12-uuid>",
  "members": ["<member-uuid>"], "externalMembers": [{ "name": "Walk-in", "phone": "+9477..." }] }
```
**Response `201`:** the cell object — `{ id, name, type, area, leaderUid, g12SupervisorUid, members[], externalMembers[], memberCount, reportCount, state, createdAt, updatedAt }`.

**`POST /cells/:id/reports` — `multipart/form-data`** with `data` (JSON) + `photos[]`; `X-Idempotency-Key` header. `data` shape (API ref §14.2), with the spec's correction that **`location` is null when `didMeet=false`**:
```json
{ "date": "2026-05-15", "didMeet": true, "leaderPresent": true, "location": "TCCR",
  "timeStarted": "2026-05-15T18:00:00+05:30", "timeEnded": "2026-05-15T19:30:00+05:30",
  "language": "si", "subjectDiscussed": "sunday_sermon", "cellType": "g12",
  "g12LeaderUid": "<g12-uuid>",
  "attendance": [ { "userUid": "<uuid>", "name": "Sapna", "status": "present", "isNew": false },
                  { "name": "Walk-in", "status": "present", "isNew": true } ],
  "contactedAbsentees": "yes", "absenteeNotes": "…", "additionalVisitors": 1,
  "childrenCount": 2, "satisfactionRate": 4, "additionalInfo": "Great session." }
```
When `didMeet=false` only `{ date, didMeet:false, noMeetReason }` is required and `location` stays null.

### 6.3 Rules this workstream enforces (from the spec)
- **Cell editable anytime; only reports have the 24h window** (author-only edit; voided reports immutable).
- **Membership rules:** ordinary members (member / member+student) → one cell (DB partial unique index); leaders → multi-cell (exempt). **Leader adds members only; G12 adds members + leaders.**
- **Unregistered people** addable at cell creation, while editing, and as report attendees.
- **Visibility:** leader sees all cells appearance-only + edits own; **G12 sees all cells view-only except own**; supervised cells view-only.
- **Report scope gate** (`reports` module): `?role=g12` → own network; `?role=master` → org-wide, allowed only if caller is `master` **or** `profiles.reportsAccess='master'` (and not expired). Any other value → `400 VALIDATION_ERROR`.

### 6.4 Implementation process (Dev 2)
1. Wait for Sprint 0 kernel. Add `cells` schema + migration.
2. Cell CRUD + ownership/visibility guards (consume `usersService.getProfile` to enrich member rosters and `getRoles` to apply the leader-exempt membership rule).
3. Members (registered + external) with the one-cell index; join requests.
4. Report photos + report filing (idempotency, location-null rule, 24h edit, void).
5. `reports` module: `resolveScope(uid, roles, reportsAccess, filters)` → own vs org-wide; date-range aggregation; `?role=` gate.
6. Admin transfer-ownership + bulk + delete; publish `cellsService.countOwnedCells(uid)` for Workstream A's demote guard.
7. Wire cell domain events (join-requested/approved/rejected, report-filed, ownership-transferred) to the `notifications` + `audit` kernels.

---

## 7. Workstream C — Bible School (Developer 3)

**Mission:** courses and their content, batches & per-semester scheduling, the student role-request, course enrollment, and progress tracking. Implements the spec's Admin course lifecycle and the Student learning journey.

### 7.1 Tables (`db/schema/courses.ts`, `enrollment.ts`, `progress.ts`)

```ts
export const courseState = pgEnum('course_state', ['draft','published','archived']);
export const batchState  = pgEnum('batch_state', ['draft','open','closed']);

export const courses   = pgTable('courses', { id, title, description, coverImageUrl, state, semesterCount, createdAt, updatedAt });
export const semesters = pgTable('semesters', { id, courseId, title, order, subjectCount });
export const subjects  = pgTable('subjects', { id, semesterId, courseId, title, order, imageUrl });
export const lessons   = pgTable('lessons', { id, subjectId, semesterId, courseId, title, description, youtubeVideoId, order });
export const attachments = pgTable('attachments', { id, lessonId, fileUrl, mimeType, sizeBytes });
export const batches   = pgTable('batches', { id, courseId, name, scheduledOpenAt, intakeStart, intakeEnd, capacity, state });
export const batchSemesters = pgTable('batch_semesters', { id, batchId, courseId, semesterId, openDate, endDate },
  t => ({ uniq: uniqueIndex().on(t.batchId, t.semesterId) }));

export const requestStatus = pgEnum('request_status', ['pending','approved','rejected']);
export const roleRequests = pgTable('role_requests', {
  id, requesterUid, requestedRole: text('requested_role').notNull(),   // constrained to 'student'
  status: requestStatus('status').notNull().default('pending'),
  applicantProfile: jsonb('applicant_profile'),                        // snapshot at submit
  decisionNote: text('decision_note'), decidedBy: uuid('decided_by'),
  createdAt, decidedAt,
}, t => ({ onePending: uniqueIndex('one_pending_role_request').on(t.requesterUid).where(sql`status='pending'`) }));

export const enrollmentState = pgEnum('enrollment_state', ['pending','approved','rejected','withdrawn']);
export const enrollments = pgTable('enrollments', { id, studentUid, courseId, batchId, state, note, createdAt },
  t => ({ uniq: uniqueIndex().on(t.studentUid, t.courseId) }));

export const subjectProgress = pgTable('subject_progress', { /* unique(studentUid, subjectId) */ });
export const lessonProgress  = pgTable('lesson_progress',  { /* unique(studentUid, lessonId)  */ });
export const videoProgress   = pgTable('video_progress',   { /* unique(studentUid, lessonId)  */ });
```

### 7.2 Endpoints

| Method & path | Roles | Purpose |
|---|---|---|
| `GET /courses` · `GET /courses/:id` | public/auth | catalogue (drafts visible to admin only) |
| `POST/PATCH/DELETE /courses…` + `/publish` `/unpublish` `/archive` `/restore` | admin, super_admin | course lifecycle |
| `POST /courses/:id/semesters`, `/subjects`, `/lessons`, attachments | admin | content authoring |
| `GET/POST /courses/:id/batches` · `PATCH /batches/:id` · `POST /batches/:id/open` `/close` | admin, super_admin | batch lifecycle (open/close) |
| `PUT /courses/:cid/batches/:bid/semester-dates` · `PATCH …/:sid` | admin, super_admin | per-batch per-semester scheduling |
| `POST /role-requests` | member | request **student** role (body `{ "requestedRole": "student" }`) |
| `GET /role-requests` · `/:id` · `POST /:id/approve` `/reject` | admin, super_admin | review (approve grants student via `usersService.addRole`) |
| `POST /enrollments` (alias `POST /courses/:id/enroll`) | student | request enrollment |
| `GET /enrollments` · `POST /enrollments/:id/approve` `/reject` | admin, super_admin | review |
| `GET /me/enrollments` · `POST /enrollments/:id/withdraw` | student | own enrollments |
| `GET /me/courses/:courseId` | student | batch-aware course view with semester states |
| `POST /progress/lessons/:id/complete` · `/video-position` (+GET) | student | progress + resume |

**`POST /role-requests` — request** (API ref §5.1): `{ "requestedRole": "student" }` — profile snapshot pulled from `profiles`. Restricted to `requestedRole: "student"` per the corrected spec. **Approve** → calls `usersService.addRole(uid, 'student')` (dual-write) and emits `role.granted`.

**`GET /me/courses/:courseId` — response** derives each semester's `state` (`unscheduled | upcoming | open | closed`) from the student's `batch_semesters` rows; a semester with no row → `unscheduled` (spec's batch-driven view).

### 7.3 Rules this workstream enforces
- Course publish requires ≥1 semester and every semester ≥1 subject; `restore` → draft.
- Batch `isEnrollable()` = `state='open'` AND now ∈ `[intakeStart, intakeEnd]`; admin opens/closes batches.
- **Member can only request `student`**; approval grants the student role through Workstream A.
- Enrollment approval gives course access; semesters open per the **student's own batch** schedule.
- Lesson-complete is idempotent and auto-rolls up to subject completion; video position is upserted.

### 7.4 Implementation process (Dev 3)
1. Wait for Sprint 0 kernel. Add course/enrollment/progress schemas + migrations.
2. Course + semester/subject/lesson CRUD with denormalised counters; lifecycle state machine.
3. Batches + `batch_semesters` scheduling + open/close; `GET /me/courses/:id` state derivation.
4. Role-request (student-only) → approve calls `usersService.addRole`; emit events.
5. Enrollment lifecycle (request/approve/reject/withdraw) with cooloff.
6. Progress (lesson/subject/video), attachment upload (Supabase Storage) + enrollment-gated signed download.

---

## 8. Shared kernel modules (built in Sprint 0, maintained by all)

- **`notifications`** — in-app rows + transactional email. One `EmailService` is the only sender. App emails: role-request decisions, enrollment decisions, direct-create credentials, promote/demote, cell-report-filed, ownership-transfer. Auth emails (verify/reset/invite) are Supabase's. Reliability via the `outbox` table + `jobs` dispatcher (retry with backoff).
- **`audit`** — append-only `audit_log` writer + `GET /audit-log` / `GET /users/:uid/audit-log` reads. Every significant action publishes an audit entry.
- **`jobs`** — in-process runner (e.g. `pg-boss`): `batchSweep` (open/close by schedule), `semesterSweep` (daily), `snapshot`/report aggregation if cached, and the outbox dispatcher.

---

## 9. Published module contracts (the only cross-workstream coupling)

Each module's `index.ts` exports a single service object. These are the **frozen interfaces** the other workstreams may call; nothing else crosses module lines.

```ts
// modules/users/index.ts  (Workstream A → consumed by B and C)
export const usersService = {
  getStatus(uid): Promise<{ status; reportsAccess; reportsAccessExpiresAt } | null>,
  getProfile(uid): Promise<ProfileDTO | null>,
  getProfiles(uids[]): Promise<ProfileDTO[]>,        // bulk enrich (cells rosters, report names)
  getRoles(uid): Promise<Role[]>,
  addRole(uid, role): Promise<void>,                 // dual-write Postgres + Supabase; used by role-request approve
  removeRole(uid, role): Promise<void>,
};

// modules/cells/index.ts  (Workstream B → consumed by A)
export const cellsService = {
  countOwnedCells(uid): Promise<number>,             // demote-to-member guard
};

// modules/enrollment/index.ts (Workstream C) — self-contained; consumes usersService.addRole only
```

**Rule:** if a workstream needs something not on a published contract, it requests the owning developer to add it to that `index.ts` — it never reaches into another module's repository or use cases.

---

## 10. Development workflow

- **Branches:** feature branches off `develop`; PRs target `develop`; `main` is release-only. Name branches `feat/auth-…`, `feat/cells-…`, `feat/bible-…` so ownership is obvious.
- **Sprint 0 is shared and blocking:** the kernel + DB foundations (§4, §8, §9 stubs) are merged first; only then do the three feature workstreams start in parallel.
- **Migrations:** each workstream owns its schema files; run `drizzle-kit generate` then `migrate`. Hand-written SQL (trigger, hook, partial unique indexes) lives beside the generated migrations and is reviewed by Dev 1 (owner of the auth foundation).
- **Contracts first:** publish and freeze the `index.ts` service signatures (§9) before deep implementation so B and C can mock A.
- **Tests:** unit-test every use case (mock repositories + published services); integration-test against a local Supabase/Postgres. Coverage gate per the team standard.
- **Definition of done per endpoint:** route guarded by `authenticate` + correct `authorize`, body/query Zod-validated, use case enforces the spec rule, response matches §3 envelope, audit/notification emitted where the spec requires, unit test green.

---

## 11. Containerization (Docker)

The single monolith ships as **one container image**. There is no DB container in production — Postgres is Supabase-hosted (§4.4); the image holds only the Node service. The same image runs migrations and the server, so what is tested is what is deployed.

### 11.1 `Dockerfile` (multi-stage)

A three-stage build keeps the runtime image small and dependency-free of build tooling. It compiles TypeScript in a builder stage, then copies only the JS output + production `node_modules` into a slim runtime.

```dockerfile
# ---- deps: install ALL deps (incl. dev) for the build ----
FROM node:20-bookworm-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# ---- build: compile TS → dist/ ----
FROM node:20-bookworm-slim AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build          # tsc → dist/  (and copies drizzle migrations into dist/ or keeps them at /app/src/db/migrations)

# ---- prod-deps: prune to production-only deps ----
FROM node:20-bookworm-slim AS prod-deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ---- runtime: slim final image ----
FROM node:20-bookworm-slim AS runtime
ENV NODE_ENV=production
WORKDIR /app
# run as the built-in non-root `node` user
COPY --from=prod-deps --chown=node:node /app/node_modules ./node_modules
COPY --from=build     --chown=node:node /app/dist ./dist
COPY --from=build     --chown=node:node /app/src/db/migrations ./dist/db/migrations
COPY --chown=node:node package.json drizzle.config.ts ./
USER node
EXPOSE 8080
# server.ts binds config.port and handles SIGTERM/SIGINT graceful shutdown (§2)
CMD ["node", "dist/server.js"]
```

### 11.2 `.dockerignore`

Keeps the build context small and prevents secrets/local state from leaking into a layer:

```
node_modules
dist
.git
.env
.env.*
*.log
coverage
.vscode
.idea
```

> **Secrets never go in the image.** `config.ts` is the only env reader (§3); all Supabase keys and connection strings are injected at runtime (`--env-file`, compose `env_file`, or the platform's secret store), never `COPY`-ed or baked via `ENV`.

### 11.3 `docker-compose.yml` (local dev)

For local development the app container reads `.env` and talks to Supabase. A local Postgres service is provided **optional** (profile `local-db`) for offline work; by default developers point at Supabase per §4.4.

```yaml
services:
  app:
    build: .
    ports: ["8080:8080"]
    env_file: .env                 # SUPABASE_URL, keys, DATABASE_URL (pooler 6543, prepare:false)
    environment:
      NODE_ENV: development
    command: sh -c "npm run db:migrate && node dist/server.js"
    restart: unless-stopped

  # optional offline Postgres — `docker compose --profile local-db up`
  db:
    image: postgres:16
    profiles: ["local-db"]
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: tccr
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

volumes:
  pgdata:
```

### 11.4 Migrations & the two connection strings

`drizzle-kit migrate` needs the **direct** connection (5432); the app runtime uses the **transaction pooler** (6543, `prepare:false`) — see §4.4. So the container takes **two** env vars: `DIRECT_URL` (migrations) and `DATABASE_URL` (runtime).

- **One-off migrate** (CI/CD release step, preferred): `docker run --rm --env-file .env <image> npm run db:migrate` before rolling out the server.
- **Compose convenience** (shown above): the dev `command` runs `db:migrate` then starts the server. Do **not** run migrations from N replicas in production — gate them to a single release job.
- Hand-written SQL (auth trigger §4.1, access-token hook §4.2, partial unique indexes) lives in `db/migrations/` and is copied into the image, so it ships and runs identically everywhere.

### 11.5 Health & operations

- Add `GET /health` (liveness; no auth, no DB) and optionally `GET /ready` (readiness; one cheap DB ping) to `app.ts`. Platforms/orchestrators probe these; a compose `healthcheck` can curl `/health`.
- The image is stateless — images/PDFs live in Supabase Storage (§1), sessions in Supabase Auth. Scale horizontally by running more replicas; the `jobs` runner (§8, e.g. `pg-boss`) coordinates through Postgres, so background sweeps are not duplicated across replicas.
- Pin the base image (`node:20-bookworm-slim`) and run as non-root (`USER node`) for a smaller attack surface.
