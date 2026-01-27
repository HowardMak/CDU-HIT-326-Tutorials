# Controllers, Routes, and Views

## Task List Summary

- [ ] **Create Event Controller** - Generate resource controller with all CRUD methods
- [ ] **Implement Index Method** - Create method to list events with pagination and eager loading
- [ ] **Implement Create Method** - Create method to show event creation form
- [ ] **Implement Store Method** - Create method to save new events with validation and file upload
- [ ] **Implement Show Method** - Create method to display single event details
- [ ] **Implement Edit Method** - Create method to show event edit form
- [ ] **Implement Update Method** - Create method to update events with image replacement
- [ ] **Implement Destroy Method** - Create method to delete events with authorization
- [ ] **Define Routes** - Set up RESTful routes for events resource
- [ ] **Create Events Index View** - Build Blade template to display list of events
- [ ] **Create Events Create View** - Build Blade template for event creation form
- [ ] **Create Events Show View** - Build Blade template to display event details
- [ ] **Create Events Edit View** - Build Blade template for event edit form
- [ ] **Add Flash Messages to Layout** - Display success/error messages in main layout
- [ ] **Test the Application** - Verify all CRUD operations work correctly
- [ ] **Add Route Model Binding (Optional)** - Configure slug-based routing

## Step 1: Create Event Controller

Open terminal and create a resource controller:

```bash
php artisan make:controller EventController --resource --model=Event
```

This creates a controller with all CRUD methods: index, create, store, show, edit, update, destroy.

## Step 2: Implement Index Method

Open `app/Http/Controllers/EventController.php` and implement the `index` method:

```php
use App\Models\Event;
use App\Models\Category;

public function index()
{
    $events = Event::published()
        ->upcoming()
        ->with('organizer', 'category')
        ->latest('date')
        ->paginate(15);

    $categories = Category::all();

    return view('events.index', compact('events', 'categories'));
}
```

## Step 3: Implement Create Method

Add the `create` method:

```php
use Illuminate\Support\Facades\Auth;

public function create()
{
    $this->authorize('create', Event::class);
  
    $categories = Category::all();
    return view('events.create', compact('categories'));
}
```

## Step 4: Implement Store Method

Add the `store` method with validation:

```php
use Illuminate\Support\Facades\Storage;

public function store(Request $request)
{
    $this->authorize('create', Event::class);

    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'description' => 'nullable|string',
        'date' => 'required|date|after:today',
        'time' => 'required',
        'location' => 'required|string|max:255',
        'capacity' => 'nullable|integer|min:0',
        'category_id' => 'nullable|exists:categories,id',
        'image' => 'nullable|image|max:2048',
    ]);

    $validated['organizer_id'] = auth()->id();
    $validated['status'] = 'draft';

    if ($request->hasFile('image')) {
        $validated['image'] = $request->file('image')->store('events', 'public');
    }

    $event = Event::create($validated);

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event created successfully.');
}
```

## Step 5: Implement Show Method

Add the `show` method:

```php
public function show(Event $event)
{
    $event->load('organizer', 'category', 'registrants');
  
    $isRegistered = auth()->check() 
        ? $event->registrants->contains(auth()->id())
        : false;

    return view('events.show', compact('event', 'isRegistered'));
}
```

## Step 6: Implement Edit Method

Add the `edit` method:

```php
public function edit(Event $event)
{
    $this->authorize('update', $event);

    $categories = Category::all();
    return view('events.edit', compact('event', 'categories'));
}
```

## Step 7: Implement Update Method

Add the `update` method:

```php
use Illuminate\Support\Facades\Storage;

public function update(Request $request, Event $event)
{
    $this->authorize('update', $event);

    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'description' => 'nullable|string',
        'date' => 'required|date',
        'time' => 'required',
        'location' => 'required|string|max:255',
        'capacity' => 'nullable|integer|min:0',
        'category_id' => 'nullable|exists:categories,id',
        'status' => 'required|in:draft,published,cancelled',
        'image' => 'nullable|image|max:2048',
    ]);

    if ($request->hasFile('image')) {
        // Delete old image
        if ($event->image) {
            Storage::disk('public')->delete($event->image);
        }
        $validated['image'] = $request->file('image')->store('events', 'public');
    }

    $event->update($validated);

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event updated successfully.');
}
```

## Step 8: Implement Destroy Method

Add the `destroy` method:

```php
public function destroy(Event $event)
{
    $this->authorize('delete', $event);

    $event->delete();

    return redirect()
        ->route('events.index')
        ->with('success', 'Event deleted successfully.');
}
```

## Step 9: Define Routes

Open `routes/web.php` and add event routes:

```php
use App\Http\Controllers\EventController;

// Event routes (protected by auth middleware)
Route::middleware(['auth'])->group(function () {
    Route::resource('events', EventController::class);
});
```

This creates all RESTful routes:

- GET `/events` - index
- GET `/events/create` - create
- POST `/events` - store
- GET `/events/{event}` - show
- GET `/events/{event}/edit` - edit
- PUT/PATCH `/events/{event}` - update
- DELETE `/events/{event}` - destroy

## Step 10: Create Events Index View

Create `resources/views/events/index.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Events')

@section('content')
<div class="py-12">
    <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
        <div class="flex justify-between items-center mb-6">
            <h2 class="text-2xl font-bold">Events</h2>
    
            @can('create', App\Models\Event::class)
                <a href="{{ route('events.create') }}" class="btn-primary">
                    Create Event
                </a>
            @endcan
        </div>

        @if(session('success'))
            <div class="alert alert-success mb-4">
                {{ session('success') }}
            </div>
        @endif

        @if($events->count() > 0)
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                @foreach($events as $event)
                    <div class="bg-white rounded-lg shadow-md p-6">
                        <h3 class="text-xl font-semibold mb-2">
                            <a href="{{ route('events.show', $event) }}">
                                {{ $event->title }}
                            </a>
                        </h3>
                        <p class="text-gray-600 mb-4">{{ Str::limit($event->description, 100) }}</p>
                        <div class="text-sm text-gray-500">
                            <p>Date: {{ $event->date->format('M d, Y') }}</p>
                            <p>Location: {{ $event->location }}</p>
                            @if($event->category)
                                <p>Category: {{ $event->category->name }}</p>
                            @endif
                        </div>
                    </div>
                @endforeach
            </div>

            <div class="mt-6">
                {{ $events->links() }}
            </div>
        @else
            <p class="text-center text-gray-500">No events found.</p>
        @endif
    </div>
</div>
@endsection
```

## Step 11: Create Events Create View

Create `resources/views/events/create.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Create Event')

@section('content')
<div class="py-12">
    <div class="max-w-3xl mx-auto sm:px-6 lg:px-8">
        <h2 class="text-2xl font-bold mb-6">Create Event</h2>

        <form method="POST" action="{{ route('events.store') }}" enctype="multipart/form-data">
            @csrf

            <div class="mb-4">
                <label for="title" class="block font-medium">Title *</label>
                <input type="text" id="title" name="title" value="{{ old('title') }}" 
                       class="w-full rounded-md" required>
                @error('title')
                    <p class="text-red-500 text-sm">{{ $message }}</p>
                @enderror
            </div>

            <div class="mb-4">
                <label for="description" class="block font-medium">Description</label>
                <textarea id="description" name="description" rows="4" 
                          class="w-full rounded-md">{{ old('description') }}</textarea>
                @error('description')
                    <p class="text-red-500 text-sm">{{ $message }}</p>
                @enderror
            </div>

            <div class="grid grid-cols-2 gap-4 mb-4">
                <div>
                    <label for="date" class="block font-medium">Date *</label>
                    <input type="date" id="date" name="date" value="{{ old('date') }}" 
                           class="w-full rounded-md" required>
                    @error('date')
                        <p class="text-red-500 text-sm">{{ $message }}</p>
                    @enderror
                </div>

                <div>
                    <label for="time" class="block font-medium">Time *</label>
                    <input type="time" id="time" name="time" value="{{ old('time') }}" 
                           class="w-full rounded-md" required>
                    @error('time')
                        <p class="text-red-500 text-sm">{{ $message }}</p>
                    @enderror
                </div>
            </div>

            <div class="mb-4">
                <label for="location" class="block font-medium">Location *</label>
                <input type="text" id="location" name="location" value="{{ old('location') }}" 
                       class="w-full rounded-md" required>
                @error('location')
                    <p class="text-red-500 text-sm">{{ $message }}</p>
                @enderror
            </div>

            <div class="grid grid-cols-2 gap-4 mb-4">
                <div>
                    <label for="capacity" class="block font-medium">Capacity</label>
                    <input type="number" id="capacity" name="capacity" value="{{ old('capacity', 0) }}" 
                           min="0" class="w-full rounded-md">
                    @error('capacity')
                        <p class="text-red-500 text-sm">{{ $message }}</p>
                    @enderror
                </div>

                <div>
                    <label for="category_id" class="block font-medium">Category</label>
                    <select id="category_id" name="category_id" class="w-full rounded-md">
                        <option value="">Select Category</option>
                        @foreach($categories as $category)
                            <option value="{{ $category->id }}" 
                                    {{ old('category_id') == $category->id ? 'selected' : '' }}>
                                {{ $category->name }}
                            </option>
                        @endforeach
                    </select>
                    @error('category_id')
                        <p class="text-red-500 text-sm">{{ $message }}</p>
                    @enderror
                </div>
            </div>

            <div class="mb-4">
                <label for="image" class="block font-medium">Event Image</label>
                <input type="file" id="image" name="image" accept="image/*" class="w-full">
                @error('image')
                    <p class="text-red-500 text-sm">{{ $message }}</p>
                @enderror
            </div>

            <div class="flex items-center justify-end space-x-4">
                <a href="{{ route('events.index') }}" class="btn-secondary">Cancel</a>
                <button type="submit" class="btn-primary">Create Event</button>
            </div>
        </form>
    </div>
</div>
@endsection
```

## Step 12: Create Events Show View

Create `resources/views/events/show.blade.php`:

```blade
@extends('layouts.app')

@section('title', $event->title)

@section('content')
<div class="py-12">
    <div class="max-w-4xl mx-auto sm:px-6 lg:px-8">
        @if(session('success'))
            <div class="alert alert-success mb-4">
                {{ session('success') }}
            </div>
        @endif

        <div class="bg-white rounded-lg shadow-md p-6">
            <div class="flex justify-between items-start mb-4">
                <h1 class="text-3xl font-bold">{{ $event->title }}</h1>
                @can('update', $event)
                    <a href="{{ route('events.edit', $event) }}" class="btn-secondary">
                        Edit
                    </a>
                @endcan
            </div>

            @if($event->image)
                <img src="{{ asset('storage/' . $event->image) }}" 
                     alt="{{ $event->title }}" 
                     class="w-full h-64 object-cover rounded-lg mb-4">
            @endif

            <div class="mb-4">
                <p class="text-gray-700">{{ $event->description }}</p>
            </div>

            <div class="grid grid-cols-2 gap-4 mb-4">
                <div>
                    <strong>Date:</strong> {{ $event->date->format('F d, Y') }}
                </div>
                <div>
                    <strong>Time:</strong> {{ $event->time }}
                </div>
                <div>
                    <strong>Location:</strong> {{ $event->location }}
                </div>
                <div>
                    <strong>Capacity:</strong> 
                    {{ $event->capacity > 0 ? $event->capacity : 'Unlimited' }}
                </div>
                @if($event->category)
                    <div>
                        <strong>Category:</strong> {{ $event->category->name }}
                    </div>
                @endif
                <div>
                    <strong>Status:</strong> {{ ucfirst($event->status) }}
                </div>
            </div>

            <div class="mb-4">
                <strong>Organized by:</strong> {{ $event->organizer->name }}
            </div>

            @auth
                @if(!$isRegistered && $event->status === 'published')
                    <form method="POST" action="{{ route('events.register', $event) }}">
                        @csrf
                        <button type="submit" class="btn-primary">
                            Register for Event
                        </button>
                    </form>
                @elseif($isRegistered)
                    <p class="text-green-600">You are registered for this event.</p>
                @endif
            @else
                <a href="{{ route('login') }}" class="btn-primary">
                    Login to Register
                </a>
            @endauth
        </div>
    </div>
</div>
@endsection
```

## Step 13: Create Events Edit View

Create `resources/views/events/edit.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Edit Event')

@section('content')
<div class="py-12">
    <div class="max-w-3xl mx-auto sm:px-6 lg:px-8">
        <h2 class="text-2xl font-bold mb-6">Edit Event</h2>

        <form method="POST" action="{{ route('events.update', $event) }}" enctype="multipart/form-data">
            @csrf
            @method('PUT')

            <!-- Same form fields as create, but with existing values -->
            <div class="mb-4">
                <label for="title" class="block font-medium">Title *</label>
                <input type="text" id="title" name="title" 
                       value="{{ old('title', $event->title) }}" 
                       class="w-full rounded-md" required>
                @error('title')
                    <p class="text-red-500 text-sm">{{ $message }}</p>
                @enderror
            </div>

            <!-- Add all other fields similar to create form -->
            <!-- Include status field for edit -->
            <div class="mb-4">
                <label for="status" class="block font-medium">Status *</label>
                <select id="status" name="status" class="w-full rounded-md" required>
                    <option value="draft" {{ old('status', $event->status) == 'draft' ? 'selected' : '' }}>
                        Draft
                    </option>
                    <option value="published" {{ old('status', $event->status) == 'published' ? 'selected' : '' }}>
                        Published
                    </option>
                    <option value="cancelled" {{ old('status', $event->status) == 'cancelled' ? 'selected' : '' }}>
                        Cancelled
                    </option>
                </select>
                @error('status')
                    <p class="text-red-500 text-sm">{{ $message }}</p>
                @enderror
            </div>

            @if($event->image)
                <div class="mb-4">
                    <img src="{{ asset('storage/' . $event->image) }}" 
                         alt="Current image" class="h-32">
                </div>
            @endif

            <div class="flex items-center justify-end space-x-4">
                <a href="{{ route('events.show', $event) }}" class="btn-secondary">Cancel</a>
                <button type="submit" class="btn-primary">Update Event</button>
            </div>
        </form>
    </div>
</div>
@endsection
```

## Step 14: Add Flash Messages to Layout

Update your main layout file to display flash messages. Add to `resources/views/layouts/app.blade.php`:

```blade
@if(session('success'))
    <div class="alert alert-success">
        {{ session('success') }}
    </div>
@endif

@if(session('error'))
    <div class="alert alert-danger">
        {{ session('error') }}
    </div>
@endif
```

## Step 15: Test the Application

1. **Start the server:**

   ```bash
   php artisan serve
   ```
2. **Test routes:**

   - Visit `http://localhost:8000/events` - Should show events list
   - Visit `http://localhost:8000/events/create` - Should show create form (if logged in as organizer)
   - Create an event and verify it appears in the list
   - Click on an event to view details
   - Edit an event and verify changes
3. **Check route list:**

   ```bash
   php artisan route:list --name=events
   ```

## Step 16: Add Route Model Binding (Optional)

If you want to use slugs instead of IDs in URLs, add to `app/Providers/RouteServiceProvider.php`:

```php
use Illuminate\Support\Facades\Route;

public function boot(): void
{
    Route::bind('event', function ($value) {
        return Event::where('slug', $value)->firstOrFail();
    });
}
```

Then update routes to use slug:

```php
Route::get('/events/{event:slug}', [EventController::class, 'show']);
```

## Complete EventController Example

Here's the complete controller file:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Event;
use App\Models\Category;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Auth;

class EventController extends Controller
{
    public function index()
    {
        $events = Event::published()
            ->upcoming()
            ->with('organizer', 'category')
            ->latest('date')
            ->paginate(15);

        $categories = Category::all();
        return view('events.index', compact('events', 'categories'));
    }

    public function create()
    {
        $this->authorize('create', Event::class);
        $categories = Category::all();
        return view('events.create', compact('categories'));
    }

    public function store(Request $request)
    {
        $this->authorize('create', Event::class);

        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'description' => 'nullable|string',
            'date' => 'required|date|after:today',
            'time' => 'required',
            'location' => 'required|string|max:255',
            'capacity' => 'nullable|integer|min:0',
            'category_id' => 'nullable|exists:categories,id',
            'image' => 'nullable|image|max:2048',
        ]);

        $validated['organizer_id'] = auth()->id();
        $validated['status'] = 'draft';

        if ($request->hasFile('image')) {
            $validated['image'] = $request->file('image')->store('events', 'public');
        }

        $event = Event::create($validated);

        return redirect()
            ->route('events.show', $event)
            ->with('success', 'Event created successfully.');
    }

    public function show(Event $event)
    {
        $event->load('organizer', 'category', 'registrants');
        $isRegistered = auth()->check() 
            ? $event->registrants->contains(auth()->id())
            : false;
        return view('events.show', compact('event', 'isRegistered'));
    }

    public function edit(Event $event)
    {
        $this->authorize('update', $event);
        $categories = Category::all();
        return view('events.edit', compact('event', 'categories'));
    }

    public function update(Request $request, Event $event)
    {
        $this->authorize('update', $event);

        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'description' => 'nullable|string',
            'date' => 'required|date',
            'time' => 'required',
            'location' => 'required|string|max:255',
            'capacity' => 'nullable|integer|min:0',
            'category_id' => 'nullable|exists:categories,id',
            'status' => 'required|in:draft,published,cancelled',
            'image' => 'nullable|image|max:2048',
        ]);

        if ($request->hasFile('image')) {
            if ($event->image) {
                Storage::disk('public')->delete($event->image);
            }
            $validated['image'] = $request->file('image')->store('events', 'public');
        }

        $event->update($validated);

        return redirect()
            ->route('events.show', $event)
            ->with('success', 'Event updated successfully.');
    }

    public function destroy(Event $event)
    {
        $this->authorize('delete', $event);
        $event->delete();
        return redirect()
            ->route('events.index')
            ->with('success', 'Event deleted successfully.');
    }
}
```
