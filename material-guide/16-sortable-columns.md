# 16 — Sortable columns on the students table

> **Commit:** `9be038d` — *feat(students): sortable columns on students table*

## Why this step?

Click a column header → table sorts by that column. Click the same header again → toggles ascending/descending. Click a different column → resets to ascending.

The user-facing UX is plain, but the security detail behind it matters: **never trust `request('sort')` directly in your SQL**. We define a `SORTABLE_COLUMNS` whitelist on the controller; an unrecognised column silently falls back to `created_at` instead of throwing an SQL error or — worse — leaking schema info via an "unknown column 'foo'" message.

## What you'll build

- `?sort=name|ic|email|matric_card|created_at` + `?direction=asc|desc` query params
- A `SORTABLE_COLUMNS` whitelist on `GraduationController`
- Column headers (Name / IC / Matric) become toggle links with ▲/▼ arrow indicator
- Default remains `created_at desc` (newest first)
- 3 tests

## Prerequisites

- Completed [15 — Status filter](./15-status-filter.md).

## Steps

### 1. Add the whitelist + update `show()`

```php
class GraduationController extends Controller implements HasMiddleware
{
    private const SORTABLE_COLUMNS = ['name', 'ic', 'email', 'matric_card', 'created_at'];

    // ... middleware() ...

    public function show(Graduation $graduation, Request $request): View
    {
        $this->authorize('view', $graduation);

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

        return view('graduations.show', compact('graduation', 'students'));
    }
}
```

The `in_array($request->sort, ..., true)` is **strict comparison** (3rd arg). Without `true`, `0` would match a column named `'0'` (yes really, PHP is weird).

### 2. Build a sortable-header partial in the Blade view

In `graduations/show.blade.php`, define a small helper at the top:

```blade
@php
    $sort = request('sort', 'created_at');
    $direction = request('direction', 'desc');
    $sortLink = function (string $column, string $label) use ($sort, $direction) {
        $isActive = $sort === $column;
        $newDirection = $isActive && $direction === 'asc' ? 'desc' : 'asc';
        $arrow = $isActive ? ($direction === 'asc' ? '▲' : '▼') : '';
        $url = url()->current() . '?' . http_build_query(array_merge(
            request()->except(['sort', 'direction', 'page']),
            ['sort' => $column, 'direction' => $newDirection],
        ));
        return '<a href="' . $url . '" class="hover:underline">' . $label . ' ' . $arrow . '</a>';
    };
@endphp
```

Then use it in the `<thead>`:

```blade
<thead class="bg-gray-50">
    <tr>
        <th class="px-6 py-3 text-left">{!! $sortLink('name', 'Name') !!}</th>
        <th class="px-6 py-3 text-left">{!! $sortLink('ic', 'IC') !!}</th>
        <th class="px-6 py-3 text-left">{!! $sortLink('matric_card', 'Matric') !!}</th>
        <th class="px-6 py-3 text-left">Email</th>
        <th class="px-6 py-3 text-left">Status</th>
        <th class="px-6 py-3"></th>
    </tr>
</thead>
```

`{!! ... !!}` (unescaped output) is required because we built `<a>` tags inside the closure. The closure outputs only column names from a whitelist + arrow characters, so there's no XSS risk.

### 3. Add three tests

```php
it('sorts students by name ascending', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->create(['name' => 'Zoe Wong']);
    Student::factory()->for($grad)->create(['name' => 'Alice Tan']);

    $resp = $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'sort' => 'name', 'direction' => 'asc']))
        ->assertOk();

    expect(strpos($resp->getContent(), 'Alice Tan'))
        ->toBeLessThan(strpos($resp->getContent(), 'Zoe Wong'));
});

it('sorts by ic desc', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->create(['ic' => '111111111111']);
    Student::factory()->for($grad)->create(['ic' => '999999999999']);

    $resp = $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'sort' => 'ic', 'direction' => 'desc']));

    expect(strpos($resp->getContent(), '999999999999'))
        ->toBeLessThan(strpos($resp->getContent(), '111111111111'));
});

it('falls back to created_at for an unknown sort column', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->count(3)->for($grad)->create();

    $this->actingAs($admin)
        ->get(route('graduations.show', ['graduation' => $grad, 'sort' => 'pwned;DROP--', 'direction' => 'asc']))
        ->assertOk();
});
```

The third test is the security check. A malicious sort param should *not* throw, *not* leak an error, and *not* execute SQL.

## Expected output

```
php artisan test
```

**50 tests / 109 assertions, all green.**

Browser:
- Click "Name" header → URL becomes `?sort=name&direction=asc`, arrow points ▲.
- Click "Name" again → `direction=desc`, arrow flips ▼.
- Click "IC" → sort flips to IC asc; arrow moves off Name.
- Combine: sort `name asc`, filter `status=verified`, search `tan` → URL chains all three params.

## Commit your work

```bash
git add .
git commit -m "feat(students): sortable columns on students table"
```

## Common pitfalls

- **`in_array(..., true)` missing** — without strict mode, a `sort=0` request would match column `0`. Always use strict.
- **`{{ ... }}` escapes your `<a>` tag** — you'll see raw HTML in the header. Use `{!! ... !!}` *only* for trusted content (here it's a whitelist + arrow).
- **Pagination breaks the sort** — you forgot `withQueryString()`. Page 2 silently resets to default sort.
- **Mass-assignment of `direction`** — if you used `request('direction')` raw, `direction=DROP TABLE` could land in `orderBy()`. Always whitelist to `'asc'` or `'desc'`.

## What's next

We can browse a filtered, sorted, paginated list. Time to **export** the current filter to CSV: [17 — CSV export](./17-csv-export.md).
