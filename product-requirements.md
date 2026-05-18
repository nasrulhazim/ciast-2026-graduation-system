# Product Requirements Document — Graduation Registration System

| Field | Value |
|---|---|
| Document version | 1.0 |
| Status | Approved for build |
| Owner | CIAST Training Team (Nasrul Hazim) |
| Last updated | 2026-05-19 |
| Source | Derived from 27 reference commits in `graduation-system/` |
| Companion | [Training Material Guide](./material-guide/README.md) — step-by-step build instructions |

---

## 1. Executive summary

The Graduation Registration System is a web application that manages a university convocation lifecycle end to end: from the registrar uploading a student roster, through students uploading their own payment receipts, to administrators verifying payments and reconciling the financial record of the ceremony.

The system targets two clearly separated audiences — **administrators** (registrar / finance / faculty office staff) and **students** (graduating candidates) — and enforces that separation with policy-based authorization, route gates, schema constraints, and UI visibility rules. There is no shared workspace; each role sees only the data and actions appropriate to them.

This document specifies the **what** and the **why**. The companion [material guide](./material-guide/README.md) specifies the **how**.

---

## 2. Background & problem statement

Today the registrar's office runs the convocation registration process on a spreadsheet plus three email threads. The pain is real:

- The roster lives in one Excel file that is mailed around and goes out of sync with every update.
- Students email scanned receipts as PDF attachments; staff manually mark them off the spreadsheet.
- There is no audit trail of who verified a payment, when, or whether a verification was reversed.
- No automated email confirms verification to the student — they keep asking by phone.
- A graduate attending two convocations (degree, then masters) is recorded twice with no link between the two rows.

The system replaces the spreadsheet + email workflow with a single source of truth that supports search, bulk operations, automatic notifications, and a complete audit trail.

---

## 3. Goals & non-goals

### 3.1 Goals

- **G1.** Eliminate Excel as the system of record for graduation rosters.
- **G2.** Allow students to self-service their own registration — verify their personal details and upload a payment receipt without an admin in the loop.
- **G3.** Give administrators bulk tools (import, export, bulk verify, bulk delete) so a 500-student ceremony is operable by one person.
- **G4.** Provide a complete audit trail for every sensitive admin action (create / verify / revoke / delete).
- **G5.** Email each student automatically when their payment is verified.
- **G6.** Enforce privacy: a student must never be able to see another student's personal data (IC, email, matric, phone).
- **G7.** Support the same person attending multiple graduations under one account.

### 3.2 Non-goals (for v1.0)

- **NG1.** Online payment gateway integration. Students upload a manual receipt; cash/transfer/cheque handling stays out of band.
- **NG2.** Robe / cap / accommodation booking subsystems.
- **NG3.** Mobile-native apps. The web app is responsive and that is sufficient.
- **NG4.** SSO with the university identity provider. Email + password via Breeze is sufficient.
- **NG5.** Multi-tenancy across faculties / campuses. A single deployment serves the institution.
- **NG6.** Public APIs for third-party integrations.
- **NG7.** Receipt OCR / automatic amount extraction.

These are explicitly out of scope for v1.0. They are listed in §13 as future considerations.

---

## 4. Personas

### 4.1 Administrator (Admin)

- **Who:** Registrar's office staff, finance staff, faculty coordinators.
- **Authentication:** Standard email + password. The `is_admin` flag on their user record is set by another admin (or by a seeder during system bootstrap).
- **Skill level:** Comfortable with spreadsheets, web forms, and CSV files. Not technical.
- **Volume:** ~5 active admins per institution.
- **Typical day:** Uploads a CSV roster from the academic system; reviews "pending review" payments; verifies in bulk; chases students whose receipts are missing or rejected.

### 4.2 Student

- **Who:** A graduating candidate listed by the registrar on an upcoming graduation roster.
- **Authentication:** Self-registers at `/register` using the email the registrar has on file. The system links them to any matching `Student` rows automatically.
- **Skill level:** Familiar with smartphones and web forms. Many will use the system on a mobile browser.
- **Volume:** 200 – 2 000 per graduation event.
- **Typical day:** Logs in, opens **My registrations**, picks the event they are graduating from, uploads a payment receipt, waits for the verification email.

### 4.3 Guest

- **Who:** Unauthenticated visitor.
- **Capability:** Can see only the public welcome page and the registration / login forms. Cannot see any graduation, student, or roster data.

---

## 5. Glossary

| Term | Definition |
|---|---|
| Graduation | A single convocation event with a title, ceremony date, fee, and status (draft / open / closed). |
| Student (row) | A registration record on a specific Graduation. Holds the candidate's identity + payment evidence. Distinct from "Student (user)". |
| Roster | The full list of Student rows attached to a Graduation. |
| Payment receipt | A PDF / JPG / PNG file uploaded by the student (or admin) showing proof of payment, ≤ 2 MB. |
| Verified | A Student row with `verified_at` set, meaning an admin has confirmed the payment. |
| Pending review | A Student row with a receipt (`paid_at` set) but no `verified_at` yet. |
| Not paid | A Student row with neither receipt nor `paid_at`. |
| Revoke | Admin action that unsets `verified_at`, returning the row to Pending review. |
| Linked user | A `User` whose email matched a `Student` row, so `user_id` was filled in automatically on registration. |
| Hybrid UUID | The pattern of using auto-increment `id` for foreign keys and a UUIDv7 `uuid` for URLs / public references. |

---

## 6. Functional requirements

Each FR is identified for traceability. Test references point at the corresponding `tests/Feature/GraduationFlowTest.php` test name.

### FR-1. Authentication & account management

| ID | Requirement |
|---|---|
| FR-1.1 | The system shall provide self-service registration at `/register` requiring name, email, password, password confirmation. |
| FR-1.2 | The system shall provide login at `/login`, password reset at `/forgot-password`, and email verification at `/verify-email`. |
| FR-1.3 | The system shall allow any authenticated user to edit their name, email, password, and to delete their own account at `/profile`. |
| FR-1.4 | The system shall log out an authenticated user on demand and redirect them to `/`. |
| FR-1.5 | Email + password authentication is sufficient; no SSO in v1.0. |

### FR-2. User roles

| ID | Requirement |
|---|---|
| FR-2.1 | The system shall recognise exactly two roles: Administrator and Student. |
| FR-2.2 | Role is represented by a boolean `is_admin` flag on the `users` table. |
| FR-2.3 | New registrations default to Student (`is_admin = false`). |
| FR-2.4 | Promotion to Administrator happens out-of-band (database seeder / database update); there is no UI to grant admin in v1.0. |
| FR-2.5 | Every user receives a UUIDv7 at creation. All public URLs that reference a user use the UUID, never the auto-increment `id`. |

### FR-3. Graduation management

| ID | Requirement |
|---|---|
| FR-3.1 | An admin shall create, read, update, and (soft-)delete graduation events. Students shall not. |
| FR-3.2 | A graduation has the fields: `title` (string, ≤ 255), `ceremony_date` (date, must be in the future at creation), `fee` (decimal 8,2; ≥ 0; ≤ 9999.99), `status` (enum: draft / open / closed; default draft). |
| FR-3.3 | The ceremony date on **create** must be after today. The ceremony date on **update** is unrestricted (typo fixes on past events are allowed). |
| FR-3.4 | Graduation deletion is **soft delete** — the row stays in the database with `deleted_at` set. |
| FR-3.5 | A graduation **shall not be deletable** if any of its students have a non-null `paid_at`. The system shall return 403 on attempted delete and shall not render the delete button. (Financial-record guard.) |
| FR-3.6 | The graduations index lists all graduations newest-first, paginated 15 per page, with a `students_count` aggregated column. |
| FR-3.7 | The graduations index supports a `?search=<term>` query param that filters by case-insensitive substring match on `title`. The search persists across pagination. |
| FR-3.8 | Only administrators may view the graduations index. Non-admins receive a 403. |
| FR-3.9 | Non-admin users may view a graduation's detail page **only if** they have at least one Student row linked to that graduation (`students.user_id = user.id` for that graduation). Otherwise 403. |

### FR-4. Student roster management

| ID | Requirement |
|---|---|
| FR-4.1 | A student row has the fields: `name`, `ic` (Malaysian IC, exactly 12 characters), `email`, `matric_card`, `phone`, `payment_receipt` (file path, nullable), `verified_at` (nullable), `paid_at` (nullable), `graduation_id` (required), `user_id` (nullable). |
| FR-4.2 | The triplet (`graduation_id`, `ic`), (`graduation_id`, `email`), and (`graduation_id`, `matric_card`) shall each be unique. Global uniqueness is **not** enforced — the same person may appear in multiple graduations. |
| FR-4.3 | An admin shall create a student via `+ Add student` on the graduation show page. |
| FR-4.4 | An admin shall delete a student row (hard delete — no soft delete on students). |
| FR-4.5 | The admin shall be able to view, search, filter, sort, and paginate the full student roster of any graduation. |
| FR-4.6 | Non-admin users shall **never** see the full roster of any graduation, even one they are registered for. |
| FR-4.7 | A linked student (a non-admin user whose `user_id` matches a Student row) shall see only their **own** registration card on the graduation show page, with a "View / upload receipt" CTA pointing at their student detail page. |
| FR-4.8 | A non-admin user without any registration on a graduation cannot reach the graduation show page (403). |
| FR-4.9 | When a graduation is hard-deleted (administratively, not via the UI), its students cascade-delete; when a user is deleted, their students' `user_id` cascade-nulls. |

### FR-5. Bulk operations

#### FR-5.1 CSV import

| ID | Requirement |
|---|---|
| FR-5.1.1 | An admin shall import a CSV roster into a graduation via `Import CSV` on the graduation show page. |
| FR-5.1.2 | The CSV shall have columns: `name`, `ic`, `email`, `matric_card`, `phone`. Header row is required. |
| FR-5.1.3 | Each row shall be validated individually using the same rules as `+ Add student` (incl. unique-within-graduation rules from FR-4.2). |
| FR-5.1.4 | Invalid rows shall be **silently skipped**, not rejected. The system shall report a summary flash message: `Imported X. Skipped Y.` |
| FR-5.1.5 | The CSV upload itself shall be validated: file type csv/txt, max size 2 MB. |
| FR-5.1.6 | The import shall stream the file (no full in-memory load) so a 50 000-row CSV does not exceed memory. |

#### FR-5.2 CSV export

| ID | Requirement |
|---|---|
| FR-5.2.1 | An admin shall export the current filtered roster of a graduation via `Export CSV`. |
| FR-5.2.2 | The export shall honour the active `?search=` and `?status=` filters. |
| FR-5.2.3 | The CSV shall include columns: `name`, `ic`, `email`, `matric_card`, `phone`, `paid_at`, `verified_at` (in ISO-8601). |
| FR-5.2.4 | The filename shall be `<slug-of-graduation-title>-students-<YYYYMMDD-HHMMSS>.csv`. |
| FR-5.2.5 | The export shall stream rows to `php://output` without buffering, supporting datasets of any size within Apache/Nginx timeout. |

#### FR-5.3 Bulk verify / delete

| ID | Requirement |
|---|---|
| FR-5.3.1 | An admin shall select one or more students on the roster via checkboxes, choose an action (Verify payments / Delete), and apply. |
| FR-5.3.2 | A header "select all" checkbox shall toggle every row checkbox. |
| FR-5.3.3 | The bulk endpoint shall scope the IDs to the current graduation. Cross-graduation IDs shall be silently dropped (security: prevents data leak across graduations). |
| FR-5.3.4 | Bulk verify shall act only on rows that have a `payment_receipt` and are not already verified. Other rows are skipped. The flash shall report: `Verified X. Skipped Y (no receipt or already verified).` |
| FR-5.3.5 | Bulk verify shall send a `PaymentVerified` email and write an `Audit` row (with `changes.via = 'bulk'`) for every successfully verified row. Skipped rows shall not be emailed and shall not be audited. |
| FR-5.3.6 | Bulk delete shall hard-delete the scoped rows and write an `Audit` row per deletion with a snapshot of the row's `name` and `ic`. |
| FR-5.3.7 | Submitting with zero rows selected shall be blocked client-side with an alert. |

### FR-6. Roster search, filter, sort, pagination

| ID | Requirement |
|---|---|
| FR-6.1 | The roster shall paginate 15 rows per page, newest first by default. |
| FR-6.2 | The roster shall support a `?search=<term>` query that filters rows whose `name`, `ic`, `email`, or `matric_card` contains the term (case-insensitive, substring). |
| FR-6.3 | The roster shall support a `?status=verified|pending|not_paid` filter mapped to: verified = `verified_at IS NOT NULL`; pending = `paid_at IS NOT NULL AND verified_at IS NULL`; not_paid = `paid_at IS NULL`. |
| FR-6.4 | The roster shall support a `?sort=<column>&direction=<asc|desc>` parameter with a strict whitelist of sortable columns: `name`, `ic`, `email`, `matric_card`, `created_at`. Unknown columns shall fall back to `created_at` silently — never error, never leak schema. |
| FR-6.5 | Search, filter, sort, and pagination shall **compose** — all combinations work, all parameters persist across pagination via `withQueryString()`. |
| FR-6.6 | Column headers Name / IC / Matric shall be click-to-sort. Clicking the same header twice toggles `asc` ↔ `desc`. Switching columns resets to `asc`. An arrow indicator (▲ / ▼) marks the active column. |
| FR-6.7 | A `Clear` link shall appear when any of search / status is active, restoring the default view. |
| FR-6.8 | The empty state shall distinguish "no matches for your search" from "roster is empty". |

### FR-7. Payment workflow

| ID | Requirement |
|---|---|
| FR-7.1 | A student row shall accept a payment receipt upload: PDF, JPG, JPEG, or PNG; max 2 MB. |
| FR-7.2 | On successful upload, `paid_at` shall be stamped with the current time. A subsequent edit without a new file shall leave `paid_at` unchanged. |
| FR-7.3 | Uploaded receipts shall be stored on the `public` filesystem disk under `receipts/`. The deployment shall provide a symlink at `public/storage → storage/app/public` so receipts are reachable via `/storage/receipts/<file>`. |
| FR-7.4 | An admin shall verify a payment via a `Verify payment` button on the student detail page. Verifying sets `verified_at` to the current time. |
| FR-7.5 | An admin shall be able to verify **only if** the student has a `payment_receipt` and is not already verified. The button shall not render otherwise. |
| FR-7.6 | An admin shall revoke a verified payment via a `Revoke verification` button (red, requires confirm) on the same page. Revoking unsets `verified_at`. |
| FR-7.7 | Verify and Revoke shall be **mutually exclusive** — at most one button renders for a given student at a time, enforced by policy. |
| FR-7.8 | Revoke shall **not** send any email to the student. (Revocation is an admin correction; emailing the student would create more confusion than clarity.) |
| FR-7.9 | Both verify and revoke shall write an `Audit` row (actions `verified` and `revoked`). |

### FR-8. Self-service registration & multi-registration

| ID | Requirement |
|---|---|
| FR-8.1 | When a new (non-admin) `User` is created, the system shall automatically link them to any existing `Student` rows whose `email` matches and `user_id` is still null. This linkage runs in a single `UPDATE` query — no N+1. |
| FR-8.2 | Admin user registration shall not trigger the linkage. |
| FR-8.3 | The linkage shall update **all** matching rows. A person who is on the Degree 2024 roster and the Masters 2026 roster gets both `user_id` columns filled in. |
| FR-8.4 | The relationship from `User` to `Student` shall be one-to-many (`HasMany`), not one-to-one. |
| FR-8.5 | The system shall provide a `/my-registrations` page that lists every Student row linked to the authenticated user, with the graduation title, ceremony date, payment-status badge, and a direct link to each student detail page. |
| FR-8.6 | The "My registration(s)" nav link shall be hidden when the user has 0 registrations, singular when they have 1, plural when ≥ 2. |
| FR-8.7 | An empty-state message shall appear on `/my-registrations` when the user has no registrations. |

### FR-9. Authorization & privacy

| ID | Requirement |
|---|---|
| FR-9.1 | All authorization decisions shall live in Policy classes (`GraduationPolicy`, `StudentPolicy`). Inline `if ($user->isAdmin())` checks in controllers or views are forbidden. |
| FR-9.2 | Policies shall be called from three places: Form Request `authorize()`, controller `$this->authorize()`, and Blade `@can(...)`. The same rule travels with the same name across all three layers. |
| FR-9.3 | The authorization matrix in §8 is normative. |
| FR-9.4 | A non-admin user shall **never** see another student's IC, email, matric, or phone — anywhere in the app. (Roster privacy.) |
| FR-9.5 | A non-admin user shall not see the Graduations nav link nor the graduations index. |
| FR-9.6 | URLs that 403 shall return a real 403 — not a 200 with a sanitised page. |

### FR-10. Audit trail

| ID | Requirement |
|---|---|
| FR-10.1 | The system shall record an `Audit` row for every sensitive admin action on a Student: create, verify, revoke, delete (single and bulk). |
| FR-10.2 | An `Audit` row shall capture: `user_id` (the actor; nullable because users can be deleted), `action` (string), `auditable_type` + `auditable_id` (polymorphic pointer to the affected model), `changes` (json), `created_at`. |
| FR-10.3 | For bulk actions, `changes.via` shall be `"bulk"`. |
| FR-10.4 | For deletes, `changes` shall include a snapshot of the deleted row's `name` and `ic` so the audit row is interpretable after the source row is gone. |
| FR-10.5 | The student detail page shall render an Audit log panel below the verify/revoke button, **admin-only**. Each entry shall display action, actor name (or "system"), bulk indicator, and a humanised timestamp. |
| FR-10.6 | The Audit table shall index `(auditable_type, auditable_id)` for fast per-model lookups. |
| FR-10.7 | Audit rows are append-only. The system shall provide no UI to edit or delete an audit row. |

### FR-11. Email notifications

| ID | Requirement |
|---|---|
| FR-11.1 | The system shall send a `PaymentVerified` email when an admin verifies a student's payment (single or bulk verify branch). |
| FR-11.2 | The email shall be routed via the **Student's** `email` column, not via the linked User's email. (The student row's email is the canonical contact admins have at hand and works even before the student has registered.) |
| FR-11.3 | The email shall include the graduation title, ceremony date (`d M Y`), fee (`RM x.xx`), and a friendly closing line. |
| FR-11.4 | The verify flash message shall append `Email sent to <email>.` (or `Email notifications sent.` for bulk). |
| FR-11.5 | Revocation shall not trigger any email. (See FR-7.8.) |
| FR-11.6 | Local development shall use `MAIL_MAILER=log` so emails write to `storage/logs/laravel.log` rather than being delivered. |

### FR-12. Dashboard

| ID | Requirement |
|---|---|
| FR-12.1 | The dashboard at `/dashboard` shall display five top-line stat cards: total graduations, total students, verified, pending review, not paid. |
| FR-12.2 | The dashboard shall display a "Latest graduations" table (10 most recent) with one row per graduation showing students count + verified / pending / not-paid counts. |
| FR-12.3 | The per-graduation breakdown shall be computed in a **single SQL query** using `withCount` with constrained alias closures — no N+1. |
| FR-12.4 | Each row in the latest-graduations table shall link to the corresponding graduation show page. |
| FR-12.5 | Empty states (no graduations) shall be handled gracefully. |
| FR-12.6 | The dashboard is the post-login landing page. |

### FR-13. UI / UX requirements

| ID | Requirement |
|---|---|
| FR-13.1 | The interface shall be in English, with Malaysian context where relevant (date format `d M Y`, currency `RM`, `Asia/Kuala_Lumpur` timezone, `ms_MY` locale for seeded names/phones). |
| FR-13.2 | The interface shall be responsive: the navigation collapses to a hamburger on small screens; tables remain horizontally scrollable. |
| FR-13.3 | The status badge palette shall be: draft = slate, open = green, closed = amber (graduation status); verified = green, pending review = amber, not paid = slate (student status). |
| FR-13.4 | All file-upload forms shall include `enctype="multipart/form-data"`. |
| FR-13.5 | All non-GET forms shall include a CSRF token and the correct `@method(...)` directive. |
| FR-13.6 | Validation errors shall be rendered inline beside each field. Old input shall be preserved on validation failure. |
| FR-13.7 | Flash messages shall appear at the top of each page (green for success). |
| FR-13.8 | Destructive actions (Archive graduation, Remove student, Revoke verification) shall require a JavaScript `confirm()` before submission. |

### FR-14. Data integrity & schema rules

| ID | Requirement |
|---|---|
| FR-14.1 | Every domain table shall have both an auto-increment `id` (for foreign keys and joins) and a UUIDv7 `uuid` (for URLs and public references). |
| FR-14.2 | All foreign keys shall be declared with `foreignIdFor(Model::class)`, not `foreignId('model_id')`. |
| FR-14.3 | Cascade rules: deleting a Graduation cascade-deletes its Students. Deleting a User nulls the `user_id` on their Students (their registrations survive). |
| FR-14.4 | `Graduation` shall use soft deletes. `Student` shall use hard deletes. |
| FR-14.5 | Every PHP file shall begin with `declare(strict_types=1);`. |
| FR-14.6 | Every controller / model method shall declare parameter and return types. |

---

## 7. Data model

```
users (id, uuid, name, email, password, is_admin, email_verified_at, …, timestamps)
   │
   │ 1:N (nullable on Student side)
   ▼
students (id, uuid, graduation_id, user_id, name, ic, email, matric_card, phone,
          payment_receipt, paid_at, verified_at, timestamps)
   ▲
   │ N:1
   │
graduations (id, uuid, title, ceremony_date, fee, status, timestamps, deleted_at)

audits (id, uuid, user_id, action, auditable_type, auditable_id, changes (json), timestamps)
```

### 7.1 Uniqueness rules

- `users.email` — globally unique.
- `students.(graduation_id, ic)` — composite unique.
- `students.(graduation_id, email)` — composite unique.
- `students.(graduation_id, matric_card)` — composite unique.

Note: `ic` and `email` on `students` are **not** globally unique. The same physical person may legitimately appear on the Degree 2024 roster *and* the Masters 2026 roster.

### 7.2 Status state machine — Student row

```
                       upload receipt
[ Not paid ]  ─────────────────────────────────►  [ Pending review ]
                                                          │
                                                          │ admin: Verify
                                                          ▼
                                                  [ Verified ]
                                                          │
                                                          │ admin: Revoke
                                                          ▼
                                                  [ Pending review ]
```

Transitions are gated by policy:
- `paid_at` is set automatically on receipt upload.
- `verified_at` is set/unset only via `verify` / `revoke` policy + controller actions.

### 7.3 Status state machine — Graduation row

```
[ Draft ]  ──►  [ Open ]  ──►  [ Closed ]
                                   │
                                   │ soft delete (only if no paid students)
                                   ▼
                          [ Archived (soft-deleted) ]
```

---

## 8. Authorization matrix

| Action | Admin | Linked Student | Other Student | Guest |
|---|:-:|:-:|:-:|:-:|
| Register an account | ✓ (out of band) | ✓ | ✓ | ✓ |
| Log in / log out | ✓ | ✓ | ✓ | — |
| Edit own profile | ✓ | ✓ | ✓ | — |
| Dashboard | ✓ | ✓ | ✓ | — |
| **Graduations index** | ✓ | ✗ | ✗ | ✗ |
| Graduation create / edit | ✓ | ✗ | ✗ | ✗ |
| Graduation delete (soft) | ✓ (if no paid students) | ✗ | ✗ | ✗ |
| **Graduation show (own)** | ✓ | ✓ | ✗ | ✗ |
| Graduation show (other) | ✓ | ✗ | ✗ | ✗ |
| **Roster view (any graduation)** | ✓ | ✗ | ✗ | ✗ |
| Student create on graduation | ✓ | ✗ | ✗ | ✗ |
| Student delete | ✓ | ✗ | ✗ | ✗ |
| Student view (any) | ✓ | own only | ✗ | ✗ |
| Student edit (any) | ✓ | own only | ✗ | ✗ |
| Upload receipt for student | ✓ (any) | own only | ✗ | ✗ |
| Verify payment | ✓ (with receipt, not yet verified) | ✗ | ✗ | ✗ |
| Revoke payment | ✓ (only if already verified) | ✗ | ✗ | ✗ |
| Bulk verify / delete | ✓ | ✗ | ✗ | ✗ |
| CSV import / export | ✓ | ✗ | ✗ | ✗ |
| View audit log on a student | ✓ | ✗ | ✗ | ✗ |
| `/my-registrations` | ✓ (if any) | ✓ | (empty if no registrations) | ✗ |

Note: "Linked Student" means a non-admin user whose `user_id` matches at least one Student row on the target graduation. "Other Student" means a non-admin user with no Student row on the target graduation.

---

## 9. User journeys

### 9.1 Admin onboards a graduation

1. Admin logs in → lands on `/dashboard`.
2. Admin navigates to **Graduations** → clicks **+ New graduation**.
3. Fills in title, future ceremony date, fee, status (typically `draft` while preparing, flips to `open` when ready) → submits.
4. On the graduation show page, admin clicks **Import CSV** → uploads the academic-office roster.
5. System imports valid rows, flashes `Imported X. Skipped Y.`
6. Admin reviews the roster, optionally edits or removes rows.

### 9.2 Student registers and uploads receipt

1. Registrar's office tells the student to register at `<system-url>/register` using their official email.
2. Student creates an account at `/register` with name, email (matches the roster), password.
3. On first login, **My registrations** appears in the nav. Student clicks it.
4. Student picks the relevant graduation → clicks **View / upload receipt**.
5. Student edits their personal info if needed (their own row only) and uploads a PDF/JPG/PNG receipt → submits.
6. Status badge flips from "Not paid" to "Pending review".
7. Student waits.

### 9.3 Admin verifies payments in bulk

1. Admin opens the graduation show page → clicks the **Pending review** filter pill.
2. Roster now shows only students with a receipt and no verification yet.
3. Admin reviews receipts (clicking through to each, optionally), selects all valid ones with the header checkbox.
4. Selects action "Verify payments" → clicks **Apply to selected**.
5. System verifies the rows, sends an email to each verified student, writes audit rows, flashes `Verified N. Skipped 0 (no receipt or already verified). Email notifications sent.`
6. Each student receives an email: subject `Your payment is verified — June 2026 Convocation`.

### 9.4 Admin revokes a wrong verification

1. Admin opens the affected student's detail page.
2. Below the receipt link, the **Revoke verification** button (red) is now visible (was hidden before verification).
3. Admin clicks Revoke → confirms.
4. Student's `verified_at` is cleared. Status returns to "Pending review". Audit row with action `revoked` appears.
5. No email is sent.

### 9.5 Same person, second graduation

1. Two years after the degree convocation, the registrar uploads the Masters 2028 roster, which includes the same person at the same email.
2. The new Student row joins their existing `user_id` automatically (the `User::booted()` hook fires on any new user; here, because the user already exists, the row is linked at insert time via the same lookup logic).
3. Student logs in → **My registrations** label changes from singular to plural → both events listed, each with its own status badge.

---

## 10. Non-functional requirements

### 10.1 Performance

- **NFR-Perf-1.** The dashboard shall render in **3 SQL queries or fewer**, regardless of the number of graduations on the system.
- **NFR-Perf-2.** Roster pages with up to 5 000 students shall paginate within 200 ms server time on a development laptop.
- **NFR-Perf-3.** CSV import and CSV export shall stream — memory usage shall not grow linearly with row count.
- **NFR-Perf-4.** No code path shall introduce N+1 queries. (Use `withCount`, eager-loading, or single-query updates.)

### 10.2 Security

- **NFR-Sec-1.** All authenticated routes shall sit behind the `auth` middleware group.
- **NFR-Sec-2.** All sort/filter parameters that map to SQL identifiers shall be validated against a strict whitelist before being passed to the query builder.
- **NFR-Sec-3.** All write actions shall be authorized via Policy, not via inline conditionals.
- **NFR-Sec-4.** Bulk actions shall be scoped to the parent resource (`$graduation->students()->whereIn(...)`); IDs from outside the parent shall be silently dropped, never executed.
- **NFR-Sec-5.** User input rendered in HTML shall be escaped by default. Unescaped rendering (`{!! !!}`) shall be used only for whitelist-controlled content (e.g. the sort header link builder).
- **NFR-Sec-6.** File uploads shall be validated by MIME type and capped at 2 MB.
- **NFR-Sec-7.** Passwords shall be hashed with bcrypt via Laravel's `hashed` cast.
- **NFR-Sec-8.** CSRF tokens shall be required on every non-GET request.

### 10.3 Reliability

- **NFR-Rel-1.** The system shall not lose an uploaded receipt. Files are stored on the `public` disk; the application database carries the file path.
- **NFR-Rel-2.** Soft-deleted graduations shall be restorable (policy supports `restore`).
- **NFR-Rel-3.** Audit rows shall persist independently of the rows they describe (snapshot fields on delete).

### 10.4 Observability

- **NFR-Obs-1.** Laravel's default logging shall capture errors to `storage/logs/laravel.log`.
- **NFR-Obs-2.** In local development, emails shall log to the same file (`MAIL_MAILER=log`) so behaviour can be inspected without a real SMTP server.
- **NFR-Obs-3.** The audit log doubles as a partial application audit; future work may add request-level audit middleware.

### 10.5 Testability

- **NFR-Test-1.** The system shall ship with ≥ 84 Pest feature tests covering happy paths, authorization boundaries, and key edge cases (no-receipt verify, cross-graduation bulk leak, fall-back sort, etc.).
- **NFR-Test-2.** Tests shall use `RefreshDatabase`, `Storage::fake`, and `Notification::fake` so the suite is hermetic — no real files written, no real emails sent.
- **NFR-Test-3.** All test assertions shall be order-precise where order matters (use `assertSeeInOrder`, not `strpos` comparisons).

### 10.6 Internationalisation & localisation

- **NFR-i18n-1.** Default locale: English. Timezone: `Asia/Kuala_Lumpur`.
- **NFR-i18n-2.** Faker locale for seeded data: `ms_MY` (Malaysian names + phone formats).
- **NFR-i18n-3.** Currency display: `RM x,xxx.xx` with two decimal places.
- **NFR-i18n-4.** Date display: `d M Y` (e.g. `15 Jun 2026`).
- **NFR-i18n-5.** Multi-language UI translation is out of scope for v1.0.

### 10.7 Accessibility

- **NFR-A11y-1.** Form fields shall have associated `<label>` elements via Breeze's `x-input-label` component.
- **NFR-A11y-2.** Status badges shall convey state by **text content**, not colour alone (e.g. the word "Verified", not just a green dot).
- **NFR-A11y-3.** Buttons shall be reachable and operable by keyboard. (Inherited from Breeze + native HTML; no custom JS controls in v1.0.)

### 10.8 Browser compatibility

- **NFR-Browser-1.** Latest two major versions of Chrome, Safari, Firefox, and Edge on desktop.
- **NFR-Browser-2.** iOS Safari 16+ and Android Chrome (latest) for mobile.
- **NFR-Browser-3.** No support for IE / pre-Chromium Edge.

---

## 11. Technical stack (informative)

This section describes the chosen stack. The PRD does not mandate a specific stack — these are the choices captured in the reference build.

- **Backend:** Laravel 13, PHP 8.3+
- **Database:** SQLite (dev), MySQL or PostgreSQL (recommended for prod)
- **Frontend:** Blade + Tailwind CSS 4 (via Vite), Alpine.js for tiny interactions
- **Auth scaffolding:** Laravel Breeze (Blade preset)
- **Tests:** Pest with `RefreshDatabase`, `Storage::fake`, `Notification::fake`
- **CSV parsing:** `spatie/simple-excel` (lazy iterator)
- **CSV export:** native `fputcsv` + `response()->streamDownload()` (no library)
- **Audit:** custom lightweight `Audit` model (no third-party package); upgrade path to `owen-it/laravel-auditing` is open

---

## 12. Acceptance criteria

A v1.0 release shall meet **all** of the following:

- [ ] `php artisan migrate:fresh --seed` runs cleanly and populates 3 graduations + 50 students + 2 demo users.
- [ ] `php artisan test` shows **≥ 84 tests passing**, **≥ 209 assertions**, zero failures.
- [ ] `php artisan route:list --path=graduations` shows all expected routes (index, show, create, store, edit, update, destroy, nested students CRUD, verify, revoke, bulk, import, export).
- [ ] An admin (`admin@devhub.test` / `password`) can execute the full admin flow described in §9.1 and §9.3 end-to-end with no JS errors and no 500s.
- [ ] A student (`student@devhub.test` / `password`) can execute the student flow in §9.2 end-to-end and see their registration card on the graduation page they belong to.
- [ ] An attempt to access `/graduations` as a non-admin returns 403.
- [ ] An attempt to access another student's `/graduations/{g}/students/{s}/edit` as a non-admin returns 403.
- [ ] All URLs use UUIDs; no auto-increment ID appears in any URL.
- [ ] The dashboard renders in ≤ 3 SQL queries (verifiable with Telescope or Debugbar).
- [ ] Storage symlink is set up and uploaded receipts are reachable via `/storage/receipts/<file>`.
- [ ] An admin can verify a payment and the student receives an email (visible in `storage/logs/laravel.log` in local dev).
- [ ] After verifying, the Verify button is gone and a red Revoke button is shown.
- [ ] Revoking writes an `audits` row with `action = 'revoked'` and sends **no** email.
- [ ] A user registering with an email that matches multiple Student rows across multiple graduations gets linked to all of them, and **My registrations** lists them all.

---

## 13. Future considerations (post-v1.0)

These were explicitly excluded from v1.0 (§3.2) but are sensible next iterations:

1. **Spatie Laravel Permission** — replace the binary `is_admin` flag with roles + granular permissions stored in DB.
2. **Online payment gateway** (Billplz, FPX, Stripe) — kill the manual receipt upload step.
3. **Receipt OCR** — extract amount + reference number from uploaded receipts; pre-fill verification.
4. **Queues** — push `PaymentVerified` notifications and CSV imports onto a queue worker so HTTP responses stay snappy.
5. **API surface** (Sanctum tokens) — same endpoints exposed to a future mobile app.
6. **Multi-tenancy** — split graduations by faculty / campus.
7. **Granular permissions per faculty** — a Faculty A admin should not edit Faculty B's graduations.
8. **Audit middleware** — log every state-changing HTTP request, not just the four sensitive ones currently audited.
9. **Bulk revoke** — currently we deliberately don't ship this; if the workflow demands it, the same pattern as bulk verify applies.
10. **SSO** — integrate with the university IdP via SAML or OAuth.
11. **CSV import error report** — currently invalid rows are silently skipped with a count; future work could download a CSV of rejected rows with per-row reasons.
12. **Telescope / Pulse** — production observability for slow queries and failed jobs.

---

## 14. Open questions

None at this time. The v1.0 scope is closed and the reference repo is feature-complete against this PRD.

Should a new requirement emerge mid-build, the workflow is:
1. Add or amend the FR in this document with a new ID (do not renumber existing IDs).
2. Add the corresponding step in `material-guide/`.
3. Add the matching commit to the reference repo.
4. Add a test (or several) to lock in the new behaviour.

---

*End of document.*
