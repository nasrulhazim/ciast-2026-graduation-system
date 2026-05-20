# 02 — Laravel Livewire starter kit

> **Commit:** `c10291e` — *feat(auth): install Livewire starter kit (Fortify + Flux UI + Pest)*

## Why this step?

Every web app needs login. Writing the auth flow by hand (controllers, views, password hashing, email verification) takes a day. The **[Laravel Livewire starter kit](https://laravel.com/docs/13.x/starter-kits#livewire)** gives us a fully wired account system the moment the project is created, including:

- Register / login / logout
- Password reset by email
- Email verification
- Password confirmation flow for sensitive actions
- Two-factor authentication
- Passkey (WebAuthn) support
- Account settings page (profile, password, appearance, security)
- A dark-mode-aware **Flux UI** sidebar shell at `<x-layouts::app>`
- **Pest** as the default test runner (no PHPUnit migration needed)

Under the hood the kit pulls in **Laravel Fortify** for the auth backend, **Livewire 4** for the reactive Volt components, and **Flux UI** for the design system primitives (`<flux:sidebar>`, `<flux:menu>`, `<flux:profile>`, etc.). You don't need to learn any of those internals in this step — just appreciate what's already there.

## What you'll build

- **None of it.** This step is a tour. You're already standing on a fully working auth system because step 01 chose Livewire when prompted by `laravel new`.
- We confirm everything works, then commit the scaffolded files.

## Prerequisites

- Completed [01 — Laravel scaffold](./01-laravel-scaffold.md) — and you picked **Livewire** at the starter-kit prompt.

> **Didn't pick Livewire in step 01?** Delete `graduation-system/` and re-run `laravel new graduation-system --livewire --pest`. The `--livewire` flag skips the interactive prompt and the `--pest` flag picks Pest over PHPUnit. Don't try to "upgrade" a vanilla Laravel project to the Livewire kit — the kit lays down dozens of files that aren't a composer package you can require later.

## Steps

### 1. Install JS dependencies and build assets

```bash
npm install
npm run build
```

`npm run build` compiles Tailwind CSS 4 + Flux UI into `public/build/`. While developing you'll usually run `npm run dev` instead, which watches files and rebuilds on save.

### 2. Run the migrations that came with the kit

```bash
php artisan migrate
```

The kit ships migrations for `users`, `password_reset_tokens`, `sessions`, **`two_factor_auth_credentials`**, and **`passkeys`**. None of them need editing.

### 3. Run the test suite

```bash
php artisan test
```

You should see a green run — the kit ships Pest feature tests for register / login / logout / password reset / email verification / two-factor / passkey flows. Working coverage from day one.

### 4. Confirm the route map

```bash
php artisan route:list
```

You'll see ~30 routes already wired up. Notable ones:

| Method | URI | Name | Why |
|---|---|---|---|
| `GET`  | `/`                       | `home`           | Welcome page |
| `GET`  | `/dashboard`              | `dashboard`      | Protected landing page |
| `GET`  | `/login`                  | `login`          | Login form (Volt component) |
| `POST` | `/login`                  | —                | Login submit (Fortify) |
| `GET`  | `/register`               | `register`       | Register form (Volt) |
| `POST` | `/logout`                 | `logout`         | Logout submit |
| `GET`  | `/settings/profile`       | `profile.edit`   | Profile + email |
| `GET`  | `/settings/appearance`    | `appearance.edit`| Dark / light toggle |
| `GET`  | `/settings/security`      | `security.edit`  | Password / 2FA / passkeys |
| `GET`  | `/forgot-password`        | `password.request` | Password reset request |
| `POST` | `/forgot-password`        | `password.email` | Email reset link |
| `GET`  | `/reset-password/{token}` | `password.reset` | Reset form |

Settings pages live under `/settings/*` and are *Livewire components* (in `app/Livewire/Settings/`) rather than blade-only views. You won't need to touch them in this course.

### 5. Walk through the UI

Start dev mode in two terminals:

```bash
# terminal 1
php artisan serve

# terminal 2
npm run dev
```

Open `http://localhost:8000` and walk through:

1. Click **Register** → fill in name, email, password → submit → redirected to `/dashboard`.
2. Click your initials avatar (sidebar bottom-left on desktop, top-right header on mobile) → **Log Out** → redirected to `/`.
3. Click **Log in** → enter the credentials you just registered → redirected to `/dashboard`.
4. Open the avatar menu → **Settings** → see the three tabs **Profile**, **Appearance** (try toggling dark mode), **Security**.

### 6. Confirm the layout component the rest of the course uses

Open `resources/views/layouts/app.blade.php`:

```blade
<x-layouts::app.sidebar :title="$title ?? null">
    <flux:main>
        {{ $slot }}
    </flux:main>
</x-layouts::app.sidebar>
```

Every page from step 9 onward will wrap its content with `<x-layouts::app :title="...">`. The component handles the entire Flux sidebar, dark-mode toggle, user menu, and toast container — you don't need to think about chrome again.

## Expected output

```bash
php artisan test         # all green (~25 tests)
php artisan route:list   # ~30 routes
```

In the browser: `/register`, `/login`, `/dashboard`, `/settings/profile`, `/settings/appearance`, `/settings/security` all load. Dark mode toggles and persists.

## Commit your work

```bash
git add .
git commit -m "feat(auth): install Livewire starter kit (Fortify + Flux UI + Pest)"
```

> If you re-ran `laravel new` to switch to the Livewire kit, this commit will include the entire kit scaffolding. That's fine — it's your starting checkpoint.

## Common pitfalls

- **`npm run build` fails with `flux` module not found** — `npm install` didn't finish, or you skipped it. Re-run from the project root.
- **Login form looks unstyled** — you forgot `npm run build` or `npm run dev`. Tailwind + Flux must compile.
- **`vite: command not found`** — same root cause: `npm install` didn't run.
- **`Class "Laravel\Fortify\Fortify" not found`** — your `composer install` didn't run during `laravel new`. Run `composer install` manually.
- **Tried to `composer require laravel/livewire-starter-kit`** — that's not how starter kits work. They're project templates, not installable packages. If you ended up here, start over with `laravel new graduation-system --livewire`.
- **"I want to use Breeze instead"** — that's a different course. The rest of this guide assumes Livewire kit conventions: `<x-layouts::app>`, Flux sidebar, no `<x-text-input>` / `<x-primary-button>` Blade components.

## What's next

You can log in but every user is identical. Next we add a role flag and UUID routing: [03 — User roles + UUID](./03-user-roles-uuid.md).
