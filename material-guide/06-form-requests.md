# 06 — Form Requests for graduation + student writes

> **Commit:** `454ff69` — *feat(http): add Form Requests for graduation + student writes*

## Why this step?

A controller's job is to **orchestrate**, not to validate. We push two concerns out of the controller and into Form Request classes:

1. **Validation rules** — `rules()` method
2. **Authorization** — `authorize()` method, which calls into our policies

This keeps controller actions to 5–10 lines: type-hint the Form Request, Laravel auto-validates and auto-authorizes before the action even runs.

Subtle but important: `StoreGraduationRequest` has `after:today` on `ceremony_date` (can't backdate a *new* event), but `UpdateGraduationRequest` *omits* it (you can edit metadata on a past event, e.g. fix a typo in the title).

## What you'll build

- `StoreGraduationRequest` — for `POST /graduations`
- `UpdateGraduationRequest` — for `PATCH /graduations/{graduation}`
- `UpdateStudentRequest` — for `PATCH /graduations/{graduation}/students/{student}`

(The Form Request stubs were already generated in step 04 by `make:model --requests`.)

## Prerequisites

- Completed [05 — Policies](./05-policies.md).

## Steps

### 1. `StoreGraduationRequest`

Replace `app/Http/Requests/StoreGraduationRequest.php`:

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use App\Models\Graduation;
use Illuminate\Foundation\Http\FormRequest;

class StoreGraduationRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Graduation::class);
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'ceremony_date' => ['required', 'date', 'after:today'],
            'fee' => ['required', 'numeric', 'min:0', 'max:9999.99'],
            'status' => ['required', 'in:draft,open,closed'],
        ];
    }
}
```

`authorize()` calls `GraduationPolicy::create()` indirectly via the `can('create', ...)` helper. If it returns false, Laravel responds with a 403 *before* `rules()` even runs.

### 2. `UpdateGraduationRequest`

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class UpdateGraduationRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('graduation'));
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'ceremony_date' => ['required', 'date'],     // no `after:today` here
            'fee' => ['required', 'numeric', 'min:0', 'max:9999.99'],
            'status' => ['required', 'in:draft,open,closed'],
        ];
    }
}
```

`$this->route('graduation')` returns the route-bound `Graduation` model (resolved via UUID). The policy gets the *actual* model, not just an ID.

### 3. `UpdateStudentRequest`

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class UpdateStudentRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('student'));
    }

    public function rules(): array
    {
        $studentId = $this->route('student')?->id;

        return [
            'name' => ['required', 'string', 'max:255'],
            'ic' => [
                'required', 'string', 'size:12',
                Rule::unique('students', 'ic')->ignore($studentId),
            ],
            'email' => [
                'required', 'email',
                Rule::unique('students', 'email')->ignore($studentId),
            ],
            'matric_card' => ['required', 'string', 'max:100'],
            'phone' => ['required', 'string', 'max:20'],
            'payment_receipt' => [
                'nullable', 'file', 'mimes:pdf,jpg,jpeg,png', 'max:2048',
            ],
        ];
    }
}
```

`Rule::unique(...)->ignore($studentId)` is the standard pattern for "must be unique except for the row I'm editing right now". Without `->ignore()`, updating a student's name would fail because their *current* IC would clash with their own row.

`payment_receipt` is `nullable` because the same form is reused for editing personal info without re-uploading the receipt every time.

### 4. (Optional) Delete the unused `StoreStudentRequest`

Step 4 generated `StoreStudentRequest` by default. The current scope (so far) doesn't expose a "create student via web" route — admins seed/import them. Leave the file alone for now; we'll wire it up in step 12.

## Expected output

There's nothing visible yet; the Form Requests aren't wired to controllers until step 07. You can verify they at least parse:

```bash
php artisan tinker
```

```php
>>> new App\Http\Requests\StoreGraduationRequest();
=> App\Http\Requests\StoreGraduationRequest { ... }
```

## Commit your work

```bash
git add .
git commit -m "feat(http): add Form Requests for graduation + student writes"
```

## Common pitfalls

- **`authorize()` returns `true` unconditionally** in the generated stub. If you forget to replace it, every write becomes public — anyone can create/update.
- **`->ignore($studentId)` is the model's `id`, not its `uuid`** — the unique constraint is on the DB primary key. Don't pass `$student->uuid` here; that won't match any row.
- **`after:today` parses string dates lazily** — if your input arrives as a `DateTime` object you may see weird parsing errors. Submitting an HTML `<input type="date">` value (e.g. `2026-06-15`) is fine.

## What's next

Time to connect everything: [07 — Controllers](./07-controllers.md).
