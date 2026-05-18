# 10 — Public storage symlink

> **Commit:** `752239c` — *chore(storage): create public storage symlink for receipt uploads*

## Why this step?

When the student edit form saves a receipt, Laravel writes it to:

```
storage/app/public/receipts/<random>.pdf
```

But your web server only serves files from `public/`. Without a bridge between the two, `Storage::url($student->payment_receipt)` would give a URL that 404s.

The standard Laravel fix is a **symlink**: `public/storage` → `storage/app/public`. After that, files inside `storage/app/public/receipts/foo.pdf` become reachable at `/storage/receipts/foo.pdf`.

## What you'll build

A symbolic link at `public/storage` pointing to `storage/app/public`.

This step is *not* a code change — it's a runtime command Laravel ships an Artisan helper for. The symlink itself is gitignored, so anyone who clones your repo needs to run this command once.

## Prerequisites

- Completed [09 — Blade views](./09-blade-views.md).

## Steps

### 1. Run the helper

```bash
php artisan storage:link
```

Output:

```
INFO  The [public/storage] link has been connected to [storage/app/public].
```

### 2. Verify

```bash
ls -la public/storage
```

You should see a symlink arrow:

```
lrwxr-xr-x  1 you  staff  ...  public/storage -> /full/path/to/storage/app/public
```

### 3. Smoke-test in the browser

Upload a receipt for any seeded student:

1. `php artisan serve` + `npm run dev`
2. Log in as admin, click on a graduation, click on any student.
3. Click **Edit**, choose a PDF or PNG file in the receipt input, **Save**.
4. Click the "view" link next to "Current receipt" — the file should open in a new tab.

## Expected output

- `ls -la public/storage` shows a symlink.
- Uploading a receipt and clicking "view" returns the file (not a 404).
- `Storage::url('receipts/<file>')` returns `/storage/receipts/<file>`.

## Commit your work

```bash
git add .
git commit -m "chore(storage): create public storage symlink for receipt uploads"
```

> Note: the symlink itself is gitignored by Laravel's default `.gitignore`. Your commit captures *only* the fact you ran the command (no files actually change). If you'd like to record the intent more explicitly, add a one-line note to your project README; the trainer's commit is symbolic — it marks the checkpoint.

## Common pitfalls

- **Permission denied on `public/`** — your shell user must be able to write inside `public/`. On macOS / Linux dev machines this is usually fine; on shared servers you may need `sudo`.
- **Already exists** — if `public/storage` exists but isn't a symlink (e.g. someone made a folder by mistake), delete it first: `rm -rf public/storage && php artisan storage:link`.
- **Windows without dev mode** — Windows requires Developer Mode or admin rights to create symlinks. The error mentions `symlink(): Operation not permitted`. Either enable Developer Mode, or use WSL.

## What's next

We have a working end-to-end app. Time to lock the behaviour in with tests so further changes don't regress what we just built: [11 — Pest feature tests](./11-pest-tests.md).
