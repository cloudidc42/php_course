# Part 58: Advanced Caching Strategies

## ขั้นตอนที่ 1601-1630: กลยุทธ์การ Cache ขั้นสูง

Caching ที่ดีไม่ใช่แค่การ cache ทุกอย่าง แต่ต้องเลือก strategy ที่เหมาะสม บริหาร invalidation อย่างชาญฉลาด

---

## ขั้นตอนที่ 1601: Cache-Aside Pattern

```php
<?php

declare(strict_types=1);

// Cache-aside (Lazy loading) - Pattern ที่ใช้บ่อยที่สุด
namespace App\Repositories;

use App\Models\Product;
use Illuminate\Support\Facades\Cache;
use Illuminate\Database\Eloquent\Collection;

class ProductRepository
{
    private const CACHE_TTL = 3600; // 1 hour
    private const CACHE_PREFIX = 'products:';

    public function findById(int $id): ?Product
    {
        $cacheKey = self::CACHE_PREFIX . $id;

        // 1. ลองดึงจาก cache ก่อน
        return Cache::remember($cacheKey, self::CACHE_TTL, function () use ($id) {
            // 2. ถ้าไม่มี ดึงจาก database แล้ว cache ไว้
            return Product::with(['category', 'brand', 'images'])->find($id);
        });
    }

    public function getByCategory(string $category, int $page = 1): array
    {
        $cacheKey = self::CACHE_PREFIX . "category:{$category}:page:{$page}";

        return Cache::remember($cacheKey, self::CACHE_TTL, function () use ($category, $page) {
            return Product::where('category', $category)
                ->where('status', 'active')
                ->orderBy('created_at', 'desc')
                ->paginate(20, ['*'], 'page', $page)
                ->toArray();
        });
    }

    public function update(Product $product, array $data): Product
    {
        $product->update($data);

        // 3. Invalidate cache หลัง update
        $this->invalidateProduct($product->id);
        $this->invalidateCategoryCache($product->category);

        return $product->fresh();
    }

    public function invalidateProduct(int $id): void
    {
        Cache::forget(self::CACHE_PREFIX . $id);
    }

    private function invalidateCategoryCache(string $category): void
    {
        // ลบ cache ทุก page ของ category นี้
        for ($page = 1; $page <= 100; $page++) {
            $deleted = Cache::forget(self::CACHE_PREFIX . "category:{$category}:page:{$page}");
            if (!$deleted) {
                break; // ถ้าไม่มีแล้วหยุด
            }
        }
    }
}
```

---

## ขั้นตอนที่ 1602: Write-Through Pattern

```php
<?php

declare(strict_types=1);

// Write-through - เขียน cache พร้อมกับ database เสมอ
namespace App\Services;

use App\Models\User;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;

class UserProfileService
{
    private const CACHE_TTL = 7200;

    public function updateProfile(User $user, array $data): User
    {
        // เขียน database และ cache พร้อมกัน
        DB::transaction(function () use ($user, $data) {
            $user->update($data);

            // Update cache ทันทีหลัง database update สำเร็จ
            Cache::put(
                "user:profile:{$user->id}",
                $user->fresh()->toArray(),
                self::CACHE_TTL
            );
        });

        return $user->fresh();
    }

    public function getProfile(int $userId): ?array
    {
        return Cache::get("user:profile:{$userId}");
    }
}
```

---

## ขั้นตอนที่ 1603: Write-Behind (Write-Back) Pattern

```php
<?php

declare(strict_types=1);

// Write-behind - เขียน cache ก่อน แล้วค่อย async เขียน database
namespace App\Services;

use App\Jobs\PersistCounterToDatabase;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

class ViewCounterService
{
    public function increment(string $contentType, int $contentId): int
    {
        $cacheKey = "views:{$contentType}:{$contentId}";
        $dirty    = "views:dirty:{$contentType}:{$contentId}";

        // เขียน Redis ก่อน (เร็วมาก)
        $newCount = Redis::incr($cacheKey);

        // Mark ว่า dirty (ยังไม่ได้ flush ไป DB)
        Redis::sadd("views:dirty_set", "{$contentType}:{$contentId}");

        // Schedule background job ถ้ายังไม่มี
        if (Redis::setnx($dirty, '1')) {
            Redis::expire($dirty, 300); // 5 minutes
            PersistCounterToDatabase::dispatch($contentType, $contentId)
                ->delay(now()->addMinutes(5))
                ->onQueue('low');
        }

        return $newCount;
    }

    public function getCount(string $contentType, int $contentId): int
    {
        $cacheKey = "views:{$contentType}:{$contentId}";

        // ลองดึงจาก Redis ก่อน
        $cached = Redis::get($cacheKey);
        if ($cached !== null) {
            return (int) $cached;
        }

        // Fallback to database
        $count = \App\Models\ViewCounter::where([
            'content_type' => $contentType,
            'content_id'   => $contentId,
        ])->value('count') ?? 0;

        Redis::setex($cacheKey, 3600, $count);

        return $count;
    }
}
```

---

## ขั้นตอนที่ 1604: Cache Tags กับ Invalidation Groups

```php
<?php

declare(strict_types=1);

// Cache Tags สำหรับ group invalidation
namespace App\Services;

use App\Models\Post;
use Illuminate\Support\Facades\Cache;

class PostCacheService
{
    public function getPost(int $id): ?Post
    {
        return Cache::tags(['posts', "post:{$id}"])
            ->remember("post:{$id}", 3600, fn () => Post::with('author', 'comments')->find($id));
    }

    public function getPostsByAuthor(int $authorId): array
    {
        return Cache::tags(['posts', "author:{$authorId}"])
            ->remember("author:{$authorId}:posts", 3600, fn () =>
                Post::where('author_id', $authorId)->get()->toArray()
            );
    }

    public function invalidatePost(int $postId): void
    {
        // ลบทุก cache ที่ tag ด้วย "post:{id}" นี้
        Cache::tags(["post:{$postId}"])->flush();
    }

    public function invalidateAuthorPosts(int $authorId): void
    {
        // ลบทุก cache ของ author นี้
        Cache::tags(["author:{$authorId}"])->flush();
    }

    public function invalidateAllPosts(): void
    {
        // ลบทุก cache ที่เกี่ยวกับ posts
        Cache::tags(['posts'])->flush();
    }
}
```

---

## ขั้นตอนที่ 1605: Redis Cluster สำหรับ Distributed Caching

```php
<?php

declare(strict_types=1);

// config/database.php - Redis Cluster configuration
return [
    'redis' => [
        'client'  => env('REDIS_CLIENT', 'phpredis'),
        'options' => [
            'cluster'    => env('REDIS_CLUSTER', 'redis'),
            'prefix'     => env('REDIS_PREFIX', 'app_'),
        ],
        'clusters' => [
            'default' => [
                [
                    'host'     => env('REDIS_HOST_1', '127.0.0.1'),
                    'password' => env('REDIS_PASSWORD', null),
                    'port'     => env('REDIS_PORT_1', 7000),
                    'database' => 0,
                ],
                [
                    'host'     => env('REDIS_HOST_2', '127.0.0.1'),
                    'password' => env('REDIS_PASSWORD', null),
                    'port'     => env('REDIS_PORT_2', 7001),
                    'database' => 0,
                ],
                [
                    'host'     => env('REDIS_HOST_3', '127.0.0.1'),
                    'password' => env('REDIS_PASSWORD', null),
                    'port'     => env('REDIS_PORT_3', 7002),
                    'database' => 0,
                ],
            ],
            'cache' => [
                [
                    'host'     => env('REDIS_HOST_1', '127.0.0.1'),
                    'password' => env('REDIS_PASSWORD', null),
                    'port'     => env('REDIS_PORT_1', 7000),
                    'database' => 1,
                ],
            ],
        ],
    ],
];
```

```php
<?php

declare(strict_types=1);

// app/Services/DistributedLockService.php
namespace App\Services;

use Illuminate\Support\Facades\Redis;
use RuntimeException;

class DistributedLockService
{
    private const LOCK_PREFIX = 'lock:';
    private const DEFAULT_TTL = 30; // seconds

    public function acquire(string $resource, int $ttl = self::DEFAULT_TTL): ?string
    {
        $lockKey = self::LOCK_PREFIX . $resource;
        $token   = bin2hex(random_bytes(16));

        // SET NX EX - atomic operation
        $acquired = Redis::set($lockKey, $token, 'EX', $ttl, 'NX');

        return $acquired ? $token : null;
    }

    public function release(string $resource, string $token): bool
    {
        $lockKey = self::LOCK_PREFIX . $resource;

        // Lua script เพื่อ atomic check-and-delete
        $script = <<<LUA
            if redis.call("get", KEYS[1]) == ARGV[1] then
                return redis.call("del", KEYS[1])
            else
                return 0
            end
        LUA;

        return (bool) Redis::eval($script, 1, $lockKey, $token);
    }

    public function withLock(string $resource, callable $callback, int $ttl = self::DEFAULT_TTL): mixed
    {
        $token = $this->acquire($resource, $ttl);

        if ($token === null) {
            throw new RuntimeException("Could not acquire lock for: {$resource}");
        }

        try {
            return $callback();
        } finally {
            $this->release($resource, $token);
        }
    }
}
```

---

## ขั้นตอนที่ 1606: HTTP Caching กับ ETags

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/ApiController.php
namespace App\Http\Controllers;

use App\Models\Product;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpFoundation\Response;

class ProductApiController extends Controller
{
    public function show(Request $request, int $id): JsonResponse
    {
        $product = Product::findOrFail($id);

        // สร้าง ETag จาก updated_at + id
        $etag    = md5($product->id . $product->updated_at->timestamp);
        $lastMod = $product->updated_at->toRfc7231String();

        // Check If-None-Match header
        if ($request->header('If-None-Match') === "\"{$etag}\"") {
            return response()->json(null, Response::HTTP_NOT_MODIFIED)
                ->header('ETag', "\"{$etag}\"")
                ->header('Last-Modified', $lastMod);
        }

        // Check If-Modified-Since header
        $ifModSince = $request->header('If-Modified-Since');
        if ($ifModSince && strtotime($ifModSince) >= $product->updated_at->timestamp) {
            return response()->json(null, Response::HTTP_NOT_MODIFIED);
        }

        return response()->json($product)
            ->header('ETag', "\"{$etag}\"")
            ->header('Last-Modified', $lastMod)
            ->header('Cache-Control', 'public, max-age=300, must-revalidate')
            ->header('Vary', 'Accept-Encoding');
    }

    public function index(Request $request): JsonResponse
    {
        $products = Product::active()->orderBy('updated_at', 'desc')->get();

        // ETag สำหรับ collection
        $latestUpdate = $products->max('updated_at');
        $etag = md5($latestUpdate . $products->count());

        if ($request->header('If-None-Match') === "\"{$etag}\"") {
            return response()->json(null, 304);
        }

        return response()->json($products)
            ->header('ETag', "\"{$etag}\"")
            ->header('Cache-Control', 'public, max-age=60, stale-while-revalidate=30');
    }
}
```

---

## ขั้นตอนที่ 1607: Cache Warming Strategies

```php
<?php

declare(strict_types=1);

// app/Console/Commands/WarmCache.php
namespace App\Console\Commands;

use App\Models\Category;
use App\Models\Product;
use App\Services\ProductCacheService;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\Cache;

class WarmCache extends Command
{
    protected $signature = 'cache:warm {--type=all : Which cache to warm (all/products/categories)}';
    protected $description = 'Warm up application caches';

    public function __construct(private ProductCacheService $cacheService)
    {
        parent::__construct();
    }

    public function handle(): int
    {
        $type = $this->option('type');
        $this->info("Warming cache: {$type}");

        match ($type) {
            'products'   => $this->warmProducts(),
            'categories' => $this->warmCategories(),
            default      => $this->warmAll(),
        };

        $this->info('Cache warming complete!');
        return self::SUCCESS;
    }

    private function warmProducts(): void
    {
        $bar = $this->output->createProgressBar(Product::count());
        $bar->start();

        Product::with(['category', 'images', 'brand'])
            ->chunk(100, function ($products) use ($bar) {
                foreach ($products as $product) {
                    Cache::put(
                        "product:{$product->id}",
                        $product->toArray(),
                        3600
                    );
                    $bar->advance();
                }
            });

        $bar->finish();
        $this->newLine();
    }

    private function warmCategories(): void
    {
        $categories = Category::withCount('products')->get();

        foreach ($categories as $category) {
            // Cache category data
            Cache::put("category:{$category->id}", $category->toArray(), 7200);

            // Cache first page of products per category
            $products = Product::where('category_id', $category->id)
                ->active()
                ->paginate(20);

            Cache::put("category:{$category->id}:products:page:1", $products->toArray(), 1800);
        }

        $this->info("Warmed " . $categories->count() . " categories");
    }

    private function warmAll(): void
    {
        $this->warmProducts();
        $this->warmCategories();
        $this->warmPopularSearches();
    }

    private function warmPopularSearches(): void
    {
        $popularTerms = \App\Models\SearchLog::popular(50)->pluck('term');

        foreach ($popularTerms as $term) {
            // Trigger search and cache results
            app(\App\Services\ProductSearchService::class)->search(['query' => $term]);
        }

        $this->info("Warmed " . $popularTerms->count() . " popular searches");
    }
}
```

---

## ขั้นตอนที่ 1608: CDN Integration และ Cache Purging

```php
<?php

declare(strict_types=1);

// app/Services/CloudflareService.php
namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Http\Client\Response;

class CloudflareService
{
    private string $zoneId;
    private string $apiToken;
    private string $baseUrl = 'https://api.cloudflare.com/client/v4';

    public function __construct()
    {
        $this->zoneId   = config('services.cloudflare.zone_id');
        $this->apiToken = config('services.cloudflare.api_token');
    }

    public function purgeUrls(array $urls): Response
    {
        return Http::withToken($this->apiToken)
            ->post("{$this->baseUrl}/zones/{$this->zoneId}/purge_cache", [
                'files' => $urls,
            ]);
    }

    public function purgeAll(): Response
    {
        return Http::withToken($this->apiToken)
            ->post("{$this->baseUrl}/zones/{$this->zoneId}/purge_cache", [
                'purge_everything' => true,
            ]);
    }

    public function purgeTags(array $tags): Response
    {
        return Http::withToken($this->apiToken)
            ->post("{$this->baseUrl}/zones/{$this->zoneId}/purge_cache", [
                'tags' => $tags,
            ]);
    }

    public function purgeByPrefix(string $prefix): Response
    {
        return Http::withToken($this->apiToken)
            ->post("{$this->baseUrl}/zones/{$this->zoneId}/purge_cache", [
                'prefixes' => [$prefix],
            ]);
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Observers/ProductObserver.php
namespace App\Observers;

use App\Models\Product;
use App\Services\CloudflareService;

class ProductObserver
{
    public function __construct(private CloudflareService $cdn) {}

    public function updated(Product $product): void
    {
        // Purge CDN cache สำหรับ product นี้
        $urls = [
            config('app.url') . "/api/products/{$product->id}",
            config('app.url') . "/products/{$product->slug}",
        ];

        $this->cdn->purgeUrls($urls);
    }

    public function deleted(Product $product): void
    {
        $this->cdn->purgeUrls([
            config('app.url') . "/api/products/{$product->id}",
        ]);
    }
}
```

---

## ขั้นตอนที่ 1609: Multi-Level Cache

```php
<?php

declare(strict_types=1);

// app/Services/MultiLevelCacheService.php
namespace App\Services;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

class MultiLevelCacheService
{
    // L1: In-memory (per-request)
    private array $l1Cache = [];

    // L2: Redis (shared between workers)
    // L3: Database (source of truth)

    public function get(string $key, callable $fallback): mixed
    {
        // Check L1 first (fastest)
        if (isset($this->l1Cache[$key])) {
            return $this->l1Cache[$key];
        }

        // Check L2 (Redis)
        $l2Value = Cache::get($key);
        if ($l2Value !== null) {
            $this->l1Cache[$key] = $l2Value; // Populate L1
            return $l2Value;
        }

        // L3: Fetch from database
        $value = $fallback();

        // Populate both L1 and L2
        $this->l1Cache[$key] = $value;
        Cache::put($key, $value, 3600);

        return $value;
    }

    public function forget(string $key): void
    {
        unset($this->l1Cache[$key]);
        Cache::forget($key);
    }

    public function clearL1(): void
    {
        $this->l1Cache = [];
    }
}
```

---

## ขั้นตอนที่ 1610: Cache Stampede Prevention

```php
<?php

declare(strict_types=1);

// app/Services/StampedeProtectedCache.php
namespace App\Services;

use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Redis;

class StampedeProtectedCache
{
    /**
     * Probabilistic Early Expiration (PER) algorithm
     * ป้องกัน cache stampede โดยให้ process บางส่วน re-fetch ก่อน TTL หมด
     */
    public function get(string $key, callable $fallback, int $ttl = 3600, float $beta = 1.0): mixed
    {
        $cachedData = Redis::get($key);

        if ($cachedData !== null) {
            $data = unserialize($cachedData);

            // PER: คำนวณว่าควร re-fetch ก่อน expire หรือไม่
            $remainingTtl = Redis::ttl($key);
            $shouldRefetch = -$data['delta'] * $beta * log(mt_rand() / mt_getrandmax()) >= $remainingTtl;

            if (!$shouldRefetch) {
                return $data['value'];
            }
        }

        // Fetch fresh data
        $start = microtime(true);
        $value = $fallback();
        $delta = microtime(true) - $start;

        // Store with delta (fetch time) for PER calculation
        Redis::setex($key, $ttl, serialize([
            'value' => $value,
            'delta' => $delta,
        ]));

        return $value;
    }
}
```

---

## สรุป Part 58

| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| Cache-Aside | อ่านบ่อย | Simple, lazy | Cache miss ช้า |
| Write-Through | ข้อมูล critical | Consistent | เขียนช้า |
| Write-Behind | เขียนบ่อย | เขียนเร็ว | Risk of data loss |
| Cache Tags | Group invalidation | Flexible | Redis required |
| HTTP ETags | API responses | Saves bandwidth | Implementation complexity |
| CDN Caching | Static assets | Global speed | Purge complexity |
| Multi-Level | High performance | Very fast | Complex sync |
| PER Algorithm | Popular data | ป้องกัน stampede | คำนวณซับซ้อน |

ถัดไป → Part 59: Event-Driven Architecture
