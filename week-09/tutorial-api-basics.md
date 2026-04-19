# API Basics

## Task List Summary

- [ ] **Install Laravel Sanctum** - Install Sanctum package via Composer
- [ ] **Configure Sanctum** - Update config files for API authentication
- [ ] **Create API Controller** - Generate API resource controller for events
- [ ] **Implement API Controller** - Add CRUD methods returning JSON responses
- [ ] **Define API Routes** - Set up API routes with Sanctum authentication
- [ ] **Create API Resource (Optional)** - Format API responses with resources
- [ ] **Generate API Token** - Create authentication controller for token generation
- [ ] **Add Authentication Routes** - Set up login/logout API endpoints
- [ ] **Test API Endpoints** - Test all endpoints using curl or Postman
- [ ] **Add Rate Limiting** - Protect API routes with throttling
- [ ] **Configure CORS** - Set up cross-origin resource sharing
- [ ] **Test API with Postman** - Create Postman collection and test all endpoints

## Step 1: Install Laravel Sanctum

Install Sanctum for API authentication:

```bash
composer require laravel/sanctum
```

Publish Sanctum configuration:

```bash
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
```

Run migrations:

```bash
php artisan migrate
```

## Step 2: Configure Sanctum

Open `config/sanctum.php` and ensure:

```php
'guard' => ['web'],
```

Update `config/auth.php` to include Sanctum guard:

```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],
    'sanctum' => [
        'driver' => 'sanctum',
        'provider' => 'users',
    ],
],
```

## Step 3: Create API Controller

Create an API resource controller:

```bash
php artisan make:controller Api/EventController --api
```

## Step 4: Implement API Controller

Open `app/Http/Controllers/Api/EventController.php`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Event;
use Illuminate\Http\Request;

class EventController extends Controller
{
    public function index()
    {
        $events = Event::published()
            ->upcoming()
            ->with('organizer', 'category')
            ->paginate(15);

        return response()->json($events);
    }

    public function show(Event $event)
    {
        $event->load('organizer', 'category', 'registrants');
  
        return response()->json($event);
    }

    public function store(Request $request)
    {
        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'description' => 'nullable|string',
            'date' => 'required|date|after:today',
            'time' => 'required',
            'location' => 'required|string|max:255',
            'capacity' => 'nullable|integer|min:0',
            'category_id' => 'nullable|exists:categories,id',
        ]);

        $validated['organizer_id'] = auth()->id();
        $validated['status'] = 'draft';

        $event = Event::create($validated);

        return response()->json($event, 201);
    }

    public function update(Request $request, Event $event)
    {
        $this->authorize('update', $event);

        $validated = $request->validate([
            'title' => 'required|string|max:255',
            'description' => 'nullable|string',
            'date' => 'required|date',
            'time' => 'required',
            'location' => 'required|string|max:255',
            'capacity' => 'nullable|integer|min:0',
            'category_id' => 'nullable|exists:categories,id',
            'status' => 'required|in:draft,published,cancelled',
        ]);

        $event->update($validated);

        return response()->json($event);
    }

    public function destroy(Event $event)
    {
        $this->authorize('delete', $event);

        $event->delete();

        return response()->json(null, 204);
    }
}
```

## Step 5: Define API Routes

Open `routes/api.php` and add:

```php
use App\Http\Controllers\Api\EventController;

Route::middleware('auth:sanctum')->group(function () {
    Route::apiResource('events', EventController::class);
});
```

## Step 6: Create API Resource (Optional)

Create a resource to format API responses:

```bash
php artisan make:resource EventResource
```

Open `app/Http/Resources/EventResource.php`:

```php
<?php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class EventResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            'date' => $this->date->format('Y-m-d'),
            'time' => $this->time->format('H:i'),
            'location' => $this->location,
            'capacity' => $this->capacity,
            'status' => $this->status,
            'organizer' => new UserResource($this->whenLoaded('organizer')),
            'category' => new CategoryResource($this->whenLoaded('category')),
            'registrations_count' => $this->when(
                $this->relationLoaded('registrants'),
                $this->registrants->count()
            ),
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }
}
```

Use the resource in controller:

```php
use App\Http\Resources\EventResource;

public function index()
{
    $events = Event::with('organizer', 'category')->get();
    return EventResource::collection($events);
}

public function show(Event $event)
{
    $event->load('organizer', 'category');
    return new EventResource($event);
}
```

## Step 7: Generate API Token

Create a method to generate tokens. Add to `UserController` or create `Api/AuthController`:

```bash
php artisan make:controller Api/AuthController
```

Open `app/Http/Controllers/Api/AuthController.php`:

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        $token = $user->createToken('api-token')->plainTextToken;

        return response()->json([
            'token' => $token,
            'user' => $user,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out successfully']);
    }
}
```

## Step 8: Add Authentication Routes

In `routes/api.php`:

```php
use App\Http\Controllers\Api\AuthController;

Route::post('/login', [AuthController::class, 'login']);
Route::post('/logout', [AuthController::class, 'logout'])->middleware('auth:sanctum');
```

## Step 9: Test API Endpoints

### Test Login

Using curl or Postman:

```bash
curl -X POST http://localhost:8000/api/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password"}'
```

Response:

```json
{
  "token": "1|abc123...",
  "user": {
    "id": 1,
    "name": "Test User",
    "email": "test@example.com"
  }
}
```

### Test Get Events

```bash
curl -X GET http://localhost:8000/api/events \
  -H "Authorization: Bearer 1|abc123..." \
  -H "Accept: application/json"
```

### Test Create Event

```bash
curl -X POST http://localhost:8000/api/events \
  -H "Authorization: Bearer 1|abc123..." \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{
    "title": "API Test Event",
    "date": "2024-12-25",
    "time": "14:00",
    "location": "Test Location"
  }'
```

## Step 10: Add Rate Limiting

Protect API routes with rate limiting. In `routes/api.php`:

```php
Route::middleware(['auth:sanctum', 'throttle:60,1'])->group(function () {
    Route::apiResource('events', EventController::class);
});
```

This limits to 60 requests per minute.

## Step 11: Configure CORS

If accessing API from different domain, configure CORS. In `config/cors.php`:

```php
'paths' => ['api/*'],
'allowed_origins' => ['http://localhost:3000', 'https://yourdomain.com'],
'allowed_methods' => ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS'],
'allowed_headers' => ['Content-Type', 'Authorization', 'Accept'],
```

## Step 12: Test API with Postman

1. **Create Collection:**

   - Open Postman
   - Create new collection "Events API"
2. **Add Login Request:**

   - Method: POST
   - URL: `http://localhost:8000/api/login`
   - Body (JSON):
     ```json
     {
       "email": "test@example.com",
       "password": "password"
     }
     ```
   - Save token from response
3. **Add Get Events Request:**

   - Method: GET
   - URL: `http://localhost:8000/api/events`
   - Headers:
     - `Authorization: Bearer {token}`
     - `Accept: application/json`
4. **Test All Endpoints:**

   - GET `/api/events` - List events
   - GET `/api/events/{id}` - Show event
   - POST `/api/events` - Create event
   - PUT `/api/events/{id}` - Update event
   - DELETE `/api/events/{id}` - Delete event

## Complete API Flow

1. **User logs in** → Receives API token
2. **User makes request** → Includes token in Authorization header
3. **API validates token** → Authenticates user
4. **API processes request** → Returns JSON response
5. **User receives data** → Uses in frontend application
