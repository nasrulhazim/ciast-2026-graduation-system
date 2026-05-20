# Graduation Registration System — Build Spec

> **Audience:** Claude Code
> **Goal:** Build a complete Laravel 13 application from scratch — `laravel new` to working app with authentication, authorization via policies, and full CRUD for a Graduation Registration domain.
> **Estimated build time:** 1.5–2 hours when followed sequentially.

---

## Context

A university convocation registration system. **Admins** upload a graduation event, **students** verify their info and upload payment receipts, **admins** then verify those payments.

Whiteboard sketch of the flow:

```
ADMIN                    STUDENT                    ADMIN
─────                    ───────                    ─────
1. Create graduation   2. Verify own info        3. Check payment
2. Upload student list    · name, IC, email         · review receipt
                          · matric card, phone      · click "Verify"
                       3. Fill questionnaire
                       4. Upload payment receipt
```

---

## Stack

- **Laravel 13** (PHP 8.3+)
- **[Livewire starter kit](https://laravel.com/docs/13.x/starter-kits#livewire)** for authentication scaffolding (Fortify + Livewire 4 + Flux UI + Tailwind CSS 4)
- **Tailwind CSS 4** via Vite
- **SQLite** for local development
- **Pest** for testing
- **Spatie Laravel Permission** for roles (optional but recommended)

---

## Hard rules — do not deviate

These conventions apply across every file you generate. They are not stylistic preferences; they are part of the deliverable.

### 1. Hybrid UUID strategy

Each domain table gets **both** identifiers:

- `id` — `bigint` auto-increment, used for **foreign keys and joins**
- `uuid` — `UUIDv7` (filled by `HasUuids` trait), used in **URLs and public references**

```php
class Graduation extends Model
{
    use HasUuids;

    public function uniqueIds(): array
    {
        return ['uuid'];                         // not 'id'
    }

    public function getRouteKeyName(): string
    {
        return 'uuid';                            // route binding via uuid
    }
}
```

Migrations declare both columns explicitly:

```php
$table->id();
$table->uuid('uuid')->unique();
```

### 2. `foreignIdFor` over `foreignId`

```php
// ❌ Don't
$table->foreignId('graduation_id')->constrained()->cascadeOnDelete();

// ✅ Do
$table->foreignIdFor(Graduation::class)->constrained()->cascadeOnDelete();
```

Cleaner refactoring if the model namespace ever moves.

### 3. Factory + Seeder pattern

Seeders compose **pre-seed** (predictable demo records) with **factory bulk** (volume):

```php
public function run(): void
{
    // 1. Pre-seed: predictable records used in demos/screenshots
    $admin = User::factory()->admin()->create([
        'name' => 'Admin User',
        'email' => 'admin@devhub.test',
    ]);

    $june = Graduation::factory()->open()->create([
        'title' => 'June 2026 Convocation',
        'fee' => 250.00,
    ]);
    Student::factory()->count(20)->verified()->for($june)->create();

    // 2. Bulk via factory — realistic volume + relations via has()
    Graduation::factory()->count(2)
        ->has(Student::factory()->count(15))
        ->create();
}
```

Factories use Malaysian locale: `fake('ms_MY')->name()`, `fake('ms_MY')->phoneNumber()`.

### 4. Thin controllers, Form Requests for validation + authorization

Controller actions stay 5–10 lines. **Validation rules and `authorize()` checks live in Form Request classes**, not in controllers.

### 5. Authorization through Policies, called in three places

- **Form Requests** — `authorize()` method calls `Gate` or `$this->user()->can()`
- **Controllers** — `$this->authorize('action', $model)` as defense in depth
- **Blade** — `@can('action', $model)` to hide UI affordances students cannot use

### 6. Soft deletes on Graduation, hard delete on Student

Graduation has audit value (financial record); Student does not. Use `SoftDeletes` trait on Graduation only.

### 7. Strict types + return type declarations

Every PHP file starts with `declare(strict_types=1);`. Every function has explicit parameter types and return type.

---

## Build phases

Execute these phases in order. After each phase, run the listed verification command(s) and confirm they pass before continuing.

---

### Phase 1 — Project scaffold

**Goal:** Empty but configured Laravel 13 project running on localhost.

```bash
composer create-project laravel/laravel graduation-system "13.*"
cd graduation-system
```

Configure `.env` for SQLite:

```bash
# In .env, change:
DB_CONNECTION=sqlite
# Remove or comment out DB_HOST, DB_PORT, DB_DATABASE, DB_USERNAME, DB_PASSWORD

touch database/database.sqlite
php artisan key:generate
```

Set Malaysian locale defaults in `config/app.php`:

```php
'timezone' => 'Asia/Kuala_Lumpur',
'faker_locale' => 'ms_MY',
```

**Verify:**
```bash
php artisan serve   # http://localhost:8000 should show Laravel welcome
```

---

### Phase 2 — Authentication via the Livewire starter kit

**Goal:** Working login/register/password reset/email verification/2FA/passkeys with Fortify + Livewire 4 + Flux UI + Tailwind CSS 4.

The kit lands at project creation in Phase 1 (`laravel new graduation-system --livewire --pest`). Phase 2 just confirms it works:

```bash
npm install
npm run build
php artisan migrate
php artisan test   # ~25 kit tests pass
```

> Don't try `composer require laravel/livewire-starter-kit` — starter kits are project templates, not installable packages. If Phase 1 was created without the `--livewire` flag, delete the directory and re-run with the flag.

**Verify:**
- Visit `/register` → can create an account
- Visit `/login` → can log in
- After login, redirected to `/dashboard`
- `/settings/profile` allows editing name/email
- `/settings/appearance` toggles dark mode (and the toggle persists)
- `/settings/security` exposes password change, 2FA enrolment, and passkey management

---

### Phase 3 — Roles & user types

**Goal:** Distinguish admin from student users, with role stored on the User model.

Decision: for this demo, use a **simple `is_admin` boolean** on users. (For production, Nasrul prefers `spatie/laravel-permission` — see "Production upgrade path" at the end.)

#### 3.1 Add column to users table

Create a new migration:

```bash
php artisan make:migration add_role_columns_to_users_table --table=users
```

```php
public function up(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->uuid('uuid')->after('id')->nullable()->unique();
        $table->boolean('is_admin')->default(false)->after('password');
    });
}

public function down(): void
{
    Schema::table('users', function (Blueprint $table) {
        $table->dropColumn(['uuid', 'is_admin']);
    });
}
```

#### 3.2 Update the User model

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Concerns\HasUuids;
use Illuminate\Foundation\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
    use HasUuids;                  // ← add this
    use Notifiable;

    protected $fillable = [
        'name', 'email', 'password', 'is_admin',
    ];

    protected $hidden = [
        'password', 'remember_token',
    ];

    public function uniqueIds(): array
    {
        return ['uuid'];           // ← uuid column, not id
    }

    public function getRouteKeyName(): string
    {
        return 'uuid';
    }

    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'password' => 'hashed',
            'is_admin' => 'boolean',
        ];
    }

    public function isAdmin(): bool
    {
        return $this->is_admin === true;
    }

    /**
     * Each User may have a linked Student record (their own registration).
     * Optional — only present for non-admin users registered for a graduation.
     */
    public function student(): \Illuminate\Database\Eloquent\Relations\HasOne
    {
        return $this->hasOne(Student::class);
    }
}
```

#### 3.3 Update UserFactory

```php
public function definition(): array
{
    return [
        'name' => fake('ms_MY')->name(),
        'email' => fake()->unique()->safeEmail(),
        'email_verified_at' => now(),
        'password' => static::$password ??= Hash::make('password'),
        'remember_token' => Str::random(10),
        'is_admin' => false,
    ];
}

public function admin(): static
{
    return $this->state(fn () => ['is_admin' => true]);
}
```

**Verify:**
```bash
php artisan migrate:fresh
php artisan tinker
>>> $u = User::factory()->admin()->create();
>>> $u->isAdmin();   // true
>>> $u->uuid;        // UUIDv7 string
```

---

### Phase 4 — Domain models, migrations, factories

**Goal:** Graduation + Student tables, models, factories ready to seed.

#### 4.1 Generate everything in one go

```bash
php artisan make:model Graduation -mfsRc --requests
php artisan make:model Student -mfsRc --requests
```

Flags decoded: `-m` migration, `-f` factory, `-s` seeder, `-R` resource controller, `-c` controller, `--requests` Form Requests for store and update.

#### 4.2 Graduations migration

```php
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

#### 4.3 Students migration

```php
use App\Models\Graduation;
use App\Models\User;

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

`user_id` is **nullable** — admins upload student records before students register; the linkage happens when the student creates a User account with a matching email.

#### 4.4 Graduation model

```php
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

#### 4.5 Student model

```php
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

#### 4.6 Factories

`GraduationFactory.php`:

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

`StudentFactory.php`:

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

#### 4.7 DatabaseSeeder

```php
namespace Database\Seeders;

use App\Models\Graduation;
use App\Models\Student;
use App\Models\User;
use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        // ─── 1. Pre-seed predictable demo accounts ───
        User::factory()->admin()->create([
            'name' => 'Admin User',
            'email' => 'admin@devhub.test',
        ]);

        User::factory()->create([
            'name' => 'Student User',
            'email' => 'student@devhub.test',
        ]);

        // ─── 2. Pre-seed showcase graduation ───
        $june = Graduation::factory()->open()->create([
            'title' => 'June 2026 Convocation',
            'ceremony_date' => '2026-06-15',
            'fee' => 250.00,
        ]);

        Student::factory()->count(10)->verified()->for($june)->create();
        Student::factory()->count(4)->paidUnverified()->for($june)->create();
        Student::factory()->count(6)->for($june)->create();

        // ─── 3. Bulk via factory ───
        Graduation::factory()->count(2)
            ->has(Student::factory()->count(15))
            ->create();
    }
}
```

**Verify:**
```bash
php artisan migrate:fresh --seed
php artisan tinker
>>> Graduation::count()           // 3
>>> Student::count()              // 50
>>> Graduation::first()->uuid     // UUIDv7 string
>>> User::where('email', 'admin@devhub.test')->first()->isAdmin()  // true
```

---

### Phase 5 — Policies

**Goal:** Authorization rules per model, registered for auto-discovery.

#### 5.1 Generate policies

```bash
php artisan make:policy GraduationPolicy --model=Graduation
php artisan make:policy StudentPolicy --model=Student
```

#### 5.2 GraduationPolicy

```php
namespace App\Policies;

use App\Models\Graduation;
use App\Models\User;

class GraduationPolicy
{
    /**
     * Any logged-in user can view the list (but they only see open ones — filtered in controller).
     */
    public function viewAny(User $user): bool
    {
        return true;
    }

    /**
     * Anyone can view a graduation (public-ish information).
     */
    public function view(User $user, Graduation $graduation): bool
    {
        return true;
    }

    /**
     * Only admins create graduations.
     */
    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    /**
     * Only admins update graduations.
     */
    public function update(User $user, Graduation $graduation): bool
    {
        return $user->isAdmin();
    }

    /**
     * Only admins archive graduations, and never if it has paid students.
     */
    public function delete(User $user, Graduation $graduation): bool
    {
        if (! $user->isAdmin()) {
            return false;
        }

        return $graduation->students()
            ->whereNotNull('paid_at')
            ->doesntExist();
    }

    public function restore(User $user, Graduation $graduation): bool
    {
        return $user->isAdmin();
    }
}
```

#### 5.3 StudentPolicy

```php
namespace App\Policies;

use App\Models\Student;
use App\Models\User;

class StudentPolicy
{
    public function viewAny(User $user): bool
    {
        return $user->isAdmin();         // only admins see the full student list
    }

    /**
     * Admin can view any student. Non-admin can only view their own record.
     */
    public function view(User $user, Student $student): bool
    {
        return $user->isAdmin() || $student->user_id === $user->id;
    }

    /**
     * Students aren't created via the public UI in this demo —
     * admin uploads a list (we'll add this in a later phase).
     */
    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    /**
     * Admin can edit any student. Student can edit their own record only.
     */
    public function update(User $user, Student $student): bool
    {
        return $user->isAdmin() || $student->user_id === $user->id;
    }

    /**
     * Only admins delete student records.
     */
    public function delete(User $user, Student $student): bool
    {
        return $user->isAdmin();
    }

    /**
     * Only admins verify payments. Verification requires a payment receipt.
     */
    public function verify(User $user, Student $student): bool
    {
        return $user->isAdmin() && $student->payment_receipt !== null;
    }
}
```

#### 5.4 Auto-discovery confirmation

Laravel 13 auto-discovers policies when:
- The Model lives in `App\Models\*` (default)
- The Policy lives in `App\Policies\*` and is named `{Model}Policy`

Both conditions are met. No manual `Gate::policy()` registration needed.

**Verify:**
```bash
php artisan tinker
>>> $admin = User::where('email', 'admin@devhub.test')->first();
>>> $student = User::where('email', 'student@devhub.test')->first();
>>> $grad = Graduation::first();
>>> $admin->can('update', $grad);     // true
>>> $student->can('update', $grad);   // false
```

---

### Phase 6 — Form Requests

**Goal:** Validation rules + authorization checks per write endpoint.

#### 6.1 StoreGraduationRequest

```php
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

#### 6.2 UpdateGraduationRequest

```php
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
            'ceremony_date' => ['required', 'date'],
            'fee' => ['required', 'numeric', 'min:0', 'max:9999.99'],
            'status' => ['required', 'in:draft,open,closed'],
        ];
    }
}
```

#### 6.3 UpdateStudentRequest

```php
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

---

### Phase 7 — Controllers

**Goal:** Wire HTTP requests to model operations. Defense in depth via `authorize()`.

#### 7.1 GraduationController

```php
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

#### 7.2 StudentController

```php
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

---

### Phase 8 — Routes

`routes/web.php`:

```php
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

    // Graduation CRUD — admin-gated via policies
    Route::resource('graduations', GraduationController::class);

    // Students nested under graduations
    Route::resource('graduations.students', StudentController::class)
        ->only(['show', 'edit', 'update']);

    // Admin verifies a payment
    Route::patch(
        'graduations/{graduation}/students/{student}/verify',
        [StudentController::class, 'verify']
    )->name('graduations.students.verify');
});

require __DIR__ . '/auth.php';
```

**Verify:**
```bash
php artisan route:list --path=graduation
```

---

### Phase 9 — Views

All views wrap content with the Livewire starter kit's `<x-layouts::app :title="...">` component (which renders the Flux sidebar shell + dark-mode aware chrome). Tailwind CSS 4 utility classes used throughout — write `<label>` / `<input>` / `<button>` directly with utility chains instead of relying on Breeze-style `<x-text-input>` / `<x-primary-button>` components (the kit doesn't ship those). Follow this file structure:

```
resources/views/
├── graduations/
│   ├── index.blade.php       # paginated list with status badges
│   ├── create.blade.php      # form (admin only — hide link via @can)
│   ├── edit.blade.php        # form + archive button
│   └── show.blade.php        # detail + nested student table
└── students/
    ├── show.blade.php        # detail + verify button (@can('verify'))
    └── edit.blade.php        # form with file upload (enctype)
```

**Key Blade patterns required:**

1. **Hide UI affordances via `@can`**:
```blade
@can('create', App\Models\Graduation::class)
    <a href="{{ route('graduations.create') }}">+ New graduation</a>
@endcan

@can('verify', $student)
    <form method="POST" action="{{ route('graduations.students.verify', [$graduation, $student]) }}">
        @csrf @method('PATCH')
        <button>Verify payment</button>
    </form>
@endcan
```

2. **File upload form must have `enctype`**:
```blade
<form action="{{ route('graduations.students.update', [$graduation, $student]) }}"
      method="POST" enctype="multipart/form-data">
    @csrf @method('PATCH')
    ...
    <input type="file" name="payment_receipt" accept=".pdf,image/jpeg,image/png">
</form>
```

3. **Old input + error display**:
```blade
<input type="text" name="title" value="{{ old('title', $graduation->title ?? '') }}">
@error('title') <span class="text-sm text-red-600">{{ $message }}</span> @enderror
```

4. **Status badges with conditional classes**:
```blade
@php
    $colors = [
        'draft' => 'bg-slate-100 text-slate-700',
        'open' => 'bg-green-100 text-green-700',
        'closed' => 'bg-amber-100 text-amber-700',
    ];
@endphp
<span class="px-2 py-1 text-xs rounded {{ $colors[$g->status] }}">
    {{ ucfirst($g->status) }}
</span>
```

5. **Verified/paid/unpaid indicator**:
```blade
@if ($student->isVerified())
    <span class="px-2 py-1 text-xs rounded bg-green-100 text-green-700">Verified</span>
@elseif ($student->hasPaid())
    <span class="px-2 py-1 text-xs rounded bg-amber-100 text-amber-700">Pending review</span>
@else
    <span class="px-2 py-1 text-xs rounded bg-slate-100 text-slate-600">Not paid</span>
@endif
```

Update the Livewire starter kit's sidebar at `resources/views/layouts/app/sidebar.blade.php` to add a `<flux:sidebar.item icon="academic-cap" :href="route('graduations.index')" :current="request()->routeIs('graduations.*')" wire:navigate>{{ __('Graduations') }}</flux:sidebar.item>` inside the existing **Platform** group. The single sidebar component covers both desktop and the mobile collapsed menu.

---

### Phase 10 — Storage symlink

```bash
php artisan storage:link
```

This makes `/storage/receipts/*.pdf` web-accessible. Required before testing file uploads.

---

### Phase 11 — Pest tests

`tests/Feature/GraduationFlowTest.php`:

```php
<?php

declare(strict_types=1);

use App\Models\Graduation;
use App\Models\Student;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

uses(RefreshDatabase::class);

it('redirects guests away from the index', function () {
    $this->get(route('graduations.index'))
        ->assertRedirect(route('login'));
});

it('lists graduations for authenticated users', function () {
    $user = User::factory()->create();
    Graduation::factory()->count(3)->create();

    $this->actingAs($user)
        ->get(route('graduations.index'))
        ->assertOk();
});

it('denies non-admin from creating a graduation', function () {
    $user = User::factory()->create();

    $this->actingAs($user)
        ->post(route('graduations.store'), [
            'title' => 'Test',
            'ceremony_date' => now()->addMonth()->format('Y-m-d'),
            'fee' => 250,
            'status' => 'open',
        ])
        ->assertForbidden();
});

it('admin can create a graduation', function () {
    $admin = User::factory()->admin()->create();

    $this->actingAs($admin)
        ->post(route('graduations.store'), [
            'title' => 'Test Convocation',
            'ceremony_date' => now()->addMonth()->format('Y-m-d'),
            'fee' => 300.00,
            'status' => 'open',
        ])
        ->assertRedirect();

    expect(Graduation::where('title', 'Test Convocation')->exists())->toBeTrue();
});

it('resolves graduations by uuid in URLs', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();

    expect(route('graduations.show', $grad))
        ->toContain($grad->uuid)
        ->not->toContain("/{$grad->id}");
});

it('admin can upload a receipt for any student', function () {
    Storage::fake('public');

    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.update', [$grad, $student]), [
            'name' => $student->name,
            'ic' => $student->ic,
            'email' => $student->email,
            'matric_card' => $student->matric_card,
            'phone' => $student->phone,
            'payment_receipt' => UploadedFile::fake()->create('receipt.pdf', 100, 'application/pdf'),
        ])
        ->assertRedirect();

    expect($student->fresh()->payment_receipt)->not->toBeNull();
});

it('student can only update their own record', function () {
    $studentUser = User::factory()->create();
    $grad = Graduation::factory()->create();

    $mine = Student::factory()->for($grad)->create(['user_id' => $studentUser->id]);
    $other = Student::factory()->for($grad)->create();

    // Can update own
    $this->actingAs($studentUser)
        ->get(route('graduations.students.edit', [$grad, $mine]))
        ->assertOk();

    // Cannot update someone else's
    $this->actingAs($studentUser)
        ->get(route('graduations.students.edit', [$grad, $other]))
        ->assertForbidden();
});

it('admin verifies a paid student', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->paidUnverified()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]))
        ->assertRedirect();

    expect($student->fresh()->verified_at)->not->toBeNull();
});

it('admin cannot verify a student with no receipt', function () {
    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create();   // no receipt

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]))
        ->assertForbidden();
});
```

**Verify:**
```bash
php artisan test
```

All tests should pass.

---

### Phase 12 — Final smoke test

Walk through the full workflow manually:

1. `php artisan migrate:fresh --seed`
2. `php artisan serve` + `npm run dev`
3. Visit `http://localhost:8000/login`
4. Log in as `admin@devhub.test` / `password`
5. Navigate to `/graduations` → see 3 graduations
6. Click "New graduation" → submit form → redirected to show page
7. Edit the new graduation → save changes
8. Click on a student → upload a PDF receipt → save
9. Back on the student show page → click "Verify payment" → success
10. Log out, log in as `student@devhub.test`
11. Visit `/graduations` → can browse but no "New graduation" button visible
12. Try to access `/graduations/{uuid}/edit` → 403 Forbidden

---

## Success criteria checklist

Before declaring complete, confirm every item:

- [ ] `php artisan migrate:fresh --seed` runs without errors
- [ ] `php artisan test` passes all tests
- [ ] `php artisan route:list --path=graduation` shows all 8 routes (resource + verify)
- [ ] All URLs use UUIDs, never bigint ids
- [ ] Admin user (`admin@devhub.test`) can perform all CRUD
- [ ] Student user (`student@devhub.test`) sees the list but cannot create/edit/delete
- [ ] File upload stores receipts under `storage/app/public/receipts/`
- [ ] `storage:link` is in place — uploaded receipts are accessible via browser
- [ ] Status badges render with correct colors (slate/green/amber)
- [ ] Logout works; login redirects to `/dashboard`
- [ ] Every Form Request has both `authorize()` and `rules()` methods
- [ ] Every controller action either type-hints a Form Request or calls `$this->authorize()`
- [ ] Every Blade form that submits non-GET has `@csrf` and `@method(...)` directives

---

## Production upgrade path (post-demo)

After the Khamis demo session, recommend these as natural extensions:

1. **Spatie Laravel Permission** — replace `is_admin` boolean with proper roles + permissions stored in DB:
   ```bash
   composer require spatie/laravel-permission
   php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
   php artisan migrate
   ```
   Replace `$user->isAdmin()` with `$user->hasRole('admin')` everywhere.

2. **Excel/CSV import** — implement the "admin uploads student list" step (1 on the whiteboard):
   ```bash
   composer require spatie/simple-excel
   php artisan make:command ImportGraduationList
   ```

3. **Email notifications** — notify students when their payment is verified:
   ```bash
   php artisan make:notification PaymentVerified
   ```

4. **Audit log** — track who changed what on the Student model:
   ```bash
   composer require owen-it/laravel-auditing
   ```

5. **Queue payment receipts** — virus-scan uploaded files in a queued job before approving.

6. **API endpoints** — add Sanctum tokens for a future mobile app:
   ```bash
   php artisan install:api
   ```

---

## File manifest

When complete, the project should have these new/modified files relative to `laravel new`:

```
graduation-system/
├── app/
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── GraduationController.php     [new]
│   │   │   └── StudentController.php        [new]
│   │   └── Requests/
│   │       ├── StoreGraduationRequest.php   [new]
│   │       ├── UpdateGraduationRequest.php  [new]
│   │       └── UpdateStudentRequest.php     [new]
│   ├── Models/
│   │   ├── Graduation.php                   [new]
│   │   ├── Student.php                      [new]
│   │   └── User.php                         [modified — HasUuids + isAdmin]
│   └── Policies/
│       ├── GraduationPolicy.php             [new]
│       └── StudentPolicy.php                [new]
├── database/
│   ├── factories/
│   │   ├── GraduationFactory.php            [new]
│   │   ├── StudentFactory.php               [new]
│   │   └── UserFactory.php                  [modified — admin state]
│   ├── migrations/
│   │   ├── XXXX_add_role_columns_to_users_table.php  [new]
│   │   ├── XXXX_create_graduations_table.php         [new]
│   │   └── XXXX_create_students_table.php            [new]
│   └── seeders/
│       └── DatabaseSeeder.php               [modified]
├── resources/views/
│   ├── graduations/                         [new directory]
│   │   ├── index.blade.php
│   │   ├── create.blade.php
│   │   ├── edit.blade.php
│   │   └── show.blade.php
│   ├── students/                            [new directory]
│   │   ├── show.blade.php
│   │   └── edit.blade.php
│   └── layouts/
│       └── navigation.blade.php             [modified — add Graduations link]
├── routes/
│   └── web.php                              [modified]
└── tests/Feature/
    └── GraduationFlowTest.php               [new]
```

---

**End of spec.** Build the project phase by phase. Run verification commands at each phase boundary. Do not skip ahead.
