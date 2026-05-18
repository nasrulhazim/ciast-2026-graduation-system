# 04 — Graduation + Student models, migrations, factories, seeder

> **Commit:** `eed392e` — *feat(domain): add Graduation + Student models, migrations, factories, seeder*

## Why this step?

This is the heart of the domain. Two tables, one relationship:

- **Graduation** — a convocation event. Has `SoftDeletes` because it's a financial record we don't want to truly lose.
- **Student** — a registration tied to one graduation. Hard-deleted because we don't need to keep abandoned/spam rows forever.

Two unusual but deliberate choices:

- **`user_id` on students is nullable**. Admins upload student rosters *before* students self-register. The user-to-student linkage happens later (step 19) when a student creates an account with a matching email.
- **`foreignIdFor(Graduation::class)` instead of `foreignId('graduation_id')`**. If we ever move the model namespace, the foreign key column auto-renames. Tiny upfront cost, no future pain.

Factories ship with **named states** (`open()`, `closed()`, `verified()`, `paidUnverified()`) so seeders and tests stay readable.

## What you'll build

- Migrations for `graduations` and `students` tables
- `Graduation` model (soft-deletable, UUIDv7-routed, with `students()` HasMany)
- `Student` model (hard-delete, UUIDv7-routed, with `graduation()` + `user()` BelongsTo)
- Factories with named states
- A `DatabaseSeeder` that pre-seeds 2 demo users + 3 graduations + 50 students

## Prerequisites

- Completed [03 — User roles + UUID](./03-user-roles-uuid.md).

## Steps

### 1. Generate everything in two commands

```bash
php artisan make:model Graduation -mfsRc --requests
php artisan make:model Student -mfsRc --requests
```

Flags decoded:

- `-m` migration
- `-f` factory
- `-s` seeder
- `-R` resource controller (one file with 7 RESTful actions)
- `-c` controller (folded into `-R` but harmless)
- `--requests` two Form Request classes (Store + Update)

You now have ~12 stub files. We'll fill them in over the next several steps; here we only touch models, migrations, factories, and the database seeder.

### 2. Graduations migration

Edit `database/migrations/XXXX_create_graduations_table.php`:

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('graduations', function (Blueprint $table) {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->string('title');
            $table->date('ceremony_date');
            $table->decimal('fee', 8, 2);
            $table->enum('status', ['draft', 'open', 'closed'])->default('draft');
            $table->timestamps();
            $table->softDeletes();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('graduations');
    }
};
```

### 3. Students migration

Edit `database/migrations/XXXX_create_students_table.php`:

```php
<?php

declare(strict_types=1);

use App\Models\Graduation;
use App\Models\User;
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('students', function (Blueprint $table) {
            $table->id();
            $table->uuid('uuid')->unique();
            $table->foreignIdFor(Graduation::class)->constrained()->cascadeOnDelete();
            $table->foreignIdFor(User::class)->nullable()->constrained()->nullOnDelete();
            $table->string('name');
            $table->string('ic')->unique();
            $table->string('email')->unique();
            $table->string('matric_card');
            $table->string('phone');
            $table->string('payment_receipt')->nullable();
            $table->timestamp('verified_at')->nullable();
            $table->timestamp('paid_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('students');
    }
};
```

### 4. `Graduation` model

Replace `app/Models/Graduation.php`:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;
use Illuminate\Database\Eloquent\SoftDeletes;

class Graduation extends Model
{
    use HasFactory;
    use HasUuids;
    use SoftDeletes;

    protected $fillable = [
        'title', 'ceremony_date', 'fee', 'status',
    ];

    public function uniqueIds(): array
    {
        return ['uuid'];
    }

    public function getRouteKeyName(): string
    {
        return 'uuid';
    }

    protected function casts(): array
    {
        return [
            'ceremony_date' => 'date',
            'fee' => 'decimal:2',
        ];
    }

    public function students(): HasMany
    {
        return $this->hasMany(Student::class);
    }

    public function isOpen(): bool
    {
        return $this->status === 'open';
    }
}
```

### 5. `Student` model

Replace `app/Models/Student.php`:

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Student extends Model
{
    use HasFactory;
    use HasUuids;

    protected $fillable = [
        'graduation_id', 'user_id',
        'name', 'ic', 'email', 'matric_card', 'phone',
        'payment_receipt', 'verified_at', 'paid_at',
    ];

    public function uniqueIds(): array
    {
        return ['uuid'];
    }

    public function getRouteKeyName(): string
    {
        return 'uuid';
    }

    protected function casts(): array
    {
        return [
            'verified_at' => 'datetime',
            'paid_at' => 'datetime',
        ];
    }

    public function graduation(): BelongsTo
    {
        return $this->belongsTo(Graduation::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function isVerified(): bool
    {
        return $this->verified_at !== null;
    }

    public function hasPaid(): bool
    {
        return $this->payment_receipt !== null && $this->paid_at !== null;
    }
}
```

### 6. `GraduationFactory`

Replace `database/factories/GraduationFactory.php` body:

```php
public function definition(): array
{
    $year = fake()->numberBetween(2024, 2027);
    $month = fake()->randomElement(['March', 'June', 'September', 'December']);

    return [
        'title' => "{$month} {$year} Convocation",
        'ceremony_date' => fake()->dateTimeBetween('+1 month', '+1 year'),
        'fee' => fake()->randomElement([150.00, 200.00, 250.00, 300.00]),
        'status' => fake()->randomElement(['draft', 'open', 'closed']),
    ];
}

public function open(): static
{
    return $this->state(fn () => ['status' => 'open']);
}

public function closed(): static
{
    return $this->state(fn () => ['status' => 'closed']);
}
```

### 7. `StudentFactory`

```php
public function definition(): array
{
    return [
        'name' => fake('ms_MY')->name(),
        'ic' => fake()->unique()->numerify('############'),
        'email' => fake()->unique()->safeEmail(),
        'matric_card' => 'M' . fake()->unique()->numerify('######'),
        'phone' => fake('ms_MY')->phoneNumber(),
    ];
}

public function verified(): static
{
    return $this->state(fn () => [
        'payment_receipt' => 'receipts/sample-receipt.pdf',
        'paid_at' => now()->subDays(fake()->numberBetween(1, 14)),
        'verified_at' => now()->subDays(fake()->numberBetween(0, 7)),
    ]);
}

public function paidUnverified(): static
{
    return $this->state(fn () => [
        'payment_receipt' => 'receipts/sample-receipt.pdf',
        'paid_at' => now()->subDays(fake()->numberBetween(1, 3)),
        'verified_at' => null,
    ]);
}
```

### 8. `DatabaseSeeder`

Replace `database/seeders/DatabaseSeeder.php`:

```php
<?php

declare(strict_types=1);

namespace Database\Seeders;

use App\Models\Graduation;
use App\Models\Student;
use App\Models\User;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        User::factory()->admin()->create([
            'name' => 'Admin User',
            'email' => 'admin@devhub.test',
        ]);

        User::factory()->create([
            'name' => 'Student User',
            'email' => 'student@devhub.test',
        ]);

        $june = Graduation::factory()->open()->create([
            'title' => 'June 2026 Convocation',
            'ceremony_date' => '2026-06-15',
            'fee' => 250.00,
        ]);

        Student::factory()->count(10)->verified()->for($june)->create();
        Student::factory()->count(4)->paidUnverified()->for($june)->create();
        Student::factory()->count(6)->for($june)->create();

        Graduation::factory()->count(2)
            ->has(Student::factory()->count(15))
            ->create();
    }
}
```

### 9. Reset + seed

```bash
php artisan migrate:fresh --seed
```

## Expected output

```bash
php artisan tinker
```

```php
>>> Graduation::count();                                        // 3
>>> Student::count();                                            // 50
>>> Graduation::first()->uuid;                                   // UUIDv7
>>> Graduation::where('title', 'like', 'June%')->first()->students()->count();  // 20
>>> User::where('email', 'admin@devhub.test')->first()->isAdmin();              // true
```

## Commit your work

```bash
git add .
git commit -m "feat(domain): add Graduation + Student models, migrations, factories, seeder"
```

## Common pitfalls

- **Migration order matters.** `students` references `graduations` and `users`. If for some reason your migration timestamps got reordered, fix the filenames so `graduations` runs before `students`.
- **`SoftDeletes` on Student?** No. Spec says hard-delete on Student, soft-delete on Graduation. The reason is audit value — see `build-spec.md` hard rule #6.
- **Factory creates `paid_at` but no receipt** — verify the `paidUnverified()` state sets both. Otherwise the policy check in step 05 (`verify` requires a receipt) won't make sense.

## What's next

Models exist but anyone could mutate them. Time to lock down with policies: [05 — Policies](./05-policies.md).
