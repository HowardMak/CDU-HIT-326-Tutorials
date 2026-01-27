# Query Optimization Tips

## Eager Loading

### The N+1 Problem

**Bad:**

```php
$events = Event::all();
foreach ($events as $event) {
    echo $event->organizer->name; // Queries database for each event
}
```

**Good:**

```php
$events = Event::with('organizer', 'category')->get();
foreach ($events as $event) {
    echo $event->organizer->name; // No additional queries
}
```

### Eager Load Nested Relationships

```php
$events = Event::with('organizer.registeredEvents', 'category')->get();
```

### Conditional Eager Loading

```php
$events = Event::with(['organizer' => function($query) {
    $query->where('role', 'organizer');
}])->get();
```

### Lazy Eager Loading

```php
$events = Event::all();
$events->load('organizer', 'category');
```

## Database Indexes

### Add Indexes in Migrations

```php
Schema::table('events', function (Blueprint $table) {
    $table->index('date');
    $table->index('status');
    $table->index('category_id');
    $table->index('organizer_id');
    $table->index(['status', 'date']); // Composite index
});
```

### Full-Text Index

```php
$table->fullText(['title', 'description', 'location']);
```

## Query Optimization Techniques

### Select Specific Columns

```php
// Bad: Selects all columns
$events = Event::all();

// Good: Select only needed columns
$events = Event::select('id', 'title', 'date', 'location')->get();
```

### Use Chunking for Large Datasets

```php
Event::chunk(100, function ($events) {
    foreach ($events as $event) {
        // Process event
    }
});
```

### Use Cursor for Large Results

```php
foreach (Event::cursor() as $event) {
    // Process event one at a time
}
```

## Query Scopes

### Create Reusable Scopes

```php
// Model
public function scopePublished($query)
{
    return $query->where('status', 'published');
}

public function scopeUpcoming($query)
{
    return $query->where('date', '>=', now()->toDateString());
}

// Usage
$events = Event::published()->upcoming()->get();
```

## Avoid Unnecessary Queries

### Count vs Exists

```php
// Bad: Counts all records
if (Event::where('status', 'published')->count() > 0) {
    // ...
}

// Good: Stops at first match
if (Event::where('status', 'published')->exists()) {
    // ...
}
```

### Pluck vs Get

```php
// Bad: Gets full models
$titles = Event::all()->pluck('title');

// Good: Gets only titles
$titles = Event::pluck('title');
```

## Caching Queries

### Cache Query Results

```php
$events = Cache::remember('published_events', 3600, function() {
    return Event::published()->with('category')->get();
});
```

### Cache Tags (Redis/Memcached)

```php
$events = Cache::tags(['events', 'published'])
    ->remember('published_events', 3600, function() {
        return Event::published()->get();
    });

// Clear all cached events
Cache::tags(['events'])->flush();
```

## Database Query Logging

### Enable Query Log

```php
DB::enableQueryLog();

// Your queries
$events = Event::with('organizer')->get();

// View queries
dd(DB::getQueryLog());
```

### Use Laravel Debugbar

Install:

```bash
composer require barryvdh/laravel-debugbar --dev
```

## Optimize Joins

### Use Join Instead of WhereHas

```php
// Bad: Subquery
$events = Event::whereHas('category', function($q) {
    $q->where('name', 'Workshop');
})->get();

// Good: Join
$events = Event::join('categories', 'events.category_id', '=', 'categories.id')
    ->where('categories.name', 'Workshop')
    ->select('events.*')
    ->get();
```

## Pagination Optimization

### Simple Pagination

```php
// For large datasets, use simplePaginate
$events = Event::simplePaginate(15);
```

### Cursor Pagination

```php
// For very large datasets
$events = Event::cursorPaginate(15);
```

## Lazy Collections

### Use Lazy Collections

```php
Event::cursor()->each(function ($event) {
    // Process without loading all into memory
});
```

## Database Connection Pooling

### Configure Connection Pool

In `config/database.php`:

```php
'mysql' => [
    // ...
    'options' => [
        PDO::ATTR_PERSISTENT => true,
    ],
],
```

## Monitoring and Profiling

### Use Telescope

Install:

```bash
composer require laravel/telescope --dev
php artisan telescope:install
```

### Analyze Slow Queries

```php
DB::listen(function ($query) {
    if ($query->time > 100) { // More than 100ms
        Log::warning('Slow query', [
            'sql' => $query->sql,
            'time' => $query->time,
        ]);
    }
});
```

## Best Practices

1. **Always Use Eager Loading** - Prevent N+1 queries
2. **Add Indexes** - For frequently queried columns
3. **Select Only Needed Columns** - Reduce memory usage
4. **Use Scopes** - Reusable query logic
5. **Cache Expensive Queries** - Reduce database load
6. **Monitor Query Performance** - Use debugging tools
7. **Optimize Joins** - Use joins instead of whereHas when possible
8. **Paginate Large Results** - Don't load everything at once

## Common Performance Issues

### Issue: Slow Page Load

**Solutions:**

- Check for N+1 queries
- Add database indexes
- Use eager loading
- Cache expensive queries

### Issue: High Memory Usage

**Solutions:**

- Use chunking or cursors
- Select specific columns
- Use lazy collections

### Issue: Too Many Queries

**Solutions:**

- Use eager loading
- Combine queries where possible
- Use database views

## Tools for Optimization

1. **Laravel Debugbar** - View all queries
2. **Laravel Telescope** - Monitor application performance
3. **MySQL EXPLAIN** - Analyze query execution
4. **New Relic** - Production monitoring
5. **Blackfire** - Profiling tool
