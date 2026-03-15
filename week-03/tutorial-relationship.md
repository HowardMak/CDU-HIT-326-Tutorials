# Eloquent Relationships

## Part A: Before You Start

### Prerequisites

- **Complete Week 2** (Database Design and Migrations) in full.
- Migrations must be run (`php artisan migrate`).
- Seeders must be run (`php artisan db:seed`) so Users, Categories, and Events exist.
- Models `Category` and `Event` must exist (`app/Models/Category.php`, `app/Models/Event.php`).

If you skip Week 2 or have not run migrations/seeders, this tutorial will fail.

### Step 0: Model Setup

Before defining relationships, ensure your models have the required configuration.

**Add to `app/Models/Event.php`:**

The `events` table uses soft deletes and the accessors below assume proper casting. Add or confirm:

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Event extends Model
{
    use SoftDeletes;

    // $fillable tells Laravel which fields are allowed to be filled when you create or update an event with a single array
    protected $fillable = [
        'title', 'slug', 'description', 'date', 'time', 'location',
        'capacity', 'status', 'image', 'category_id', 'organizer_id',
    ];

    // $casts tells Laravel how to convert values when reading from and writing to the database. Here, it converts the date column into a date object (Carbon instance) instead of a raw string.
    protected $casts = [
        'date' => 'date',
    ];
}
```

 **Add to `app/Models/User.php`:** Ensure `'role'` is in the `$fillable` array so the role column can be mass-assigned.

**Add to `app/Models/Category.php`:** Add `$fillable = ['name', 'slug', 'description']` if you use `Category::create()` in seeders.

---

## Part B: Define Relationships

**What you will do:** Edit model files to add relationship methods. No Tinker or terminal execution in this part.

### Step 1: User Model Relationships

 **Add to `app/Models/User.php`**

Ensure `use App\Models\Event;` is at the top (add it if missing), then add these methods:

```php
/**
 * Get the events organized by this user.
 */
public function organizedEvents()
{
    // It tells Laravel that one user can organize many events. The link is the organizer_id column on the events table.
    return $this->hasMany(Event::class, 'organizer_id');
}

/**
 * Get the events this user has registered for.
 */
public function registeredEvents()
{
    // Defines a many-to-many link: a user can register for many events, and an event can have many registered users. The connection is stored in the registrations table, which also holds extra details (status, date registered, etc.).
    // Like a class enrollment list: each row links a student (user) and a class (event), and has extra info such as enrollment status and date.
    return $this->belongsToMany(Event::class, 'registrations')
                ->withPivot('status', 'registered_at')
                ->withTimestamps();
}
```

### Step 2: Event Model Relationships

**Add to `app/Models/Event.php`**

Ensure these `use` statements exist at the top:

```php
use App\Models\Category;
use App\Models\User;
```

Add these relationship methods:

```php
/**
 * Get the category that owns the event.
 */
public function category()
{
    // It tells Laravel that an event belongs to one category: each event is linked to a single category via its category_id column.
    return $this->belongsTo(Category::class);
}

/**
 * Get the user that organized the event.
 */
public function organizer()
{
    // It tells Laravel that each event belongs to one user (the organizer). The link is the organizer_id column on the events table.
    return $this->belongsTo(User::class, 'organizer_id');
}

/**
 * Get the users registered for this event.
 */
public function registrants()
{
    // An event can have many registered users (registrants). The registrations table records who is registered for each event and stores status and registration date. withPivot makes those extra columns available; withTimestamps lets Laravel manage the pivot table timestamps.
    return $this->belongsToMany(User::class, 'registrations')
                ->withPivot('status', 'registered_at')
                ->withTimestamps();
}
```

### Step 3: Category Model Relationship

 **Add to `app/Models/Category.php`**

Ensure `use App\Models\Event;` is at the top (add it if missing), then add:

```php
/**
 * Get the events for the category.
 */
public function events()
{
    return $this->hasMany(Event::class);
}
```

---

## Part C: Enhance Models

**What you will do:** Add query scopes, accessors, mutators, and helper methods to the model files. All edits in this part are file changes—no Tinker.

### Step 4: Query Scopes

**Add to `app/Models/Event.php`**

```php
/**
 * Scope to filter published events.
 */
public function scopePublished($query)
{
    // Adds a filter to the query so only events with status = 'published' are returned.
    return $query->where('status', 'published');
}

/**
 * Scope to filter upcoming events.
 */
public function scopeUpcoming($query)
{
    // Adds a filter so only events on or after today’s date are returned. Past events are excluded.
    return $query->where('date', '>=', now()->toDateString());
}

/**
 * Scope to filter events by category.
 */
public function scopeByCategory($query, $categoryId)
{
    // Filters events so only those in a given category are returned. You pass the category ID as the argument.
    return $query->where('category_id', $categoryId);
}
```

### Step 5: Accessors and Mutators

**Add to `app/Models/Event.php`**

**Accessor** – formats date and time as a readable string (e.g. "March 15, 2024 at 2:00 PM"):

```php
/**
 * Get the full date and time formatted.
 */
public function getFullDateTimeAttribute()
{
    return $this->date->format('F d, Y') . ' at ' . 
           \Carbon\Carbon::parse($this->time)->format('g:i A');
}
```

**Mutator** – auto-generates slug when creating an event:

```php
protected static function boot()
{
    // Before saving a new event, if no slug was provided, Laravel builds one from the title. For example, 'Laravel Workshop' becomes 'laravel-workshop'.
    parent::boot();
    // static::creating(...) – Runs right before a new event is saved.
    static::creating(function ($event) {
        // if (empty($event->slug)) – Only runs when no slug is set.
        if (empty($event->slug)) {
            // Str::slug($event->title) – Turns the title into a slug.
            // $event->slug = ... – Saves that value in the slug field.
            $event->slug = \Illuminate\Support\Str::slug($event->title);
        }
    });
}
```

### Step 6: Helper Methods

**Add to `app/Models/Event.php`**

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
 * Check if user is registered for this event.
 */
public function isRegisteredBy($userId)
{
    return $this->registrants->contains('id', $userId);
}

/**
 * Get confirmed registrations count.
 */
public function getConfirmedRegistrationsCountAttribute()
{
    // Counts how many people have confirmed registration for this event by looking at the status column in the registrations pivot table.

    // $this->registrants() – Uses the event’s registrants relationship.
    return $this->registrants()
        // ->wherePivot('status', 'confirmed') – Filters the pivot table so only rows with status = 'confirmed' are included.
        ->wherePivot('status', 'confirmed')
        //->count() – Returns the number of matching rows
        ->count();
}
```

**Add to `app/Models/User.php`**

```php
/**
 * Check if user is registered for an event.
 */
public function isRegisteredFor($eventId)
{
    return $this->registeredEvents->contains('id', $eventId);
}
```

---

## Part D: Use Relationships

**What you will do:** Run code in **Tinker** to test and explore the relationships. Open Tinker once and run each block in sequence.

**How to run:** Open terminal, run `php artisan tinker` (if Tinker is missing: `composer require laravel/tinker`), then paste each code block.

**Important:** Use `\App\Models\Event::` (with leading backslash) instead of `Event::`. Laravel's built-in `Event` facade will otherwise cause "Method find does not exist" errors.

**Note:** Use `dump()` to inspect output without exiting. If you use `dd()`, Tinker will exit and you must run `php artisan tinker` again.

### Step 7: Test Relationships

**▶️ Run in Tinker**

```php
// Test 1: Get all events organized by user ID 1
$user = \App\Models\User::find(1);
$events = $user->organizedEvents;
dump($events);

// Test 2: Get the organizer of event ID 1
$event = \App\Models\Event::find(1);
$organizer = $event->organizer;
dump($organizer->name);

// Test 3: Get all events in category ID 1
$category = \App\Models\Category::find(1);
$events = $category->events;
dump($events);
```

### Step 8: Many-to-Many and Pivot Data

**Run in Tinker**

Register a user for an event and read pivot data:

```php
// Register a user for an event
$user = \App\Models\User::find(1);
$event = \App\Models\Event::find(1);

$user->registeredEvents()->attach($event->id, [
    'status' => 'confirmed',
    'registered_at' => now(),
]);

// Get user's registered events with pivot data
$registeredEvents = $user->registeredEvents;
foreach ($registeredEvents as $regEvent) {
    echo $regEvent->title . ' - ' . $regEvent->pivot->status . "\n";
}

// Get registration status for a specific event (use $event from above)
$registeredEvent = $user->registeredEvents()
    ->where('events.id', $event->id)
    ->first();
if ($registeredEvent) {
    dump($registeredEvent->pivot->status);
    dump($registeredEvent->pivot->registered_at);
}

// Get only confirmed registrations
$confirmedRegistrations = $user->registeredEvents()
    ->wherePivot('status', 'confirmed')
    ->get();
dump($confirmedRegistrations);
```

### Step 9: Eager Loading

**Run in Tinker**

Compare N+1 problem vs eager loading:

```php
// Bad: N+1 problem (don't do this)
$events = \App\Models\Event::all();
foreach ($events as $event) {
    echo $event->organizer->name; // Queries database for each event
    echo $event->category->name;  // Queries database for each event
}

// Good: Eager loading
$events = \App\Models\Event::with('organizer', 'category')->get();
foreach ($events as $event) {
    echo $event->organizer->name; // No additional queries
    echo $event->category->name;  // No additional queries
}

// Nested eager loading
$events = \App\Models\Event::with('organizer.registeredEvents', 'category')->get();
```

### Step 10: Advanced Queries

**Run in Tinker**

```php
// Use scopes
$events = \App\Models\Event::published()->upcoming()->get();
$events = \App\Models\Event::published()->byCategory(1)->get();
$events = \App\Models\Event::published()
    ->upcoming()
    ->byCategory(1)
    ->orderBy('date')
    ->get();

// Use accessor
$event = \App\Models\Event::find(1);
echo $event->full_date_time; // "March 15, 2024 at 2:00 PM"

// Count relationships
$count = \App\Models\User::find(1)->organizedEvents()->count();
$count = \App\Models\Event::find(1)->registrants()->count();
$count = \App\Models\Category::find(1)->events()->count();

// Filter by relationship
$events = \App\Models\User::find(1)
    ->organizedEvents()
    ->where('status', 'published')
    ->get();
$users = \App\Models\Event::find(1)->registrants;
$events = \App\Models\Event::has('registrants', '>=', 5)->get();

// Using whereHas
$categories = \App\Models\Category::whereHas('events', function($query) {
    $query->where('status', 'published');
})->get();
$events = \App\Models\Event::doesntHave('registrants')->get();
$users = \App\Models\User::has('registeredEvents')->get();
```

---

## Part E: Reference

### Task List Summary

- **Step 0** – Model setup (Event, User, Category)
- **Step 1** – User model relationships
- **Step 2** – Event model relationships
- **Step 3** – Category model relationship
- **Step 4** – Query scopes (Event)
- **Step 5** – Accessors and mutators (Event)
- **Step 6** – Helper methods (Event, User)
- **Step 7** – Test relationships (Tinker)
- **Step 8** – Many-to-many and pivot (Tinker)
- **Step 9** – Eager loading (Tinker)
- **Step 10** – Advanced queries (Tinker)

### Resources

- [Laravel Eloquent Relationships](https://laravel.com/docs/eloquent-relationships)
- [Laravel Query Builder](https://laravel.com/docs/queries)
- [Laravel Collections](https://laravel.com/docs/collections)

