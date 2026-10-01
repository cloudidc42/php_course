# Part 67: PHP Concurrency

## ขั้นตอนที่ 1871-1900: Parallel Processing ใน PHP

PHP มีหลาย approach สำหรับ concurrency ตั้งแต่ pcntl_fork ไปจนถึง ReactPHP และ Amp

---

## ขั้นตอนที่ 1871: pcntl_fork Pattern

```php
<?php

declare(strict_types=1);

// process_pool.php
function processInParallel(array $items, callable $processor, int $maxProcesses = 4): array
{
    $results    = [];
    $children   = [];
    $chunks     = array_chunk($items, (int) ceil(count($items) / $maxProcesses));

    foreach ($chunks as $index => $chunk) {
        $pid = pcntl_fork();

        if ($pid === -1) {
            throw new RuntimeException('Cannot fork process');
        }

        if ($pid === 0) {
            // Child process
            $chunkResults = array_map($processor, $chunk);

            // Write results to temp file
            $tmpFile = sys_get_temp_dir() . "/parallel_result_{$index}_" . getmypid();
            file_put_contents($tmpFile, serialize($chunkResults));

            exit(0); // Child exits
        }

        // Parent: track child PID and temp file
        $children[$pid] = sys_get_temp_dir() . "/parallel_result_{$index}_{$pid}";
    }

    // Parent: wait for all children
    foreach ($children as $pid => $tmpFile) {
        pcntl_waitpid($pid, $status);

        if (pcntl_wexitstatus($status) === 0 && file_exists($tmpFile)) {
            $chunkResults = unserialize(file_get_contents($tmpFile));
            $results = array_merge($results, $chunkResults);
            unlink($tmpFile);
        }
    }

    return $results;
}

// ตัวอย่างใช้งาน
$urls = [
    'https://api1.example.com/data',
    'https://api2.example.com/data',
    'https://api3.example.com/data',
    'https://api4.example.com/data',
];

$results = processInParallel($urls, function (string $url): array {
    $response = file_get_contents($url);
    return json_decode($response, true);
}, maxProcesses: 4);
```

---

## ขั้นตอนที่ 1872: Parallel Extension

```php
<?php

declare(strict_types=1);

// ต้องติดตั้ง ext-parallel
// pecl install parallel

use parallel\Runtime;
use parallel\Future;
use parallel\Channel;
use parallel\Events;

// สร้าง parallel runtimes
function runParallelTasks(array $tasks): array
{
    $runtimes = [];
    $futures  = [];

    foreach ($tasks as $id => $task) {
        $runtime = new Runtime('/path/to/bootstrap.php');

        $futures[$id] = $runtime->run(function () use ($task): mixed {
            // โค้ดที่รันใน thread แยก
            return $task['callable'](...$task['args']);
        });

        $runtimes[$id] = $runtime;
    }

    // รวบรวมผลลัพธ์
    $results = [];
    foreach ($futures as $id => $future) {
        $results[$id] = $future->value(); // Blocking จนกว่าจะเสร็จ
    }

    return $results;
}

// Channel สำหรับ communication ระหว่าง threads
function processWithChannel(): void
{
    $channel = new Channel(Channel::Infinite);

    // Producer thread
    $producer = new Runtime();
    $producer->run(function (Channel $chan): void {
        for ($i = 0; $i < 10; $i++) {
            $chan->send(['item' => $i, 'data' => rand(1, 100)]);
            usleep(100000); // 100ms
        }
        $chan->send(null); // Signal done
    }, [$channel]);

    // Consumer in main thread
    $results = [];
    while (true) {
        $item = $channel->recv();
        if ($item === null) break;

        $results[] = $item['data'] * 2;
    }

    print_r($results);
}
```

---

## ขั้นตอนที่ 1873: ReactPHP Async

```bash
composer require react/event-loop react/http react/promise
```

```php
<?php

declare(strict_types=1);

// async_http_server.php
require 'vendor/autoload.php';

use React\EventLoop\Loop;
use React\Http\HttpServer;
use React\Http\Message\Response;
use Psr\Http\Message\ServerRequestInterface;
use React\Promise\Promise;

$server = new HttpServer(function (ServerRequestInterface $request): Promise {
    return new Promise(function ($resolve) use ($request) {
        // Simulate async work
        Loop::addTimer(0.1, function () use ($resolve, $request) {
            $data = [
                'path'    => $request->getUri()->getPath(),
                'method'  => $request->getMethod(),
                'time'    => microtime(true),
            ];

            $resolve(Response::json($data));
        });
    });
});

$socket = new React\Socket\SocketServer('0.0.0.0:8080');
$server->listen($socket);

echo "Server running at http://localhost:8080\n";

Loop::run();
```

---

## ขั้นตอนที่ 1874: Guzzle Concurrent HTTP Requests

```php
<?php

declare(strict_types=1);

// app/Services/ConcurrentHttpService.php
namespace App\Services;

use GuzzleHttp\Client;
use GuzzleHttp\Pool;
use GuzzleHttp\Promise\Utils;
use GuzzleHttp\Psr7\Request;
use Illuminate\Support\Collection;

class ConcurrentHttpService
{
    private Client $client;

    public function __construct()
    {
        $this->client = new Client([
            'timeout'         => 30,
            'connect_timeout' => 10,
        ]);
    }

    /**
     * ส่ง HTTP requests แบบ concurrent
     */
    public function fetchAll(array $urls): array
    {
        $promises = [];

        foreach ($urls as $key => $url) {
            $promises[$key] = $this->client->getAsync($url);
        }

        // รอทุก promises (settled = ทั้ง fulfilled และ rejected)
        $responses = Utils::settle($promises)->wait();

        $results = [];
        foreach ($responses as $key => $response) {
            if ($response['state'] === 'fulfilled') {
                $results[$key] = [
                    'status' => 'success',
                    'data'   => json_decode($response['value']->getBody(), true),
                    'code'   => $response['value']->getStatusCode(),
                ];
            } else {
                $results[$key] = [
                    'status' => 'error',
                    'error'  => $response['reason']->getMessage(),
                ];
            }
        }

        return $results;
    }

    /**
     * Pool requests - ควบคุม concurrency
     */
    public function poolRequests(array $urls, int $concurrency = 10): array
    {
        $results  = [];
        $requests = function (array $urls) {
            foreach ($urls as $key => $url) {
                yield $key => new Request('GET', $url);
            }
        };

        $pool = new Pool($this->client, $requests($urls), [
            'concurrency' => $concurrency,
            'fulfilled'   => function ($response, $key) use (&$results) {
                $results[$key] = [
                    'status' => $response->getStatusCode(),
                    'body'   => json_decode($response->getBody(), true),
                ];
            },
            'rejected'    => function ($reason, $key) use (&$results) {
                $results[$key] = [
                    'error' => $reason->getMessage(),
                ];
            },
        ]);

        $promise = $pool->promise();
        $promise->wait();

        return $results;
    }

    /**
     * เรียก multiple APIs พร้อมกัน
     */
    public function fetchDashboardData(int $userId): array
    {
        $promises = [
            'profile'       => $this->client->getAsync("/api/users/{$userId}"),
            'orders'        => $this->client->getAsync("/api/orders?user_id={$userId}"),
            'notifications' => $this->client->getAsync("/api/notifications/{$userId}"),
            'recommendations' => $this->client->getAsync("/api/recommendations/{$userId}"),
        ];

        $responses = Utils::all($promises)->wait();

        return array_map(
            fn ($r) => json_decode($r->getBody(), true),
            $responses
        );
    }
}
```

---

## ขั้นตอนที่ 1875: Amp Async PHP

```bash
composer require amphp/amp amphp/http-client
```

```php
<?php

declare(strict_types=1);

// async_amp.php
use Amp\Http\Client\HttpClientBuilder;
use Amp\Http\Client\Request;
use Amp\Future;
use function Amp\async;
use function Amp\await;

require 'vendor/autoload.php';

// Amp v3 style
function fetchAllUrls(array $urls): array
{
    $client = HttpClientBuilder::buildDefault();

    $futures = array_map(function (string $url) use ($client): Future {
        return async(function () use ($url, $client): array {
            $response = $client->request(new Request($url));
            $body     = $response->getBody()->buffer();

            return [
                'url'  => $url,
                'data' => json_decode($body, true),
                'code' => $response->getStatus(),
            ];
        });
    }, $urls);

    return Future\awaitAll($futures);
}

// Job Queue ด้วย Amp
function processJobQueue(array $jobs, int $concurrency = 5): void
{
    $semaphore = new \Amp\Sync\LocalSemaphore($concurrency);

    $futures = [];
    foreach ($jobs as $job) {
        $futures[] = async(function () use ($job, $semaphore): void {
            $lock = $semaphore->acquire();

            try {
                processJob($job);
            } finally {
                $lock->release();
            }
        });
    }

    Future\awaitAll($futures);
}

function processJob(array $job): void
{
    // Process job logic
    echo "Processing job: {$job['id']}\n";
}
```

---

## ขั้นตอนที่ 1876: Shared Memory กับ Semaphores

```php
<?php

declare(strict_types=1);

// Shared memory สำหรับ inter-process communication
class SharedCounter
{
    private \SysvSharedMemory $shm;
    private \SysvSemaphore $semaphore;
    private int $key;

    public function __construct(int $key)
    {
        $this->key       = $key;
        $this->shm       = shm_attach($key, 1024); // 1KB shared memory
        $this->semaphore = sem_get($key);           // Semaphore สำหรับ lock
    }

    public function increment(): int
    {
        sem_acquire($this->semaphore); // Lock

        try {
            $count = shm_has_var($this->shm, 1) ? shm_get_var($this->shm, 1) : 0;
            $count++;
            shm_put_var($this->shm, 1, $count);
            return $count;
        } finally {
            sem_release($this->semaphore); // Unlock
        }
    }

    public function get(): int
    {
        return shm_has_var($this->shm, 1) ? shm_get_var($this->shm, 1) : 0;
    }

    public function cleanup(): void
    {
        shm_remove($this->shm);
        sem_remove($this->semaphore);
    }
}

// ตัวอย่าง: multiple processes เพิ่ม counter พร้อมกัน
$counter = new SharedCounter(ftok(__FILE__, 'a'));

$pid = pcntl_fork();
if ($pid === 0) {
    // Child process
    for ($i = 0; $i < 1000; $i++) {
        $counter->increment();
    }
    exit(0);
} else {
    // Parent process
    for ($i = 0; $i < 1000; $i++) {
        $counter->increment();
    }

    pcntl_waitpid($pid, $status);
    echo "Final count: " . $counter->get() . "\n"; // Should be 2000
    $counter->cleanup();
}
```

---

## ขั้นตอนที่ 1877: Laravel Queue Worker Parallel

```php
<?php

declare(strict_types=1);

// config/queue.php
return [
    'connections' => [
        'redis' => [
            'driver'      => 'redis',
            'connection'  => 'default',
            'queue'       => ['default', 'emails', 'reports'],
            'retry_after' => 90,
            'block_for'   => null,
            'after_commit' => false,
        ],
    ],
];

// Horizon config for parallel workers
// config/horizon.php
return [
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'maxProcesses'  => 10,
                'balanceMaxShift' => 1,
                'balanceCooldown' => 3,
                'memory'        => 128,
                'tries'         => 3,
                'queues'        => ['emails'],
                'timeout'       => 60,
            ],
            'supervisor-2' => [
                'maxProcesses'  => 5,
                'queues'        => ['reports'],
                'timeout'       => 300,
            ],
            'supervisor-3' => [
                'maxProcesses'  => 20,
                'queues'        => ['default'],
                'timeout'       => 30,
            ],
        ],
    ],
];
```

```bash
# รัน multiple queue workers แบบ parallel
php artisan queue:work redis --queue=emails --workers=5 &
php artisan queue:work redis --queue=reports --workers=2 &
php artisan queue:work redis --queue=default --workers=10 &

# หรือใช้ Supervisor
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
numprocs=8
startsecs=0
stopwaitsecs=3600
```

---

## สรุป Part 67

| Approach | Use Case | Pros | Cons |
|---------|---------|------|------|
| pcntl_fork | CPU-intensive | Full isolation | Complex IPC |
| parallel ext | Multi-threading | Shared data | Experimental |
| ReactPHP | I/O-bound | Event loop | Single-threaded |
| Guzzle Concurrent | HTTP requests | Easy to use | PHP-only |
| Amp | Async PHP | Modern API | Learning curve |
| Queue Workers | Background jobs | Scalable | Eventually processed |

ถัดไป → Part 68: API Gateway Patterns
