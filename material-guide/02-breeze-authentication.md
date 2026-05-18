# 02 — Breeze authentication + Pest

> **Commit:** `c10291e` — *feat(auth): install Breeze Blade + Pest auth scaffolding*

## Why this step?

Every web app needs login. Writing the auth flow by hand (controllers, views, password hashing, email verification) takes a day. Laravel Breeze does it in two commands and gives us:

- Register / login / logout
- Password reset by email
- Email verification
- Profile edit page

We also install **Pest** instead of PHPUnit because Pest has a friendlier syntax (`it()`, `expect()`) and Breeze can scaffold tests for both.

## What you'll build

- `/register`, `/login`, `/forgot-password`, `/reset-password`, `/verify-email` routes
- `/dashboard` (protected) and `/profile` (edit your account)
- Tailwind CSS 4 wired through Vite
- Alpine.js for tiny client-side interactivity
- A `tests/` directory wired for Pest

## Prerequisites

- Completed [01 — Laravel scaffold](./01-laravel-scaffold.md).

## Steps

### 1. Install Breeze as a dev dependency

```bash
composer require laravel/breeze --dev
```

`--dev` because we only need the scaffolder during setup. Once views/controllers are generated we don't need the package at runtime.

### 2. Run the Breeze installer with the Blade + Pest preset

```bash
php artisan breeze:install blade --pest
```

This:

- Publishes auth controllers (`RegisteredUserController`, `AuthenticatedSessionController`, etc.)
- Publishes Blade views under `resources/views/auth/`
- Adds Tailwind 4 + Vite config
- Replaces PHPUnit with Pest
- Wires the routes file at `routes/auth.php`

When prompted, accept the defaults.

### 3. Install JS dependencies and build assets

```bash
npm install
npm run build
```

`npm run build` compiles Tailwind + Alpine into `public/build/`. While developing you'll usually run `npm run dev` instead, which watches files and rebuilds on save.

### 4. Re-add `resources/js/bootstrap.js`

Laravel 13 stopped shipping `bootstrap.js` by default, but Breeze still references it. If `npm run build` complains about a missing file, create it:

```js
// resources/js/bootstrap.js
import axios from 'axios';
window.axios = axios;
window.axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest';
```

### 5. Run the new auth migrations

```bash
php artisan migrate
```

Breeze doesn't add new tables (it reuses `users`), but `migrate` ensures everything is in sync.

### 6. Run the test suite

```bash
php artisan test
```

You should see ~24 tests passing — Breeze ships its own auth tests so you have working coverage from day one.

## Expected output

Start dev mode in two terminals:

```bash
# terminal 1
php artisan serve

# terminal 2
npm run dev
```

Open `http://localhost:8000` and walk through:

1. Click **Register** → fill in name, email, password → submit → redirected to `/dashboard`.
2. Click your name (top right) → **Log Out** → redirected to `/`.
3. Click **Log in** → enter the credentials you just registered → redirected to `/dashboard`.
4. Visit `/profile` → you can edit your name / email / password and delete your account.

`php artisan route:list` should show roughly 20 routes (welcome, dashboard, login, register, password reset, verify-email, profile).

## Commit your work

```bash
git add .
git commit -m "feat(auth): install Breeze Blade + Pest auth scaffolding"
```

## Common pitfalls

- **`npm run build` fails on `bootstrap.js`** — see step 4 above.
- **Login form looks unstyled** — you forgot `npm run build` or `npm run dev`. Tailwind must compile.
- **`vite: command not found`** — `npm install` didn't run. Re-run it from the project root.
- **Tests fail with `RefreshDatabase` errors** — your `.env.testing` (if any) may point at a different DB. By default Breeze uses an in-memory SQLite for tests, which is fine.

## What's next

You can log in but every user is identical. Next we add a role flag and UUID routing: [03 — User roles + UUID](./03-user-roles-uuid.md).
