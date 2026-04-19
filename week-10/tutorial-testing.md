# Testing and Debugging

## Task List Summary

- [ ] **Create Feature Test for Events** - Generate feature test class
- [ ] **Write Basic Feature Tests** - Test guest access, authentication, authorization, and registration
- [ ] **Create Unit Test for Models** - Test model relationships, methods, and scopes
- [ ] **Run Tests** - Execute tests and verify they pass
- [ ] **Install Laravel Debugbar** - Install debugbar for development debugging
- [ ] **Use Logging for Debugging** - Add logging statements to track application flow
- [ ] **Use dd() and dump() for Quick Debugging** - Use dump functions for variable inspection
- [ ] **Enable Query Logging** - Log database queries to identify N+1 problems
- [ ] **Test File Uploads** - Write tests for file upload functionality
- [ ] **Test Email Sending** - Write tests to verify email notifications
- [ ] **Use Tinker for Interactive Testing** - Test code interactively using Tinker
- [ ] **Check Common Errors** - Verify routes, configuration, and clear caches
- [ ] **View Error Pages** - Customize error pages for better user experience
- [ ] **Monitor Application with Telescope (Optional)** - Install Telescope for advanced monitoring

## Step 1: Create Feature Test for Events

Create a feature test:

```bash
php artisan make:test EventTest
```

## Step 2: Write Basic Feature Tests

Open `tests/Feature/EventTest.php` and add:

```php
<?php

namespace Tests\Feature;

use App\Models\Event;
use App\Models\User;
use App\Models\Category;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class EventTest extends TestCase
{
    use RefreshDatabase;

    public function test_guest_can_view_events_list()
    {
        Event::factory()->count(3)->create(['status' => 'published']);

        $response = $this->get('/events');

        $response->assertStatus(200)
                 ->assertViewIs('events.index');
    }

    public function test_guest_can_view_published_event()
    {
        $event = Event::factory()->create(['status' => 'published']);

        $response = $this->get("/events/{$event->id}");

        $response->assertStatus(200)
                 ->assertViewIs('events.show')
                 ->assertViewHas('event');
    }

    public function test_guest_cannot_view_draft_event()
    {
        $event = Event::factory()->create(['status' => 'draft']);

        $response = $this->get("/events/{$event->id}");

        $response->assertStatus(403);
    }

    public function test_organizer_can_create_event()
    {
        $user = User::factory()->create(['role' => 'organizer']);
        $category = Category::factory()->create();

        $response = $this->actingAs($user)
            ->post('/events', [
                'title' => 'Test Event',
                'description' => 'Test Description',
                'date' => now()->addDay()->format('Y-m-d'),
                'time' => '14:00',
                'location' => 'Test Location',
                'capacity' => 50,
                'category_id' => $category->id,
            ]);

        $response->assertRedirect()
                 ->assertSessionHas('success');

        $this->assertDatabaseHas('events', [
            'title' => 'Test Event',
            'organizer_id' => $user->id,
            'status' => 'draft',
        ]);
    }

    public function test_attendee_cannot_create_event()
    {
        $user = User::factory()->create(['role' => 'attendee']);

        $response = $this->actingAs($user)
            ->post('/events', [
                'title' => 'Test Event',
                'date' => now()->addDay()->format('Y-m-d'),
                'time' => '14:00',
                'location' => 'Test Location',
            ]);

        $response->assertStatus(403);
    }

    public function test_user_can_register_for_event()
    {
        $user = User::factory()->create();
        $event = Event::factory()->create([
            'status' => 'published',
            'capacity' => 10,
        ]);

        $response = $this->actingAs($user)
            ->post("/events/{$event->id}/register");

        $response->assertRedirect()
                 ->assertSessionHas('success');

        $this->assertDatabaseHas('registrations', [
            'user_id' => $user->id,
            'event_id' => $event->id,
            'status' => 'confirmed',
        ]);
    }
}
```

## Step 3: Create Unit Test for Models

Create a unit test:

```bash
php artisan make:test EventModelTest --unit
```

Open `tests/Unit/EventModelTest.php`:

```php
<?php

namespace Tests\Unit;

use App\Models\Event;
use App\Models\User;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class EventModelTest extends TestCase
{
    use RefreshDatabase;

    public function test_event_belongs_to_organizer()
    {
        $user = User::factory()->create();
        $event = Event::factory()->create(['organizer_id' => $user->id]);

        $this->assertEquals($user->id, $event->organizer->id);
    }

    public function test_event_is_full_when_capacity_reached()
    {
        $event = Event::factory()->create(['capacity' => 2]);
        $user1 = User::factory()->create();
        $user2 = User::factory()->create();
  
        $event->registrants()->attach($user1->id, ['status' => 'confirmed']);
        $event->registrants()->attach($user2->id, ['status' => 'confirmed']);

        $this->assertTrue($event->isFull());
    }

    public function test_event_scope_published()
    {
        Event::factory()->create(['status' => 'published']);
        Event::factory()->create(['status' => 'draft']);

        $publishedEvents = Event::published()->get();

        $this->assertCount(1, $publishedEvents);
        $this->assertEquals('published', $publishedEvents->first()->status);
    }
}
```

## Step 4: Run Tests

Run all tests:

```bash
php artisan test
```

Run specific test:

```bash
php artisan test tests/Feature/EventTest.php
```

Run with verbose output:

```bash
php artisan test --verbose
```

## Steps 5–8: Debugging Tools

> **These debugging tools are covered in full detail in `week-10/tutorial-debugging-tools.md`.** Refer to that tutorial for Debugbar, Logging, `dd()`/`dump()`, Query Logging, and Telescope.

When debugging a failing test, the most useful tools are:

- **`dd()` inside the controller** — dump the value that the test is triggering and stop execution
- **Query Logging** — check what SQL the test causes (`DB::enableQueryLog()` then `dd(DB::getQueryLog())`)
- **`tail -f storage/logs/laravel.log`** — watch the log file while running tests

> **Important:** Remove all `dd()` and `dump()` calls before committing — they will make your test suite hang.

## Step 9: Test File Uploads

Add test for file uploads in `EventTest.php`:

```php
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

public function test_organizer_can_upload_event_image()
{
    Storage::fake('public');
  
    $user = User::factory()->create(['role' => 'organizer']);
    $file = UploadedFile::fake()->image('event.jpg', 800, 600);

    $response = $this->actingAs($user)
        ->post('/events', [
            'title' => 'Test Event',
            'date' => now()->addDay()->format('Y-m-d'),
            'time' => '14:00',
            'location' => 'Test Location',
            'image' => $file,
        ]);

    Storage::disk('public')->assertExists('events/' . $file->hashName());
}
```

## Step 10: Test Email Sending

Add test for email notifications:

```php
use Illuminate\Support\Facades\Mail;
use App\Mail\EventRegistrationNotification;

public function test_email_sent_on_registration()
{
    Mail::fake();

    $user = User::factory()->create();
    $event = Event::factory()->create(['status' => 'published']);

    $this->actingAs($user)
        ->post("/events/{$event->id}/register");

    Mail::assertSent(EventRegistrationNotification::class, function ($mail) use ($user) {
        return $mail->hasTo($user->email);
    });
}
```

## Step 11: Use Tinker for Interactive Testing

Open Tinker:

```bash
php artisan tinker
```

Test in Tinker:

```php
// Create a test event
$event = Event::create([
    'title' => 'Test Event',
    'date' => now()->addDay(),
    'time' => '14:00',
    'location' => 'Test Location',
    'organizer_id' => 1,
    'status' => 'published',
]);

// Test relationships
$event->organizer;
$event->category;
$event->registrants;

// Test methods
$event->isFull();
$event->isRegisteredBy(1);
```

## Step 12: Check Common Errors

### Check Route List

```bash
php artisan route:list
```

### Check Configuration

```bash
php artisan config:show database
```

### Clear All Caches

```bash
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

## Step 13: View Error Pages

Laravel provides helpful error pages. Customize them in `resources/views/errors/`:

- `404.blade.php` - Not found
- `500.blade.php` - Server error
- `403.blade.php` - Forbidden

Example `404.blade.php`:

```blade
@extends('layouts.app')

@section('content')
<div class="text-center py-12">
    <h1 class="text-4xl font-bold mb-4">404</h1>
    <p class="text-xl text-gray-600 mb-4">Page not found</p>
    <a href="{{ route('events.index') }}" class="btn-primary">Go Home</a>
</div>
@endsection
```

## Step 14: Monitor Application with Telescope (Optional)

Install Telescope for advanced debugging:

```bash
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
```

Access at `/telescope` to view:

- Requests
- Queries
- Commands
- Jobs
- Mail
- Notifications

## Complete Testing Workflow

1. **Write test** → Define what you want to test
2. **Run test** → Execute and see if it passes
3. **Fix issues** → Debug using logs, dd(), Debugbar
4. **Re-run test** → Verify fix works
5. **Refactor** → Improve code while keeping tests green

## Common Debugging Scenarios

### Issue: Test Fails - Database Error

**Solution:**

- Ensure `RefreshDatabase` trait is used
- Check database connection in `phpunit.xml`
- Verify migrations are run

### Issue: Route Not Found in Test

**Solution:**

- Check route is defined
- Verify middleware isn't blocking
- Use `route:list` to verify route exists

### Issue: Authentication Fails in Test

**Solution:**

- Use `actingAs($user)` helper
- Verify user factory creates valid users
- Check authentication middleware
