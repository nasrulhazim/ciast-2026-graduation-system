# 11 — Pest feature tests for the graduation flow

> **Commit:** `2dedd89` — *test: add Pest feature tests for graduation flow*

## Why this step?

The app works in the browser — but every future commit risks regressing something. We write **feature tests** (HTTP-level, not unit) that exercise the policy surface and the happy path.

Nine tests, two minutes of effort each, that pay rent for the next year. They cover:

- Authentication boundary (guests vs. logged-in users)
- Authorization rules (admin vs. student vs. non-owner)
- URL convention (UUID-only, never bigint)
- File upload happy path
- Verification gating (no receipt → forbidden)

After this step the suite reaches **34 tests / 76 assertions, all green**.

## What you'll build

A new file `tests/Feature/GraduationFlowTest.php` with 9 Pest tests using `RefreshDatabase`, `Storage::fake('public')`, and `UploadedFile::fake()`.

## Prerequisites

- Completed [10 — Storage symlink](./10-storage-symlink.md).

## Steps

### 1. Create `tests/Feature/GraduationFlowTest.php`

```php
<?php

declare(strict_types=1);

use App\Models\Graduation;
use App\Models\Student;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

uses(RefreshDatabase::class);

it('redirects guests away from the index', function () {
    $this->get(route('graduations.index'))
        ->assertRedirect(route('login'));
});

it('lists graduations for authenticated users', function () {
    $user = User::factory()->create();
    Graduation::factory()->count(3)->create();

    $this->actingAs($user)
        ->get(route('graduations.index'))
        ->assertOk();
});

it('denies non-admin from creating a graduation', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('graduations.store'), [
            'title' => 'Test',
            'ceremony_date' => now()->addMonth()->format('Y-m-d'),
            'fee' => 250,
            'status' => 'open',
        ])
        ->assertForbidden();
});

it('admin can create a graduation', function () {
    $admin = User::factory()->admin()->create();

    $this->actingAs($admin)
        ->post(route('graduations.store'), [
            'title' => 'Test Convocation',
            'ceremony_date' => now()->addMonth()->format('Y-m-d'),
            'fee' => 300.00,
            'status' => 'open',
        ])
        ->assertRedirect();

    expect(Graduation::where('title', 'Test Convocation')->exists())->toBeTrue();
});

it('resolves graduations by uuid in URLs', function () {
    Graduation::factory()->create();
    $grad = Graduation::first();

    expect(route('graduations.show', $grad))
        ->toContain($grad->uuid)
        ->not->toContain("/{$grad->id}");
});

it('admin can upload a receipt for any student', function () {
    Storage::fake('public');

    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.update', [$grad, $student]), [
            'name' => $student->name,
            'ic' => $student->ic,
            'email' => $student->email,
            'matric_card' => $student->matric_card,
            'phone' => $student->phone,
            'payment_receipt' => UploadedFile::fake()->create('receipt.pdf', 100, 'application/pdf'),
        ])
        ->assertRedirect();

    expect($student->fresh()->payment_receipt)->not->toBeNull();
});

it('student can only update their own record', function () {
    $studentUser = User::factory()->create();
    $grad = Graduation::factory()->create();

    $mine = Student::factory()->for($grad)->create(['user_id' => $studentUser->id]);
    $other = Student::factory()->for($grad)->create();

    $this->actingAs($studentUser)
        ->get(route('graduations.students.edit', [$grad, $mine]))
        ->assertOk();

    $this->actingAs($studentUser)
        ->get(route('graduations.students.edit', [$grad, $other]))
        ->assertForbidden();
});

it('admin verifies a paid student', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->paidUnverified()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]))
        ->assertRedirect();

    expect($student->fresh()->verified_at)->not->toBeNull();
});

it('admin cannot verify a student with no receipt', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();   // no receipt

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]))
        ->assertForbidden();
});
```

### 2. Confirm Pest's `RefreshDatabase` is wired in `tests/Pest.php`

Open `tests/Pest.php` and confirm Feature tests get `RefreshDatabase` by default:

```php
uses(
    Tests\TestCase::class,
    Illuminate\Foundation\Testing\RefreshDatabase::class,
)->in('Feature');
```

If your file doesn't have that, add it. Otherwise your tests will dirty the dev SQLite file.

### 3. Run the suite

```bash
php artisan test
```

## Expected output

```
   PASS  Tests\Feature\GraduationFlowTest
  ✓ it redirects guests away from the index
  ✓ it lists graduations for authenticated users
  ✓ it denies non-admin from creating a graduation
  ✓ it admin can create a graduation
  ✓ it resolves graduations by uuid in URLs
  ✓ it admin can upload a receipt for any student
  ✓ it student can only update their own record
  ✓ it admin verifies a paid student
  ✓ it admin cannot verify a student with no receipt

  Tests:    34 passed (76 assertions)
```

## Commit your work

```bash
git add .
git commit -m "test: add Pest feature tests for graduation flow"
```

## Common pitfalls

- **Tests pollute your dev SQLite file** — symptom: `php artisan tinker` shows test rows after running tests. Cause: `RefreshDatabase` not wired in `tests/Pest.php` for the `Feature` directory. See step 2.
- **`Storage::fake('public')` not called** — symptom: `tests/storage/app/public/receipts/*.pdf` files start piling up. `Storage::fake()` keeps the test in memory.
- **`assertForbidden()` fails with 419** — your form submission was rejected for CSRF, not for authorization. Pest's HTTP helper handles CSRF automatically; if you see 419, it's likely a session middleware issue, not an auth one.
- **Cannot create a `Graduation` with `after:today` rule failing** — your test uses `now()->subDay()` for `ceremony_date`. The `StoreGraduationRequest` requires future dates.

## What's next

The core build is complete (everything in `build-spec.md`). The next 12 commits add real-world features the spec hints at as "production upgrade path". First up: **full Student CRUD** — let admins add and delete students through the UI: [12 — Student add/delete](./12-student-add-delete.md).
