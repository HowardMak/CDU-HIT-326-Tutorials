# Using Generative AI in Development: Best Practices

## Principle: Trust but Verify

**Never blindly trust AI-generated code.** Always:

1. Understand what the code does
2. Test it thoroughly
3. Verify it follows Laravel best practices
4. Check for security vulnerabilities
5. Ensure it integrates properly with your existing code

## Example: AI-Generated Controller Method

### AI Prompt:

"Create a Laravel controller method to get all events with their organizers and categories"

### AI Response:

```php
public function index()
{
    $events = Event::all();
    $events->load('organizer', 'category');
    return view('events.index', compact('events'));
}
```

### Issues to Identify:

1. **N+1 Query Problem:**

   - `Event::all()` loads all events
   - Then `load()` runs additional queries for each event
   - This is inefficient
2. **No Pagination:**

   - Loading all events at once can cause performance issues
   - No limit on results
3. **No Filtering:**

   - No way to filter by status, date, category
   - Not suitable for production
4. **Missing Authorization:**

   - No check if user can view events
   - Draft events might be visible to everyone

### How to Verify:

1. **Check Query Log:**

   ```php
   DB::enableQueryLog();
   $events = Event::all();
   $events->load('organizer', 'category');
   dd(DB::getQueryLog());
   ```

   You'll see multiple queries (1 for events + N for organizers + N for categories)
2. **Test with Large Dataset:**

   - Create 100+ events
   - Check page load time
   - Monitor database queries
3. **Check Laravel Debugbar:**

   - Install Debugbar
   - View queries tab
   - See actual number of queries executed

### Corrected Version:

```php
public function index(Request $request)
{
    $query = Event::with('organizer', 'category'); // Eager loading
  
    // Only show published events to guests
    if (!auth()->check() || !auth()->user()->isOrganizer()) {
        $query->published();
    }
  
    $events = $query->latest('date')->paginate(15); // Pagination
  
    $categories = Category::all();
  
    return view('events.index', compact('events', 'categories'));
}
```

### Testing the Corrected Version:

```php
// In tinker or test
DB::enableQueryLog();
$events = Event::with('organizer', 'category')->paginate(15);
dd(DB::getQueryLog());
// Should show only 3-4 queries regardless of number of events
```

## Common AI Mistakes to Watch For

### 1. Security Issues

**AI might generate:**

```php
// Directly using user input without validation
$event = Event::find($request->id);
$event->update($request->all()); // Dangerous!
```

**Problem:** No validation, mass assignment vulnerability

**Correct:**

```php
$validated = $request->validate([
    'title' => 'required|string|max:255',
    // ... other rules
]);
$event->update($validated);
```

### 2. Missing Authorization

**AI might generate:**

```php
public function destroy(Event $event)
{
    $event->delete(); // No authorization check!
    return redirect()->back();
}
```

**Problem:** Anyone can delete any event

**Correct:**

```php
public function destroy(Event $event)
{
    $this->authorize('delete', $event);
    $event->delete();
    return redirect()->back();
}
```

### 3. Incorrect Relationships

**AI might generate:**

```php
// Wrong relationship type
public function events()
{
    return $this->hasOne(Event::class); // Should be hasMany
}
```

**Problem:** Relationship doesn't match database structure

**How to verify:**

```php
// Test in tinker
$user = User::find(1);
$user->events; // Should return collection, not single model
```

### 4. Hardcoded Values

**AI might generate:**

```php
$events = Event::where('status', 'published')
    ->where('date', '>', '2024-01-01') // Hardcoded date!
    ->get();
```

**Problem:** Hardcoded date will break over time

**Correct:**

```php
$events = Event::where('status', 'published')
    ->where('date', '>', now()->toDateString())
    ->get();
```

### 5. Missing Error Handling

**AI might generate:**

```php
$event = Event::find($id);
$event->delete(); // What if $event is null?
```

**Problem:** No check if event exists

**Correct:**

```php
$event = Event::findOrFail($id);
$event->delete();
```

## Verification Checklist

Before using AI-generated code, always check:

- [ ] **Security:** No SQL injection, XSS, CSRF vulnerabilities?
- [ ] **Authorization:** Proper permission checks?
- [ ] **Validation:** Input validated and sanitized?
- [ ] **Performance:** No N+1 queries, proper indexing?
- [ ] **Error Handling:** Handles edge cases and errors?
- [ ] **Laravel Conventions:** Follows Laravel best practices?
- [ ] **Database Integrity:** Proper foreign keys, constraints?
- [ ] **Testing:** Can be tested easily?
- [ ] **Documentation:** Code is clear and understandable?

## How to Prompt AI Effectively

### Good Prompts:

1. **Be Specific:**

   ```
   "Create a Laravel controller method to list published events 
   with eager loading of organizer and category relationships, 
   with pagination of 15 items per page, and filter by category 
   if provided in the request"
   ```
2. **Include Context:**

   ```
   "In my Laravel events management system, I have an Event model 
   with relationships to User (organizer) and Category. Create a 
   method to register a user for an event, checking capacity and 
   preventing duplicates"
   ```
3. **Request Best Practices:**

   ```
   "Create a Laravel controller method following best practices: 
   use eager loading, proper validation, authorization, and 
   handle errors gracefully"
   ```

### Bad Prompts:

1. **Too Vague:**

   ```
   "Make a function to get events"

   ```
2. **No Context:**

   ```
   "Create a registration method"
   ```
3. **Asking for Everything:**

   ```
   "Build the entire events management system" 
   ```

## Resources

- [Laravel Documentation](https://laravel.com/docs) - Always verify against official docs
- [Laravel Best Practices](https://github.com/alexeymezenin/laravel-best-practices)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) - Security best practices
- [PHP The Right Way](https://phptherightway.com/) - PHP best practices
