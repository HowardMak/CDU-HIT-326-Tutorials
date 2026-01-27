# Laravel Breeze Authentication

## Task List Summary

- [ ] **Install Laravel Breeze** - Install Breeze package via Composer
- [ ] **Install Breeze with Blade Stack** - Set up authentication scaffolding
- [ ] **Install NPM Dependencies** - Install frontend packages
- [ ] **Build Frontend Assets** - Compile CSS and JavaScript
- [ ] **Run Migrations** - Execute database migrations for users table
- [ ] **Test Authentication Pages** - Verify login, register, and dashboard pages work
- [ ] **Customize Registration to Include Role** - Add role field to registration
- [ ] **Test Registration** - Create a test user account
- [ ] **Test Login** - Verify login functionality works
- [ ] **Customize Breeze Views** - Review and understand authentication views
- [ ] **Add Role-Based Navigation** - Show different menu items based on user role
- [ ] **Protect Routes with Authentication** - Add auth middleware to routes
- [ ] **Use Authentication in Controllers** - Access current user in controllers
- [ ] **Use Authentication in Blade Templates** - Display user info and conditional content
- [ ] **Test Authentication Flow** - Verify guest, authenticated, and role-based access

## Step 1: Install Laravel Breeze

Open terminal in your project directory and run:

```bash
composer require laravel/breeze --dev
```

Wait for the installation to complete.

## Step 2: Install Breeze with Blade Stack

```bash
php artisan breeze:install blade
```

This command will:

- Create authentication views
- Set up authentication routes
- Create authentication controllers
- Configure authentication middleware

## Step 3: Install NPM Dependencies

```bash
npm install
```

Wait for npm packages to install.

## Step 4: Build Frontend Assets

```bash
npm run build
```

This compiles CSS and JavaScript assets for the authentication pages.

## Step 5: Run Migrations

Breeze creates a migration for the users table. Run migrations:

```bash
php artisan migrate
```

## Step 6: Test Authentication Pages

Start the development server:

```bash
php artisan serve
```

Visit these URLs in your browser:

- `http://localhost:8000/register` - Registration page
- `http://localhost:8000/login` - Login page
- `http://localhost:8000/dashboard` - Dashboard (requires login)

## Step 7: Customize Registration to Include Role

### Update Registration Request

Open `app/Http/Requests/Auth/RegisteredUserRequest.php` and add role validation:

```php
public function rules(): array
{
    return [
        'name' => ['required', 'string', 'max:255'],
        'email' => ['required', 'string', 'email', 'max:255', 'unique:'.User::class],
        'password' => ['required', 'confirmed', Rules\Password::defaults()],
        'role' => ['sometimes', 'in:admin,organizer,attendee'],
    ];
}
```

### Update Registration Controller

Open `app/Http/Controllers/Auth/RegisteredUserController.php` and modify the `store` method:

```php
public function store(RegisteredUserRequest $request): RedirectResponse
{
    $user = User::create([
        'name' => $request->name,
        'email' => $request->email,
        'password' => Hash::make($request->password),
        'role' => $request->role ?? 'attendee', // Default to attendee
    ]);

    event(new Registered($user));

    Auth::login($user);

    return redirect(RouteServiceProvider::HOME);
}
```

## Step 8: Test Registration

1. Visit `http://localhost:8000/register`
2. Fill in the registration form:
   - Name: Test User
   - Email: test@example.com
   - Password: password
   - Confirm Password: password
   - Role: attendee (or organizer)
3. Click "Register"
4. You should be redirected to the dashboard

## Step 9: Test Login

1. Logout first (if logged in)
2. Visit `http://localhost:8000/login`
3. Enter credentials:
   - Email: test@example.com
   - Password: password
4. Click "Log In"
5. You should be redirected to the dashboard

## Step 10: Customize Breeze Views

Breeze views are located in `resources/views/auth/`. You can customize:

### Login Page

- `resources/views/auth/login.blade.php`

### Registration Page

- `resources/views/auth/register.blade.php`

### Dashboard

- `resources/views/dashboard.blade.php`

### Profile

- `resources/views/profile/edit.blade.php`

## Step 11: Add Role-Based Navigation

Update your main layout or navigation to show different options based on role.

In `resources/views/layouts/navigation.blade.php` or your main layout:

```blade
@auth
    <div class="flex items-center space-x-4">
        <span>{{ Auth::user()->name }}</span>
  
        @if(Auth::user()->isOrganizer())
            <a href="{{ route('events.create') }}">Create Event</a>
        @endif
  
        @if(Auth::user()->isAdmin())
            <a href="{{ route('admin.dashboard') }}">Admin Panel</a>
        @endif
  
        <form method="POST" action="{{ route('logout') }}">
            @csrf
            <button type="submit">Logout</button>
        </form>
    </div>
@else
    <a href="{{ route('login') }}">Login</a>
    <a href="{{ route('register') }}">Register</a>
@endauth
```

## Step 12: Protect Routes with Authentication

### Protect Individual Routes

In `routes/web.php`:

```php
Route::get('/dashboard', function () {
    return view('dashboard');
})->middleware('auth');
```

### Protect Route Groups

```php
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/events/create', [EventController::class, 'create']);
    Route::post('/events', [EventController::class, 'store']);
});
```

### Protect in Controller

In your controller:

```php
public function __construct()
{
    $this->middleware('auth');
}
```

## Step 13: Use Authentication in Controllers

### Get Current User

```php
use Illuminate\Support\Facades\Auth;

public function index()
{
    $user = Auth::user();
    $events = Event::where('organizer_id', $user->id)->get();
  
    return view('events.index', compact('events'));
}
```

### Check Authentication

```php
if (Auth::check()) {
    // User is authenticated
    $userId = Auth::id();
}
```

## Step 14: Use Authentication in Blade Templates

### Check if User is Authenticated

```blade
@auth
    <p>Welcome, {{ Auth::user()->name }}!</p>
    <p>Your role: {{ Auth::user()->role }}</p>
@else
    <a href="{{ route('login') }}">Please login</a>
@endauth
```

### Check if User is Guest

```blade
@guest
    <a href="{{ route('register') }}">Register</a>
@endguest
```

### Check User Role

```blade
@if(Auth::user()->role === 'organizer')
    <a href="{{ route('events.create') }}">Create Event</a>
@endif
```

## Step 15: Test Authentication Flow

1. **Test Guest Access:**

   - Logout
   - Try to access `/dashboard`
   - Should redirect to login page
2. **Test Authenticated Access:**

   - Login
   - Access `/dashboard`
   - Should show dashboard
3. **Test Role-Based Access:**

   - Login as organizer
   - Check if "Create Event" link appears
   - Login as attendee
   - Check if "Create Event" link is hidden

## What Breeze Provides

After installation, Breeze gives you:

- ✅ Login page (`/login`)
- ✅ Registration page (`/register`)
- ✅ Password reset functionality
- ✅ Email verification (optional)
- ✅ Password confirmation
- ✅ Profile management
- ✅ Dashboard page
- ✅ Authentication middleware
- ✅ Session management
- ✅ CSRF protection

## File Structure Created by Breeze

```
app/
├── Http/
│   └── Controllers/
│       └── Auth/
│           ├── AuthenticatedSessionController.php
│           ├── ConfirmablePasswordController.php
│           ├── EmailVerificationNotificationController.php
│           ├── EmailVerificationPromptController.php
│           ├── NewPasswordController.php
│           ├── PasswordResetLinkController.php
│           ├── RegisteredUserController.php
│           └── VerifyEmailController.php
├── Http/
│   └── Requests/
│       └── Auth/
│           ├── LoginRequest.php
│           └── RegisteredUserRequest.php
resources/
└── views/
    ├── auth/
    │   ├── login.blade.php
    │   ├── register.blade.php
    │   ├── forgot-password.blade.php
    │   └── reset-password.blade.php
    └── components/
        └── auth-session-status.blade.php
```

## Common Customizations

### Change Redirect After Login

Edit `app/Providers/RouteServiceProvider.php`:

```php
public const HOME = '/dashboard'; // Change this
```

### Customize Login/Register Forms

Edit the Blade templates in `resources/views/auth/` to match your design.

### Add Remember Me Functionality

Breeze includes this by default in the login form.
