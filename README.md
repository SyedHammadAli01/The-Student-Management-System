# SchoolERP

Multi-tenant School Management System / School ERP — a commercial SaaS product.

This repository is the monorepo foundation for the platform. It is currently at **Phase 9
(Communications & Notifications — announcements/notices with per-viewer read state; database
integration still deferred)**. The student-records domain (admissions → approval → enrollment), the
academic backbone (academic years/terms, classes, sections, subjects, class–subject assignments,
placements, promotions, teacher-assignment structure), the secondary academic modules (attendance,
timetables, homework, exams, results), and the student & parent portal surfaces are implemented;
fees, finance, library, transport, HR/payroll, and real-time notification delivery are **planned**.

---

## Overview

SchoolERP is a configurable, multi-tenant school management / ERP SaaS. A single school organization
(tenant) manages students, guardians, staff, academics, attendance, exams, fees, and more, across one
or more branches/campuses. Strict tenant isolation is enforced at the backend data layer.

> **Status:** Backend core + authentication/authorization + multi-tenant system administration
> (tenants, schools, branches, users, roles, permissions, audit) + the Phase 5 student-records
> domain (admissions, students, guardians, enrollments, document metadata) + the Phase 6 core
> academics structure (academic years, terms, classes, sections, subjects, class–subject
> assignments, academic placements, academic promotions, teacher-assignment structure) + the
> Phase 7 student & parent portals (read-only, own-record dashboard/profile/academic/enrollment
> views) + the Phase 8 secondary academic modules (attendance, timetables, homework, exams,
> results) + the Phase 9 communications module (announcements/notices with per-viewer read
> state) are implemented (database-deferred). Fees and other business modules are **planned**
> and arrive in later phases. Nothing below implies those features currently exist.

---

## Architecture

See the authoritative architecture documents in [`docs/`](docs/):

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — system architecture and request lifecycles
- [`docs/DATABASE_ARCHITECTURE.md`](docs/DATABASE_ARCHITECTURE.md) — MongoDB data design
- [`docs/API_ARCHITECTURE.md`](docs/API_ARCHITECTURE.md) — REST API standards
- [`docs/SECURITY_ARCHITECTURE.md`](docs/SECURITY_ARCHITECTURE.md) — security architecture
- [`docs/DEVELOPMENT_STANDARDS.md`](docs/DEVELOPMENT_STANDARDS.md) — coding standards
- [`docs/DECISIONS.md`](docs/DECISIONS.md) — architecture decision records + implementation roadmap

### High-level topology

```
Next.js Frontend (frontend)
      │
      │ REST API (/api/v1)
      ▼
Express Backend (backend)
      │
      ▼
Services → Repositories → Mongoose → MongoDB
```

The frontend is a **Next.js App Router** app. The main business API is a **separate Express
application** (not Next.js API routes).

---

## Technology Stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js (App Router), React, TypeScript |
| Styling | CSS Modules + Global CSS |
| Frontend data | TanStack Query (server state), React Context (client state) |
| Forms / validation | React Hook Form, Zod |
| Backend | Node.js, Express.js, TypeScript |
| Database | MongoDB |
| ODM | Mongoose |
| API | REST under `/api/v1`, OpenAPI |
| Background jobs | Redis + BullMQ (extension point, later) |
| Logging | Pino |
| Testing | Vitest, Supertest, Playwright |
| Tooling | npm workspaces |

---

## Monorepo Structure

```
/
├── apps/
│   ├── web/       # Next.js frontend
│   └── api/       # Express backend
├── packages/
│   ├── shared/    # shared utilities + Zod schemas
│   ├── types/     # shared TypeScript types / contracts
│   └── config/    # shared tsconfig/eslint presets
├── docs/          # architecture documentation
├── infrastructure/ # docker, docker-compose, deployment (Docker-ready)
├── scripts/       # dev/ops helper scripts
├── .env.example
├── .gitignore
└── package.json   # npm workspaces root
```

---

## Prerequisites

- Node.js **>= 20**
- npm **>= 10**
- MongoDB — **optional at this stage.** The MongoDB integration is intentionally deferred to the
  end of the project; the backend runs and stays healthy (`mongo: not_configured`) without it.

---

## Installation

```bash
npm install
```

Copy the environment template and set real values (do not commit real secrets):

```bash
cp .env.example .env
```

---

## Environment Setup

Set the following in `.env` (see `.env.example` for the full list):

- `NODE_ENV`
- `PORT` (API port, default 4000)
- `API_URL` / `WEB_URL`
- `CORS_ORIGINS` (comma-separated allowlist; defaults to `WEB_URL`)
- `LOG_LEVEL` (optional; defaults to `info`)
- `MONGODB_URI` (**optional** — database work is deferred; leave unset to run without MongoDB)
- `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` (dev placeholders; **required and must be real,
  unique 32+ char secrets in production** — production boot fails otherwise)
- `JWT_ALGORITHM` / `JWT_ACCESS_TTL` / `JWT_REFRESH_TTL` / `JWT_ISSUER` / `JWT_AUDIENCE`
- `AUTH_LOGIN_*` / `AUTH_REFRESH_*` / `AUTH_LOGOUT_*` (auth endpoint rate limits)
- `REDIS_URL`, `STORAGE_*` (used in later phases)

---

## Development Commands

Run the full stack (both web and api in watch mode):

```bash
npm run dev
```

Run each app independently:

```bash
npm run dev:web   # Next.js frontend on http://localhost:3000
npm run dev:api   # Express backend on http://localhost:4000
```

---

## Build

```bash
npm run build          # build all workspaces
npm run build:web      # build frontend only
npm run build:api      # build backend only
```

---

## Lint / Typecheck / Test

```bash
npm run lint           # lint all workspaces
npm run typecheck      # typecheck all workspaces
npm run test           # backend API tests (Vitest + Supertest)
npm run test:e2e       # frontend E2E tests (Playwright)
```

Testing setup:

- **Backend (Vitest + Supertest)** in `backend/test/` — drives the Express app
  directly (no running server required). The database module is mocked so the
  health/404 foundation tests are deterministic without MongoDB. Coverage:
  application startup, health endpoints, 404 handling, validation (body/query/
  params), error handling, request ID, API response format, security middleware
  (Helmet, CORS, rate limiting), and the full auth suite (Argon2id password
  hashing, JWT sign/verify/tamper/expiry, login/refresh rotation/reuse-detection/
  logout/me flow, CSRF origin guard, auth rate limiting, permission/role
  enforcement, tenant-scope enforcement).
- **Phase 5 registry suite** in `backend/test/registry/` (harness + 5 suites,
  41 tests) covering the admissions pipeline transitions, approval→enrollment
  conversion, cross-tenant IDOR, branch-scope enforcement (lists and single
  reads), the single-ACTIVE-enrollment rule, soft-archive lifecycle, guardian
  link/primary semantics, service-level permission assertion on enroll, and
  strict-schema rejection.
- **Phase 6 academics suite** in `backend/test/academics/` (harness + 71
  tests) covering school-scope enforcement on create/list/single-id operations,
  cross-school IDOR (404), DRAFT→ACTIVE→CLOSED year/term lifecycles, class/
  section/subject soft-archive and restore, class–subject assignment rules,
  placement invariants (single ACTIVE placement per student) and transfer,
  promotion execution (PROMOTE/RETAIN/TRANSFER), service-level 422 business
  rules (`sectionId` requires `classId`, `termId` requires `academicYearId`),
  teacher-assignment reference validation, permission 403s and strict-schema
  rejection.
- **Phase 7 portal suite** in `backend/test/portal/` (harness + 27 tests)
  covering the student portal (own-record dashboard/profile/academic/enrollment
  with full view shaping), the parent portal (guardian profile + linked
  students, branch-pruned), and portal security (unauthenticated 401, foreign-
  role 403, unbound-subject 403, subjectType-mismatch 403, forged
  tenantId/branchId/guardianId ignored, tenant isolation, wrong-branch 404,
  unlinked-student 404 for parents, unmatched `/api/v1/*` 404).
- **Frontend Vitest suite** in `frontend` (auth/component tests, e.g.
  `components/academics/academic-years-client.test.tsx`) covering rendering,
  empty/error states, form validation and confirm-guarded actions.
- **Frontend/E2E (Playwright)** in `frontend/e2e/` — requires the production
  build (`npm run build:frontend` or `npm run build:web`) and a Chromium browser:
  ```bash
  npx playwright install chromium
  npm run test:e2e
  ```
- Business-module tests (admission, fees, attendance, etc.) are added in later
  phases alongside those features.

---

## Health Endpoints

- `GET /health` — liveness (process is up; never depends on MongoDB)
- `GET /api/v1/health` — API health (reports uptime/memory and database state:
  `connected`, `disconnected` → 503 degraded, or `not_configured` while the
  database integration is deferred)

## Authentication & Authorization (Phase 3)

Auth is implemented database-free behind repository ports (in-memory adapters for
dev/test; Mongoose adapters arrive with the database phase).

- `POST /api/v1/auth/login` — email + password → access token (body) + refresh
  token (HTTP-only `SameSite=Lax` cookie on `/api/v1/auth`). Generic 401s; no
  account enumeration.
- `POST /api/v1/auth/refresh` — rotates the refresh token (reuse of a rotated
  token revokes the whole session family).
- `POST /api/v1/auth/logout` — revokes the refresh session and clears the cookie.
- `GET /api/v1/auth/me` — current user (requires `Authorization: Bearer <accessToken>`).

Notes:

- Passwords: **Argon2id** (OWASP parameters), never stored in plaintext; hashes
  never leave the server in responses.
- Access tokens: short-lived HS256 JWTs with minimal claims, bound to a
  server-side session. Expire in `JWT_ACCESS_TTL` (default 15m).
- **Dev bootstrap** (non-production only): `admin@schoolerp.local` /
  `Admin@123456` (Super Admin role) is seeded into the in-memory store.
- Password reset and email verification are documented but **not implemented**
  (future work). Permission/role catalogs are code-level; the RBAC administration
  endpoints land in Phase 4 (below).

## Multi-tenant System Administration (Phase 4)

Phase 4 adds database-free system administration under `/api/v1`, reusing the same
repository-port pattern. Every tenant-scoped endpoint resolves against the caller's
tenant — cross-tenant ids return 404, never data.

- **Tenants** (`GET/POST /api/v1/tenants`, `GET/PATCH /tenants/:id`,
  `PATCH /tenants/:id/status`) — platform-scoped, super admin only.
- **Schools** / **Branches** — tenant-scoped lifecycle + status toggle; a branch
  must attach to an owned, ACTIVE school.
- **Users** — tenant-scoped CRUD, status, and role assignment with a
  **privileged-account guard**: no self status/role changes, no grants at or
  above the caller's privilege level, no SUPER_ADMIN escalation.
- **Roles** — tenant custom roles (globally unique keys) and replaceable
  permission sets; system roles are immutable and double as the seed catalog.
  Custom roles resolve at request time.
- **Permissions** — read-only platform catalog (`GET /api/v1/permissions`).
- **Audit** — every mutating action emits a structured audit event (actor,
  entity, tenant) via the audit repository port.
- Status hierarchy rule: an INACTIVE tenant rejects tenant-scoped writes (422);
  INACTIVE schools read-lock their branches.

Additional dev bootstrap (non-production): `demo-admin@schoolerp.local` /
`Demo@123456` (TENANT_ADMIN) with a `demo` tenant, `DEMO` school and `MAIN` branch
for exercising the admin flow locally.

## Students, Guardians, Admissions & Enrollments (Phase 5)

Phase 5 adds the first business module under `/api/v1`, still database-free behind the same
repository-port pattern. Applications are branch-scoped; students/guardians are tenant-scoped;
everything resolves against the caller's tenant and branch scope (cross-tenant ids → 404).

- **Admissions** (`GET/POST /api/v1/admissions`, `GET/PATCH /admissions/:id`,
  `POST /admissions/:id/{submit|review|interview|approve|reject|waitlist}`,
  `POST /admissions/:id/enroll`) — an approved-document pipeline:
  `DRAFT → SUBMITTED → UNDER_REVIEW → INTERVIEWED → APPROVED` (or REJECTED / WAITLISTED),
  with a service-enforced transition matrix (anything else → 422). Application numbers
  (`APP-####`) are sequential per tenant+branch.
- **Enroll conversion** — only an APPROVED application can be converted (one-way,
  idempotency-guarded). The enroll service asserts `admissions.approve` +
  `students.create` + `guardians.create` + `enrollments.create` together.
- **Students** (`GET/POST /api/v1/students`, `GET/PATCH/DELETE /students/:id`,
  `PATCH /students/:id/status`, `PATCH /students/:id/restore`,
  `GET /students/:id/profile`, guardians + documents sub-resources) — tenant-scoped with
  sequential `STU-####` numbers. `DELETE` is a **soft-archive** (never destructive). The
  composed profile returns the student + linked guardians + enrollments (with branch/school)
  + source admission. `nationalIdNumber` is PII: persisted and returned on direct reads but
  omitted from lists.
- **Guardians** (`GET/POST /api/v1/guardians`, `GET/PATCH /guardians/:guardianId`) —
  tenant-scoped records shared across students through `student_guardians` links (relation +
  at most one primary per student).
- **Enrollments** (`GET/POST /api/v1/enrollments`, `GET/PATCH /enrollments/:id`) —
  branch-scoped; `schoolId` is always derived from the branch; a student may hold at most
  ONE ACTIVE enrollment (409).
- **Documents** — metadata only (type, file name, storage key, status, uploader) under
  `/admissions/:id/documents` and `/students/:id/documents`; actual file storage is deferred.
- **Branch restriction** — a principal whose tenant is pinned to a branch only sees
  applications at that branch and students enrolled at that branch (enforced on lists AND on
  every single-student read/mutation; 404 outside the branch, never leaked).
- Every mutating action emits a structured audit event (actor, entity, tenant).

Additional dev bootstrap (non-production): the `demo` tenant is seeded with a converted
example — admission `APP-0001` (APPROVED), student `STU-0001`, guardian, ACTIVE enrollment
and a birth-certificate document record, so the local UI has realistic data.

## Core Academics Structure (Phase 6)

Phase 6 adds the academic backbone under `/api/v1`, still database-free behind the same
repository-port pattern. Every resource is school-scoped: create/list requests REQUIRE an
explicit `schoolId` for unrestricted principals (403 if absent; 404 if foreign or non-existent),
and every id-based operation derives the school from the fetched record — a caller can never
observe or mutate across schools.

- **Academic years** (`GET/POST /api/v1/academic-years`, `GET/PATCH /academic-years/:id`,
  `POST /academic-years/:id/{activate|close}`) — the school calendar, `DRAFT → ACTIVE → CLOSED`;
  only one ACTIVE year is `current`.
- **Terms** (`GET/POST /api/v1/terms`, `GET/PATCH /terms/:id`,
  `POST /terms/:id/{activate|close}`) — term structure inside an academic year, same lifecycle.
- **Classes / Sections / Subjects** (`GET/POST /api/v1/classes`, `/sections`, `/subjects`;
  `GET/PATCH/DELETE /:id`, `PATCH /:id/restore`) — the catalog; `DELETE` is a soft-archive with
  restore, never destructive.
- **Class–subject assignments** (`GET/POST /api/v1/classes/:id/subjects`,
  `GET/PATCH/DELETE /class-subjects/:id`) — which subject each class studies, optionally scoped
  to an academic year/term (`termId` requires `academicYearId` → 422).
- **Academic placements** (`GET/POST /api/v1/academic-placements`, `GET/PATCH /:id`,
  `POST /:id/transfer`) — a student's class/section placement for a given academic year, with a
  single-ACTIVE (`isCurrent`) placement invariant and one-way transfer.
- **Academic promotions** (`GET/POST /api/v1/academic-promotions`, `GET /:id`) — executes
  PROMOTE / RETAIN / TRANSFER from a placement in one flow, recording an immutable promotion
  result with the new placement.
- **Teacher assignments** (`GET/POST /api/v1/teacher-assignments`, `GET/PATCH/DELETE /:id`) —
  reference-only structure: `teacherUserId` must exist in the tenant, but the full teacher/staff
  module itself is Phase 9.
- Business rules (`sectionId` requires `classId`, `termId` requires `academicYearId`) are
  service-level 422s, not boundary/envelope errors. Every mutating action emits a structured
  audit event.
- The academics router mounts one sub-router per resource, each applying auth internally, so an
  unmatched `/api/v1/*` request still falls through to the 404 handler (health suite verifies
  this).

Additional dev bootstrap (non-production): the `demo` tenant is seeded with a 2026–27 academic
year (ACTIVE/current), terms T1–T3, classes KG/GRD-1/GRD-2, four sections, four subjects, six
class–subject assignments, a placement for `STU-0001` in Grade 1 Alpha, one RETAIN promotion,
and a `demo-teacher` user with two teacher assignments.

The frontend adds nine client pages (`/academic-years`, `/terms`, `/classes`, `/sections`,
`/subjects`, `/class-subjects`, `/academic-placements`, `/academic-promotions`,
`/teacher-assignments`) with list tables (loading/empty/error), React Hook Form + Zod create
forms, and confirm-guarded lifecycle actions, wired through `lib/hooks/use-academics.ts`.

## Student & Parent Portals (Phase 7)

Phase 7 adds read-only, own-record portal surfaces under `/api/v1`, still database-free. The
subject is resolved from a **server-side portal identity binding** (`userId → tenant → subjectId`
+ `subjectType`), never from client params — there are no `tenantId`/`branchId`/`guardianId`/`id`
inputs to forge, and every identity parameter on parent routes is the `studentId` that the parent
must actually be linked to.

- **Student portal** — `GET /api/v1/portal/student/{dashboard|profile|academic|enrollment}`:
  the bound student's own identity, school/branch, current academic year/class/section/placement,
  enrollment, guardian summaries/details. Requires the STUDENT role (`portal.student.read`).
- **Parent portal** — `GET /api/v1/portal/parent/{dashboard|profile|students}` and
  `GET /api/v1/portal/parent/students/:studentId{,/academic,/enrollment}`: the guardian's own
  profile plus ONLY the students linked to them via the Phase 5 `student_guardians` association
  (branch-pruned); per-student routes return 404 for any unlinked student. Requires the GUARDIAN
  role (`portal.parent.read` + `portal.parent.students.read` for `/students`).
- **Enforcement** — tenant context from the principal only; binding must exist and match the
  portal type (403); subjects must exist and not be archived (404); branch-restricted principals
  only see students with an ACTIVE enrollment in their branch (404); "not yours / doesn't exist"
  always collapse to the same 404 so nothing can be enumerated.
- **Data privacy** — views are hand-shaped read models: no `nationalIdNumber`, address, archive
  metadata, passwords or internal timestamps ever leave the server. Dashboard guardians are
  summaries; profiles add phone/email/occupation. Academic views list the class's non-archived,
  ACTIVE subjects only (no results/grades — those arrive with the exams phase).
- Every portal read writes an audit event (`portal.student.dashboard_viewed`,
  `portal.parent.student_academic_viewed`, … — 10 actions).

Additional dev bootstrap (non-production): `demo-student@schoolerp.local` /
`Demo@123456` (STUDENT → `STU-0001`) and `demo-parent@schoolerp.local` / `Demo@123456`
(GUARDIAN → `dev-guardian-001`, linked to both `STU-0001` and `STU-0002`).

The frontend adds a role-aware `AppShell` (STUDENT and GUARDIAN users see portal nav instead of
admin items) and ten client pages wired through `lib/hooks/use-portal.ts`:
`/portal/student{/profile,/academic,/enrollment}` and
`/portal/parent{/profile,/students}` +
`/portal/parent/students/:id{,/academic,/enrollment}`.

## Secondary Academic Modules (Phase 8)

Phase 8 adds the secondary academic modules under `/api/v1`, still database-free behind the same
repository-port pattern. Every record is tenant- and school-scoped, reusing the Phase 4 system
stores, the Phase 5 `RegistryDb` (students, enrollments) and the Phase 6 `AcademicsDb` (academic
years, terms, classes, sections, subjects, placements) through one shared `SecondaryDb` store.
Branch-restricted principals operate inside their branch's school only, and every mutating action
emits a structured audit event.

- **Attendance** (`POST /api/v1/attendance` batch entry, `GET /api/v1/attendance`,
  `GET/PATCH /api/v1/attendance/:id`, `GET /attendance/student/:studentId{,/summary}`,
  `GET /attendance/class/:classId/summary`) — daily `PRESENT / ABSENT / LATE / LEAVE` records per
  student/class/term with bulk entry and per-student / per-class summaries.
- **Timetables + Entries** (`GET/POST /api/v1/timetables`, `GET/PATCH /api/v1/timetables/:id`,
  `GET/POST /api/v1/timetable-entries`, `GET/PATCH/DELETE /api/v1/timetable-entries/:id`) — weekly
  per class/section timetables with day-of-week entries (subject, teacher, room, time slot); one
  ACTIVE timetable per class/section.
- **Homework** (`GET/POST /api/v1/homework`, `GET/PATCH/DELETE /api/v1/homework/:id`,
  `POST /homework/:id/publish`) — `DRAFT → PUBLISHED → ARCHIVED` lifecycle.
- **Exams + Exam Subjects** (`GET/POST /api/v1/exams`, `GET/PATCH/DELETE /api/v1/exams/:id`,
  `POST /exams/:id/{publish|complete}`, `GET/POST /api/v1/exams/:id/subjects`,
  `GET/PATCH/DELETE /api/v1/exam-subjects/:id`) — `DRAFT → SCHEDULED → PUBLISHED → COMPLETED`
  lifecycle with linked subject papers (date, time, duration, total/pass marks).
- **Results** (`GET/POST /api/v1/results`, `GET/PATCH /api/v1/results/:id`,
  `GET /results/student/:studentId`, `GET/POST /results/exams/:examId{,/publish}`) — marks entry
  per exam subject and student, a publish workflow, and computed percentages + letter grades.

**RBAC permission keys** — `attendance.{read,create,update}`, `timetables.{read,create,update}`,
`timetable_entries.{read,create,update,delete}`, `homework.{read,create,update,publish,delete}`,
`exams.{read,create,update,publish,complete,delete}`, `exam_subjects.{read,create,update,delete}`,
`results.{read,enter,update,publish}` — added to SUPER_ADMIN / TENANT_ADMIN / SCHOOL_ADMIN and
(selectively) TEACHER role sets; the portal roles are untouched.

**Portal read-only expansion** — ten new read-only endpoints under
`/api/v1/portal/{student,parent/students/:studentId}/{attendance,timetable,homework,exams,results}`,
reusing the Phase 7 server-side portal identity binding and the `student_guardians` gating. Portal
views are privacy-shaped (no `nationalIdNumber`, addresses or archive metadata); homework shows
published records only, exams show published subjects, results show published marks.

All Phase 8 data remains **in-memory** behind repository ports (database deferred, ADR-031); the
`SecondaryDb` adapters are swapped for a Mongoose-backed implementation behind the same ports in the
DB phase. Non-production dev bootstrap seeds a full demo via `seedSecondaryDemo()` (attendance
history, `Demo Timetable` with six sessions, PUBLISHED + DRAFT homework, `Term 1 Midterm
Examination` with three subject papers, published results) exercised by `demo-admin@schoolerp.local`
and `demo-teacher@schoolerp.local`, while `demo-student@schoolerp.local` and
`demo-parent@schoolerp.local` see it through the portals (password `Demo@123456` for all four).

The frontend adds five admin client pages (`/attendance`, `/timetables`, `/homework`, `/exams`,
`/results`) wired through `lib/hooks/use-secondary.ts`, plus ten portal pages
(`/portal/student/{attendance,timetable,homework,exams,results}`) and
(`/portal/parent/students/[id]/{attendance,timetable,homework,exams,results}`) wired through
`lib/hooks/use-portal-secondary.ts`.

Test commands (from the repo root):

```bash
npm run test --workspace @schoolerp/api     # 36 files / 391 tests (incl. new secondary suites)
npm run test --workspace @schoolerp/web     # 10 files / 38 tests
npm run typecheck                           # typecheck all workspaces
npm run lint                                # lint all workspaces
npm run build                               # api (tsc) + web (Next.js, 46 routes)
npm run test:e2e                            # Playwright incl. secondary-admin 5/5 + portal-phase8
```

OpenAPI now documents **51 Phase 8 + portal path blocks** across 7 new tags (Attendance,
Timetables, Timetable Entries, Homework, Exams, Exam Subjects, Results); Swagger UI serves it at
`/api-docs` in dev.

## Communications & Notifications (Phase 9)

Phase 9 adds the Communications module — structured announcements and notices — under `/api/v1`,
still database-free behind the repository-port pattern. Records are tenant- and school-scoped with
audience targeting (SCHOOL / BRANCH / CLASS / STUDENT), a `DRAFT → PUBLISHED → ARCHIVED` lifecycle,
and a per-viewer `read / unread` state that is independent for students vs. guardians.

- **Communications** (`POST /api/v1/communications`, `GET /api/v1/communications`,
  `GET/PATCH/DELETE /api/v1/communications/:id`, `POST /api/v1/communications/:id/publish`) —
  announcement/notice records with optional `expiresAt`; published items that have expired are
  hidden from the portals; `DELETE` archives (never destructive).
- **Student portal** (`GET /api/v1/portal/student/communications`,
  `GET /api/v1/portal/student/communications/:communicationId`,
  `POST /api/v1/portal/student/communications/:communicationId/read`) — the bound student sees all
  published, unexpired communications targeting their school / branch / class / student id, with an
  `unreadCount`.
- **Parent portal** (`GET /api/v1/portal/parent/students/:studentId/communications`,
  `GET /api/v1/portal/parent/students/:studentId/communications/:communicationId`,
  `POST .../communications/:communicationId/read`) — per linked student, same visibility rules;
  read state is tracked per viewer, so a guardian marking something read never changes the
  student's state (and vice versa).

**RBAC permission keys** — `communication.{read,create,update,publish,archive}` — granted to
SUPER_ADMIN / TENANT_ADMIN / SCHOOL_ADMIN in full; TEACHER gets `read/create/update` only (publish
is admin-only). Portal roles get the read + mark-read surfaces through their existing
`portal.*.read` permissions.

**Audit events** — `communication.created`, `communication.updated`,
`communication.published`, `communication.archived`, `communication.read`,
`portal.student.communication_viewed`, `portal.parent.student_communication_viewed` — each carrying
actorId + tenantId.

All Phase 9 data remains **in-memory** behind repository ports (database deferred, ADR-031):
`CommunicationsDb` holds the records and per-viewer read markers, exposed through the
`CommunicationRepository` and `CommunicationReadRepository` ports for a drop-in Mongoose swap in the
DB phase. Real-time push / email / SMS delivery is intentionally **deferred** to a future
notification-provider phase (ADR-038 documents the extension point). Non-production dev bootstrap
seeds demo communications via `seedCommunicationDemo()` (announcements + notices across every
audience, one DRAFT, one expired PUBLISHED notice, one pre-read branch notice), exercised by the
demo accounts (password `Demo@123456`).

The frontend adds one admin client page (`/communications`) and two portal pages
(`/portal/student/communications` and `/portal/parent/students/[id]/communications`), wired through
`lib/hooks/use-communication.ts`.

Test commands (from the repo root):

```bash
npm run test --workspace @schoolerp/api     # 38 files / 419 tests (incl. new communication suites)
npm run test --workspace @schoolerp/web     # 11 files / 46 tests
npm run typecheck                           # typecheck all workspaces
npm run lint                                # lint all workspaces
npm run build                               # api (tsc) + web (Next.js, new routes)
npm run test:e2e                            # Playwright incl. communication.spec.ts (11 passed)
```

OpenAPI now documents the Communications tag (admin + portal paths) with full request/response
schemas; Swagger UI serves it at `/api-docs` in dev.

## API Documentation

- `GET /api/v1/openapi.json` — machine-readable OpenAPI 3.0 document
- `/api-docs` — Swagger UI (served in development/staging; not in production)

---

## What is Implemented (so far)

- npm workspaces monorepo
- Next.js (App Router) frontend foundation
- Express + TypeScript backend core: app/server separation, centralized validated
  environment config, middleware pipeline (request ID, structured logging,
  security headers, CORS, request-size limits, rate limiting)
- Canonical API response standard + centralized error architecture (400/401/403/
  404/409/422/429/500), 404 handling, request-ID propagation into envelopes
- Zod validation foundation (body/query/params) + reusable pagination/sort/search
  utilities and allowlisted sort parsing
- Repository + authentication/authorization ports: users, sessions, roles
  (in-memory adapters for dev/test; database implementation deferred)
- Authentication (Phase 3): Argon2id password hashing, JWT access tokens,
  rotating opaque refresh tokens with reuse detection, session revocation,
  login/refresh/logout/me endpoints, auth rate limiting, CSRF origin guard on
  cookie-authenticated endpoints
- Authorization (Phase 3): code-level role catalog (Super Admin, Tenant Admin,
  School Admin, Teacher, Staff, Accountant, Student, Guardian) + granular
  permission keys, `requireAuthentication` / `requirePermission` / `requireRole`
  middleware, default-deny enforcement and tenant-scope checks
- Multi-tenant system administration (Phase 4): tenants, schools and branches
  lifecycle + status (ACTIVE/INACTIVE hierarchy), tenant-scoped user management
  with a privileged-account guard, tenant custom roles with runtime-resolved
  replaceable permission sets, read-only platform permission catalog, structured
  audit events on every mutating action, and a two-tenant dev/demo bootstrap
- OpenAPI 3.0 document covering every implemented endpoint (auth + Phase 4
  administration + Phase 5 admissions/students/guardians/enrollments + Phase 6
  academics + Phase 7 student & parent portals) with synchronous Swagger UI
- Students, guardians, admissions and enrollments (Phase 5): approved-document
  pipeline, one-way approval→enrollment conversion, soft-archive, shared
  guardians, single-ACTIVE-enrollment rule, sequential numbering, document
  metadata, and branch-scope enforcement on lists and single reads
- Core academics structure (Phase 6): academic years + terms with
  DRAFT→ACTIVE→CLOSED lifecycles, classes/sections/subjects catalog with
  soft-archive and restore, class–subject assignments, academic placements with
  a single-ACTIVE invariant + transfer, promotion execution (PROMOTE/RETAIN/
  TRANSFER), and reference-only teacher assignments — all school-scoped with
  service-level business rules (422), per-resource auth sub-routers, and a
  nine-page Next.js frontend
- Student & Parent Portals (Phase 7): read-only, own-record `/portal/student/*`
  and `/portal/parent/*` views resolved from a server-side portal identity
  binding, relationship-scoped (IDOR-safe, indistinguishable 404s), tenant +
  branch isolated, audit-tracked (10 actions), OpenAPI-documented, with a
  role-aware `AppShell` and ten client pages wired through
  `lib/hooks/use-portal.ts`
- Communications & Notifications (Phase 9): announcements/notices with
  SCHOOL / BRANCH / CLASS / STUDENT audiences, `DRAFT → PUBLISHED → ARCHIVED`
  lifecycle, optional `expiresAt`, per-viewer read/unread state (independent for
  students vs guardians), `unreadCount` for the portal surfaces, RBAC permission
  family + 7 audit events, demo seed, an admin page (`/communications`), and
  student + parent portal pages with mark-as-read — real-time push/email/SMS
  delivery deferred to a future notification-provider phase
- Trusted auth context types (`req.auth`)
- MongoDB connection infrastructure (Mongoose) that stays dormant when
  `MONGODB_URI` is unset (deferred integration)
- Helmet + CORS security middleware
- Lint / typecheck / build tooling
- Testing foundation (Vitest + Supertest for the API, Playwright for E2E)

## What is Planned (Not Yet Implemented)

MongoDB models/schemas/repositories (database integration deferred to the
end of the project), password reset & email verification, persistent audit
log storage & querying, teachers/staff, attendance, timetable, exams, results,
fees, finance, payments, library, transport, leave, HR, payroll, real-time
notification delivery (push / email / SMS providers on top of the Phase 9
communications records), messaging, certificates, discipline, health, events,
reports, SaaS/subscriptions, support, integrations, background jobs, document
file storage, AI features, and the full portal build-out (results, fees and
notification views on top of the Phase 7 read-only foundation).

---

> This project is a work in progress. Features listed as "planned" are not present in the codebase.
