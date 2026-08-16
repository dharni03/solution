# Problem 2 — Backend Developer Challenge: Student Management CRUD

**Module:** `backend/src/modules/students/`
**Stack exercised:** Node.js · Express · PostgreSQL · REST API design · error handling

---

## Contents

1. [Objective](#1-objective)
2. [Starting State](#2-starting-state)
3. [Architecture](#3-architecture)
4. [Files Changed](#4-files-changed)
5. [Endpoints Implemented](#5-endpoints-implemented)
6. [API Reference & Examples](#6-api-reference--examples)
7. [Database Approach](#7-database-approach)
8. [Validation & Error Handling](#8-validation--error-handling)
9. [Authentication & CSRF](#9-authentication--csrf)
10. [Bug Fixed](#10-bug-fixed)
11. [Testing Performed](#11-testing-performed)
12. [Verification Summary](#12-verification-summary)
13. [Known Follow-Ups](#13-known-follow-ups)
14. [Running & Testing Locally](#14-running--testing-locally)

---

## 1. Objective

Implement complete CRUD operations — Create, Read, Update, Delete — for Student Management across:

- `students-controller.js`
- `students-service.js`
- `students-repository.js`
- `sudents-router.js` *(existing filename typo preserved)*

## 2. Starting State

- All five controller handlers were empty stubs (`//write your code`).
- No delete route existed at all.

**Design principle:** no architectural changes. The module was completed using the conventions already visible in sibling modules (`staffs`, `departments`) rather than reinventing them.

---

## 3. Architecture

| Layer | File | Responsibility |
|---|---|---|
| **Router** | `sudents-router.js` | Maps HTTP verb + path → controller function. Mounted at `/api/v1/students` in `routes/v1.js`, behind `authenticateToken` and `csrfProtection` applied at the route-group level. |
| **Controller** | `students-controller.js` | Thin request/response layer. Each handler wrapped in `express-async-handler`, so thrown errors flow to Express's error middleware instead of per-route try/catch. |
| **Service** | `students-service.js` | Business logic: existence checks (`checkStudentId`), orchestrating the add/update stored-procedure call, dispatching the verification email, translating failure states into `ApiError`s with the correct HTTP status. |
| **Repository** | `students-repository.js` | The only layer that talks to PostgreSQL, via the shared `processDBRequest` helper wrapping the `pg` pool. |
| **Database** | `seed_db/tables.sql` | `student_add_update(jsonb)` PL/pgSQL procedure handles insert **and** update in one call, branching on whether `userId` is present. Students are rows in `users` (`role_id = 3`) joined to `user_profiles`. |

---

## 4. Files Changed

| File | Change |
|---|---|
| `students-controller.js` | Implemented all five stub handlers; added a sixth, `handleDeleteStudent`. |
| `students-service.js` | Implemented `getAllStudents`, `getStudentDetail`, `addNewStudent`, `updateStudent`, `setStudentStatus`; added `removeStudent`; fixed an error-handling bug in `addNewStudent` (see §10). |
| `students-repository.js` | Implemented `findAllStudents` (with filtering); reused existing `addOrUpdateStudent` / `findStudentDetail` / `findStudentToSetStatus`; added a new `deleteStudent` query. |
| `sudents-router.js` | Added `router.delete("/:id", ...)`. |
| `Readme.md` | This section. |

**Untouched:** `seed_db/tables.sql`, `backend/src/routes/v1.js`, and every other module.

---

## 5. Endpoints Implemented

All routes require authentication.

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/students` | List students; optional `name`, `className`, `section`, `roll` query filters |
| `GET` | `/api/v1/students/:id` | Full profile detail for one student |
| `POST` | `/api/v1/students` | Create a student; dispatches an account-verification email on success |
| `PUT` | `/api/v1/students/:id` | Update an existing student's full profile |
| `POST` | `/api/v1/students/:id/status` | Enable/disable a student's system access |
| `DELETE` | `/api/v1/students/:id` | Delete a student (removes both `user_profiles` and `users` rows) |

---

## 6. API Reference & Examples

### Create — `POST /api/v1/students`

**Request**

```json
{
  "name": "Jane Doe",
  "gender": "female",
  "dob": "2011-03-14",
  "phone": "9998887777",
  "email": "jane.doe@example.com",
  "class": "5",
  "section": "A",
  "roll": "21",
  "admissionDate": "2021-06-01",
  "currentAddress": "12 Elm Street",
  "permanentAddress": "12 Elm Street",
  "fatherName": "John Doe",
  "fatherPhone": "9991112222",
  "motherName": "Mary Doe",
  "motherPhone": "9993334444",
  "guardianName": "John Doe",
  "guardianPhone": "9991112222",
  "relationOfGuardian": "Father",
  "systemAccess": true
}
```

**Response — `200`**

```json
{ "message": "Student added and verification email sent successfully." }
```

### List — `GET /api/v1/students?className=5&section=A`

```json
{
  "students": [
    {
      "id": 6,
      "name": "Jane Doe",
      "email": "jane.doe@example.com",
      "lastLogin": null,
      "systemAccess": true
    }
  ]
}
```

### Detail — `GET /api/v1/students/6`

```json
{
  "id": 6,
  "name": "Jane Doe",
  "email": "jane.doe@example.com",
  "systemAccess": true,
  "phone": "9998887777",
  "gender": "female",
  "dob": "2011-03-14T00:00:00.000Z",
  "class": "5",
  "section": "A",
  "roll": 21,
  "fatherName": "John Doe",
  "fatherPhone": "9991112222",
  "motherName": "Mary Doe",
  "motherPhone": "9993334444",
  "guardianName": "John Doe",
  "guardianPhone": "9991112222",
  "relationOfGuardian": "Father",
  "currentAddress": "12 Elm Street",
  "permanentAddress": "12 Elm Street",
  "admissionDate": "2021-06-01T00:00:00.000Z",
  "reporterName": "John Doe"
}
```

### Status toggle — `POST /api/v1/students/6/status`

```json
// request
{ "status": false }

// response
{ "message": "Student status changed successfully" }
```

### Delete — `DELETE /api/v1/students/6`

```json
{ "message": "Student deleted successfully" }
```

A subsequent `GET /api/v1/students/6` correctly returns `404 { "error": "Student not found" }`.

---

## 7. Database Approach

**Create / Update** — both go through the existing `student_add_update(jsonb)` PL/pgSQL function via `addOrUpdateStudent()`. No `userId` in the payload → insert; `userId` present → update. Reused as-is rather than replaced with hand-written SQL, since it already encodes the insert-vs-update branching, reporter-teacher lookup, and duplicate-email guard.

**Read** — `findAllStudents` and `findStudentDetail` are plain parameterized `SELECT`s joining `users` to `user_profiles` (and `users` again for the reporting teacher's name), filtered to `role_id = 3`.

**Delete** — `deleteStudent` (new) is two explicit statements: delete from `user_profiles` first, then `users`. There is no `ON DELETE CASCADE` on the `user_profiles.user_id` foreign key.

**Referential integrity** — `user_profiles.class_name` / `section_name` are foreign keys into `classes` / `sections`. Creating or updating a student with a class/section that doesn't exist fails at the database layer rather than silently succeeding.

---

## 8. Validation & Error Handling

- All service functions throw the codebase's existing `ApiError(statusCode, message)`, caught centrally by the existing `handleGlobalError` middleware. No new error-handling pattern was introduced.
- `getStudentDetail`, `setStudentStatus`, and `removeStudent` each call `checkStudentId()` first, returning `404 "Student not found"` for any operation on a non-existent ID rather than a generic failure deeper in the stack.
- `getAllStudents` returns `404 "Students not found"` when a filter matches zero rows — mirroring the existing `staffs` module's convention.
- **Body payload shape is not re-validated at the Express layer** (no Zod schema added here). The stored procedure's own `COALESCE`/casts and the database's `NOT NULL` / foreign-key constraints are the enforcement point, consistent with how `addOrUpdateStudent` already worked before this change.

---

## 9. Authentication & CSRF

No changes were made to the auth system — the students route group was already wired to use it, and the implementation was tested against it as-is.

- **`authenticateToken`** — requires valid `accessToken` and `refreshToken` JWT cookies, set at login via `POST /api/v1/auth/login`.
- **`csrfProtection`** — requires an `x-csrf-token` header matching an HMAC hash embedded in the access token's claims. The raw token value is delivered separately as a non-`httpOnly` `csrfToken` cookie at login. This applies to **every** method, including `GET`.

Every test request below was issued with both auth cookies and the `x-csrf-token` header. Omitting either was separately confirmed to return `401` / `400` / `403`.

---

## 10. Bug Fixed

`addNewStudent` previously wrapped the create step in a try/catch that discarded the actual failure reason and always threw a generic `500 "Unable to add student"` — even for a client-correctable error such as a duplicate email.

| | `POST /students` with a duplicate email |
|---|---|
| **Before** | `500 { "error": "Unable to add student" }` |
| **After** | `400 { "error": "Email already exists" }` |

The fix removes the outer catch-and-genericize block, so the specific `ApiError` thrown when the stored procedure reports `status: false` propagates unchanged. Only the separate verification-email send remains wrapped in its own try/catch — a failed email is non-fatal and still returns `200` with a *"but failed to send verification email"* message, unchanged from before.

---

## 11. Testing Performed

**Postman** — all six endpoints exercised manually against a locally running instance (`npm run dev:server` from `backend/`, PostgreSQL seeded from `seed_db/tables.sql` + `seed_db/seed-db.sql`):

1. Logged in via `POST /api/v1/auth/login` with the demo admin credentials to obtain auth cookies and the CSRF token.
2. Ran the full happy path: **create → appears in list → appears in detail → update → re-fetch confirms persistence → status toggle → delete → follow-up GET correctly 404s.**
3. Verified each of the four list filters (`name`, `className`, `section`, `roll`) narrows results correctly.
4. Verified error paths:
   - missing auth cookies → `401`
   - missing/invalid CSRF header → `400` / `403`
   - operations on a non-existent student ID → `404`
   - duplicate-email create → `400` (confirming the bug fix)
   - class/section not present in the database → create fails rather than silently succeeding

**Direct service-layer check** — the `addNewStudent` fix was additionally verified by invoking the service function directly against the live database with a known-duplicate email, confirming it threw `ApiError(400, "Email already exists")`.

---

## 12. Verification Summary

| Claim | Verified | How / Evidence |
|---|---|---|
| All 6 endpoints implemented and registered | ✅ | Source trace: router → controller → service → repository → DB; confirmed against `sudents-router.js` and `seed_db/tables.sql` |
| Full CRUD flow (create → list → detail → update → status → delete → 404) | ✅ | Postman, cookie-based session against a local instance |
| List filters narrow results correctly | ✅ | Postman |
| GET / status / DELETE on a nonexistent ID → `404` | ✅ | Postman |
| Duplicate-email create → `400 "Email already exists"` | ✅ | Postman, plus direct service-layer invocation (previously a generic `500`) |
| Invalid class/section on create fails rather than silently succeeding | ✅ | Postman; enforced by `user_profiles` foreign keys |
| Auth/CSRF required on every route | ✅ | Postman (missing cookie → `401`, missing/invalid CSRF header → `400`/`403`) |

---

## 13. Known Follow-Ups

Both identified by source-code trace and **flagged rather than fixed**, so the scope of this submission stays honest:

| Issue | Current behaviour | Cause |
|---|---|---|
| `PUT` on a nonexistent student ID | Inserts a new record instead of failing | `student_add_update` branches on payload contents, and the service layer doesn't check existence first |
| `PUT` with a duplicate email | Returns a generic `500` instead of the clean `400` that `POST` now returns | The §10 fix wasn't mirrored to the update path |

---

## 14. Running & Testing Locally

**1. Database**

```bash
createdb school_mgmt
psql -d school_mgmt -f seed_db/tables.sql
psql -d school_mgmt -f seed_db/seed-db.sql
```

**2. Install** *(auto-creates `.env` / `frontend/.env` from the `.env.example` files on first run)*

```bash
cd backend && npm install
```

**3. Run**

```bash
npm run dev:server   # API only
npm run dev          # API + frontend
```

**4. Authenticate** — `POST /api/v1/auth/login` with cookie storage enabled:

```json
{ "username": "admin@school-admin.com", "password": "3OU4zn3q6Zh9" }
```

**5. CSRF** — copy the `csrfToken` cookie value into an `x-csrf-token` header on all further requests.

**6. Exercise the endpoints** listed in [§5](#5-endpoints-implemented).

> **Note:** creating or updating a student requires a class/section pair that already exists in the `classes` / `sections` tables. Add one first — via the Classes module in the frontend, or directly in the database — if the seed data doesn't include one.
