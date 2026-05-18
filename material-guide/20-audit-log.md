# 20 — Audit log for sensitive actions

> **Commit:** `9d2a0e5` — *feat(audit): track who verified/created/deleted students and when*

## Why this step?

Verification is a financial trust event ("this admin marked this student as paid"). Deletion is irreversible. We want answers to questions like:

- Who verified student #57 last week?
- When was this student deleted, and by whom?

We build a tiny audit table — not the full `owen-it/laravel-auditing` package (overkill for the demo). It's a polymorphic, append-only log with a `record()` static helper:

```php
Audit::record('verified', $student);
Audit::record('deleted', $student, ['name' => $student->name, 'ic' => $student->ic]);
```

We write rows at four chokepoints in `StudentController`: `store`, `verify`, `destroy`, `bulk`. The bulk verify/delete branches add `{via: 'bulk'}` to `changes` so a reader can tell single from bulk.

A panel on the student show page renders the log — admin-only.

## What you'll build

- Migration: `audits` table (uuid PK, polymorphic `auditable_type/id`, json `changes`)
- `Audit` model with `record()` helper
- Four audit writes in `StudentController`
- Audit panel on `students/show.blade.php`
- 4 tests

## Prerequisites

- Completed [19 — Self-service registration](./19-self-service-registration.md).

## Steps

### 1. Generate the model + migration

```bash
php artisan make:model Audit -m
```

### 2. Fill in the migration

```php
public function up(): void
{
    Schema::create('audits', function (Blueprint $table) {
        $table->id();
        $table->uuid('uuid')->unique();
        $table->foreignIdFor(User::class)->nullable()->constrained()->nullOnDelete();
        $table->string('action');
        $table->string('auditable_type');
        $table->unsignedBigInteger('auditable_id');
        $table->json('changes')->nullable();
        $table->timestamps();

        $table->index(['auditable_type', 'auditable_id']);
    });
}

public function down(): void
{
    Schema::dropIfExists('audits');
}
```

The index makes "give me all audits for this student" fast.

### 3. `Audit` model

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class Audit extends Model
{
    use HasFactory;
    use HasUuids;

    protected $fillable = ['user_id', 'action', 'auditable_type', 'auditable_id', 'changes'];

    public function uniqueIds(): array
    {
        return ['uuid'];
    }

    protected function casts(): array
    {
        return [
            'changes' => 'array',
        ];
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function auditable(): MorphTo
    {
        return $this->morphTo();
    }

    public static function record(string $action, Model $model, array $changes = []): self
    {
        return static::create([
            'user_id' => auth()->id(),
            'action' => $action,
            'auditable_type' => $model::class,
            'auditable_id' => $model->id,
            'changes' => $changes,
        ]);
    }
}
```

### 4. Run the migration

```bash
php artisan migrate
```

### 5. Wire audit writes into `StudentController`

```php
use App\Models\Audit;

public function store(StoreStudentRequest $request, Graduation $graduation): RedirectResponse
{
    $student = $graduation->students()->create($request->validated());

    Audit::record('created', $student);

    return redirect()->route('graduations.show', $graduation)
        ->with('status', "Added {$student->name}.");
}

public function verify(Graduation $graduation, Student $student): RedirectResponse
{
    $this->authorize('verify', $student);

    $student->update(['verified_at' => now()]);

    Audit::record('verified', $student);

    return redirect()->route('graduations.students.show', [$graduation, $student])
        ->with('status', 'Payment verified.');
}

public function destroy(Graduation $graduation, Student $student): RedirectResponse
{
    $this->authorize('delete', $student);

    // Snapshot identifying info BEFORE delete — the row's about to vanish
    Audit::record('deleted', $student, [
        'name' => $student->name,
        'ic' => $student->ic,
    ]);

    $student->delete();

    return redirect()->route('graduations.show', $graduation)
        ->with('status', 'Student removed.');
}

public function bulk(BulkStudentActionRequest $request, Graduation $graduation): RedirectResponse
{
    $ids = $request->validated()['ids'];
    $action = $request->validated()['action'];
    $scope = $graduation->students()->whereIn('uuid', $ids);

    if ($action === 'verify') {
        $toVerify = (clone $scope)->whereNotNull('payment_receipt')->get();
        $skipped = (clone $scope)->whereNull('payment_receipt')->count();

        foreach ($toVerify as $student) {
            $student->update(['verified_at' => now()]);
            Audit::record('verified', $student, ['via' => 'bulk']);
        }

        return redirect()->route('graduations.show', $graduation)
            ->with('status', "Verified {$toVerify->count()}. Skipped {$skipped} (no receipt).");
    }

    $rows = (clone $scope)->get();
    foreach ($rows as $student) {
        Audit::record('deleted', $student, [
            'via' => 'bulk',
            'name' => $student->name,
            'ic' => $student->ic,
        ]);
    }
    $scope->delete();

    return redirect()->route('graduations.show', $graduation)
        ->with('status', "Removed {$rows->count()}.");
}
```

### 6. Render the audit panel on `students/show.blade.php`

```blade
@if (auth()->user()->isAdmin())
    @php
        $audits = App\Models\Audit::query()
            ->where('auditable_type', App\Models\Student::class)
            ->where('auditable_id', $student->id)
            ->with('user')
            ->latest()
            ->get();
    @endphp

    @if ($audits->isNotEmpty())
        <div class="mt-8 bg-white shadow rounded p-6">
            <h3 class="font-semibold mb-4">Audit log</h3>
            <ul class="space-y-2 text-sm">
                @foreach ($audits as $audit)
                    <li class="flex justify-between border-b pb-2">
                        <span>
                            <strong>{{ ucfirst($audit->action) }}</strong>
                            by {{ $audit->user?->name ?? 'system' }}
                            @if (($audit->changes['via'] ?? null) === 'bulk')
                                <span class="text-xs text-slate-500">(bulk)</span>
                            @endif
                        </span>
                        <span class="text-slate-500">{{ $audit->created_at->diffForHumans() }}</span>
                    </li>
                @endforeach
            </ul>
        </div>
    @endif
@endif
```

### 7. Add four tests

```php
it('writes an audit on single verify with the right user', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->paidUnverified()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]));

    $audit = App\Models\Audit::where('action', 'verified')
        ->where('auditable_id', $student->id)
        ->first();

    expect($audit)->not->toBeNull();
    expect($audit->user_id)->toBe($admin->id);
});

it('writes N audits on bulk verify with via=bulk', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $a = Student::factory()->for($grad)->paidUnverified()->create();
    $b = Student::factory()->for($grad)->paidUnverified()->create();

    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'verify',
            'ids' => [$a->uuid, $b->uuid],
        ]);

    $audits = App\Models\Audit::where('action', 'verified')->get();
    expect($audits)->toHaveCount(2);
    foreach ($audits as $audit) {
        expect($audit->changes['via'])->toBe('bulk');
    }
});

it('audit on delete snapshots name and ic', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create([
        'name' => 'Soon Gone',
        'ic' => '900101011000',
    ]);

    $this->actingAs($admin)
        ->delete(route('graduations.students.destroy', [$grad, $student]));

    $audit = App\Models\Audit::where('action', 'deleted')->first();
    expect($audit->changes['name'])->toBe('Soon Gone');
    expect($audit->changes['ic'])->toBe('900101011000');
});

it('non-admin does not see the Audit log panel', function () {
    $studentUser = User::factory()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create(['user_id' => $studentUser->id]);

    App\Models\Audit::record('verified', $student);

    $this->actingAs($studentUser)
        ->get(route('graduations.students.show', [$grad, $student]))
        ->assertDontSee('Audit log');
});
```

## Expected output

```
php artisan test
```

**66 tests / 157 assertions, all green.**

Browser:
1. As admin, verify a student → reload their detail page → see an "Audit log" panel with one row: `Verified by Admin User · a few seconds ago`.
2. Delete a student → audit's `changes` column still has their name + IC even though the row is gone.
3. Log in as the student user → audit panel is absent.

## Commit your work

```bash
git add .
git commit -m "feat(audit): track who verified/created/deleted students and when"
```

## Common pitfalls

- **Snapshot AFTER delete** — `Audit::record('deleted', ...)` after `$student->delete()` would lose the data. Snapshot *first*, then delete.
- **Polymorphic type stored as `'student'`** — the convention is the full class name (`App\Models\Student`). Use `$model::class` rather than `class_basename($model)` to be safe with model namespace moves.
- **Audit panel shows for students** — your `@if (auth()->user()->isAdmin())` is in the wrong place. The whole panel must be gated, not just the data fetch.
- **`auth()->id()` is `null` in console (artisan/tinker)** — that's expected. The `nullOnDelete` + nullable `user_id` covers it; the row gets `user_id = null` and renders as "by system".

## What's next

We track who verifies. Now we also tell the student: [21 — Email notifications](./21-email-notifications.md).
