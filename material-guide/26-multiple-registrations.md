# 26 — Support multiple student registrations per user

> **Commit:** `283d330` — *feat(users): support multiple student registrations per user*

## Why this step?

A real person might graduate **twice** — Bachelor in 2024, Masters in 2026. The same `User` account should own two `Student` rows, one per graduation. But our current schema enforces *global* uniqueness on `ic`, `email`, and `matric_card`. Their second registration fails with "IC already taken".

The fix is two-pronged:

1. **Schema:** drop the global unique indexes; add composite uniques scoped to `graduation_id`. Now you can have one `900101011234` per graduation, never two in the same one.
2. **Relations:** `User::student()` (HasOne) becomes `User::students()` (HasMany). The booted hook already does `UPDATE students SET user_id = ?`, which now happily updates *all* matching rows.

Plus a new page: `/my-registrations` lists every Student row linked to the authenticated user, one card per graduation.

While we're in there, we **stabilise the sort tests** from step 16. They were comparing `strpos()` of student names, but `fake('ms_MY')` admin names could accidentally embed a substring (e.g. "Aiman" appearing in an admin row before the table). We rename test fixtures to `TestAiman` / `TestZainal` / `Test-Admin` and switch to `assertSeeInOrder` which is order-precise.

## What you'll build

- Migration: drop global uniques on `ic`/`email`, add composite uniques scoped to `graduation_id` (also add one for `matric_card`)
- `User::student()` → `User::students()` (HasMany)
- All form-request unique rules rebuilt with `->where(fn ($q) => $q->where('graduation_id', $gradId))`
- CSV-import per-row validator updated the same way
- `MyRegistrationsController` (invokable) + `my-registrations.blade.php`
- Nav link: "My registration" → "My registrations" (pluralised when 2+, hidden when 0)
- Sort tests stabilised
- 3 new tests + a handful of updates

## Prerequisites

- Completed [25 — Restrict roster](./25-restrict-roster.md).

## Steps

### 1. Generate the migration

```bash
php artisan make:migration scope_student_uniqueness_per_graduation --table=students
```

### 2. Fill in the migration

```php
public function up(): void
{
    Schema::table('students', function (Blueprint $table) {
        // Drop the global unique indexes (Laravel's default name is {table}_{column}_unique)
        $table->dropUnique(['ic']);
        $table->dropUnique(['email']);

        // Add composite uniques scoped to graduation
        $table->unique(['graduation_id', 'ic']);
        $table->unique(['graduation_id', 'email']);
        $table->unique(['graduation_id', 'matric_card']);
    });
}

public function down(): void
{
    Schema::table('students', function (Blueprint $table) {
        $table->dropUnique(['graduation_id', 'ic']);
        $table->dropUnique(['graduation_id', 'email']);
        $table->dropUnique(['graduation_id', 'matric_card']);

        $table->unique('ic');
        $table->unique('email');
    });
}
```

Then:

```bash
php artisan migrate
```

> Note: if you've already seeded with the old constraints, run `php artisan migrate:fresh --seed` instead.

### 3. Convert `User::student()` to `User::students()`

In `app/Models/User.php`:

```php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function students(): HasMany
{
    return $this->hasMany(Student::class);
}
```

Delete the old `student()` HasOne. Find-replace `$user->student` across the codebase — most callers should now be `$user->students()->count()` or `$user->students->first()` depending on context.

The `booted()` hook stays the same; the SQL update was already plural-safe.

### 4. Rescope Form Request unique rules

`StoreStudentRequest` — make `ic`/`email`/`matric_card` unique *within the graduation*:

```php
public function rules(): array
{
    $gradId = $this->route('graduation')?->id;

    return [
        'name' => ['required', 'string', 'max:255'],
        'ic' => [
            'required', 'string', 'size:12',
            Rule::unique('students', 'ic')->where(fn ($q) => $q->where('graduation_id', $gradId)),
        ],
        'email' => [
            'required', 'email',
            Rule::unique('students', 'email')->where(fn ($q) => $q->where('graduation_id', $gradId)),
        ],
        'matric_card' => [
            'required', 'string', 'max:100',
            Rule::unique('students', 'matric_card')->where(fn ($q) => $q->where('graduation_id', $gradId)),
        ],
        'phone' => ['required', 'string', 'max:20'],
    ];
}
```

`UpdateStudentRequest` — same pattern, plus `->ignore($studentId)`:

```php
public function rules(): array
{
    $student = $this->route('student');
    $gradId = $student?->graduation_id;
    $studentId = $student?->id;

    return [
        'name' => ['required', 'string', 'max:255'],
        'ic' => [
            'required', 'string', 'size:12',
            Rule::unique('students', 'ic')
                ->where(fn ($q) => $q->where('graduation_id', $gradId))
                ->ignore($studentId),
        ],
        'email' => [
            'required', 'email',
            Rule::unique('students', 'email')
                ->where(fn ($q) => $q->where('graduation_id', $gradId))
                ->ignore($studentId),
        ],
        'matric_card' => [
            'required', 'string', 'max:100',
            Rule::unique('students', 'matric_card')
                ->where(fn ($q) => $q->where('graduation_id', $gradId))
                ->ignore($studentId),
        ],
        'phone' => ['required', 'string', 'max:20'],
        'payment_receipt' => ['nullable', 'file', 'mimes:pdf,jpg,jpeg,png', 'max:2048'],
    ];
}
```

### 5. Rescope the CSV-import per-row validator

In `StudentController@import`, update the `Validator::make(...)` rules block to use the same `->where(fn ($q) => $q->where('graduation_id', $graduation->id))` clause on `ic`, `email`, and `matric_card`. Otherwise importing a roster for the second graduation would falsely reject duplicates from the first.

### 6. Create `MyRegistrationsController`

```bash
php artisan make:controller MyRegistrationsController --invokable
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\View\View;

class MyRegistrationsController extends Controller
{
    public function __invoke(Request $request): View
    {
        $registrations = $request->user()->students()
            ->with('graduation')
            ->latest()
            ->get();

        return view('my-registrations', compact('registrations'));
    }
}
```

### 7. Add the route

In `routes/web.php`, inside the `auth` group:

```php
use App\Http\Controllers\MyRegistrationsController;

Route::get('/my-registrations', MyRegistrationsController::class)
    ->name('my-registrations.index');
```

### 8. Create `resources/views/my-registrations.blade.php`

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">My registrations</h2>
    </x-slot>

    <div class="py-12 max-w-4xl mx-auto sm:px-6 lg:px-8 space-y-4">
        @forelse ($registrations as $r)
            <div class="bg-white shadow rounded p-6 flex justify-between items-center">
                <div>
                    <div class="font-semibold">{{ $r->graduation->title }}</div>
                    <div class="text-sm text-gray-500">
                        {{ $r->graduation->ceremony_date->format('d M Y') }}
                    </div>
                    <div class="mt-2">
                        @if ($r->isVerified())
                            <span class="px-2 py-1 text-xs rounded bg-green-100 text-green-700">Verified</span>
                        @elseif ($r->hasPaid())
                            <span class="px-2 py-1 text-xs rounded bg-amber-100 text-amber-700">Pending review</span>
                        @else
                            <span class="px-2 py-1 text-xs rounded bg-slate-100 text-slate-600">Not paid</span>
                        @endif
                    </div>
                </div>
                <a href="{{ route('graduations.students.show', [$r->graduation, $r]) }}"
                   class="bg-indigo-600 text-white px-3 py-2 rounded text-sm">
                    View / upload receipt
                </a>
            </div>
        @empty
            <p class="text-gray-600">You have no registrations yet.</p>
        @endforelse
    </div>
</x-app-layout>
```

### 9. Update the navigation link

In `resources/views/layouts/navigation.blade.php`, replace the "My registration" link with:

```blade
@php
    $count = auth()->user()?->students()->count() ?? 0;
@endphp

@if ($count > 0)
    <x-nav-link :href="route('my-registrations.index')"
        :active="request()->routeIs('my-registrations.*')">
        {{ $count > 1 ? __('My registrations') : __('My registration') }}
    </x-nav-link>
@endif
```

Mirror the same block in the responsive nav. The link disappears entirely when the user has zero registrations.

### 10. Stabilise the sort tests

In `tests/Feature/GraduationFlowTest.php`, the sort tests from step 16 used names like `'Alice Tan'` and `'Zoe Wong'` plus `User::factory()->admin()->create()` with random `ms_MY` names. Substrings collide. Rename and switch to `assertSeeInOrder`:

```php
it('sorts students by name ascending', function () {
    $admin = User::factory()->admin()->create(['name' => 'Test-Admin']);
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->create(['name' => 'TestZainal']);
    Student::factory()->for($grad)->create(['name' => 'TestAiman']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'sort' => 'name', 'direction' => 'asc']))
        ->assertSeeInOrder(['TestAiman', 'TestZainal']);
});

it('sorts by ic desc', function () {
    $admin = User::factory()->admin()->create(['name' => 'Test-Admin']);
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->create(['ic' => '111111111111']);
    Student::factory()->for($grad)->create(['ic' => '999999999999']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'sort' => 'ic', 'direction' => 'desc']))
        ->assertSeeInOrder(['999999999999', '111111111111']);
});
```

### 11. Update tests that referenced `$user->student`

Any test calling `$user->student` becomes `$user->students()->first()` or `$user->students()->count()` depending on intent.

### 12. Add three new tests

```php
it('links a user to multiple student rows by email across graduations', function () {
    $gradA = Graduation::factory()->create();
    $gradB = Graduation::factory()->create();

    Student::factory()->for($gradA)->create(['email' => 'multi@x.test', 'user_id' => null]);
    Student::factory()->for($gradB)->create(['email' => 'multi@x.test', 'user_id' => null]);

    $this->post(route('register'), [
        'name' => 'Multi',
        'email' => 'multi@x.test',
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);

    $user = User::where('email', 'multi@x.test')->first();
    expect($user->students()->count())->toBe(2);
});

it('my-registrations page lists all linked registrations', function () {
    $user = User::factory()->create();
    $gradA = Graduation::factory()->create(['title' => 'June 2026 Convocation']);
    $gradB = Graduation::factory()->create(['title' => 'December 2028 Convocation']);

    Student::factory()->for($gradA)->create(['user_id' => $user->id]);
    Student::factory()->for($gradB)->create(['user_id' => $user->id]);

    $this->actingAs($user)
        ->get(route('my-registrations.index'))
        ->assertOk()
        ->assertSee('June 2026 Convocation')
        ->assertSee('December 2028 Convocation');
});

it('my-registrations page shows empty state when none', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->get(route('my-registrations.index'))
        ->assertOk()
        ->assertSee('You have no registrations yet.');
});
```

## Expected output

```
php artisan test
```

**83 tests / 209 assertions, all green.**

Browser:
1. Manually add a Student to two different graduations with the same email + IC + matric. Save both.
2. Register a User with that email → log in.
3. Nav shows **My registrations** (plural).
4. Click it → page lists both registrations, each with its own status badge and **View / upload receipt** button.

## Commit your work

```bash
git add .
git commit -m "feat(users): support multiple student registrations per user"
```

## Common pitfalls

- **`SQLSTATE[42000]: ... no such index`** when dropping uniques on SQLite — SQLite is picky about index names. If `$table->dropUnique(['ic'])` doesn't work, use the literal name: `$table->dropUnique('students_ic_unique')`.
- **`students_graduation_id_ic_unique` is the new index name** — useful to know if you have to debug constraint violations.
- **Find-replace missed a `$user->student`** somewhere — the tests will tell you. If the test passes silently but the prod code throws, you have a coverage gap.
- **Stale dev DB after migration** — symptom: importing a CSV with a duplicate IC from a different graduation rejects it. Cause: dev DB still has the old global unique. Run `php artisan migrate:fresh --seed`.
- **`$count = auth()->user()?->students()->count()`** issues a query on every page render. For a small app it's fine; if you'd rather cache it, eager-load the relationship in a middleware.

## What's next

Students still see `/graduations` and can navigate to any graduation's show page. That's not how this app should work — students should reach graduations *only* through their registrations. Tighten authorization: [27 — Students reach graduations only via My Registrations](./27-students-via-my-registrations.md).
