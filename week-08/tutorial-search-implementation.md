# Search and Categories

## Task List Summary

- [ ] **Add Search Scope to Event Model** - Create search scope method for filtering events
- [ ] **Update Index Method with Search and Filters** - Add search, category, and date filtering to controller
- [ ] **Add Search Form to Index View** - Create search and filter form in events index page
- [ ] **Create Category Management (Admin Only)** - Build CategoryController for admin category management
- [ ] **Add Category Routes** - Set up routes for category CRUD operations
- [ ] **Add Category Navigation** - Display categories in navigation and make available to all views
- [ ] **Add Search Highlighting (Optional)** - Create helper function to highlight search terms in results
- [ ] **Test Search Functionality** - Verify search, category filter, date range, and pagination work
- [ ] **Add Search Results Count** - Display number of search results to users
- [ ] **Optimize Search with Full-Text Index (Optional)** - Add full-text index for better search performance

## Step 1: Add Search Scope to Event Model

Open `app/Models/Event.php` and add a search scope:

```php
/**
 * Scope to search events.
 */
public function scopeSearch($query, $term)
{
    if (empty($term)) {
        return $query;
    }

    return $query->where(function($q) use ($term) {
        $q->where('title', 'like', "%{$term}%")
          ->orWhere('description', 'like', "%{$term}%")
          ->orWhere('location', 'like', "%{$term}%");
    });
}
```

## Step 2: Update Index Method with Search and Filters

> **This replaces the `index` method you wrote in Week 7.** It keeps the role-based visibility logic from Week 7 and adds search, date range, and pagination parameter preservation.

Open `app/Http/Controllers/EventController.php` and update the `index` method:

```php
public function index(Request $request)
{
    $query = Event::query();

    // Access control from Week 7 — keep published-only for guests/attendees,
    // allow organizers to filter by status.
    if (!auth()->check() || !auth()->user()->isOrganizer()) {
        // Guests and regular attendees can only see published events.
        $query->published();
    } else {
        // Organizers can see all statuses, with an optional status filter.
        if ($request->filled('status')) {
            $query->where('status', $request->status);
        }
    }

    // NEW: Search across title, description, and location.
    if ($request->filled('search')) {
        $query->search($request->search);
    }

    // Filter by category
    if ($request->filled('category')) {
        $query->byCategory($request->category);
    }

    // Filter by date range (added date_to compared to Week 7)
    if ($request->filled('date_from')) {
        $query->where('date', '>=', $request->date_from);
    }

    if ($request->filled('date_to')) {
        $query->where('date', '<=', $request->date_to);
    }

    // NEW: appends() preserves search/filter params across paginated pages.
    $events = $query->with('organizer', 'category')
        ->latest('date')
        ->paginate(15)
        ->appends($request->query());

    $categories = Category::all();

    return view('events.index', compact('events', 'categories'));
}
```

## Step 3: Add Search Form to Index View

Update `resources/views/events/index.blade.php` to include search and filter form:

```blade
@extends('layouts.app')

@section('title', 'Events')

@section('content')
<div class="py-12">
    <div class="max-w-7xl mx-auto sm:px-6 lg:px-8">
        <h2 class="text-2xl font-bold mb-6">Events</h2>

        <!-- Search and Filter Form -->
        <form method="GET" action="{{ route('events.index') }}" class="mb-6 bg-white p-4 rounded-lg shadow-md">
            <div class="grid grid-cols-1 md:grid-cols-4 gap-4">
                <!-- Search Input -->
                <div>
                    <label for="search" class="block text-sm font-medium mb-1">Search</label>
                    <input type="text" 
                           id="search" 
                           name="search" 
                           value="{{ request('search') }}"
                           placeholder="Search events..."
                           class="w-full rounded-md border-gray-300">
                </div>

                <!-- Category Filter -->
                <div>
                    <label for="category" class="block text-sm font-medium mb-1">Category</label>
                    <select id="category" name="category" class="w-full rounded-md border-gray-300">
                        <option value="">All Categories</option>
                        @foreach($categories as $category)
                            <option value="{{ $category->id }}" 
                                    {{ request('category') == $category->id ? 'selected' : '' }}>
                                {{ $category->name }}
                            </option>
                        @endforeach
                    </select>
                </div>

                <!-- Date From -->
                <div>
                    <label for="date_from" class="block text-sm font-medium mb-1">From Date</label>
                    <input type="date" 
                           id="date_from" 
                           name="date_from" 
                           value="{{ request('date_from') }}"
                           class="w-full rounded-md border-gray-300">
                </div>

                <!-- Date To -->
                <div>
                    <label for="date_to" class="block text-sm font-medium mb-1">To Date</label>
                    <input type="date" 
                           id="date_to" 
                           name="date_to" 
                           value="{{ request('date_to') }}"
                           class="w-full rounded-md border-gray-300">
                </div>
            </div>

            <div class="mt-4 flex space-x-2">
                <button type="submit" class="btn-primary">Search</button>
                <a href="{{ route('events.index') }}" class="btn-secondary">Clear Filters</a>
            </div>
        </form>

        <!-- Show active filters -->
        @if(request()->hasAny(['search', 'category', 'date_from', 'date_to']))
            <div class="mb-4 text-sm text-gray-600">
                <strong>Active filters:</strong>
                @if(request('search'))
                    Search: "{{ request('search') }}"
                @endif
                @if(request('category'))
                    | Category: {{ $categories->find(request('category'))->name ?? '' }}
                @endif
                @if(request('date_from'))
                    | From: {{ request('date_from') }}
                @endif
                @if(request('date_to'))
                    | To: {{ request('date_to') }}
                @endif
            </div>
        @endif

        <!-- Events List -->
        @if($events->count() > 0)
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
                @foreach($events as $event)
                    <div class="bg-white rounded-lg shadow-md p-6">
                        <h3 class="text-xl font-semibold mb-2">
                            <a href="{{ route('events.show', $event) }}" class="hover:text-blue-600">
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

            <!-- Pagination with preserved search parameters -->
            <div class="mt-6">
                {{ $events->links() }}
            </div>
        @else
            <div class="text-center py-12">
                <p class="text-gray-500 text-lg">No events found.</p>
                @if(request()->hasAny(['search', 'category', 'date_from', 'date_to']))
                    <a href="{{ route('events.index') }}" class="btn-primary mt-4">
                        Clear Filters
                    </a>
                @endif
            </div>
        @endif
    </div>
</div>
@endsection
```

## Step 4: Create Category Management (Admin Only)

Create CategoryController:

```bash
php artisan make:controller CategoryController --resource
```

Open `app/Http/Controllers/CategoryController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Category;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class CategoryController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth');
        $this->middleware('role:admin');
    }

    public function index()
    {
        $categories = Category::withCount('events')->paginate(15);
        return view('categories.index', compact('categories'));
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255|unique:categories',
            'description' => 'nullable|string',
        ]);

        Category::create([
            'name' => $validated['name'],
            'slug' => Str::slug($validated['name']),
            'description' => $validated['description'],
        ]);

        return redirect()->route('categories.index')
            ->with('success', 'Category created successfully.');
    }

    public function update(Request $request, Category $category)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255|unique:categories,name,' . $category->id,
            'description' => 'nullable|string',
        ]);

        $category->update([
            'name' => $validated['name'],
            'slug' => Str::slug($validated['name']),
            'description' => $validated['description'],
        ]);

        return redirect()->route('categories.index')
            ->with('success', 'Category updated successfully.');
    }

    public function destroy(Category $category)
    {
        $category->delete();
        return redirect()->route('categories.index')
            ->with('success', 'Category deleted successfully.');
    }
}
```

## Step 5: Add Category Routes

In `routes/web.php`:

```php
use App\Http\Controllers\CategoryController;

Route::middleware(['auth', 'role:admin'])->group(function () {
    Route::resource('categories', CategoryController::class);
});
```

## Step 6: Add Category Navigation

Update your navigation to show categories. In `resources/views/layouts/navigation.blade.php` or main layout:

```blade
<div class="flex space-x-4">
    <a href="{{ route('events.index') }}">All Events</a>
  
    @foreach($categories as $category)
        <a href="{{ route('events.index', ['category' => $category->id]) }}">
            {{ $category->name }}
        </a>
    @endforeach
</div>
```

To make categories available in all views, add to `app/Providers/AppServiceProvider.php`:

```php
use Illuminate\Support\Facades\View;
use App\Models\Category;

public function boot(): void
{
    View::composer('*', function ($view) {
        $view->with('categories', Category::all());
    });
}
```

## Step 7: Add Search Highlighting (Optional)

Create a helper function. In `app/Helpers/helpers.php` (create if doesn't exist):

```php
if (!function_exists('highlight')) {
    function highlight($text, $search)
    {
        if (empty($search)) {
            return $text;
        }

        return preg_replace(
            "/\b(" . preg_quote($search, '/') . ")\b/i",
            "<mark>$1</mark>",
            $text
        );
    }
}
```

Register in `composer.json`:

```json
"autoload": {
    "files": [
        "app/Helpers/helpers.php"
    ]
}
```

Then run:

```bash
composer dump-autoload
```

Use in Blade:

```blade
{!! highlight($event->title, request('search')) !!}
```

## Step 8: Test Search Functionality

1. **Test Basic Search:**

   - Go to `/events`
   - Enter a search term (e.g., "workshop")
   - Click Search
   - Verify results show matching events
2. **Test Category Filter:**

   - Select a category from dropdown
   - Click Search
   - Verify only events in that category are shown
3. **Test Date Range:**

   - Select date_from and date_to
   - Click Search
   - Verify only events in date range are shown
4. **Test Combined Filters:**

   - Use search + category + date range together
   - Verify all filters work together
5. **Test Pagination:**

   - Search for results
   - Click page 2
   - Verify search parameters are preserved

## Step 9: Add Search Results Count

Update the index view to show result count:

```blade
<div class="mb-4">
    <p class="text-gray-600">
        Found {{ $events->total() }} event(s)
        @if(request('search'))
            for "{{ request('search') }}"
        @endif
    </p>
</div>
```

## Step 10: Optimize Search with Full-Text Index (Optional)

For better performance, add full-text index. Create a migration:

```bash
php artisan make:migration add_fulltext_index_to_events
```

In the migration:

```php
public function up()
{
    DB::statement('ALTER TABLE events ADD FULLTEXT(title, description, location)');
}

public function down()
{
    DB::statement('ALTER TABLE events DROP INDEX title');
}
```

Then update the search scope:

```php
public function scopeSearch($query, $term)
{
    if (empty($term)) {
        return $query;
    }

    return $query->whereRaw(
        "MATCH(title, description, location) AGAINST(? IN BOOLEAN MODE)",
        [$term]
    );
}
```

## Complete Search Flow

1. **User enters search term** → Form submits to index with search parameter
2. **Controller processes search** → Uses search scope to filter events
3. **Results displayed** → Events matching search criteria shown
4. **Pagination preserves search** → Search term maintained across pages
5. **Filters can be combined** → Search + category + date range work together
