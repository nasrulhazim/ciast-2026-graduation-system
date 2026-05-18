# 15 — Filter students by payment status

> **Commit:** `ea92444` — *feat(students): filter students table by payment status*

## Why this step?

"Show me who hasn't paid" and "show me who's waiting for verification" are the two most common admin questions. Right now they have to scroll through every row looking at badges. We add a `?status=` query param backed by a row of coloured pill links.

Three statuses, mapped to three SQL filters:

| Status | URL param | SQL |
|---|---|---|
| Verified | `?status=verified` | `whereNotNull(verified_at)` |
| Pending review | `?status=pending` | `whereNotNull(paid_at)->whereNull(verified_at)` |
| Not paid | `?status=not_paid` | `whereNull(paid_at)` |

Both filters compose — `?search=ali&status=pending` returns students named Ali whose payment is waiting for verification.

## What you'll build

- `GraduationController@show` honours `?status=`
- Row of pill links above the table (All / Verified / Pending / Not paid)
- Hidden `<input name="status">` in the search form so the two filters compose
- Clear link shows when *either* filter is active
- 3 tests (one per status)

## Prerequisites

- Completed [14 — Search + pagination](./14-search-pagination.md).

## Steps

### 1. Update `GraduationController@show`

```php
public function show(Graduation $graduation, Request $request): View
{
    $this->authorize('view', $graduation);

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
        ->latest()
        ->paginate(15)
        ->withQueryString();

    return view('graduations.show', compact('graduation', 'students'));
}
```

### 2. Add the pill row above the table

```blade
@php
    $statuses = [
        '' => ['label' => 'All', 'class' => 'bg-slate-100 text-slate-700'],
        'verified' => ['label' => 'Verified', 'class' => 'bg-green-100 text-green-700'],
        'pending' => ['label' => 'Pending review', 'class' => 'bg-amber-100 text-amber-700'],
        'not_paid' => ['label' => 'Not paid', 'class' => 'bg-slate-100 text-slate-600'],
    ];
    $current = request('status', '');
@endphp

<div class="flex gap-2 mb-4">
    @foreach ($statuses as $value => $cfg)
        <a href="{{ route('graduations.show', array_filter([
                'graduation' => $graduation,
                'status' => $value ?: null,
                'search' => request('search'),
           ])) }}"
           class="px-3 py-1 text-xs rounded {{ $cfg['class'] }}
                  {{ $current === $value ? 'ring-2 ring-indigo-500' : '' }}">
            {{ $cfg['label'] }}
        </a>
    @endforeach
</div>
```

### 3. Add the hidden status field to the search form

```blade
<form method="GET" action="{{ route('graduations.show', $graduation) }}"
      class="flex gap-2 flex-1">
    <input type="hidden" name="status" value="{{ request('status') }}">
    <input type="text" name="search" value="{{ request('search') }}" ...>
    {{-- ... --}}
</form>
```

### 4. Update the "Clear" link to clear *both* filters

```blade
@if (request('search') || request('status'))
    <a href="{{ route('graduations.show', $graduation) }}"
       class="text-sm text-slate-600 self-center">Clear</a>
@endif
```

### 5. Add three tests

```php
it('filters by verified status', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->verified()->create(['name' => 'Done Tan']);
    Student::factory()->for($grad)->paidUnverified()->create(['name' => 'Wait Lim']);
    Student::factory()->for($grad)->create(['name' => 'Empty Ng']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'status' => 'verified']))
        ->assertSee('Done Tan')
        ->assertDontSee('Wait Lim')
        ->assertDontSee('Empty Ng');
});

it('filters by pending status', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->verified()->create(['name' => 'Done']);
    Student::factory()->for($grad)->paidUnverified()->create(['name' => 'Wait']);
    Student::factory()->for($grad)->create(['name' => 'Empty']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'status' => 'pending']))
        ->assertDontSee('Done')
        ->assertSee('Wait')
        ->assertDontSee('Empty');
});

it('filters by not_paid status', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->verified()->create(['name' => 'Done']);
    Student::factory()->for($grad)->paidUnverified()->create(['name' => 'Wait']);
    Student::factory()->for($grad)->create(['name' => 'Empty']);

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'status' => 'not_paid']))
        ->assertDontSee('Done')
        ->assertDontSee('Wait')
        ->assertSee('Empty');
});
```

## Expected output

```
php artisan test
```

**47 tests / 102 assertions, all green.**

Browser: click each pill in turn and confirm:
- **All** → 20 rows visible
- **Verified** → 10 rows
- **Pending review** → 4 rows
- **Not paid** → 6 rows

Now type a name in search → results still respect the status filter. Click **Clear** → both reset.

## Commit your work

```bash
git add .
git commit -m "feat(students): filter students table by payment status"
```

## Common pitfalls

- **Pending shows verified rows too** — your SQL is `whereNotNull(paid_at)` alone. You must also `whereNull(verified_at)`.
- **Status query string gets lost on search submit** — the hidden `<input name="status">` is missing from the search form.
- **Pill links don't preserve search term** — `array_filter()` is the trick; it drops empty values from the query string but keeps your `search` term if present.

## What's next

We can filter. Last table-UX feature: sorting. Click a column header to toggle direction: [16 — Sortable columns](./16-sortable-columns.md).
