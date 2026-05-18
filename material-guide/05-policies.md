# 05 — Graduation + Student policies

> **Commit:** `8e7f8ce` — *feat(authz): add Graduation + Student policies*

## Why this step?

Authorization rules belong in **one place per model**. Spreading `if ($user->isAdmin())` across controllers, Form Requests, and Blade views means every new rule is a hunt-and-pick job. Policies solve that:

- One method per ability (`viewAny`, `create`, `update`, `delete`, plus custom like `verify`)
- Called from Form Requests, controllers, and Blade with the same `$user->can('verify', $student)` API
- Auto-discovered by Laravel 13 when models live in `App\Models\*` and policies in `App\Policies\*` — zero registration

Two interesting rules here:

- **Graduation can only be deleted if no student has paid yet.** Financial-record guard — once money has moved, the row must be archivable (soft delete) only after manual review. We encode the guard in the policy, not in the controller.
- **`verify` requires a payment receipt.** Admins can't approve a payment that doesn't exist.

## What you'll build

- `GraduationPolicy` with `viewAny`, `view`, `create`, `update`, `delete`, `restore`
- `StudentPolicy` with `viewAny`, `view`, `create`, `update`, `delete`, `verify`

## Prerequisites

- Completed [04 — Domain models](./04-domain-models.md).

## Steps

### 1. Generate the policy files

```bash
php artisan make:policy GraduationPolicy --model=Graduation
php artisan make:policy StudentPolicy --model=Student
```

This creates `app/Policies/GraduationPolicy.php` and `app/Policies/StudentPolicy.php` with method stubs for each ability.

### 2. Fill in `GraduationPolicy`

```php
<?php

declare(strict_types=1);

namespace App\Policies;

use App\Models\Graduation;
use App\Models\User;

class GraduationPolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(User $user, Graduation $graduation): bool
    {
        return true;
    }

    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    public function update(User $user, Graduation $graduation): bool
    {
        return $user->isAdmin();
    }

    public function delete(User $user, Graduation $graduation): bool
    {
        if (! $user->isAdmin()) {
            return false;
        }

        return $graduation->students()
            ->whereNotNull('paid_at')
            ->doesntExist();
    }

    public function restore(User $user, Graduation $graduation): bool
    {
        return $user->isAdmin();
    }
}
```

Why `viewAny` and `view` return `true`? In this app the *list* of graduations is browsable by any authenticated user. Students need to see what's coming up. The *create/edit/delete* abilities are admin-only.

### 3. Fill in `StudentPolicy`

```php
<?php

declare(strict_types=1);

namespace App\Policies;

use App\Models\Student;
use App\Models\User;

class StudentPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->isAdmin();
    }

    public function view(User $user, Student $student): bool
    {
        return $user->isAdmin() || $student->user_id === $user->id;
    }

    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    public function update(User $user, Student $student): bool
    {
        return $user->isAdmin() || $student->user_id === $user->id;
    }

    public function delete(User $user, Student $student): bool
    {
        return $user->isAdmin();
    }

    public function verify(User $user, Student $student): bool
    {
        return $user->isAdmin() && $student->payment_receipt !== null;
    }
}
```

The `view` and `update` rules use **the same pattern**: admin can act on anyone, a student can act on their own record (`user_id === user->id`). This is the foundation of the self-service flow we wire up in step 19.

### 4. Confirm auto-discovery

Laravel 13 auto-registers policies when both:

- The Model lives in `App\Models\*`
- The Policy lives in `App\Policies\*` and is named `{Model}Policy`

We meet both conditions. No `Gate::policy()` call needed.

## Expected output

Spin up tinker and test the gates manually:

```bash
php artisan tinker
```

```php
>>> $admin = User::where('email', 'admin@devhub.test')->first();
>>> $student = User::where('email', 'student@devhub.test')->first();
>>> $grad = Graduation::first();
>>> $studentRow = Student::first();

>>> $admin->can('update', $grad);          // true
>>> $student->can('update', $grad);        // false
>>> $admin->can('verify', $studentRow);    // depends on receipt presence
>>> $admin->can('delete', $grad);          // true ONLY if no $grad->students have paid_at
```

## Commit your work

```bash
git add .
git commit -m "feat(authz): add Graduation + Student policies"
```

## Common pitfalls

- **Policy methods return `null` silently** — if you forget the `bool` return type or return nothing, Laravel treats `null` as deny. Always return an explicit `true`/`false`.
- **Auto-discovery silently fails** if your model is namespaced under e.g. `App\Domain\Graduation`. Either move the model back to `App\Models` or register the policy manually in a service provider.
- **`verify` allows null receipts** — re-read the rule: `&& $student->payment_receipt !== null`. Without that clause, an admin could mark a non-paid student verified.

## What's next

Policies are in place. Now we plug them into Form Requests so validation *and* authorization happen before our controllers run: [06 — Form Requests](./06-form-requests.md).
