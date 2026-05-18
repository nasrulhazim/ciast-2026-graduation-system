# 25 — Restrict roster + admin tools to admins on graduation show

> **Commit:** `60f5ea0` — *feat(students): restrict roster + admin tools to admins on graduation show*

## Why this step?

**This is a privacy bug.** Up to step 24 the graduation show page renders the full students table — names, IC numbers, emails, matric cards — to *any* authenticated user. A student looking up "what time is the ceremony?" can see all 200 of their batch-mates' personal data.

We fix it in two layers:

1. **The admin block disappears entirely for non-admins.** Wrap the import card, the search/filter/sort/bulk action bar, the roster table, and the pagination in `@can('viewAny', Student::class)`. Don't even build the paginated query for non-admins.
2. **Show non-admins a focused alternative.** If they're registered for this graduation, render a small "Your registration" card with their own name, matric, payment-status badge, and a button to upload their receipt. If they're not registered, render a one-line note: "Contact the registrar to be added to this graduation."

This is also the first time the controller branches by user role — and that's deliberate. Roles drive different *queries*, not just different *views*.

## What you'll build

- `GraduationController@show` short-circuits the paginated student query for non-admins
- The whole admin operations block wrapped in `@can('viewAny', Student::class)`
- Non-admin: their own student card (or contact-registrar note) under the graduation details
- 4 new tests

## Prerequisites

- Completed [24 — Verify + revoke](./24-verify-revoke.md).

## Steps

### 1. Short-circuit the query for non-admins

In `GraduationController@show`:

```php
public function show(Graduation $graduation, Request $request): View
{
    $this->authorize('view', $graduation);

    $isAdmin = $request->user()->isAdmin();
    $students = null;
    $ownStudent = null;

    if ($isAdmin) {
        $sort = in_array($request->sort, self::SORTABLE_COLUMNS, true)
            ? $request->sort
            : 'created_at';
        $direction = $request->direction === 'asc' ? 'asc' : 'desc';

        $students = $graduation->students()
            ->when($request->filled('search'), function ($q) use ($request) {
                $term = '%' . $request->string('search')->trim() . '%';
                $q->where(function ($inner) use ($term) {
                    $inner->where('name', 'like', $term)
                        ->orWhere('ic', 'like', $term)
                        ->orWhere('email', 'like', $term)
                        ->orWhere('matric_card', 'like', $term);
                });
            })
            ->when($request->status === 'verified',
                fn ($q) => $q->whereNotNull('verified_at'))
            ->when($request->status === 'pending',
                fn ($q) => $q->whereNotNull('paid_at')->whereNull('verified_at'))
            ->when($request->status === 'not_paid',
                fn ($q) => $q->whereNull('paid_at'))
            ->orderBy($sort, $direction)
            ->paginate(15)
            ->withQueryString();
    } else {
        $ownStudent = $graduation->students()
            ->where('user_id', $request->user()->id)
            ->first();
    }

    return view('graduations.show', compact('graduation', 'students', 'ownStudent'));
}
```

No paginated query for non-admins. They get exactly one row (or none) — `ownStudent`.

### 2. Restructure `graduations/show.blade.php`

Wrap the entire admin block in `@can('viewAny', Student::class)`:

```blade
@can('viewAny', App\Models\Student::class)
    {{-- Total students count, CSV import card, search/filter/sort bar, bulk action bar, --}}
    {{-- the roster table, pagination — everything admin-facing — lives here. --}}
@endcan
```

Below that, add the non-admin block:

```blade
@cannot('viewAny', App\Models\Student::class)
    @if ($ownStudent)
        <div class="bg-white shadow rounded p-6 mt-6">
            <h3 class="font-semibold mb-4">Your registration</h3>
            <dl class="grid grid-cols-2 gap-4 text-sm">
                <dt class="text-gray-500">Name</dt>
                <dd>{{ $ownStudent->name }}</dd>

                <dt class="text-gray-500">Matric</dt>
                <dd>{{ $ownStudent->matric_card }}</dd>

                <dt class="text-gray-500">Status</dt>
                <dd>
                    @if ($ownStudent->isVerified())
                        <span class="px-2 py-1 text-xs rounded bg-green-100 text-green-700">Verified</span>
                    @elseif ($ownStudent->hasPaid())
                        <span class="px-2 py-1 text-xs rounded bg-amber-100 text-amber-700">Pending review</span>
                    @else
                        <span class="px-2 py-1 text-xs rounded bg-slate-100 text-slate-600">Not paid</span>
                    @endif
                </dd>
            </dl>

            <a href="{{ route('graduations.students.show', [$graduation, $ownStudent]) }}"
               class="inline-block mt-4 bg-indigo-600 text-white px-3 py-2 rounded text-sm">
                View / upload receipt
            </a>
        </div>
    @else
        <p class="mt-6 text-sm text-gray-600">
            You are not on the roster for this graduation. Please contact the registrar to be added.
        </p>
    @endif
@endcannot
```

### 3. Add four tests

```php
it('non-admin sees no roster or admin tools on graduation show', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();
    Student::factory()->count(3)->for($grad)->create(['ic' => fake()->numerify('############')]);

    $resp = $this->actingAs($user)
        ->get(route('graduations.show', $grad))
        ->assertOk();

    expect($resp->getContent())
        ->not->toContain('Import CSV')
        ->not->toContain('Export CSV')
        ->not->toContain('Apply to selected');
});

it('non-admin with a registration sees their own card', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();
    Student::factory()->for($grad)->create([
        'user_id' => $user->id,
        'name' => 'Mine Tan',
        'matric_card' => 'M9999',
    ]);

    $this->actingAs($user)
        ->get(route('graduations.show', $grad))
        ->assertOk()
        ->assertSee('Your registration')
        ->assertSee('Mine Tan')
        ->assertSee('M9999');
});

it('non-admin without a registration sees the contact-registrar note', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();

    $this->actingAs($user)
        ->get(route('graduations.show', $grad))
        ->assertOk()
        ->assertSee('contact the registrar');
});

it('admin still sees the full roster and tools', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    Student::factory()->count(2)->for($grad)->create();

    $this->actingAs($admin)
        ->get(route('graduations.show', $grad))
        ->assertOk()
        ->assertSee('Import CSV')
        ->assertSee('Export CSV');
});
```

## Expected output

```
php artisan test
```

**80 tests / 199 assertions, all green.**

Browser:
1. Log in as admin → graduation page is unchanged (full table, all tools).
2. Log in as a registered student → graduation page shows just the graduation details + "Your registration" card with a "View / upload receipt" button.
3. Log in as an unregistered user → just the graduation details + "Contact the registrar" note.

## Commit your work

```bash
git add .
git commit -m "feat(students): restrict roster + admin tools to admins on graduation show"
```

## Common pitfalls

- **`$students` is `null` in the admin partial** — you forgot the `if ($isAdmin)` branch. The variable still has to be passed to the view (we compact `null`), but Blade should never access it outside `@can('viewAny', Student::class)`.
- **`$ownStudent` is `null` and the page errors on `$ownStudent->isVerified()`** — wrap with `@if ($ownStudent)` *before* the dl. Don't access properties on null.
- **Pagination disappears for admins too** — you accidentally moved the pagination outside the admin block. Re-read the Blade structure.
- **Privacy regression slips back** — write the failing test (non-admin shouldn't see "Apply to selected" anywhere in the body) *before* you fix the view. The test is your guard rail.

## What's next

A real person might attend multiple convocations (degree, then masters). Right now our schema enforces global unique IC/email — that breaks the second registration. Fix the data model: [26 — Multiple registrations per user](./26-multiple-registrations.md).
