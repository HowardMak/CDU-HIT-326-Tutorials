# Frontend Modernisation

## Task List Summary

- [ ] **Build the CSS Design System** - Add reusable utility classes to `app.css` using Tailwind's `@layer components`
- [ ] **Compile Assets** - Run `npm run build` (or `npm run dev` during development) to compile the updated CSS
- [ ] **Update App Layout** - Add floating toast notifications with Alpine.js, a persistent footer, and slate-50 background
- [ ] **Update Guest Layout** - Create a two-column auth layout with a branded gradient left panel
- [ ] **Update Navigation** - Build a glassmorphism sticky navbar with role-aware CTAs and an avatar dropdown
- [ ] **Polish Shared Components** - Modernise buttons, inputs, labels, errors, dropdowns, and nav-links
- [ ] **Create `<x-breadcrumb>` Component** - Reusable breadcrumb trail from an array of items
- [ ] **Create `<x-page-hero>` Component** - Gradient banner header with optional action slot
- [ ] **Create `<x-form-section>` Component** - Titled card wrapper for groups of form fields
- [ ] **Create `<x-stat-card>` Component** - Icon + number stat block for dashboards
- [ ] **Create `<x-event-card>` Component** - Dual-variant card (browse mode and dashboard mode)
- [ ] **Update Dashboard View** - Add greeting, stat bar, filter tabs, and redesigned event cards
- [ ] **Update Events Index View** - Add page hero, floating search card, filter pills, and redesigned cards
- [ ] **Update Event Show View** - Redesign as two-column layout with breadcrumb and contextual registration sidebar
- [ ] **Update Create and Edit Views** - Group fields into `form-section` cards with icon-prefixed inputs
- [ ] **Update Registrants View** - Add stat cards, avatar initials, and redesigned table
- [ ] **Update Auth and Welcome Pages** - Polish login/register forms and replace the default Laravel welcome page

---

## Step 1: Build the CSS Design System

Open `resources/css/app.css`. Before this change the file contained only three Tailwind directives. You will add an `@layer components` block that defines every reusable class used across the app. Defining classes here (rather than scattering inline Tailwind strings everywhere) means you can change a design token in one place and it applies everywhere.

```css
@import 'tailwindcss/base';
@import 'tailwindcss/components';
@import 'tailwindcss/utilities';

@layer components {

    /* ── Buttons ─────────────────────────────────────── */
    .btn-primary {
        @apply inline-flex items-center px-6 py-2.5 bg-gradient-to-r from-indigo-600 to-purple-600
               text-white font-semibold rounded-full shadow-sm
               hover:shadow-md hover:-translate-y-0.5 active:scale-95
               transition-all duration-200 cursor-pointer;
    }

    .btn-secondary {
        @apply inline-flex items-center px-6 py-2.5 bg-white border border-slate-300
               text-slate-700 font-semibold rounded-full
               hover:bg-slate-50 hover:border-slate-400 active:scale-95
               transition-all duration-200 cursor-pointer;
    }

    .btn-danger {
        @apply inline-flex items-center px-6 py-2.5 bg-red-600
               text-white font-semibold rounded-full shadow-sm
               hover:bg-red-700 hover:shadow-md active:scale-95
               transition-all duration-200 cursor-pointer;
    }

    /* ── Cards ───────────────────────────────────────── */
    .card {
        @apply bg-white rounded-2xl shadow-sm hover:shadow-md transition-shadow duration-200;
    }

    /* ── Badges ──────────────────────────────────────── */
    .badge       { @apply inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-semibold; }
    .badge-green { @apply badge bg-emerald-100 text-emerald-700; }
    .badge-yellow{ @apply badge bg-amber-100   text-amber-700; }
    .badge-blue  { @apply badge bg-indigo-100  text-indigo-700; }
    .badge-red   { @apply badge bg-red-100     text-red-700; }
    .badge-gray  { @apply badge bg-slate-100   text-slate-600; }

    /* ── Forms ───────────────────────────────────────── */
    .form-section {
        @apply bg-white rounded-2xl shadow-sm p-6 mb-6;
    }

    .form-group  { @apply mb-5; }

    .form-label  {
        @apply block text-sm font-semibold text-slate-700 mb-1.5;
    }

    .form-input  {
        @apply w-full border border-slate-300 rounded-xl px-4 py-2.5 text-slate-900
               placeholder-slate-400
               focus:outline-none focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500
               disabled:bg-slate-50 disabled:text-slate-500
               transition duration-150;
    }

    .form-error  { @apply text-sm text-red-600 mt-1; }

    /* ── Stats ───────────────────────────────────────── */
    .stat-card {
        @apply bg-white rounded-2xl shadow-sm p-5 flex items-center gap-4;
    }

    /* ── Typography helpers ───────────────────────────── */
    .page-title    { @apply text-3xl font-extrabold text-slate-900; }
    .section-title { @apply text-lg font-bold text-slate-800; }
}
```

**Why `@layer components`?** Tailwind processes this block during the build step, so PurgeCSS/JIT still tree-shakes unused utilities, but your custom classes are available everywhere in Blade templates just like any Tailwind class.

---

## Step 2: Compile Assets

After editing `app.css` you must recompile the frontend bundle — Laravel serves the compiled file from `public/build/`, not `resources/css/app.css` directly. If you skip this step none of the custom classes will appear in the browser, even though they exist in the source file.

### During development

Run Vite in watch mode. It monitors `resources/` for changes and recompiles automatically on every save:

```bash
npm run dev
```

Leave this terminal running while you work. Vite also enables Hot Module Replacement (HMR), so CSS changes reflect instantly in the browser without a full page reload.

### For a production build

When you are ready to deploy, or simply want to verify the final output:

```bash
npm run build
```

This writes the compiled and hashed files into `public/build/assets/` and updates `public/build/manifest.json` so Laravel's `@vite` directive in `layouts/app.blade.php` points to the new file.

> **What went wrong without this step:** The compiled CSS at `public/build/assets/app-*.css` was generated from an old build — before the `@layer components` block was added. It was ~40 KB and contained none of the custom classes. After running `npm run build` the output grew to ~71 KB and all 93+ usages of `.btn-primary`, `.card`, `.badge-*`, `.stat-card`, `.form-input`, etc. across 11 Blade view files were resolved correctly.

---

## Step 3: Update the App Layout

Open `resources/views/layouts/app.blade.php`. You will make three changes: the body background, flash messages, and a persistent footer.

### 3a — Background and structure

Change the body from `bg-gray-100` to `bg-slate-50` and wrap the body contents in a flex column so the footer always sits at the bottom:

```blade
<body class="font-sans antialiased bg-slate-50 flex flex-col min-h-screen">
    @include('layouts.navigation')

    <main class="flex-1">
        {{ $slot }}
    </main>

    {{-- Footer --}}
    <footer class="bg-white border-t border-slate-100 mt-auto">
        <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8 flex flex-col sm:flex-row items-center justify-between gap-4">
            <div class="flex items-center gap-2">
                <div class="w-7 h-7 bg-gradient-to-br from-indigo-600 to-purple-600 rounded-lg flex items-center justify-center">
                    <svg class="w-4 h-4 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                              d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
                    </svg>
                </div>
                <span class="font-bold text-slate-800">{{ config('app.name') }}</span>
            </div>
            <nav class="flex gap-6 text-sm text-slate-500">
                <a href="{{ route('events.index') }}" class="hover:text-indigo-600 transition-colors">Events</a>
                @auth
                    <a href="{{ route('dashboard') }}" class="hover:text-indigo-600 transition-colors">Dashboard</a>
                @endauth
            </nav>
            <p class="text-sm text-slate-400">© {{ date('Y') }} {{ config('app.name') }}</p>
        </div>
    </footer>
```

### 3b — Floating toast notifications

Replace the old inline flash banners (which pushed page content down) with fixed top-right toasts that auto-dismiss after 4 seconds. This uses **Alpine.js** — already included with Laravel Breeze.

```blade
{{-- Floating toasts --}}
@if (session('success') || session('error') || session('warning'))
<div class="fixed top-4 right-4 z-50 flex flex-col gap-2" x-data>

    @if (session('success'))
    <div x-data="{ show: true }" x-show="show" x-init="setTimeout(() => show = false, 4000)"
         x-transition:enter="transition ease-out duration-300"
         x-transition:enter-start="opacity-0 translate-y-2"
         x-transition:enter-end="opacity-100 translate-y-0"
         x-transition:leave="transition ease-in duration-200"
         x-transition:leave-start="opacity-100"
         x-transition:leave-end="opacity-0"
         class="flex items-center gap-3 bg-white border border-emerald-200 text-emerald-800
                rounded-2xl shadow-lg px-4 py-3 min-w-72">
        <svg class="w-5 h-5 text-emerald-500 shrink-0" fill="currentColor" viewBox="0 0 20 20">
            <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd"/>
        </svg>
        <span class="text-sm font-medium">{{ session('success') }}</span>
        <button @click="show = false" class="ml-auto text-emerald-400 hover:text-emerald-600">
            <svg class="w-4 h-4" fill="currentColor" viewBox="0 0 20 20">
                <path fill-rule="evenodd" d="M4.293 4.293a1 1 0 011.414 0L10 8.586l4.293-4.293a1 1 0 111.414 1.414L11.414 10l4.293 4.293a1 1 0 01-1.414 1.414L10 11.414l-4.293 4.293a1 1 0 01-1.414-1.414L8.586 10 4.293 5.707a1 1 0 010-1.414z" clip-rule="evenodd"/>
            </svg>
        </button>
    </div>
    @endif

    {{-- Repeat the same pattern for session('error') and session('warning') --}}
    {{-- Use border-red-200 / text-red-800 for error, border-amber-200 / text-amber-800 for warning --}}

</div>
@endif
```

**Key Alpine.js concepts used here:**
- `x-data` — initialises an Alpine component scope
- `x-show` — toggles `display:none` based on an expression
- `x-init` — runs JS when the component is first rendered (`setTimeout` auto-dismiss)
- `x-transition` — applies enter/leave CSS transitions automatically

---

## Step 4: Update the Guest Layout

The guest layout (`resources/views/layouts/guest.blade.php`) wraps login and register pages. Change it from a plain centred card to a **two-column split** on large screens.

```blade
<body class="font-sans antialiased bg-slate-50">
<div class="min-h-screen flex">

    {{-- Left branding panel (hidden on small screens) --}}
    <div class="hidden lg:flex lg:w-1/2 bg-gradient-to-br from-indigo-600 to-purple-700
                flex-col justify-between p-12 relative overflow-hidden">

        {{-- Decorative background blobs --}}
        <div class="absolute inset-0 overflow-hidden">
            <div class="absolute -top-40 -right-40 w-80 h-80 bg-white/10 rounded-full blur-3xl"></div>
            <div class="absolute -bottom-40 -left-40 w-80 h-80 bg-white/10 rounded-full blur-3xl"></div>
        </div>

        <div class="relative z-10">
            <div class="flex items-center gap-3 mb-12">
                <div class="w-10 h-10 bg-white/20 backdrop-blur rounded-xl flex items-center justify-center">
                    <svg class="w-6 h-6 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                              d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
                    </svg>
                </div>
                <span class="text-white font-bold text-xl">{{ config('app.name') }}</span>
            </div>

            <h1 class="text-4xl font-extrabold text-white mb-4">Discover & manage events with ease</h1>
            <p class="text-indigo-200 text-lg mb-10">Join thousands of organisers and attendees on our platform.</p>

            <ul class="space-y-4">
                @foreach(['Browse hundreds of local events', 'Register in seconds', 'Manage your own events'] as $feature)
                <li class="flex items-center gap-3 text-white/90">
                    <div class="w-6 h-6 bg-white/20 rounded-full flex items-center justify-center shrink-0">
                        <svg class="w-3.5 h-3.5 text-white" fill="currentColor" viewBox="0 0 20 20">
                            <path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd"/>
                        </svg>
                    </div>
                    {{ $feature }}
                </li>
                @endforeach
            </ul>
        </div>
    </div>

    {{-- Right form panel --}}
    <div class="flex-1 flex items-center justify-center p-6 lg:p-12">
        <div class="w-full max-w-md">
            {{-- Mobile logo --}}
            <div class="flex justify-center mb-8 lg:hidden">
                <div class="flex items-center gap-2">
                    <div class="w-9 h-9 bg-gradient-to-br from-indigo-600 to-purple-600 rounded-xl flex items-center justify-center">
                        <svg class="w-5 h-5 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                                  d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
                        </svg>
                    </div>
                    <span class="font-bold text-xl text-slate-900">{{ config('app.name') }}</span>
                </div>
            </div>

            <div class="bg-white rounded-3xl shadow-xl p-8 lg:p-10">
                {{ $slot }}
            </div>
        </div>
    </div>

</div>
</body>
```

---

## Step 5: Update the Navigation

Open `resources/views/layouts/navigation.blade.php`. The changes introduce glassmorphism, a gradient logo, pill nav-links, role-aware CTAs, and an avatar dropdown.

### Key structural changes

```blade
{{-- Sticky glassmorphism bar --}}
<nav class="sticky top-0 z-40 bg-white/95 backdrop-blur-sm border-b border-slate-100 shadow-sm">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between h-16 items-center">

            {{-- Logo --}}
            <a href="{{ route('dashboard') }}" class="flex items-center gap-2.5 group">
                <div class="w-8 h-8 bg-gradient-to-br from-indigo-600 to-purple-600 rounded-xl
                            flex items-center justify-center shadow-sm group-hover:shadow-md transition-shadow">
                    <svg class="w-4 h-4 text-white" .../>
                </div>
                <span class="font-extrabold text-transparent bg-clip-text bg-gradient-to-r from-indigo-600 to-purple-600 text-lg">
                    {{ config('app.name') }}
                </span>
            </a>

            {{-- Desktop nav links --}}
            <div class="hidden lg:flex items-center gap-1">
                <x-nav-link :href="route('events.index')" :active="request()->routeIs('events.*')">
                    Events
                </x-nav-link>
                @auth
                    <x-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
                        Dashboard
                    </x-nav-link>
                @endauth
            </div>

            {{-- Right side --}}
            <div class="hidden lg:flex items-center gap-3">
                @auth
                    {{-- Create Event — visible to organiser/admin only --}}
                    @if(in_array(Auth::user()->role, ['organiser', 'admin']))
                    <a href="{{ route('events.create') }}" class="btn-primary text-sm py-2 px-4">
                        + Create Event
                    </a>
                    @endif

                    {{-- Avatar dropdown --}}
                    <x-dropdown align="right" width="56">
                        <x-slot name="trigger">
                            <button class="flex items-center gap-2 px-3 py-1.5 rounded-full
                                           hover:bg-slate-50 transition-colors">
                                <div class="w-7 h-7 bg-gradient-to-br from-indigo-500 to-purple-600
                                            rounded-full flex items-center justify-center text-white text-xs font-bold">
                                    {{ strtoupper(substr(Auth::user()->name, 0, 1)) }}
                                </div>
                                <span class="text-sm font-medium text-slate-700 max-w-24 truncate">
                                    {{ Auth::user()->name }}
                                </span>
                                <svg class="w-4 h-4 text-slate-400" fill="currentColor" viewBox="0 0 20 20">
                                    <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd"/>
                                </svg>
                            </button>
                        </x-slot>

                        <x-slot name="content">
                            <div class="px-4 py-3 border-b border-slate-100">
                                <p class="text-sm font-semibold text-slate-900">{{ Auth::user()->name }}</p>
                                <p class="text-xs text-slate-500 truncate">{{ Auth::user()->email }}</p>
                            </div>
                            <x-dropdown-link :href="route('profile.edit')">Profile</x-dropdown-link>
                            <form method="POST" action="{{ route('logout') }}">
                                @csrf
                                <x-dropdown-link :href="route('logout')"
                                    onclick="event.preventDefault(); this.closest('form').submit();">
                                    Log Out
                                </x-dropdown-link>
                            </form>
                        </x-slot>
                    </x-dropdown>
                @else
                    <a href="{{ route('login') }}" class="btn-secondary text-sm py-2 px-4">Log In</a>
                    <a href="{{ route('register') }}" class="btn-primary text-sm py-2 px-4">Get Started</a>
                @endauth
            </div>

        </div>
    </div>
</nav>
```

**Design concepts:**
- `bg-white/95 backdrop-blur-sm` — semi-transparent white with CSS backdrop blur (glassmorphism)
- `sticky top-0 z-40` — navbar stays at the top as you scroll, sitting above page content
- `bg-clip-text text-transparent` — renders the gradient through the text characters

---

## Step 6: Polish Shared Components

These are small components in `resources/views/components/` that are used everywhere. Updating them here means every view that references them gets the new look automatically.

### `primary-button.blade.php`

```blade
<button {{ $attributes->merge(['type' => 'submit', 'class' => 'btn-primary']) }}>
    {{ $slot }}
</button>
```

### `secondary-button.blade.php`

```blade
<button {{ $attributes->merge(['type' => 'button', 'class' => 'btn-secondary']) }}>
    {{ $slot }}
</button>
```

### `danger-button.blade.php`

```blade
<button {{ $attributes->merge(['type' => 'submit', 'class' => 'btn-danger']) }}>
    {{ $slot }}
</button>
```

### `text-input.blade.php`

```blade
@php $classes = 'form-input ' . ($attributes->get('class', '')); @endphp

<input {{ $attributes->merge(['class' => $classes]) }}>
```

### `input-label.blade.php`

```blade
<label {{ $attributes }}>
    <span class="form-label">{{ $value ?? $slot }}</span>
</label>
```

### `input-error.blade.php`

Add an icon to each error line for visual attention:

```blade
@if ($messages)
<ul {{ $attributes->merge(['class' => 'form-error space-y-1']) }}>
    @foreach ($messages as $message)
    <li class="flex items-center gap-1.5">
        <svg class="w-3.5 h-3.5 shrink-0" fill="currentColor" viewBox="0 0 20 20">
            <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7 4a1 1 0 11-2 0 1 1 0 012 0zm-1-9a1 1 0 00-1 1v4a1 1 0 102 0V6a1 1 0 00-1-1z" clip-rule="evenodd"/>
        </svg>
        {{ $message }}
    </li>
    @endforeach
</ul>
@endif
```

### `nav-link.blade.php`

Switch from an underline active indicator to a pill background:

```blade
@php
$classes = $active
    ? 'px-3 py-1.5 rounded-full text-sm font-semibold bg-indigo-50 text-indigo-600'
    : 'px-3 py-1.5 rounded-full text-sm font-medium text-slate-600 hover:bg-indigo-50 hover:text-indigo-600 transition-colors';
@endphp

<a {{ $attributes->merge(['class' => $classes]) }}>{{ $slot }}</a>
```

### `dropdown.blade.php`

Update the panel classes to match the card design language:

```blade
{{-- inside the panel div, change from rounded-md shadow-lg to: --}}
class="... rounded-2xl shadow-xl border border-slate-100 ..."
```

---

## Step 7: Create `<x-breadcrumb>`

Create `resources/views/components/breadcrumb.blade.php`.

The component accepts an `$items` array where each item is `['label' => '...', 'url' => '...']`. The last item is always plain text; all preceding items are links if a `url` key is present.

```blade
{{-- @props declares the variables this component accepts.
     'items' defaults to an empty array so the component won't crash
     if you forget to pass it. --}}
@props(['items' => []])

<nav class="flex items-center gap-1.5 text-sm text-slate-500 mb-6">
    {{-- Loop through each breadcrumb item, keeping track of its position ($index). --}}
    @foreach ($items as $index => $item)

        {{-- Add a ">" arrow before every item except the first. --}}
        @if ($index > 0)
            <svg class="w-3.5 h-3.5 text-slate-300 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
            </svg>
        @endif

        {{-- The last item is the current page — show it as bold text, not a link. --}}
        @if ($index === count($items) - 1)
            <span class="font-semibold text-slate-900 truncate max-w-48">{{ $item['label'] }}</span>

        {{-- Any earlier item that has a 'url' key becomes a clickable link. --}}
        @elseif (isset($item['url']))
            <a href="{{ $item['url'] }}" class="hover:text-indigo-600 transition-colors truncate max-w-32">
                {{ $item['label'] }}
            </a>

        {{-- An item without a 'url' is shown as plain text (no link, not the active page either). --}}
        @else
            <span>{{ $item['label'] }}</span>
        @endif

    @endforeach
</nav>
```

**Usage example** (in any view):
```blade
<x-breadcrumb :items="[
    ['label' => 'Events', 'url' => route('events.index')],
    ['label' => $event->title],
]"/>
```

---

## Step 8: Create `<x-page-hero>`

Create `resources/views/components/page-hero.blade.php`.

The component renders the indigo→purple gradient banner used at the top of the events index page. It accepts a `$title`, `$subtitle`, and an optional `$action` named slot for right-aligned content.

The SVG pattern ID is namespaced with `Str::slug($title)` to avoid duplicate IDs if more than one hero appears on the same page.

```blade
{{-- 'title' is required. 'subtitle' is optional — it defaults to null so
     the component still works if you don't pass it. --}}
@props(['title', 'subtitle' => null])

{{-- Build a unique ID for the SVG pattern using the title text.
     If two heroes appeared on the same page with the same hard-coded ID,
     the browser would only use the first one, breaking the other. --}}
@php $patternId = 'hero-grid-' . Str::slug($title); @endphp

<div class="relative bg-gradient-to-r from-indigo-600 to-purple-700 overflow-hidden">

    {{-- Subtle grid lines drawn with an SVG repeating pattern.
         opacity-10 keeps it very faint so it doesn't compete with the text. --}}
    <svg class="absolute inset-0 w-full h-full opacity-10" xmlns="http://www.w3.org/2000/svg">
        <defs>
            <pattern id="{{ $patternId }}" x="0" y="0" width="40" height="40" patternUnits="userSpaceOnUse">
                <path d="M 40 0 L 0 0 0 40" fill="none" stroke="white" stroke-width="1"/>
            </pattern>
        </defs>
        {{-- Fill the whole banner with the pattern defined above. --}}
        <rect width="100%" height="100%" fill="url(#{{ $patternId }})"/>
    </svg>

    {{-- 'relative' here lifts the text above the absolutely-positioned SVG. --}}
    <div class="relative max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-12">
        <div class="flex items-center justify-between gap-6">
            <div>
                <h1 class="text-3xl font-extrabold text-white">{{ $title }}</h1>
                {{-- Only render the subtitle paragraph if one was passed in. --}}
                @if ($subtitle)
                    <p class="mt-2 text-indigo-200 text-lg">{{ $subtitle }}</p>
                @endif
            </div>
            {{-- $action is a named slot. It only exists if the caller provides
                 <x-slot name="action">...</x-slot>, so we check with isset(). --}}
            @if (isset($action))
                <div class="shrink-0">{{ $action }}</div>
            @endif
        </div>
    </div>

</div>
```

**Usage example:**
```blade
<x-page-hero
    title="Upcoming Events"
    :subtitle="$events->total() . ' events available'">

    <x-slot name="action">
        @can('create', App\Models\Event::class)
            <a href="{{ route('events.create') }}" class="btn-primary">+ Create Event</a>
        @endcan
    </x-slot>

</x-page-hero>
```

---

## Step 9: Create `<x-form-section>`

Create `resources/views/components/form-section.blade.php`.

This wraps a group of form fields in a white card with a title and optional description, replacing the manually repeated section cards across `create`, `edit`, and `profile/edit`.

```blade
{{-- 'title' is required. 'description' is optional — leave it out and
     only the title line will appear above the divider. --}}
@props(['title', 'description' => null])

<div class="form-section">
    <div class="mb-5">
        <h3 class="section-title">{{ $title }}</h3>
        {{-- Optional hint text shown below the title, before the divider. --}}
        @if ($description)
            <p class="text-sm text-slate-500 mt-1">{{ $description }}</p>
        @endif
        {{-- Thin horizontal line that separates the header from the fields. --}}
        <div class="border-t border-slate-100 mt-4"></div>
    </div>

    {{-- $slot is the default slot — everything you put between the opening
         and closing <x-form-section> tags lands here. --}}
    {{ $slot }}
</div>
```

**Usage example:**
```blade
<x-form-section title="Basic Information" description="Give your event a clear title and description.">
    <div class="form-group">
        <x-input-label for="title" value="Event Title *"/>
        <x-text-input id="title" name="title" class="form-input" :value="old('title')" required/>
        <x-input-error :messages="$errors->get('title')"/>
    </div>
    ...
</x-form-section>
```

---

## Step 10: Create `<x-stat-card>`

Create `resources/views/components/stat-card.blade.php`.

The `$color` prop drives the icon container and icon colour via a PHP `match` expression. Adding support for a new colour only requires one new entry in that array.

```blade
{{-- 'value' and 'label' are required (the number and its description).
     'color' defaults to indigo if you don't pass one. --}}
@props(['value', 'label', 'color' => 'indigo'])

@php
{{-- match() is like a switch statement. It maps the color name you passed in
     to the actual Tailwind classes for the icon background and icon colour.
     'default' is the fallback if you pass an unrecognised colour. --}}
$colors = match($color) {
    'indigo'  => ['bg' => 'bg-indigo-100',  'text' => 'text-indigo-600'],
    'purple'  => ['bg' => 'bg-purple-100',  'text' => 'text-purple-600'],
    'emerald' => ['bg' => 'bg-emerald-100', 'text' => 'text-emerald-600'],
    'amber'   => ['bg' => 'bg-amber-100',   'text' => 'text-amber-600'],
    default   => ['bg' => 'bg-slate-100',   'text' => 'text-slate-600'],
};
@endphp

<div class="stat-card">
    {{-- Coloured square that holds the icon.
         The classes come from the $colors array built above. --}}
    <div class="w-12 h-12 {{ $colors['bg'] }} {{ $colors['text'] }} rounded-xl flex items-center justify-center shrink-0">
        {{-- $icon is a named slot — put your SVG inside <x-slot name="icon">. --}}
        {{ $icon }}
    </div>
    <div>
        {{-- The big number, e.g. "42". --}}
        <p class="text-2xl font-extrabold text-slate-900">{{ $value }}</p>
        {{-- The description below it, e.g. "Total Registered". --}}
        <p class="text-sm text-slate-500 font-medium">{{ $label }}</p>
    </div>
</div>
```

**Usage example:**
```blade
<x-stat-card :value="$totalCount" label="Total Registrations" color="indigo">
    <x-slot name="icon">
        <svg class="w-6 h-6" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0"/>
        </svg>
    </x-slot>
</x-stat-card>
```

---

## Step 11: Create `<x-event-card>`

Create `resources/views/components/event-card.blade.php`.

This is a **dual-variant** component. Without the optional `:registration` prop it renders the browse-mode card (with image/gradient and description). With it, it renders the dashboard-mode card (with status badge and registered date).

```blade
{{-- 'event' is required — pass in the Event model to display.
     'registration' is optional. Pass $event->pivot when rendering from the dashboard
     so the card knows the user's registration status.
     Leave it out on the public listing and you get the browse-mode style. --}}
@props(['event', 'registration' => null])

@php
{{-- Five gradient colour pairs used as placeholder banners when there is no image.
     The % (modulo) operator picks one based on the event ID, so the same
     event always gets the same colour no matter which page it appears on. --}}
$gradients = [
    'from-indigo-400 to-purple-500',
    'from-rose-400 to-pink-500',
    'from-amber-400 to-orange-500',
    'from-teal-400 to-cyan-500',
    'from-emerald-400 to-green-500',
];
$gradient = $gradients[$event->id % 5];

{{-- Carbon's isPast() returns true when the event date has already passed. --}}
$isPast   = $event->date->isPast();
@endphp

{{-- 'group' lets child elements react to a hover on this parent div.
     E.g. group-hover:text-indigo-600 on the title fires only when THIS card is hovered. --}}
<div class="card overflow-hidden group hover:-translate-y-0.5 transition-all duration-200">

    {{-- ── DASHBOARD MODE: renders when $registration is passed in ── --}}
    @if ($registration)
        {{-- Thin colour bar at the top: indigo = upcoming, gray = past. --}}
        <div class="h-2 bg-gradient-to-r {{ $isPast ? 'from-slate-400 to-slate-500' : 'from-indigo-500 to-purple-600' }}"></div>

        <div class="p-5">
            <div class="flex items-start justify-between gap-3 mb-3">
                <h3 class="font-bold text-slate-900 line-clamp-2 group-hover:text-indigo-600 transition-colors">
                    <a href="{{ route('events.show', $event) }}">{{ $event->title }}</a>
                </h3>
                {{-- Two small badges stacked on the right:
                     top = confirmed or waitlisted, bottom = upcoming or past. --}}
                <div class="flex flex-col items-end gap-1.5 shrink-0">
                    @if ($registration->status === 'confirmed')
                        <span class="badge-green">✓ Confirmed</span>
                    @else
                        <span class="badge-yellow">⏳ Waitlisted</span>
                    @endif
                    <span class="{{ $isPast ? 'badge-gray' : 'badge-blue' }}">
                        {{ $isPast ? 'Past' : 'Upcoming' }}
                    </span>
                </div>
            </div>

            <div class="flex items-center gap-1.5 text-slate-500 text-sm">
                <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                          d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
                </svg>
                {{ $event->date->format('M d, Y') }}
            </div>
            <p class="text-xs text-slate-400 mt-2">
                {{-- diffForHumans() gives a friendly string like "3 days ago". --}}
                Registered {{ $registration->registered_at->diffForHumans() }}
            </p>
        </div>

    {{-- ── BROWSE MODE: renders when no $registration is passed in ── --}}
    @else
        <div class="h-40 overflow-hidden relative">
            {{-- Show the real image if one was uploaded, otherwise fall back to the gradient placeholder. --}}
            @if ($event->image_path)
                {{-- group-hover:scale-105 gently zooms the image when the card is hovered. --}}
                <img src="{{ Storage::url($event->image_path) }}" alt="{{ $event->title }}"
                     class="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300">
            @else
                {{-- No image — show a coloured gradient with a faint calendar icon in the centre. --}}
                <div class="w-full h-full bg-gradient-to-br {{ $gradient }} flex items-center justify-center">
                    <svg class="w-12 h-12 text-white/60" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5"
                              d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
                    </svg>
                </div>
            @endif
            {{-- Category badge pinned to the top-left corner of the image area. --}}
            @if ($event->category)
                <span class="absolute top-3 left-3 badge-blue text-xs">{{ $event->category }}</span>
            @endif
        </div>

        <div class="p-5">
            {{-- line-clamp-2 cuts text to 2 lines max so all cards stay the same height. --}}
            <h3 class="font-bold text-slate-900 line-clamp-2 mb-2 group-hover:text-indigo-600 transition-colors">
                <a href="{{ route('events.show', $event) }}">{{ $event->title }}</a>
            </h3>
            <p class="text-sm text-slate-500 line-clamp-2 mb-4">{{ $event->description }}</p>
            {{-- Footer row: date icon on the left, location icon only if a location is set. --}}
            <div class="flex items-center gap-3 text-xs text-slate-400 pt-3 border-t border-slate-50">
                <span class="flex items-center gap-1">
                    <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                              d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 002-2z"/>
                    </svg>
                    {{ $event->date->format('M d, Y') }}
                </span>
                @if ($event->location)
                <span class="flex items-center gap-1">
                    <svg class="w-3.5 h-3.5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                              d="M17.657 16.657L13.414 20.9a1.998 1.998 0 01-2.827 0l-4.244-4.243a8 8 0 1111.314 0z"/>
                    </svg>
                    {{ $event->location }}
                </span>
                @endif
            </div>
        </div>
    @endif

</div>
```

**Key design decision:** The gradient colour is derived deterministically from `$event->id % 5`. This means the same event always renders with the same gradient on every page, giving visual consistency without storing any extra data.

---

## Step 12: Update the Dashboard View

Open `resources/views/dashboard.blade.php`. Use the new components to build a personalised, data-rich page.

```blade
<x-app-layout>
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">

    {{-- Greeting --}}
    <div class="mb-8">
        <h1 class="page-title">
            Welcome back,
            <span class="text-transparent bg-clip-text bg-gradient-to-r from-indigo-600 to-purple-600">
                {{ Auth::user()->name }}
            </span>
        </h1>
        <p class="text-slate-500 mt-1">Here's what's happening with your events.</p>
    </div>

    {{-- Stat bar --}}
    @php
        $total      = $registrations->count();
        $upcoming   = $registrations->where('date', '>=', now()->toDateString())->count();
        $confirmed  = $registrations->where('pivot.status', 'confirmed')->count();
        $waitlisted = $registrations->where('pivot.status', 'waitlisted')->count();
    @endphp

    <div class="grid grid-cols-2 lg:grid-cols-4 gap-4 mb-8">
        <x-stat-card :value="$total" label="Total Registered" color="indigo">
            <x-slot name="icon"><!-- calendar SVG --></x-slot>
        </x-stat-card>
        <x-stat-card :value="$upcoming" label="Upcoming" color="purple">
            <x-slot name="icon"><!-- clock SVG --></x-slot>
        </x-stat-card>
        <x-stat-card :value="$confirmed" label="Confirmed" color="emerald">
            <x-slot name="icon"><!-- check-circle SVG --></x-slot>
        </x-stat-card>
        <x-stat-card :value="$waitlisted" label="Waitlisted" color="amber">
            <x-slot name="icon"><!-- exclamation SVG --></x-slot>
        </x-stat-card>
    </div>

    {{-- Filter tabs --}}
    <div class="flex items-center gap-2 mb-6">
        @foreach (['all' => 'All', 'upcoming' => 'Upcoming', 'past' => 'Past'] as $value => $label)
        <a href="{{ route('dashboard', ['filter' => $value]) }}"
           class="{{ request('filter', 'all') === $value
               ? 'btn-primary text-sm py-1.5 px-4'
               : 'btn-secondary text-sm py-1.5 px-4' }}">
            {{ $label }}
        </a>
        @endforeach
    </div>

    {{-- Event cards grid --}}
    @if ($registrations->isEmpty())
        {{-- Empty state --}}
        <div class="text-center py-16">
            <div class="w-16 h-16 bg-indigo-50 rounded-2xl flex items-center justify-center mx-auto mb-4">
                <!-- calendar SVG -->
            </div>
            <h3 class="section-title mb-2">No events yet</h3>
            <p class="text-slate-500 mb-6">Start by browsing events and registering for the ones you like.</p>
            <a href="{{ route('events.index') }}" class="btn-primary">Browse Events</a>
        </div>
    @else
        <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
            @foreach ($registrations as $event)
                <x-event-card :event="$event" :registration="$event->pivot"/>
            @endforeach
        </div>
    @endif

</div>
</x-app-layout>
```

---

## Step 13: Update Events Views

### `events/index.blade.php` — Page hero and floating search

```blade
{{-- Hero --}}
<x-page-hero
    title="Upcoming Events"
    :subtitle="$events->total() . ' events available'">
    <x-slot name="action">
        @can('create', App\Models\Event::class)
            <a href="{{ route('events.create') }}" class="btn-primary">+ Create Event</a>
        @endcan
    </x-slot>
</x-page-hero>

{{-- Floating search card (negative margin pulls it up over the hero) --}}
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
    <div class="card -mt-6 z-10 relative p-6 mb-8">
        <form method="GET" action="{{ route('events.index') }}" class="flex gap-4 flex-wrap">
            <div class="relative flex-1 min-w-48">
                <svg class="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-slate-400"
                     fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                          d="M21 21l-6-6m2-5a7 7 0 11-14 0 7 7 0 0114 0z"/>
                </svg>
                <input type="text" name="search" value="{{ request('search') }}"
                       class="form-input pl-10" placeholder="Search events...">
            </div>
            <button type="submit" class="btn-primary">Search</button>
        </form>

        {{-- Active filter pills --}}
        @if (request('search') || request('category'))
        <div class="flex gap-2 flex-wrap mt-3">
            @if (request('search'))
                <span class="badge-blue">
                    "{{ request('search') }}"
                    <a href="{{ request()->fullUrlWithoutQuery(['search']) }}" class="ml-1 hover:text-indigo-800">×</a>
                </span>
            @endif
        </div>
        @endif
    </div>

    {{-- Cards grid --}}
    <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6">
        @forelse ($events as $event)
            <x-event-card :event="$event"/>
        @empty
            <div class="col-span-3 text-center py-16">
                <h3 class="section-title mb-2">No events found</h3>
                <p class="text-slate-500">Try adjusting your search filters.</p>
            </div>
        @endforelse
    </div>

    <div class="mt-8">{{ $events->withQueryString()->links() }}</div>
</div>
```

### `events/show.blade.php` — Two-column layout with breadcrumb

```blade
<div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-8">

    <x-breadcrumb :items="[
        ['label' => 'Events', 'url' => route('events.index')],
        ['label' => $event->title],
    ]"/>

    <div class="grid grid-cols-1 lg:grid-cols-3 gap-8">

        {{-- Main column (2/3) --}}
        <div class="lg:col-span-2 space-y-6">
            <div class="card overflow-hidden">
                {{-- Image or gradient placeholder --}}
                <div class="h-64 relative">
                    @if ($event->image_path)
                        <img src="{{ Storage::url($event->image_path) }}" class="w-full h-full object-cover">
                    @else
                        @php $g = ['from-indigo-400 to-purple-500','from-rose-400 to-pink-500','from-amber-400 to-orange-500','from-teal-400 to-cyan-500','from-emerald-400 to-green-500']; @endphp
                        <div class="w-full h-full bg-gradient-to-br {{ $g[$event->id % 5] }} flex items-center justify-center">
                            <svg class="w-20 h-20 text-white/40" .../>
                        </div>
                    @endif
                </div>

                <div class="p-8">
                    <div class="flex flex-wrap gap-2 mb-4">
                        @if ($event->category) <span class="badge-blue">{{ $event->category }}</span> @endif
                        <span class="{{ $event->status === 'published' ? 'badge-green' : 'badge-gray' }}">
                            {{ ucfirst($event->status) }}
                        </span>
                    </div>
                    <h1 class="page-title mb-3">{{ $event->title }}</h1>
                    <p class="text-slate-500 mb-2">Organised by <strong>{{ $event->organiser->name }}</strong></p>
                    <div class="prose prose-slate max-w-none mt-6">{{ $event->description }}</div>
                </div>
            </div>
        </div>

        {{-- Sidebar (1/3) --}}
        <div class="space-y-4">

            {{-- Details card --}}
            <div class="card p-6 space-y-4">
                <h2 class="section-title">Event Details</h2>
                <div class="border-t border-slate-100 pt-4 space-y-4">
                    @foreach ([
                        ['label' => 'DATE',     'value' => $event->date->format('l, F d, Y'), 'color' => 'indigo'],
                        ['label' => 'TIME',     'value' => $event->time, 'color' => 'purple'],
                        ['label' => 'LOCATION', 'value' => $event->location, 'color' => 'emerald'],
                    ] as $detail)
                    <div class="flex items-start gap-3">
                        <div class="w-9 h-9 bg-{{ $detail['color'] }}-100 text-{{ $detail['color'] }}-600 rounded-lg flex items-center justify-center shrink-0">
                            <!-- SVG icon -->
                        </div>
                        <div>
                            <p class="text-xs font-bold text-slate-400 uppercase tracking-wider">{{ $detail['label'] }}</p>
                            <p class="font-semibold text-slate-900 text-sm mt-0.5">{{ $detail['value'] }}</p>
                        </div>
                    </div>
                    @endforeach
                </div>
            </div>

            {{-- Registration card --}}
            <div class="card p-6">
                <h2 class="section-title mb-4">Registration</h2>
                @auth
                    @if ($userRegistration)
                        <div class="bg-emerald-50 border border-emerald-200 rounded-xl p-4 mb-4">
                            <p class="text-emerald-800 font-semibold text-sm">✓ You are registered</p>
                        </div>
                        <form method="POST" action="{{ route('events.unregister', $event) }}">
                            @csrf @method('DELETE')
                            <button class="btn-danger w-full justify-center">Cancel Registration</button>
                        </form>
                    @elseif ($event->isFull())
                        <div class="bg-amber-50 border border-amber-200 rounded-xl p-4 mb-4">
                            <p class="text-amber-800 font-semibold text-sm">Event is full</p>
                        </div>
                        <form method="POST" action="{{ route('events.register', $event) }}">
                            @csrf
                            <button class="btn-secondary w-full justify-center">Join Waitlist</button>
                        </form>
                    @elseif ($event->status === 'published')
                        <form method="POST" action="{{ route('events.register', $event) }}">
                            @csrf
                            <button class="btn-primary w-full justify-center">
                                Register for Event →
                            </button>
                        </form>
                    @endif
                @else
                    <a href="{{ route('login') }}" class="btn-primary w-full justify-center block text-center">
                        Log In to Register
                    </a>
                @endauth
            </div>

        </div>
    </div>
</div>
```

---

## Step 14: Update Remaining Views

### `events/create.blade.php` and `events/edit.blade.php` — Form sections

Wrap field groups in `<x-form-section>` components and add a drag-and-drop image upload zone:

```blade
<x-breadcrumb :items="[
    ['label' => 'Events', 'url' => route('events.index')],
    ['label' => 'Create Event'],
]"/>

<form method="POST" action="{{ route('events.store') }}" enctype="multipart/form-data">
    @csrf

    <x-form-section title="Basic Information" description="Give your event a clear title and description.">
        <div class="form-group">
            <x-input-label for="title" value="Event Title *"/>
            <x-text-input id="title" name="title" :value="old('title')" required/>
            <x-input-error :messages="$errors->get('title')"/>
        </div>
        <div class="form-group">
            <x-input-label for="description" value="Description"/>
            <textarea id="description" name="description" rows="4" class="form-input">{{ old('description') }}</textarea>
        </div>
    </x-form-section>

    <x-form-section title="Date, Time & Location">
        <div class="grid grid-cols-1 sm:grid-cols-2 gap-5">
            <div class="form-group">
                <x-input-label for="date" value="Date *"/>
                <x-text-input id="date" type="date" name="date" :value="old('date')" required/>
            </div>
            <div class="form-group">
                <x-input-label for="time" value="Time *"/>
                <x-text-input id="time" type="time" name="time" :value="old('time')" required/>
            </div>
        </div>
        <div class="form-group">
            <x-input-label for="location" value="Location"/>
            <div class="relative">
                <svg class="absolute left-3 top-1/2 -translate-y-1/2 w-4 h-4 text-slate-400" .../>
                <x-text-input id="location" name="location" class="pl-10" :value="old('location')"/>
            </div>
        </div>
    </x-form-section>

    <x-form-section title="Event Settings">
        <div class="grid grid-cols-1 sm:grid-cols-3 gap-5">
            <div class="form-group">
                <x-input-label for="capacity" value="Capacity"/>
                <x-text-input id="capacity" type="number" name="capacity" :value="old('capacity')"/>
                <p class="text-xs text-slate-400 mt-1">Leave blank for unlimited</p>
            </div>
            <div class="form-group">
                <x-input-label for="category" value="Category"/>
                <x-text-input id="category" name="category" :value="old('category')"/>
            </div>
        </div>

        {{-- Image upload zone --}}
        <div class="form-group">
            <x-input-label value="Event Image"/>
            <label for="image"
                   class="flex flex-col items-center gap-3 p-8 border-2 border-dashed border-slate-200
                          rounded-xl cursor-pointer hover:border-indigo-400 hover:bg-indigo-50/30 transition-colors">
                <svg class="w-10 h-10 text-slate-300" .../>
                <span class="text-sm text-slate-500">Click to upload or drag and drop</span>
                <input id="image" type="file" name="image" accept="image/*" class="sr-only">
            </label>
        </div>
    </x-form-section>

    <div class="flex justify-end gap-3">
        <a href="{{ route('events.index') }}" class="btn-secondary">Cancel</a>
        <button type="submit" class="btn-primary">Create Event</button>
    </div>
</form>
```

### `events/registrants.blade.php` — Stat cards and avatar initials

```blade
<x-breadcrumb :items="[
    ['label' => 'Events', 'url' => route('events.index')],
    ['label' => $event->title, 'url' => route('events.show', $event)],
    ['label' => 'Registrants'],
]"/>

<div class="grid grid-cols-3 gap-4 mb-6">
    <x-stat-card :value="$registrants->count()" label="Total" color="indigo">
        <x-slot name="icon"><!-- users SVG --></x-slot>
    </x-stat-card>
    <x-stat-card :value="$registrants->where('pivot.status','confirmed')->count()" label="Confirmed" color="emerald">
        <x-slot name="icon"><!-- check-circle SVG --></x-slot>
    </x-stat-card>
    <x-stat-card :value="$registrants->where('pivot.status','waitlisted')->count()" label="Waitlisted" color="amber">
        <x-slot name="icon"><!-- clock SVG --></x-slot>
    </x-stat-card>
</div>

<div class="card overflow-hidden">
    <table class="w-full">
        <thead>
            <tr class="bg-slate-50 border-b border-slate-100">
                <th class="text-left px-6 py-4 text-xs font-bold text-slate-500 uppercase tracking-wider">Attendee</th>
                <th class="text-left px-6 py-4 text-xs font-bold text-slate-500 uppercase tracking-wider">Status</th>
                <th class="text-left px-6 py-4 text-xs font-bold text-slate-500 uppercase tracking-wider">Registered</th>
            </tr>
        </thead>
        <tbody class="divide-y divide-slate-50">
            @foreach ($registrants as $attendee)
            <tr class="hover:bg-slate-50 transition-colors">
                <td class="px-6 py-4">
                    <div class="flex items-center gap-3">
                        <div class="w-9 h-9 bg-gradient-to-br from-purple-500 to-indigo-600 rounded-full
                                    flex items-center justify-center text-white text-sm font-bold shrink-0">
                            {{ strtoupper(substr($attendee->name, 0, 1)) }}
                        </div>
                        <div>
                            <p class="font-semibold text-slate-900 text-sm">{{ $attendee->name }}</p>
                            <p class="text-slate-400 text-xs">{{ $attendee->email }}</p>
                        </div>
                    </div>
                </td>
                <td class="px-6 py-4">
                    @if ($attendee->pivot->status === 'confirmed')
                        <span class="badge-green">Confirmed</span>
                    @else
                        <span class="badge-yellow">Waitlisted</span>
                    @endif
                </td>
                <td class="px-6 py-4 text-sm text-slate-500">
                    {{ $attendee->pivot->registered_at->format('M d, Y') }}
                </td>
            </tr>
            @endforeach
        </tbody>
    </table>
</div>
```

---

## Summary of Key Concepts

| Concept | Where Applied |
|---|---|
| **`@layer components`** | `app.css` — defines reusable classes that play well with Tailwind's JIT compiler |
| **Alpine.js `x-data` / `x-show` / `x-init`** | Toast notifications in `layouts/app.blade.php` |
| **Blade `@props`** | All five new components — declares expected props and sets defaults |
| **Named slots (`<x-slot>`)** | `<x-page-hero>` action slot, `<x-stat-card>` icon slot |
| **PHP `match` expression** | `<x-stat-card>` colour map — cleaner than chains of `@if` |
| **Deterministic gradient (`$id % 5`)** | `<x-event-card>` — consistent colour without storing extra data |
| **Glassmorphism (`bg-white/95 backdrop-blur-sm`)** | Sticky navbar — modern "frosted glass" effect |
| **Gradient text (`bg-clip-text text-transparent`)** | Brand logo, dashboard greeting — gradient colour through text |
| **Two-column auth layout** | `layouts/guest.blade.php` — branded left panel only on `lg:` screens |
| **Negative margin (`-mt-6`)** | Floating search card in `events/index` — overlaps the hero section |
