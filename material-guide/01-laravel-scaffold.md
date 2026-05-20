# 01 — Laravel 13 scaffold with SQLite + MY locale

> **Commit:** `04bbd59` — *chore: initial Laravel 13 scaffold with SQLite + MY locale*

## Why this step?

We need a clean Laravel 13 project to build on. Two early decisions matter for the rest of the course:

1. **SQLite for local dev** — zero-config database, lives in a single file, fast to reset. You don't need MySQL/Postgres installed.
2. **`Asia/Kuala_Lumpur` timezone + `ms_MY` Faker locale** — seeded data will look realistic for Malaysian students (names, phone numbers, IC patterns), and timestamps in your DB will match your wall clock.

Setting these in step 1 is much easier than retrofitting them in step 10.

## What you'll build

- A fresh Laravel 13 project named `graduation-system/`
- SQLite as the default DB connection
- `Asia/Kuala_Lumpur` timezone, `ms_MY` faker locale
- The default `welcome` page accessible at `http://localhost:8000`

## Prerequisites

- [00 — Prerequisites](./00-prerequisites.md) completed.

## Steps

### 1. Create the project with the Livewire starter kit

From your training root directory:

```bash
laravel new graduation-system --livewire --pest
cd graduation-system
```

This downloads Laravel 13 + the **[Livewire starter kit](https://laravel.com/docs/13.x/starter-kits#livewire)** (Fortify auth, Livewire 4, Flux UI sidebar shell, dark mode), installs dependencies via `composer install`, picks **Pest** as the test runner, and copies `.env.example` to `.env`.

> If you'd rather use the interactive prompt: run `laravel new graduation-system` and pick **Livewire** at the starter-kit prompt, then **Pest** at the testing-framework prompt.

### 2. Initialize git

```bash
git init
git add .
git commit -m "chore: initial Laravel 13 scaffold"
```

You'll commit after every step from here on — this trains the muscle memory.

### 3. Switch the database to SQLite

Edit `.env`. Change the DB block to:

```env
DB_CONNECTION=sqlite
# DB_HOST=127.0.0.1
# DB_PORT=3306
# DB_DATABASE=laravel
# DB_USERNAME=root
# DB_PASSWORD=
```

Comment out (or delete) the host/port/database/user/password lines — SQLite doesn't need them.

### 4. Create the SQLite database file

```bash
touch database/database.sqlite
```

Laravel will write here automatically.

### 5. Generate the app key (if not already set)

```bash
php artisan key:generate
```

Confirm `APP_KEY=base64:...` is now populated in `.env`.

### 6. Set the timezone and faker locale

Edit `config/app.php`:

```php
'timezone' => 'Asia/Kuala_Lumpur',

// ... a few lines down ...

'faker_locale' => 'ms_MY',
```

### 7. Run the default migrations

```bash
php artisan migrate
```

This creates the default `users`, `password_reset_tokens`, `sessions`, and `cache` tables in your SQLite file.

### 8. Smoke test

```bash
php artisan serve
```

Visit `http://localhost:8000`. You should see the default Laravel welcome page.

## Expected output

- `php artisan --version` → `Laravel Framework 13.x.x`
- `php artisan migrate:status` → shows 4–5 ran migrations
- `php artisan tinker` and run `now()->timezone->getName();` → `'Asia/Kuala_Lumpur'`
- `php artisan tinker` and run `fake()->name();` → looks like a Malaysian name (e.g. `Encik Aiman bin Yusof`)
- Browser at `localhost:8000` shows the welcome page

## Commit your work

```bash
git add .
git commit -m "chore: initial Laravel 13 scaffold with SQLite + MY locale"
```

## Common pitfalls

- **`composer create-project` fails with memory errors** — run `COMPOSER_MEMORY_LIMIT=-1 composer create-project ...` to lift the limit.
- **SQLite file not found** — Laravel reads the *absolute* path. If you moved the project, delete and re-`touch database/database.sqlite`.
- **Welcome page 500s** — check `storage/logs/laravel.log`. Usually it's `APP_KEY` not generated or `storage/` not writable.

## What's next

The project is up with the Livewire starter kit already in place. Take a tour of what came pre-wired (register/login/dashboard/settings + Pest tests): [02 — Livewire starter kit tour](./02-livewire-starter-kit.md).
