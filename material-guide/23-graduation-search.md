# 23 — Search the graduations index by title

> **Commit:** `b042599` — *feat(graduations): search the index page by title*

## Why this step?

We can search students within a graduation (step 14). The graduations index itself lacks the same control — once you have 30+ graduation events listed, scrolling becomes annoying. We add the smallest possible search: title `LIKE %term%`, plus `withQueryString()` so the term persists across pagination.

This is the final step. After this:

- 71 Pest tests pass
- The whole graduation flow works end-to-end
- The codebase mirrors the trainer's reference repo commit-for-commit

## What you'll build

- `GraduationController@index` honours `?search=`
- Compact GET search form above the table
- Clear link when search is active
- 2 tests

## Prerequisites

- Completed [22 — Dashboard](./22-dashboard.md).

## Steps

### 1. Update `GraduationController@index`

```php
use Illuminate\Http\Request;

public function index(Request $request): View
{
    $graduations = Graduation::query()
        ->withCount('students')
        ->when($request->filled('search'), function ($q) use ($request) {
            $term = '%' . $request->string('search')->trim() . '%';
            $q->where('title', 'like', $term);
        })
        ->latest()
        ->paginate(15)
        ->withQueryString();

    return view('graduations.index', compact('graduations'));
}
```

### 2. Add the search form in `graduations/index.blade.php`

Just below the header (or anywhere above the table):

```blade
<form method="GET" action="{{ route('graduations.index') }}"
      class="mb-4 flex gap-2">
    <input type="text" name="search" value="{{ request('search') }}"
           placeholder="Search by title…"
           class="rounded border-gray-300 text-sm w-full max-w-md">
    <button class="bg-slate-600 text-white px-3 py-2 rounded text-sm">Search</button>

    @if (request('search'))
        <a href="{{ route('graduations.index') }}"
           class="text-sm text-slate-600 self-center">Clear</a>
    @endif
</form>
```

### 3. Add two tests

```php
it('searches graduations by title fragment', function () {
    $user = User::factory()->create();

    Graduation::factory()->create(['title' => 'June 2026 Convocation']);
    Graduation::factory()->create(['title' => 'March 2027 Convocation']);

    $this->actingAs($user)
        ->get(route('graduations.index', ['search' => 'June']))
        ->assertOk()
        ->assertSee('June 2026 Convocation')
        ->assertDontSee('March 2027 Convocation');
});

it('empty search returns everything', function () {
    $user = User::factory()->create();

    Graduation::factory()->create(['title' => 'A']);
    Graduation::factory()->create(['title' => 'B']);

    $this->actingAs($user)
        ->get(route('graduations.index'))
        ->assertSee('A')
        ->assertSee('B');
});
```

## Expected output

```
php artisan test
```

**71 tests / 170 assertions, all green.** 🎉

Browser:
1. Visit `/graduations` with 5 seeded rows.
2. Type "June" → only rows with "June" in the title remain.
3. Click **Clear** → all rows return.
4. Pagination: search for a term that has 20+ matches → paginate; URL keeps `?search=…&page=2`.

## Commit your work

```bash
git add .
git commit -m "feat(graduations): search the index page by title"
```

## Common pitfalls

- **Empty-state from step 09 was generic** — it just said "No graduations yet." Update it to handle the zero-match case if you like:

```blade
@if (request('search'))
    No graduations match "{{ request('search') }}".
@else
    No graduations yet.
@endif
```

- **`withQueryString()` missing on the paginator** — pagination silently drops the search term.

---

## You're done

That's all 23 commits. To recap, you now have:

| Capability | Step |
|---|---|
| Laravel 13 + SQLite + MY locale | 01 |
| Breeze auth + Pest | 02 |
| UUID routing + admin role | 03 |
| Graduation + Student domain | 04 |
| Policies + Form Requests | 05–06 |
| Controllers + routes + views | 07–10 |
| Pest feature tests | 11 |
| Full student CRUD + CSV import | 12–13 |
| Searchable, filterable, sortable, paginated tables | 14–16 |
| Stream-friendly CSV export | 17 |
| Bulk actions with graduation-scoped guard | 18 |
| Self-service registration → user/student link | 19 |
| Audit log on sensitive actions | 20 |
| Email notification on verify | 21 |
| Dashboard with N+1-free counts | 22 |
| Graduation search | 23 |

**71 tests / 170 assertions.**

## What's next (after the course)

Some natural next steps the trainer would explore in production:

1. **Spatie Laravel Permission** — replace `is_admin` with proper roles + permissions.
2. **Telescope or Pulse** — observability for queries / slow endpoints / errors.
3. **Queues** — move email notifications and CSV imports onto a queue worker.
4. **API tokens with Sanctum** — expose the same endpoints to a future mobile app.
5. **Multi-tenancy** — split graduations by faculty/department.
6. **Receipt OCR** — extract amount + transaction ID from the receipt PDF automatically.

Pick one, scaffold it, ship it. You've got the muscle now.

🎓 *Congratulations.*
