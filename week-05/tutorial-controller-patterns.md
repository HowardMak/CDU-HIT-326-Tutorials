# Controllers, Routes, and Views

## Before You Start

Complete these earlier weeks first. If you skip them, the code in this tutorial may fail.

- Week 3: relationships, scopes, and the event slug mutator already exist
- Week 4: Laravel Breeze authentication is installed and working
- Run `php artisan storage:link` once so uploaded images to display in the browser

---

## Task List Summary

- [ ] Create the resource controller
- [ ] Add controller imports
- [ ] Implement `index()`
- [ ] Implement `create()`
- [ ] Implement `store()`
- [ ] Implement `show()`
- [ ] Implement `edit()`
- [ ] Implement `update()`
- [ ] Implement `destroy()`
- [ ] Define public and protected routes correctly
- [ ] Build the index view
- [ ] Build the create view
- [ ] Build the show view
- [ ] Build the edit view
- [ ] Add flash messages to the main layout
- [ ] Test the CRUD flow
- [ ] Add optional slug route model binding

---

## Part A: Build the Controller

## Step 1: Create Event Controller

Open your terminal and run:

```bash
# Create a controller with all 7 resource methods.
php artisan make:controller EventController --resource --model=Event
```

Laravel creates these methods for you:

- `index`
- `create`
- `store`
- `show`
- `edit`
- `update`
- `destroy`

## Step 2: Add Controller Imports

Open `app/Http/Controllers/EventController.php` and make sure these imports exist:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Category;
use App\Models\Event;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
```

## Step 3: Implement `index()`

This page should be public, so it only shows published upcoming events.

```php
public function index()
{
    // Start with published events so drafts stay hidden from public users.
    $events = Event::published()
        // Only show events that are today or later.
        ->upcoming()
        // Load related data now to avoid extra queries in the Blade file.
        ->with(['organizer', 'category'])
        // Show the soonest event first.
        ->orderBy('date')
        ->orderBy('time')
        // Split the list into pages so it stays manageable.
        ->paginate(15);

    // Send the event list to the view.
    return view('events.index', compact('events'));
}
```

## Step 4: Implement `create()`

Only organizers and admins should see the create form.

```php
public function create(Request $request)
{
    // Only organizers and admins should create events.
    abort_if(! in_array($request->user()->role, ['admin', 'organizer']), 403);

    // Categories are used in the dropdown list.
    $categories = Category::orderBy('name')->get();

    // Show the create form.
    return view('events.create', compact('categories'));
}
```

## Step 5: Implement `store()`

This method validates the form, uploads the image if needed, and saves the event.

```php
public function store(Request $request)
{
    // Only organizers and admins should create events.
    abort_if(! in_array($request->user()->role, ['admin', 'organizer']), 403);

    // Validate the incoming form data before saving anything.
    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'description' => 'nullable|string',
        'date' => 'required|date|after_or_equal:today',
        'time' => 'required|date_format:H:i',
        'location' => 'required|string|max:255',
        'capacity' => 'nullable|integer|min:0',
        'category_id' => 'nullable|exists:categories,id',
        'image' => 'nullable|image|max:2048',
    ]);

    // Save the logged-in user as the organizer.
    $validated['organizer_id'] = $request->user()->id;

    // New events start as drafts until the organizer publishes them.
    $validated['status'] = 'draft';

    // If the user uploaded an image, store it in storage/app/public/events.
    if ($request->hasFile('image')) {
        $validated['image'] = $request->file('image')->store('events', 'public');
    }

    // Create the event record in the database.
    $event = Event::create($validated);

    // Send the user to the new event page with a success message.
    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event created successfully.');
}
```

## Step 6: Implement `show()`

The show page is public, so keep it simple and only load the related data the page actually needs.

```php
public function show(Event $event)
{
    // Load the related models used on the page.
    $event->load(['organizer', 'category']);

    // Show the event details page.
    return view('events.show', compact('event'));
}
```

## Step 7: Implement `edit()`

Only the event owner or an admin should be allowed to edit an event.

```php
public function edit(Request $request, Event $event)
{
    // Allow the event owner or an admin to open the edit form.
    abort_if(
        $request->user()->role !== 'admin' && $request->user()->id !== $event->organizer_id,
        403
    );

    // Categories are needed for the category dropdown.
    $categories = Category::orderBy('name')->get();

    // Show the edit form with the existing event data.
    return view('events.edit', compact('event', 'categories'));
}
```

## Step 8: Implement `update()`

This method updates the event and replaces the image when a new file is uploaded.

```php
public function update(Request $request, Event $event)
{
    // Allow the event owner or an admin to update the event.
    abort_if(
        $request->user()->role !== 'admin' && $request->user()->id !== $event->organizer_id,
        403
    );

    // Validate the edited form data.
    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'description' => 'nullable|string',
        'date' => 'required|date',
        'time' => 'required|date_format:H:i',
        'location' => 'required|string|max:255',
        'capacity' => 'nullable|integer|min:0',
        'category_id' => 'nullable|exists:categories,id',
        'status' => 'required|in:draft,published,cancelled',
        'image' => 'nullable|image|max:2048',
    ]);

    // If a new image was uploaded, remove the old one first.
    if ($request->hasFile('image')) {
        if ($event->image) {
            Storage::disk('public')->delete($event->image);
        }

        $validated['image'] = $request->file('image')->store('events', 'public');
    }

    // Save the updated values to the database.
    $event->update($validated);

    // Return to the event page with a success message.
    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event updated successfully.');
}
```

## Step 9: Implement `destroy()`

Delete access should also be limited to the event owner or an admin.

```php
public function destroy(Request $request, Event $event)
{
    // Allow the event owner or an admin to delete the event.
    abort_if(
        $request->user()->role !== 'admin' && $request->user()->id !== $event->organizer_id,
        403
    );

    // Soft-delete the event record from the database.
    $event->delete();

    // Go back to the event list with a success message.
    return redirect()
        ->route('events.index')
        ->with('success', 'Event deleted successfully.');
}
```

---

## Part B: Define the Routes

## Step 10: Add Event Routes

Open `routes/web.php` and add these routes:

```php
use App\Http\Controllers\EventController;
use Illuminate\Support\Facades\Route;

// Anyone can view the event list and event details.
Route::get('events', [EventController::class, 'index'])->name('events.index');

Route::get('events/{event}', [EventController::class, 'show'])
    ->whereNumber('event')
    ->name('events.show');

// Logged-in users can create, edit, update, and delete events.
Route::middleware('auth')->group(function () {
    // other routes
    Route::get('events/create', [EventController::class, 'create'])->name('events.create');
    Route::post('events', [EventController::class, 'store'])->name('events.store');
    Route::get('events/{event}/edit', [EventController::class, 'edit'])
        ->whereNumber('event')
        ->name('events.edit');
    Route::match(['put', 'patch'], 'events/{event}', [EventController::class, 'update'])
        ->whereNumber('event')
        ->name('events.update');
    Route::delete('events/{event}', [EventController::class, 'destroy'])
        ->whereNumber('event')
        ->name('events.destroy');
});
```

Why this structure is better:

- `index` and `show` stay public
- create, edit, update, and delete stay protected
- it matches the controller logic and the later Blade views

This creates these main routes:

- `GET /events` -> `index`
- `GET /events/create` -> `create`
- `POST /events` -> `store`
- `GET /events/{event}` -> `show`
- `GET /events/{event}/edit` -> `edit`
- `PUT/PATCH /events/{event}` -> `update`
- `DELETE /events/{event}` -> `destroy`

---

## Part C: Build the Views

## Step 11: Create `resources/views/events/index.blade.php`

```blade
<!-- this page uses resources/views/layouts/app.blade.php as its main layout -->
@extends('layouts.app')

@section('title', 'Events')

<!-- everything below this belongs to the content part of the layout -->
@section('content')
    <div class="py-12">
        <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
            <div class="flex justify-between items-center mb-6">
                <h1 class="text-2xl font-bold">Events</h1>

                @auth
                    @if (in_array(Auth::user()->role, ['admin', 'organizer']))
                        <a href="{{ route('events.create') }}" class="btn-primary">
                            Create Event
                        </a>
                    @endif
                @endauth
            </div>

            @if ($events->count())
                <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                    @foreach ($events as $event)
                        <article class="bg-white rounded-lg shadow-md p-6">
                            <h2 class="text-xl font-semibold mb-2">
                                <a href="{{ route('events.show', $event) }}" class="hover:underline">
                                    {{ $event->title }}
                                </a>
                            </h2>

                            {{-- Show a short preview so the card stays neat. --}}
                            <p class="text-gray-600 mb-4">
                                {{ \Illuminate\Support\Str::limit($event->description ?? 'No description provided yet.', 100) }}
                            </p>

                            <div class="text-sm text-gray-500 space-y-1">
                                <p>Date: {{ $event->date->format('M d, Y') }}</p>
                                <p>Time: {{ \Carbon\Carbon::parse($event->time)->format('g:i A') }}</p>
                                <p>Location: {{ $event->location }}</p>
                                <p>Organizer: {{ $event->organizer->name }}</p>

                                @if ($event->category)
                                    <p>Category: {{ $event->category->name }}</p>
                                @endif
                            </div>
                        </article>
                    @endforeach
                </div>

                <div class="mt-6">
                    {{ $events->links() }}
                </div>
            @else
                <div class="bg-white rounded-lg shadow-md p-6 text-center text-gray-500">
                    No published upcoming events were found.
                </div>
            @endif
        </div>
    </div>
@endsection
```

## Step 12: Create `resources/views/events/create.blade.php`

```blade
@extends('layouts.app')

@section('title', 'Create Event')

@section('content')
    <div class="py-12">
        <div class="max-w-3xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white rounded-lg shadow-md p-6">
                <h1 class="text-2xl font-bold mb-6">Create Event</h1>

                <form method="POST" action="{{ route('events.store') }}" enctype="multipart/form-data">
                    @csrf

                    {{-- Basic event details. --}}
                    <div class="mb-4">
                        <label for="title" class="block font-medium">Title *</label>
                        <input
                            type="text"
                            id="title"
                            name="title"
                            value="{{ old('title') }}"
                            class="w-full rounded-md"
                            required
                        >
                        @error('title')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="mb-4">
                        <label for="description" class="block font-medium">Description</label>
                        <textarea
                            id="description"
                            name="description"
                            rows="4"
                            class="w-full rounded-md"
                        >{{ old('description') }}</textarea>
                        @error('description')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    {{-- Date and time are grouped together because students enter them as a pair. --}}
                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                        <div>
                            <label for="date" class="block font-medium">Date *</label>
                            <input
                                type="date"
                                id="date"
                                name="date"
                                value="{{ old('date') }}"
                                class="w-full rounded-md"
                                required
                            >
                            @error('date')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>

                        <div>
                            <label for="time" class="block font-medium">Time *</label>
                            <input
                                type="time"
                                id="time"
                                name="time"
                                value="{{ old('time') }}"
                                class="w-full rounded-md"
                                required
                            >
                            @error('time')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>
                    </div>

                    <div class="mb-4">
                        <label for="location" class="block font-medium">Location *</label>
                        <input
                            type="text"
                            id="location"
                            name="location"
                            value="{{ old('location') }}"
                            class="w-full rounded-md"
                            required
                        >
                        @error('location')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                        <div>
                            <label for="capacity" class="block font-medium">Capacity</label>
                            <input
                                type="number"
                                id="capacity"
                                name="capacity"
                                value="{{ old('capacity') }}"
                                min="0"
                                class="w-full rounded-md"
                            >
                            @error('capacity')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>

                        <div>
                            <label for="category_id" class="block font-medium">Category</label>
                            <select id="category_id" name="category_id" class="w-full rounded-md">
                                <option value="">Select Category</option>

                                @foreach ($categories as $category)
                                    <option
                                        value="{{ $category->id }}"
                                        {{ old('category_id') == $category->id ? 'selected' : '' }}
                                    >
                                        {{ $category->name }}
                                    </option>
                                @endforeach
                            </select>
                            @error('category_id')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>
                    </div>

                    <div class="mb-6">
                        <label for="image" class="block font-medium">Event Image</label>
                        <input type="file" id="image" name="image" accept="image/*" class="w-full">
                        @error('image')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="flex items-center justify-end gap-4">
                        <a href="{{ route('events.index') }}" class="btn-secondary">Cancel</a>
                        <button type="submit" class="btn-primary">Create Event</button>
                    </div>
                </form>
            </div>
        </div>
    </div>
@endsection
```

## Step 13: Create `resources/views/events/show.blade.php`

Note: this page no longer includes the registration form. That route is introduced later in Week 8, so adding it here would create a broken example.

```blade
@extends('layouts.app')

@section('title', $event->title)

@section('content')
    <div class="py-12">
        <div class="max-w-4xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white rounded-lg shadow-md p-6">
                <div class="flex justify-between items-start gap-4 mb-6">
                    <div>
                        <h1 class="text-3xl font-bold">{{ $event->title }}</h1>
                        <p class="text-sm text-gray-500 mt-1">
                            Organized by {{ $event->organizer->name }}
                        </p>
                    </div>

                    <div class="flex gap-2">
                        @auth
                            @if (Auth::user()->role === 'admin' || Auth::id() === $event->organizer_id)
                                <a href="{{ route('events.edit', $event) }}" class="btn-secondary">
                                    Edit
                                </a>

                                <form method="POST" action="{{ route('events.destroy', $event) }}">
                                    @csrf
                                    @method('DELETE')

                                    <button
                                        type="submit"
                                        class="btn-danger"
                                        onclick="return confirm('Are you sure you want to delete this event?')"
                                    >
                                        Delete
                                    </button>
                                </form>
                            @endif
                        @endauth
                    </div>
                </div>

                @if ($event->image)
                    {{-- Only show the image section when a file exists. --}}
                    <img
                        src="{{ asset('storage/' . $event->image) }}"
                        alt="{{ $event->title }}"
                        class="w-full h-64 object-cover rounded-lg mb-6"
                    >
                @endif

                <div class="prose max-w-none mb-6">
                    <p>{{ $event->description ?: 'No description provided yet.' }}</p>
                </div>

                <div class="grid grid-cols-1 md:grid-cols-2 gap-4 text-gray-700">
                    <div>
                        <strong>Date:</strong>
                        {{ $event->date->format('F d, Y') }}
                    </div>

                    <div>
                        <strong>Time:</strong>
                        {{ \Carbon\Carbon::parse($event->time)->format('g:i A') }}
                    </div>

                    <div>
                        <strong>Location:</strong>
                        {{ $event->location }}
                    </div>

                    <div>
                        <strong>Capacity:</strong>
                        {{ $event->capacity ? $event->capacity : 'Unlimited' }}
                    </div>

                    <div>
                        <strong>Status:</strong>
                        {{ ucfirst($event->status) }}
                    </div>

                    @if ($event->category)
                        <div>
                            <strong>Category:</strong>
                            {{ $event->category->name }}
                        </div>
                    @endif
                </div>
            </div>
        </div>
    </div>
@endsection
```

## Step 14: Create `resources/views/events/edit.blade.php`

The original example only showed the title and status fields, but the `update()` method validates more fields than that. The full form below matches the controller properly.

```blade
@extends('layouts.app')

@section('title', 'Edit Event')

@section('content')
    <div class="py-12">
        <div class="max-w-3xl mx-auto sm:px-6 lg:px-8">
            <div class="bg-white rounded-lg shadow-md p-6">
                <h1 class="text-2xl font-bold mb-6">Edit Event</h1>

                <form method="POST" action="{{ route('events.update', $event) }}" enctype="multipart/form-data">
                    @csrf
                    @method('PUT')

                    {{-- Reuse the same fields as the create page, but prefill existing values. --}}
                    <div class="mb-4">
                        <label for="title" class="block font-medium">Title *</label>
                        <input
                            type="text"
                            id="title"
                            name="title"
                            value="{{ old('title', $event->title) }}"
                            class="w-full rounded-md"
                            required
                        >
                        @error('title')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="mb-4">
                        <label for="description" class="block font-medium">Description</label>
                        <textarea
                            id="description"
                            name="description"
                            rows="4"
                            class="w-full rounded-md"
                        >{{ old('description', $event->description) }}</textarea>
                        @error('description')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                        <div>
                            <label for="date" class="block font-medium">Date *</label>
                            <input
                                type="date"
                                id="date"
                                name="date"
                                value="{{ old('date', $event->date->format('Y-m-d')) }}"
                                class="w-full rounded-md"
                                required
                            >
                            @error('date')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>

                        <div>
                            <label for="time" class="block font-medium">Time *</label>
                            <input
                                type="time"
                                id="time"
                                name="time"
                                value="{{ old('time', \Carbon\Carbon::parse($event->time)->format('H:i')) }}"
                                class="w-full rounded-md"
                                required
                            >
                            @error('time')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>
                    </div>

                    <div class="mb-4">
                        <label for="location" class="block font-medium">Location *</label>
                        <input
                            type="text"
                            id="location"
                            name="location"
                            value="{{ old('location', $event->location) }}"
                            class="w-full rounded-md"
                            required
                        >
                        @error('location')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="grid grid-cols-1 md:grid-cols-2 gap-4 mb-4">
                        <div>
                            <label for="capacity" class="block font-medium">Capacity</label>
                            <input
                                type="number"
                                id="capacity"
                                name="capacity"
                                value="{{ old('capacity', $event->capacity) }}"
                                min="0"
                                class="w-full rounded-md"
                            >
                            @error('capacity')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>

                        <div>
                            <label for="category_id" class="block font-medium">Category</label>
                            <select id="category_id" name="category_id" class="w-full rounded-md">
                                <option value="">Select Category</option>

                                @foreach ($categories as $category)
                                    <option
                                        value="{{ $category->id }}"
                                        {{ old('category_id', $event->category_id) == $category->id ? 'selected' : '' }}
                                    >
                                        {{ $category->name }}
                                    </option>
                                @endforeach
                            </select>
                            @error('category_id')
                                <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                            @enderror
                        </div>
                    </div>

                    <div class="mb-4">
                        <label for="status" class="block font-medium">Status *</label>
                        <select id="status" name="status" class="w-full rounded-md" required>
                            <option value="draft" {{ old('status', $event->status) === 'draft' ? 'selected' : '' }}>
                                Draft
                            </option>
                            <option value="published" {{ old('status', $event->status) === 'published' ? 'selected' : '' }}>
                                Published
                            </option>
                            <option value="cancelled" {{ old('status', $event->status) === 'cancelled' ? 'selected' : '' }}>
                                Cancelled
                            </option>
                        </select>
                        @error('status')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    <div class="mb-4">
                        <label for="image" class="block font-medium">Replace Image</label>
                        <input type="file" id="image" name="image" accept="image/*" class="w-full">
                        @error('image')
                            <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
                        @enderror
                    </div>

                    @if ($event->image)
                        <div class="mb-6">
                            <p class="font-medium mb-2">Current Image</p>
                            <img
                                src="{{ asset('storage/' . $event->image) }}"
                                alt="Current event image"
                                class="h-32 rounded-lg"
                            >
                        </div>
                    @endif

                    <div class="flex items-center justify-end gap-4">
                        <a href="{{ route('events.show', $event) }}" class="btn-secondary">Cancel</a>
                        <button type="submit" class="btn-primary">Update Event</button>
                    </div>
                </form>
            </div>
        </div>
    </div>
@endsection
```

## Step 15: Fix bugs on `resources/views/layouts/app.blade.php`

Replace the file wit the following code:

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta name="csrf-token" content="{{ csrf_token() }}">

        <title>@yield('title', config('app.name', 'Laravel'))</title>

        <!-- Fonts -->
        <link rel="preconnect" href="https://fonts.bunny.net">
        <link href="https://fonts.bunny.net/css?family=figtree:400,500,600&display=swap" rel="stylesheet" />

        <!-- Scripts -->
        @vite(['resources/css/app.css', 'resources/js/app.js'])
    </head>
    <body class="font-sans antialiased">
        @if (session('success'))
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8 mt-4">
                <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded">
                    {{ session('success') }}
                </div>
            </div>
        @endif

        @if (session('error'))
            <div class="max-w-7xl mx-auto sm:px-6 lg:px-8 mt-4">
                <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded">
                    {{ session('error') }}
                </div>
            </div>
        @endif
        <div class="min-h-screen bg-gray-100">
            @include('layouts.navigation')

            <!-- Page Heading -->
            @isset($header)
                <header class="bg-white shadow">
                    <div class="max-w-7xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
                        {{ $header }}
                    </div>
                </header>
            @endisset

            <!-- Page Content -->
            <main>
                @hasSection('content')
                    @yield('content')
                @else
                    {{ $slot ?? '' }}
                @endif
            </main>
        </div>
    </body>
</html>
```

## Step 16: Fix bugs on `resources/views/layouts/navigation.blade.php`

Replace the file wit the following code:

```blade
<nav x-data="{ open: false }" class="bg-white border-b border-gray-100">
    <!-- Primary Navigation Menu -->
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
        <div class="flex justify-between h-16">
            <div class="flex">
                <!-- Logo -->
                <div class="shrink-0 flex items-center">
                    <a href="{{ route('events.index') }}">
                        <x-application-logo class="block h-9 w-auto fill-current text-gray-800" />
                    </a>
                </div>

                <!-- Navigation Links -->
                <div class="hidden space-x-8 sm:-my-px sm:ms-10 sm:flex">
                    <x-nav-link :href="route('events.index')" :active="request()->routeIs('events.*')">
                        {{ __('Events') }}
                    </x-nav-link>

                    @auth
                        <x-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
                            {{ __('Dashboard') }}
                        </x-nav-link>
                    @endauth
                </div>
            </div>

            <!-- Settings Dropdown -->
            <div class="hidden sm:flex sm:items-center sm:ms-6">
                @auth
                    <x-dropdown align="right" width="48">
                        <x-slot name="trigger">
                            <button class="inline-flex items-center px-3 py-2 border border-transparent text-sm leading-4 font-medium rounded-md text-gray-500 bg-white hover:text-gray-700 focus:outline-none transition ease-in-out duration-150">
                                <div>{{ Auth::user()->name }}</div>

                                <div class="ms-1">
                                    <svg class="fill-current h-4 w-4" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20">
                                        <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
                                    </svg>
                                </div>
                            </button>
                        </x-slot>

                        <x-slot name="content">
                            <x-dropdown-link :href="route('profile.edit')">
                                {{ __('Profile') }}
                            </x-dropdown-link>

                            <!-- Authentication -->
                            <form method="POST" action="{{ route('logout') }}">
                                @csrf

                                <x-dropdown-link :href="route('logout')"
                                        onclick="event.preventDefault();
                                                    this.closest('form').submit();">
                                    {{ __('Log Out') }}
                                </x-dropdown-link>
                            </form>
                        </x-slot>
                    </x-dropdown>
                @else
                    <div class="hidden sm:flex sm:items-center sm:gap-4">
                        <x-nav-link :href="route('login')" :active="request()->routeIs('login')">
                            {{ __('Log In') }}
                        </x-nav-link>

                        @if (Route::has('register'))
                            <x-nav-link :href="route('register')" :active="request()->routeIs('register')">
                                {{ __('Register') }}
                            </x-nav-link>
                        @endif
                    </div>
                @endauth
            </div>

            <!-- Hamburger -->
            <div class="-me-2 flex items-center sm:hidden">
                <button @click="open = ! open" class="inline-flex items-center justify-center p-2 rounded-md text-gray-400 hover:text-gray-500 hover:bg-gray-100 focus:outline-none focus:bg-gray-100 focus:text-gray-500 transition duration-150 ease-in-out">
                    <svg class="h-6 w-6" stroke="currentColor" fill="none" viewBox="0 0 24 24">
                        <path :class="{'hidden': open, 'inline-flex': ! open }" class="inline-flex" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16" />
                        <path :class="{'hidden': ! open, 'inline-flex': open }" class="hidden" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12" />
                    </svg>
                </button>
            </div>
        </div>
    </div>

    <!-- Responsive Navigation Menu -->
    <div :class="{'block': open, 'hidden': ! open}" class="hidden sm:hidden">
        <div class="pt-2 pb-3 space-y-1">
            <x-responsive-nav-link :href="route('events.index')" :active="request()->routeIs('events.*')">
                {{ __('Events') }}
            </x-responsive-nav-link>

            @auth
                <x-responsive-nav-link :href="route('dashboard')" :active="request()->routeIs('dashboard')">
                    {{ __('Dashboard') }}
                </x-responsive-nav-link>
            @endauth
        </div>

        <!-- Responsive Settings Options -->
        @auth
            <div class="pt-4 pb-1 border-t border-gray-200">
                <div class="px-4">
                    <div class="font-medium text-base text-gray-800">{{ Auth::user()->name }}</div>
                    <div class="font-medium text-sm text-gray-500">{{ Auth::user()->email }}</div>
                </div>

                <div class="mt-3 space-y-1">
                    <x-responsive-nav-link :href="route('profile.edit')">
                        {{ __('Profile') }}
                    </x-responsive-nav-link>

                    <!-- Authentication -->
                    <form method="POST" action="{{ route('logout') }}">
                        @csrf

                        <x-responsive-nav-link :href="route('logout')"
                                onclick="event.preventDefault();
                                            this.closest('form').submit();">
                            {{ __('Log Out') }}
                        </x-responsive-nav-link>
                    </form>
                </div>
            </div>
        @else
            <div class="pt-4 pb-1 border-t border-gray-200">
                <div class="mt-3 space-y-1">
                    <x-responsive-nav-link :href="route('login')" :active="request()->routeIs('login')">
                        {{ __('Log In') }}
                    </x-responsive-nav-link>

                    @if (Route::has('register'))
                        <x-responsive-nav-link :href="route('register')" :active="request()->routeIs('register')">
                            {{ __('Register') }}
                        </x-responsive-nav-link>
                    @endif
                </div>
            </div>
        @endauth
    </div>
</nav>
```

---

## Part D: Test the Application

## Step 17: Manual Testing Checklist

Start the Laravel development server:

```bash
# Start the local development server.
php artisan serve
```

Then test these pages and actions:

1. Visit `http://localhost:8000/events`
2. Confirm the index page loads without logging in
3. Log in as a user who is allowed to create events
4. Visit `http://localhost:8000/events/create`
5. Create a new event with and without an image
6. Open the event details page
7. Edit the event and confirm the changes save correctly
8. Delete the event and confirm you return to the index page

Check the route list too:

```bash
# Show routes whose names contain "events".
php artisan route:list --name=events
```