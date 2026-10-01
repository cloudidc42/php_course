# Part 55: Laravel Octane

## ขั้นตอนที่ 1511-1540: Laravel Octane สำหรับ High Performance

Laravel Octane เพิ่มความเร็วแอปพลิเคชันด้วยการ bootstrap ครั้งเดียวและรับ requests หลายๆ ครั้ง

---

## ขั้นตอนที่ 1511: Laravel Octane กับ Swoole

```bash
# ติดตั้ง Octane
composer require laravel/octane

# ติดตั้ง Swoole extension
pecl install swoole

# Publish config
php artisan octane:install --server=swoole

# รัน server
php artisan octane:start --server=swoole --host=0.0.0.0 --port=8000 --workers=8

# รัน พร้อม watch (development)
php artisan octane:start --watch
```

```php
<?php

declare(strict_types=1);

// config/octane.php
return [
    'server'  => env('OCTANE_SERVER', 'swoole'),
    'https'   => env('OCTANE_HTTPS', false),
    'listeners' => [
        WorkerStarting::class => [
            \App\Octane\EnsureHotCachesAreWarm::class,
        ],
        RequestReceived::class => [
            \Laravel\Octane\Listeners\DisconnectFromDatabases::class,
            \Laravel\Octane\Listeners\CollectGarbage::class,
        ],
        RequestHandled::class => [
            \App\Octane\FlushDirtyModels::class,
        ],
        RequestTerminated::class => [
            // Listeners after request completes
        ],
    ],
    'warm'    => [
        \App\Services\CriticalService::class,
    ],
    'flush'   => [
        // Singleton instances ที่ต้อง flush หลัง request
    ],
    'garbage_collection' => [
        'threshold' => 50,    // Every 50 requests
    ],
    'swoole' => [
        'options' => [
            'log_file'                => storage_path('logs/swoole_http.log'),
            'worker_num'              => env('OCTANE_SWOOLE_WORKERS', cpu_count() * 2),
            'task_worker_num'         => env('OCTANE_SWOOLE_TASK_WORKERS', cpu_count()),
            'package_max_length'      => 10 * 1024 * 1024, // 10MB
            'reload_async'            => true,
            'max_request'             => 500,    // Restart worker after 500 requests
            'max_wait_time'           => 60,
            'enable_coroutine'        => true,
            'send_yield'              => true,
            'socket_buffer_size'      => 2 * 1024 * 1024,
        ],
    ],
];
```

---

## ขั้นตอนที่ 1512: Concurrent Requests ด้วย Octane

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Laravel\Octane\Facades\Octane;

class DashboardController
{
    public function index(Request $request)
    {
        // รัน queries แบบ concurrent ด้วย Swoole coroutines
        [$users, $orders, $revenue, $stats] = Octane::concurrently([
            fn () => \App\Models\User::active()->count(),
            fn () => \App\Models\Order::today()->count(),
            fn () => \App\Models\Order::today()->sum('total'),
            fn () => app(\App\Services\StatsService::class)->getMetrics(),
        ]);

        return response()->json([
            'users'   => $users,
            'orders'  => $orders,
            'revenue' => $revenue,
            'stats'   => $stats,
        ]);
    }

    public function apiAggregator(Request $request)
    {
        // Fetch from multiple APIs concurrently
        [$weather, $news, $exchange] = Octane::concurrently([
            fn () => app(\App\Services\WeatherService::class)->getCurrent(),
            fn () => app(\App\Services\NewsService::class)->getLatest(5),
            fn () => app(\App\Services\ExchangeService::class)->getRates(),
        ], timeout: 5000); // 5 second timeout

        return response()->json(compact('weather', 'news', 'exchange'));
    }
}
```

---

## ขั้นตอนที่ 1513: Memory Management ใน Long-running Processes

```php
<?php

declare(strict_types=1);

namespace App\Octane;

use Laravel\Octane\Contracts\ServesRequests;

// สิ่งที่ต้องระวังใน Octane (long-running process):
// 1. Static variables - ยังคงอยู่ระหว่าง requests
// 2. Singleton instances - อาจมี state เก่า
// 3. Global state - ไม่ reset อัตโนมัติ

class MemoryLeakExample
{
    // BAD: Static state สะสม
    private static array $processedIds = [];

    public static function processItemBad(int $id): void
    {
        static::$processedIds[] = $id; // Memory leak!
    }

    // GOOD: ใช้ request-scoped state
    public function processItemGood(int $id, array &$processedIds): void
    {
        $processedIds[] = $id; // Reset ทุก request
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Octane\Listeners;

use Laravel\Octane\Events\RequestHandled;

class FlushDirtyModels
{
    public function handle(RequestHandled $event): void
    {
        // Flush Eloquent model states
        // ป้องกัน state รั่วไหลระหว่าง requests

        // ล้าง authentication state
        auth()->forgetGuards();

        // ล้าง session state
        app('session')->flush();
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Services;

// Service ที่ปลอดภัยสำหรับ Octane - Stateless
class OrderService
{
    // ไม่มี instance variables ที่เก็บ state ระหว่าง requests
    public function __construct(
        private readonly \App\Repositories\OrderRepository $repository
    ) {}

    // ทุก method รับ parameters และ return results
    // ไม่มี side effects บน $this
    public function createOrder(array $data): \App\Models\Order
    {
        return $this->repository->create($data);
    }
}

// Service ที่ต้องระวัง - Stateful
class CacheWarmer
{
    private array $warmedCache = []; // State ที่อาจ stale

    // GOOD: Reset ใน constructor หรือ method เสมอ
    public function warm(): void
    {
        $this->warmedCache = []; // Reset before use

        $data = \DB::table('settings')->get();
        foreach ($data as $setting) {
            $this->warmedCache[$setting->key] = $setting->value;
        }

        cache()->putMany($this->warmedCache, now()->addHour());
        $this->warmedCache = []; // Cleanup
    }
}
```

---

## ขั้นตอนที่ 1514: Octane กับ FrankenPHP

```bash
# ติดตั้ง FrankenPHP (alternative to Swoole)
composer require laravel/octane

php artisan octane:install --server=frankenphp

# รัน FrankenPHP
php artisan octane:frankenphp

# Docker
docker run -p 8080:8080 \
    -v $PWD:/app \
    dunglas/frankenphp \
    php artisan octane:frankenphp --port=8080
```

```php
<?php

declare(strict_types=1);

// config/octane.php สำหรับ FrankenPHP
return [
    'server' => env('OCTANE_SERVER', 'frankenphp'),
    'frankenphp' => [
        'workers' => env('OCTANE_WORKERS', 2),
        'max_requests' => env('OCTANE_MAX_REQUESTS', 500),
    ],
];
```

---

## ขั้นตอนที่ 1515: Performance Benchmarks

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Symfony\Component\Process\Process;

class BenchmarkServer extends Command
{
    protected $signature = 'octane:benchmark
                            {--requests=1000 : Number of requests}
                            {--concurrency=50 : Concurrent requests}
                            {--url=http://localhost:8000 : URL to benchmark}';

    protected $description = 'Benchmark the Octane server';

    public function handle(): int
    {
        $requests    = (int)$this->option('requests');
        $concurrency = (int)$this->option('concurrency');
        $url         = $this->option('url');

        $this->info("Benchmarking {$url}");
        $this->info("Requests: {$requests}, Concurrency: {$concurrency}");

        // ใช้ Apache Bench
        $process = new Process([
            'ab',
            '-n', $requests,
            '-c', $concurrency,
            '-H', 'Accept: application/json',
            $url,
        ]);

        $process->setTimeout(300);
        $process->run();

        if (!$process->isSuccessful()) {
            $this->error('Benchmark failed: ' . $process->getErrorOutput());
            return Command::FAILURE;
        }

        $output = $process->getOutput();

        // Parse results
        preg_match('/Requests per second:\s+([\d.]+)/', $output, $rpsMatch);
        preg_match('/Time per request:\s+([\d.]+) \[ms\] \(mean\)/', $output, $latencyMatch);
        preg_match('/Failed requests:\s+(\d+)/', $output, $failedMatch);

        $this->table(
            ['Metric', 'Value'],
            [
                ['Requests/sec',    ($rpsMatch[1] ?? 'N/A') . ' req/s'],
                ['Mean Latency',    ($latencyMatch[1] ?? 'N/A') . ' ms'],
                ['Failed Requests', $failedMatch[1] ?? '0'],
            ]
        );

        return Command::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1516: Production Configuration

```php
<?php

declare(strict_types=1);

// Supervisor config สำหรับ Octane
// /etc/supervisor/conf.d/octane.conf

/*
[program:octane]
command=php /var/www/myapp/artisan octane:start --server=swoole --host=0.0.0.0 --port=8000 --workers=16
directory=/var/www/myapp
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/www/myapp/storage/logs/octane.log
stopwaitsecs=60
*/

// Nginx config สำหรับ Octane
/*
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Connection "keep-alive";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        root /var/www/myapp/public;
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }
}
*/
```

```php
<?php

declare(strict_types=1);

// Octane Health Check Endpoint
namespace App\Http\Controllers;

use Illuminate\Http\JsonResponse;
use Laravel\Octane\Facades\Octane;

class OctaneHealthController
{
    public function health(): JsonResponse
    {
        return response()->json([
            'status'     => 'healthy',
            'octane'     => [
                'server'   => config('octane.server'),
                'workers'  => $this->getWorkerCount(),
            ],
            'php'        => [
                'version'  => PHP_VERSION,
                'sapi'     => PHP_SAPI,
                'memory'   => round(memory_get_usage(true) / 1024 / 1024, 2) . 'MB',
            ],
            'timestamp'  => now()->toISOString(),
        ]);
    }

    private function getWorkerCount(): int
    {
        return (int)(config('octane.swoole.options.worker_num')
            ?? config('octane.frankenphp.workers', 2));
    }
}
```

---

## ขั้นตอนที่ 1517: Performance Tips สำหรับ Octane

```php
<?php

declare(strict_types=1);

// Tips สำหรับ code ที่ทำงานใน Octane

// 1. Cache expensive operations ใน worker memory (ไม่ต้องผ่าน Redis)
namespace App\Services;

class InMemoryCache
{
    private static array $cache = [];

    public static function remember(string $key, int $ttl, callable $callback): mixed
    {
        $now = time();

        if (isset(self::$cache[$key]) && self::$cache[$key]['expires'] > $now) {
            return self::$cache[$key]['value'];
        }

        $value = $callback();
        self::$cache[$key] = [
            'value'   => $value,
            'expires' => $now + $ttl,
        ];

        return $value;
    }

    public static function flush(): void
    {
        self::$cache = [];
    }
}

// 2. ใช้ connection pooling ใน Swoole
// Swoole จัดการ connection pool ให้อัตโนมัติ
// ไม่ต้องสร้าง connection ใหม่ทุก request

// 3. Octane::concurrently สำหรับ I/O bound operations
// 4. ระวัง global state และ static variables
// 5. ใช้ Request::flush() / Auth::forgetGuards()
```

---

## ขั้นตอนที่ 1518: Octane vs Traditional PHP-FPM

```
Performance Comparison (1000 requests, 50 concurrent):

PHP-FPM (traditional):
  Requests/sec: ~150
  Latency: ~330ms
  Memory: 64MB per worker

Laravel Octane + Swoole:
  Requests/sec: ~1200 (8x faster!)
  Latency: ~42ms
  Memory: 64MB (shared, bootstrapped once)

Laravel Octane + FrankenPHP:
  Requests/sec: ~900 (6x faster!)
  Latency: ~56ms
  Memory: 80MB (HTTP/3 + TLS included)

Notes:
- Octane benefits most from:
  * Heavy framework bootstrapping (service providers)
  * Many composer packages
  * Complex middleware stacks

- Octane benefits less from:
  * Pure I/O bound apps
  * Simple CRUD with mostly DB time
```

---

## สรุปบทที่ 55

| เครื่องมือ | Requests/sec | ความยาก | เมื่อใช้ |
|-----------|-------------|---------|---------|
| PHP-FPM | ~150 | ง่าย | Default, stable |
| Octane + Swoole | ~1200 | ปานกลาง | High traffic |
| Octane + FrankenPHP | ~900 | ง่าย | HTTP/3 needed |
| Concurrent | 3-5x I/O | ปานกลาง | Multiple APIs |

**จบหลักสูตร Parts 39-55!** กลับไปดู [Part 01](../part-01-introduction/README.md) สำหรับพื้นฐาน

---

## สรุปภาพรวม Parts 39-55

| Part | หัวข้อ | Steps |
|------|--------|-------|
| 39 | Advanced Testing | 1031-1060 |
| 40 | Security Advanced | 1061-1090 |
| 41 | Laravel Advanced | 1091-1120 |
| 42 | Laravel API | 1121-1150 |
| 43 | Design Patterns | 1151-1180 |
| 44 | Laravel Livewire | 1181-1210 |
| 45 | Database Advanced | 1211-1240 |
| 46 | Laravel Queues | 1241-1270 |
| 47 | PHP CLI | 1271-1300 |
| 48 | Laravel Notifications | 1301-1330 |
| 49 | WordPress Headless | 1331-1360 |
| 50 | Monitoring & Logging | 1361-1390 |
| 51 | Laravel Broadcasting | 1391-1420 |
| 52 | PHP Internals | 1421-1450 |
| 53 | Laravel Telescope | 1451-1480 |
| 54 | API Integration | 1481-1510 |
| 55 | Laravel Octane | 1511-1540 |

---
