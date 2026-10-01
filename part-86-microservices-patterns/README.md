# Part 86: Microservices Patterns
## ขั้นตอนที่ 2441-2470: รูปแบบสถาปัตยกรรม Microservices

Circuit Breaker, Service Discovery, Bulkhead Pattern,
Retry/Timeout และ Distributed Tracing

---

## ขั้นตอนที่ 2441: Circuit Breaker Pattern

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\CircuitBreaker;

use Illuminate\Support\Facades\Cache;

enum CircuitState: string
{
    case CLOSED = 'closed';       // ปกติ
    case OPEN = 'open';           // ปิดกั้น request
    case HALF_OPEN = 'half_open'; // ทดสอบ
}

class CircuitBreaker
{
    private string $cacheKey;
    private int $failureThreshold;
    private int $successThreshold;
    private int $timeout; // seconds

    public function __construct(
        private readonly string $name,
        array $config = []
    ) {
        $this->cacheKey = "circuit_breaker:{$name}";
        $this->failureThreshold = $config['failure_threshold'] ?? 5;
        $this->successThreshold = $config['success_threshold'] ?? 2;
        $this->timeout = $config['timeout'] ?? 60;
    }

    public function call(callable $command, callable $fallback = null): mixed
    {
        $state = $this->getState();

        if ($state === CircuitState::OPEN) {
            // ตรวจสอบว่าควรเปลี่ยนเป็น HALF_OPEN
            if ($this->shouldAttemptReset()) {
                $this->setState(CircuitState::HALF_OPEN);
            } else {
                if ($fallback) {
                    return $fallback();
                }
                throw new CircuitBreakerOpenException("Circuit breaker '{$this->name}' is OPEN");
            }
        }

        try {
            $result = $command();
            $this->recordSuccess();
            return $result;
        } catch (\Throwable $e) {
            $this->recordFailure();
            if ($fallback) {
                return $fallback();
            }
            throw $e;
        }
    }

    private function getState(): CircuitState
    {
        $data = Cache::get($this->cacheKey, ['state' => CircuitState::CLOSED->value]);
        return CircuitState::from($data['state']);
    }

    private function setState(CircuitState $state): void
    {
        $data = Cache::get($this->cacheKey, []);
        $data['state'] = $state->value;
        $data['state_changed_at'] = now()->timestamp;
        Cache::put($this->cacheKey, $data, now()->addHour());
    }

    private function recordFailure(): void
    {
        $data = Cache::get($this->cacheKey, [
            'state' => CircuitState::CLOSED->value,
            'failures' => 0,
            'successes' => 0,
        ]);

        $data['failures'] = ($data['failures'] ?? 0) + 1;
        $data['last_failure_at'] = now()->timestamp;

        if ($data['failures'] >= $this->failureThreshold) {
            $data['state'] = CircuitState::OPEN->value;
            $data['opened_at'] = now()->timestamp;
        }

        Cache::put($this->cacheKey, $data, now()->addHour());
    }

    private function recordSuccess(): void
    {
        $data = Cache::get($this->cacheKey, []);
        $state = CircuitState::from($data['state'] ?? CircuitState::CLOSED->value);

        if ($state === CircuitState::HALF_OPEN) {
            $data['successes'] = ($data['successes'] ?? 0) + 1;

            if ($data['successes'] >= $this->successThreshold) {
                $data['state'] = CircuitState::CLOSED->value;
                $data['failures'] = 0;
                $data['successes'] = 0;
            }
        } else {
            $data['failures'] = 0;
        }

        Cache::put($this->cacheKey, $data, now()->addHour());
    }

    private function shouldAttemptReset(): bool
    {
        $data = Cache::get($this->cacheKey, []);
        $openedAt = $data['opened_at'] ?? 0;
        return (now()->timestamp - $openedAt) >= $this->timeout;
    }

    public function getMetrics(): array
    {
        $data = Cache::get($this->cacheKey, []);
        return [
            'name' => $this->name,
            'state' => $data['state'] ?? CircuitState::CLOSED->value,
            'failures' => $data['failures'] ?? 0,
            'last_failure_at' => isset($data['last_failure_at'])
                ? date('Y-m-d H:i:s', $data['last_failure_at'])
                : null,
        ];
    }
}

class CircuitBreakerOpenException extends \RuntimeException {}
```

---

## ขั้นตอนที่ 2442: Service Discovery Client

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\ServiceDiscovery;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Http;

class ConsulServiceDiscovery
{
    private string $consulUrl;

    public function __construct()
    {
        $this->consulUrl = config('services.consul.url', 'http://localhost:8500');
    }

    public function resolve(string $serviceName): string
    {
        $instances = $this->getHealthyInstances($serviceName);

        if (empty($instances)) {
            throw new \RuntimeException("No healthy instances found for service: {$serviceName}");
        }

        // Load balancing - Round Robin
        $instance = $instances[array_rand($instances)];

        return sprintf(
            'http://%s:%d',
            $instance['Service']['Address'],
            $instance['Service']['Port']
        );
    }

    private function getHealthyInstances(string $serviceName): array
    {
        return Cache::remember(
            "consul:service:{$serviceName}",
            30, // cache for 30 seconds
            function () use ($serviceName) {
                $response = Http::get("{$this->consulUrl}/v1/health/service/{$serviceName}", [
                    'passing' => true,
                ]);

                if (!$response->successful()) {
                    return [];
                }

                return $response->json();
            }
        );
    }

    public function register(array $serviceConfig): bool
    {
        $response = Http::put("{$this->consulUrl}/v1/agent/service/register", $serviceConfig);
        return $response->successful();
    }

    public function deregister(string $serviceId): bool
    {
        $response = Http::put("{$this->consulUrl}/v1/agent/service/deregister/{$serviceId}");
        return $response->successful();
    }
}
```

---

## ขั้นตอนที่ 2443: Retry Pattern

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Resilience;

class RetryPolicy
{
    public function __construct(
        private readonly int $maxAttempts = 3,
        private readonly int $baseDelay = 100, // milliseconds
        private readonly float $multiplier = 2.0,
        private readonly int $maxDelay = 5000,
        private readonly array $retryOn = [\Exception::class],
    ) {}

    public function execute(callable $operation): mixed
    {
        $attempt = 0;
        $lastException = null;

        while ($attempt < $this->maxAttempts) {
            try {
                return $operation();
            } catch (\Throwable $e) {
                $lastException = $e;

                if (!$this->shouldRetry($e, $attempt)) {
                    throw $e;
                }

                $attempt++;

                if ($attempt < $this->maxAttempts) {
                    $delay = $this->calculateDelay($attempt);
                    usleep($delay * 1000); // convert to microseconds
                }
            }
        }

        throw new \RuntimeException(
            "Operation failed after {$this->maxAttempts} attempts",
            0,
            $lastException
        );
    }

    private function shouldRetry(\Throwable $e, int $attempt): bool
    {
        foreach ($this->retryOn as $exceptionClass) {
            if ($e instanceof $exceptionClass) {
                return $attempt < $this->maxAttempts - 1;
            }
        }
        return false;
    }

    private function calculateDelay(int $attempt): int
    {
        // Exponential backoff with jitter
        $exponential = $this->baseDelay * pow($this->multiplier, $attempt - 1);
        $jitter = rand(0, (int)($exponential * 0.1));
        return (int)min($exponential + $jitter, $this->maxDelay);
    }
}
```

---

## ขั้นตอนที่ 2444: Bulkhead Pattern

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Resilience;

use Illuminate\Support\Facades\Cache;

class Bulkhead
{
    private string $semaphoreKey;

    public function __construct(
        private readonly string $name,
        private readonly int $maxConcurrent = 10,
        private readonly int $timeout = 5000, // milliseconds
    ) {
        $this->semaphoreKey = "bulkhead:{$name}";
    }

    public function call(callable $operation): mixed
    {
        if (!$this->acquire()) {
            throw new BulkheadFullException(
                "Bulkhead '{$this->name}' is full ({$this->maxConcurrent} concurrent executions)"
            );
        }

        try {
            return $operation();
        } finally {
            $this->release();
        }
    }

    private function acquire(): bool
    {
        $acquired = false;
        $startTime = microtime(true);
        $timeoutSeconds = $this->timeout / 1000;

        while (!$acquired) {
            $count = Cache::get($this->semaphoreKey, 0);

            if ($count < $this->maxConcurrent) {
                // Atomic increment
                Cache::increment($this->semaphoreKey);
                Cache::add($this->semaphoreKey, 0, 60);
                $acquired = true;
            } else {
                if ((microtime(true) - $startTime) >= $timeoutSeconds) {
                    return false;
                }
                usleep(10000); // 10ms
            }
        }

        return $acquired;
    }

    private function release(): void
    {
        $count = Cache::get($this->semaphoreKey, 1);
        if ($count > 0) {
            Cache::decrement($this->semaphoreKey);
        }
    }

    public function getStats(): array
    {
        $current = Cache::get($this->semaphoreKey, 0);
        return [
            'name' => $this->name,
            'max_concurrent' => $this->maxConcurrent,
            'current_concurrent' => $current,
            'available' => $this->maxConcurrent - $current,
        ];
    }
}

class BulkheadFullException extends \RuntimeException {}
```

---

## ขั้นตอนที่ 2445: Distributed Tracing

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Tracing;

use Illuminate\Support\Str;

class DistributedTracer
{
    private static ?string $traceId = null;
    private static array $spans = [];

    public static function startTrace(string $traceId = null): string
    {
        self::$traceId = $traceId ?? Str::uuid()->toString();
        return self::$traceId;
    }

    public static function getTraceId(): ?string
    {
        return self::$traceId;
    }

    public static function startSpan(string $name, array $tags = []): string
    {
        $spanId = Str::uuid()->toString();
        self::$spans[$spanId] = [
            'span_id' => $spanId,
            'trace_id' => self::$traceId,
            'name' => $name,
            'tags' => $tags,
            'start_time' => microtime(true),
            'logs' => [],
        ];
        return $spanId;
    }

    public static function finishSpan(string $spanId, array $tags = []): void
    {
        if (!isset(self::$spans[$spanId])) return;

        $span = &self::$spans[$spanId];
        $span['end_time'] = microtime(true);
        $span['duration_ms'] = ($span['end_time'] - $span['start_time']) * 1000;
        $span['tags'] = array_merge($span['tags'], $tags);

        // ส่ง span ไปยัง Jaeger/Zipkin
        self::exportSpan($span);
    }

    public static function log(string $spanId, string $message, array $fields = []): void
    {
        if (!isset(self::$spans[$spanId])) return;

        self::$spans[$spanId]['logs'][] = [
            'timestamp' => microtime(true),
            'message' => $message,
            'fields' => $fields,
        ];
    }

    private static function exportSpan(array $span): void
    {
        // ส่งไปยัง Jaeger collector
        $jaegerEndpoint = config('services.jaeger.endpoint');
        if (!$jaegerEndpoint) return;

        try {
            app('http')->post($jaegerEndpoint . '/api/traces', [
                'json' => [
                    'spans' => [$span],
                ],
                'timeout' => 1,
            ]);
        } catch (\Exception $e) {
            // ไม่ให้ tracing ทำให้ application หยุดทำงาน
        }
    }
}
```

---

## ขั้นตอนที่ 2446: HTTP Client กับ Resilience

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Http;

use App\Infrastructure\CircuitBreaker\CircuitBreaker;
use App\Infrastructure\Resilience\RetryPolicy;
use Illuminate\Http\Client\Response;
use Illuminate\Support\Facades\Http;

class ResilientHttpClient
{
    private CircuitBreaker $circuitBreaker;
    private RetryPolicy $retryPolicy;

    public function __construct(string $serviceName, array $config = [])
    {
        $this->circuitBreaker = new CircuitBreaker($serviceName, $config['circuit_breaker'] ?? []);
        $this->retryPolicy = new RetryPolicy(
            maxAttempts: $config['retry']['max_attempts'] ?? 3,
            baseDelay: $config['retry']['base_delay'] ?? 100,
        );
    }

    public function get(string $url, array $options = []): Response
    {
        return $this->circuitBreaker->call(
            fn () => $this->retryPolicy->execute(
                fn () => Http::timeout($options['timeout'] ?? 10)->get($url, $options)
            ),
            fn () => throw new \RuntimeException("Service unavailable: {$url}")
        );
    }

    public function post(string $url, array $data = [], array $options = []): Response
    {
        return $this->circuitBreaker->call(
            fn () => $this->retryPolicy->execute(
                fn () => Http::timeout($options['timeout'] ?? 10)->post($url, $data)
            )
        );
    }
}

// ใช้งาน
$client = new ResilientHttpClient('payment-service', [
    'circuit_breaker' => [
        'failure_threshold' => 5,
        'timeout' => 30,
    ],
    'retry' => [
        'max_attempts' => 3,
        'base_delay' => 200,
    ],
]);

try {
    $response = $client->get('http://payment-service/api/status');
} catch (\RuntimeException $e) {
    // Fallback behavior
    logger()->error('Payment service unavailable', ['error' => $e->getMessage()]);
}
```

---

## สรุปบทที่ 86

| Pattern | ประโยชน์ | เมื่อใช้ |
|---------|---------|---------|
| Circuit Breaker | ป้องกัน cascade failure | External API calls |
| Service Discovery | Dynamic routing | Multiple instances |
| Retry Policy | จัดการ transient failures | Unreliable services |
| Bulkhead | Isolate resources | Concurrent requests |
| Distributed Tracing | Debug across services | Production debugging |
| Rate Limiting | ป้องกัน overload | Public APIs |

ถัดไป → Part 87: Container Orchestration กับ Kubernetes
