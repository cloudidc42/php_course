# Part 23: Laravel Advanced - Queue, Events, Cache
## ขั้นตอนที่ 581-610: Enterprise Features

---

## ขั้นตอนที่ 581: Queue System

```bash
# config/queue.php
# Drivers: sync, database, redis, sqs, beanstalkd

# Database Driver
php artisan queue:table
php artisan migrate

# สร้าง Job
php artisan make:job SendWelcomeEmail
php artisan make:job ProcessOrderPayment

# รัน Worker
php artisan queue:work
php artisan queue:work --queue=high,default,low
php artisan queue:work redis --queue=emails --tries=3 --timeout=60
php artisan queue:work --sleep=3 --tries=3 --max-jobs=1000
php artisan queue:work --daemon  # Keep running

# Queue Management
php artisan queue:listen
php artisan queue:restart   # Graceful restart
php artisan queue:monitor   # Monitor queues
php artisan queue:failed    # List failed jobs
php artisan queue:retry all # Retry failed
php artisan queue:flush     # Delete all failed
```

```php
<?php
// app/Jobs/SendWelcomeEmail.php

namespace App\Jobs;

use App\Models\User;
use App\Mail\WelcomeEmail;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue {
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public int $tries = 3;          // Retry 3 times
    public int $timeout = 30;       // 30 second timeout
    public int $backoff = 60;       // Wait 60s between retries
    public int $maxExceptions = 2;  // Max unhandled exceptions
    
    // Delete job if model no longer exists
    public bool $deleteWhenMissingModels = true;
    
    public function __construct(
        private User $user,
        private string $temporaryPassword = '',
    ) {}
    
    public function handle(): void {
        Mail::to($this->user->email)
            ->send(new WelcomeEmail($this->user, $this->temporaryPassword));
    }
    
    // Called when all retries fail
    public function failed(\Throwable $exception): void {
        // Notify admin, log to monitoring service
        \Log::critical('SendWelcomeEmail failed', [
            'user_id' => $this->user->id,
            'error' => $exception->getMessage(),
        ]);
        
        // Notify via Slack, PagerDuty, etc.
    }
    
    // Custom retry delay (exponential backoff)
    public function retryUntil(): \DateTime {
        return now()->addHours(6);
    }
    
    // Queue to use
    public function queue(): string {
        return 'emails';
    }
    
    // Tags for monitoring
    public function tags(): array {
        return ['email', 'user:' . $this->user->id];
    }
}

// Dispatching Jobs
SendWelcomeEmail::dispatch($user);
SendWelcomeEmail::dispatch($user)->onQueue('high');
SendWelcomeEmail::dispatch($user)->delay(now()->addMinutes(5));
SendWelcomeEmail::dispatch($user)->onConnection('redis');

// Chain Jobs (run in sequence)
SendWelcomeEmail::withChain([
    new CreateUserProfile($user),
    new SendOnboardingTips($user),
])->dispatch($user);

// Batch Jobs (run in parallel)
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

$batch = Bus::batch([
    new ProcessReport($report, 'page1'),
    new ProcessReport($report, 'page2'),
    new ProcessReport($report, 'page3'),
])->then(function (Batch $batch) {
    // All jobs completed
    $report->markComplete();
})->catch(function (Batch $batch, \Throwable $e) {
    // Some jobs failed
})->finally(function (Batch $batch) {
    // Always runs
})->name('Process Report')
 ->onQueue('reports')
 ->dispatch();

echo $batch->id; // Track batch
```

---

## ขั้นตอนที่ 582: Events & Listeners

```php
<?php
// app/Events/OrderPlaced.php
namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderPlaced {
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function __construct(
        public Order $order,
        public string $paymentMethod,
    ) {}
    
    // For Broadcasting (WebSockets)
    public function broadcastOn(): array {
        return [
            new \Illuminate\Broadcasting\PrivateChannel('user.' . $this->order->user_id),
        ];
    }
    
    public function broadcastAs(): string {
        return 'order.placed';
    }
    
    public function broadcastWith(): array {
        return [
            'order_id' => $this->order->id,
            'total' => $this->order->total,
        ];
    }
}

// ========================
// Listeners
// ========================
// php artisan make:listener SendOrderConfirmation --event=OrderPlaced

namespace App\Listeners;

use App\Events\OrderPlaced;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendOrderConfirmation implements ShouldQueue {
    public string $queue = 'emails';
    
    public function handle(OrderPlaced $event): void {
        \Mail::to($event->order->user->email)
            ->send(new \App\Mail\OrderConfirmation($event->order));
    }
    
    public function shouldQueue(OrderPlaced $event): bool {
        return $event->order->total > 100; // Only queue for large orders
    }
    
    public function failed(OrderPlaced $event, \Throwable $exception): void {
        \Log::error('SendOrderConfirmation failed', [
            'order_id' => $event->order->id,
            'error' => $exception->getMessage(),
        ]);
    }
}

class UpdateInventory {
    public function handle(OrderPlaced $event): void {
        foreach ($event->order->items as $item) {
            $item->product->decrement('stock', $item->quantity);
        }
    }
}

class NotifyWarehouse {
    public function handle(OrderPlaced $event): void {
        // Send to warehouse system
    }
}

// ========================
// Register in EventServiceProvider (Laravel 10)
// OR bootstrap/app.php (Laravel 11)
// ========================

// bootstrap/app.php (Laravel 11)
->withEvents(function (Discover $events) {
    $events->in([__DIR__ . '/../app/Listeners']);
})

// OR manual registration
\Illuminate\Support\Facades\Event::listen(
    OrderPlaced::class,
    [SendOrderConfirmation::class, 'handle']
);

// Auto discovery (recommended)
// Laravel scans Listeners directory automatically

// ========================
// Dispatching Events
// ========================
event(new OrderPlaced($order, 'credit_card'));
OrderPlaced::dispatch($order, 'paypal');

// Fire and forget with delay
event(new OrderPlaced($order, 'qr_code'))->delay(now()->addMinutes(1));

// Observer (Model Events)
// php artisan make:observer UserObserver --model=User
class UserObserver {
    public function creating(User $user): void {
        $user->uuid = \Str::uuid();
    }
    
    public function created(User $user): void {
        SendWelcomeEmail::dispatch($user);
        CreateUserProfile::dispatch($user);
    }
    
    public function updating(User $user): void {
        if ($user->isDirty('email')) {
            $user->email_verified_at = null;
        }
    }
    
    public function deleting(User $user): void {
        $user->posts()->delete();
        $user->profile()->delete();
    }
}

// Register observer
User::observe(UserObserver::class);
// Or in AppServiceProvider: User::observe(UserObserver::class);
```

---

## ขั้นตอนที่ 583: Caching

```php
<?php
use Illuminate\Support\Facades\Cache;

// ========================
// Basic Cache Operations
// ========================
// Store
Cache::put('key', 'value', now()->addHours(1));
Cache::put('key', 'value', 3600); // seconds
Cache::forever('key', 'value'); // Never expires

// Get
$value = Cache::get('key');
$value = Cache::get('key', 'default');
$value = Cache::get('key', fn() => compute_expensive_value());

// Remember (get or store if not exists)
$users = Cache::remember('users.all', 3600, function () {
    return User::with('posts')->get();
});

// Remember forever
$config = Cache::rememberForever('app.config', fn() => loadConfig());

// Check & Remove
Cache::has('key');
Cache::missing('key');
Cache::forget('key');
Cache::flush(); // Clear all

// Get & Delete
$value = Cache::pull('key');

// Increment / Decrement
Cache::increment('visitors');
Cache::increment('visitors', 5);
Cache::decrement('stock');

// Store multiple
Cache::putMany(['key1' => 'val1', 'key2' => 'val2'], 3600);
Cache::many(['key1', 'key2']);

// Tags (Redis/Memcached only)
Cache::tags(['users', 'permissions'])->put('user:1:perms', $perms, 3600);
$perms = Cache::tags(['users', 'permissions'])->get('user:1:perms');
Cache::tags('users')->flush(); // Clear all user cache

// ========================
// Cache Strategies
// ========================
class PostRepository {
    private const CACHE_TTL = 3600;
    
    public function findPopular(int $limit = 10): \Illuminate\Database\Eloquent\Collection {
        return Cache::remember("posts.popular.{$limit}", self::CACHE_TTL, function () use ($limit) {
            return Post::where('status', 'published')
                ->orderByDesc('views')
                ->limit($limit)
                ->with(['user', 'tags'])
                ->get();
        });
    }
    
    public function findBySlug(string $slug): ?Post {
        return Cache::remember("post.{$slug}", self::CACHE_TTL, fn() => 
            Post::where('slug', $slug)->with(['user', 'comments', 'tags'])->first()
        );
    }
    
    public function create(array $data): Post {
        $post = Post::create($data);
        $this->clearCache();
        return $post;
    }
    
    public function update(Post $post, array $data): Post {
        $post->update($data);
        Cache::forget("post.{$post->slug}");
        $this->clearCache();
        return $post->fresh();
    }
    
    private function clearCache(): void {
        Cache::forget('posts.popular.10');
        Cache::forget('posts.popular.5');
        // Or with tags:
        // Cache::tags('posts')->flush();
    }
}

// ========================
// HTTP Response Caching
// ========================
Route::get('/sitemap.xml', [SitemapController::class, 'index'])
    ->middleware('cache.headers:public;max_age=3600;etag');

// Cache a slow API response
public function stats(): JsonResponse {
    $stats = Cache::remember('dashboard.stats', 300, function () {
        return [
            'users' => User::count(),
            'orders' => Order::count(),
            'revenue' => Order::sum('total'),
            'top_products' => Product::orderByDesc('sales')->limit(5)->get(),
        ];
    });
    
    return response()->json($stats)
        ->header('Cache-Control', 'max-age=300, public');
}

// ========================
// Cache Lock (Prevents race conditions)
// ========================
$lock = Cache::lock('process-payment', 10); // 10 second lock

if ($lock->get()) {
    try {
        processPayment($order);
    } finally {
        $lock->release();
    }
} else {
    // Another process has the lock
    return response()->json(['message' => 'Processing in progress'], 429);
}

// Block until available
Cache::lock('sync-inventory')->block(5, function () {
    syncInventory();
});
```

---

## ขั้นตอนที่ 584: Laravel Horizon & Telescope

```bash
# Laravel Horizon (Queue monitoring dashboard)
composer require laravel/horizon
php artisan horizon:install
php artisan migrate
php artisan horizon  # Start Horizon
# /horizon to see dashboard

# config/horizon.php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'maxProcesses' => 10,
            'balanceMaxShift' => 1,
            'balanceCooldown' => 3,
        ],
    ],
    'local' => [
        'supervisor-1' => ['maxProcesses' => 3],
    ],
],

# Laravel Telescope (Debug tool)
composer require laravel/telescope --dev
php artisan telescope:install
php artisan migrate
# /telescope to see dashboard
# Shows: requests, queries, jobs, cache, mail, etc.
```

---

## 🎯 สรุป Part 23

| หัวข้อ | สิ่งที่สำคัญ |
|--------|-------------|
| Queue | ShouldQueue, dispatch, chain, batch |
| Jobs | tries, timeout, backoff, failed() |
| Events | Dispatchable, Broadcasting |
| Listeners | ShouldQueue, handle() |
| Observer | Model lifecycle hooks |
| Cache | remember(), tags, lock |

**ถัดไป → Part 24: Laravel API & API Resources**
