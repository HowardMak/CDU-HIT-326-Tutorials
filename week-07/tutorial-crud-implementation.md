# Event CRUD Operations

## Task List Summary

- [ ] **Create Storage Link for File Uploads** - Set up symbolic link for storing uploaded images
- [ ] **Update Store Method for File Upload** - Add image upload handling to event creation
- [ ] **Update Update Method to Handle Image Replacement** - Implement image replacement with old image deletion
- [ ] **Update Destroy Method for Soft Delete** - Add soft delete functionality and image cleanup
- [ ] **Add Image Upload Field to Create Form** - Add file input field to event creation form
- [ ] **Add Image Display to Show View** - Display event images on event detail page
- [ ] **Add Image Preview to Edit Form** - Show current image in edit form with option to replace
- [ ] **Add Status Management to Index** - Filter events by status based on user role
- [ ] **Add Status Badge to Event Cards** - Display event status visually in event list
- [ ] **Add Publish/Cancel Actions** - Create methods and routes for publishing and cancelling events
- [ ] **Add Action Buttons to Show View** - Add publish, cancel, and delete buttons to event detail page
- [ ] **Test File Upload** - Verify image upload, display, and replacement functionality
- [ ] **Test Soft Delete** - Verify soft delete works and events can be restored
- [ ] **Add Restore Functionality (Optional)** - Implement method to restore soft-deleted events

## Step 1: Create Storage Link for File Uploads

First, create a symbolic link for storing uploaded files:

```bash
php artisan storage:link
```

This creates a link from `public/storage` to `storage/app/public`.

## Step 2: Update Store Method for File Upload

Open `app/Http/Controllers/EventController.php` and update the `store` method:

```php
use Illuminate\Support\Facades\Storage;

public function store(Request $request)
{
    $this->authorize('create', Event::class);

    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'description' => 'nullable|string',
        'date' => 'required|date|after:today',
        'time' => 'required',
        'location' => 'required|string|max:255',
        'capacity' => 'nullable|integer|min:0',
        'category_id' => 'nullable|exists:categories,id',
        'image' => 'nullable|image|mimes:jpeg,png,jpg,gif,webp|max:2048',
    ]);

    $validated['organizer_id'] = auth()->id();
    $validated['status'] = 'draft';

    // Handle file upload
    if ($request->hasFile('image')) {
        $validated['image'] = $request->file('image')->store('events', 'public');
    }

    $event = Event::create($validated);

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event created successfully.');
}
```

## Step 3: Update Update Method to Handle Image Replacement

Update the `update` method in the same controller:

```php
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
        'image' => 'nullable|image|mimes:jpeg,png,jpg,gif,webp|max:2048',
    ]);

    // Handle image update
    if ($request->hasFile('image')) {
        // Delete old image if exists
        if ($event->image) {
            Storage::disk('public')->delete($event->image);
        }
        // Store new image
        $validated['image'] = $request->file('image')->store('events', 'public');
    }

    $event->update($validated);

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event updated successfully.');
}
```

## Step 4: Update Destroy Method for Soft Delete

Update the `destroy` method to handle soft deletes and delete associated images:

```php
public function destroy(Event $event)
{
    $this->authorize('delete', $event);

    // Delete associated image
    if ($event->image) {
        Storage::disk('public')->delete($event->image);
    }

    // Soft delete the event
    $event->delete();

    return redirect()
        ->route('events.index')
        ->with('success', 'Event deleted successfully.');
}
```

## Step 5: Add Image Upload Field to Create Form

Update `resources/views/events/create.blade.php` to include image upload:

```blade
<div class="mb-4">
    <label for="image" class="block font-medium">Event Image</label>
    <input type="file" 
           id="image" 
           name="image" 
           accept="image/*"
           class="w-full rounded-md">
    <p class="text-sm text-gray-500 mt-1">Max size: 2MB. Formats: JPEG, PNG, GIF, WebP</p>
    @error('image')
        <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
    @enderror
</div>
```

## Step 6: Add Image Display to Show View

Update `resources/views/events/show.blade.php` to display the image:

```blade
@if($event->image)
    <div class="mb-6">
        <img src="{{ asset('storage/' . $event->image) }}" 
             alt="{{ $event->title }}" 
             class="w-full h-96 object-cover rounded-lg shadow-md">
    </div>
@else
    <div class="mb-6 bg-gray-200 h-96 rounded-lg flex items-center justify-center">
        <p class="text-gray-500">No image available</p>
    </div>
@endif
```

## Step 7: Add Image Preview to Edit Form

Update `resources/views/events/edit.blade.php` to show current image:

```blade
<div class="mb-4">
    <label for="image" class="block font-medium">Event Image</label>
  
    @if($event->image)
        <div class="mb-2">
            <p class="text-sm text-gray-600 mb-2">Current image:</p>
            <img src="{{ asset('storage/' . $event->image) }}" 
                 alt="Current image" 
                 class="h-32 w-auto rounded">
        </div>
    @endif
  
    <input type="file" 
           id="image" 
           name="image" 
           accept="image/*"
           class="w-full rounded-md">
    <p class="text-sm text-gray-500 mt-1">Leave empty to keep current image</p>
    @error('image')
        <p class="text-red-500 text-sm mt-1">{{ $message }}</p>
    @enderror
</div>
```

## Step 8: Add Status Management to Index

Update the `index` method to filter by status:

```php
public function index(Request $request)
{
    $query = Event::query();

    // Show published events to guests, all to organizers
    if (!auth()->check() || !auth()->user()->isOrganizer()) {
        $query->published();
    } else {
        // Organizers can see their own draft events
        if ($request->filled('status')) {
            $query->where('status', $request->status);
        }
    }

    // Filter by category
    if ($request->filled('category')) {
        $query->byCategory($request->category);
    }

    // Filter by date
    if ($request->filled('date_from')) {
        $query->where('date', '>=', $request->date_from);
    }

    $events = $query->with('organizer', 'category')
        ->latest('date')
        ->paginate(15);

    $categories = Category::all();

    return view('events.index', compact('events', 'categories'));
}
```

## Step 9: Add Status Badge to Event Cards

Update `resources/views/events/index.blade.php` to show status:

```blade
@foreach($events as $event)
    <div class="bg-white rounded-lg shadow-md p-6">
        <div class="flex justify-between items-start mb-2">
            <h3 class="text-xl font-semibold">
                <a href="{{ route('events.show', $event) }}">{{ $event->title }}</a>
            </h3>
            <span class="px-2 py-1 text-xs rounded 
                {{ $event->status === 'published' ? 'bg-green-100 text-green-800' : '' }}
                {{ $event->status === 'draft' ? 'bg-gray-100 text-gray-800' : '' }}
                {{ $event->status === 'cancelled' ? 'bg-red-100 text-red-800' : '' }}">
                {{ ucfirst($event->status) }}
            </span>
        </div>
        <!-- Rest of event card -->
    </div>
@endforeach
```

## Step 10: Add Publish/Cancel Actions

Add methods to publish or cancel events. In `EventController.php`:

```php
public function publish(Event $event)
{
    $this->authorize('update', $event);

    $event->update(['status' => 'published']);

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event published successfully.');
}

public function cancel(Event $event)
{
    $this->authorize('update', $event);

    $event->update(['status' => 'cancelled']);

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event cancelled successfully.');
}
```

Add routes in `routes/web.php`:

```php
Route::middleware(['auth'])->group(function () {
    Route::resource('events', EventController::class);
    Route::post('/events/{event}/publish', [EventController::class, 'publish'])->name('events.publish');
    Route::post('/events/{event}/cancel', [EventController::class, 'cancel'])->name('events.cancel');
});
```

## Step 11: Add Action Buttons to Show View

Update `resources/views/events/show.blade.php` to add publish/cancel buttons:

```blade
@can('update', $event)
    <div class="flex space-x-4 mb-4">
        <a href="{{ route('events.edit', $event) }}" class="btn-secondary">Edit</a>
  
        @if($event->status === 'draft')
            <form method="POST" action="{{ route('events.publish', $event) }}">
                @csrf
                <button type="submit" class="btn-primary">Publish Event</button>
            </form>
        @endif
  
        @if($event->status === 'published')
            <form method="POST" action="{{ route('events.cancel', $event) }}">
                @csrf
                <button type="submit" class="btn-danger">Cancel Event</button>
            </form>
        @endif
  
        <form method="POST" action="{{ route('events.destroy', $event) }}" 
              onsubmit="return confirm('Are you sure you want to delete this event?');">
            @csrf
            @method('DELETE')
            <button type="submit" class="btn-danger">Delete</button>
        </form>
    </div>
@endcan
```

## Step 12: Test File Upload

1. Start the server:

   ```bash
   php artisan serve
   ```
2. Create a new event with an image:

   - Go to `/events/create`
   - Fill in the form
   - Select an image file
   - Submit the form
3. Verify the image:

   - Check `storage/app/public/events/` directory
   - View the event detail page to see the image
4. Update the event with a new image:

   - Edit the event
   - Upload a new image
   - Verify old image is deleted and new one appears

## Step 13: Test Soft Delete

1. Delete an event:

   - Go to event detail page
   - Click delete
   - Confirm deletion
2. Verify soft delete:

   ```bash
   php artisan tinker
   ```

   ```php
   // Check deleted events
   Event::withTrashed()->get();

   // Restore an event
   $event = Event::withTrashed()->find(1);
   $event->restore();
   ```

## Step 14: Add Restore Functionality (Optional)

Add a restore method to the controller:

```php
public function restore($id)
{
    $event = Event::withTrashed()->findOrFail($id);
  
    $this->authorize('restore', $event);

    $event->restore();

    return redirect()
        ->route('events.show', $event)
        ->with('success', 'Event restored successfully.');
}
```

Add route:

```php
Route::post('/events/{id}/restore', [EventController::class, 'restore'])->name('events.restore');
```

## Complete CRUD Flow

1. **Create:** Form → Validation → File Upload → Save → Redirect
2. **Read:** List (index) → Detail (show) with image display
3. **Update:** Edit Form → Validation → Image Replacement → Update → Redirect
4. **Delete:** Soft Delete → Image Cleanup → Redirect

## Common Issues and Solutions

### Issue: Image Not Displaying

**Solution:**

```bash
php artisan storage:link
```

Ensure the symbolic link exists.

### Issue: File Upload Fails

**Solution:**

- Check `storage/app/public` permissions: `chmod -R 775 storage`
- Verify file size limit in `php.ini`: `upload_max_filesize = 2M`
- Check form has `enctype="multipart/form-data"`

### Issue: Old Image Not Deleted

**Solution:**
Always check if image exists before deleting:

```php
if ($event->image && Storage::disk('public')->exists($event->image)) {
    Storage::disk('public')->delete($event->image);
}
```
