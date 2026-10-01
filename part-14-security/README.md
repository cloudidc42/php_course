# Part 14: PHP Security
## ขั้นตอนที่ 331-370: เขียน PHP ให้ปลอดภัย

---

## ขั้นตอนที่ 331: OWASP Top 10 ใน PHP

```
OWASP Top 10 (2021):
A01 - Broken Access Control
A02 - Cryptographic Failures
A03 - Injection (SQL, NoSQL, OS, LDAP)
A04 - Insecure Design
A05 - Security Misconfiguration
A06 - Vulnerable and Outdated Components
A07 - Identification and Authentication Failures
A08 - Software and Data Integrity Failures
A09 - Security Logging and Monitoring Failures
A10 - Server-Side Request Forgery (SSRF)
```

---

## ขั้นตอนที่ 332: SQL Injection Prevention

```php
<?php
declare(strict_types=1);

// ========================
// VULNERABLE CODE - อย่าทำแบบนี้!
// ========================
// $id = $_GET['id'];
// $query = "SELECT * FROM users WHERE id = $id";
// // URL: /users?id=1 OR 1=1 -- จะดึงข้อมูลทั้งหมด!

// ========================
// SAFE - ใช้ Prepared Statements
// ========================
$pdo = new PDO('mysql:host=localhost;dbname=mydb', 'user', 'pass', [
    PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    PDO::ATTR_EMULATE_PREPARES => false, // ต้องปิด!
    PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
]);

// Safe Query
$id = (int) $_GET['id']; // Cast to int
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
$stmt->execute([$id]);
$user = $stmt->fetch();

// Named Parameters
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = :email AND active = :active");
$stmt->execute([':email' => $_POST['email'], ':active' => true]);

// Dynamic WHERE (safe)
function buildUserQuery(array $filters): array {
    $where = ['1=1'];
    $bindings = [];
    
    if (!empty($filters['name'])) {
        $where[] = 'name LIKE ?';
        $bindings[] = '%' . $filters['name'] . '%';
    }
    
    if (!empty($filters['role'])) {
        $where[] = 'role = ?';
        $bindings[] = $filters['role'];
    }
    
    if (!empty($filters['active'])) {
        $where[] = 'active = ?';
        $bindings[] = 1;
    }
    
    return ['WHERE ' . implode(' AND ', $where), $bindings];
}

// Dynamic ORDER BY (safe - whitelist)
function buildOrderBy(string $column, string $direction): string {
    $allowedColumns = ['id', 'name', 'email', 'created_at'];
    $allowedDirections = ['ASC', 'DESC'];
    
    $column = in_array($column, $allowedColumns) ? $column : 'id';
    $direction = in_array(strtoupper($direction), $allowedDirections) ? strtoupper($direction) : 'ASC';
    
    return "ORDER BY {$column} {$direction}";
}
```

---

## ขั้นตอนที่ 333: XSS Prevention

```php
<?php
declare(strict_types=1);

// ========================
// VULNERABLE - อย่าทำ!
// ========================
// echo $_GET['search']; // XSS: ?search=<script>alert(1)</script>

// ========================
// SAFE - Escape Output
// ========================

// Function สำหรับ escape
function e(string $value): string {
    return htmlspecialchars($value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
}

// HTML context
echo e($_GET['name'] ?? '');

// HTML attribute context
echo '<input value="' . e($value) . '">';

// URL context
echo '<a href="' . e(rawurlencode($url)) . '">';

// JavaScript context (ระวัง!)
echo '<script>var name = ' . json_encode($name, JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT) . '</script>';

// CSS context
echo '<style>color: ' . preg_replace('/[^a-zA-Z0-9#]/', '', $color) . '</style>';

// ========================
// Content Security Policy (CSP)
// ========================
function setSecurityHeaders(): void {
    // CSP
    header("Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-" . generateNonce() . "'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:");
    
    // XSS Protection (for older browsers)
    header("X-XSS-Protection: 1; mode=block");
    
    // Content Type
    header("X-Content-Type-Options: nosniff");
    
    // Frame Options (Clickjacking)
    header("X-Frame-Options: SAMEORIGIN");
    
    // HSTS (HTTPS only)
    if (isset($_SERVER['HTTPS'])) {
        header("Strict-Transport-Security: max-age=31536000; includeSubDomains; preload");
    }
    
    // Referrer Policy
    header("Referrer-Policy: strict-origin-when-cross-origin");
    
    // Permissions Policy
    header("Permissions-Policy: camera=(), microphone=(), geolocation=()");
}

function generateNonce(): string {
    return base64_encode(random_bytes(16));
}

setSecurityHeaders();

// ========================
// HTML Purifier (สำหรับ Rich Text)
// ========================
// ติดตั้ง: composer require ezyang/htmlpurifier
// require 'vendor/autoload.php';
// 
// $config = HTMLPurifier_Config::createDefault();
// $config->set('HTML.AllowedElements', 'p,br,b,i,u,h1,h2,h3,ul,ol,li,a');
// $config->set('HTML.AllowedAttributes', 'a.href,a.title');
// $purifier = new HTMLPurifier($config);
// $clean = $purifier->purify($dirtyHtml);
```

---

## ขั้นตอนที่ 334: Password Security

```php
<?php
declare(strict_types=1);

class PasswordManager {
    private const MIN_LENGTH = 8;
    private const ALGORITHM = PASSWORD_ARGON2ID;
    private const OPTIONS = [
        'memory_cost' => 65536,  // 64MB
        'time_cost' => 4,         // 4 iterations
        'threads' => 2,
    ];
    
    public function hash(string $password): string {
        return password_hash($password, self::ALGORITHM, self::OPTIONS);
    }
    
    public function verify(string $password, string $hash): bool {
        return password_verify($password, $hash);
    }
    
    public function needsRehash(string $hash): bool {
        return password_needs_rehash($hash, self::ALGORITHM, self::OPTIONS);
    }
    
    public function validate(string $password): array {
        $errors = [];
        
        if (strlen($password) < self::MIN_LENGTH) {
            $errors[] = "รหัสผ่านต้องมีอย่างน้อย " . self::MIN_LENGTH . " ตัวอักษร";
        }
        
        if (!preg_match('/[A-Z]/', $password)) {
            $errors[] = "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว";
        }
        
        if (!preg_match('/[a-z]/', $password)) {
            $errors[] = "ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว";
        }
        
        if (!preg_match('/[0-9]/', $password)) {
            $errors[] = "ต้องมีตัวเลขอย่างน้อย 1 ตัว";
        }
        
        if (!preg_match('/[@$!%*?&]/', $password)) {
            $errors[] = "ต้องมีอักขระพิเศษ (@$!%*?&) อย่างน้อย 1 ตัว";
        }
        
        return $errors;
    }
    
    public function generate(int $length = 16): string {
        $lowercase = 'abcdefghijklmnopqrstuvwxyz';
        $uppercase = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ';
        $numbers = '0123456789';
        $special = '@$!%*?&';
        
        $all = $lowercase . $uppercase . $numbers . $special;
        
        // Ensure at least one of each type
        $password = [
            $lowercase[random_int(0, strlen($lowercase) - 1)],
            $uppercase[random_int(0, strlen($uppercase) - 1)],
            $numbers[random_int(0, strlen($numbers) - 1)],
            $special[random_int(0, strlen($special) - 1)],
        ];
        
        // Fill remaining
        for ($i = 4; $i < $length; $i++) {
            $password[] = $all[random_int(0, strlen($all) - 1)];
        }
        
        // Shuffle
        shuffle($password);
        return implode('', $password);
    }
    
    // Check against common passwords
    public function isCommon(string $password): bool {
        $commonPasswords = ['password', '123456', '123456789', 'qwerty', 'abc123', 'monkey', '1234567', 'letmein', 'dragon', '111111'];
        return in_array(strtolower($password), $commonPasswords);
    }
}

// Password Reset Token
class PasswordResetTokenManager {
    private const TABLE = 'password_resets';
    private const EXPIRY = 3600; // 1 hour
    
    public function __construct(private \PDO $pdo) {}
    
    public function create(string $email): string {
        $token = bin2hex(random_bytes(32));
        $hashedToken = hash('sha256', $token);
        $expiry = date('Y-m-d H:i:s', time() + self::EXPIRY);
        
        // Delete old tokens for this email
        $this->pdo->prepare("DELETE FROM " . self::TABLE . " WHERE email = ?")->execute([$email]);
        
        // Create new token
        $stmt = $this->pdo->prepare(
            "INSERT INTO " . self::TABLE . " (email, token, expires_at) VALUES (?, ?, ?)"
        );
        $stmt->execute([$email, $hashedToken, $expiry]);
        
        return $token; // Return unhashed token
    }
    
    public function validate(string $token): ?string {
        $hashedToken = hash('sha256', $token);
        
        $stmt = $this->pdo->prepare(
            "SELECT email FROM " . self::TABLE . " WHERE token = ? AND expires_at > NOW()"
        );
        $stmt->execute([$hashedToken]);
        $result = $stmt->fetch(\PDO::FETCH_ASSOC);
        
        return $result['email'] ?? null;
    }
    
    public function delete(string $token): void {
        $hashedToken = hash('sha256', $token);
        $this->pdo->prepare("DELETE FROM " . self::TABLE . " WHERE token = ?")->execute([$hashedToken]);
    }
}
```

---

## ขั้นตอนที่ 335: Encryption & Hashing

```php
<?php
declare(strict_types=1);

class Encryption {
    private string $key;
    private const CIPHER = 'AES-256-GCM';
    
    public function __construct(string $key) {
        // Key ต้องยาว 32 bytes สำหรับ AES-256
        $this->key = hash('sha256', $key, true);
    }
    
    public function encrypt(string $plaintext): string {
        $ivLength = openssl_cipher_iv_length(self::CIPHER);
        $iv = random_bytes($ivLength);
        $tag = '';
        
        $encrypted = openssl_encrypt(
            $plaintext,
            self::CIPHER,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag,
            '',
            16
        );
        
        if ($encrypted === false) {
            throw new \RuntimeException('Encryption failed');
        }
        
        // IV + Tag + Ciphertext (all base64 encoded)
        return base64_encode($iv . $tag . $encrypted);
    }
    
    public function decrypt(string $ciphertext): string {
        $data = base64_decode($ciphertext);
        
        $ivLength = openssl_cipher_iv_length(self::CIPHER);
        $iv = substr($data, 0, $ivLength);
        $tag = substr($data, $ivLength, 16);
        $encrypted = substr($data, $ivLength + 16);
        
        $decrypted = openssl_decrypt(
            $encrypted,
            self::CIPHER,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );
        
        if ($decrypted === false) {
            throw new \RuntimeException('Decryption failed or data tampered');
        }
        
        return $decrypted;
    }
    
    // Encrypt array/object
    public function encryptJson(mixed $data): string {
        return $this->encrypt(json_encode($data));
    }
    
    public function decryptJson(string $ciphertext): mixed {
        return json_decode($this->decrypt($ciphertext), true);
    }
}

// HMAC Signature
class MessageSigner {
    public function __construct(private string $secret) {}
    
    public function sign(string $payload): string {
        return hash_hmac('sha256', $payload, $this->secret);
    }
    
    public function verify(string $payload, string $signature): bool {
        $expected = $this->sign($payload);
        return hash_equals($expected, $signature);
    }
    
    public function signedPayload(array $data, int $ttl = 3600): string {
        $payload = json_encode([
            'data' => $data,
            'exp' => time() + $ttl,
            'iat' => time(),
        ]);
        
        $signature = $this->sign($payload);
        return base64_encode($payload) . '.' . $signature;
    }
    
    public function verifySignedPayload(string $token): ?array {
        $parts = explode('.', $token, 2);
        if (count($parts) !== 2) return null;
        
        [$encodedPayload, $signature] = $parts;
        $payload = base64_decode($encodedPayload);
        
        if (!$this->verify($payload, $signature)) return null;
        
        $data = json_decode($payload, true);
        if (!$data || $data['exp'] < time()) return null;
        
        return $data['data'];
    }
}

// Secure Token Generation
class TokenGenerator {
    public static function generate(int $bytes = 32): string {
        return bin2hex(random_bytes($bytes));
    }
    
    public static function generateBase64(int $bytes = 32): string {
        return rtrim(strtr(base64_encode(random_bytes($bytes)), '+/', '-_'), '=');
    }
    
    public static function generateNumeric(int $length = 6): string {
        $max = (int) str_repeat('9', $length);
        return str_pad((string) random_int(0, $max), $length, '0', STR_PAD_LEFT);
    }
    
    public static function generateOtp(int $length = 6): string {
        return static::generateNumeric($length);
    }
}

// Usage
$enc = new Encryption($_ENV['ENCRYPTION_KEY']);
$encrypted = $enc->encrypt('Credit Card: 1234-5678-9012-3456');
$decrypted = $enc->decrypt($encrypted);

$signer = new MessageSigner($_ENV['APP_SECRET']);
$token = $signer->signedPayload(['user_id' => 42, 'role' => 'admin'], ttl: 900);
$data = $signer->verifySignedPayload($token);
```

---

## ขั้นตอนที่ 336: Rate Limiting & Brute Force Protection

```php
<?php
declare(strict_types=1);

class RateLimiter {
    public function __construct(
        private \PDO $pdo,
        private int $maxAttempts = 5,
        private int $decaySeconds = 60
    ) {}
    
    public function tooManyAttempts(string $key): bool {
        $this->clearExpired($key);
        
        $stmt = $this->pdo->prepare(
            "SELECT COUNT(*) FROM rate_limit_hits 
             WHERE key_hash = ? AND created_at > DATE_SUB(NOW(), INTERVAL ? SECOND)"
        );
        $stmt->execute([hash('sha256', $key), $this->decaySeconds]);
        
        return (int) $stmt->fetchColumn() >= $this->maxAttempts;
    }
    
    public function hit(string $key): int {
        $stmt = $this->pdo->prepare(
            "INSERT INTO rate_limit_hits (key_hash, created_at) VALUES (?, NOW())"
        );
        $stmt->execute([hash('sha256', $key)]);
        
        $stmt = $this->pdo->prepare(
            "SELECT COUNT(*) FROM rate_limit_hits 
             WHERE key_hash = ? AND created_at > DATE_SUB(NOW(), INTERVAL ? SECOND)"
        );
        $stmt->execute([hash('sha256', $key), $this->decaySeconds]);
        
        return (int) $stmt->fetchColumn();
    }
    
    public function clear(string $key): void {
        $stmt = $this->pdo->prepare("DELETE FROM rate_limit_hits WHERE key_hash = ?");
        $stmt->execute([hash('sha256', $key)]);
    }
    
    public function remaining(string $key): int {
        $stmt = $this->pdo->prepare(
            "SELECT COUNT(*) FROM rate_limit_hits 
             WHERE key_hash = ? AND created_at > DATE_SUB(NOW(), INTERVAL ? SECOND)"
        );
        $stmt->execute([hash('sha256', $key), $this->decaySeconds]);
        
        $hits = (int) $stmt->fetchColumn();
        return max(0, $this->maxAttempts - $hits);
    }
    
    private function clearExpired(string $key): void {
        $this->pdo->prepare(
            "DELETE FROM rate_limit_hits 
             WHERE key_hash = ? AND created_at < DATE_SUB(NOW(), INTERVAL ? SECOND)"
        )->execute([hash('sha256', $key), $this->decaySeconds]);
    }
}

// Login protection
function handleLogin(): void {
    $pdo = /* ... */;
    $rateLimiter = new RateLimiter($pdo, maxAttempts: 5, decaySeconds: 900);
    
    $email = $_POST['email'] ?? '';
    $key = 'login:' . $_SERVER['REMOTE_ADDR'] . ':' . $email;
    
    if ($rateLimiter->tooManyAttempts($key)) {
        http_response_code(429);
        echo json_encode(['error' => 'Too many login attempts. Please try again later.']);
        return;
    }
    
    // Attempt login...
    $success = false; // Auth::attempt($email, $password)
    
    if (!$success) {
        $hits = $rateLimiter->hit($key);
        $remaining = $rateLimiter->remaining($key);
        
        http_response_code(401);
        echo json_encode([
            'error' => 'Invalid credentials',
            'remaining_attempts' => $remaining,
        ]);
        return;
    }
    
    // Clear on success
    $rateLimiter->clear($key);
    
    // Continue with login...
}

// Header-based rate limiting for APIs
function checkApiRateLimit(): void {
    $apiKey = $_SERVER['HTTP_X_API_KEY'] ?? '';
    $limit = 100; // requests per minute
    
    // Using Redis for high-performance rate limiting
    // $redis = new Redis();
    // $key = "rate_limit:{$apiKey}:" . floor(time() / 60);
    // $count = $redis->incr($key);
    // if ($count === 1) $redis->expire($key, 60);
    // 
    // header("X-RateLimit-Limit: {$limit}");
    // header("X-RateLimit-Remaining: " . max(0, $limit - $count));
    // 
    // if ($count > $limit) {
    //     http_response_code(429);
    //     header('Retry-After: ' . (60 - (time() % 60)));
    //     die('Rate limit exceeded');
    // }
}
```

---

## ขั้นตอนที่ 337: Input Validation & Sanitization

```php
<?php
declare(strict_types=1);

class InputFilter {
    // Integer
    public static function int(mixed $value, ?int $min = null, ?int $max = null): int {
        $filtered = filter_var($value, FILTER_VALIDATE_INT);
        
        if ($filtered === false) {
            throw new \InvalidArgumentException("Value must be an integer");
        }
        
        if ($min !== null && $filtered < $min) {
            throw new \RangeException("Value must be >= {$min}");
        }
        
        if ($max !== null && $filtered > $max) {
            throw new \RangeException("Value must be <= {$max}");
        }
        
        return (int) $filtered;
    }
    
    // String (sanitized)
    public static function string(mixed $value, int $maxLength = 255): string {
        $str = trim((string) $value);
        $str = strip_tags($str);
        
        if (mb_strlen($str) > $maxLength) {
            $str = mb_substr($str, 0, $maxLength);
        }
        
        return $str;
    }
    
    // Email
    public static function email(mixed $value): string {
        $email = strtolower(trim((string) $value));
        $filtered = filter_var($email, FILTER_VALIDATE_EMAIL);
        
        if ($filtered === false) {
            throw new \InvalidArgumentException("Invalid email address");
        }
        
        return (string) $filtered;
    }
    
    // URL
    public static function url(mixed $value, array $allowedSchemes = ['http', 'https']): string {
        $url = trim((string) $value);
        $filtered = filter_var($url, FILTER_VALIDATE_URL);
        
        if ($filtered === false) {
            throw new \InvalidArgumentException("Invalid URL");
        }
        
        $scheme = parse_url($filtered, PHP_URL_SCHEME);
        if (!in_array($scheme, $allowedSchemes)) {
            throw new \InvalidArgumentException("URL scheme not allowed");
        }
        
        return $filtered;
    }
    
    // SSRF Prevention
    public static function safeUrl(string $url): string {
        $url = self::url($url);
        $host = parse_url($url, PHP_URL_HOST);
        
        // Block private/internal IPs
        $ip = gethostbyname($host);
        
        if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE) === false) {
            throw new \RuntimeException("URL points to internal/private network");
        }
        
        return $url;
    }
    
    // File Path (prevent directory traversal)
    public static function filePath(string $path, string $basePath): string {
        $fullPath = realpath($basePath . '/' . ltrim($path, '/'));
        
        if ($fullPath === false || !str_starts_with($fullPath, realpath($basePath))) {
            throw new \RuntimeException("Invalid file path");
        }
        
        return $fullPath;
    }
    
    // Array of values
    public static function array(mixed $value, int $maxItems = 100): array {
        if (!is_array($value)) {
            return [];
        }
        return array_slice($value, 0, $maxItems);
    }
}

// Security Headers Middleware
class SecurityMiddleware {
    public function handle(): void {
        // Remove server info
        header_remove('X-Powered-By');
        header_remove('Server');
        
        $this->setSecurityHeaders();
        $this->checkHttps();
    }
    
    private function setSecurityHeaders(): void {
        header("X-Content-Type-Options: nosniff");
        header("X-Frame-Options: DENY");
        header("X-XSS-Protection: 1; mode=block");
        header("Referrer-Policy: strict-origin-when-cross-origin");
        header("Permissions-Policy: camera=(), microphone=(), payment=()");
        
        if ($this->isHttps()) {
            header("Strict-Transport-Security: max-age=31536000; includeSubDomains; preload");
        }
    }
    
    private function checkHttps(): void {
        if (!$this->isHttps() && isset($_SERVER['HTTP_HOST'])) {
            $redirect = 'https://' . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
            header("Location: {$redirect}", true, 301);
            exit;
        }
    }
    
    private function isHttps(): bool {
        return isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off';
    }
}
```

---

## 🎯 สรุป Part 14

| หัวข้อ | Best Practice |
|--------|---------------|
| SQL Injection | Prepared Statements, PDO::ATTR_EMULATE_PREPARES = false |
| XSS | htmlspecialchars ENT_QUOTES, CSP Headers |
| CSRF | Synchronizer Token Pattern |
| Passwords | password_hash/verify Argon2id |
| Encryption | AES-256-GCM, openssl |
| Rate Limiting | Track by IP+email, exponential backoff |
| Input | Whitelist validation, filter_var |

**ถัดไป → Part 15: REST API Development**
