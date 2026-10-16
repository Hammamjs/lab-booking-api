# Lab Booking & Attendance System — Back-end Backlog

This file contains the backlog tickets planned for the backend repo (lab-booking-api). Tickets are small (1–3 days). Follow the Definition of Done in the project brief.

---

## Foundation (first priority)

### Foundation-01 — Define OpenAPI contract (core endpoints)
- Description: Create an OpenAPI v3 spec in `/api/spec/openapi.yaml` covering core P0 endpoints used by frontend and mobile app. Include schemas, security schemes (JWT), and basic examples.
- Acceptance criteria:
  - [ ] `api/spec/openapi.yaml` exists in the repo.
  - [ ] Contains paths for: /auth (register/login/refresh), /resources, /resources/{id}/availability, /bookings (POST, GET /bookings/me, DELETE), /attendance/sessions (POST), /attendance/sessions/{id}/qr (GET), /attendance/scan (POST), /devices (POST), /users/me (GET).
  - [ ] SecuritySchemes defines bearer JWT and refresh token approach.
  - [ ] Basic request/response examples for key endpoints.
- Dependencies: none
- Label: backend
- Priority: P0
- Size: M

### Foundation-02 — Database schema & initial migration
- Description: Add ORM schema (Prisma or Drizzle) and initial migration files matching the data model from the brief.
- Acceptance criteria:
  - [ ] `prisma/schema.prisma` (or `drizzle` schema) committed and matches users, devices, resources, blocked_periods, bookings, booking_attendees, attendance_sessions, attendance_records.
  - [ ] Initial SQL migration(s) added under `prisma/migrations` (or migrations folder) and run successfully against a local Postgres container.
  - [ ] Timestamps use `timestamptz`.
- Dependencies: Foundation-01
- Label: db
- Priority: P0
- Size: M

### Foundation-03 — Add btree_gist extension and bookings exclusion constraint migration
- Description: Add a raw SQL migration enabling `btree_gist` and create the exclusion constraint to prevent overlapping CONFIRMED bookings.
- Acceptance criteria:
  - [ ] Raw SQL migration file added and documented (path: `migrations/XXXX_btress_gist_and_exclusion.sql`).
  - [ ] Migration enables `btree_gist` and creates the exclusion constraint on (resource_id, tstzrange(start_time, end_time)) WHERE status='CONFIRMED'.
  - [ ] Integration test demonstrates conflicting insert results in a DB error mapped to 409.
- Dependencies: Foundation-02
- Label: db
- Priority: P0
- Size: M

### Foundation-04 — Backend NestJS project skeleton with modules & Swagger
- Description: Bootstrapped NestJS app with empty modules: auth, users, devices, resources, bookings, attendance, notifications, reports, shared. Add Swagger config, global validation pipe, global exception filter.
- Acceptance criteria:
  - [ ] `src/` structure with modules exists and app boots via `npm run start:dev`.
  - [ ] Swagger integration present and serves API docs at `/api/docs` in dev.
  - [ ] Global validation pipe and exception filter wired.
  - [ ] README includes run instructions and Docker Compose entry.
- Dependencies: Foundation-01, Foundation-02
- Label: backend
- Priority: P0
- Size: M

### Foundation-05 — (shared) Frontend OpenAPI placeholder (used by backend teams)
- Description: Add the initial `api/spec/openapi.yaml` placeholder and CI step ensuring OpenAPI is present; allows frontend to generate client.
- Acceptance criteria:
  - [ ] `api/spec/openapi.yaml` in the backend repo with basic info section and minimal schemas.
  - [ ] CI step validates OpenAPI is present.
- Dependencies: Foundation-01
- Label: backend
- Priority: P0
- Size: S

---

## Auth

### Auth-01 — POST /auth/register
- Description: Implement registration endpoint. Hash passwords with bcrypt and create user with role `student`.
- Acceptance criteria:
  - [ ] Validation of input using class-validator.
  - [ ] Password stored hashed (bcrypt).
  - [ ] Duplicate email returns 409.
  - [ ] Tests and OpenAPI docs updated.
- Dependencies: Foundation-02, Foundation-04, Foundation-01
- Label: backend
- Priority: P0
- Size: S

### Auth-02 — POST /auth/login
- Description: Implement login to return short-lived access JWT and refresh token; persist refresh token or revocation data.
- Acceptance criteria:
  - [ ] Valid credential returns { accessToken, refreshToken }.
  - [ ] Access token contains user id and role.
  - [ ] Tests and OpenAPI docs updated.
- Dependencies: Auth-01
- Label: backend
- Priority: P0
- Size: M

### Auth-03 — POST /auth/refresh
- Description: Implement refresh token exchange and revocation handling.
- Acceptance criteria:
  - [ ] Valid refresh issues new access token; invalid/expired returns 401.
  - [ ] Tests included.
- Dependencies: Auth-02
- Label: backend
- Priority: P0
- Size: M

### Auth-04 — Login rate limiting
- Description: Implement basic rate-limiting for /auth/login (in-memory for dev, configurable).
- Acceptance criteria:
  - [ ] 429 returned after configured failed attempts.
  - [ ] Tests for rate-limiting behavior.
- Dependencies: Foundation-06 (devops infra optional)
- Label: backend
- Priority: P0
- Size: M

### Auth-05 — Role guards and JWT validation
- Description: Add NestJS guards for roles and global auth guard validating access tokens.
- Acceptance criteria:
  - [ ] Guard can be applied at controller/route level.
  - [ ] Unauthorized/forbidden cases return 401/403.
  - [ ] Tests included.
- Dependencies: Auth-02, Foundation-04
- Label: backend
- Priority: P0
- Size: M

---

## Users (backend tickets)

### Users-01 — GET /users/me and PATCH /users/me
- Description: Read/update profile (name, contact). Prevent updating role.
- Acceptance criteria:
  - [ ] GET returns current user without password_hash.
  - [ ] PATCH validates inputs and updates allowed fields.
  - [ ] Tests for security (cannot update another user).
- Dependencies: Auth-05, Foundation-04
- Label: backend
- Priority: P0
- Size: S

### Users-02 — Admin role management (PUT /users/:id/role)
- Description: Admin endpoint to change role and log changes.
- Acceptance criteria:
  - [ ] Only admin can call; audit log entry created.
  - [ ] Tests and docs.
- Dependencies: Users-01, Auth-05
- Label: backend
- Priority: P1
- Size: S

---

## Devices (backend tickets)

### Devices-01 — POST /devices (register)
- Description: Register device row with active=false; record public_key and registered_at.
- Acceptance criteria:
  - [ ] Device created with active=false.
  - [ ] If existing active device exists, don't auto-deactivate (approval flow will handle active switch).
  - [ ] Tests and OpenAPI docs.
- Dependencies: Auth-05, Foundation-02
- Label: backend
- Priority: P0
- Size: S

### Devices-02 — PATCH /devices/:id/approve
- Description: Staff/Admin approves device; set others inactive in a transaction to enforce one active device per user.
- Acceptance criteria:
  - [ ] Approve sets device.active=true and sets prior active device(s) to false transactionally.
  - [ ] Audit log entry created.
  - [ ] Tests for concurrency.
- Dependencies: Devices-01
- Label: backend
- Priority: P0
- Size: M

### Devices-04 — Attendance DB uniqueness constraints migration
- Description: Add unique constraints for attendance_records (session_id,user_id) and (session_id,device_id).
- Acceptance criteria:
  - [ ] Migration added and tested.
  - [ ] Service maps constraint violations to 409 responses.
- Dependencies: Foundation-02, Attendance tickets
- Label: db
- Priority: P0
- Size: S

---

## Resources (backend tickets)

### Resources-01 — Resources CRUD (admin)
- Description: Implement resources CRUD endpoints (admin for writes).
- Acceptance criteria:
  - [ ] POST/PUT/DELETE require admin; GET endpoints public for browsing active resources.
  - [ ] Validation on capacity and opening hours.
  - [ ] Tests and docs.
- Dependencies: Auth-05, Foundation-04
- Label: backend
- Priority: P0
- Size: M

### Resources-03 — Blocked periods (admin)
- Description: POST/GET/DELETE blocked periods for a resource; server prevents overlapping blocked periods.
- Acceptance criteria:
  - [ ] Overlapping blocked periods are rejected with 400.
  - [ ] Tests included.
- Dependencies: Resources-01
- Label: backend
- Priority: P0
- Size: S

### Resources-04 — GET /resources/{id}/availability
- Description: Return availability ranges considering opening hours, blocked periods, and confirmed bookings.
- Acceptance criteria:
  - [ ] Excludes blocked periods and CONFIRMED bookings.
  - [ ] Returns time ranges in UTC; frontend will convert.
  - [ ] Tests for edge cases.
- Dependencies: Bookings-01, Foundation-02
- Label: backend
- Priority: P0
- Size: M

---

## Bookings (backend tickets)

### Bookings-01 — POST /bookings (create)
- Description: Create booking after validating times, opening hours, blocked periods, and attempt DB insert; handle DB exclusion conflict -> 409.
- Acceptance criteria:
  - [ ] Validations implemented: future time, within open hours, not in blocked period, within max duration.
  - [ ] Insert returns CONFIRMED on success and emits booking.created event.
  - [ ] 409 returned if exclusion constraint violated.
  - [ ] Tests for validation and success.
- Dependencies: Foundation-02, Foundation-03, Resources-04, Auth-05
- Label: backend
- Priority: P0
- Size: M

### Bookings-02 — GET /bookings/me and DELETE /bookings/:id
- Description: List user's bookings and cancel own future bookings. Admin can cancel any.
- Acceptance criteria:
  - [ ] Users can list and cancel only their future bookings.
  - [ ] Cancellation sets status=CANCELLED and emits booking.cancelled event.
  - [ ] Audit log entry created.
- Dependencies: Bookings-01
- Label: backend
- Priority: P0
- Size: S

### Bookings-04 — Concurrency e2e test for bookings (CI)
- Description: Add e2e test sending concurrent booking requests to the same slot; assert only one succeeds.
- Acceptance criteria:
  - [ ] Test runs against real Postgres in CI and passes.
- Dependencies: Foundation-06, Foundation-03, Bookings-01
- Label: test
- Priority: P1
- Size: M

---

## Attendance (backend tickets)

### Attendance-01 — POST /attendance/sessions (start session)
- Description: Staff can start an attendance session for a booking; create session secret and set status OPEN.
- Acceptance criteria:
  - [ ] Only staff may start session; session stored with secret, start_time, end_time.
  - [ ] Tests for permission.
- Dependencies: Bookings-01, Auth-05
- Label: backend
- Priority: P0
- Size: S

### Attendance-02 — GET /attendance/sessions/{id}/qr
- Description: Return an HMAC-signed rotating token for current slot; server accepts current and previous slot on validate.
- Acceptance criteria:
  - [ ] Token signed with session secret.
  - [ ] Endpoint for staff only.
  - [ ] Tests for generation and acceptance.
- Dependencies: Attendance-01
- Label: backend
- Priority: P0
- Size: M

### Attendance-03 — POST /attendance/scan
- Description: Validate signed token, device ownership/active, roster membership, and uniqueness then insert attendance_records as PRESENT.
- Acceptance criteria:
  - [ ] All validation checks implemented in order.
  - [ ] Successful scan creates PRESENT record with scanned_at and IP.
  - [ ] Duplicates return 409.
  - [ ] Tests cover rejection reasons.
- Dependencies: Attendance-01, Devices-01, Devices-04
- Label: backend
- Priority: P0
- Size: M

### Attendance-06 — Soft signals & NEEDS_REVIEW flagging
- Description: Implement heuristics (IP range, many scans from one IP) to flag records NEEDS_REVIEW.
- Acceptance criteria:
  - [ ] Flagging logic runs and stores status NEEDS_REVIEW when thresholds hit.
  - [ ] Tests for detection.
- Dependencies: Attendance-03
- Label: backend
- Priority: P0
- Size: M

### Attendance-08 — Scheduled job: close sessions past end_time
- Description: Scheduled task to close sessions and mark bookings COMPLETED (tie-in with bookings job).
- Acceptance criteria:
  - [ ] Job runs every minute and sets session status appropriately.
  - [ ] Tests simulate closure.
- Dependencies: Foundation-06, Attendance-01
- Label: backend
- Priority: P1
- Size: S

---

## Notifications (backend)

### Notifications-01 — Nodemailer integration and booking confirmation template
- Description: Implement a Notification service using Nodemailer and add a booking confirmation template; send emails asynchronously via event listener.
- Acceptance criteria:
  - [ ] Booking created event triggers an asynchronous email send.
  - [ ] Dev mailbox or logging used in dev.
  - [ ] Tests stub SMTP and assert send.
- Dependencies: Bookings-01, Foundation-04
- Label: backend
- Priority: P1
- Size: M

---

## Reports & DevOps (backend)

### Reports-02 — GET /reports/attendance/{sessionId}
- Description: Return attendance records for a session with device/IP info for staff/admin.
- Acceptance criteria:
  - [ ] Endpoint restricted to staff/admin.
  - [ ] Returns records with statuses and metadata.
  - [ ] Tests included.
- Dependencies: Attendance-03, Auth-05
- Label: backend
- Priority: P1
- Size: M

### DevOps-01 — Jest unit & integration test scaffolding
- Description: Add Jest config, integration test setup to run migrations against a test Postgres container.
- Acceptance criteria:
  - [ ] Unit and integration tests run in CI.
  - [ ] README documents test commands.
- Dependencies: Foundation-06
- Label: test
- Priority: P0
- Size: M

---

If you want these tickets created as GitHub Issues instead of a backlog file, say so and I will create issues in the backend repo next (I will need to call the github-issue tool once).