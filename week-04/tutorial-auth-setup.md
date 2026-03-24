# Laravel Breeze Authentication

## Prerequisites

Complete **all** of the following before starting this tutorial. Missing any item will cause errors.

### 1. Week 1–3 Complete

- **Week 1:** PHP, MySQL, Composer installed. Laravel project created (`events-management`).
- **Week 2:** Migrations run (`php artisan migrate`). Seeders run (`php artisan db:seed`). `users` table has a `role` column.
- **Week 3:** Eloquent relationships defined (User, Event, Category models).

### 2. Node.js and npm (Required)

Laravel Breeze uses **npm** to install and build frontend assets (Tailwind CSS, layout files). Without Node.js/npm, `php artisan breeze:install blade` will fail:

- **Windows:** `'npm' is not recognized as an internal or external command`
- **macOS:** `command not found: npm`

**Install Node.js (LTS):**

**Option A: Official installer (Windows & macOS)**

1. Download from [https://nodejs.org](https://nodejs.org) (LTS version).
2. Run the installer.
   - **Windows:** Leave "Add to PATH" checked. Or you could add Node.js to PATH manually:
    - Press Win + S → search for Environment Variables → Edit the system environment variables
    - Environment Variables → under User variables select Path → Edit
    - New → add: C:\Program Files\nodejs
    - OK on all dialogs
   - **macOS:** Follow the prompts; the installer adds Node to your PATH.
3. Close and reopen your terminal.

**Option B: Homebrew (macOS only)**

If you have [Homebrew](https://brew.sh) installed:

```bash
brew install node
```

**Verify:**

```bash
node --version   # Should show v18.x.x or v20.x.x
npm --version   # Should show 9.x.x or 10.x.x
```

#### If `npm` or `node` still fails after editing environment variables

Adding PATH in Windows does **not** update terminals or editors that are already open. Try these in order:

1. **Quit VS Code completely**, then reopen it and open a new terminal. Integrated terminals inherit the environment from when the editor started.

2. **Open a new terminal outside the editor:** Win + R → type `powershell` → Enter. Run `node --version` and `npm --version` there. If it works in that window but not inside Cursor, the editor was started before PATH was updated—fully quit and reopen Cursor.

3. **Refresh PATH in the current PowerShell session** (does not fix the editor until you restart it):

   ```powershell
   $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
   ```

4. **Confirm Node is installed:** In PowerShell run `Test-Path "C:\Program Files\nodejs\npm.cmd"`. If this is `True`, Node is installed; PATH is the problem. If `False`, reinstall Node from [nodejs.org](https://nodejs.org).

5. **Run npm by full path**:

   ```powershell
   & "C:\Program Files\nodejs\npm.cmd" --version
   ```

   You can prepend `C:\Program Files\nodejs` to PATH for the current session: `$env:Path += ";C:\Program Files\nodejs"`.

6. **User vs System PATH:** If you lack permission to edit **System** variables, add `C:\Program Files\nodejs` under **User** variables → Path (same steps as above). Both are valid.

7. **Restart the computer** if PATH still does not apply after closing all apps.

### 3. Project Setup

- Work from your Laravel project directory:
  - **Windows:** `cd C:\Projects\events-management`
  - **macOS:** `cd ~/Projects/events-management`
- Database configured in `.env` (DB_DATABASE, DB_USERNAME, DB_PASSWORD).
- MySQL server running.

**Verify:**

```bash
php -v
composer --version
npm --version
php artisan --version
```

### 4. Clean State (If Repeating the Tutorial)

If you have already run Breeze before and want to redo it:

- Remove Breeze: `composer remove laravel/breeze --dev`
- Delete `package.json` and `node_modules` if you want a fresh npm setup
- Revert any manual changes to auth views, routes, or controllers

---

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
- [ ] **Customize Breeze Views** - (Optional) View file locations for reference
- [ ] **Add Role-Based Navigation** - (Optional) Defer until event/admin routes exist
- [ ] **Protect Routes with Authentication** - (No action—Breeze already does this)
- [ ] **Display User on Dashboard** - Show logged-in user's name and role on dashboard
- [ ] **Test Authentication Flow** - Verify guest, authenticated, and role-based access

## Step 1: Install Laravel Breeze

A starter kit for login, registration, and password reset. --dev means it’s used mainly when building the app (when you run php artisan breeze:install), not in production.

Open terminal in your project directory (e.g. `C:\Projects\events-management` or `~/Projects/events-management`) and run:

```bash
composer require laravel/breeze --dev
```

Wait for the installation to complete.

## Step 2: Install Breeze with Blade Stack

Install the login, registration, and password reset pages and set them up to use plain PHP/Blade templates. You’ll get HTML views you can edit in resources/views, not React or Vue files.

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

Run migrations (if you completed Week 2, you may see "Nothing to migrate"—that is OK; the tables already exist):

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

**File:** `app/Http/Controllers/Auth/RegisteredUserController.php`

**Task:** Update the `store` method to validate `role` and pass it to `User::create()`.

**1.** At the top of the file, ensure these `use` statements exist (add any that are missing):

```php
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Auth\Events\Registered;
use Illuminate\Validation\Rules;
```

**2.** Replace the entire `store` method with:

```php
public function store(Request $request): RedirectResponse
{
    // Checks the form data (name, email, password, role). If something is invalid, the user is sent back to the form with errors. If valid, $validated contains the cleaned data.
    $validated = $request->validate([
        // Name is required, must be a string, max 255 characters.
        'name' => ['required', 'string', 'max:255'],
        // Email is required and must not be used by another user.
        'email' => ['required', 'string', 'email', 'max:255', 'unique:'.User::class],
        // Password is required and must match the confirmation field.
        'password' => ['required', 'confirmed', Rules\Password::defaults()],
        // Role is optional. If present, it must be one of these three values.
        'role' => ['sometimes', 'in:admin,organizer,attendee'],
    ]);

    $user = User::create([
        'name' => $validated['name'],
        'email' => $validated['email'],
        // Encrypts the password before saving (never store plain passwords).
        'password' => Hash::make($validated['password']),
        'role' => $validated['role'] ?? 'attendee',
    ]);

    // Fires Laravel’s “user registered” event (e.g. for sending welcome emails).
    event(new Registered($user));

    Auth::login($user);

    return redirect()->route('dashboard');
}
```

**3.** Open `app/Models/User.php` and ensure `'role'` is in the `$fillable` array (from Week 2).

### Add Role Field to Registration Form

**File:** `resources/views/auth/register.blade.php`

**Task:** Add a role select field.

**Where:** After the email field `<div>`, before the password field `<div>`.

```blade
<div class="mt-4">
    <x-input-label for="role" :value="__('Role')" />
    <select id="role" name="role" required
        class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-indigo-500 focus:ring-indigo-500">
        <option value="attendee" {{ old('role') === 'attendee' ? 'selected' : '' }}>{{ __('Attendee') }}</option>
        <option value="organizer" {{ old('role') === 'organizer' ? 'selected' : '' }}>{{ __('Organizer') }}</option>
        <option value="admin" {{ old('role') === 'admin' ? 'selected' : '' }}>{{ __('Admin') }}</option>
    </select>
    <x-input-error :messages="$errors->get('role')" class="mt-2" />
</div>
```

## Step 8: Test Registration

1. Visit `http://localhost:8000/register`
2. Fill in the registration form:
   - Name: Test User
   - Email: test@example.com
   - Role: Attendee (or Organizer)
   - Password: password
   - Confirm Password: password
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

## Step 10: Customize Breeze Views (Optional)

Breeze view files (for reference—no action required):

- Login: `resources/views/auth/login.blade.php`
- Registration: `resources/views/auth/register.blade.php`
- Dashboard: `resources/views/dashboard.blade.php`
- Profile: `resources/views/profile/edit.blade.php`

## Step 11: Role-Based Navigation (Optional—skip for now)

**You can skip this step.** Breeze authentication, registration with `role`, and the dashboard (Step 13) are enough for Week 4.

Role-based menu links (`Create Event`, `Admin Panel`) need named routes that do not exist until you build event and admin features in a later week. Adding them now means placeholder routes and edits in three places in `navigation.blade.php`.

**When you are ready** (after you have real `events.create` and `admin.dashboard` routes), follow **Optional: Role-based navigation (full instructions)** at the end of this document.

---

## Step 12: Protect Routes with Authentication

Breeze already protects the dashboard and profile with the `auth` middleware—no changes needed here.

**For reference:** When you add new routes later (e.g. event CRUD), put them inside the same auth middleware group in `routes/web.php`, or add `->middleware('auth')` to individual routes.

## Step 13: Display User Name and Role on the Dashboard

**File:** `resources/views/dashboard.blade.php`

**Task:** Add a welcome message that shows the logged-in user's name and role.

**Where:** At the top of the main content area—inside the first `<div>` or section that contains the dashboard content, before any other text or elements.

**Add this:**

```blade
<p>Welcome, {{ Auth::user()->name }}!</p>
<p>Your role: {{ Auth::user()->role }}</p>
```

The dashboard is only shown to logged-in users, so you can use `Auth::user()` directly without `@auth`.

## Step 14: Test Authentication Flow

1. **Test Guest Access:**

   - Logout
   - Try to access `/dashboard`
   - Should redirect to login page
2. **Test Authenticated Access:**

   - Login
   - Access `/dashboard`
   - Should show dashboard
3. **Roles (optional):** If you added Step 11 navigation links, log in as organizer vs attendee and confirm the menu matches role. If you skipped Step 11, confirm Step 13 shows the correct `role` on the dashboard after registering with different roles.

## What Breeze Provides

After installation, Breeze gives you:

- Login page (`/login`)
- Registration page (`/register`)
- Password reset functionality
- Email verification (optional)
- Password confirmation
- Profile management
- Dashboard page
- Authentication middleware
- Session management
- CSRF protection

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
│           └── LoginRequest.php
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

## Optional: Role-based navigation (full instructions)

Use this **after** you have named routes `events.create` and `admin.dashboard` (real controllers or placeholders).

### 1. Placeholder routes (if you do not have real pages yet)

**File:** `routes/web.php`

**Where:** Inside `Route::middleware(['auth', 'verified'])->group(function () { ... });` (same group as dashboard):

```php
Route::get('/events/create', function () {
    return view('dashboard');
})->name('events.create');

Route::get('/admin', function () {
    return view('dashboard');
})->name('admin.dashboard');
```

### 2. Navigation snippets

**File:** `resources/views/layouts/navigation.blade.php` (do **not** replace the whole file)

Use `Route::has()` so missing routes do not crash the page.

**Main nav (desktop):** In `<!-- Navigation Links -->`, after the Dashboard `x-nav-link`, add:

```blade
@if(Auth::user()->role === 'organizer' && Route::has('events.create'))
    <x-nav-link :href="route('events.create')" :active="request()->routeIs('events.create')">
        {{ __('Create Event') }}
    </x-nav-link>
@endif
@if(Auth::user()->role === 'admin' && Route::has('admin.dashboard'))
    <x-nav-link :href="route('admin.dashboard')" :active="request()->routeIs('admin.dashboard')">
        {{ __('Admin Panel') }}
    </x-nav-link>
@endif
```

**Dropdown (desktop):** Inside `<x-slot name="content">`, after `route('profile.edit')`, before the logout `<form>`:

```blade
@if(Auth::user()->role === 'organizer' && Route::has('events.create'))
    <x-dropdown-link :href="route('events.create')">
        {{ __('Create Event') }}
    </x-dropdown-link>
@endif
@if(Auth::user()->role === 'admin' && Route::has('admin.dashboard'))
    <x-dropdown-link :href="route('admin.dashboard')">
        {{ __('Admin Panel') }}
    </x-dropdown-link>
@endif
```

**Mobile menu:** In `<!-- Responsive Settings Options -->`, inside `<div class="mt-3 space-y-1">`, after `route('profile.edit')`, before logout `<form>`:

```blade
@if(Auth::user()->role === 'admin' && Route::has('admin.dashboard'))
    <x-responsive-nav-link :href="route('admin.dashboard')">
        {{ __('Admin Panel') }}
    </x-responsive-nav-link>
@endif
@if(Auth::user()->role === 'organizer' && Route::has('events.create'))
    <x-responsive-nav-link :href="route('events.create')">
        {{ __('Create Event') }}
    </x-responsive-nav-link>
@endif
```

### 3. If you see no new links

1. Routes must exist (`php artisan route:list` should list `events.create` and `admin.dashboard`).
2. User role must be `organizer` or `admin` (not `attendee`).
3. Add snippets in **all three** places (main nav is hidden on small screens).
4. Rare: `php artisan view:clear`

## Common Customizations

### Change Redirect After Login

**Laravel 10 and earlier:** Edit `app/Providers/RouteServiceProvider.php` and change `public const HOME = '/dashboard';`

**Laravel 11+:** RouteServiceProvider was removed. Edit `app/Providers/AppServiceProvider.php` and add this inside the `boot()` method:

```php
use Illuminate\Auth\Middleware\RedirectIfAuthenticated;

public function boot(): void
{
    RedirectIfAuthenticated::redirectUsing(fn () => redirect('/dashboard'));
}
```

### Customize Login/Register Forms

Edit `resources/views/auth/login.blade.php` and `resources/views/auth/register.blade.php` to match your design.

### Add Remember Me Functionality

Breeze includes this by default in the login form.
