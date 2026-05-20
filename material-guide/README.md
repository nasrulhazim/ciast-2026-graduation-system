# Graduation System — Training Material Guide

> **Audience:** Training participants (Laravel beginner → intermediate).
> **Goal:** Build a full **Graduation Registration System** with Laravel 13, step by step, by replaying the same 27 commits the trainer made.
> **Estimated duration:** 2 days at a comfortable pace; 6 – 7 hours if you race.

---

## What you'll build

A university convocation registration system where:

- **Admins** create graduation events, upload student rosters, verify payments, and audit changes.
- **Students** self-register, fill in their info, upload a payment receipt, and get an email when their payment is verified.

By the last step you will have built:

- Authentication via the [Livewire starter kit](https://laravel.com/docs/13.x/starter-kits#livewire) (Fortify + Flux UI + Pest)
- Role-based authorization via Policies + Form Requests
- Full CRUD with UUID routing and `foreignIdFor` foreign keys
- CSV bulk import + export
- Searchable, filterable, sortable, paginated tables
- Bulk actions (verify / delete)
- Self-service flow that links a `User` to a `Student` row on registration
- Audit log for sensitive admin actions
- Email notifications on payment verification
- A stats dashboard with a single non-N+1 query
- Verify / revoke flow with mutual-exclusion policies
- Roster privacy: students never see other students' data
- Multiple registrations per user (degree this year, masters two years later)
- Two-track UX: admins drive Graduations; students drive My Registrations
- 84 Pest tests across the whole flow

---

## How to use this guide

Each numbered file maps to **exactly one git commit** in the trainer's reference repo. Work through them in order.

For every step:

1. Read the **Why** — understand the motivation before touching code.
2. Read the **What you'll build** — set expectations.
3. Follow the **Steps** — type the code; don't copy-paste blindly.
4. Run the **Expected output** verification — confirm it works before moving on.
5. Commit your own work with the same conventional-commit prefix shown at the top of the file (e.g. `feat(students): ...`). This builds the habit you'll use in real projects.

> **Tip:** If you get stuck at step *N*, you can compare your code against the trainer's reference repo at commit `<sha>` (shown at the top of each file).

---

## Steps

### Setup

- [00 — Prerequisites](./00-prerequisites.md) — install PHP 8.3+, Composer, Node, your editor.

### Phase 1 — Foundation

- [01 — Laravel 13 scaffold with SQLite + MY locale](./01-laravel-scaffold.md)
- [02 — Livewire starter kit tour (Fortify + Flux UI + Pest)](./02-livewire-starter-kit.md)
- [03 — UUID + `is_admin` role on users](./03-user-roles-uuid.md)

### Phase 2 — Domain & Authorization

- [04 — Graduation + Student models, migrations, factories, seeder](./04-domain-models.md)
- [05 — Graduation + Student policies](./05-policies.md)
- [06 — Form Requests for graduation + student writes](./06-form-requests.md)

### Phase 3 — HTTP Layer

- [07 — Graduation + Student controllers](./07-controllers.md)
- [08 — Resource routes (and a nested resource)](./08-routes.md)
- [09 — Blade views + nav link](./09-blade-views.md)
- [10 — Public storage symlink](./10-storage-symlink.md)

### Phase 4 — Tests

- [11 — Pest feature tests for the graduation flow](./11-pest-tests.md)

### Phase 5 — Student management upgrades

- [12 — Full CRUD: admin can add + delete students](./12-student-add-delete.md)
- [13 — Admin can bulk-import students from CSV](./13-csv-import.md)

### Phase 6 — Table UX

- [14 — Searchable + paginated students table](./14-search-pagination.md)
- [15 — Filter students by payment status](./15-status-filter.md)
- [16 — Sortable columns on the students table](./16-sortable-columns.md)

### Phase 7 — Bulk operations & export

- [17 — Export students to CSV](./17-csv-export.md)
- [18 — Bulk verify + delete from the graduation page](./18-bulk-actions.md)

### Phase 8 — Self-service, audit, notifications

- [19 — Link a User to a Student by email on registration](./19-self-service-registration.md)
- [20 — Track who verified / created / deleted students](./20-audit-log.md)
- [21 — Email the student when payment is verified](./21-email-notifications.md)

### Phase 9 — Dashboard & polish

- [22 — Dashboard stats overview](./22-dashboard.md)
- [23 — Search the graduations index by title](./23-graduation-search.md)

### Phase 10 — Workflow refinement & privacy hardening

- [24 — Hide verify once done + admin can revoke](./24-verify-revoke.md)
- [25 — Restrict roster + admin tools to admins on graduation show](./25-restrict-roster.md)
- [26 — Support multiple student registrations per user](./26-multiple-registrations.md)
- [27 — Students reach graduations only via My Registrations](./27-students-via-my-registrations.md)

---

## Conventions used throughout

These are non-negotiable across every step:

| Rule | Why |
|---|---|
| Strict types (`declare(strict_types=1);`) on every PHP file | Catches type bugs at parse time, not in production |
| Hybrid UUID — `id` for joins, `uuid` for URLs | Keeps query performance while hiding incremental IDs from users |
| `foreignIdFor(Model::class)` over `foreignId('model_id')` | Auto-renames if the model namespace moves |
| Soft deletes on `Graduation`, hard delete on `Student` | Graduation = financial record (audit value), Student = expendable row |
| Form Requests own validation **and** `authorize()` | Controllers stay 5–10 lines |
| Policies, not inline `if`s | One source of truth for "can this user do X" |
| `@can('action', $model)` in Blade | Hide UI affordances that would 403 anyway |
| `ms_MY` Faker locale | Realistic Malaysian seed data |

---

## Reference repository

The trainer's reference repo sits beside this guide at `../graduation-system/`. After each step you can run:

```bash
git log --oneline --reverse
```

…to see the trainer's commit at the same step.

---

**Ready?** Start with [00 — Prerequisites](./00-prerequisites.md).
