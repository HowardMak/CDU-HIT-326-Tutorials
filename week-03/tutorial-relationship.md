# Eloquent Relationships

## Task List Summary

- [ ] **Define User Model Relationships** - Add organizedEvents and registeredEvents relationships
- [ ] **Define Event Model Relationships** - Add category, organizer, and registrants relationships
- [ ] **Define Category Model Relationship** - Add events relationship
- [ ] **Test Relationships in Tinker** - Verify all relationships work correctly
- [ ] **Implement Many-to-Many Registration** - Use attach/detach for event registrations
- [ ] **Implement Eager Loading** - Prevent N+1 query problems with with() method
- [ ] **Add Query Scopes to Event Model** - Create published, upcoming, and byCategory scopes
- [ ] **Add Accessors and Mutators** - Create fullDateTime accessor and slug mutator
- [ ] **Test Advanced Queries** - Practice counting, filtering, and whereHas queries
- [ ] **Access Pivot Table Data** - Retrieve and filter registration status from pivot table
- [ ] **Add Helper Methods** - Create isFull(), isRegisteredBy(), and other helper methods

## Step 1: Define Basic Relationships

### User Model Relationships

Open `app/Models/User.php` and add these relationships:

```php
/**
 * Get the events organized by this user.
 */
public function organizedEvents()
{
    return $this->hasMany(Event::class, 'organizer_id');
}

/**
 * Get the events this user has registered for.
 */
public function registeredEvents()
{
    return $this->belongsToMany(Event::class, 'registrations')
                ->withPivot('status', 'registered_at')
                ->withTimestamps();
}
```

### Event Model Relationships

Open `app/Models/Event.php` and add:

```php
/**
 * Get the category that owns the event.
 */
public function category()
{
    return $this->belongsTo(Category::class);
}

/**
 * Get the user that organized the event.
 */
public function organizer()
{
    return $this->belongsTo(User::class, 'organizer_id');
}

/**
 * Get the users registered for this event.
 */
public function registrants()
{
    return $this->belongsToMany(User::class, 'registrations')
                ->withPivot('status', 'registered_at')
                ->withTimestamps();
}
```

### Category Model Relationship

Open `app/Models/Category.php` and add:

```php
/**
 * Get the events for the category.
 */
public function events()
{
    return $this->hasMany(Event::class);
}
```

## Step 2: Test Relationships in Tinker

Open terminal and run:

```bash
php artisan tinker
```

Then test each relationship:

```php
// Test 1: Get all events organized by user ID 1
$user = User::find(1);
$events = $user->organizedEvents;
dd($events);

// Test 2: Get the organizer of event ID 1
$event = Event::find(1);
$organizer = $event->organizer;
dd($organizer->name);

// Test 3: Get all events in category ID 1
$category = Category::find(1);
$events = $category->events;
dd($events);
```

## Step 3: Implement Many-to-Many Registration

The relationships are already defined above. Now let's demonstrate how to use them:

```php
// Register a user for an event
$user = User::find(1);
$event = Event::find(1);

$user->registeredEvents()->attach($event->id, [
    'status' => 'confirmed',
    'registered_at' => now(),
]);

// Get user's registered events
$registeredEvents = $user->registeredEvents;
foreach ($registeredEvents as $event) {
    echo $event->title . ' - ' . $event->pivot->status . "\n";
}
```

## Step 4: Implement Eager Loading

To prevent N+1 query problems, always use eager loading:

```php
// Bad: N+1 problem (don't do this)
$events = Event::all();
foreach ($events as $event) {
    echo $event->organizer->name; // Queries database for each event
    echo $event->category->name;  // Queries database for each event
}

// Good: Eager loading
$events = Event::with('organizer', 'category')->get();
foreach ($events as $event) {
    echo $event->organizer->name; // No additional queries
    echo $event->category->name;  // No additional queries
}

// Nested eager loading
$events = Event::with('organizer.registeredEvents', 'category')->get();
```

## Step 5: Add Query Scopes to Event Model

Add these scopes to `app/Models/Event.php`:

```php
/**
 * Scope to filter published events.
 */
public function scopePublished($query)
{
    return $query->where('status', 'published');
}

/**
 * Scope to filter upcoming events.
 */
public function scopeUpcoming($query)
{
    return $query->where('date', '>=', now()->toDateString());
}

/**
 * Scope to filter events by category.
 */
public function scopeByCategory($query, $categoryId)
{
    return $query->where('category_id', $categoryId);
}
```

Use the scopes:

```php
// Get all published upcoming events
$events = Event::published()->upcoming()->get();

// Get published events in category 1
$events = Event::published()->byCategory(1)->get();

// Get published upcoming events in category 1, ordered by date
$events = Event::published()
    ->upcoming()
    ->byCategory(1)
    ->orderBy('date')
    ->get();
```

## Step 6: Add Accessors and Mutators

### Accessor Example

Add to `app/Models/Event.php`:

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

Usage:

```php
$event = Event::find(1);
echo $event->full_date_time; // "March 15, 2024 at 2:00 PM"
```

### Mutator Example (Auto-generate Slug)

Add to `app/Models/Event.php`:

```php
protected static function boot()
{
    parent::boot();
  
    static::creating(function ($event) {
        if (empty($event->slug)) {
            $event->slug = \Illuminate\Support\Str::slug($event->title);
        }
    });
}
```

## Step 7: Advanced Query Examples

### Count Relationships

```php
// Count events organized by user ID 1
$count = User::find(1)->organizedEvents()->count();

// Count registrants for event ID 1
$count = Event::find(1)->registrants()->count();

// Count events in category ID 1
$count = Category::find(1)->events()->count();
```

### Filter by Relationship

```php
// Get events organized by user ID 1 that are published
$events = User::find(1)
    ->organizedEvents()
    ->where('status', 'published')
    ->get();

// Get users who have registered for event ID 1
$users = Event::find(1)->registrants;

// Get events that have at least 5 registrations
$events = Event::has('registrants', '>=', 5)->get();
```

### Using whereHas

```php
// Get categories that have at least one published event
$categories = Category::whereHas('events', function($query) {
    $query->where('status', 'published');
})->get();

// Get events that have no registrations
$events = Event::doesntHave('registrants')->get();

// Get users who have registered for at least one event
$users = User::has('registeredEvents')->get();
```

## Step 8: Access Pivot Table Data

```php
$user = User::find(1);
$event = Event::find(1);

// Get registration status
$registeredEvent = $user->registeredEvents()
    ->where('events.id', $event->id)
    ->first();

if ($registeredEvent) {
    $status = $registeredEvent->pivot->status;
    $registeredAt = $registeredEvent->pivot->registered_at;
}

// Get user's confirmed registrations only
$confirmedRegistrations = $user->registeredEvents()
    ->wherePivot('status', 'confirmed')
    ->get();
```

## Step 9: Add Helper Methods

### Event Model Helpers

Add to `app/Models/Event.php`:

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
    return $this->registrants->contains($userId);
}

/**
 * Get confirmed registrations count.
 */
public function getConfirmedRegistrationsCountAttribute()
{
    return $this->registrations()
        ->where('status', 'confirmed')
        ->count();
}
```

### User Model Helper

Add to `app/Models/User.php`:

```php
/**
 * Check if user is registered for an event.
 */
public function isRegisteredFor($eventId)
{
    return $this->registeredEvents->contains($eventId);
}
```

## Resources

- [Laravel Eloquent Relationships](https://laravel.com/docs/eloquent-relationships)
- [Laravel Query Builder](https://laravel.com/docs/queries)
- [Laravel Collections](https://laravel.com/docs/collections)
