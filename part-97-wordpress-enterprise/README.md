# Part 97: WordPress Enterprise
## ขั้นตอนที่ 2771-2800: High-Performance WordPress สำหรับองค์กร

Redis Caching, Page Caching, CDN Integration,
Security Hardening และ High Availability

---

## ขั้นตอนที่ 2771: Redis Object Cache

```php
<?php
declare(strict_types=1);

/**
 * wp-content/plugins/enterprise-redis/enterprise-redis.php
 *
 * Plugin Name: Enterprise Redis Cache
 * Description: Advanced Redis object caching for WordPress
 */

namespace EnterpriseRedis;

class RedisObjectCache
{
    private \Redis $redis;
    private string $prefix;
    private int $defaultTtl;

    public function __construct()
    {
        $this->prefix = defined('WP_CACHE_KEY_SALT') ? WP_CACHE_KEY_SALT : 'wp_';
        $this->defaultTtl = 3600;
        $this->connect();
    }

    private function connect(): void
    {
        $this->redis = new \Redis();
        $host = defined('REDIS_HOST') ? REDIS_HOST : '127.0.0.1';
        $port = defined('REDIS_PORT') ? (int)REDIS_PORT : 6379;
        $password = defined('REDIS_PASSWORD') ? REDIS_PASSWORD : null;
        $database = defined('REDIS_DATABASE') ? (int)REDIS_DATABASE : 0;

        $this->redis->connect($host, $port, 1.0);

        if ($password) {
            $this->redis->auth($password);
        }

        $this->redis->select($database);
        $this->redis->setOption(\Redis::OPT_SERIALIZER, \Redis::SERIALIZER_IGBINARY);
    }

    public function get(string $key, string $group = 'default'): mixed
    {
        $cacheKey = $this->buildKey($key, $group);
        $value = $this->redis->get($cacheKey);
        return $value === false ? false : $value;
    }

    public function set(string $key, mixed $value, string $group = 'default', int $ttl = 0): bool
    {
        $cacheKey = $this->buildKey($key, $group);
        $ttl = $ttl > 0 ? $ttl : $this->getGroupTtl($group);

        if ($ttl > 0) {
            return $this->redis->setex($cacheKey, $ttl, $value);
        }

        return $this->redis->set($cacheKey, $value);
    }

    public function delete(string $key, string $group = 'default'): bool
    {
        $cacheKey = $this->buildKey($key, $group);
        return $this->redis->del($cacheKey) > 0;
    }

    public function flush(): bool
    {
        // ลบเฉพาะ keys ของ prefix นี้ ไม่ flush ทั้ง Redis
        $pattern = $this->prefix . '*';
        $keys = $this->redis->keys($pattern);

        if (empty($keys)) {
            return true;
        }

        return $this->redis->del($keys) >= 0;
    }

    public function flushGroup(string $group): bool
    {
        $pattern = $this->prefix . $group . ':*';
        $keys = $this->redis->keys($pattern);

        if (empty($keys)) {
            return true;
        }

        return $this->redis->del($keys) >= 0;
    }

    public function increment(string $key, int $offset = 1, string $group = 'default'): int|false
    {
        $cacheKey = $this->buildKey($key, $group);
        return $this->redis->incrBy($cacheKey, $offset);
    }

    private function buildKey(string $key, string $group): string
    {
        return $this->prefix . $group . ':' . md5($key);
    }

    private function getGroupTtl(string $group): int
    {
        $groupTtls = [
            'posts' => 3600,
            'terms' => 7200,
            'options' => 86400,
            'transient' => 3600,
            'default' => 1800,
        ];

        return $groupTtls[$group] ?? $this->defaultTtl;
    }

    public function getStats(): array
    {
        $info = $this->redis->info();
        return [
            'connected_clients' => $info['connected_clients'],
            'used_memory_human' => $info['used_memory_human'],
            'keyspace_hits' => $info['keyspace_hits'],
            'keyspace_misses' => $info['keyspace_misses'],
            'hit_rate' => $info['keyspace_hits'] > 0
                ? round($info['keyspace_hits'] / ($info['keyspace_hits'] + $info['keyspace_misses']) * 100, 2)
                : 0,
            'total_keys' => count($this->redis->keys($this->prefix . '*')),
        ];
    }
}
```

---

## ขั้นตอนที่ 2772: Full Page Caching

```php
<?php
declare(strict_types=1);

namespace EnterpriseWordPress\Cache;

class FullPageCache
{
    private \Redis $redis;
    private array $excludedPaths;
    private array $excludedQueryParams;
    private int $ttl;

    public function __construct()
    {
        $this->redis = new \Redis();
        $this->redis->connect(REDIS_HOST ?? '127.0.0.1', REDIS_PORT ?? 6379);
        $this->ttl = 3600;

        $this->excludedPaths = [
            '/wp-admin',
            '/wp-login.php',
            '/cart',
            '/checkout',
            '/my-account',
        ];

        $this->excludedQueryParams = ['preview', 'nocache', 's'];
    }

    public function serve(): void
    {
        if (!$this->isCacheable()) {
            return;
        }

        $key = $this->getCacheKey();
        $cached = $this->redis->get('fpc:' . $key);

        if ($cached !== false) {
            $data = json_decode($cached, true);
            header('Content-Type: text/html; charset=UTF-8');
            header('X-Cache: HIT');
            header('X-Cache-Age: ' . (time() - $data['cached_at']));
            echo $data['html'];
            exit;
        }

        // เริ่ม output buffering เพื่อ cache response
        header('X-Cache: MISS');
        ob_start(function (string $buffer) use ($key): string {
            $this->store($key, $buffer);
            return $buffer;
        });
    }

    private function store(string $key, string $html): void
    {
        $data = json_encode([
            'html' => $html,
            'cached_at' => time(),
            'url' => $_SERVER['REQUEST_URI'],
        ]);

        $this->redis->setex('fpc:' . $key, $this->ttl, $data);
    }

    public function invalidate(int $postId): void
    {
        $permalink = get_permalink($postId);
        if ($permalink) {
            $key = $this->buildKey($permalink);
            $this->redis->del('fpc:' . $key);
        }

        // Invalidate home page และ archive pages
        $this->redis->del('fpc:' . $this->buildKey(home_url('/')));
        $this->redis->del('fpc:' . $this->buildKey(home_url('/blog/')));
    }

    public function invalidateAll(): int
    {
        $keys = $this->redis->keys('fpc:*');
        if (empty($keys)) return 0;
        return $this->redis->del($keys);
    }

    private function isCacheable(): bool
    {
        // ไม่ cache ถ้าผู้ใช้ login อยู่
        if (is_user_logged_in()) return false;

        // ไม่ cache ถ้ามี session หรือ cart
        if (!empty($_SESSION) || (defined('WC_COOKIE') && isset($_COOKIE[WC_COOKIE]))) {
            return false;
        }

        $uri = $_SERVER['REQUEST_URI'] ?? '';

        // ตรวจสอบ excluded paths
        foreach ($this->excludedPaths as $path) {
            if (str_starts_with($uri, $path)) return false;
        }

        // ตรวจสอบ excluded query params
        foreach ($this->excludedQueryParams as $param) {
            if (isset($_GET[$param])) return false;
        }

        // Cache เฉพาะ GET requests
        return $_SERVER['REQUEST_METHOD'] === 'GET';
    }

    private function getCacheKey(): string
    {
        $url = (isset($_SERVER['HTTPS']) ? 'https' : 'http') .
               '://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
        return $this->buildKey($url);
    }

    private function buildKey(string $url): string
    {
        return md5($url);
    }
}
```

---

## ขั้นตอนที่ 2773: CDN Integration

```php
<?php
declare(strict_types=1);

namespace EnterpriseWordPress\CDN;

class CloudFrontIntegration
{
    private string $cdnDomain;
    private array $assetExtensions;

    public function __construct()
    {
        $this->cdnDomain = defined('CDN_DOMAIN') ? CDN_DOMAIN : '';
        $this->assetExtensions = ['js', 'css', 'png', 'jpg', 'jpeg', 'gif', 'webp', 'svg', 'woff', 'woff2', 'ttf', 'ico'];
    }

    public function register(): void
    {
        if (empty($this->cdnDomain)) return;

        add_filter('style_loader_src', [$this, 'rewriteUrl'], 10, 1);
        add_filter('script_loader_src', [$this, 'rewriteUrl'], 10, 1);
        add_filter('wp_get_attachment_url', [$this, 'rewriteUrl'], 10, 1);
        add_filter('the_content', [$this, 'rewriteContentUrls'], 20, 1);
    }

    public function rewriteUrl(string $url): string
    {
        if (empty($url) || !$this->isLocalAsset($url)) {
            return $url;
        }

        $siteUrl = get_option('siteurl');
        return str_replace($siteUrl, 'https://' . $this->cdnDomain, $url);
    }

    public function rewriteContentUrls(string $content): string
    {
        if (empty($this->cdnDomain)) return $content;

        $siteUrl = get_option('siteurl');
        $cdnUrl = 'https://' . $this->cdnDomain;

        // Rewrite image URLs in content
        $pattern = '/(src|href)=["\'](' . preg_quote($siteUrl, '/') . '[^"\']+\.(' .
                   implode('|', $this->assetExtensions) . '))["\']$/i';

        return preg_replace_callback($pattern, function ($matches) use ($siteUrl, $cdnUrl): string {
            $newUrl = str_replace($siteUrl, $cdnUrl, $matches[2]);
            return $matches[1] . '="' . $newUrl . '"';
        }, $content);
    }

    private function isLocalAsset(string $url): bool
    {
        $siteUrl = get_option('siteurl');
        if (!str_starts_with($url, $siteUrl)) return false;

        $extension = strtolower(pathinfo(parse_url($url, PHP_URL_PATH), PATHINFO_EXTENSION));
        return in_array($extension, $this->assetExtensions);
    }

    public function invalidateCDN(array $paths): bool
    {
        $client = new \Aws\CloudFront\CloudFrontClient([
            'region' => AWS_REGION ?? 'ap-southeast-1',
            'version' => 'latest',
            'credentials' => [
                'key' => AWS_ACCESS_KEY_ID,
                'secret' => AWS_SECRET_ACCESS_KEY,
            ],
        ]);

        $result = $client->createInvalidation([
            'DistributionId' => CDN_DISTRIBUTION_ID,
            'InvalidationBatch' => [
                'CallerReference' => uniqid(),
                'Paths' => [
                    'Quantity' => count($paths),
                    'Items' => $paths,
                ],
            ],
        ]);

        return $result['Invalidation']['Status'] === 'InProgress';
    }
}
```

---

## ขั้นตอนที่ 2774: Security Hardening

```php
<?php
declare(strict_types=1);

namespace EnterpriseWordPress\Security;

class SecurityHardening
{
    public function register(): void
    {
        // ซ่อน WordPress version
        remove_action('wp_head', 'wp_generator');
        add_filter('the_generator', '__return_empty_string');

        // ปิด XML-RPC
        add_filter('xmlrpc_enabled', '__return_false');

        // Security headers
        add_action('send_headers', [$this, 'addSecurityHeaders']);

        // Rate limiting
        add_action('wp_login_failed', [$this, 'handleFailedLogin']);

        // ป้องกัน user enumeration
        add_action('template_redirect', [$this, 'preventUserEnumeration']);

        // Force HTTPS
        add_action('init', [$this, 'forceHttps']);

        // ป้องกัน file editing ใน admin
        define('DISALLOW_FILE_EDIT', true);
    }

    public function addSecurityHeaders(): void
    {
        header('X-Content-Type-Options: nosniff');
        header('X-Frame-Options: SAMEORIGIN');
        header('X-XSS-Protection: 1; mode=block');
        header('Referrer-Policy: strict-origin-when-cross-origin');
        header('Permissions-Policy: geolocation=(), microphone=(), camera=()');

        if (is_ssl()) {
            header('Strict-Transport-Security: max-age=31536000; includeSubDomains; preload');
        }

        // Content Security Policy
        $csp = implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'nonce-" . $this->getNonce() . "' https://www.google-analytics.com",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
            "font-src 'self' https://fonts.gstatic.com",
            "img-src 'self' data: https:",
            "connect-src 'self'",
            "frame-ancestors 'none'",
        ]);

        header("Content-Security-Policy: {$csp}");
    }

    public function handleFailedLogin(string $username): void
    {
        $redis = new \Redis();
        $redis->connect(REDIS_HOST ?? '127.0.0.1');

        $ip = $this->getClientIp();
        $key = 'wp:login_attempts:' . md5($ip);

        $attempts = (int)$redis->get($key);
        $attempts++;
        $redis->setex($key, 900, $attempts); // 15 minutes window

        if ($attempts >= 5) {
            // Block IP
            $blockKey = 'wp:blocked_ip:' . md5($ip);
            $redis->setex($blockKey, 3600, 1); // Block for 1 hour

            // Log security event
            error_log("Blocked IP {$ip} after {$attempts} failed login attempts");

            // ส่งแจ้งเตือนให้ admin
            wp_mail(
                get_option('admin_email'),
                'Security Alert: Brute Force Attempt',
                "IP {$ip} has been blocked after {$attempts} failed login attempts."
            );
        }
    }

    public function preventUserEnumeration(): void
    {
        if (is_author() && get_query_var('author')) {
            wp_redirect(home_url('/'), 301);
            exit;
        }

        if (isset($_GET['author'])) {
            wp_redirect(home_url('/'), 301);
            exit;
        }
    }

    public function forceHttps(): void
    {
        if (!is_ssl() && !is_admin() && isset($_SERVER['HTTP_HOST'])) {
            wp_redirect('https://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'], 301);
            exit;
        }
    }

    private function getNonce(): string
    {
        static $nonce = null;
        if ($nonce === null) {
            $nonce = base64_encode(random_bytes(16));
        }
        return $nonce;
    }

    private function getClientIp(): string
    {
        $headers = ['HTTP_CF_CONNECTING_IP', 'HTTP_X_FORWARDED_FOR', 'REMOTE_ADDR'];
        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                $ip = trim(explode(',', $_SERVER[$header])[0]);
                if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
                    return $ip;
                }
            }
        }
        return $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
    }
}
```

---

## ขั้นตอนที่ 2775: Database Optimization

```php
<?php
declare(strict_types=1);

namespace EnterpriseWordPress\Database;

class DatabaseOptimizer
{
    private \wpdb $wpdb;

    public function __construct()
    {
        global $wpdb;
        $this->wpdb = $wpdb;
    }

    public function addIndexes(): void
    {
        // เพิ่ม indexes ที่ขาดหายไป
        $indexes = [
            "ALTER TABLE {$this->wpdb->posts} ADD INDEX idx_post_date (post_date)",
            "ALTER TABLE {$this->wpdb->posts} ADD INDEX idx_post_status_type (post_status, post_type)",
            "ALTER TABLE {$this->wpdb->postmeta} ADD INDEX idx_meta_key_value (meta_key, meta_value(100))",
            "ALTER TABLE {$this->wpdb->options} ADD INDEX idx_autoload (autoload)",
        ];

        foreach ($indexes as $sql) {
            $this->wpdb->query($sql);
        }
    }

    public function cleanupOrphanedData(): array
    {
        $stats = [];

        // ลบ post revisions เก่า
        $deleted = $this->wpdb->query(
            "DELETE FROM {$this->wpdb->posts}
             WHERE post_type = 'revision'
             AND post_modified < DATE_SUB(NOW(), INTERVAL 30 DAY)"
        );
        $stats['deleted_revisions'] = $deleted;

        // ลบ transients หมดอายุ
        $deleted = $this->wpdb->query(
            "DELETE a, b FROM {$this->wpdb->options} a
             LEFT JOIN {$this->wpdb->options} b ON b.option_name = REPLACE(a.option_name, '_transient_', '_transient_timeout_')
             WHERE a.option_name LIKE '%_transient_%'
             AND b.option_value < UNIX_TIMESTAMP()"
        );
        $stats['deleted_transients'] = $deleted;

        // ลบ orphaned postmeta
        $deleted = $this->wpdb->query(
            "DELETE FROM {$this->wpdb->postmeta}
             WHERE post_id NOT IN (SELECT ID FROM {$this->wpdb->posts})"
        );
        $stats['deleted_orphaned_meta'] = $deleted;

        // ลบ spam comments
        $deleted = $this->wpdb->query(
            "DELETE FROM {$this->wpdb->comments}
             WHERE comment_approved = 'spam'
             AND comment_date < DATE_SUB(NOW(), INTERVAL 15 DAY)"
        );
        $stats['deleted_spam'] = $deleted;

        return $stats;
    }

    public function getSlowQueries(): array
    {
        return $this->wpdb->get_results(
            "SELECT query_time, sql_text
             FROM mysql.slow_log
             WHERE query_time > 1
             AND sql_text LIKE '%{$this->wpdb->prefix}%'
             ORDER BY query_time DESC
             LIMIT 20"
        );
    }

    public function optimizeTables(): array
    {
        $tables = $this->wpdb->get_col("SHOW TABLES LIKE '{$this->wpdb->prefix}%'");
        $results = [];

        foreach ($tables as $table) {
            $result = $this->wpdb->get_results("OPTIMIZE TABLE {$table}");
            $results[$table] = $result[0]->Msg_text ?? 'OK';
        }

        return $results;
    }
}
```

---

## ขั้นตอนที่ 2776: WordPress wp-config.php สำหรับ Enterprise

```php
<?php
declare(strict_types=1);

/**
 * wp-config.php สำหรับ Enterprise WordPress
 * Environment-aware configuration
 */

// โหลด environment variables
if (file_exists(__DIR__ . '/.env')) {
    $lines = file(__DIR__ . '/.env', FILE_IGNORE_NEW_LINES | FILE_SKIP_EMPTY_LINES);
    foreach ($lines as $line) {
        if (str_starts_with($line, '#')) continue;
        [$name, $value] = array_map('trim', explode('=', $line, 2));
        if (!empty($name)) {
            putenv("{$name}={$value}");
            $_ENV[$name] = $value;
        }
    }
}

$env = getenv('WP_ENV') ?: 'production';

// Database
define('DB_NAME', getenv('DB_NAME'));
define('DB_USER', getenv('DB_USER'));
define('DB_PASSWORD', getenv('DB_PASSWORD'));
define('DB_HOST', getenv('DB_HOST') ?: '127.0.0.1');
define('DB_CHARSET', 'utf8mb4');
define('DB_COLLATE', 'utf8mb4_unicode_ci');

// Redis
define('REDIS_HOST', getenv('REDIS_HOST') ?: '127.0.0.1');
define('REDIS_PORT', (int)(getenv('REDIS_PORT') ?: 6379));
define('REDIS_PASSWORD', getenv('REDIS_PASSWORD') ?: '');
define('WP_REDIS_HOST', REDIS_HOST);
define('WP_REDIS_DATABASE', 1);

// Performance
define('WP_MEMORY_LIMIT', '512M');
define('WP_MAX_MEMORY_LIMIT', '1024M');
define('WP_POST_REVISIONS', 5);
define('AUTOSAVE_INTERVAL', 300);
define('EMPTY_TRASH_DAYS', 14);

// Security
define('DISALLOW_FILE_EDIT', true);
define('DISALLOW_FILE_MODS', $env === 'production');
define('FORCE_SSL_ADMIN', true);
define('WP_DEBUG', $env !== 'production');
define('WP_DEBUG_LOG', $env !== 'production');
define('WP_DEBUG_DISPLAY', false);

// CDN
define('CDN_DOMAIN', getenv('CDN_DOMAIN') ?: '');

// WP keys & salts (ควร generate ใหม่ทุก instance)
define('AUTH_KEY', getenv('AUTH_KEY'));
define('SECURE_AUTH_KEY', getenv('SECURE_AUTH_KEY'));
define('LOGGED_IN_KEY', getenv('LOGGED_IN_KEY'));
define('NONCE_KEY', getenv('NONCE_KEY'));
define('AUTH_SALT', getenv('AUTH_SALT'));
define('SECURE_AUTH_SALT', getenv('SECURE_AUTH_SALT'));
define('LOGGED_IN_SALT', getenv('LOGGED_IN_SALT'));
define('NONCE_SALT', getenv('NONCE_SALT'));

$table_prefix = 'wp_';

// Load WordPress
if (!defined('ABSPATH')) {
    define('ABSPATH', __DIR__ . '/');
}

require_once ABSPATH . 'wp-settings.php';
```

---

## สรุปบทที่ 97

| หัวข้อ | เทคนิค | ผลลัพธ์ |
|--------|--------|---------|
| Redis Cache | Object + Page Cache | Fast responses |
| CDN | CloudFront rewrites | Global delivery |
| Security | Headers + Rate limit | Protected site |
| DB Optimization | Indexes + Cleanup | Fast queries |
| Config Management | ENV-based wp-config | Portable setup |
| High Availability | Load balancer ready | Zero downtime |

ถัดไป → Part 98: Final Project Planning
