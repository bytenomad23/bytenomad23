# The Solo Developer’s Guide to a High-Performance, Low-Maintenance Laravel Stack

As a solo developer, my most valuable resource isn't CPU or RAM—it's my own time and mental energy. I wear all the hats: developer, sysadmin, DBA, and support desk. Every piece of my stack that demands constant attention is a tax on my ability to build and ship features.

For years, I followed the standard playbook for deploying my Laravel apps: a VPS running Nginx, MySQL or PostgreSQL, Redis for queues, and a separate backup service. It’s powerful, but it’s also a stack of independently moving parts that I alone am responsible for. Every server update, database patch, or configuration drift is a potential late-night emergency.

I became obsessed with a question: what if I could radically simplify my stack without sacrificing performance? What if my database didn't need a separate server?

That question led me to a setup that has completely changed how I build and deploy applications: **Laravel + SQLite, supercharged by Laravel Octane (or FrankenPHP).**

I know what you're thinking. "SQLite? For production? Is that safe?" I thought the same thing. But the conventional wisdom that SQLite can't handle concurrency is based on its default settings. With a few simple tweaks, it transforms into a production-ready powerhouse that is perfect for the solo developer.

### The Problem: Why Default SQLite Breaks Under Pressure

Modern PHP servers like Octane and FrankenPHP keep our Laravel app booted in memory and handle requests across multiple workers. This is where the speed comes from. But when you have 4, 8, or 16 workers all trying to write to a single SQLite file at once, they collide. The default locking behavior is to fail immediately, leading to the dreaded `SQLSTATE[HY000]: General error: 5 database is locked`.

This is the point where most developers give up and go back to MySQL. But the fix is surprisingly simple.

### My High-Performance Configuration: A Three-Step Checklist

This is my go-to setup for every new project. It’s a simple checklist that turns SQLite into a concurrent, high-performance database ready for a multi-worker environment.

#### Step 1: Flip the Switch to WAL Mode

This is the most critical step. You need to tell SQLite to use **Write-Ahead Logging (WAL)**. In simple terms, WAL mode lets read operations happen at the same time as write operations. It’s the key to concurrency.

Because Octane and FrankenPHP manage long-lived processes, you have to ensure this is set for every new database connection. The best place to do this is in your `AppServiceProvider`.

```php
// app/Providers/AppServiceProvider.php

use Illuminate\Support\Facades\Event;
use Illuminate\Database\Events\ConnectionEstablished;

public function boot(): void
{
    // This event fires every time a new database connection is made.
    Event::listen(ConnectionEstablished::class, function ($event) {
        if ($event->connection->getDriverName() === 'sqlite') {
            // Enable Write-Ahead Logging
            $event->connection->statement('PRAGMA journal_mode = WAL;');
            // Enforce foreign key constraints
            $event->connection->statement('PRAGMA foreign_keys = ON;');
        }
    });
}
```

#### Step 2: Add a Safety Net with `busy_timeout`

Even with WAL mode, only one process can write at a time. So what happens when two workers try to write at the exact same millisecond? Without a timeout, one will fail.

We solve this by setting a `busy_timeout`. This tells a worker to wait patiently for a few seconds if the database is locked by another writer. This completely smooths out write contention.

The place to set this is in your database configuration file.

```php
// config/database.php

'sqlite' => [
    'driver' => 'sqlite',
    'database' => env('DB_DATABASE', database_path('database.sqlite')),
    // ...
    'options' => [
        // Wait up to 5 seconds if the database is locked.
        PDO::ATTR_TIMEOUT => 5,
    ],
],
```

#### Step 3: Write Smarter, Not Harder

The final piece is to be efficient in your application code.

- **Batch Your Inserts:** Never insert records in a loop with `Model::create()`. If you need to insert 1,000 records, put them in an array and use `Model::insert($data)`. This wraps the entire operation in a single transaction and is over 100x faster. It’s the difference between a one-second operation and a one-minute operation.
- **Index Your `WHERE` Clauses:** If you have a query like `User::where('status', 'active')->get()`, make sure you have an index on the `status` column. In your migration, just add `$table->index('status');`. It’s that simple, and it prevents slow, full-table scans.

### The Solo Dev Dream Stack in Action

With this configuration, my stack is now radically simpler:

- **My App Server:** Laravel Octane with 4-8 workers.
- **My Database:** A single `database.sqlite` file, sitting right next to my code. There is zero network latency between my app and my database.
- **My Queue:** I use Laravel's `database` queue driver, which now also runs on my supercharged SQLite instance. No need for a separate Redis server.

My deployment process? A simple `git pull` and `php artisan octane:reload`. That's it.

### "But What About Backups?" — The Litestream Solution

This was my biggest concern as a solo dev. The solution, and what makes this whole stack viable for serious projects, is **Litestream**.

Litestream is a lightweight, standalone tool that runs as a background process on my server. It continuously streams changes from my SQLite database file to an S3-compatible bucket in real-time.

- I get disaster recovery down to the second.
- It costs pennies per month to run.
- The setup took me 15 minutes.

It's the perfect backup solution for this simple, powerful stack.

### Why This Matters for Solo Developers

Every decision I make is about reducing complexity and mental overhead. By running Laravel on a high-performance SQLite database, I've eliminated entire categories of work from my plate:

- No external database server to provision, patch, or secure.
- No database credentials to manage in my `.env` files.
- No separate Redis server to manage for my queues.
- No complex backup scripts to write and maintain.

I get to spend more time building features and less time managing infrastructure, all while running an application that is faster than most traditional setups. If you're a solo developer looking for that perfect balance of power and simplicity, I urge you to give this stack a try. It might just change the way you build.
