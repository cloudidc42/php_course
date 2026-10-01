# Part 68: API Gateway

## ขั้นตอนที่ 1901-1930: API Gateway Patterns

API Gateway เป็น single entry point สำหรับ clients ทำหน้าที่ routing, authentication, rate limiting และ aggregation

---

## ขั้นตอนที่ 1901: สร้าง Simple API Gateway ใน Laravel

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/GatewayController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;
use Symfony\Component\HttpFoundation\Response;

class GatewayController extends Controller
{
    private array $serviceMap = [
        'users'    => 'http://user-service:8001',
        'products' => 'http://product-service:8002',
        'orders'   => 'http://order-service:8003',
        'payments' => 'http://payment-service:8004',
    ];

    public function route(Request $request, string $service, string $path = ''): JsonResponse
    {
        // Check ว่า service มีอยู่
        if (!isset($this->serviceMap[$service])) {
            return response()->json(['error' => 'Service not found'], 404);
        }

        $serviceUrl = $this->serviceMap[$service];
        $targetUrl  = "{$serviceUrl}/{$path}";

        // Forward request ไปยัง service
        $response = Http::withHeaders($this->buildForwardHeaders($request))
            ->withOptions(['timeout' => 30])
            ->{strtolower($request->method())}($targetUrl, $request->all());

        return response()->json(
            $response->json(),
            $response->status()
        );
    }

    private function buildForwardHeaders(Request $request): array
    {
        return [
            'X-Forwarded-For'   => $request->ip(),
            'X-Request-ID'      => $request->header('X-Request-ID', (string) \Str::uuid()),
            'X-User-ID'         => $request->user()?->id,
            'X-User-Role'       => $request->user()?->role,
            'Authorization'     => $request->header('Authorization'),
            'Accept'            => 'application/json',
            'Content-Type'      => 'application/json',
        ];
    }
}
```

---

## ขั้นตอนที่ 1902: Request Aggregation

```php
<?php

declare(strict_types=1);

// app/Services/RequestAggregator.php
namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;

class RequestAggregator
{
    /**
     * Aggregate หลาย API calls เป็น response เดียว
     * ลด roundtrips ของ client
     */
    public function getProductDetails(int $productId, int $userId): array
    {
        // ส่ง requests แบบ parallel
        $responses = Http::pool(fn ($pool) => [
            $pool->as('product')
                 ->get("http://product-service:8002/products/{$productId}"),

            $pool->as('reviews')
                 ->get("http://review-service:8005/reviews?product_id={$productId}&limit=5"),

            $pool->as('inventory')
                 ->get("http://inventory-service:8006/stock/{$productId}"),

            $pool->as('user_history')
                 ->get("http://user-service:8001/users/{$userId}/history?product_id={$productId}"),

            $pool->as('recommendations')
                 ->get("http://recommendation-service:8007/similar/{$productId}?limit=4"),
        ]);

        // รวม responses
        $product = $responses['product']->ok() ? $responses['product']->json() : null;

        if (!$product) {
            throw new \RuntimeException('Product not found');
        }

        return [
            'product'         => $product,
            'reviews'         => $responses['reviews']->ok() ? $responses['reviews']->json() : [],
            'stock'           => $responses['inventory']->ok() ? $responses['inventory']->json() : null,
            'user_purchased'  => $responses['user_history']->ok()
                ? !empty($responses['user_history']->json())
                : false,
            'recommendations' => $responses['recommendations']->ok()
                ? $responses['recommendations']->json()
                : [],
        ];
    }

    /**
     * Batch requests
     */
    public function batchGetProducts(array $productIds): array
    {
        $cacheKey = 'products:batch:' . md5(implode(',', sort($productIds)));

        return Cache::remember($cacheKey, 300, function () use ($productIds) {
            $responses = Http::pool(fn ($pool) => collect($productIds)->map(
                fn ($id) => $pool->as("product_{$id}")
                                 ->get("http://product-service:8002/products/{$id}")
            )->toArray());

            return collect($productIds)
                ->mapWithKeys(fn ($id) => [
                    $id => $responses["product_{$id}"]->ok()
                        ? $responses["product_{$id}"]->json()
                        : null,
                ])
                ->filter()
                ->toArray();
        });
    }
}
```

---

## ขั้นตอนที่ 1903: Rate Limiting Middleware

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/GatewayRateLimiter.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Symfony\Component\HttpFoundation\Response;

class GatewayRateLimiter
{
    private array $limits = [
        'free'       => ['requests' => 100, 'minutes' => 60],
        'pro'        => ['requests' => 1000, 'minutes' => 60],
        'enterprise' => ['requests' => 10000, 'minutes' => 60],
        'anonymous'  => ['requests' => 30, 'minutes' => 60],
    ];

    public function handle(Request $request, Closure $next, string $tier = 'anonymous'): Response
    {
        $user     = $request->user();
        $plan     = $user?->plan ?? 'anonymous';
        $limit    = $this->limits[$plan] ?? $this->limits['anonymous'];

        // Key unique per user or IP
        $key = $user ? "rate_limit:user:{$user->id}" : "rate_limit:ip:{$request->ip()}";

        if (RateLimiter::tooManyAttempts($key, $limit['requests'])) {
            $retryAfter = RateLimiter::availableIn($key);

            return response()->json([
                'error'       => 'Too Many Requests',
                'retry_after' => $retryAfter,
            ], 429)
                ->withHeaders([
                    'Retry-After'               => $retryAfter,
                    'X-RateLimit-Limit'         => $limit['requests'],
                    'X-RateLimit-Remaining'     => 0,
                    'X-RateLimit-Reset'         => now()->addSeconds($retryAfter)->timestamp,
                ]);
        }

        RateLimiter::hit($key, $limit['minutes'] * 60);

        $response = $next($request);

        // Add rate limit headers
        $remaining = RateLimiter::remaining($key, $limit['requests']);

        return $response->withHeaders([
            'X-RateLimit-Limit'     => $limit['requests'],
            'X-RateLimit-Remaining' => $remaining,
            'X-RateLimit-Reset'     => RateLimiter::availableIn($key) + time(),
        ]);
    }
}
```

---

## ขั้นตอนที่ 1904: Authentication at Gateway Level

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/GatewayAuth.php
namespace App\Http\Middleware;

use App\Services\TokenService;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class GatewayAuth
{
    public function __construct(private TokenService $tokenService) {}

    public function handle(Request $request, Closure $next, string ...$guards): Response
    {
        $token = $this->extractToken($request);

        if (!$token) {
            return response()->json(['error' => 'Unauthorized'], 401);
        }

        try {
            $payload = $this->tokenService->verify($token);

            // เพิ่ม user info เข้า request headers
            // เพื่อส่งต่อไปยัง downstream services
            $request->headers->set('X-User-ID', $payload['sub']);
            $request->headers->set('X-User-Email', $payload['email']);
            $request->headers->set('X-User-Role', $payload['role']);
            $request->headers->set('X-Token-Exp', $payload['exp']);

            // Set authenticated user
            auth()->setUser($this->getUserFromPayload($payload));

        } catch (\Exception $e) {
            return response()->json([
                'error'   => 'Invalid or expired token',
                'message' => $e->getMessage(),
            ], 401);
        }

        return $next($request);
    }

    private function extractToken(Request $request): ?string
    {
        // Bearer token
        if ($bearerToken = $request->bearerToken()) {
            return $bearerToken;
        }

        // API key
        if ($apiKey = $request->header('X-API-Key')) {
            return $this->tokenService->apiKeyToJwt($apiKey);
        }

        // Query parameter (for webhooks)
        if ($request->has('access_token')) {
            return $request->get('access_token');
        }

        return null;
    }

    private function getUserFromPayload(array $payload): \App\Models\User
    {
        return new \App\Models\User([
            'id'    => $payload['sub'],
            'email' => $payload['email'],
            'role'  => $payload['role'],
        ]);
    }
}
```

---

## ขั้นตอนที่ 1905: Response Caching at Gateway

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/GatewayCache.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Illuminate\Support\Facades\Cache;
use Symfony\Component\HttpFoundation\Response;

class GatewayCache
{
    private array $cacheableRoutes = [
        ['pattern' => '/api/products', 'ttl' => 300],
        ['pattern' => '/api/categories', 'ttl' => 3600],
        ['pattern' => '/api/config', 'ttl' => 86400],
    ];

    public function handle(Request $request, Closure $next): Response
    {
        // Only cache GET requests
        if (!$request->isMethod('GET')) {
            return $next($request);
        }

        $ttl = $this->getCacheTtl($request->getPathInfo());

        if ($ttl === null) {
            return $next($request);
        }

        $cacheKey = $this->buildCacheKey($request);

        // ตรวจสอบ cache
        $cached = Cache::get($cacheKey);
        if ($cached) {
            return response()
                ->json($cached['data'], $cached['status'])
                ->withHeaders(array_merge($cached['headers'], [
                    'X-Cache'     => 'HIT',
                    'X-Cache-Key' => $cacheKey,
                ]));
        }

        // Execute request
        $response = $next($request);

        // Cache ถ้า successful
        if ($response->isSuccessful()) {
            Cache::put($cacheKey, [
                'data'    => json_decode($response->getContent(), true),
                'status'  => $response->getStatusCode(),
                'headers' => $this->filterHeaders($response->headers->all()),
            ], $ttl);
        }

        return $response->withHeaders([
            'X-Cache'     => 'MISS',
            'X-Cache-TTL' => $ttl,
        ]);
    }

    private function getCacheTtl(string $path): ?int
    {
        foreach ($this->cacheableRoutes as $route) {
            if (str_starts_with($path, $route['pattern'])) {
                return $route['ttl'];
            }
        }
        return null;
    }

    private function buildCacheKey(Request $request): string
    {
        return 'gateway:' . md5(
            $request->getMethod() .
            $request->getPathInfo() .
            http_build_query($request->query->all()) .
            ($request->header('Accept-Language') ?? '') .
            ($request->header('X-Tenant-ID') ?? '')
        );
    }

    private function filterHeaders(array $headers): array
    {
        $keepHeaders = ['content-type', 'x-request-id'];
        return array_intersect_key($headers, array_flip($keepHeaders));
    }
}
```

---

## ขั้นตอนที่ 1906: Circuit Breaker Pattern

```php
<?php

declare(strict_types=1);

// app/Services/CircuitBreaker.php
namespace App\Services;

use Illuminate\Support\Facades\Cache;

class CircuitBreaker
{
    private const STATE_CLOSED   = 'closed';   // Normal operation
    private const STATE_OPEN     = 'open';     // Blocking requests
    private const STATE_HALF_OPEN = 'half_open'; // Testing

    public function __construct(
        private string $service,
        private int    $failureThreshold = 5,
        private int    $successThreshold = 2,
        private int    $timeout          = 60,  // seconds
    ) {}

    public function call(callable $fn): mixed
    {
        $state = $this->getState();

        if ($state === self::STATE_OPEN) {
            throw new \RuntimeException("Circuit breaker OPEN for service: {$this->service}");
        }

        try {
            $result = $fn();
            $this->onSuccess();
            return $result;
        } catch (\Throwable $e) {
            $this->onFailure();
            throw $e;
        }
    }

    private function onSuccess(): void
    {
        $state = $this->getState();

        if ($state === self::STATE_HALF_OPEN) {
            $this->incrementSuccesses();

            if ($this->getSuccesses() >= $this->successThreshold) {
                $this->closeCircuit();
            }
        }

        // Reset failures
        Cache::forget("circuit:{$this->service}:failures");
    }

    private function onFailure(): void
    {
        $failures = $this->incrementFailures();

        if ($failures >= $this->failureThreshold) {
            $this->openCircuit();
        }
    }

    private function openCircuit(): void
    {
        Cache::put("circuit:{$this->service}:state", self::STATE_OPEN, $this->timeout);
        Cache::put("circuit:{$this->service}:opened_at", time(), $this->timeout);

        \Log::warning("Circuit breaker OPENED for: {$this->service}");
    }

    private function closeCircuit(): void
    {
        Cache::forget("circuit:{$this->service}:state");
        Cache::forget("circuit:{$this->service}:failures");
        Cache::forget("circuit:{$this->service}:successes");

        \Log::info("Circuit breaker CLOSED for: {$this->service}");
    }

    private function getState(): string
    {
        $state    = Cache::get("circuit:{$this->service}:state", self::STATE_CLOSED);
        $openedAt = Cache::get("circuit:{$this->service}:opened_at");

        // Transition from OPEN to HALF_OPEN after timeout
        if ($state === self::STATE_OPEN && $openedAt && (time() - $openedAt) >= $this->timeout) {
            Cache::put("circuit:{$this->service}:state", self::STATE_HALF_OPEN, $this->timeout);
            return self::STATE_HALF_OPEN;
        }

        return $state;
    }

    private function incrementFailures(): int
    {
        return Cache::increment("circuit:{$this->service}:failures", 1);
    }

    private function incrementSuccesses(): int
    {
        return Cache::increment("circuit:{$this->service}:successes", 1);
    }

    private function getSuccesses(): int
    {
        return (int) Cache::get("circuit:{$this->service}:successes", 0);
    }
}
```

---

## สรุป Part 68

| Pattern | รายละเอียด |
|---------|-----------|
| Request Routing | Forward ไปยัง services |
| Request Aggregation | รวม multiple API calls |
| Rate Limiting | ควบคุม requests per user/IP |
| Auth at Gateway | Single auth point |
| Response Caching | Cache ที่ gateway layer |
| Circuit Breaker | ป้องกัน cascade failures |
| Load Balancing | กระจาย traffic |

ถัดไป → Part 69: Database Scaling
