# Debugging Tools

## Task List Summary

- [ ] **Install Laravel Debugbar** - Install debugbar package for development
- [ ] **Use Logging** - Implement logging with different log levels and context
- [ ] **Use dd() and dump()** - Use dump functions for quick variable inspection
- [ ] **Enable Query Logging** - Log database queries and identify slow queries
- [ ] **Install Laravel Telescope (Optional)** - Install Telescope for application monitoring
- [ ] **Use Tinker** - Test code interactively using Tinker REPL
- [ ] **Create Custom Error Pages** - Customize 404, 500, and 403 error pages
- [ ] **Handle Exceptions** - Implement custom exception handling
- [ ] **Debug in Blade** - Use Blade directives for debugging in templates
- [ ] **Debug Common Issues** - Troubleshoot routes, database, authentication, and cache
- [ ] **Use Browser DevTools** - Leverage browser tools for frontend debugging
- [ ] **Monitor Production Errors** - Set up error tracking services for production

## Laravel Debugbar

### Installation

```bash
composer require barryvdh/laravel-debugbar --dev
```

### Usage

Debugbar automatically appears in development mode. It shows:

- Queries executed
- Route information
- View data
- Session data
- Request/Response information

## Logging

### Basic Logging

```php
use Illuminate\Support\Facades\Log;

Log::info('User registered', ['user_id' => $user->id]);
Log::error('Payment failed', ['order_id' => $order->id]);
Log::warning('Low disk space');
Log::debug('Debug information');
```

### Log Levels

- `emergency` - System is unusable
- `alert` - Action must be taken immediately
- `critical` - Critical conditions
- `error` - Error conditions
- `warning` - Warning conditions
- `notice` - Normal but significant condition
- `info` - Informational messages
- `debug` - Debug-level messages

### Context Data

```php
Log::info('Event created', [
    'event_id' => $event->id,
    'organizer_id' => $event->organizer_id,
    'title' => $event->title,
]);
```

## Dump and Die

### dd() - Dump and Die

```php
dd($variable);
dd($event, $user, $category);
```

### dump() - Dump Without Dying

```php
dump($variable);
// Code continues execution
```

### Dump Multiple Variables

```php
dump($event, $user, $category);
```

## Query Logging

### Enable Query Log

```php
DB::enableQueryLog();

// Your queries
$events = Event::with('organizer')->get();

// View queries
dd(DB::getQueryLog());
```

### Log Slow Queries

```php
DB::listen(function ($query) {
    if ($query->time > 100) {
        Log::warning('Slow query', [
            'sql' => $query->sql,
            'bindings' => $query->bindings,
            'time' => $query->time,
        ]);
    }
});
```

## Laravel Telescope

### Installation

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

### Access Telescope

Visit `/telescope` to view:

- Requests
- Queries
- Commands
- Jobs
- Mail
- Notifications
- Cache
- Logs

## Tinker

### Access Tinker

```bash
php artisan tinker
```

### Use Tinker

```php
// In tinker
$event = Event::find(1);
$event->title;
$event->organizer->name;

// Create records
Event::create(['title' => 'Test', ...]);

// Test relationships
$user = User::find(1);
$user->organizedEvents;
```

## Error Pages

### Custom Error Pages

Create in `resources/views/errors/`:

- `404.blade.php` - Not found
- `500.blade.php` - Server error
- `403.blade.php` - Forbidden

### Example 404 Page

```blade
@extends('layouts.app')

@section('content')
<div class="text-center">
    <h1>404</h1>
    <p>Page not found</p>
    <a href="{{ route('home') }}">Go Home</a>
</div>
@endsection
```

## Exception Handling

### Report Exceptions

```php
try {
    // Your code
} catch (\Exception $e) {
    report($e);
    return redirect()->back()->with('error', 'Something went wrong.');
}
```

### Custom Exception Handler

In `app/Exceptions/Handler.php`:

```php
public function render($request, Throwable $exception)
{
    if ($exception instanceof ModelNotFoundException) {
        return response()->json(['error' => 'Resource not found'], 404);
    }

    return parent::render($request, $exception);
}
```

## Debugging in Blade

### Dump in Blade

```blade
@dump($variable)
@dd($variable)
```

### Debug Directives

```blade
@if(config('app.debug'))
    <pre>{{ print_r($data, true) }}</pre>
@endif
```

## Common Debugging Scenarios

### Debug Route Not Found

```php
// List all routes
php artisan route:list

// Check specific route
php artisan route:list --name=events.show
```

### Debug Database Issues

```php
// Check connection
DB::connection()->getPdo();

// Test query
DB::select('SELECT * FROM events LIMIT 1');
```

### Debug Authentication

```php
// Check if user is authenticated
dd(auth()->check());
dd(auth()->user());

// Check user roles
dd(auth()->user()->role);
```

### Debug Cache

```php
// Clear cache
php artisan cache:clear
php artisan config:clear
php artisan view:clear
php artisan route:clear
```

## Browser DevTools

### Network Tab

- Inspect API requests
- Check request/response headers
- View response data
- Check status codes

### Console Tab

- View JavaScript errors
- Log debug information
- Test API calls

### Application Tab

- View cookies
- Check local storage
- Inspect session storage

## Best Practices

1. **Use Logging** - Log important events and errors
2. **Don't Commit Debug Code** - Remove dd() and dump() before committing
3. **Use Telescope** - Monitor application in development
4. **Test Locally** - Reproduce issues locally
5. **Check Logs** - Review log files regularly
6. **Use Tinker** - Test code interactively
7. **Enable Debug Mode** - Only in development
8. **Handle Errors Gracefully** - Provide user-friendly error messages

## Production Debugging

### Enable Debug Mode (Temporary)

In `.env`:

```env
APP_DEBUG=true
```

**Warning:** Never leave debug mode enabled in production permanently.

### View Logs

```bash
tail -f storage/logs/laravel.log
```

### Monitor Errors

Use services like:

- Sentry
- Bugsnag
- Rollbar

## Tools Summary

1. **Laravel Debugbar** - Development debugging
2. **Laravel Telescope** - Application monitoring
3. **Logging** - Track application events
4. **Tinker** - Interactive testing
5. **dd/dump** - Quick variable inspection
6. **Query Logging** - Database query analysis
7. **Browser DevTools** - Frontend debugging
8. **Error Tracking** - Production error monitoring
