# Part 53: Laravel Telescope & Debugging

## ขั้นตอนที่ 1451-1480: Debugging และ Profiling

Telescope, Debugbar, Ray และเครื่องมือ debug ต่างๆ ช่วยวิเคราะห์และแก้ไขปัญหาได้รวดเร็ว

---

## ขั้นตอนที่ 1451: Laravel Telescope Setup

```bash
# ติดตั้ง Telescope
composer require laravel/telescope --dev

# Publish
php artisan telescope:install

# Migrate
php artisan migrate
```

```php
<?php

declare(strict_types=1);

// app/Providers/TelescopeServiceProvider.php
namespace App\Providers;

use Laravel\Telescope\IncomingEntry;
use Laravel\Telescope\Telescope;
use Laravel\Telescope\TelescopeApplicationServiceProvider;

class TelescopeServiceProvider extends TelescopeApplicationServiceProvider
{
    public function register(): void
    {
        Telescope::night(); // Dark mode

        $this->hideSensitiveRequestDetails();

        // กรอง entries ที่บันทึก
        Telescope::filter(function (IncomingEntry $entry) {
            if ($this->app->environment('local')) {
                return true; // บันทึกทั้งหมดใน local
            }

            return $entry->isReportableException()
                || $entry->isFailedRequest()
                || $entry->isFailedJob()
                || $entry->isScheduledTask()
                || $entry->hasMonitoredTag();
        });
    }

    protected function hideSensitiveRequestDetails(): void
    {
        if ($this->app->environment('local')) {
            return;
        }

        Telescope::hideRequestParameters([
            'password',
            'password_confirmation',
            'credit_card',
            'cvv',
            '_token',
        ]);

        Telescope::hideRequestHeaders([
            'cookie',
            'x-csrf-token',
            'authorization',
        ]);
    }

    public function boot(): void
    {
        parent::boot();

        // กำหนดใครที่เข้า Telescope ได้
        Telescope::auth(function ($request) {
            return $request->user()?->hasRole('developer')
                ?? app()->environment('local');
        });

        // Tag entries สำหรับ filtering
        Telescope::tag(function (IncomingEntry $entry) {
            $tags = [];

            if ($entry->type === 'request') {
                $tags[] = 'api:' . ($entry->content['uri'] ?? '');

                if (isset($entry->content['response_status'])) {
                    $status = $entry->content['response_status'];
                    if ($status >= 400) {
                        $tags[] = 'error';
                    }
                    if ($status >= 500) {
                        $tags[] = 'server-error';
                    }
                }
            }

            return $tags;
        });
    }
}
```

---

## ขั้นตอนที่ 1452: Laravel Debugbar

```bash
# ติดตั้ง Debugbar
composer require barryvdh/laravel-debugbar --dev

# Publish config
php artisan vendor:publish --provider="Barryvdh\Debugbar\ServiceProvider"
```

```php
<?php

declare(strict_types=1);

// config/debugbar.php
return [
    'enabled' => env('DEBUGBAR_ENABLED', false),
    'storage' => [
        'enabled'  => true,
        'driver'   => 'file',
        'path'     => storage_path('debugbar'),
        'open'     => 'phpfile',
        'lifetime' => 60,
    ],
    'collectors' => [
        'phpinfo'         => true,
        'messages'        => true,
        'time'            => true,
        'memory'          => true,
        'exceptions'      => true,
        'log'             => true,
        'db'              => true,
        'views'           => true,
        'route'           => true,
        'auth'            => true,
        'gate'            => true,
        'session'         => true,
        'symfony_request' => true,
        'mail'            => true,
        'laravel'         => false,
        'events'          => false,
        'default_request' => false,
        'logs'            => false,
        'files'           => false,
        'config'          => false,
        'cache'           => true,
        'models'          => true,
        'livewire'        => true,
    ],
];
```

```php
<?php

declare(strict_types=1);

// การใช้ Debugbar ใน code
use Barryvdh\Debugbar\Facades\Debugbar;

class OrderController
{
    public function index()
    {
        // เพิ่ม message
        Debugbar::info('Loading orders...');
        Debugbar::warning('This is expensive!');

        // Measure time
        Debugbar::startMeasure('orders_query', 'Loading orders');
        $orders = Order::with('items')->paginate(20);
        Debugbar::stopMeasure('orders_query');

        // Add data
        Debugbar::addMessage(['count' => $orders->count()], 'orders');

        // Exception
        try {
            $this->doSomething();
        } catch (\Exception $e) {
            Debugbar::addException($e);
        }

        return view('orders.index', compact('orders'));
    }
}
```

---

## ขั้นตอนที่ 1453: Ray Debugging Tool

```bash
# ติดตั้ง Ray
composer require spatie/laravel-ray --dev
```

```php
<?php

declare(strict_types=1);

// การใช้ Ray
use Spatie\Ray\Ray;

function processOrder(array $orderData): void
{
    // ส่งข้อมูลไป Ray desktop app
    ray('Processing order', $orderData);

    // Label
    ray($orderData)->label('Order Data');

    // Measure
    ray()->measure();
    $result = expensiveOperation($orderData);
    ray()->measure(); // Shows elapsed time

    // Conditional
    ray()->showIf($result === null, 'Result is null!');

    // Table
    ray($orderData)->table('Order items');

    // Type
    ray($orderData)->showMemoryUsage();

    // Color
    ray('Payment received')->green();
    ray('Payment failed')->red();
    ray('Processing...')->orange();

    // Track model changes
    $order = Order::create($orderData);
    ray($order)->blue();

    // Queries
    ray()->showQueries();
    $orders = Order::where('status', 'pending')->get();
    ray()->stopShowingQueries();

    // Pause execution (ใน dev เท่านั้น!)
    // ray()->pause();
}
```

---

## ขั้นตอนที่ 1454: Query Debugging

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Database\Events\QueryExecuted;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\ServiceProvider;

class QueryDebugServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        if (!app()->environment('production')) {
            DB::listen(function (QueryExecuted $query) {
                $sql = vsprintf(
                    str_replace(['%', '?'], ['%%', "'%s'"], $query->sql),
                    $query->bindings
                );

                if ($query->time > 100) { // มากกว่า 100ms
                    Log::channel('slow_queries')->warning('Slow query', [
                        'sql'      => $sql,
                        'time_ms'  => $query->time,
                        'trace'    => collect(debug_backtrace())
                            ->filter(fn ($t) => str_starts_with($t['file'] ?? '', base_path('app')))
                            ->take(3)
                            ->values()
                            ->toArray(),
                    ]);
                }
            });
        }
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class DetectNPlusOne
{
    private array $queryPatterns = [];

    public function handle(Request $request, Closure $next): mixed
    {
        if (!config('app.debug')) {
            return $next($request);
        }

        DB::enableQueryLog();

        $response = $next($request);

        $queries = DB::getQueryLog();

        // ตรวจหา queries ที่ซ้ำกัน
        $normalized = array_map(function ($q) {
            return preg_replace('/\b\d+\b/', '?', $q['query']);
        }, $queries);

        $counts = array_count_values($normalized);
        $repeated = array_filter($counts, fn ($c) => $c > 3);

        if (!empty($repeated)) {
            Log::warning('Potential N+1 queries detected', [
                'url'      => $request->url(),
                'patterns' => $repeated,
                'total'    => count($queries),
            ]);

            if (app()->environment('local')) {
                // เพิ่ม header เตือนใน local
                $response->headers->set('X-N-Plus-One-Warning', 'true');
                $response->headers->set('X-Query-Count', count($queries));
            }
        }

        return $response;
    }
}
```

---

## ขั้นตอนที่ 1455: Performance Profiling

```php
<?php

declare(strict_types=1);

namespace App\Profiling;

class SimpleProfiler
{
    private array $timers  = [];
    private array $records = [];

    public function start(string $name): void
    {
        $this->timers[$name] = [
            'start'  => microtime(true),
            'memory' => memory_get_usage(true),
        ];
    }

    public function stop(string $name): float
    {
        if (!isset($this->timers[$name])) {
            throw new \RuntimeException("Timer '{$name}' was not started");
        }

        $elapsed = (microtime(true) - $this->timers[$name]['start']) * 1000;
        $memoryDelta = memory_get_usage(true) - $this->timers[$name]['memory'];

        $this->records[] = [
            'name'         => $name,
            'elapsed_ms'   => round($elapsed, 3),
            'memory_delta' => round($memoryDelta / 1024, 1) . 'KB',
        ];

        unset($this->timers[$name]);

        return $elapsed;
    }

    public function measure(string $name, callable $callback): mixed
    {
        $this->start($name);
        $result = $callback();
        $this->stop($name);
        return $result;
    }

    public function getReport(): array
    {
        return $this->records;
    }

    public function printReport(): void
    {
        foreach ($this->records as $record) {
            printf(
                "%-40s %8.3fms  %s\n",
                $record['name'],
                $record['elapsed_ms'],
                $record['memory_delta']
            );
        }
    }
}
```

```php
<?php

declare(strict_types=1);

// การใช้งาน Profiler
$profiler = new SimpleProfiler();

// Profile ทีละส่วน
$profiler->start('database');
$users = User::with('orders')->get();
$profiler->stop('database');

$profiler->start('processing');
$result = processUsers($users);
$profiler->stop('processing');

$profiler->start('rendering');
$html = renderTemplate($result);
$profiler->stop('rendering');

$profiler->printReport();
/*
database                                 45.234ms  2.5MB
processing                              125.890ms  8.1MB
rendering                                12.456ms  0.8MB
*/
```

---

## ขั้นตอนที่ 1456: N+1 Detection และ Resolution

```php
<?php

declare(strict_types=1);

// N+1 Problem
// BAD: N+1 - 1 query for orders + N queries for each user
$orders = Order::all();
foreach ($orders as $order) {
    echo $order->user->name; // Query ทุกครั้ง!
}

// GOOD: Eager Loading - 2 queries total
$orders = Order::with('user')->get();
foreach ($orders as $order) {
    echo $order->user->name; // No extra query
}

// GOOD: Lazy eager loading
$orders = Order::all();
$orders->load('user', 'items.product');

// GOOD: Specific columns only
$orders = Order::with([
    'user:id,name,email',
    'items:id,order_id,quantity,price',
    'items.product:id,name',
])->get();

// When you need count only
$users = User::withCount(['orders', 'posts'])->get();
foreach ($users as $user) {
    echo "{$user->name}: {$user->orders_count} orders, {$user->posts_count} posts";
}

// Complex nested eager loading
$posts = Post::with([
    'author:id,name',
    'comments' => function ($query) {
        $query->latest()->limit(5)->with('user:id,name');
    },
    'tags:id,name',
])->published()->paginate(20);
```

---

## สรุปบทที่ 53

| เครื่องมือ | ประโยชน์ | Environment |
|-----------|---------|-------------|
| Telescope | Full request/query/job logging | Dev + Staging |
| Debugbar | In-browser debug info | Dev only |
| Ray | Desktop debug app | Dev only |
| Query Log | SQL analysis | Dev |
| N+1 Detector | Find missing eager loads | Dev + Staging |
| SimpleProfiler | Custom performance tracking | Dev |

**ต่อไป**: Part 54 - API Integration

---
