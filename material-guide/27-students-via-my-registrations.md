# 27 — Students reach graduations only via My Registrations

> **Commit:** `955da3f` — *feat(authz): students reach graduations only via My Registrations*

## Why this step?

Combined with step 25, this gives the app a clean two-track UX:

- **Admins** use the admin tools — Graduations list, search, roster, bulk actions, export.
- **Students** stick to **My registrations** and their own student show page for receipt upload.

Right now a curious (or malicious) student can:

- Visit `/graduations` and see the full list of upcoming events
- Open `/graduations/{uuid}` for any graduation, even ones they're not registered for

Step 25 hid the *roster* from them, but the graduation details (title, date, fee) were still browseable. We tighten further:

- `GraduationPolicy::viewAny` becomes admin-only
- `GraduationPolicy::view` requires the user to have a linked Student row scoped to that graduation
- The "Graduations" nav link disappears for non-admins

A student URL-typing their way to `/graduations` or `/graduations/{someone-elses-uuid}` now hits **403**, not a sanitised-but-still-visible admin shell.

## What you'll build

- `GraduationPolicy::viewAny` admin-only
- `GraduationPolicy::view` admin OR linked-student
- Nav link hidden for non-admins (`@can('viewAny', Graduation::class)`)
- Test updates: split "lists graduations for authenticated users" into admin happy path + non-admin 403; the contact-registrar test becomes a 403 test; graduation search tests switch to admin actors

## Prerequisites

- Completed [26 — Multiple registrations](./26-multiple-registrations.md).

## Steps

### 1. Tighten `GraduationPolicy`

In `app/Policies/GraduationPolicy.php`:

```php
public function viewAny(User $user): bool
{
    return $user->isAdmin();
}

public function view(User $user, Graduation $graduation): bool
{
    if ($user->isAdmin()) {
        return true;
    }

    return $graduation->students()
        ->where('user_id', $user->id)
        ->exists();
}
```

`create`, `update`, `delete`, `restore` already return `$user->isAdmin()` — leave them.

### 2. Hide the Graduations nav link for non-admins

In `resources/views/layouts/navigation.blade.php`, both the desktop and mobile blocks:

```blade
@can('viewAny', App\Models\Graduation::class)
    <x-nav-link :href="route('graduations.index')" :active="request()->routeIs('graduations.*')">
        {{ __('Graduations') }}
    </x-nav-link>
@endcan
```

The link silently disappears for students. They navigate by **Dashboard** + **My registrations**.

### 3. Update tests that assume non-admins can list graduations

In `tests/Feature/GraduationFlowTest.php`:

- Split the existing **lists graduations for authenticated users** test:

```php
it('admin can list graduations', function () {
    $admin = User::factory()->admin()->create();
    Graduation::factory()->count(3)->create();

    $this->actingAs($admin)
        ->get(route('graduations.index'))
        ->assertOk();
});

it('non-admin is forbidden from listing graduations', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->get(route('graduations.index'))
        ->assertForbidden();
});
```

- Step 25's **non-admin sees graduation details but not roster** must now seed a linked Student row first, because otherwise the user can't `view` the graduation at all:

```php
it('non-admin with a registration sees their own card but no roster', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();
    Student::factory()->for($grad)->create([
        'user_id' => $user->id,
        'name' => 'Mine Tan',
    ]);

    $resp = $this->actingAs($user)
        ->get(route('graduations.show', $grad))
        ->assertOk();

    expect($resp->getContent())
        ->toContain('Mine Tan')
        ->not->toContain('Import CSV');
});
```

- Step 25's **non-admin without a registration sees the contact-registrar note** is no longer reachable — the view returns 403 before the note could render. Convert to a forbidden test:

```php
it('non-admin without a registration is forbidden from viewing a graduation', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();

    $this->actingAs($user)
        ->get(route('graduations.show', $grad))
        ->assertForbidden();
});
```

- Step 23's two graduation search tests need an **admin** actor now (they were using a non-admin User to verify the search worked):

```php
it('searches graduations by title fragment', function () {
    $admin = User::factory()->admin()->create();

    Graduation::factory()->create(['title' => 'June 2026 Convocation']);
    Graduation::factory()->create(['title' => 'March 2027 Convocation']);

    $this->actingAs($admin)
        ->get(route('graduations.index', ['search' => 'June']))
        ->assertOk()
        ->assertSee('June 2026 Convocation')
        ->assertDontSee('March 2027 Convocation');
});

it('empty search returns everything', function () {
    $admin = User::factory()->admin()->create();

    Graduation::factory()->create(['title' => 'A']);
    Graduation::factory()->create(['title' => 'B']);

    $this->actingAs($admin)
        ->get(route('graduations.index'))
        ->assertSee('A')
        ->assertSee('B');
});
```

### 4. (Optional) Remove the now-unreachable contact-registrar branch

In `graduations/show.blade.php`, the `@cannot('viewAny', Student::class)` block from step 25 had an `@else` that rendered "Contact the registrar to be added". That branch is now unreachable because non-registered students hit a 403 before the view renders. You can either:

- Leave it as defense-in-depth (the view stays consistent if a future change loosens the policy)
- Or simplify by removing the `@else` — your call

The trainer's reference repo keeps it. Defensive HTML rarely hurts.

## Expected output

```
php artisan test
```

**84 tests / 209 assertions, all green.**

Browser:
1. Log in as `student@devhub.test`:
   - **Graduations** nav link is gone.
   - Visit `/graduations` directly → 403.
   - Visit `/graduations/{some-uuid}` you're not registered for → 403.
   - Visit `/graduations/{uuid-you-ARE-registered-for}` → 200, you see the details + your registration card.
2. Log in as admin:
   - Everything works as before.

## Commit your work

```bash
git add .
git commit -m "feat(authz): students reach graduations only via My Registrations"
```

## Common pitfalls

- **Student suddenly can't even see their own graduation** — your `view` policy is missing the linked-student check. Re-read step 1.
- **403 on the dashboard latest-graduations table** — your dashboard template links to `route('graduations.show', $g)`. That's fine for admins (the dashboard is admin-only territory). If you exposed the dashboard to students, wrap that link in `@can('view', $g)`.
- **`my-registrations` page errors** — students click "View / upload receipt" → 403 on the graduation show page even though they ARE registered. Cause: the link is to `graduations.students.show`, not `graduations.show`. Re-check the link target — `view` on `Student` is independent of `view` on `Graduation`.
- **Old tests fail** with "Expected 200, got 403" — that's the point of this step. Convert them as shown in step 3.

---

## You're done — for now

That's all 27 commits. The two-track UX is complete:

| Role | What they see |
|---|---|
| **Admin** | Full Graduations list, search, roster, bulk actions, export, dashboard, audit log |
| **Student** | Dashboard + My registrations + their own student show pages |

**84 tests / 209 assertions.**

The journey from `laravel new` to here:

1. Foundation (01–03) — Laravel + Breeze + roles
2. Domain (04–06) — models, policies, form requests
3. HTTP (07–10) — controllers, routes, views, storage
4. Tests (11) — pin the behaviour
5. Student management (12–13) — CRUD + CSV import
6. Table UX (14–16) — search, filter, sort
7. Export & bulk (17–18) — streaming CSV, bulk verify/delete
8. Self-service + audit + notify (19–21)
9. Dashboard + graduation search (22–23)
10. Verify revoke + privacy hardening + multi-registrations + two-track UX (24–27)

If new commits land in the reference repo, follow the same template:
1. Get the commit message with full body
2. Identify the **Why** (often called out in the commit's opening paragraph)
3. Identify the **What you'll build** (mid-paragraph)
4. Pull the code snippets from the commit's diff (if no body example exists)
5. Pull the **Expected output** from the commit's "Full suite: X tests / Y assertions" footer
6. Add **Common pitfalls** from real questions you've heard in training

The format is repeatable. Future-you (or a co-trainer) can drop in step 28 in 15 minutes.

🎓 *Now truly done.*
