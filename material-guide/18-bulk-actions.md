# 18 — Bulk verify + delete from the graduation page

> **Commit:** `f37d996` — *feat(students): bulk verify + delete from graduation page*

## Why this step?

The admin filters to "pending review", sees 50 rows, all with valid receipts — they want to verify all 50 in one click. Same for "delete all spam registrations". We add a **bulk action bar**:

- Checkbox column on each row (admin only)
- "Select all" header checkbox
- Action `<select>` (Verify payments / Delete)
- Apply button → POSTs the chosen action + selected UUIDs

Two non-obvious design details:

1. **HTML5 `form=` attribute, not nested `<form>`s.** A `<form>` inside a `<table>` cell breaks HTML semantics. Instead we render the bulk-action form outside the table, give it `id="bulk-form"`, and the row checkboxes reference it with `form="bulk-form"`. The per-row Remove buttons get the same treatment via `form="row-delete-{$id}"`.
2. **Cross-graduation ID leak guard.** The bulk endpoint must scope to `$graduation->students()->whereIn('uuid', $ids)`. If we `whereIn` against all `students` without the graduation scope, an admin could delete a student from a *different* graduation by passing their UUID.

The verify branch reports `Verified X. Skipped Y (no receipt)` so admins know what was unactionable. Delete just reports `Removed X.`.

## What you'll build

- `BulkStudentActionRequest` validating `action: in:verify,delete` + `ids: required array of uuid`
- `StudentController@bulk` that scopes by graduation, dispatches to verify/delete branches
- Bulk action bar UI above the table, gated by `@can('viewAny', Student::class)`
- 5 tests

## Prerequisites

- Completed [17 — CSV export](./17-csv-export.md).

## Steps

### 1. Create `BulkStudentActionRequest`

```bash
php artisan make:request BulkStudentActionRequest
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use App\Models\Student;
use Illuminate\Foundation\Http\FormRequest;

class BulkStudentActionRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('viewAny', Student::class);  // admin-only proxy
    }

    public function rules(): array
    {
        return [
            'action' => ['required', 'in:verify,delete'],
            'ids' => ['required', 'array', 'min:1'],
            'ids.*' => ['uuid'],
        ];
    }
}
```

### 2. Register the bulk route

In `routes/web.php`, before the resource line:

```php
Route::post(
    'graduations/{graduation}/students/bulk',
    [StudentController::class, 'bulk']
)->name('graduations.students.bulk');
```

### 3. Add `bulk()` to `StudentController`

```php
use App\Http\Requests\BulkStudentActionRequest;

public function bulk(BulkStudentActionRequest $request, Graduation $graduation): RedirectResponse
{
    $ids = $request->validated()['ids'];
    $action = $request->validated()['action'];

    // Scope to THIS graduation — silently drops cross-graduation ids
    $scope = $graduation->students()->whereIn('uuid', $ids);

    if ($action === 'verify') {
        $toVerify = (clone $scope)->whereNotNull('payment_receipt')->get();
        $skipped = (clone $scope)->whereNull('payment_receipt')->count();

        foreach ($toVerify as $student) {
            $student->update(['verified_at' => now()]);
        }

        return redirect()
            ->route('graduations.show', $graduation)
            ->with('status', "Verified {$toVerify->count()}. Skipped {$skipped} (no receipt).");
    }

    // delete branch
    $removed = (clone $scope)->count();
    $scope->delete();

    return redirect()
        ->route('graduations.show', $graduation)
        ->with('status', "Removed {$removed}.");
}
```

### 4. Add the bulk action bar in `graduations/show.blade.php`

Render the bulk form *outside* the table (e.g. just above it), so the row checkboxes can reference it via `form=`:

```blade
@can('viewAny', App\Models\Student::class)
    <form method="POST"
          action="{{ route('graduations.students.bulk', $graduation) }}"
          id="bulk-form"
          onsubmit="
              if (document.querySelectorAll('input[name=&quot;ids[]&quot;]:checked').length === 0) {
                  alert('Select at least one student.');
                  return false;
              }
          "
          class="flex gap-2 items-center mb-4">
        @csrf
        <select name="action" class="rounded border-gray-300 text-sm">
            <option value="verify">Verify payments</option>
            <option value="delete">Delete</option>
        </select>
        <button class="bg-indigo-600 text-white px-3 py-2 rounded text-sm">Apply to selected</button>
    </form>
@endcan
```

> ⚠️ **Don't scope the selector to `#bulk-form`.** The row checkboxes live in the `<table>`, *not* inside the form — they only *reference* it via the HTML5 `form="bulk-form"` attribute. A CSS descendant selector like `#bulk-form input[name="ids[]"]:checked` matches zero rows and the "Select at least one student." alert fires every time. Either query by name alone (as above) or by the `form=` attribute: `input[form="bulk-form"][name="ids[]"]:checked`.

Inside the table `<thead>`, prepend a checkbox column:

```blade
<th class="px-6 py-3 text-left">
    @can('viewAny', App\Models\Student::class)
        <input type="checkbox"
               onchange="
                   document.querySelectorAll('input[name=\'ids[]\']').forEach(cb => cb.checked = this.checked)
               ">
    @endcan
</th>
```

Inside each `<tr>`:

```blade
<td class="px-6 py-4">
    @can('viewAny', App\Models\Student::class)
        <input type="checkbox"
               form="bulk-form"
               name="ids[]"
               value="{{ $student->uuid }}">
    @endcan
</td>
```

For the per-row Remove button, render the delete form before the table and reference it:

```blade
{{-- Above the table --}}
@foreach ($students as $student)
    @can('delete', $student)
        <form method="POST"
              action="{{ route('graduations.students.destroy', [$graduation, $student]) }}"
              id="row-delete-{{ $student->id }}"
              onsubmit="return confirm('Remove {{ $student->name }}?')">
            @csrf @method('DELETE')
        </form>
    @endcan
@endforeach

{{-- Inside the row --}}
<td class="px-6 py-4 text-right">
    {{-- view link --}}
    @can('delete', $student)
        <button form="row-delete-{{ $student->id }}"
                class="text-red-600">Remove</button>
    @endcan
</td>
```

### 5. Add five tests

```php
it('admin can bulk verify students with receipts', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    $a = Student::factory()->for($grad)->paidUnverified()->create();
    $b = Student::factory()->for($grad)->paidUnverified()->create();

    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'verify',
            'ids' => [$a->uuid, $b->uuid],
        ])
        ->assertRedirect();

    expect($a->fresh()->verified_at)->not->toBeNull();
    expect($b->fresh()->verified_at)->not->toBeNull();
});

it('bulk verify skips students without a receipt', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    $paid = Student::factory()->for($grad)->paidUnverified()->create();
    $unpaid = Student::factory()->for($grad)->create();   // no receipt

    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'verify',
            'ids' => [$paid->uuid, $unpaid->uuid],
        ])
        ->assertRedirect()
        ->assertSessionHas('status', 'Verified 1. Skipped 1 (no receipt).');

    expect($paid->fresh()->verified_at)->not->toBeNull();
    expect($unpaid->fresh()->verified_at)->toBeNull();
});

it('admin can bulk delete students', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $students = Student::factory()->count(3)->for($grad)->create();

    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'delete',
            'ids' => $students->pluck('uuid')->all(),
        ])
        ->assertRedirect();

    expect($grad->students()->count())->toBe(0);
});

it('non-admin cannot bulk verify', function () {
    $user = User::factory()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->paidUnverified()->create();

    $this->actingAs($user)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'verify',
            'ids' => [$student->uuid],
        ])
        ->assertForbidden();
});

it('bulk action does not leak across graduations', function () {
    $admin = User::factory()->admin()->create();
    $gradA = Graduation::factory()->create();
    $gradB = Graduation::factory()->create();

    $studentInB = Student::factory()->for($gradB)->create();

    // Try to delete a B-student via the A bulk endpoint
    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $gradA), [
            'action' => 'delete',
            'ids' => [$studentInB->uuid],
        ])
        ->assertRedirect();

    expect(Student::find($studentInB->id))->not->toBeNull();   // still there
});
```

## Expected output

```
php artisan test
```

**58 tests / 134 assertions, all green.**

Browser:
1. Tick 3 rows → action "Verify payments" → Apply → flash: `Verified 3. Skipped 0 (no receipt).`
2. Tick rows with no receipt → action "Verify payments" → flash reports the skip count.
3. Tick rows → action "Delete" → Apply → rows disappear.
4. Submit empty selection → alert: "Select at least one student."

## Commit your work

```bash
git add .
git commit -m "feat(students): bulk verify + delete from graduation page"
```

## Common pitfalls

- **Nested `<form>` tags** — browsers silently break the inner form. Use HTML5 `form=` attribute as shown above.
- **No cross-graduation scoping** — without `$graduation->students()->whereIn(...)`, an admin on graduation A can delete a student from graduation B. The test on line 5 (above) catches it; **don't skip the test**.
- **`->delete()` on a relation builder** — `$scope->delete()` works on a query builder. Sending an array to `Student::destroy([...])` would also work but bypass model events. Either is fine here; pick the one your team prefers.
- **`onchange` "select all" doesn't toggle row checkboxes** — your row checkboxes are inside `<form id="bulk-form">` reference. The vanilla JS query selector `document.querySelectorAll('input[name="ids[]"]')` only matches if `name="ids[]"` is actually on the inputs, not just on the form.
- **"Select at least one student." fires even though boxes are ticked** — same root cause as the previous bullet but in the submit handler. If the selector is `#bulk-form input[name="ids[]"]:checked`, the `#bulk-form` parent scope is wrong: the row checkboxes are inside the `<table>`, *not* inside the form. They only *reference* the form via HTML5 `form=`, which the descendant combinator does not cross. Drop `#bulk-form` and select by name (or by `[form="bulk-form"]` attribute).

## What's next

So far, only admins manage students. Time for self-service: a student registers, and their User account auto-links to the Student row admins pre-uploaded: [19 — Self-service registration](./19-self-service-registration.md).
