# Part 40: Advanced Security

## ขั้นตอนที่ 1061-1090: ความปลอดภัยขั้นสูงสำหรับ PHP

การรักษาความปลอดภัยเป็นสิ่งสำคัญที่สุดในการพัฒนาแอปพลิเคชัน บทนี้ครอบคลุม OWASP Top 10, การเข้ารหัส, JWT, OAuth2 และการป้องกันภัยคุกคามต่างๆ

---

## ขั้นตอนที่ 1061: OWASP Top 10 ใน PHP

### 1. SQL Injection Prevention

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use PDO;
use PDOStatement;

class SecureUserRepository
{
    public function __construct(private PDO $db) {}

    // WRONG - SQL Injection vulnerability
    public function findByEmailUnsafe(string $email): array
    {
        $query = "SELECT * FROM users WHERE email = '$email'";
        return $this->db->query($query)->fetchAll();
    }

    // CORRECT - Prepared statements
    public function findByEmail(string $email): ?array
    {
        $stmt = $this->db->prepare('SELECT * FROM users WHERE email = :email');
        $stmt->execute([':email' => $email]);

        $result = $stmt->fetch(PDO::FETCH_ASSOC);
        return $result ?: null;
    }

    // CORRECT - With multiple conditions
    public function searchUsers(string $name, string $role, int $minAge): array
    {
        $stmt = $this->db->prepare(
            'SELECT * FROM users 
             WHERE name LIKE :name 
               AND role = :role 
               AND age >= :min_age'
        );

        $stmt->execute([
            ':name'    => "%{$name}%",
            ':role'    => $role,
            ':min_age' => $minAge,
        ]);

        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

### 2. XSS Prevention

```php
<?php

declare(strict_types=1);

namespace App\Security;

class XssPrevention
{
    // Escape output
    public static function escape(string $value): string
    {
        return htmlspecialchars($value, ENT_QUOTES | ENT_HTML5, 'UTF-8');
    }

    // Sanitize HTML with allowed tags
    public static function sanitizeHtml(string $html): string
    {
        $allowedTags = '<p><br><strong><em><ul><ol><li><h1><h2><h3><a>';

        // Remove all tags except allowed
        $clean = strip_tags($html, $allowedTags);

        // Remove javascript: from href
        $clean = preg_replace('/javascript:/i', '', $clean);

        // Remove event handlers
        $clean = preg_replace('/\s+on\w+\s*=/i', '', $clean);

        return $clean;
    }

    // Content Security Policy header
    public static function getCspHeader(): string
    {
        return implode('; ', [
            "default-src 'self'",
            "script-src 'self' 'nonce-" . self::generateNonce() . "'",
            "style-src 'self' 'unsafe-inline'",
            "img-src 'self' data: https:",
            "font-src 'self' https://fonts.gstatic.com",
            "connect-src 'self'",
            "frame-ancestors 'none'",
        ]);
    }

    private static function generateNonce(): string
    {
        return base64_encode(random_bytes(16));
    }
}
```

---

## ขั้นตอนที่ 1062: AES-256-GCM Encryption

```php
<?php

declare(strict_types=1);

namespace App\Security;

class AesEncryption
{
    private const CIPHER    = 'aes-256-gcm';
    private const KEY_SIZE  = 32; // 256 bits
    private const IV_SIZE   = 12; // 96 bits for GCM
    private const TAG_SIZE  = 16; // 128 bits

    public function __construct(private string $key)
    {
        if (strlen($key) !== self::KEY_SIZE) {
            throw new \InvalidArgumentException(
                'Key must be exactly 32 bytes (256 bits)'
            );
        }
    }

    public function encrypt(string $plaintext): string
    {
        $iv = random_bytes(self::IV_SIZE);
        $tag = '';

        $ciphertext = openssl_encrypt(
            $plaintext,
            self::CIPHER,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag,
            '',
            self::TAG_SIZE
        );

        if ($ciphertext === false) {
            throw new \RuntimeException('Encryption failed');
        }

        // Format: iv (12 bytes) + tag (16 bytes) + ciphertext
        return base64_encode($iv . $tag . $ciphertext);
    }

    public function decrypt(string $encryptedData): string
    {
        $data = base64_decode($encryptedData, true);

        if ($data === false) {
            throw new \InvalidArgumentException('Invalid base64 data');
        }

        $minLength = self::IV_SIZE + self::TAG_SIZE;
        if (strlen($data) <= $minLength) {
            throw new \InvalidArgumentException('Data too short');
        }

        $iv         = substr($data, 0, self::IV_SIZE);
        $tag        = substr($data, self::IV_SIZE, self::TAG_SIZE);
        $ciphertext = substr($data, $minLength);

        $plaintext = openssl_decrypt(
            $ciphertext,
            self::CIPHER,
            $this->key,
            OPENSSL_RAW_DATA,
            $iv,
            $tag
        );

        if ($plaintext === false) {
            throw new \RuntimeException('Decryption failed - data may be tampered');
        }

        return $plaintext;
    }

    public static function generateKey(): string
    {
        return random_bytes(self::KEY_SIZE);
    }

    public static function generateKeyHex(): string
    {
        return bin2hex(self::generateKey());
    }
}
```

```php
<?php

declare(strict_types=1);

// การใช้งาน AES Encryption
$key = hex2bin(env('ENCRYPTION_KEY')); // 64 hex chars = 32 bytes
$encryptor = new AesEncryption($key);

// เข้ารหัสข้อมูลสำคัญ
$sensitiveData = json_encode([
    'card_number' => '4111111111111111',
    'cvv'         => '123',
    'expiry'      => '12/25',
]);

$encrypted = $encryptor->encrypt($sensitiveData);
echo "Encrypted: " . $encrypted . PHP_EOL;

// ถอดรหัส
$decrypted = $encryptor->decrypt($encrypted);
$data = json_decode($decrypted, true);
echo "Card: " . $data['card_number'] . PHP_EOL;

// Field-level encryption in Eloquent
use Illuminate\Database\Eloquent\Model;

class CreditCard extends Model
{
    protected $fillable = ['user_id', 'card_number_encrypted', 'last_four', 'expiry'];

    public function setCardNumberAttribute(string $cardNumber): void
    {
        $this->attributes['card_number_encrypted'] = app(AesEncryption::class)
            ->encrypt($cardNumber);
        $this->attributes['last_four'] = substr($cardNumber, -4);
    }

    public function getCardNumberAttribute(): ?string
    {
        if (isset($this->attributes['card_number_encrypted'])) {
            return app(AesEncryption::class)
                ->decrypt($this->attributes['card_number_encrypted']);
        }
        return null;
    }
}
```

---

## ขั้นตอนที่ 1063: JWT Authentication ด้วย RS256

```php
<?php

declare(strict_types=1);

namespace App\Auth;

use Firebase\JWT\JWT;
use Firebase\JWT\Key;

class JwtService
{
    private const ALGORITHM   = 'RS256';
    private const ACCESS_TTL  = 900;       // 15 minutes
    private const REFRESH_TTL = 2592000;   // 30 days

    public function __construct(
        private string $privateKey,
        private string $publicKey,
        private string $issuer = 'https://api.example.com'
    ) {}

    public function generateAccessToken(array $payload): string
    {
        $now = time();

        return JWT::encode([
            'iss'  => $this->issuer,
            'iat'  => $now,
            'exp'  => $now + self::ACCESS_TTL,
            'nbf'  => $now,
            'jti'  => bin2hex(random_bytes(16)),
            'type' => 'access',
            ...$payload,
        ], $this->privateKey, self::ALGORITHM);
    }

    public function generateRefreshToken(int $userId): string
    {
        $now = time();

        return JWT::encode([
            'iss'     => $this->issuer,
            'iat'     => $now,
            'exp'     => $now + self::REFRESH_TTL,
            'jti'     => bin2hex(random_bytes(16)),
            'type'    => 'refresh',
            'user_id' => $userId,
        ], $this->privateKey, self::ALGORITHM);
    }

    public function verifyToken(string $token): object
    {
        try {
            $decoded = JWT::decode($token, new Key($this->publicKey, self::ALGORITHM));

            if ($decoded->exp < time()) {
                throw new \RuntimeException('Token expired');
            }

            return $decoded;
        } catch (\Exception $e) {
            throw new \App\Exceptions\InvalidTokenException(
                'Invalid token: ' . $e->getMessage()
            );
        }
    }

    public function refreshTokens(string $refreshToken): array
    {
        $decoded = $this->verifyToken($refreshToken);

        if ($decoded->type !== 'refresh') {
            throw new \App\Exceptions\InvalidTokenException('Not a refresh token');
        }

        $user = \App\Models\User::findOrFail($decoded->user_id);

        return [
            'access_token'  => $this->generateAccessToken([
                'sub'   => $user->id,
                'email' => $user->email,
                'roles' => $user->roles->pluck('name')->toArray(),
            ]),
            'refresh_token' => $this->generateRefreshToken($user->id),
        ];
    }

    public static function generateKeyPair(): array
    {
        $config = [
            'digest_alg'       => 'sha256',
            'private_key_bits' => 2048,
            'private_key_type' => OPENSSL_KEYTYPE_RSA,
        ];

        $res = openssl_pkey_new($config);
        openssl_pkey_export($res, $privateKey);

        $publicKeyDetails = openssl_pkey_get_details($res);
        $publicKey = $publicKeyDetails['key'];

        return [
            'private_key' => $privateKey,
            'public_key'  => $publicKey,
        ];
    }
}
```

---

## ขั้นตอนที่ 1064: OAuth2/OIDC Integration

```php
<?php

declare(strict_types=1);

namespace App\Auth;

use GuzzleHttp\Client;
use GuzzleHttp\Exception\RequestException;

class OAuth2Client
{
    public function __construct(
        private string $clientId,
        private string $clientSecret,
        private string $redirectUri,
        private string $authorizationEndpoint,
        private string $tokenEndpoint,
        private string $userInfoEndpoint,
        private Client $httpClient
    ) {}

    public function getAuthorizationUrl(
        array $scopes = ['openid', 'profile', 'email'],
        string $state = ''
    ): string {
        $state = $state ?: bin2hex(random_bytes(16));

        // บันทึก state ลง session เพื่อป้องกัน CSRF
        session(['oauth_state' => $state]);

        $params = http_build_query([
            'response_type' => 'code',
            'client_id'     => $this->clientId,
            'redirect_uri'  => $this->redirectUri,
            'scope'         => implode(' ', $scopes),
            'state'         => $state,
        ]);

        return "{$this->authorizationEndpoint}?{$params}";
    }

    public function exchangeCodeForTokens(string $code, string $state): array
    {
        // ตรวจสอบ state เพื่อป้องกัน CSRF
        if ($state !== session('oauth_state')) {
            throw new \RuntimeException('Invalid state parameter - possible CSRF attack');
        }

        session()->forget('oauth_state');

        try {
            $response = $this->httpClient->post($this->tokenEndpoint, [
                'form_params' => [
                    'grant_type'    => 'authorization_code',
                    'code'          => $code,
                    'redirect_uri'  => $this->redirectUri,
                    'client_id'     => $this->clientId,
                    'client_secret' => $this->clientSecret,
                ],
            ]);

            return json_decode($response->getBody()->getContents(), true);
        } catch (RequestException $e) {
            throw new \RuntimeException('Failed to exchange code: ' . $e->getMessage());
        }
    }

    public function getUserInfo(string $accessToken): array
    {
        $response = $this->httpClient->get($this->userInfoEndpoint, [
            'headers' => ['Authorization' => "Bearer {$accessToken}"],
        ]);

        return json_decode($response->getBody()->getContents(), true);
    }

    public function refreshAccessToken(string $refreshToken): array
    {
        $response = $this->httpClient->post($this->tokenEndpoint, [
            'form_params' => [
                'grant_type'    => 'refresh_token',
                'refresh_token' => $refreshToken,
                'client_id'     => $this->clientId,
                'client_secret' => $this->clientSecret,
            ],
        ]);

        return json_decode($response->getBody()->getContents(), true);
    }
}
```

---

## ขั้นตอนที่ 1065: Rate Limiting และ Brute Force Protection

```php
<?php

declare(strict_types=1);

namespace App\Security;

use Illuminate\Cache\RateLimiter;
use Illuminate\Http\Request;

class BruteForceProtection
{
    private const MAX_ATTEMPTS      = 5;
    private const DECAY_MINUTES     = 15;
    private const LOCKOUT_MINUTES   = 60;

    public function __construct(private RateLimiter $limiter) {}

    public function attempt(string $key, callable $callback): mixed
    {
        if ($this->isLockedOut($key)) {
            $seconds = $this->limiter->availableIn($key);
            throw new \App\Exceptions\TooManyAttemptsException(
                "Too many attempts. Try again in {$seconds} seconds."
            );
        }

        try {
            $result = $callback();
            $this->limiter->clear($key);
            return $result;
        } catch (\App\Exceptions\AuthenticationException $e) {
            $this->recordFailedAttempt($key);
            throw $e;
        }
    }

    public function recordFailedAttempt(string $key): void
    {
        $this->limiter->hit($key, self::DECAY_MINUTES * 60);

        if ($this->limiter->attempts($key) >= self::MAX_ATTEMPTS) {
            // บล็อก IP เพิ่มเติม
            $this->lockoutKey($key);
        }
    }

    public function isLockedOut(string $key): bool
    {
        return $this->limiter->tooManyAttempts($key, self::MAX_ATTEMPTS);
    }

    private function lockoutKey(string $key): void
    {
        cache()->put(
            "lockout:{$key}",
            true,
            now()->addMinutes(self::LOCKOUT_MINUTES)
        );
    }

    public static function getLoginKey(Request $request): string
    {
        return 'login:' . sha1(
            $request->input('email') . '|' . $request->ip()
        );
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Cache\RateLimiter;
use Symfony\Component\HttpFoundation\Response;

class ApiRateLimit
{
    public function __construct(private RateLimiter $limiter) {}

    public function handle(Request $request, Closure $next, string $limiterName = 'api'): Response
    {
        $key = $this->resolveRequestSignature($request, $limiterName);

        $maxAttempts = match ($limiterName) {
            'api'   => 60,   // 60 requests per minute
            'auth'  => 5,    // 5 attempts per minute
            'heavy' => 10,   // 10 requests per minute
            default => 60,
        };

        if ($this->limiter->tooManyAttempts($key, $maxAttempts)) {
            return $this->buildRateLimitResponse($key, $maxAttempts);
        }

        $this->limiter->hit($key, 60);

        $response = $next($request);

        return $this->addRateLimitHeaders(
            $response,
            $key,
            $maxAttempts
        );
    }

    private function resolveRequestSignature(Request $request, string $name): string
    {
        $user = $request->user();

        if ($user) {
            return sha1("{$name}|user:{$user->id}");
        }

        return sha1("{$name}|ip:{$request->ip()}");
    }

    private function buildRateLimitResponse(string $key, int $maxAttempts): Response
    {
        $retryAfter = $this->limiter->availableIn($key);

        return response()->json([
            'error'   => 'Too Many Requests',
            'message' => "Rate limit exceeded. Retry after {$retryAfter} seconds.",
        ], 429)->withHeaders([
            'Retry-After'       => $retryAfter,
            'X-RateLimit-Limit' => $maxAttempts,
        ]);
    }

    private function addRateLimitHeaders(
        Response $response,
        string $key,
        int $maxAttempts
    ): Response {
        $remaining = $maxAttempts - $this->limiter->attempts($key);

        $response->headers->set('X-RateLimit-Limit', (string)$maxAttempts);
        $response->headers->set('X-RateLimit-Remaining', (string)max(0, $remaining));

        return $response;
    }
}
```

---

## ขั้นตอนที่ 1066: Content Security Policy Headers

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        // Content Security Policy
        $nonce = base64_encode(random_bytes(16));
        $response->headers->set('Content-Security-Policy', $this->buildCsp($nonce));

        // Other security headers
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Referrer-Policy', 'strict-origin-when-cross-origin');
        $response->headers->set(
            'Permissions-Policy',
            'camera=(), microphone=(), geolocation=(self)'
        );
        $response->headers->set(
            'Strict-Transport-Security',
            'max-age=31536000; includeSubDomains; preload'
        );

        // CORS
        if (config('app.env') === 'production') {
            $response->headers->set(
                'Access-Control-Allow-Origin',
                config('app.url')
            );
        }

        return $response;
    }

    private function buildCsp(string $nonce): string
    {
        $policies = [
            "default-src 'self'",
            "script-src 'self' 'nonce-{$nonce}' https://cdn.example.com",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
            "img-src 'self' data: https: blob:",
            "font-src 'self' https://fonts.gstatic.com",
            "connect-src 'self' https://api.example.com wss://ws.example.com",
            "media-src 'self'",
            "object-src 'none'",
            "frame-src 'none'",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'",
            "upgrade-insecure-requests",
        ];

        return implode('; ', $policies);
    }
}
```

---

## ขั้นตอนที่ 1067: Secure File Upload Handling

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;
use Intervention\Image\ImageManager;

class SecureFileUploadService
{
    private const ALLOWED_MIMES = [
        'image/jpeg', 'image/png', 'image/gif', 'image/webp',
    ];

    private const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

    private const DANGEROUS_EXTENSIONS = [
        'php', 'php3', 'php4', 'php5', 'phtml',
        'exe', 'sh', 'bat', 'cmd', 'ps1',
        'js', 'html', 'htm', 'svg',
    ];

    public function upload(UploadedFile $file, string $disk = 'private'): string
    {
        $this->validateFile($file);

        // สร้างชื่อไฟล์ที่ปลอดภัย (ไม่ใช้ชื่อเดิม)
        $safeFilename = $this->generateSafeFilename($file);

        // Strip metadata จากรูปภาพ
        if (str_starts_with($file->getMimeType(), 'image/')) {
            $processedPath = $this->processImage($file);
            return Storage::disk($disk)->putFileAs('uploads', $processedPath, $safeFilename);
        }

        return Storage::disk($disk)->putFileAs('uploads', $file, $safeFilename);
    }

    private function validateFile(UploadedFile $file): void
    {
        // ตรวจสอบขนาด
        if ($file->getSize() > self::MAX_FILE_SIZE) {
            throw new \App\Exceptions\FileUploadException(
                'File size exceeds maximum allowed size of 5MB'
            );
        }

        // ตรวจสอบ MIME type จาก content (ไม่ใช้จาก extension)
        $detectedMime = mime_content_type($file->getRealPath());
        if (!in_array($detectedMime, self::ALLOWED_MIMES, true)) {
            throw new \App\Exceptions\FileUploadException(
                "File type '{$detectedMime}' is not allowed"
            );
        }

        // ตรวจสอบ extension
        $extension = strtolower($file->getClientOriginalExtension());
        if (in_array($extension, self::DANGEROUS_EXTENSIONS, true)) {
            throw new \App\Exceptions\FileUploadException(
                "File extension '.{$extension}' is not allowed"
            );
        }

        // ตรวจสอบว่าเป็นรูปภาพจริงๆ ด้วย GD
        if (str_starts_with($detectedMime, 'image/')) {
            $imageInfo = getimagesize($file->getRealPath());
            if ($imageInfo === false) {
                throw new \App\Exceptions\FileUploadException('File is not a valid image');
            }
        }
    }

    private function generateSafeFilename(UploadedFile $file): string
    {
        $extension = match ($file->getMimeType()) {
            'image/jpeg' => 'jpg',
            'image/png'  => 'png',
            'image/gif'  => 'gif',
            'image/webp' => 'webp',
            default      => 'bin',
        };

        return bin2hex(random_bytes(16)) . '.' . $extension;
    }

    private function processImage(UploadedFile $file): string
    {
        $manager = new ImageManager(['driver' => 'gd']);
        $image = $manager->make($file->getRealPath());

        // Strip EXIF data (อาจมีข้อมูล GPS)
        $image->strip();

        // Resize ถ้าใหญ่เกินไป
        if ($image->width() > 2048 || $image->height() > 2048) {
            $image->resize(2048, 2048, function ($constraint) {
                $constraint->aspectRatio();
                $constraint->upsize();
            });
        }

        $tempPath = sys_get_temp_dir() . '/' . uniqid('upload_', true) . '.jpg';
        $image->save($tempPath, 85); // 85% quality JPEG

        return $tempPath;
    }
}
```

---

## ขั้นตอนที่ 1068: CSRF Protection

```php
<?php

declare(strict_types=1);

namespace App\Security;

class CsrfProtection
{
    private const TOKEN_LENGTH = 32;
    private const SESSION_KEY  = '_csrf_token';

    public function generateToken(): string
    {
        $token = bin2hex(random_bytes(self::TOKEN_LENGTH));
        session([self::SESSION_KEY => $token]);
        return $token;
    }

    public function validateToken(string $token): bool
    {
        $storedToken = session(self::SESSION_KEY);

        if (!$storedToken) {
            return false;
        }

        // ใช้ hash_equals เพื่อป้องกัน timing attacks
        return hash_equals($storedToken, $token);
    }

    public function rotateToken(): string
    {
        session()->forget(self::SESSION_KEY);
        return $this->generateToken();
    }
}
```

---

## ขั้นตอนที่ 1069: Password Security

```php
<?php

declare(strict_types=1);

namespace App\Security;

class PasswordSecurity
{
    private const MIN_ENTROPY_BITS = 50;

    public function hash(string $password): string
    {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,  // 64MB
            'time_cost'   => 4,      // 4 iterations
            'threads'     => 2,      // 2 threads
        ]);
    }

    public function verify(string $password, string $hash): bool
    {
        return password_verify($password, $hash);
    }

    public function needsRehash(string $hash): bool
    {
        return password_needs_rehash($hash, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost'   => 4,
            'threads'     => 2,
        ]);
    }

    public function validateStrength(string $password): array
    {
        $errors = [];

        if (strlen($password) < 12) {
            $errors[] = 'Password must be at least 12 characters';
        }

        if (!preg_match('/[A-Z]/', $password)) {
            $errors[] = 'Password must contain uppercase letter';
        }

        if (!preg_match('/[a-z]/', $password)) {
            $errors[] = 'Password must contain lowercase letter';
        }

        if (!preg_match('/[0-9]/', $password)) {
            $errors[] = 'Password must contain a number';
        }

        if (!preg_match('/[!@#$%^&*()_+\-=\[\]{}|;:,.<>?]/', $password)) {
            $errors[] = 'Password must contain a special character';
        }

        // ตรวจสอบ entropy
        if ($this->calculateEntropy($password) < self::MIN_ENTROPY_BITS) {
            $errors[] = 'Password is too predictable';
        }

        return $errors;
    }

    private function calculateEntropy(string $password): float
    {
        $uniqueChars = count(array_unique(str_split($password)));
        return strlen($password) * log($uniqueChars, 2);
    }

    // ตรวจสอบกับ HaveIBeenPwned
    public function isCompromised(string $password): bool
    {
        $hash   = strtoupper(sha1($password));
        $prefix = substr($hash, 0, 5);
        $suffix = substr($hash, 5);

        $response = file_get_contents(
            "https://api.pwnedpasswords.com/range/{$prefix}"
        );

        foreach (explode("\n", $response) as $line) {
            [$hashSuffix] = explode(':', $line);
            if (trim($hashSuffix) === $suffix) {
                return true;
            }
        }

        return false;
    }
}
```

---

## สรุปบทที่ 40

| หัวข้อ | เทคนิค | ความสำคัญ |
|--------|--------|-----------|
| SQL Injection | Prepared Statements | Critical |
| XSS | htmlspecialchars, CSP | Critical |
| AES-256-GCM | OpenSSL | High |
| JWT RS256 | firebase/php-jwt | High |
| OAuth2/OIDC | GuzzleHttp | High |
| Rate Limiting | Cache/RateLimiter | High |
| File Upload | MIME validation, Strip EXIF | High |
| CSRF | Token generation, hash_equals | High |

**ต่อไป**: Part 41 - Laravel Advanced

---
