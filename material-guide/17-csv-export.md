# 17 — Export students to CSV

> **Commit:** `529b145` — *feat(students): export students to CSV*

## Why this step?

Admins want a spreadsheet of "pending review" students to chase up via email. They can already filter the table to that subset — we just need an **Export CSV** button that respects the current filter.

Two design choices worth attention:

1. **Stream the CSV instead of building it in memory.** `response()->streamDownload()` + `fputcsv()` writes to `php://output` row-by-row. Combined with `->lazy()` on the Eloquent query, memory stays flat even for 10 000 students.
2. **Use native `fputcsv` for export, not `spatie/simple-excel`.** Simple Excel is great for parsing; for streaming output, `fputcsv` is one line, no library, no buffering.

The filename slugifies the graduation title and stamps a timestamp:

```
june-2026-convocation-students-20260518-231542.csv
```

## What you'll build

- Route: `GET /graduations/{graduation}/students/export`, **admin-only**, registered before the resource route
- `StudentController@export` — streams a CSV honouring `?search` + `?status`
- **Export CSV** button on the graduation show page
- 3 tests

## Prerequisites

- Completed [16 — Sortable columns](./16-sortable-columns.md).

## Steps

### 1. Register the export route **before** the resource route

```php
Route::get(
    'graduations/{graduation}/students/export',
    [StudentController::class, 'export']
)->name('graduations.students.export');

Route::post(
    'graduations/{graduation}/students/import',
    [StudentController::class, 'import']
)->name('graduations.students.import');

Route::resource('graduations.students', StudentController::class);
```

### 2. Add `export()` to `StudentController`

```php
use Symfony\Component\HttpFoundation\StreamedResponse;
use Illuminate\Support\Str;

public function export(Request $request, Graduation $graduation): StreamedResponse
{
    $this->authorize('viewAny', Student::class);

    $query = $graduation->students()
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
            fn ($q) => $q->whereNull('paid_at'));

    $filename = Str::slug($graduation->title) . '-students-' . now()->format('Ymd-His') . '.csv';

    return response()->streamDownload(function () use ($query) {
        $out = fopen('php://output', 'w');
        fputcsv($out, ['name', 'ic', 'email', 'matric_card', 'phone', 'paid_at', 'verified_at']);

        $query->lazy()->each(function (Student $student) use ($out) {
            fputcsv($out, [
                $student->name,
                $student->ic,
                $student->email,
                $student->matric_card,
                $student->phone,
                $student->paid_at?->toIso8601String(),
                $student->verified_at?->toIso8601String(),
            ]);
        });

        fclose($out);
    }, $filename, [
        'Content-Type' => 'text/csv; charset=UTF-8',
    ]);
}
```

`lazy()` is the magic word. It iterates DB rows in chunks of 1000 (configurable), keeping memory flat regardless of total row count.

### 3. Add the Export button on `graduations/show.blade.php`

Near the search form / pill filters:

```blade
@can('viewAny', App\Models\Student::class)
    <a href="{{ route('graduations.students.export',
            array_filter([
                'graduation' => $graduation,
                'search' => request('search'),
                'status' => request('status'),
            ])) }}"
       class="bg-green-600 text-white px-3 py-2 rounded text-sm">
        Export CSV
    </a>
@endcan
```

`array_filter()` strips empty values so the URL stays clean.

### 4. Add three tests

```php
it('admin can export students to CSV', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create(['title' => 'June 2026 Convocation']);
    Student::factory()->count(2)->for($grad)->create();

    $resp = $this->actingAs($admin)
        ->get(route('graduations.students.export', $grad))
        ->assertOk();

    expect($resp->headers->get('Content-Type'))->toContain('text/csv');
    expect($resp->streamedContent())->toContain('name,ic,email,matric_card,phone,paid_at,verified_at');
});

it('export honours status filter', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    Student::factory()->for($grad)->verified()->create(['name' => 'Done Tan']);
    Student::factory()->for($grad)->create(['name' => 'Empty Ng']);

    $resp = $this->actingAs($admin)
        ->get(route('graduations.students.export',
            ['graduation' => $grad, 'status' => 'verified']))
        ->assertOk();

    $body = $resp->streamedContent();
    expect($body)->toContain('Done Tan')->not->toContain('Empty Ng');
});

it('non-admin cannot export', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();

    $this->actingAs($user)
        ->get(route('graduations.students.export', $grad))
        ->assertForbidden();
});
```

## Expected output

```
php artisan test
```

**53 tests / 119 assertions, all green.**

Browser:
1. Visit a graduation as admin → click **Export CSV** → file downloads.
2. Open the file in Numbers / Excel — 7 columns, headers in row 1.
3. Filter to "Pending review" → click Export → only those rows download.

## Commit your work

```bash
git add .
git commit -m "feat(students): export students to CSV"
```

## Common pitfalls

- **Memory blow-up on big exports** — you used `->get()` instead of `->lazy()`. With 50 000 rows, you'll OOM.
- **Download dialog never opens / CSV is empty** — you forgot `fclose($out)` or the callback wasn't returned. The streamed response only flushes when the callback returns.
- **`Content-Type` is `text/html`** — pass the header explicitly as shown; without it some browsers try to display the CSV as HTML.
- **Route collision** — same trap as the import route in step 13. Register `/students/export` *before* the resource line.

## What's next

We've made the table powerful. Time to make it batchable too — bulk verify and bulk delete from the same page: [18 — Bulk actions](./18-bulk-actions.md).
