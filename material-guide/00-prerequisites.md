# 00 — Prerequisites

> Before you write a single line of code, make sure the toolchain on your laptop is ready.

## Why this step?

Half of all "Laravel doesn't work on my machine" issues are a wrong PHP version, a missing extension, or a stale Node. We pin the toolchain up front so the next 23 steps are about the framework, not the environment.

## What you'll install

| Tool | Version | Why |
|---|---|---|
| PHP | 8.3 or higher | Laravel 13 requires PHP 8.3+ |
| Composer | 2.x | Manages PHP dependencies |
| Node.js | 20 LTS or higher | Required by Vite (Tailwind build) |
| NPM | bundled with Node | Manages JS dependencies |
| SQLite | 3.35+ | Default DB for this project |
| Git | 2.x | Required for committing your work |
| A code editor | VS Code / PhpStorm | Your choice |

## Steps

### 1. Check what's already installed

```bash
php -v          # must show 8.3.x or higher
composer -V     # must show Composer 2.x
node -v         # must show v20.x or higher
npm -v
git --version
```

If any of these is missing or the wrong version, install / upgrade it before continuing.

### 2. Verify required PHP extensions

```bash
php -m | grep -iE 'pdo|sqlite|mbstring|openssl|tokenizer|xml|ctype|fileinfo|gd|curl|zip'
```

You should see (at minimum):

- `pdo_sqlite`
- `mbstring`
- `openssl`
- `tokenizer`
- `xml`
- `ctype`
- `fileinfo`
- `curl`
- `zip`

On macOS with Homebrew PHP these are all enabled by default. On Linux you may need:

```bash
sudo apt install php8.3-{sqlite3,mbstring,xml,curl,zip,gd}   # Debian/Ubuntu
```

### 3. Pick a working directory

```bash
mkdir -p ~/Trainings/2026/ciast
cd ~/Trainings/2026/ciast
```

This is where your `graduation-system/` will live.

### 4. Configure git identity (one time)

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

### 5. (Optional but recommended) Install Laravel installer

```bash
composer global require laravel/installer
```

The guide uses `composer create-project` instead (works without the installer), so this is optional.

## Expected output

When all of the above pass, you should be able to run:

```bash
php -r 'echo "PHP " . PHP_VERSION . PHP_EOL;'   # PHP 8.3.x
composer about                                    # Composer is up to date
node -p "process.versions.node"                  # v20.x or higher
sqlite3 --version                                 # 3.35 or higher
```

## Common pitfalls

- **Multiple PHP versions** (Homebrew + Valet + system) — `php -v` may not be the one your shell uses. Run `which php` and confirm it points to the 8.3+ install.
- **Node bundled with Linux distro** is often years old. Use [nvm](https://github.com/nvm-sh/nvm) instead of the apt package.
- **No write permission on `/Trainings`** — make sure the directory is inside your home folder, not `/opt` or `/var`.

## What's next

You're ready. Continue to [01 — Laravel scaffold](./01-laravel-scaffold.md).
