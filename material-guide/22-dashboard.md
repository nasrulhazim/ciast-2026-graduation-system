# 22 — Dashboard stats overview

> **Commit:** `7c82563` — *feat(dashboard): replace placeholder with stats overview*

## Why this step?

Breeze's `/dashboard` is empty: "You're logged in!" That's useless after step 1. We replace it with five summary cards (total graduations, total students, verified, pending, not paid) and a per-graduation breakdown table linking back to each show page.

Two things make this step interesting:

1. **Invokable controller** — the dashboard is a single page, so we promote it from inline route closure to a single-action controller with `__invoke()`. Cleaner; testable; can grow later.
2. **One query, four counts** — instead of N+1 (`Graduation::all()->each(fn $g => $g->students()->whereNotNull('verified_at')->count())`), we use `withCount()` with **constrained aliases** to compute all three derived counts in a single SQL query.

```php
->withCount([
    'students',
    'students as verified_count' => fn ($q) => $q->whereNotNull('verified_at'),
    'students as pending_count'  => fn ($q) => $q->whereNotNull('paid_at')->whereNull('verified_at'),
    'students as not_paid_count' => fn ($q) => $q->whereNull('paid_at'),
])
```

Each alias becomes a column on the returned Graduation rows. Zero extra queries.

## What you'll build

- `DashboardController` (invokable, single action)
- Updated route to use the controller class
- New dashboard view with summary cards + per-graduation table
- 1 test

## Prerequisites

- Completed [21 — Email notifications](./21-email-notifications.md).

## Steps

### 1. Generate the invokable controller

```bash
php artisan make:controller DashboardController --invokable
```

### 2. Fill in `DashboardController`

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Graduation;
use App\Models\Student;
use Illuminate\Http\Request;
use Illuminate\View\View;

class DashboardController extends Controller
{
    public function __invoke(Request $request): View
    {
        $totals = [
            'graduations' => Graduation::count(),
            'students' => Student::count(),
            'verified' => Student::whereNotNull('verified_at')->count(),
            'pending' => Student::whereNotNull('paid_at')->whereNull('verified_at')->count(),
            'not_paid' => Student::whereNull('paid_at')->count(),
        ];

        $graduations = Graduation::query()
            ->withCount([
                'students',
                'students as verified_count' => fn ($q) => $q->whereNotNull('verified_at'),
                'students as pending_count'  => fn ($q) => $q
                    ->whereNotNull('paid_at')
                    ->whereNull('verified_at'),
                'students as not_paid_count' => fn ($q) => $q->whereNull('paid_at'),
            ])
            ->latest()
            ->limit(10)
            ->get();

        return view('dashboard', compact('totals', 'graduations'));
    }
}
```

### 3. Update the dashboard route

In `routes/web.php`:

```php
use App\Http\Controllers\DashboardController;

// Old
Route::get('/dashboard', function () { return view('dashboard'); })
    ->middleware(['auth', 'verified'])->name('dashboard');

// New
Route::get('/dashboard', DashboardController::class)
    ->middleware(['auth', 'verified'])
    ->name('dashboard');
```

### 4. Replace `resources/views/dashboard.blade.php`

```blade
<x-app-layout>
    <x-slot name="header">
        <h2 class="font-semibold text-xl text-gray-800 leading-tight">Dashboard</h2>
    </x-slot>

    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8 space-y-8">

            <div class="grid grid-cols-2 md:grid-cols-5 gap-4">
                @php
                    $cards = [
                        ['label' => 'Graduations',  'value' => $totals['graduations'], 'class' => 'bg-indigo-100 text-indigo-700'],
                        ['label' => 'Students',     'value' => $totals['students'],    'class' => 'bg-gray-100 text-gray-700'],
                        ['label' => 'Verified',     'value' => $totals['verified'],    'class' => 'bg-green-100 text-green-700'],
                        ['label' => 'Pending',      'value' => $totals['pending'],     'class' => 'bg-amber-100 text-amber-700'],
                        ['label' => 'Not paid',     'value' => $totals['not_paid'],    'class' => 'bg-slate-100 text-slate-600'],
                    ];
                @endphp
                @foreach ($cards as $c)
                    <div class="rounded shadow p-6 {{ $c['class'] }}">
                        <div class="text-xs uppercase tracking-wide">{{ $c['label'] }}</div>
                        <div class="text-3xl font-semibold mt-2">{{ $c['value'] }}</div>
                    </div>
                @endforeach
            </div>

            <div class="bg-white shadow rounded">
                <div class="p-4 border-b font-semibold">Latest graduations</div>
                <table class="w-full text-sm">
                    <thead class="bg-gray-50">
                        <tr>
                            <th class="px-6 py-3 text-left">Title</th>
                            <th class="px-6 py-3 text-left">Students</th>
                            <th class="px-6 py-3 text-left">Verified</th>
                            <th class="px-6 py-3 text-left">Pending review</th>
                            <th class="px-6 py-3 text-left">Not paid</th>
                        </tr>
                    </thead>
                    <tbody>
                        @forelse ($graduations as $g)
                            <tr class="border-t">
                                <td class="px-6 py-3">
                                    <a href="{{ route('graduations.show', $g) }}"
                                       class="text-indigo-600">{{ $g->title }}</a>
                                </td>
                                <td class="px-6 py-3">{{ $g->students_count }}</td>
                                <td class="px-6 py-3 text-green-700">{{ $g->verified_count }}</td>
                                <td class="px-6 py-3 text-amber-700">{{ $g->pending_count }}</td>
                                <td class="px-6 py-3 text-slate-600">{{ $g->not_paid_count }}</td>
                            </tr>
                        @empty
                            <tr><td colspan="5" class="px-6 py-12 text-center text-gray-500">
                                No graduations yet.
                            </td></tr>
                        @endforelse
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</x-app-layout>
```

### 5. Add a test

```php
it('admin loads the dashboard with stats', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create(['title' => 'Test Convocation']);

    Student::factory()->for($grad)->verified()->create();
    Student::factory()->for($grad)->paidUnverified()->create();
    Student::factory()->for($grad)->create();

    $this->actingAs($admin)
        ->get(route('dashboard'))
        ->assertOk()
        ->assertSee('Test Convocation')
        ->assertSee('Verified')
        ->assertSee('Pending')
        ->assertSee('Not paid');
});
```

## Expected output

```
php artisan test
```

**69 tests / 166 assertions, all green.**

Browser:
1. Log in as admin → land on `/dashboard`.
2. See five coloured cards with counts that match `Student::count()` etc.
3. Scroll down → see "Latest graduations" table with four count columns per row.
4. Click a graduation title → land on its show page.

Profile the query count (e.g. with Telescope or Debugbar): the page should be **one query** for `users` (auth), one for the totals (5 small counts), and **one** for the graduations-with-counts. Three queries total, regardless of how many graduations exist.

## Commit your work

```bash
git add .
git commit -m "feat(dashboard): replace placeholder with stats overview"
```

## Common pitfalls

- **N+1 explosion** — symptoms: the `Latest graduations` row count goes up linearly with the SQL count. Cause: you wrote `$g->students()->verified()->count()` inside the Blade loop. Move that into a single `withCount(['students as verified_count' => ...])` at controller time.
- **`withCount` returns 0 everywhere** — the alias name must match the column you read in Blade. `verified_count` in controller ↔ `$g->verified_count` in view.
- **Dashboard isn't admin-only** — by design here. Students see counts too (their own dashboard). If you want admin-only, wrap the controller's `__invoke()` with `$this->authorize('admin-dashboard', User::class)` plus a gate definition.
- **Old closure route still wins** — Laravel uses the first matching route. If you didn't delete the old closure, the new controller route is dead. Replace, don't append.

## What's next

One last polish step: search the graduation index by title, the same way we search students: [23 — Graduation search](./23-graduation-search.md).
