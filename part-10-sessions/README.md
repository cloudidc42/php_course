# Part 10: Sessions & Cookies
## ขั้นตอนที่ 241-270: State Management ใน PHP

---

## ขั้นตอนที่ 241: PHP Sessions

```php
<?php
declare(strict_types=1);

// ========================
// Session Configuration
// ========================
ini_set('session.cookie_httponly', '1');      // ป้องกัน XSS
ini_set('session.cookie_secure', '1');         // HTTPS only
ini_set('session.cookie_samesite', 'Strict'); // CSRF protection
ini_set('session.gc_maxlifetime', '3600');     // 1 hour
ini_set('session.use_strict_mode', '1');       // Reject uninitialized IDs

session_start();

// ========================
// Basic Session Operations
// ========================
// Set
$_SESSION['user_id'] = 42;
$_SESSION['username'] = 'somchai';
$_SESSION['role'] = 'admin';
$_SESSION['last_activity'] = time();

// Get
$userId = $_SESSION['user_id'] ?? null;
$username = $_SESSION['username'] ?? 'Guest';

// Delete specific key
unset($_SESSION['role']);

// Check
if (isset($_SESSION['user_id'])) {
    echo "Logged in as " . $_SESSION['username'];
}

// Destroy Session
session_unset();      // ลบ session data ทั้งหมด
session_destroy();    // ทำลาย session

// Regenerate ID (ป้องกัน Session Fixation)
session_regenerate_id(true);

// ========================
// Session Info
// ========================
echo session_id();     // Session ID
echo session_name();   // Session name (PHPSESSID)
echo session_status(); // PHP_SESSION_NONE|PHP_SESSION_ACTIVE|PHP_SESSION_DISABLED

// Session ตาม custom path
session_save_path('/var/lib/php/sessions/myapp');
```

---

## ขั้นตอนที่ 242: Session Manager Class

```php
<?php
declare(strict_types=1);

class Session {
    private static bool $started = false;
    
    public static function start(array $options = []): void {
        if (self::$started) return;
        
        $defaults = [
            'cookie_httponly' => true,
            'cookie_secure' => isset($_SERVER['HTTPS']),
            'cookie_samesite' => 'Strict',
            'gc_maxlifetime' => 3600,
            'use_strict_mode' => true,
            'use_only_cookies' => true,
        ];
        
        session_start(array_merge($defaults, $options));
        self::$started = true;
        
        // Check for session timeout
        static::checkTimeout();
        
        // Regenerate periodically
        static::periodicRegenerate();
    }
    
    private static function checkTimeout(): void {
        $timeout = 1800; // 30 minutes of inactivity
        
        if (isset($_SESSION['_last_activity'])) {
            if (time() - $_SESSION['_last_activity'] > $timeout) {
                static::destroy();
                static::start();
                return;
            }
        }
        
        $_SESSION['_last_activity'] = time();
    }
    
    private static function periodicRegenerate(int $interval = 300): void {
        if (!isset($_SESSION['_regen_time'])) {
            $_SESSION['_regen_time'] = time();
            return;
        }
        
        if (time() - $_SESSION['_regen_time'] > $interval) {
            session_regenerate_id(true);
            $_SESSION['_regen_time'] = time();
        }
    }
    
    public static function get(string $key, mixed $default = null): mixed {
        static::start();
        
        // Support dot notation: 'user.profile.name'
        $keys = explode('.', $key);
        $data = $_SESSION;
        
        foreach ($keys as $k) {
            if (!isset($data[$k])) return $default;
            $data = $data[$k];
        }
        
        return $data;
    }
    
    public static function set(string $key, mixed $value): void {
        static::start();
        
        $keys = explode('.', $key);
        $data = &$_SESSION;
        
        foreach ($keys as $i => $k) {
            if ($i === count($keys) - 1) {
                $data[$k] = $value;
            } else {
                if (!isset($data[$k]) || !is_array($data[$k])) {
                    $data[$k] = [];
                }
                $data = &$data[$k];
            }
        }
    }
    
    public static function has(string $key): bool {
        return static::get($key) !== null;
    }
    
    public static function forget(string $key): void {
        static::start();
        
        $keys = explode('.', $key);
        $data = &$_SESSION;
        
        foreach ($keys as $i => $k) {
            if ($i === count($keys) - 1) {
                unset($data[$k]);
            } elseif (isset($data[$k])) {
                $data = &$data[$k];
            }
        }
    }
    
    public static function flash(string $key, mixed $value = null): mixed {
        static::start();
        
        if ($value !== null) {
            // Set flash message
            $_SESSION['_flash'][$key] = $value;
            return null;
        }
        
        // Get and remove flash message
        $data = $_SESSION['_flash'][$key] ?? null;
        unset($_SESSION['_flash'][$key]);
        return $data;
    }
    
    public static function reflash(): void {
        // Keep all flash messages for one more request
        $_SESSION['_reflash'] = true;
    }
    
    public static function destroy(): void {
        static::start();
        
        // Clear session data
        $_SESSION = [];
        
        // Delete cookie
        if (isset($_COOKIE[session_name()])) {
            $params = session_get_cookie_params();
            setcookie(
                session_name(), '', time() - 42000,
                $params['path'], $params['domain'],
                $params['secure'], $params['httponly']
            );
        }
        
        session_destroy();
        self::$started = false;
    }
    
    public static function all(): array {
        static::start();
        return $_SESSION;
    }
    
    public static function id(): string {
        static::start();
        return session_id();
    }
}

// ========================
// Usage
// ========================
Session::start();

// Login
Session::set('auth.user_id', 42);
Session::set('auth.username', 'somchai');
Session::set('auth.role', 'admin');
Session::set('auth.logged_in', true);

// Get nested
$userId = Session::get('auth.user_id');
$isLoggedIn = Session::get('auth.logged_in', false);

// Flash messages
Session::flash('success', 'บันทึกข้อมูลสำเร็จ!');
Session::flash('error', 'เกิดข้อผิดพลาด!');

// Next request
$message = Session::flash('success'); // Gets and removes

// Logout
if (isset($_GET['logout'])) {
    Session::destroy();
    header('Location: /login');
    exit;
}
```

---

## ขั้นตอนที่ 243: PHP Cookies

```php
<?php
declare(strict_types=1);

class Cookie {
    public static function set(
        string $name,
        string $value,
        int $expires = 0,
        string $path = '/',
        string $domain = '',
        bool $secure = true,
        bool $httpOnly = true,
        string $sameSite = 'Strict'
    ): bool {
        return setcookie($name, $value, [
            'expires' => $expires,
            'path' => $path,
            'domain' => $domain,
            'secure' => $secure,
            'httponly' => $httpOnly,
            'samesite' => $sameSite,
        ]);
    }
    
    public static function get(string $name, mixed $default = null): mixed {
        return $_COOKIE[$name] ?? $default;
    }
    
    public static function has(string $name): bool {
        return isset($_COOKIE[$name]);
    }
    
    public static function delete(string $name): void {
        static::set($name, '', time() - 3600);
        unset($_COOKIE[$name]);
    }
    
    // Encrypted cookie
    public static function setEncrypted(string $name, mixed $value, int $expires = 0, string $secret = ''): bool {
        $data = json_encode($value);
        $encrypted = static::encrypt($data, $secret);
        return static::set($name, base64_encode($encrypted), $expires);
    }
    
    public static function getEncrypted(string $name, mixed $default = null, string $secret = ''): mixed {
        $encoded = static::get($name);
        if ($encoded === null) return $default;
        
        $encrypted = base64_decode($encoded);
        $decrypted = static::decrypt($encrypted, $secret);
        
        if ($decrypted === false) return $default;
        
        return json_decode($decrypted, true);
    }
    
    private static function encrypt(string $data, string $key): string {
        $key = hash('sha256', $key, true);
        $iv = random_bytes(16);
        $encrypted = openssl_encrypt($data, 'AES-256-CBC', $key, 0, $iv);
        return $iv . $encrypted;
    }
    
    private static function decrypt(string $data, string $key): string|false {
        $key = hash('sha256', $key, true);
        $iv = substr($data, 0, 16);
        $encrypted = substr($data, 16);
        return openssl_decrypt($encrypted, 'AES-256-CBC', $key, 0, $iv);
    }
    
    // Remember Me Token
    public static function setRememberToken(int $userId, string $token, int $days = 30): void {
        $expires = time() + ($days * 24 * 60 * 60);
        $value = base64_encode(json_encode(['user_id' => $userId, 'token' => $token]));
        static::set('remember_token', $value, $expires);
    }
    
    public static function getRememberToken(): ?array {
        $value = static::get('remember_token');
        if (!$value) return null;
        
        $data = json_decode(base64_decode($value), true);
        return $data;
    }
}

// ========================
// Remember Me Authentication
// ========================
class RememberMe {
    private \PDO $pdo;
    
    public function __construct(\PDO $pdo) {
        $this->pdo = $pdo;
    }
    
    public function remember(int $userId): void {
        // Generate secure token
        $selector = bin2hex(random_bytes(8));
        $validator = bin2hex(random_bytes(32));
        $token = $selector . ':' . $validator;
        
        // Hash the validator for storage
        $hashedValidator = hash('sha256', $validator);
        $expires = time() + (30 * 24 * 60 * 60); // 30 days
        
        // Store in database
        $stmt = $this->pdo->prepare(
            "INSERT INTO remember_tokens (user_id, selector, hashed_validator, expires) 
             VALUES (?, ?, ?, FROM_UNIXTIME(?))"
        );
        $stmt->execute([$userId, $selector, $hashedValidator, $expires]);
        
        // Set cookie
        Cookie::set('remember_token', $token, $expires);
    }
    
    public function check(): ?int {
        $token = Cookie::get('remember_token');
        if (!$token) return null;
        
        [$selector, $validator] = explode(':', $token, 2);
        
        // Find token by selector
        $stmt = $this->pdo->prepare(
            "SELECT * FROM remember_tokens 
             WHERE selector = ? AND expires > NOW()"
        );
        $stmt->execute([$selector]);
        $stored = $stmt->fetch();
        
        if (!$stored) return null;
        
        // Verify validator
        if (!hash_equals($stored['hashed_validator'], hash('sha256', $validator))) {
            return null;
        }
        
        return (int) $stored['user_id'];
    }
    
    public function forget(): void {
        $token = Cookie::get('remember_token');
        
        if ($token && str_contains($token, ':')) {
            [$selector] = explode(':', $token, 2);
            $stmt = $this->pdo->prepare("DELETE FROM remember_tokens WHERE selector = ?");
            $stmt->execute([$selector]);
        }
        
        Cookie::delete('remember_token');
    }
}
```

---

## ขั้นตอนที่ 244: Authentication System

```php
<?php
declare(strict_types=1);

class Auth {
    private static ?array $user = null;
    private static ?\PDO $pdo = null;
    
    public static function setPdo(\PDO $pdo): void {
        self::$pdo = $pdo;
    }
    
    public static function attempt(string $email, string $password, bool $remember = false): bool {
        $stmt = self::$pdo->prepare(
            "SELECT id, email, password, name, role, active FROM users WHERE email = ?"
        );
        $stmt->execute([$email]);
        $user = $stmt->fetch(\PDO::FETCH_ASSOC);
        
        if (!$user) return false;
        if (!$user['active']) return false;
        if (!password_verify($password, $user['password'])) {
            // Rate limiting
            static::incrementLoginAttempts($email);
            return false;
        }
        
        // Clear failed attempts
        static::clearLoginAttempts($email);
        
        // Login
        static::login($user, $remember);
        
        return true;
    }
    
    private static function login(array $user, bool $remember = false): void {
        Session::start();
        session_regenerate_id(true);
        
        // Store user in session
        Session::set('auth', [
            'user_id' => $user['id'],
            'email' => $user['email'],
            'name' => $user['name'],
            'role' => $user['role'],
        ]);
        
        self::$user = $user;
        
        // Update last login
        $stmt = self::$pdo->prepare("UPDATE users SET last_login = NOW() WHERE id = ?");
        $stmt->execute([$user['id']]);
        
        // Remember Me
        if ($remember) {
            $rememberMe = new RememberMe(self::$pdo);
            $rememberMe->remember($user['id']);
        }
    }
    
    public static function check(): bool {
        Session::start();
        
        if (Session::get('auth.user_id')) {
            return true;
        }
        
        // Check remember token
        $rememberMe = new RememberMe(self::$pdo);
        $userId = $rememberMe->check();
        
        if ($userId) {
            $stmt = self::$pdo->prepare("SELECT * FROM users WHERE id = ? AND active = 1");
            $stmt->execute([$userId]);
            $user = $stmt->fetch(\PDO::FETCH_ASSOC);
            
            if ($user) {
                static::login($user);
                return true;
            }
        }
        
        return false;
    }
    
    public static function user(): ?array {
        if (self::$user !== null) return self::$user;
        
        $userId = Session::get('auth.user_id');
        if (!$userId) return null;
        
        $stmt = self::$pdo->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$userId]);
        self::$user = $stmt->fetch(\PDO::FETCH_ASSOC) ?: null;
        
        return self::$user;
    }
    
    public static function id(): ?int {
        return Session::get('auth.user_id');
    }
    
    public static function logout(): void {
        $rememberMe = new RememberMe(self::$pdo);
        $rememberMe->forget();
        
        Session::destroy();
        self::$user = null;
    }
    
    public static function hasRole(string ...$roles): bool {
        $userRole = Session::get('auth.role');
        return in_array($userRole, $roles);
    }
    
    public static function guest(): bool {
        return !static::check();
    }
    
    // Rate Limiting
    private static function incrementLoginAttempts(string $email): void {
        $key = 'login_attempts_' . md5($email);
        $attempts = (int) Session::get($key, 0);
        Session::set($key, $attempts + 1);
        
        if ($attempts >= 5) {
            Session::set('login_locked_until', time() + 900); // 15 min
        }
    }
    
    private static function clearLoginAttempts(string $email): void {
        $key = 'login_attempts_' . md5($email);
        Session::forget($key);
        Session::forget('login_locked_until');
    }
    
    public static function isLocked(): bool {
        $lockedUntil = Session::get('login_locked_until');
        return $lockedUntil !== null && time() < $lockedUntil;
    }
}

// ========================
// Middleware
// ========================
function requireAuth(): void {
    if (!Auth::check()) {
        $returnUrl = urlencode($_SERVER['REQUEST_URI']);
        header("Location: /login?redirect={$returnUrl}");
        exit;
    }
}

function requireRole(string ...$roles): void {
    requireAuth();
    
    if (!Auth::hasRole(...$roles)) {
        http_response_code(403);
        include '403.php';
        exit;
    }
}

// Usage in pages
requireAuth();       // ต้อง login ก่อน
requireRole('admin'); // ต้องเป็น admin
```

---

## 🎯 สรุป Part 10

| หัวข้อ | สิ่งที่สำคัญ |
|--------|-------------|
| Session | session_start, $_SESSION, session_regenerate_id |
| Session Manager | Dot notation, Flash messages, Timeout |
| Cookies | setcookie, secure flags, encryption |
| Remember Me | Selector-validator pattern |
| Auth System | Password verify, rate limiting, roles |

**ถัดไป → Part 11: OOP Basics** (เรียนแล้ว) **→ Part 12: Design Patterns**
