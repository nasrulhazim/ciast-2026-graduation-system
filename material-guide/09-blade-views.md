# 09 — Blade views + nav link

> **Commit:** `dbbcbe0` — *feat(ui): add graduation + student Blade views + nav link*

## Why this step?

We have routes, controllers, and authorization. We're one piece away from a working app: the actual HTML.

Six Blade views — four for graduations, two for students — all wrapped in the Livewire starter kit's **`<x-layouts::app>`** component (which wraps the **Flux sidebar shell** already wired up in step 02). Forms use plain Tailwind utility classes with `@error` border swaps — the starter kit doesn't ship Breeze-style `<x-text-input>` / `<x-primary-button>` blade components, so we write the inputs out directly.

Three patterns to internalize from this step:

1. **`@can('ability', $model)` in Blade** — hide UI affordances students would 403 on anyway. The button never renders for non-admins, so it's not "client-side security theatre" — it's a UX thing.
2. **File-upload forms need `enctype="multipart/form-data"`** — without it, the file disappears between browser and Laravel.
3. **Status badges as a single `$statusStyles` lookup array** — light + dark mode variants in one place, no `if/elseif` ladder for the colour.

## What you'll build

- `resources/views/graduations/index.blade.php` — paginated list with status badges
- `resources/views/graduations/create.blade.php` — form (admin only)
- `resources/views/graduations/edit.blade.php` — form + archive button
- `resources/views/graduations/show.blade.php` — detail + nested student table
- `resources/views/students/show.blade.php` — detail + verify button
- `resources/views/students/edit.blade.php` — form with file upload
- Updated `resources/views/layouts/app/sidebar.blade.php` — adds "Graduations" link to the Flux sidebar (works for both desktop and the collapsed mobile menu)

## Prerequisites

- Completed [08 — Routes](./08-routes.md).

## Steps

### 1. Add the navigation link

This project uses the **Livewire starter kit** with **Flux UI**, not Breeze. The nav lives in a single Flux sidebar component that handles both the desktop sidebar and the mobile collapsed menu — no separate desktop/mobile blocks to keep in sync.

Open `resources/views/layouts/app/sidebar.blade.php` and find the existing **Platform** group containing the Dashboard item:

```blade
<flux:sidebar.nav>
    <flux:sidebar.group :heading="__('Platform')" class="grid">
        <flux:sidebar.item icon="home" :href="route('dashboard')" :current="request()->routeIs('dashboard')" wire:navigate>
            {{ __('Dashboard') }}
        </flux:sidebar.item>
    </flux:sidebar.group>
</flux:sidebar.nav>
```

Add a sibling `flux:sidebar.item` for graduations:

```blade
<flux:sidebar.item icon="academic-cap" :href="route('graduations.index')" :current="request()->routeIs('graduations.*')" wire:navigate>
    {{ __('Graduations') }}
</flux:sidebar.item>
```

The `:current="request()->routeIs('graduations.*')"` highlights the link on any `graduations.index|create|show|edit` page. `wire:navigate` keeps navigation SPA-fast via Livewire's [navigate](https://livewire.laravel.com/docs/navigate) feature.

> If you started from a Breeze-based starter instead, your equivalent file is `resources/views/layouts/navigation.blade.php` — add an `<x-nav-link>` (and its `<x-responsive-nav-link>` mobile twin) pointing at `route('graduations.index')`.

### 2. `graduations/index.blade.php`

Every page in this app follows the same skeleton:

```blade
<x-layouts::app :title="__('Graduations')">
    <div class="flex h-full w-full flex-1 flex-col gap-4 rounded-xl">
        {{-- header row: title + primary action --}}
        {{-- flash status banner --}}
        {{-- main card --}}
        {{-- pagination links --}}
    </div>
</x-layouts::app>
```

The `:title` prop drives the `<title>` tag and the sidebar's "current page" highlight. The outer `flex flex-col gap-4` gives consistent vertical rhythm. Here's the index in full:

```blade
<x-layouts::app :title="__('Graduations')">
    <div class="flex h-full w-full flex-1 flex-col gap-4 rounded-xl">

        <div class="flex items-center justify-between">
            <div>
                <h1 class="text-lg font-semibold text-gray-900 dark:text-neutral-100">
                    Graduations
                </h1>
                <p class="text-sm text-gray-500 dark:text-neutral-400">
                    {{ $graduations->total() }} {{ Str::plural('graduation', $graduations->total()) }} on file.
                </p>
            </div>

            @can('create', App\Models\Graduation::class)
                <a href="{{ route('graduations.create') }}"
                   class="inline-flex items-center rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700">
                    + New graduation
                </a>
            @endcan
        </div>

        @if (session('status'))
            <div class="rounded-md border border-emerald-200 bg-emerald-50 px-4 py-2 text-sm text-emerald-700 dark:border-emerald-500/30 dark:bg-emerald-500/10 dark:text-emerald-300">
                {{ session('status') }}
            </div>
        @endif

        <div class="overflow-hidden rounded-xl border border-neutral-200 bg-white shadow-sm dark:border-neutral-700 dark:bg-neutral-900">
            <table class="min-w-full divide-y divide-neutral-200 dark:divide-neutral-700">
                <thead class="bg-neutral-50 dark:bg-neutral-800/50">
                    <tr>
                        <th class="px-6 py-3 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Title</th>
                        <th class="px-6 py-3 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Date</th>
                        <th class="px-6 py-3 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Fee</th>
                        <th class="px-6 py-3 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Status</th>
                        <th class="px-6 py-3 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Students</th>
                        <th class="px-6 py-3 text-right text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Actions</th>
                    </tr>
                </thead>
                <tbody class="divide-y divide-neutral-200 dark:divide-neutral-700">
                    @php
                        $statusStyles = [
                            'draft'  => 'bg-slate-50 text-slate-700 ring-1 ring-slate-600/20 dark:bg-slate-500/10 dark:text-slate-300 dark:ring-slate-500/30',
                            'open'   => 'bg-emerald-50 text-emerald-700 ring-1 ring-emerald-600/20 dark:bg-emerald-500/10 dark:text-emerald-300 dark:ring-emerald-500/30',
                            'closed' => 'bg-amber-50 text-amber-700 ring-1 ring-amber-600/20 dark:bg-amber-500/10 dark:text-amber-300 dark:ring-amber-500/30',
                        ];
                    @endphp

                    @forelse ($graduations as $g)
                        <tr class="hover:bg-neutral-50 dark:hover:bg-neutral-800/40">
                            <td class="px-6 py-4 text-sm font-medium text-gray-900 dark:text-neutral-100">
                                <a href="{{ route('graduations.show', $g) }}" class="hover:text-indigo-600 dark:hover:text-indigo-400">
                                    {{ $g->title }}
                                </a>
                            </td>
                            <td class="px-6 py-4 text-sm text-gray-500 dark:text-neutral-400">
                                {{ $g->ceremony_date->format('d M Y') }}
                            </td>
                            <td class="px-6 py-4 text-sm text-gray-500 dark:text-neutral-400">
                                RM {{ number_format((float) $g->fee, 2) }}
                            </td>
                            <td class="px-6 py-4 text-sm">
                                <span class="inline-flex items-center rounded-full px-2 py-0.5 text-xs font-medium {{ $statusStyles[$g->status] }}">
                                    {{ ucfirst($g->status) }}
                                </span>
                            </td>
                            <td class="px-6 py-4 text-sm text-gray-500 dark:text-neutral-400">
                                {{ $g->students_count }}
                            </td>
                            <td class="px-6 py-4 text-right text-sm">
                                <a href="{{ route('graduations.show', $g) }}"
                                   class="text-indigo-600 hover:text-indigo-800 dark:text-indigo-400 dark:hover:text-indigo-300">
                                    View
                                </a>
                                @can('update', $g)
                                    <span class="text-neutral-300 dark:text-neutral-600 mx-2">|</span>
                                    <a href="{{ route('graduations.edit', $g) }}"
                                       class="text-gray-700 hover:text-gray-900 dark:text-neutral-300 dark:hover:text-neutral-100">
                                        Edit
                                    </a>
                                @endcan
                            </td>
                        </tr>
                    @empty
                        <tr>
                            <td colspan="6" class="px-6 py-12 text-center text-sm text-gray-500 dark:text-neutral-400">
                                No graduations yet.
                            </td>
                        </tr>
                    @endforelse
                </tbody>
            </table>
        </div>

        <div>{{ $graduations->links() }}</div>

    </div>
</x-layouts::app>
```

A few things to notice:

- **Status colours live in one `$statusStyles` array.** Each entry pairs a light-mode and dark-mode palette plus the ring (1px outer border) — single source of truth, easy to add new statuses.
- **`@can('update', $g)`** gates the `Edit` link per-row. Non-admins see only `View`.
- **`hover:bg-neutral-50 dark:hover:bg-neutral-800/40`** — every row needs a hover state in dark mode too.

### 3. `graduations/create.blade.php`

Forms use plain `<input>` / `<select>` elements with a long-but-consistent utility chain. The `@error('field') border-rose-500 @enderror` at the end of the class list swaps the border colour on validation failure.

A breadcrumb row across the top mirrors the existing `users` views so the kit stays cohesive:

```blade
<x-layouts::app :title="__('New Graduation')">
    <div class="flex h-full w-full flex-1 flex-col gap-4 rounded-xl">

        <div class="flex items-center justify-between">
            <div class="flex items-center gap-2 text-sm text-gray-500 dark:text-neutral-400">
                <a href="{{ route('graduations.index') }}"
                   class="hover:text-gray-700 dark:hover:text-neutral-200">
                    Graduations
                </a>
                <span aria-hidden="true">/</span>
                <span class="font-medium text-gray-900 dark:text-neutral-100">
                    New graduation
                </span>
            </div>

            <a href="{{ route('graduations.index') }}"
               class="inline-flex items-center rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm font-medium text-gray-700 hover:bg-gray-50 dark:border-neutral-700 dark:bg-neutral-900 dark:text-neutral-200 dark:hover:bg-neutral-800">
                Cancel
            </a>
        </div>

        <div class="overflow-hidden rounded-xl border border-neutral-200 bg-white shadow-sm dark:border-neutral-700 dark:bg-neutral-900">
            <div class="border-b border-neutral-200 px-6 py-5 dark:border-neutral-700">
                <h2 class="text-lg font-semibold text-gray-900 dark:text-neutral-100">
                    Add a new graduation
                </h2>
                <p class="mt-1 text-sm text-gray-500 dark:text-neutral-400">
                    Set the ceremony date and registration fee. Status controls whether students can register.
                </p>
            </div>

            <form method="POST" action="{{ route('graduations.store') }}" class="space-y-6 px-6 py-6">
                @csrf

                <div>
                    <label for="title" class="block text-sm font-medium text-gray-700 dark:text-neutral-200">
                        Title
                    </label>
                    <input type="text" id="title" name="title"
                           value="{{ old('title') }}" required autofocus
                           class="mt-1 block w-full rounded-md border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 dark:border-neutral-700 dark:bg-neutral-800 dark:text-neutral-100 @error('title') border-rose-500 @enderror" />
                    @error('title')
                        <p class="mt-1 text-sm text-rose-600 dark:text-rose-400">{{ $message }}</p>
                    @enderror
                </div>

                <div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
                    <div>
                        <label for="ceremony_date" class="block text-sm font-medium text-gray-700 dark:text-neutral-200">
                            Ceremony date
                        </label>
                        <input type="date" id="ceremony_date" name="ceremony_date"
                               value="{{ old('ceremony_date') }}" required
                               class="mt-1 block w-full rounded-md border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 dark:border-neutral-700 dark:bg-neutral-800 dark:text-neutral-100 @error('ceremony_date') border-rose-500 @enderror" />
                        @error('ceremony_date')
                            <p class="mt-1 text-sm text-rose-600 dark:text-rose-400">{{ $message }}</p>
                        @enderror
                    </div>

                    <div>
                        <label for="fee" class="block text-sm font-medium text-gray-700 dark:text-neutral-200">
                            Fee (RM)
                        </label>
                        <input type="number" id="fee" name="fee" step="0.01" min="0"
                               value="{{ old('fee') }}" required
                               class="mt-1 block w-full rounded-md border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 dark:border-neutral-700 dark:bg-neutral-800 dark:text-neutral-100 @error('fee') border-rose-500 @enderror" />
                        @error('fee')
                            <p class="mt-1 text-sm text-rose-600 dark:text-rose-400">{{ $message }}</p>
                        @enderror
                    </div>
                </div>

                <div>
                    <label for="status" class="block text-sm font-medium text-gray-700 dark:text-neutral-200">
                        Status
                    </label>
                    <select id="status" name="status"
                            class="mt-1 block w-full rounded-md border-gray-300 bg-white px-3 py-2 text-sm text-gray-900 shadow-sm focus:border-indigo-500 focus:ring-indigo-500 dark:border-neutral-700 dark:bg-neutral-800 dark:text-neutral-100">
                        @foreach (['draft', 'open', 'closed'] as $s)
                            <option value="{{ $s }}" @selected(old('status', 'draft') === $s)>
                                {{ ucfirst($s) }}
                            </option>
                        @endforeach
                    </select>
                </div>

                <div class="flex items-center justify-end gap-2 border-t border-neutral-200 pt-5 dark:border-neutral-700">
                    <a href="{{ route('graduations.index') }}"
                       class="inline-flex items-center rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm font-medium text-gray-700 hover:bg-gray-50 dark:border-neutral-700 dark:bg-neutral-900 dark:text-neutral-200 dark:hover:bg-neutral-800">
                        Cancel
                    </a>
                    <button type="submit"
                            class="inline-flex items-center rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700">
                        Create graduation
                    </button>
                </div>
            </form>
        </div>

    </div>
</x-layouts::app>
```

> **Pro tip — extract a Blade component when you find yourself copy-pasting input markup.** A few inputs is fine inline; once you're past a dozen, make `<x-form.input>` and stop repeating yourself. We deliberately leave it inline here so the pattern is visible.

### 4. `graduations/edit.blade.php`

Mirrors `create` with three differences:

1. **`@method('PATCH')`** under `@csrf` so the form routes to `update`.
2. **`old('title', $graduation->title)`** etc. so the form pre-fills the current values when the page loads.
3. **An Archive button** in the footer, gated by `@can('delete', $graduation)` — submitted via a *separate hidden form* so the Save form and the Delete form don't nest (browsers don't allow nested `<form>`):

```blade
{{-- inside the footer of the main edit form --}}
<div class="flex items-center justify-between border-t border-neutral-200 pt-5 dark:border-neutral-700">
    @can('delete', $graduation)
        <button type="button"
                onclick="document.getElementById('archive-graduation-form').submit();"
                class="inline-flex items-center rounded-md border border-rose-300 bg-white px-3 py-1.5 text-sm font-medium text-rose-700 hover:bg-rose-50 dark:border-rose-900/60 dark:bg-neutral-900 dark:text-rose-300 dark:hover:bg-rose-950/40">
            Archive
        </button>
    @else
        <span></span>
    @endcan

    <div class="flex items-center gap-2">
        <a href="{{ route('graduations.show', $graduation) }}"
           class="inline-flex items-center rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm font-medium text-gray-700 hover:bg-gray-50 dark:border-neutral-700 dark:bg-neutral-900 dark:text-neutral-200 dark:hover:bg-neutral-800">
            Cancel
        </a>
        <button type="submit"
                class="inline-flex items-center rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700">
            Save changes
        </button>
    </div>
</div>
{{-- end of main form --}}

@can('delete', $graduation)
    <form id="archive-graduation-form"
          method="POST"
          action="{{ route('graduations.destroy', $graduation) }}"
          onsubmit="return confirm('Archive {{ $graduation->title }}? Soft-deleted — recoverable but hidden.');"
          class="hidden">
        @csrf
        @method('DELETE')
    </form>
@endcan
```

The pattern (button outside the form referencing a hidden form by ID via `document.getElementById(...).submit()`) is the same one used by the existing `users` views — keep things consistent.

### 5. `graduations/show.blade.php`

Two stacked cards:

- **Header card** — title + `<dl>` with ceremony date, fee, status badge (same `$statusStyles` lookup as the index).
- **Students card** — nested table of `$graduation->students` with payment-status badges per row.

Key chunk for the students table:

```blade
<div class="px-6 py-5">
    <h3 class="text-sm font-semibold text-gray-900 dark:text-neutral-100">
        Students ({{ $graduation->students->count() }})
    </h3>

    <div class="mt-3 overflow-hidden rounded-lg border border-neutral-200 dark:border-neutral-700">
        <table class="min-w-full divide-y divide-neutral-200 dark:divide-neutral-700">
            <thead class="bg-neutral-50 dark:bg-neutral-800/50">
                <tr>
                    <th class="px-4 py-2 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Name</th>
                    <th class="px-4 py-2 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Matric</th>
                    <th class="px-4 py-2 text-left text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400">Payment</th>
                    <th class="px-4 py-2 text-right text-xs font-medium uppercase tracking-wide text-gray-500 dark:text-neutral-400"></th>
                </tr>
            </thead>
            <tbody class="divide-y divide-neutral-200 dark:divide-neutral-700">
                @forelse ($graduation->students as $student)
                    <tr class="hover:bg-neutral-50 dark:hover:bg-neutral-800/40">
                        <td class="px-4 py-2 text-sm font-medium text-gray-900 dark:text-neutral-100">
                            {{ $student->name }}
                        </td>
                        <td class="px-4 py-2 text-sm text-gray-500 dark:text-neutral-400">
                            {{ $student->matric_card }}
                        </td>
                        <td class="px-4 py-2 text-sm">
                            @if ($student->isVerified())
                                <span class="inline-flex items-center rounded-full bg-emerald-50 px-2 py-0.5 text-xs font-medium text-emerald-700 ring-1 ring-emerald-600/20 dark:bg-emerald-500/10 dark:text-emerald-300 dark:ring-emerald-500/30">
                                    Verified
                                </span>
                            @elseif ($student->hasPaid())
                                <span class="inline-flex items-center rounded-full bg-amber-50 px-2 py-0.5 text-xs font-medium text-amber-700 ring-1 ring-amber-600/20 dark:bg-amber-500/10 dark:text-amber-300 dark:ring-amber-500/30">
                                    Pending review
                                </span>
                            @else
                                <span class="inline-flex items-center rounded-full bg-slate-50 px-2 py-0.5 text-xs font-medium text-slate-700 ring-1 ring-slate-600/20 dark:bg-slate-500/10 dark:text-slate-300 dark:ring-slate-500/30">
                                    Not paid
                                </span>
                            @endif
                        </td>
                        <td class="px-4 py-2 text-right text-sm">
                            @can('view', $student)
                                <a href="{{ route('graduations.students.show', [$graduation, $student]) }}"
                                   class="text-indigo-600 hover:text-indigo-800 dark:text-indigo-400 dark:hover:text-indigo-300">
                                    View
                                </a>
                            @endcan
                        </td>
                    </tr>
                @empty
                    <tr>
                        <td colspan="4" class="px-4 py-6 text-center text-sm text-gray-500 dark:text-neutral-400">
                            No students registered for this graduation yet.
                        </td>
                    </tr>
                @endforelse
            </tbody>
        </table>
    </div>
</div>
```

Refer to the reference repo's full `graduations/show.blade.php` for the header dl and breadcrumb — same patterns as `create`/`edit`.

### 6. `students/show.blade.php`

Breadcrumb (`Graduations / {grad} / {student}`), a `<dl>` of personal details, and a payment block. The payment block toggles between three states and exposes the **Verify payment** button only when `@can('verify', $student)` is true:

```blade
<div class="border-t border-neutral-200 px-6 py-5 dark:border-neutral-700">
    <h3 class="text-sm font-semibold text-gray-900 dark:text-neutral-100">Payment</h3>

    <div class="mt-2 flex items-center gap-3">
        @if ($student->isVerified())
            <span class="inline-flex items-center rounded-full bg-emerald-50 px-2 py-0.5 text-xs font-medium text-emerald-700 ring-1 ring-emerald-600/20 dark:bg-emerald-500/10 dark:text-emerald-300 dark:ring-emerald-500/30">
                Verified
            </span>
            <span class="text-sm text-gray-500 dark:text-neutral-400">
                on {{ $student->verified_at->format('d M Y H:i') }}
            </span>
        @elseif ($student->hasPaid())
            <span class="inline-flex items-center rounded-full bg-amber-50 px-2 py-0.5 text-xs font-medium text-amber-700 ring-1 ring-amber-600/20 dark:bg-amber-500/10 dark:text-amber-300 dark:ring-amber-500/30">
                Pending review
            </span>
            <span class="text-sm text-gray-500 dark:text-neutral-400">
                paid {{ $student->paid_at->format('d M Y H:i') }}
            </span>
        @else
            <span class="inline-flex items-center rounded-full bg-slate-50 px-2 py-0.5 text-xs font-medium text-slate-700 ring-1 ring-slate-600/20 dark:bg-slate-500/10 dark:text-slate-300 dark:ring-slate-500/30">
                Not paid
            </span>
        @endif
    </div>

    @if ($student->payment_receipt)
        <p class="mt-3 text-sm">
            <a class="text-indigo-600 hover:text-indigo-800 dark:text-indigo-400 dark:hover:text-indigo-300"
               target="_blank"
               href="{{ Storage::url($student->payment_receipt) }}">
                View payment receipt
            </a>
        </p>
    @endif

    @can('verify', $student)
        <form method="POST"
              action="{{ route('graduations.students.verify', [$graduation, $student]) }}"
              class="mt-4">
            @csrf
            @method('PATCH')
            <button type="submit"
                    class="inline-flex items-center rounded-md bg-emerald-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-emerald-700">
                Verify payment
            </button>
        </form>
    @endcan
</div>
```

The Verify button is **emerald**, deliberately different from the indigo primary — verifying is the celebratory finish line, not just another save.

### 7. `students/edit.blade.php`

Personal details in a 2-column grid, then a separator, then the receipt upload block. Three things matter:

- **`enctype="multipart/form-data"`** on the `<form>` — without it, the file disappears between browser and Laravel.
- **`@method('PATCH')`** — the route is `PATCH /graduations/{graduation}/students/{student}` but browsers only send POST.
- **Native file input styled via `file:` Tailwind variants** — no Flux dependency.

```blade
<form method="POST"
      action="{{ route('graduations.students.update', [$graduation, $student]) }}"
      enctype="multipart/form-data"
      class="space-y-6 px-6 py-6">
    @csrf
    @method('PATCH')

    <div class="grid grid-cols-1 gap-6 sm:grid-cols-2">
        {{-- name / ic / email / matric_card / phone — each one a label + input + @error block --}}
    </div>

    <div class="border-t border-neutral-200 pt-5 dark:border-neutral-700">
        <label for="payment_receipt" class="block text-sm font-medium text-gray-700 dark:text-neutral-200">
            Payment receipt
        </label>
        <p class="mt-1 text-xs text-gray-500 dark:text-neutral-400">
            PDF / JPG / PNG, max 2 MB. Uploading a new file stamps the payment time automatically.
        </p>
        <input type="file" id="payment_receipt" name="payment_receipt"
               accept=".pdf,image/jpeg,image/png"
               class="mt-2 block w-full text-sm text-gray-700 file:mr-3 file:rounded-md file:border-0 file:bg-indigo-50 file:px-3 file:py-1.5 file:text-sm file:font-medium file:text-indigo-700 hover:file:bg-indigo-100 dark:text-neutral-300 dark:file:bg-indigo-500/10 dark:file:text-indigo-300" />
        @error('payment_receipt')
            <p class="mt-1 text-sm text-rose-600 dark:text-rose-400">{{ $message }}</p>
        @enderror

        @if ($student->payment_receipt)
            <p class="mt-2 text-sm text-gray-600 dark:text-neutral-400">
                Current receipt:
                <a class="text-indigo-600 hover:text-indigo-800 dark:text-indigo-400 dark:hover:text-indigo-300"
                   target="_blank"
                   href="{{ Storage::url($student->payment_receipt) }}">
                    view
                </a>
            </p>
        @endif
    </div>

    <div class="flex items-center justify-end gap-2 border-t border-neutral-200 pt-5 dark:border-neutral-700">
        <a href="{{ route('graduations.students.show', [$graduation, $student]) }}"
           class="inline-flex items-center rounded-md border border-gray-300 bg-white px-3 py-1.5 text-sm font-medium text-gray-700 hover:bg-gray-50 dark:border-neutral-700 dark:bg-neutral-900 dark:text-neutral-200 dark:hover:bg-neutral-800">
            Cancel
        </a>
        <button type="submit"
                class="inline-flex items-center rounded-md bg-indigo-600 px-3 py-1.5 text-sm font-medium text-white hover:bg-indigo-700">
            Save changes
        </button>
    </div>
</form>
```

The `enctype="multipart/form-data"` and `@method('PATCH')` are both required for the file upload to land in `$request->file('payment_receipt')`.

## Expected output

```bash
php artisan migrate:fresh --seed
php artisan serve
```

In another terminal: `npm run dev`.

Then walk through:

1. Log in as `admin@devhub.test` / `password`.
2. Click **Graduations** in the sidebar → see 3 rows with status badges (try toggling dark mode — badges should stay readable).
3. Click **+ New graduation** → fill in the form → submit → land on the show page.
4. Edit it → see the **Archive** button in the footer left.
5. Click a student → see their detail page; if they're verified the Verify button is hidden.
6. Log out, log in as `student@devhub.test` → no **+ New graduation** button visible; index Edit links also gone.

## Commit your work

```bash
git add .
git commit -m "feat(ui): add graduation + student Blade views + nav link"
```

## Common pitfalls

- **File upload silently saves an empty `payment_receipt`** — you forgot `enctype="multipart/form-data"` on the form.
- **`@method('PATCH')` missing on file-upload form** — the route is `PATCH /graduations/{graduation}/students/{student}` but the browser only knows POST. `@method('PATCH')` rewrites it server-side.
- **Status badge unstyled in dark mode** — every status entry in `$statusStyles` needs both light *and* `dark:` variants. Forgetting `dark:bg-…` makes the badge invisible on the dark sidebar background.
- **Nested `<form>` from Archive button** — browsers don't allow `<form>` inside `<form>`. Use the hidden-form + `document.getElementById(...).submit()` pattern shown in `edit.blade.php`.
- **`<x-app-layout>` / `<x-input-label>` "view not found" errors** — those are Breeze components; this kit uses `<x-layouts::app>` and raw `<label>` / `<input>` markup instead.

## What's next

Receipts are saving, but they're not viewable in the browser yet. We need a symlink: [10 — Storage symlink](./10-storage-symlink.md).
