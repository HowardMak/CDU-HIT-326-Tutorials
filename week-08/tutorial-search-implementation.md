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

> **Prerequisite — add `isOrganizer()` to the User model first.**
> The `index` method below calls `auth()->user()->isOrganizer()`. If that method is missing you will get a `BadMethodCallException` and a 500 error on the `/events` page. Open `app/Models/User.php` and add this method before continuing:
>
> ```php
> public function isOrganizer(): bool
> {
>     // Treat both admins and organizers as organizers for visibility purposes
>     return in_array($this->role, ['admin', 'organizer']);
> }
> ```

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

            <div class="mt-4 flex space-x-2 gap-4">
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

// Import the Category model so we can query the categories table
use App\Models\Category;
// Import Request so we can receive and validate form data
use Illuminate\Http\Request;
// Import Str so we can generate URL-friendly slugs (e.g. "My Category" -> "my-category")
use Illuminate\Support\Str;

class CategoryController extends Controller
{
    // This runs automatically before every method in this controller
    public function __construct()
    {
        // Only logged-in users can access any category page
        $this->middleware('auth');
        // On top of that, only admins are allowed (regular users are blocked)
        $this->middleware('role:admin');
    }

    // Shows the list of all categories (GET /categories)
    public function index()
    {
        // Fetch all categories from the database, 15 per page.
        // withCount('events') adds an extra column telling us how many events
        // belong to each category — useful for displaying in the table.
        $categories = Category::withCount('events')->paginate(15);

        // Pass the $categories variable to the view so the blade template can loop over it
        return view('categories.index', compact('categories'));
    }

    // Saves a brand-new category submitted from the create form (POST /categories)
    public function store(Request $request)
    {
        // Validate the incoming form data before touching the database.
        // 'required' = field must not be empty
        // 'unique:categories' = the name must not already exist in the categories table
        $validated = $request->validate([
            'name' => 'required|string|max:255|unique:categories',
            'description' => 'nullable|string', // description is optional
        ]);

        // Insert a new row into the categories table.
        // Str::slug() converts the name into a URL-safe string, e.g. "My Category" -> "my-category"
        Category::create([
            'name' => $validated['name'],
            'slug' => Str::slug($validated['name']),
            'description' => $validated['description'],
        ]);

        // Send the admin back to the category list with a success message
        return redirect()->route('categories.index')
            ->with('success', 'Category created successfully.');
    }

    // Saves changes to an existing category (PUT /categories/{category})
    // Laravel automatically finds the Category record matching the ID in the URL
    public function update(Request $request, Category $category)
    {
        // Same validation as store, but the unique check ignores the *current* category's
        // own row so it doesn't falsely report a duplicate when the name hasn't changed.
        // The ", $category->id" part tells Laravel: "skip this ID when checking uniqueness"
        $validated = $request->validate([
            'name' => 'required|string|max:255|unique:categories,name,' . $category->id,
            'description' => 'nullable|string',
        ]);

        // Update only the columns we care about; leave everything else untouched
        $category->update([
            'name' => $validated['name'],
            'slug' => Str::slug($validated['name']), // regenerate slug in case the name changed
            'description' => $validated['description'],
        ]);

        // Redirect back to the list with a success message
        return redirect()->route('categories.index')
            ->with('success', 'Category updated successfully.');
    }

    // Deletes a category from the database (DELETE /categories/{category})
    public function destroy(Category $category)
    {
        // Remove this category's row from the database permanently
        $category->delete();

        // Send the admin back to the category list with a confirmation message
        return redirect()->route('categories.index')
            ->with('success', 'Category deleted successfully.');
    }
}
```

## Step 5: Add Category Routes

In `routes/web.php`:

```php
use App\Http\Controllers\CategoryController;

// Apply two protection layers to every route inside this group:
//   'auth'       – the user must be logged in
//   'role:admin' – the user must also have the admin role
Route::middleware(['auth', 'role:admin'])->group(function () {
    Route::resource('categories', CategoryController::class);
});
```

## Step 6: Add Category Navigation

Update your navigation to show categories. In `resources/views/layouts/navigation.blade.php` at the end of the navigation links:

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
<?php

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

Update the index view to show result count. Put it between the "Active filters" block and the "Events List":

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
