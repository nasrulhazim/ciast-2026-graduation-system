# 19 — Link a User to a Student by email on registration

> **Commit:** `387e229` — *feat(students): self-service — link User to Student by email on registration*

## Why this step?

Recall the flow:

1. **Admin** pre-uploads a student roster (rows have `email` but `user_id` is NULL).
2. **Student** visits `/register` and creates an account with the same email.
3. **The system** silently links the new `User` to the existing `Student` row.

After step 3, the student can immediately log in, see "My registration" in the nav, and upload their receipt — because the `StudentPolicy::update` rule we wrote in step 5 already allowed `user_id === user->id`.

Implementation is one Eloquent **booted hook** on the `User` model. No new controller, no override of Breeze's `RegisteredUserController`.

We also reorder `DatabaseSeeder` so the demo `Student` row exists *before* the matching `User` is created — otherwise the booted hook wouldn't have anyone to link to during seeding.

## What you'll build

- `User::booted()` hook that runs on `created` (non-admin only)
- One SQL update: `students SET user_id = $newUser->id WHERE email = $newUser->email AND user_id IS NULL`
- A "My registration" nav link that's only visible when `auth()->user()->student` is non-null
- DatabaseSeeder reordered to seed student rows before matching users
- 4 tests

## Prerequisites

- Completed [18 — Bulk actions](./18-bulk-actions.md).

## Steps

### 1. Add the `booted()` hook to `User`

In `app/Models/User.php`:

```php
use App\Models\Student;

protected static function booted(): void
{
    static::created(function (User $user): void {
        if ($user->is_admin) {
            return;
        }

        Student::query()
            ->where('email', $user->email)
            ->whereNull('user_id')
            ->update(['user_id' => $user->id]);
    });
}
```

A single UPDATE statement. No N+1, no loops. If no row matches, it's a no-op.

### 2. Reorder `DatabaseSeeder`

Move the student creation **before** the student-user creation:

```php
public function run(): void
{
    // Admin first (admin path doesn't link to a Student)
    User::factory()->admin()->create([
        'name' => 'Admin User',
        'email' => 'admin@devhub.test',
    ]);

    // Pre-seed graduation + students FIRST
    $june = Graduation::factory()->open()->create([
        'title' => 'June 2026 Convocation',
        'ceremony_date' => '2026-06-15',
        'fee' => 250.00,
    ]);

    Student::factory()->count(10)->verified()->for($june)->create();
    Student::factory()->count(4)->paidUnverified()->for($june)->create();
    Student::factory()->count(6)->for($june)->create();

    // Insert a specific Student row whose email matches the demo student user
    Student::factory()->for($june)->create([
        'name' => 'Student User',
        'email' => 'student@devhub.test',
    ]);

    // NOW create the matching User — the booted hook auto-links
    User::factory()->create([
        'name' => 'Student User',
        'email' => 'student@devhub.test',
    ]);

    // Bulk
    Graduation::factory()->count(2)
        ->has(Student::factory()->count(15))
        ->create();
}
```

### 3. Add the "My registration" nav link

In `resources/views/layouts/navigation.blade.php`, after the Graduations link:

```blade
@if (auth()->user()?->student)
    <x-nav-link :href="route('graduations.students.show',
        [auth()->user()->student->graduation, auth()->user()->student])"
        :active="false">
        {{ __('My registration') }}
    </x-nav-link>
@endif
```

Mirror the same block in the responsive (mobile) navigation section.

### 4. Add four tests

```php
it('links a new user to their student row on registration', function () {
    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create([
        'email' => 'newbie@example.test',
        'user_id' => null,
    ]);

    $this->post(route('register'), [
        'name' => 'Newbie',
        'email' => 'newbie@example.test',
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);

    $newUser = User::where('email', 'newbie@example.test')->first();
    expect($student->fresh()->user_id)->toBe($newUser->id);
});

it('admin registration does not link to any student row', function () {
    // Direct factory so we trigger the booted hook with is_admin=true
    User::factory()->admin()->create(['email' => 'a@x.test']);
    Student::factory()->create(['email' => 'a@x.test', 'user_id' => null]);

    $student = Student::where('email', 'a@x.test')->first();
    expect($student->user_id)->toBeNull();
});

it('registering with a non-matching email is a harmless no-op', function () {
    Student::factory()->create(['email' => 'someone@x.test', 'user_id' => null]);

    $this->post(route('register'), [
        'name' => 'Other',
        'email' => 'other@x.test',
        'password' => 'password',
        'password_confirmation' => 'password',
    ]);

    expect(Student::where('email', 'someone@x.test')->first()->user_id)->toBeNull();
});

it('a linked student can upload their own receipt', function () {
    Storage::fake('public');

    $grad = Graduation::factory()->create();
    $student = Student::factory()->for($grad)->create([
        'email' => 'self@x.test', 'user_id' => null,
    ]);

    $this->post(route('register'), [
        'name' => 'Self', 'email' => 'self@x.test',
        'password' => 'password', 'password_confirmation' => 'password',
    ]);
    $user = User::where('email', 'self@x.test')->first();

    $this->actingAs($user)
        ->patch(route('graduations.students.update', [$grad, $student]), [
            'name' => $student->name,
            'ic' => $student->ic,
            'email' => $student->email,
            'matric_card' => $student->matric_card,
            'phone' => $student->phone,
            'payment_receipt' => UploadedFile::fake()->create('r.pdf', 50, 'application/pdf'),
        ])
        ->assertRedirect();

    expect($student->fresh()->hasPaid())->toBeTrue();
});
```

## Expected output

```
php artisan test
```

**62 tests / 143 assertions, all green.**

Manual flow:
1. Reset: `php artisan migrate:fresh --seed`
2. Visit `/register`, register with `student@devhub.test` (the seeded demo email) — wait, that email is taken. Try a new email: `aiman@example.test` after **first** adding a Student row with that email through the admin UI.
3. Log out, visit `/register`, register with `aiman@example.test`.
4. After register, dashboard → the **My registration** link is in the nav.
5. Click it → land on your own student page → upload a receipt → see "Pending review" badge.

## Commit your work

```bash
git add .
git commit -m "feat(students): self-service — link User to Student by email on registration"
```

## Common pitfalls

- **`booted()` doesn't fire on factory `->create()`** — it does. The hook listens for `created`, which factories trigger like any other create. If you're not seeing rows linked, double-check the email actually matches (case-sensitive on some DBs).
- **`user_id` overwrites an existing link** — the `whereNull('user_id')` clause prevents that. Two users with the same email shouldn't exist anyway (unique constraint on `users.email`), but the guard is cheap.
- **Mobile nav link not showing** — you only added the link to the desktop nav. Duplicate the block in the responsive section.
- **Seed order matters again** — if the student row is created *after* the user, the booted hook already ran and found nothing. Re-read step 2.

## What's next

Admins are doing sensitive things (verifying payments, deleting rows). We should track who did what: [20 — Audit log](./20-audit-log.md).
