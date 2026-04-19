# Database Design and Migrations

## Task List Summary

- [ ] **Create the Database** - Create MySQL database for events management
- [ ] **Update .env Configuration** - Configure database connection settings
- [ ] **Add Role Column to Users Table** - Create migration to add role field
- [ ] **Create Categories Table Migration** - Build categories table with name, slug, description
- [ ] **Create Events Table Migration** - Build events table with all fields and relationships
- [ ] **Create Registrations Table Migration** - Build pivot table for user-event registrations
- [ ] **Run All Migrations** - Execute migrations to create database tables
- [ ] **Verify Database Structure** - Check that all tables were created correctly
- [ ] **Create Eloquent Models** - Create Category and Event models (User model exists by default)
- [ ] **Create Category Seeder** - Generate seeder for sample categories
- [ ] **Create User Seeder** - Generate seeder for sample users with different roles
- [ ] **Create Event Seeder** - Generate seeder for sample events
- [ ] **Update DatabaseSeeder** - Configure main seeder to call all seeders
- [ ] **Run Seeders** - Populate database with sample data
- [ ] **Verify Data** - Test that all data was seeded correctly using Tinker

## Step 1: Create the Database

First, create the MySQL database:

```bash
mysql -u root -p
```

Then in MySQL:

```sql
CREATE DATABASE events_management;
EXIT;
```

## Step 2: Update .env Configuration

Open `.env` file and update database settings:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=events_management
DB_USERNAME=root
DB_PASSWORD=your_password
```

## Step 3: Add Role Column to Users Table

The users table already exists from Laravel's default migration. We need to add the `role` column:

```bash
php artisan make:migration add_role_to_users_table --table=users

# php artisan – run an Artisan command.
# make:migration – Artisan subcommand to create a migration.
# add_role_to_users_table – name for the migration (used in the filename).
# --table=users – (optional) the existing table this migration will change.
```

Open the created migration file in `database/migrations/` and add:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->enum('role', ['admin', 'organizer', 'attendee'])
                  ->default('attendee')
                  ->after('password'); //  tells the schema builder where to place the new column in the table
        });
    }

    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('role');
        });
    }
};
```

## Step 4: Create Categories Table Migration

```bash
php artisan make:migration create_categories_table
```

Open the migration file and add:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('categories', function (Blueprint $table) {
            $table->id();
            $table->string('name')->unique(); // enforces a unique constraint on that column: every value in the column must be different
            $table->string('slug')->unique();
            $table->text('description')->nullable(); // means the column may be NULL — it does not have to have a value.
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('categories');
    }
};
```

## Step 5: Create Events Table Migration

```bash
php artisan make:migration create_events_table
```

Open the migration file and add:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('events', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->string('slug')->unique()->nullable();
            $table->text('description')->nullable();
            $table->date('date');
            $table->time('time');
            $table->string('location');
            $table->unsignedInteger('capacity')->default(0); // creates an integer column that cannot be negative; it only holds 0 and positive whole numbers.
            $table->enum('status', ['draft', 'published', 'cancelled'])->default('draft');
            $table->string('image')->nullable();
            $table->foreignId('category_id')->nullable()->constrained('categories')->onDelete('set null');
            $table->foreignId('organizer_id')->constrained('users')->onDelete('cascade');
            $table->timestamps();
            $table->softDeletes(); // adds a deleted_at column and makes “deletions” reversible. Rows are hidden from normal queries instead of being removed from the database.
    
            // Indexes for better query performance
            $table->index('date');
            $table->index('status');
            $table->index('organizer_id');
            $table->index('category_id');
    
            // Creates a full-text index on those columns so the database can do natural-language search over them, instead of simple equality or prefix matching.
            $table->fullText(['title', 'description', 'location']);
        });
    }

    public function down()
    {
        Schema::dropIfExists('events');
    }
};
```

## Step 6: Create Registrations Table Migration

```bash
php artisan make:migration create_registrations_table
```

Open the migration file and add:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('registrations', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->foreignId('event_id')->constrained()->onDelete('cascade');
            $table->enum('status', ['confirmed', 'cancelled', 'waitlisted'])->default('confirmed');
            $table->timestamp('registered_at')->nullable();
            $table->timestamps();
    
            // Prevent duplicate registrations
            $table->unique(['user_id', 'event_id']);
    
            // Indexes for better query performance
            $table->index('user_id');
            $table->index('event_id');
            $table->index('status');
        });
    }

    public function down()
    {
        Schema::dropIfExists('registrations');
    }
};
```

## Step 7: Run All Migrations

Run the migrations to create all tables:

```bash
php artisan migrate
```

You should see output like:

```
Migrating: 2014_10_12_000000_create_users_table
Migrated:  2014_10_12_000000_create_users_table
Migrating: 2024_01_01_000001_add_role_to_users_table
Migrated:  2024_01_01_000001_add_role_to_users_table
Migrating: 2024_01_01_000002_create_categories_table
Migrated:  2024_01_01_000002_create_categories_table
Migrating: 2024_01_01_000003_create_events_table
Migrated:  2024_01_01_000003_create_events_table
Migrating: 2024_01_01_000004_create_registrations_table
Migrated:  2024_01_01_000004_create_registrations_table
```

## Step 8: Verify Database Structure

Check that all tables were created:

```bash
php artisan migrate:status
```

Or check directly in MySQL:

```bash
mysql -u root -p
```

```sql
USE events_management;
SHOW TABLES;
DESCRIBE users;
DESCRIBE categories;
DESCRIBE events;
DESCRIBE registrations;
EXIT;
```

## Step 9: Create Eloquent Models

Seeders use Eloquent models (`Category`, `Event`) to insert data. The `User` model already exists in Laravel. Create the missing models before creating seeders:

```bash
php artisan make:model Category
php artisan make:model Event
```

Laravel places models in `app/Models/` and assumes table names (`categories`, `events`) by convention, so no extra configuration is needed.

## Step 10: Create Seeders for Sample Data

### Create Category Seeder

```bash
php artisan make:seeder CategorySeeder

# Creates a new database seeder that Laravel uses to insert predefined data into your tables.
# php artisan	Runs Laravel’s Artisan CLI
# make:seeder   Subcommand to create a new seeder class
# CategorySeeder Name of the seeder (convention: <Model>Seeder)
```

Open `database/seeders/CategorySeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Illuminate\Support\Str;
use App\Models\Category;

class CategorySeeder extends Seeder
{
    public function run()
    {
        $categories = [
            ['name' => 'Academic', 'description' => 'Academic events and lectures'],
            ['name' => 'Sports', 'description' => 'Sports and fitness events'],
            ['name' => 'Social', 'description' => 'Social gatherings and networking'],
            ['name' => 'Cultural', 'description' => 'Cultural and arts events'],
            ['name' => 'Workshop', 'description' => 'Educational workshops'],
        ];

        foreach ($categories as $category) {
            Category::create([
                'name' => $category['name'],
                // Str::slug takes a readable string and returns a slug suitable for URLs and identifiers
                // 'Workshops & Training'-> 'workshops-training'
                'slug' => Str::slug($category['name']),
                'description' => $category['description'],
            ]);
        }
    }
}
```

### Create User Seeder

```bash
php artisan make:seeder UserSeeder
```

Open `database/seeders/UserSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run()
    {
        // Create admin user
        User::create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => Hash::make('password'),
            'role' => 'admin',
        ]);

        // Create organizer user
        User::create([
            'name' => 'Event Organizer',
            'email' => 'organizer@example.com',
            'password' => Hash::make('password'),
            'role' => 'organizer',
        ]);

        // Create attendee users
        User::create([
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => Hash::make('password'),
            'role' => 'attendee',
        ]);

        User::create([
            'name' => 'Jane Smith',
            'email' => 'jane@example.com',
            'password' => Hash::make('password'),
            'role' => 'attendee',
        ]);
    }
}
```

### Create Event Seeder

```bash
php artisan make:seeder EventSeeder
```

Open `database/seeders/EventSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\Event;
use App\Models\User;
use App\Models\Category;
use Illuminate\Support\Str;
use Carbon\Carbon; // Carbon is a date/time library built on PHP’s DateTime

class EventSeeder extends Seeder
{
    public function run()
    {
        $organizer = User::where('role', 'organizer')->first();
        $categories = Category::all();

        $events = [
            [
                'title' => 'Laravel Workshop',
                'description' => 'Learn Laravel framework from scratch',
                'date' => Carbon::now()->addDays(7),
                'time' => '10:00:00',
                'location' => 'Main Hall',
                'capacity' => 50,
                'status' => 'published',
                'category_id' => $categories->where('name', 'Workshop')->first()->id,
            ],
            [
                'title' => 'Basketball Tournament',
                'description' => 'Annual university basketball tournament',
                'date' => Carbon::now()->addDays(14),
                'time' => '14:00:00',
                'location' => 'Sports Complex',
                'capacity' => 200,
                'status' => 'published',
                'category_id' => $categories->where('name', 'Sports')->first()->id,
            ],
            [
                'title' => 'Networking Event',
                'description' => 'Meet professionals and expand your network',
                'date' => Carbon::now()->addDays(21),
                'time' => '18:00:00',
                'location' => 'Conference Center',
                'capacity' => 100,
                'status' => 'published',
                'category_id' => $categories->where('name', 'Social')->first()->id,
            ],
        ];

        foreach ($events as $eventData) {
            Event::create(array_merge($eventData, [
                'organizer_id' => $organizer->id,
                'slug' => Str::slug($eventData['title']),
            ]));
        }
    }
}
```

## Step 11: Update DatabaseSeeder

Open `database/seeders/DatabaseSeeder.php`:

```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        $this->call([
            CategorySeeder::class,
            UserSeeder::class,
            EventSeeder::class,
        ]);
    }
}
```

## Step 12: Run Seeders

Run all seeders to populate the database:

```bash
php artisan db:seed
```

Or run specific seeders:

```bash
php artisan db:seed --class=CategorySeeder
php artisan db:seed --class=UserSeeder
php artisan db:seed --class=EventSeeder
```

## Step 13: Verify Data

## Database Schema Overview

### Tables Created

1. **users** - User accounts with roles (admin, organizer, attendee)
2. **categories** - Event categories
3. **events** - Event information with relationships
4. **registrations** - Many-to-many relationship between users and events

### Key Relationships

- **User** has many **Events** (as organizer)
- **User** belongs to many **Events** (through registrations)
- **Event** belongs to **Category**
- **Event** belongs to **User** (organizer)
- **Event** belongs to many **Users** (through registrations)

### Foreign Keys

- `events.category_id` → `categories.id` (SET NULL on delete)
- `events.organizer_id` → `users.id` (CASCADE on delete)
- `registrations.user_id` → `users.id` (CASCADE on delete)
- `registrations.event_id` → `events.id` (CASCADE on delete)

## Common Commands

```bash
# Run migrations
php artisan migrate

# Rollback last migration
php artisan migrate:rollback

# Rollback all migrations
php artisan migrate:reset

# Refresh (rollback + migrate)
php artisan migrate:refresh

# Fresh (drop all + migrate)
php artisan migrate:fresh

# Run seeders
php artisan db:seed

# Fresh migration with seeding
php artisan migrate:fresh --seed
```
