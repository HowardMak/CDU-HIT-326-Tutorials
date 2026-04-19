# Event Registration System

## Task List Summary

- [ ] **Create Registration Controller** - Generate controller for handling event registrations
- [ ] **Implement Register Method** - Create method to register users for events with validation
- [ ] **Implement Cancel Method** - Create method to cancel user registrations
- [ ] **Add Helper Methods to Event Model** - Add isFull(), confirmed_registrations_count, and available_spots methods
- [ ] **Add Registration Routes** - Set up POST and DELETE routes for registration actions
- [ ] **Add Registration Button to Event Show Page** - Display register/cancel button with capacity checks
- [ ] **Create User Dashboard Controller** - Generate DashboardController for registered events
- [ ] **Create Dashboard View** - Build view to display user's registered events with filters
- [ ] **Add Dashboard Route** - Configure route for user dashboard
- [ ] **Create Organizer View for Registrants** - Add method to view all event registrants
- [ ] **Create Registrants View** - Build view to display list of event registrants
- [ ] **Add Link to View Registrants** - Add navigation link for organizers to view registrants
- [ ] **Test Registration Flow** - Verify registration, duplicate prevention, capacity, and cancellation
- [ ] **Add Registration Count to Event Index** - Display registration count on events list page

## Step 1: Create Registration Controller

Create a controller for handling registrations:

```bash
php artisan make:controller RegistrationController
```

## Step 2: Implement Register Method

Open `app/Http/Controllers/RegistrationController.php` and add:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Event;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class RegistrationController extends Controller
{
    public function store(Request $request, Event $event)
    {
        // Check if user is authenticated (Auth facade — IDE-friendly)
        if (! Auth::check()) {
            return redirect()->route('login')
                ->with('error', 'Please login to register for events.');
        }

        $user = Auth::user();

        // Check if already registered
        if ($event->registrants->contains($user->id)) {
            return redirect()->back()
                ->with('error', 'You are already registered for this event.');
        }

        // Check if event is published
        if ($event->status !== 'published') {
            return redirect()->back()
                ->with('error', 'This event is not available for registration.');
        }

        // Check if event is full
        $status = 'confirmed';
        if ($event->isFull()) {
            $status = 'waitlisted';
            $event->registrants()->attach($user->id, [
                'status' => $status,
                'registered_at' => now(),
            ]);
            return redirect()->back()
                ->with('info', 'Event is full. You have been added to the waitlist.');
        }

        // Create registration
        $event->registrants()->attach($user->id, [
            'status' => $status,
            'registered_at' => now(),
        ]);

        return redirect()->back()
            ->with('success', 'Successfully registered for the event.');
    }

    public function destroy(Event $event)
    {
        $user = Auth::user();

        // Check if user is registered
        if (!$event->registrants->contains($user->id)) {
            return redirect()->back()
                ->with('error', 'You are not registered for this event.');
        }

        // Remove registration
        $event->registrants()->detach($user->id);

        return redirect()->back()
            ->with('success', 'Registration cancelled successfully.');
    }
}
```

## Step 3: Add Helper Methods to Event Model

Open `app/Models/Event.php` and add these methods:

```php
/**
 * Check if event is full (capacity reached).
 */
public function isFull()
{
    if ($this->capacity == 0) {
        return false; // Unlimited capacity
    }
    return $this->confirmed_registrations_count >= $this->capacity;
}

/**
 * Get confirmed registrations count.
 */
public function getConfirmedRegistrationsCountAttribute()
{
    // If the relation is already loaded (e.g. via withCount or eager load),
    // filter the collection to avoid an extra DB query per event (N+1).
    if ($this->relationLoaded('registrants')) {
        return $this->registrants->where('pivot.status', 'confirmed')->count();
    }
    return $this->registrants()->wherePivot('status', 'confirmed')->count();
}

/**
 * Check if user is registered for this event.
 */
public function isRegisteredBy($userId)
{
    return $this->registrants->contains($userId);
}

/**
 * Get available spots.
 */
public function getAvailableSpotsAttribute()
{
    if ($this->capacity == 0) {
        return null; // Unlimited
    }
    return max(0, $this->capacity - $this->confirmed_registrations_count);
}
```

## Step 4: Add Registration Routes

Open `routes/web.php` and add the `use` imports at the **top of the file** alongside the other imports, then add the routes:

```php
// Add these use statements at the top of web.php (missing imports cause "route not defined" errors)
use App\Http\Controllers\RegistrationController;
use App\Http\Controllers\DashboardController;

Route::middleware(['auth'])->group(function () {
    Route::post('/events/{event}/register', [RegistrationController::class, 'store'])
        ->name('events.register');
    Route::delete('/events/{event}/register', [RegistrationController::class, 'destroy'])
        ->name('events.unregister');
});
```

## Step 5: Add Registration Button to Event Show Page

Update `resources/views/events/show.blade.php`:
Add it after the event details grid

```blade
<div class="mt-6">
    @auth
        @php
            $isRegistered = $event->registrants->contains(auth()->id());
        @endphp

        @if(!$isRegistered && $event->status === 'published')
            @if($event->isFull())
                <div class="bg-yellow-100 border border-yellow-400 text-yellow-700 px-4 py-3 rounded mb-4">
                    <p>Event is full. You can join the waitlist.</p>
                </div>
                <form method="POST" action="{{ route('events.register', $event) }}">
                    @csrf
                    <button type="submit" class="btn-primary">
                        Join Waitlist
                    </button>
                </form>
            @else
                <form method="POST" action="{{ route('events.register', $event) }}">
                    @csrf
                    <button type="submit" class="btn-primary">
                        Register for Event
                    </button>
                </form>
            @endif
        @elseif(!$isRegistered)
            {{-- Event exists but is not published — show a notice instead of silently hiding the button --}}
            <div class="bg-gray-100 border border-gray-300 text-gray-600 px-4 py-3 rounded mb-4">
                <p>Registration is not available for this event.</p>
            </div>
        @elseif($isRegistered)
            <div class="bg-green-100 border border-green-400 text-green-700 px-4 py-3 rounded mb-4">
                <p>You are registered for this event.</p>
            </div>
            <form method="POST" action="{{ route('events.unregister', $event) }}">
                @csrf
                @method('DELETE')
                <button type="submit" class="btn-danger">
                    Cancel Registration
                </button>
            </form>
        @endif
    @else
        <a href="{{ route('login') }}" class="btn-primary">
            Login to Register
        </a>
    @endauth

    <!-- Show registration count -->
    <div class="mt-4">
        <p class="text-sm text-gray-600">
            <strong>Registrations:</strong> 
            {{ $event->confirmed_registrations_count }} 
            @if($event->capacity > 0)
                / {{ $event->capacity }}
            @else
                (Unlimited)
            @endif
        </p>
        @if($event->capacity > 0 && !$event->isFull())
            <p class="text-sm text-green-600">
                {{ $event->available_spots }} spots available
            </p>
        @endif
    </div>
</div>
```

## Step 6: Create User Dashboard for Registered Events

Create a controller for user dashboard:

```bash
php artisan make:controller DashboardController
```

Open `app/Http/Controllers/DashboardController.php`:

```php
<?php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class DashboardController extends Controller
{
    public function index(Request $request)
    {
        $user = Auth::user();
  
        $query = $user->registeredEvents()
            ->with('category', 'organizer');

        // Filter by status
        if ($request->filled('status')) {
            $query->wherePivot('status', $request->status);
        }

        // Filter upcoming/past
        if ($request->filled('filter')) {
            if ($request->filter === 'upcoming') {
                $query->where('date', '>=', now()->toDateString());
            } elseif ($request->filter === 'past') {
                $query->where('date', '<', now()->toDateString());
            }
        }

        $registeredEvents = $query->orderBy('date')->paginate(15);

        return view('dashboard', compact('registeredEvents'));
    }
}
```

## Step 7: Create Dashboard View

Update `resources/views/dashboard.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Dashboard')

@section('content')
<div class="py-12">
    <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
        <h2 class="text-2xl font-bold mb-6">My Registered Events</h2>

        <!-- Filters -->
        <div class="mb-6 flex space-x-4">
            <a href="{{ route('dashboard', ['filter' => 'upcoming']) }}" 
               class="btn-secondary">Upcoming</a>
            <a href="{{ route('dashboard', ['filter' => 'past']) }}" 
               class="btn-secondary">Past</a>
            <a href="{{ route('dashboard') }}" 
               class="btn-secondary">All</a>
        </div>

        @if($registeredEvents->count() > 0)
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                @foreach($registeredEvents as $event)
                    <div class="bg-white rounded-lg shadow-md p-6">
                        <h3 class="text-xl font-semibold mb-2">
                            <a href="{{ route('events.show', $event) }}">
                                {{ $event->title }}
                            </a>
                        </h3>
                        <p class="text-sm text-gray-600 mb-2">
                            Date: {{ $event->date->format('M d, Y') }}
                        </p>
                        <p class="text-sm text-gray-600 mb-2">
                            Location: {{ $event->location }}
                        </p>
                        <p class="text-sm mb-4">
                            Status: 
                            <span class="px-2 py-1 rounded 
                                {{ $event->pivot->status === 'confirmed' ? 'bg-green-100 text-green-800' : '' }}
                                {{ $event->pivot->status === 'waitlisted' ? 'bg-yellow-100 text-yellow-800' : '' }}">
                                {{ ucfirst($event->pivot->status) }}
                            </span>
                        </p>
                        <p class="text-xs text-gray-500">
                            Registered: {{ $event->pivot->registered_at->format('M d, Y') }}
                        </p>
                    </div>
                @endforeach
            </div>

            <div class="mt-6">
                {{ $registeredEvents->links() }}
            </div>
        @else
            <p class="text-center text-gray-500">You haven't registered for any events yet.</p>
            <div class="text-center mt-4">
                <a href="{{ route('events.index') }}" class="btn-primary">
                    Browse Events
                </a>
            </div>
        @endif
    </div>
</div>
@endsection
```

## Step 8: Add Dashboard Route

In `routes/web.php`:

```php
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    // ... other routes
});
```

## Step 9: Create Organizer View for Registrants

Add method to `EventController.php`:

```php
public function registrants(Event $event)
{
    $this->authorize('viewRegistrants', $event);

    $registrants = $event->registrants()
        ->withPivot('status', 'registered_at')
        ->orderBy('registrations.created_at')
        ->get();

    return view('events.registrants', compact('event', 'registrants'));
}
```

Add route:

```php
Route::get('/events/{event}/registrants', [EventController::class, 'registrants'])
    ->name('events.registrants')
    ->middleware('auth');
```

## Step 10: Create Registrants View

Create `resources/views/events/registrants.blade.php`:

```blade
@extends('layouts.app')

@section('title', 'Event Registrants')

@section('content')
<div class="py-12">
    <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
        <div class="flex justify-between items-center mb-6">
            <h2 class="text-2xl font-bold">Registrants for: {{ $event->title }}</h2>
            <a href="{{ route('events.show', $event) }}" class="btn-secondary">
                Back to Event
            </a>
        </div>

        <div class="bg-white rounded-lg shadow-md p-6">
            <div class="mb-4">
                <p><strong>Total Registrations:</strong> {{ $registrants->count() }}</p>
                <p><strong>Confirmed:</strong> {{ $registrants->where('pivot.status', 'confirmed')->count() }}</p>
                <p><strong>Waitlisted:</strong> {{ $registrants->where('pivot.status', 'waitlisted')->count() }}</p>
            </div>

            <table class="min-w-full divide-y divide-gray-200">
                <thead class="bg-gray-50">
                    <tr>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Name</th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Email</th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Status</th>
                        <th class="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase">Registered At</th>
                    </tr>
                </thead>
                <tbody class="bg-white divide-y divide-gray-200">
                    @foreach($registrants as $registrant)
                        <tr>
                            <td class="px-6 py-4 whitespace-nowrap">{{ $registrant->name }}</td>
                            <td class="px-6 py-4 whitespace-nowrap">{{ $registrant->email }}</td>
                            <td class="px-6 py-4 whitespace-nowrap">
                                <span class="px-2 py-1 text-xs rounded 
                                    {{ $registrant->pivot->status === 'confirmed' ? 'bg-green-100 text-green-800' : 'bg-yellow-100 text-yellow-800' }}">
                                    {{ ucfirst($registrant->pivot->status) }}
                                </span>
                            </td>
                            <td class="px-6 py-4 whitespace-nowrap">
                                {{ $registrant->pivot->registered_at->format('M d, Y H:i') }}
                            </td>
                        </tr>
                    @endforeach
                </tbody>
            </table>
        </div>
    </div>
</div>
@endsection
```

## Step 11: Add Link to View Registrants

In `resources/views/events/show.blade.php`, add for organizers:

Put it in the existing organizer buttons area (<div class="flex gap-2">), alongside the Edit and Delete buttons:

```blade
@can('update', $event)
    <div class="mb-4">
        <a href="{{ route('events.registrants', $event) }}" class="btn-secondary">
            View Registrants ({{ $event->registrants->count() }})
        </a>
    </div>
@endcan
```

## Step 12: Test Registration Flow

1. **Test Registration:**

   - Login as an attendee
   - View a published event
   - Click "Register for Event"
   - Verify success message
   - Check dashboard to see registered event
2. **Test Duplicate Prevention:**

   - Try to register for the same event again
   - Should show error message
3. **Test Capacity:**

   - Create an event with capacity of 2
   - Register 2 users
   - Try to register a 3rd user
   - Should be added to waitlist
4. **Test Cancellation:**

   - Go to event detail page
   - Click "Cancel Registration"
   - Verify registration is removed

## Step 13: Add Registration Count to Event Index

**First**, update `EventController::index()` to eager-load the registrants relation so the count accessor doesn't fire a separate query per event card (N+1):

```php
$events = Event::with(['organizer', 'category', 'registrants'])
    ->where('status', 'published')
    ->orderBy('date')
    ->paginate(12);
```

**Then**, update `resources/views/events/index.blade.php` to show the count:

Add the registration count inside the existing `<div class="text-sm text-gray-500 space-y-1">`, after the other details:

```blade
<p>
    <strong>Registrations:</strong> 
    {{ $event->confirmed_registrations_count }}
    @if($event->capacity > 0)
        / {{ $event->capacity }}
    @endif
</p>
```

## Complete Registration Flow

1. **User views event** → Sees registration button (if published and not registered)
2. **User clicks register** → Controller checks:
   - Authentication
   - Duplicate registration
   - Event status
   - Capacity
3. **Registration created** → Status set to 'confirmed' or 'waitlisted'
4. **User sees confirmation** → Redirected with success message
5. **User views dashboard** → Sees all registered events

## Troubleshooting

**Button not visible (silent gap)**
- Cause: `@if(!$isRegistered && $event->status === 'published')` has no fallback — if the event isn't published, nothing renders.
- Fix: Add `@elseif(!$isRegistered)` to show a "Registration not available" notice.

**`isFull()` — undefined method error**
- Cause: `getConfirmedRegistrationsCountAttribute()` called `$this->registrations()`, which doesn't exist. The correct relationship is `registrants()`.
- Fix: Use `$this->registrants()->wherePivot('status', 'confirmed')->count()`.

**Route `events.register` not defined**
- Cause: `RegistrationController` and `DashboardController` used in `web.php` without `use` imports — Laravel can't resolve the class.
- Fix: Add the missing `use` statements at the top of `web.php`.

**Registration count not showing on index page (N+1 query)**
- Cause 1: `EventController::index()` only eager loaded `organizer` and `category` — the `confirmed_registrations_count` accessor fired a new COUNT query for every event card.
- Cause 2: The accessor always called `$this->registrants()->wherePivot(...)` (fresh query) even when registrants were already loaded.
- Fix 1: Add `'registrants'` to the `with([...])` call in `EventController::index()`.
- Fix 2: In the accessor, check `$this->relationLoaded('registrants')` first and filter the collection instead of hitting the DB again.

**Controller method still undefined after adding imports**
- Cause: Stale compiled class/route cache from before the imports were added.
- Fix: Run the following, then restart `php artisan serve`:
  ```bash
  composer dump-autoload && php artisan optimize:clear
  ```