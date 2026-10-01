# Part 90: Performance Testing
## ขั้นตอนที่ 2561-2590: Load Testing, Stress Testing และ Bottleneck Analysis

k6, JMeter, การทดสอบ endpoints, หาคอขวด
และการวางแผน capacity

---

## ขั้นตอนที่ 2561: k6 Load Testing

```javascript
// k6/basic-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const apiDuration = new Trend('api_duration', true);

export const options = {
    stages: [
        { duration: '2m', target: 50 },   // ramp up
        { duration: '5m', target: 50 },   // steady state
        { duration: '2m', target: 100 },  // ramp up more
        { duration: '5m', target: 100 },  // steady state
        { duration: '2m', target: 0 },    // ramp down
    ],
    thresholds: {
        'http_req_duration': ['p(95)<500', 'p(99)<1000'],
        'http_req_failed': ['rate<0.01'],
        'errors': ['rate<0.05'],
    },
};

const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';

export default function () {
    const params = {
        headers: {
            'Content-Type': 'application/json',
            'Accept': 'application/json',
        },
    };

    // Test 1: Get posts list
    let res = http.get(`${BASE_URL}/api/v1/posts`, params);
    const postListCheck = check(res, {
        'posts: status 200': (r) => r.status === 200,
        'posts: has data': (r) => JSON.parse(r.body).data !== undefined,
        'posts: response time < 500ms': (r) => r.timings.duration < 500,
    });
    errorRate.add(!postListCheck);
    apiDuration.add(res.timings.duration, { endpoint: 'posts_list' });

    sleep(1);

    // Test 2: Get single post
    res = http.get(`${BASE_URL}/api/v1/posts/hello-world`, params);
    check(res, {
        'post: status 200 or 404': (r) => [200, 404].includes(r.status),
    });

    sleep(0.5);
}
```

```javascript
// k6/spike-test.js - ทดสอบ spike traffic
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '10s', target: 10 },
        { duration: '1m', target: 10 },
        { duration: '10s', target: 500 },  // spike!
        { duration: '3m', target: 500 },
        { duration: '10s', target: 10 },   // back to normal
        { duration: '3m', target: 10 },
        { duration: '10s', target: 0 },
    ],
    thresholds: {
        'http_req_duration': ['p(99)<5000'],
        'http_req_failed': ['rate<0.1'],
    },
};

export default function () {
    const res = http.get(`${__ENV.BASE_URL}/api/v1/posts`);
    check(res, { 'status is 200': (r) => r.status === 200 });
    sleep(1);
}
```

---

## ขั้นตอนที่ 2562: Benchmark Controller

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\DB;

class BenchmarkController extends \App\Http\Controllers\Controller
{
    // Endpoint ที่ optimize แล้ว
    public function optimized(Request $request): JsonResponse
    {
        $posts = Cache::remember('posts:list:optimized', 300, function () {
            return Post::query()
                ->select(['id', 'title', 'slug', 'excerpt', 'created_at'])
                ->with(['author:id,name', 'categories:id,name'])
                ->withCount('comments')
                ->published()
                ->latest()
                ->limit(20)
                ->get();
        });

        return response()->json(['data' => $posts]);
    }

    // Endpoint ที่มีปัญหา N+1
    public function unoptimized(Request $request): JsonResponse
    {
        $posts = Post::published()->latest()->limit(20)->get();

        $result = $posts->map(function ($post) {
            return [
                'id' => $post->id,
                'title' => $post->title,
                'author' => $post->author->name, // N+1!
                'categories' => $post->categories->pluck('name'), // N+1!
                'comments_count' => $post->comments->count(), // N+1!
            ];
        });

        return response()->json(['data' => $result]);
    }

    // ทดสอบ database performance
    public function dbBenchmark(): JsonResponse
    {
        $start = microtime(true);

        // Query ที่ซับซ้อน
        $stats = DB::select("
            SELECT
                COUNT(*) as total_posts,
                AVG(views) as avg_views,
                MAX(views) as max_views,
                SUM(CASE WHEN status = 'published' THEN 1 ELSE 0 END) as published_count
            FROM posts
            WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
        ");

        $queryTime = (microtime(true) - $start) * 1000;

        return response()->json([
            'stats' => $stats[0],
            'query_time_ms' => round($queryTime, 2),
            'memory_usage_mb' => round(memory_get_usage(true) / 1024 / 1024, 2),
        ]);
    }
}
```

---

## ขั้นตอนที่ 2563: Performance Profiling

```php
<?php
declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\DB;

class PerformanceProfiler
{
    private array $checkpoints = [];
    private float $startTime;
    private float $startMemory;

    public function __construct()
    {
        $this->startTime = microtime(true);
        $this->startMemory = memory_get_usage(true);
        DB::enableQueryLog();
    }

    public function checkpoint(string $name): void
    {
        $this->checkpoints[$name] = [
            'time' => (microtime(true) - $this->startTime) * 1000,
            'memory' => memory_get_usage(true),
        ];
    }

    public function getReport(): array
    {
        $totalTime = (microtime(true) - $this->startTime) * 1000;
        $queries = DB::getQueryLog();

        return [
            'total_time_ms' => round($totalTime, 2),
            'memory_peak_mb' => round(memory_get_peak_usage(true) / 1024 / 1024, 2),
            'memory_used_mb' => round((memory_get_usage(true) - $this->startMemory) / 1024 / 1024, 2),
            'checkpoints' => $this->checkpoints,
            'queries' => [
                'count' => count($queries),
                'total_time_ms' => round(array_sum(array_column($queries, 'time')), 2),
                'slowest' => $this->getSlowestQueries($queries, 5),
                'duplicates' => $this->findDuplicateQueries($queries),
            ],
        ];
    }

    private function getSlowestQueries(array $queries, int $limit): array
    {
        usort($queries, fn ($a, $b) => $b['time'] <=> $a['time']);
        return array_map(fn ($q) => [
            'sql' => $q['query'],
            'time_ms' => round($q['time'], 2),
            'bindings' => $q['bindings'],
        ], array_slice($queries, 0, $limit));
    }

    private function findDuplicateQueries(array $queries): array
    {
        $counts = [];
        foreach ($queries as $query) {
            $key = $query['query'];
            $counts[$key] = ($counts[$key] ?? 0) + 1;
        }

        return array_filter($counts, fn ($count) => $count > 1);
    }
}
```

---

## ขั้นตอนที่ 2564: Capacity Planning

```php
<?php
declare(strict_types=1);

namespace App\Services;

class CapacityPlanner
{
    /**
     * คำนวณ capacity ที่ต้องการ
     */
    public function calculateRequired(
        int $peakRPS,      // requests per second
        float $avgLatency, // milliseconds
        float $cpuPerReq,  // CPU time per request (ms)
        float $memPerReq,  // Memory per request (MB)
        int $concurrency   // concurrent requests
    ): array {
        // Little's Law: N = λ × W
        // N = number of concurrent requests
        // λ = arrival rate (RPS)
        // W = average response time
        $requiredConcurrency = $peakRPS * ($avgLatency / 1000);

        // CPU cores needed
        $cpuCores = ceil(($peakRPS * $cpuPerReq) / 1000);

        // Memory needed (GB)
        $memoryGB = ceil(($concurrency * $memPerReq) / 1024);

        // ค่าปลอดภัย (20% buffer)
        $safetyFactor = 1.2;

        return [
            'required_concurrency' => round($requiredConcurrency),
            'recommended_concurrency' => round($requiredConcurrency * $safetyFactor),
            'cpu_cores_min' => $cpuCores,
            'cpu_cores_recommended' => (int)ceil($cpuCores * $safetyFactor),
            'memory_gb_min' => $memoryGB,
            'memory_gb_recommended' => (int)ceil($memoryGB * $safetyFactor),
            'instances_3_cores' => (int)ceil(($cpuCores * $safetyFactor) / 3),
            'instances_4_cores' => (int)ceil(($cpuCores * $safetyFactor) / 4),
        ];
    }

    /**
     * วิเคราะห์ bottleneck
     */
    public function analyzeBottlenecks(array $metrics): array
    {
        $bottlenecks = [];

        // CPU bottleneck
        if ($metrics['cpu_usage_percent'] > 80) {
            $bottlenecks[] = [
                'type' => 'cpu',
                'severity' => $metrics['cpu_usage_percent'] > 90 ? 'critical' : 'warning',
                'value' => $metrics['cpu_usage_percent'],
                'recommendation' => 'เพิ่ม CPU cores หรือ optimize code',
            ];
        }

        // Memory bottleneck
        if ($metrics['memory_usage_percent'] > 85) {
            $bottlenecks[] = [
                'type' => 'memory',
                'severity' => 'warning',
                'value' => $metrics['memory_usage_percent'],
                'recommendation' => 'เพิ่ม RAM หรือ optimize memory usage',
            ];
        }

        // Database bottleneck
        if ($metrics['db_query_time_avg_ms'] > 100) {
            $bottlenecks[] = [
                'type' => 'database',
                'severity' => $metrics['db_query_time_avg_ms'] > 500 ? 'critical' : 'warning',
                'value' => $metrics['db_query_time_avg_ms'],
                'recommendation' => 'เพิ่ม index, optimize queries หรือ scale database',
            ];
        }

        // Cache hit rate
        if ($metrics['cache_hit_rate'] < 0.8) {
            $bottlenecks[] = [
                'type' => 'cache',
                'severity' => 'info',
                'value' => $metrics['cache_hit_rate'],
                'recommendation' => 'เพิ่ม caching strategy',
            ];
        }

        return $bottlenecks;
    }
}
```

---

## ขั้นตอนที่ 2565: Artillery Load Test Configuration

```yaml
# artillery/load-test.yml
config:
  target: "https://api.example.com"
  phases:
    - duration: 120
      arrivalRate: 10
      name: "Warm up"
    - duration: 300
      arrivalRate: 50
      name: "Sustained load"
    - duration: 60
      arrivalRate: 100
      name: "Peak load"
  http:
    timeout: 10
    maxSockets: 100
  plugins:
    metrics-by-endpoint:
      useOnlyRequestNames: true

scenarios:
  - name: "Browse posts"
    weight: 60
    flow:
      - get:
          url: "/api/v1/posts"
          expect:
            - statusCode: 200
            - hasProperty: "data"
      - think: 2
      - get:
          url: "/api/v1/posts/{{ postSlug }}"
          expect:
            - statusCode: 200

  - name: "User authentication"
    weight: 20
    flow:
      - post:
          url: "/api/v1/login"
          json:
            email: "test@example.com"
            password: "password"
          capture:
            - json: "$.token"
              as: "authToken"
      - think: 1
      - get:
          url: "/api/v1/user"
          headers:
            Authorization: "Bearer {{ authToken }}"

  - name: "Search"
    weight: 20
    flow:
      - get:
          url: "/api/v1/posts"
          qs:
            search: "laravel"
          expect:
            - statusCode: 200
```

---

## สรุปบทที่ 90

| เครื่องมือ | Use Case | Metric |
|-----------|---------|-------|
| k6 | Load/Stress testing | RPS, p95, p99 |
| Artillery | Scenario testing | Throughput |
| JMeter | Complex scenarios | Response time |
| PHP Profiler | Code profiling | CPU, Memory |
| Query Log | DB optimization | Query count/time |
| Little's Law | Capacity planning | Concurrency |

ถัดไป → Part 91: PHP Standards (PSR)
