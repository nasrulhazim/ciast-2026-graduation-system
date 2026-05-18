# 08 — Resource routes (and a nested resource)

> **Commit:** `eaac2e2` — *feat(routes): wire graduation + student resource routes*

## Why this step?

`Route::resource()` is Laravel's way of generating the seven RESTful routes (index, create, store, show, edit, update, destroy) in one line. It saves typing and — more importantly — enforces convention.

We use a **nested resource** for students because every student belongs to a graduation, and the URL should make that clear:

```
/graduations/{graduation}/students/{student}
```

`->only(['show', 'edit', 'update'])` filters the nested resource down to what we currently expose. Students aren't created via the web UI in this scope (admins seed them or — in step 13 — bulk-import them).

We also add an **explicit named route** for the custom `verify` action because `Route::resource()` only generates the seven REST verbs.

## What you'll build

- Resource routes for graduations: `GET /graduations`, `POST /graduations`, etc.
- Nested resource routes for students under each graduation, limited to show/edit/update
- A custom `PATCH /graduations/{graduation}/students/{student}/verify` route
- All wrapped in the existing `auth` middleware group from Breeze

## Prerequisites

- Completed [07 — Controllers](./07-controllers.md).

## Steps

### 1. Replace `routes/web.php`

```php
<?php

declare(strict_types=1);

use App\Http\Controllers\GraduationController;
use App\Http\Controllers\ProfileController;
use App\Http\Controllers\StudentController;
use Illuminate\Support\Facades\Route;

Route::get('/', function () {
    return view('welcome');
});

Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware(['auth', 'verified'])->name('dashboard');

Route::middleware('auth')->group(function () {
    Route::get('/profile', [ProfileController::class, 'edit'])->name('profile.edit');
    Route::patch('/profile', [ProfileController::class, 'update'])->name('profile.update');
    Route::delete('/profile', [ProfileController::class, 'destroy'])->name('profile.destroy');

    Route::resource('graduations', GraduationController::class);

    Route::resource('graduations.students', StudentController::class)
        ->only(['show', 'edit', 'update']);

    Route::patch(
        'graduations/{graduation}/students/{student}/verify',
        [StudentController::class, 'verify']
    )->name('graduations.students.verify');
});

require __DIR__ . '/auth.php';
```

### 2. Verify the route list

```bash
php artisan route:list --path=graduations
```

You should see something like:

```
GET      /graduations                                graduations.index
GET      /graduations/create                         graduations.create
POST     /graduations                                graduations.store
GET      /graduations/{graduation}                   graduations.show
GET      /graduations/{graduation}/edit              graduations.edit
PUT|PATCH /graduations/{graduation}                  graduations.update
DELETE   /graduations/{graduation}                   graduations.destroy
GET      /graduations/{graduation}/students/{student}        graduations.students.show
GET      /graduations/{graduation}/students/{student}/edit   graduations.students.edit
PUT|PATCH /graduations/{graduation}/students/{student}       graduations.students.update
PATCH    /graduations/{graduation}/students/{student}/verify graduations.students.verify
```

Note: the URL parameters bind by **uuid** because both models override `getRouteKeyName()`. You will never see a numeric `{graduation}` in a URL.

## Expected output

```bash
php artisan route:list --path=graduations
```

…shows the 11 routes above. Routes resolve, but visiting any of them right now hits a "view not found" error — views land in the next step.

## Commit your work

```bash
git add .
git commit -m "feat(routes): wire graduation + student resource routes"
```

## Common pitfalls

- **Route collision:** if you add static segments (like `students/import`) later, you must register them *before* the resource line, or `{student}` will swallow `import` as a UUID.
- **Wrong order of nested params:** `graduations.students.show` URL is `/graduations/{graduation}/students/{student}` — parent first, child last. The controller method signature `show(Graduation $graduation, Student $student)` matches that order.
- **`->only([...])`** is *exclusive*: it lists what to keep. There's also `->except([...])` for the opposite. Don't accidentally combine them.

## What's next

Routes hit controllers, but controllers want views. Time to build the UI: [09 — Blade views](./09-blade-views.md).
