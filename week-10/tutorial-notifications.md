# Email Notifications

## Task List Summary

- [ ] **Configure Mail Settings** - Set up SMTP settings in .env file (use Mailtrap for testing)
- [ ] **Create Mailable Class** - Generate mailable for event registration notification
- [ ] **Implement Mailable Class** - Build email class with event and user data
- [ ] **Create Email Template** - Design HTML email template in Blade
- [ ] **Send Email on Registration** - Integrate email sending in registration controller
- [ ] **Test Email Sending** - Verify emails are sent and received in Mailtrap
- [ ] **Use Queues for Better Performance (Optional)** - Configure queues for async email sending
- [ ] **Create Event Reminder Notification** - Build notification class for event reminders
- [ ] **Create Scheduled Command for Reminders** - Build command to send daily reminders
- [ ] **Schedule the Command** - Add command to Laravel scheduler
- [ ] **Test Scheduled Command** - Manually run and verify reminder command
- [ ] **Add Email to Event Cancellation** - Create cancellation notification and send to registrants
- [ ] **Test All Email Notifications** - Verify registration, cancellation, and reminder emails

## Step 1: Configure Mail Settings

Open `.env` file and configure mail settings (for testing, use Mailtrap):

```env
MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=your_mailtrap_username
MAIL_PASSWORD=your_mailtrap_password
MAIL_ENCRYPTION=tls
MAIL_FROM_ADDRESS=noreply@example.com
MAIL_FROM_NAME="${APP_NAME}"
```

**Note:** Sign up at mailtrap.io to get free SMTP credentials for testing.

## Step 2: Create Mailable Class

Generate a mailable class for event registration notification:

```bash
php artisan make:mail EventRegistrationNotification
```

## Step 3: Implement Mailable Class

Open `app/Mail/EventRegistrationNotification.php` and update:

```php
<?php

namespace App\Mail;

use App\Models\Event;
use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class EventRegistrationNotification extends Mailable
{
    use Queueable, SerializesModels;

    public $event;
    public $user;

    public function __construct(Event $event, User $user)
    {
        $this->event = $event;
        $this->user = $user;
    }

    public function build()
    {
        return $this->subject('Event Registration Confirmation')
                    ->view('emails.event-registration')
                    ->with([
                        'event' => $this->event,
                        'user' => $this->user,
                    ]);
    }
}
```

## Step 4: Create Email Template

Create `resources/views/emails/event-registration.blade.php`:

```blade
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <style>
        body {
            font-family: Arial, sans-serif;
            line-height: 1.6;
            color: #333;
        }
        .container {
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
        }
        .header {
            background-color: #3490dc;
            color: white;
            padding: 20px;
            text-align: center;
            border-radius: 5px 5px 0 0;
        }
        .content {
            background-color: #f9fafb;
            padding: 20px;
            border: 1px solid #e5e7eb;
        }
        .button {
            display: inline-block;
            background-color: #3490dc;
            color: white;
            padding: 12px 24px;
            text-decoration: none;
            border-radius: 5px;
            margin-top: 20px;
        }
        .footer {
            text-align: center;
            padding: 20px;
            color: #6b7280;
            font-size: 12px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Event Registration Confirmation</h1>
        </div>
        <div class="content">
            <p>Hello {{ $user->name }},</p>
      
            <p>You have successfully registered for the following event:</p>
      
            <h2>{{ $event->title }}</h2>
      
            <p><strong>Date:</strong> {{ $event->date->format('F d, Y') }}</p>
            <p><strong>Time:</strong> {{ $event->time }}</p>
            <p><strong>Location:</strong> {{ $event->location }}</p>
      
            @if($event->description)
                <p><strong>Description:</strong></p>
                <p>{{ $event->description }}</p>
            @endif
      
            <p>We look forward to seeing you there!</p>
      
            <a href="{{ route('events.show', $event) }}" class="button">
                View Event Details
            </a>
        </div>
        <div class="footer">
            <p>This is an automated email. Please do not reply.</p>
            <p>© {{ date('Y') }} {{ config('app.name') }}. All rights reserved.</p>
        </div>
    </div>
</body>
</html>
```

## Step 5: Send Email on Registration

Update `app/Http/Controllers/RegistrationController.php`:

```php
use App\Mail\EventRegistrationNotification;
use Illuminate\Support\Facades\Mail;

public function store(Request $request, Event $event)
{
    // ... existing registration logic ...

    // Create registration
    $event->registrants()->attach($user->id, [
        'status' => $status,
        'registered_at' => now(),
    ]);

    // Send email notification
    Mail::to($user->email)->send(new EventRegistrationNotification($event, $user));

    return redirect()->back()
        ->with('success', 'Successfully registered for the event. Check your email for confirmation.');
}
```

## Step 6: Test Email Sending

1. **Setup Mailtrap:**

   - Go to mailtrap.io and sign up
   - Get SMTP credentials from your inbox
   - Update `.env` with Mailtrap credentials
2. **Test Registration:**

   - Register for an event
   - Check Mailtrap inbox
   - Verify email was received
3. **View Email:**

   - Open email in Mailtrap
   - Check formatting and content
   - Verify links work

## Step 7: Use Queues for Better Performance (Optional)

Instead of sending emails synchronously, use queues:

### Setup Queue

In `.env`:

```env
QUEUE_CONNECTION=database
```

### Create Jobs Table

```bash
php artisan queue:table
php artisan migrate
```

### Update Mailable to Use Queue

Update `app/Mail/EventRegistrationNotification.php`:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class EventRegistrationNotification extends Mailable implements ShouldQueue
{
    // ... existing code ...
}
```

### Update Controller to Queue Email

```php
// Instead of send(), use queue()
Mail::to($user->email)->queue(new EventRegistrationNotification($event, $user));
```

### Process Queue

In a separate terminal:

```bash
php artisan queue:work
```

## Step 8: Create Event Reminder Notification

Generate notification class:

```bash
php artisan make:notification EventReminder
```

Open `app/Notifications/EventReminder.php`:

```php
<?php

namespace App\Notifications;

use App\Models\Event;
use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;

class EventReminder extends Notification
{
    use Queueable;

    public $event;

    public function __construct(Event $event)
    {
        $this->event = $event;
    }

    public function via($notifiable)
    {
        return ['mail'];
    }

    public function toMail($notifiable)
    {
        return (new MailMessage)
                    ->subject('Event Reminder: ' . $this->event->title)
                    ->line('This is a reminder that you have an event coming up.')
                    ->line('Event: ' . $this->event->title)
                    ->line('Date: ' . $this->event->date->format('F d, Y'))
                    ->line('Time: ' . $this->event->time)
                    ->line('Location: ' . $this->event->location)
                    ->action('View Event', route('events.show', $this->event))
                    ->line('Thank you for using our application!');
    }
}
```

## Step 9: Create Scheduled Command for Reminders

Create a command to send reminders:

```bash
php artisan make:command SendEventReminders
```

Open `app/Console/Commands/SendEventReminders.php`:

```php
<?php

namespace App\Console\Commands;

use App\Models\Event;
use App\Notifications\EventReminder;
use Illuminate\Console\Command;

class SendEventReminders extends Command
{
    protected $signature = 'events:send-reminders';
    protected $description = 'Send email reminders for upcoming events';

    public function handle()
    {
        // Get events happening tomorrow
        $events = Event::where('date', now()->addDay()->toDateString())
            ->where('status', 'published')
            ->get();

        foreach ($events as $event) {
            $registrants = $event->registrants()
                ->wherePivot('status', 'confirmed')
                ->get();

            foreach ($registrants as $user) {
                $user->notify(new EventReminder($event));
            }
        }

        $this->info('Reminders sent successfully.');
        return 0;
    }
}
```

## Step 10: Schedule the Command

Open `app/Console/Kernel.php` and add to the `schedule` method:

```php
use Illuminate\Console\Scheduling\Schedule;

protected function schedule(Schedule $schedule)
{
    $schedule->command('events:send-reminders')
             ->daily()
             ->at('09:00');
}
```

## Step 11: Test Scheduled Command

Test the command manually:

```bash
php artisan events:send-reminders
```

Verify emails are sent to Mailtrap.

## Step 12: Add Email to Event Cancellation

Create another mailable:

```bash
php artisan make:mail EventCancelledNotification
```

Update `app/Mail/EventCancelledNotification.php`:

```php
<?php

namespace App\Mail;

use App\Models\Event;
use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Queue\SerializesModels;

class EventCancelledNotification extends Mailable
{
    use Queueable, SerializesModels;

    public $event;
    public $user;

    public function __construct(Event $event, User $user)
    {
        $this->event = $event;
        $this->user = $user;
    }

    public function build()
    {
        return $this->subject('Event Cancelled: ' . $this->event->title)
                    ->view('emails.event-cancelled')
                    ->with([
                        'event' => $this->event,
                        'user' => $this->user,
                    ]);
    }
}
```

Update `EventController.php` cancel method:

```php
use App\Mail\EventCancelledNotification;

public function cancel(Event $event)
{
    $this->authorize('update', $event);

    $event->update(['status' => 'cancelled']);

    // Notify all registrants
    $registrants = $event->registrants()->wherePivot('status', 'confirmed')->get();
  
    foreach ($registrants as $user) {
        Mail::to($user->email)->send(new EventCancelledNotification($event, $user));
    }

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event cancelled and all registrants notified.');
}
```

## Step 13: Test All Email Notifications

1. **Test Registration Email:**

   - Register for an event
   - Check Mailtrap for confirmation email
2. **Test Cancellation Email:**

   - Cancel an event
   - Check Mailtrap for cancellation emails to all registrants
3. **Test Reminder Email:**

   - Create an event for tomorrow
   - Register a user
   - Run `php artisan events:send-reminders`
   - Check Mailtrap for reminder email

## Complete Email Flow

1. **User registers** → Registration created → Email sent immediately
2. **Event cancelled** → Status updated → Emails sent to all registrants
3. **Daily reminder** → Scheduled command runs → Reminders sent for tomorrow's events
