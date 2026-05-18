# 21 — Email the student when payment is verified

> **Commit:** `29af9b9` — *feat(notify): email student when payment is verified*

## Why this step?

Verification is the moment the student wants to know about. We send an email — "Your payment for *June 2026 Convocation* on *15 June 2026* (RM 250.00) has been received. See you there." — both for single verifies and for each row in a bulk verify.

Subtle design choice: **route the notification through the Student's own `email` column**, not through the linked User's email. Why? Because admins have the Student row from day one (since import), but the User account may not exist yet at verification time. The Student email is the canonical contact.

We do this by adding the `Notifiable` trait to the `Student` model and calling `$student->notify(...)`.

## What you'll build

- `PaymentVerified` notification (mail channel)
- `Notifiable` trait on `Student`
- Both `verify()` and `bulk()` (verify branch) dispatch the notification
- Flash message tail: `Email sent to me@example.test`
- 2 tests using `Notification::fake()`

## Prerequisites

- Completed [20 — Audit log](./20-audit-log.md).

## Steps

### 1. Generate the notification

```bash
php artisan make:notification PaymentVerified
```

### 2. Fill in the notification

```php
<?php

declare(strict_types=1);

namespace App\Notifications;

use App\Models\Student;
use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class PaymentVerified extends Notification
{
    use Queueable;

    public function __construct(public Student $student)
    {
    }

    public function via(object $notifiable): array
    {
        return ['mail'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        $graduation = $this->student->graduation;

        return (new MailMessage())
            ->subject('Your payment is verified — ' . $graduation->title)
            ->greeting("Hi {$this->student->name},")
            ->line("Your payment for **{$graduation->title}** has been received.")
            ->line('Ceremony date: ' . $graduation->ceremony_date->format('d M Y'))
            ->line('Fee paid: RM ' . number_format((float) $graduation->fee, 2))
            ->line('See you there!');
    }
}
```

### 3. Add `Notifiable` to `Student`

```php
use Illuminate\Notifications\Notifiable;

class Student extends Model
{
    use HasFactory;
    use HasUuids;
    use Notifiable;          // ← new

    // ...
}
```

Eloquent's `Notifiable` looks for a `routeNotificationFor*` method, falling back to the `email` column by default. Our column is already named `email`, so no override needed.

### 4. Dispatch from `verify()`

```php
use App\Notifications\PaymentVerified;

public function verify(Graduation $graduation, Student $student): RedirectResponse
{
    $this->authorize('verify', $student);

    $student->update(['verified_at' => now()]);
    Audit::record('verified', $student);
    $student->notify(new PaymentVerified($student));

    return redirect()
        ->route('graduations.students.show', [$graduation, $student])
        ->with('status', "Payment verified. Email sent to {$student->email}.");
}
```

### 5. Dispatch from the `bulk()` verify branch

In the verify branch (skip students with no receipt — both for verification *and* for the notification):

```php
if ($action === 'verify') {
    $toVerify = (clone $scope)->whereNotNull('payment_receipt')->get();
    $skipped = (clone $scope)->whereNull('payment_receipt')->count();

    foreach ($toVerify as $student) {
        $student->update(['verified_at' => now()]);
        Audit::record('verified', $student, ['via' => 'bulk']);
        $student->notify(new PaymentVerified($student));
    }

    $tail = $toVerify->isNotEmpty() ? ' Email notifications sent.' : '';

    return redirect()->route('graduations.show', $graduation)
        ->with('status', "Verified {$toVerify->count()}. Skipped {$skipped} (no receipt).{$tail}");
}
```

### 6. Add two tests

```php
use Illuminate\Support\Facades\Notification;
use App\Notifications\PaymentVerified;

it('sends an email to the student on single verify', function () {
    Notification::fake();

    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $student = Student::factory()->paidUnverified()->for($grad)->create();

    $this->actingAs($admin)
        ->patch(route('graduations.students.verify', [$grad, $student]));

    Notification::assertSentTo($student, PaymentVerified::class);
});

it('sends emails on bulk verify, skipping no-receipt students', function () {
    Notification::fake();

    $admin = User::factory()->admin()->create();
    $grad = Graduation::factory()->create();
    $paid = Student::factory()->for($grad)->paidUnverified()->create();
    $unpaid = Student::factory()->for($grad)->create();   // no receipt

    $this->actingAs($admin)
        ->post(route('graduations.students.bulk', $grad), [
            'action' => 'verify',
            'ids' => [$paid->uuid, $unpaid->uuid],
        ]);

    Notification::assertSentTo($paid, PaymentVerified::class);
    Notification::assertNotSentTo($unpaid, PaymentVerified::class);
});
```

### 7. Configure mail (local dev)

For local development, set `.env`:

```env
MAIL_MAILER=log
```

This writes emails to `storage/logs/laravel.log` instead of trying to deliver. Inspect the log to see the rendered HTML email.

For production / staging you'd point at SES, Mailgun, or a Mailpit instance. None of that affects the code.

## Expected output

```
php artisan test
```

**68 tests / 161 assertions, all green.**

Manual smoke:
1. Verify a student in the UI → flash message reads `Payment verified. Email sent to <email>.`
2. Open `storage/logs/laravel.log` → you'll find the rendered email near the bottom.

## Commit your work

```bash
git add .
git commit -m "feat(notify): email student when payment is verified"
```

## Common pitfalls

- **`Trying to access property notify on null`** — you didn't import `Notifiable` or you imported the wrong one (`Illuminate\Notifications\Notifiable`, not the Auth-foundation one).
- **Email sent to the linked `User` instead of the `Student`** — that's only true if Student doesn't have its own `email` column, or if you call `$student->user->notify(...)`. Always call on `$student` directly.
- **Bulk verify sends an email for no-receipt students** — you forgot to filter to `whereNotNull('payment_receipt')` before the loop.
- **Job sent but no log entry** — your `MAIL_MAILER` is `smtp` with a real host that's silently failing. Switch to `log` for local debugging.

## What's next

The student experience is complete. Time to improve the admin overview: [22 — Dashboard stats](./22-dashboard.md).
