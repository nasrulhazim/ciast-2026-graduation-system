# 07 — Graduation + Student controllers

> **Commit:** `c39d305` — *feat(http): add Graduation + Student controllers*

## Why this step?

Controllers are wired now that policies and Form Requests are in place. Two patterns we use everywhere from here on:

1. **HasMiddleware (Laravel 13)** — middleware is declared via a static method on the controller instead of in route files. Cleaner; the controller carries its own access requirements.
2. **Defense in depth** — write actions type-hint a Form Request (which authorizes), but read/destroy actions also call `$this->authorize(...)` directly. If someone misconfigures the route or skips the Form Request later, the controller still checks.

The `Student` update action also handles file uploads (payment receipt) and stamps `paid_at` automatically when a file is provided.

## What you'll build

- `GraduationController` — full resource controller (index, create, store, show, edit, update, destroy)
- `StudentController` — show, edit, update, plus a custom `verify` action
- Base `Controller` updated to include `AuthorizesRequests` (so `$this->authorize()` works)

## Prerequisites

- Completed [06 — Form Requests](./06-form-requests.md).

## Steps

### 1. Enable `AuthorizesRequests` on the base controller

Open `app/Http/Controllers/Controller.php`:

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use Illuminate\Foundation\Auth\Access\AuthorizesRequests;

abstract class Controller
{
    use AuthorizesRequests;
}
```

This gives every child controller the `$this->authorize('ability', $model)` helper.

### 2. `GraduationController`

Replace `app/Http/Controllers/GraduationController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\StoreGraduationRequest;
use App\Http\Requests\UpdateGraduationRequest;
use App\Models\Graduation;
use Illuminate\Http\RedirectResponse;
use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;
use Illuminate\View\View;

class GraduationController extends Controller implements HasMiddleware
{
    public static function middleware(): array
    {
        return [
            new Middleware('auth'),
            new Middleware('can:viewAny,App\\Models\\Graduation', only: ['index']),
        ];
    }

    public function index(): View
    {
        $graduations = Graduation::query()
            ->withCount('students')
            ->latest()
            ->paginate(15);

        return view('graduations.index', compact('graduations'));
    }

    public function create(): View
    {
        $this->authorize('create', Graduation::class);
        return view('graduations.create');
    }

    public function store(StoreGraduationRequest $request): RedirectResponse
    {
        $graduation = Graduation::create($request->validated());

        return redirect()
            ->route('graduations.show', $graduation)
            ->with('status', 'Graduation created.');
    }

    public function show(Graduation $graduation): View
    {
        $this->authorize('view', $graduation);
        $graduation->load('students');

        return view('graduations.show', compact('graduation'));
    }

    public function edit(Graduation $graduation): View
    {
        $this->authorize('update', $graduation);
        return view('graduations.edit', compact('graduation'));
    }

    public function update(UpdateGraduationRequest $request, Graduation $graduation): RedirectResponse
    {
        $graduation->update($request->validated());

        return redirect()
            ->route('graduations.show', $graduation)
            ->with('status', 'Graduation updated.');
    }

    public function destroy(Graduation $graduation): RedirectResponse
    {
        $this->authorize('delete', $graduation);
        $graduation->delete();

        return redirect()
            ->route('graduations.index')
            ->with('status', 'Graduation archived.');
    }
}
```

Notice `index()` doesn't call `$this->authorize()` because the middleware already did (`can:viewAny,App\Models\Graduation`). For `create`, `show`, `edit`, `destroy` we call `authorize()` *because* there's no equivalent middleware on those routes.

### 3. `StudentController`

Replace `app/Http/Controllers/StudentController.php`:

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Http\Requests\UpdateStudentRequest;
use App\Models\Graduation;
use App\Models\Student;
use Illuminate\Http\RedirectResponse;
use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;
use Illuminate\View\View;

class StudentController extends Controller implements HasMiddleware
{
    public static function middleware(): array
    {
        return [new Middleware('auth')];
    }

    public function show(Graduation $graduation, Student $student): View
    {
        $this->authorize('view', $student);

        return view('students.show', compact('graduation', 'student'));
    }

    public function edit(Graduation $graduation, Student $student): View
    {
        $this->authorize('update', $student);

        return view('students.edit', compact('graduation', 'student'));
    }

    public function update(
        UpdateStudentRequest $request,
        Graduation $graduation,
        Student $student
    ): RedirectResponse {
        $data = $request->validated();

        if ($request->hasFile('payment_receipt')) {
            $data['payment_receipt'] = $request->file('payment_receipt')
                ->store('receipts', 'public');
            $data['paid_at'] = now();
        }

        $student->update($data);

        return redirect()
            ->route('graduations.students.show', [$graduation, $student])
            ->with('status', 'Student details updated.');
    }

    public function verify(Graduation $graduation, Student $student): RedirectResponse
    {
        $this->authorize('verify', $student);

        $student->update(['verified_at' => now()]);

        return redirect()
            ->route('graduations.students.show', [$graduation, $student])
            ->with('status', 'Payment verified.');
    }
}
```

The `update()` action's `if ($request->hasFile(...))` branch is the only place `paid_at` is auto-stamped. Re-saving without a new file leaves `paid_at` alone.

## Expected output

Controllers exist but aren't wired to routes yet. You can confirm they instantiate:

```bash
php artisan tinker
```

```php
>>> app(\App\Http\Controllers\GraduationController::class);
=> App\Http\Controllers\GraduationController { ... }
```

`php artisan route:list` won't show anything new until step 08.

## Commit your work

```bash
git add .
git commit -m "feat(http): add Graduation + Student controllers"
```

## Common pitfalls

- **`Call to undefined method authorize()`** — you forgot to add `AuthorizesRequests` to the base controller in step 1.
- **`Class "App\\Models\\Graduation" not found`** in the middleware string — note the double backslash. Laravel parses string middleware syntax, so single `\` would be eaten by PHP string escaping.
- **Order of arguments in `update()`** matters: Form Request first, then the nested route parameters in the order they appear in the URL (`graduation`, then `student`).

## What's next

Controllers without routes are dead code. Wire them up: [08 — Routes](./08-routes.md).
