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

Without this fix, importing a roster for the second graduation will falsely reject any IC / email / matric that already appears in the first graduation, because the validator is still checking *global* uniqueness against the `students` table.

Add `use Illuminate\Validation\Rule;` at the top of `StudentController` (if not already imported), then update the `Validator::make(...)` rules block inside `import()`:

```php
SimpleExcelReader::create($request->file('csv')->getRealPath(), 'csv')
    ->getRows()
    ->each(function (array $row) use ($graduation, &$imported, &$skipped) {
        $validator = Validator::make($row, [
            'name' => ['required', 'string', 'max:255'],
            'ic' => [
                'required', 'string', 'size:12',
                Rule::unique('students', 'ic')
                    ->where(fn ($q) => $q->where('graduation_id', $graduation->id)),
            ],
            'email' => [
                'required', 'email',
                Rule::unique('students', 'email')
                    ->where(fn ($q) => $q->where('graduation_id', $graduation->id)),
            ],
            'matric_card' => [
                'required', 'string', 'max:100',
                Rule::unique('students', 'matric_card')
                    ->where(fn ($q) => $q->where('graduation_id', $graduation->id)),
            ],
            'phone' => ['required', 'string', 'max:20'],
        ]);

        if ($validator->fails()) {
            $skipped++;

            return;
        }

        $graduation->students()->create($validator->validated());
        $imported++;
    });
```

The closure captures `$graduation` from the outer `import()` signature, so the same `graduation_id` is bound on every row.

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

The Livewire starter kit wraps every page in `<x-layouts::app>` (a Flux layout component), not the Breeze `<x-app-layout>` / `<x-slot name="header">` pattern. Match the convention used by `graduations/show.blade.php` and `students/show.blade.php`:

```blade
<x-layouts::app :title="__('My registrations')">
    <div class="flex h-full w-full flex-1 flex-col gap-4 rounded-xl">

        <div>
            <h1 class="text-lg font-semibold text-gray-900 dark:text-neutral-100">
                {{ __('My registrations') }}
            </h1>
            <p class="text-sm text-gray-500 dark:text-neutral-400">
                {{ $registrations->count() }} {{ Str::plural('registration', $registrations->count()) }} on file.
            </p>
        </div>

        @forelse ($registrations as $r)
            <div class="flex flex-col gap-3 rounded-xl border border-neutral-200 bg-white p-6 shadow-sm dark:border-neutral-700 dark:bg-neutral-900 sm:flex-row sm:items-center sm:justify-between">
                <div>
                    <div class="text-base font-semibold text-gray-900 dark:text-neutral-100">
                        {{ $r->graduation->title }}
                    </div>
                    <div class="text-sm text-gray-500 dark:text-neutral-400">
                        {{ $r->graduation->ceremony_date->format('d M Y') }}
                    </div>
                    <div class="mt-2">
                        @if ($r->isVerified())
                            <span class="inline-flex items-center rounded-full bg-emerald-50 px-2 py-0.5 text-xs font-medium text-emerald-700 ring-1 ring-emerald-600/20 dark:bg-emerald-500/10 dark:text-emerald-300 dark:ring-emerald-500/30">
                                Verified
                            </span>
                        @elseif ($r->hasPaid())
                            <span class="inline-flex items-center rounded-full bg-amber-50 px-2 py-0.5 text-xs font-medium text-amber-700 ring-1 ring-amber-600/20 dark:bg-amber-500/10 dark:text-amber-300 dark:ring-amber-500/30">
                                Pending review
                            </span>
                        @else
                            <span class="inline-flex items-center rounded-full bg-slate-50 px-2 py-0.5 text-xs font-medium text-slate-700 ring-1 ring-slate-600/20 dark:bg-slate-500/10 dark:text-slate-300 dark:ring-slate-500/30">
                                Not paid
                            </span>
                        @endif
                    </div>
                </div>

                <a href="{{ route('graduations.students.show', [$r->graduation, $r]) }}"
                    class="inline-flex items-center self-start rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700 sm:self-auto">
                    View / upload receipt
                </a>
            </div>
        @empty
            <p class="text-sm text-gray-600 dark:text-neutral-300">
                You have no registrations yet.
            </p>
        @endforelse

    </div>
</x-layouts::app>
```

### 9. Update the navigation link

The Livewire starter kit uses a Flux sidebar (`resources/views/layouts/app/sidebar.blade.php`), not the Breeze `layouts/navigation.blade.php` + `<x-nav-link>` pattern. The sidebar is responsive on its own (`collapsible="mobile"`) — there is no separate mobile nav file to mirror.

In `resources/views/layouts/app/sidebar.blade.php`, inside the `<flux:sidebar.group :heading="__('Platform')">` block, replace the "My registration" item from step 19 with:

```blade
@php
    $registrationCount = auth()->user()?->students()->count() ?? 0;
@endphp

@if ($registrationCount > 0)
    <flux:sidebar.item
        icon="identification"
        :href="route('my-registrations.index')"
        :current="request()->routeIs('my-registrations.*')"
        wire:navigate>
        {{ $registrationCount > 1 ? __('My registrations') : __('My registration') }}
    </flux:sidebar.item>
@endif
```

The label pluralises automatically with the count, and the item disappears entirely when the user has zero registrations.

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
