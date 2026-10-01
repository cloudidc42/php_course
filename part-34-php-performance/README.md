# Part 34: PHP Performance & Best Practices
## ขั้นตอนที่ 861-900: เขียน PHP ให้เร็วและมีประสิทธิภาพสูง

---

## ขั้นตอนที่ 861: Profiling & Benchmarking

```php
<?php
declare(strict_types=1);

// Simple Profiler
class Profiler {
    private static array $checkpoints = [];
    private static float $start = 0;
    
    public static function start(): void {
        self::$start = microtime(true);
        self::$checkpoints = [];
    }
    
    public static function checkpoint(string $label): void {
        self::$checkpoints[] = [
            'label' => $label,
            'time'  => microtime(true) - self::$start,
            'memory'=> memory_get_usage(true),
        ];
    }
    
    public static function report(): void {
        $total = microtime(true) - self::$start;
        
        foreach (self::$checkpoints as $i => $cp) {
            $prev = $i > 0 ? self::$checkpoints[$i - 1]['time'] : 0;
            $diff = $cp['time'] - $prev;
            printf(
                "%-30s %8.3fms (+%8.3fms) %s\n",
                $cp['label'],
                $cp['time'] * 1000,
                $diff * 1000,
                number_format($cp['memory'] / 1024 / 1024, 2) . 'MB'
            );
        }
        printf("Total: %.3fms | Peak: %sMB\n",
            $total * 1000,
            number_format(memory_get_peak_usage(true) / 1024 / 1024, 2)
        );
    }
}

// Benchmark functions
function benchmark(callable $fn, int $iterations = 1000): float {
    $start = microtime(true);
    for ($i = 0; $i < $iterations; $i++) {
        $fn();
    }
    return (microtime(true) - $start) / $iterations * 1000; // ms per call
}

// Install Xdebug for profiling:
// xdebug.mode=profile
// xdebug.output_dir=/tmp/xdebug
// xdebug.profiler_output_name=cachegrind.out.%p

// Use Blackfire.io for production profiling
// composer require blackfire/php-sdk --dev
```

---

## ขั้นตอนที่ 862: String & Array Optimization

```php
<?php
declare(strict_types=1);

// ❌ Slow: string concatenation in loop
function buildHtmlSlow(array $items): string {
    $html = '';
    foreach ($items as $item) {
        $html .= '<li>' . htmlspecialchars($item) . '</li>';
    }
    return '<ul>' . $html . '</ul>';
}

// ✅ Fast: array_map + implode
function buildHtmlFast(array $items): string {
    $items = array_map(fn(string $i) => '<li>' . htmlspecialchars($i, ENT_QUOTES) . '</li>', $items);
    return '<ul>' . implode('', $items) . '</ul>';
}

// ✅ Even faster for large lists: output buffering
function buildHtmlBuffer(array $items): string {
    ob_start();
    echo '<ul>';
    foreach ($items as $item) {
        echo '<li>', htmlspecialchars($item, ENT_QUOTES), '</li>';
    }
    echo '</ul>';
    return ob_get_clean();
}

// Array Optimization
// ❌ Slow: count() in loop condition
function sumSlice1(array $arr): int {
    $sum = 0;
    for ($i = 0; $i < count($arr); $i++) { // count() called each iteration!
        $sum += $arr[$i];
    }
    return $sum;
}

// ✅ Fast: cache count
function sumSlice2(array $arr): int {
    $sum = 0;
    $len = count($arr);  // Cache once
    for ($i = 0; $i < $len; $i++) {
        $sum += $arr[$i];
    }
    return $sum;
}

// ✅ Best: use array functions
function sumSlice3(array $arr): int {
    return (int) array_sum($arr);  // C implementation, fastest
}

// SplFixedArray for large numeric arrays (less memory)
$arr = SplFixedArray::fromArray(range(1, 100000));
echo $arr->getSize();  // 100000
echo $arr[50000];      // 50001

// SplStack, SplQueue for better semantics
$stack = new SplStack();
$stack->push('first');
$stack->push('second');
echo $stack->pop(); // second

// Pre-allocate arrays when size is known
$result = array_fill(0, 1000, null);
for ($i = 0; $i < 1000; $i++) {
    $result[$i] = $i * 2;
}
```

---

## ขั้นตอนที่ 863: Database Query Optimization

```php
<?php
declare(strict_types=1);

// ❌ N+1 query problem
function getUsersWithPostsSlow(PDO $pdo): array {
    $users = $pdo->query('SELECT * FROM users')->fetchAll();
    foreach ($users as &$user) {
        $stmt = $pdo->prepare('SELECT * FROM posts WHERE user_id = ?');
        $stmt->execute([$user['id']]);
        $user['posts'] = $stmt->fetchAll();  // 1 query per user!
    }
    return $users;
}

// ✅ Single query with JOIN
function getUsersWithPostsFast(PDO $pdo): array {
    $stmt = $pdo->query('
        SELECT u.*, p.id as post_id, p.title as post_title, p.created_at as post_created
        FROM users u
        LEFT JOIN posts p ON p.user_id = u.id
        ORDER BY u.id, p.created_at DESC
    ');
    
    $users = [];
    foreach ($stmt->fetchAll() as $row) {
        if (!isset($users[$row['id']])) {
            $users[$row['id']] = [
                'id'    => $row['id'],
                'name'  => $row['name'],
                'posts' => [],
            ];
        }
        if ($row['post_id']) {
            $users[$row['id']]['posts'][] = [
                'id'    => $row['post_id'],
                'title' => $row['post_title'],
            ];
        }
    }
    return array_values($users);
}

// ✅ Batch loading (similar to DataLoader)
function loadPostsForUsers(PDO $pdo, array $user_ids): array {
    if (empty($user_ids)) return [];
    
    $placeholders = implode(',', array_fill(0, count($user_ids), '?'));
    $stmt = $pdo->prepare("SELECT * FROM posts WHERE user_id IN ({$placeholders}) ORDER BY created_at DESC");
    $stmt->execute(array_values($user_ids));
    
    $posts_by_user = [];
    foreach ($stmt->fetchAll() as $post) {
        $posts_by_user[$post['user_id']][] = $post;
    }
    return $posts_by_user;
}

// Connection Pooling with PgBouncer / ProxySQL
// For MySQL: use persistent connections carefully
$pdo = new PDO(
    'mysql:host=localhost;dbname=mydb',
    'user',
    'pass',
    [
        PDO::ATTR_PERSISTENT => true,  // Reuse connections
        PDO::MYSQL_ATTR_COMPRESS => true,  // Compress data transfer
    ]
);

// Query result caching
class CachedRepository {
    private array $cache = [];
    
    public function __construct(private PDO $pdo, private int $ttl = 300) {}
    
    public function findById(int $id): ?array {
        $key = "user_{$id}";
        
        if (isset($this->cache[$key]) && $this->cache[$key]['expires'] > time()) {
            return $this->cache[$key]['data'];
        }
        
        $stmt = $this->pdo->prepare('SELECT * FROM users WHERE id = ?');
        $stmt->execute([$id]);
        $result = $stmt->fetch() ?: null;
        
        $this->cache[$key] = [
            'data'    => $result,
            'expires' => time() + $this->ttl,
        ];
        
        return $result;
    }
}
```

---

## ขั้นตอนที่ 864: Memory Management

```php
<?php
declare(strict_types=1);

// Generators for memory-efficient iteration
function readLargeFile(string $path): Generator {
    $handle = fopen($path, 'r');
    if (!$handle) throw new RuntimeException("Cannot open: {$path}");
    
    try {
        while (!feof($handle)) {
            $line = fgets($handle);
            if ($line !== false) {
                yield trim($line);
            }
        }
    } finally {
        fclose($handle);
    }
}

// Process 1M lines without loading all into memory
foreach (readLargeFile('/path/to/huge.csv') as $line) {
    processLine($line);
}

// Streaming CSV parser
function streamCsv(string $path, string $separator = ','): Generator {
    $handle = fopen($path, 'r');
    if (!$handle) return;
    
    $headers = fgetcsv($handle, 0, $separator);
    
    try {
        while (($row = fgetcsv($handle, 0, $separator)) !== false) {
            yield array_combine($headers, $row);
        }
    } finally {
        fclose($handle);
    }
}

// Process 10GB CSV in ~10MB memory
$count = 0;
$total = 0.0;
foreach (streamCsv('/data/sales.csv') as $row) {
    $total += (float) $row['amount'];
    $count++;
    
    if ($count % 10000 === 0) {
        echo "Processed {$count} rows, memory: " . memory_get_usage(true) / 1024 / 1024 . "MB\n";
    }
}

// Weak references (PHP 8.0+) - prevent memory leaks in caches
class WeakCache {
    private \WeakMap $cache;
    
    public function __construct() {
        $this->cache = new \WeakMap();
    }
    
    public function get(object $key): mixed {
        return $this->cache[$key] ?? null;
    }
    
    public function set(object $key, mixed $value): void {
        $this->cache[$key] = $value;  // Removed when $key goes out of scope
    }
}

// Unset large variables explicitly
function processLargeDataset(): void {
    $data = loadHugeArray();  // 500MB
    $result = process($data);
    unset($data);             // Free 500MB immediately
    
    saveResult($result);
}

// Limit memory in long-running scripts
ini_set('memory_limit', '256M');

// Monitor memory
function checkMemory(int $threshold_mb = 200): void {
    $usage = memory_get_usage(true) / 1024 / 1024;
    if ($usage > $threshold_mb) {
        trigger_error("High memory usage: {$usage}MB", E_USER_WARNING);
    }
}
```

---

## ขั้นตอนที่ 865: Caching Strategies

```php
<?php
declare(strict_types=1);

// APCu (local in-process cache)
if (function_exists('apcu_enabled') && apcu_enabled()) {
    $value = apcu_fetch('key', $success);
    if (!$success) {
        $value = computeExpensive();
        apcu_store('key', $value, 3600);
    }
}

// Redis (shared cache)
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);
$redis->auth('password');
$redis->select(1); // Database 1

// Basic operations
$redis->set('user:1', json_encode($user), 3600);
$user = json_decode($redis->get('user:1'), true);

// Hash (efficient for objects)
$redis->hMSet('user:1', [
    'name'  => 'John',
    'email' => 'john@example.com',
]);
$name = $redis->hGet('user:1', 'name');
$all  = $redis->hGetAll('user:1');

// List (queue)
$redis->rPush('queue:emails', json_encode($emailData));
$item = $redis->lPop('queue:emails');

// Set (unique values)
$redis->sAdd('online:users', $userId);
$redis->sRem('online:users', $userId);
$count = $redis->sCard('online:users');

// Sorted Set (leaderboard)
$redis->zAdd('leaderboard', $score, $userId);
$top10 = $redis->zRevRange('leaderboard', 0, 9, true); // With scores

// Pub/Sub
// Publisher
$redis->publish('notifications', json_encode(['type' => 'order.placed', 'id' => 123]));

// Cache-aside pattern
class RedisCache {
    public function __construct(private Redis $redis) {}
    
    public function remember(string $key, int $ttl, callable $callback): mixed {
        $cached = $this->redis->get($key);
        
        if ($cached !== false) {
            return unserialize($cached);
        }
        
        $value = $callback();
        $this->redis->setex($key, $ttl, serialize($value));
        
        return $value;
    }
    
    public function tags(string ...$tags): TaggedCache {
        return new TaggedCache($this, ...$tags);
    }
    
    public function invalidateTag(string $tag): void {
        $keys = $this->redis->sMembers("tag:{$tag}");
        if ($keys) {
            $this->redis->del(...$keys);
        }
        $this->redis->del("tag:{$tag}");
    }
}
```

---

## ขั้นตอนที่ 866: Async PHP with Fibers (PHP 8.1+)

```php
<?php
declare(strict_types=1);

// PHP Fibers - cooperative multitasking
$fiber = new Fiber(function(): void {
    $value = Fiber::suspend('first'); // Yield control, receive value
    echo "Got: {$value}\n";
    
    $value = Fiber::suspend('second');
    echo "Got: {$value}\n";
});

$result1 = $fiber->start();         // Returns 'first'
echo "Fiber suspended with: {$result1}\n";

$result2 = $fiber->resume('hello'); // Returns 'second', sends 'hello'
echo "Fiber suspended with: {$result2}\n";

$fiber->resume('world');            // Fiber completes

// Async HTTP requests with ReactPHP
// composer require react/http react/event-loop

use React\EventLoop\Loop;
use React\Http\Browser;
use Psr\Http\Message\ResponseInterface;

$browser = new Browser();

// Non-blocking HTTP requests
$promises = [
    $browser->get('https://api1.example.com/data'),
    $browser->get('https://api2.example.com/data'),
    $browser->get('https://api3.example.com/data'),
];

\React\Promise\all($promises)->then(function(array $responses): void {
    foreach ($responses as $response) {
        $data = json_decode($response->getBody(), true);
        processData($data);
    }
});

Loop::run(); // Start event loop

// Simple concurrency with promises
class AsyncProcessor {
    private array $results = [];
    
    public function process(array $tasks): array {
        $fibers = [];
        
        foreach ($tasks as $id => $task) {
            $fibers[$id] = new Fiber($task);
        }
        
        // Start all fibers
        foreach ($fibers as $id => $fiber) {
            $this->results[$id] = $fiber->start();
        }
        
        // Resume until all complete
        do {
            $any_running = false;
            foreach ($fibers as $id => $fiber) {
                if ($fiber->isSuspended()) {
                    $this->results[$id] = $fiber->resume();
                    $any_running = true;
                }
            }
        } while ($any_running);
        
        return $this->results;
    }
}
```

---

## 🎯 สรุป Part 34

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Profiling | Profiler class, Xdebug, Blackfire |
| String Optimization | array_map+implode vs concatenation |
| DB Optimization | N+1 prevention, batch loading |
| Memory Management | Generators, WeakMap, unset |
| Caching | APCu, Redis strategies |
| Async | Fibers, ReactPHP promises |

**ถัดไป → Part 35: Docker & Microservices**
