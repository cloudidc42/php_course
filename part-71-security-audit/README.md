# Part 71: Security Audit

## ขั้นตอนที่ 1991-2020: Security Scanning และ Audit

Security audit ช่วยค้นหาช่องโหว่ก่อนที่ attackers จะเจอ

---

## ขั้นตอนที่ 1991: Dependency Vulnerability Scanning

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6am

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'

      - name: Install dependencies
        run: composer install --no-interaction

      # Composer audit
      - name: Composer audit
        run: composer audit --format=json > audit-results.json || true

      # Symfony Security Checker
      - name: Security Checker
        run: |
          curl -H "Accept: application/json" https://security.symfony.com/check_lock \
            -d lock="$(cat composer.lock)" | python3 -m json.tool

      # Snyk PHP scan
      - name: Snyk PHP vulnerability scan
        uses: snyk/actions/php@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

      # npm audit for JS dependencies
      - name: npm audit
        run: |
          npm ci
          npm audit --audit-level=high

      - name: Upload audit results
        uses: actions/upload-artifact@v3
        with:
          name: security-audit
          path: audit-results.json
```

---

## ขั้นตอนที่ 1992: PHP Security Scanner

```bash
# ติดตั้ง tools
composer require --dev vimeo/psalm phpstan/phpstan squizlabs/php_codesniffer

# Psalm Security Analysis
./vendor/bin/psalm --taint-analysis

# PHPStan Security Level
./vendor/bin/phpstan analyse --level=max app/

# PHPCS Security
./vendor/bin/phpcs --standard=Generic app/ --sniffs=Generic.PHP.DisallowExec

# RIPS / SemGrep
semgrep --config=p/php-security app/
```

```php
<?php

declare(strict_types=1);

// psalm.xml - Security configuration
// เพิ่ม taint analysis

// app/Security/Sanitizer.php
namespace App\Security;

class Sanitizer
{
    /**
     * ป้องกัน SQL Injection
     */
    public static function sanitizeSql(string $input): string
    {
        // ใช้ prepared statements แทนการ sanitize เสมอ
        // นี่แค่ตัวอย่าง - ไม่ควรใช้ใน production
        return addslashes(strip_tags($input));
    }

    /**
     * ป้องกัน XSS
     */
    public static function sanitizeHtml(string $input, array $allowedTags = []): string
    {
        if (empty($allowedTags)) {
            return htmlspecialchars($input, ENT_QUOTES | ENT_HTML5, 'UTF-8');
        }

        // HTMLPurifier สำหรับ rich content
        $config = \HTMLPurifier_Config::createDefault();
        $config->set('HTML.AllowedElements', implode(',', $allowedTags));
        $config->set('HTML.AllowedAttributes', 'a.href,a.title,img.src,img.alt');
        $config->set('URI.AllowedSchemes', ['http' => true, 'https' => true]);

        $purifier = new \HTMLPurifier($config);
        return $purifier->purify($input);
    }

    /**
     * ป้องกัน Path Traversal
     */
    public static function sanitizePath(string $path, string $baseDir): string
    {
        $realBase = realpath($baseDir);
        $realPath = realpath($baseDir . '/' . $path);

        if ($realPath === false || !str_starts_with($realPath, $realBase)) {
            throw new \SecurityException("Path traversal detected: {$path}");
        }

        return $realPath;
    }

    /**
     * ป้องกัน Command Injection
     */
    public static function escapeShellCommand(string $command): string
    {
        return escapeshellcmd($command);
    }
}
```

---

## ขั้นตอนที่ 1993: Security Headers Audit

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/SecurityHeaders.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SecurityHeaders
{
    private array $headers = [
        // ป้องกัน clickjacking
        'X-Frame-Options' => 'SAMEORIGIN',

        // ป้องกัน MIME sniffing
        'X-Content-Type-Options' => 'nosniff',

        // XSS Protection (legacy browsers)
        'X-XSS-Protection' => '1; mode=block',

        // Referrer Policy
        'Referrer-Policy' => 'strict-origin-when-cross-origin',

        // Permissions Policy
        'Permissions-Policy' => 'camera=(), microphone=(), geolocation=(self), payment=(self)',

        // ลบ PHP version
        'X-Powered-By' => '',
    ];

    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        foreach ($this->headers as $header => $value) {
            if (empty($value)) {
                $response->headers->remove($header);
            } else {
                $response->headers->set($header, $value);
            }
        }

        // Content Security Policy
        if (!$request->is('admin/*')) {
            $response->headers->set(
                'Content-Security-Policy',
                $this->buildCsp()
            );
        }

        // HSTS (production only)
        if (app()->isProduction() && $request->isSecure()) {
            $response->headers->set(
                'Strict-Transport-Security',
                'max-age=31536000; includeSubDomains; preload'
            );
        }

        return $response;
    }

    private function buildCsp(): string
    {
        $directives = [
            "default-src 'self'",
            "script-src 'self' 'nonce-" . $this->getNonce() . "' cdn.jsdelivr.net",
            "style-src 'self' 'unsafe-inline' fonts.googleapis.com",
            "font-src 'self' fonts.gstatic.com",
            "img-src 'self' data: blob: https:",
            "connect-src 'self' wss: https:",
            "frame-ancestors 'self'",
            "base-uri 'self'",
            "form-action 'self'",
            "upgrade-insecure-requests",
        ];

        return implode('; ', $directives);
    }

    private function getNonce(): string
    {
        static $nonce;
        if (!$nonce) {
            $nonce = base64_encode(random_bytes(16));
            request()->attributes->set('csp_nonce', $nonce);
        }
        return $nonce;
    }
}
```

---

## ขั้นตอนที่ 1994: GDPR Compliance ใน PHP

```php
<?php

declare(strict_types=1);

// app/Services/GdprService.php
namespace App\Services;

use App\Models\User;
use App\Models\DataExport;
use Illuminate\Support\Facades\Storage;

class GdprService
{
    /**
     * Right to Access - ส่งข้อมูลทั้งหมดที่เรามีของ user
     */
    public function exportUserData(User $user): string
    {
        $data = [
            'profile'          => $user->toArray(),
            'orders'           => $user->orders()->with('items')->get()->toArray(),
            'comments'         => $user->comments()->get()->toArray(),
            'activities'       => $user->activityLogs()->latest()->limit(1000)->get()->toArray(),
            'login_history'    => $user->loginHistory()->latest()->limit(100)->get()->toArray(),
            'consents'         => $user->consents()->get()->toArray(),
            'exported_at'      => now()->toISOString(),
        ];

        $filename = "gdpr_export_{$user->id}_" . now()->format('YmdHis') . '.json';
        $path     = "gdpr/exports/{$user->id}/{$filename}";

        Storage::disk('private')->put($path, json_encode($data, JSON_PRETTY_PRINT));

        // Log export
        DataExport::create([
            'user_id'    => $user->id,
            'type'       => 'gdpr_access',
            'file_path'  => $path,
            'expires_at' => now()->addDays(30),
        ]);

        return Storage::disk('private')->temporaryUrl($path, now()->addHours(24));
    }

    /**
     * Right to Erasure - ลบข้อมูลของ user
     */
    public function eraseUserData(User $user, string $reason = ''): void
    {
        // Log erasure request
        \Log::channel('gdpr')->info("User erasure requested", [
            'user_id' => $user->id,
            'email'   => $user->email,
            'reason'  => $reason,
            'by'      => auth()->id(),
        ]);

        // Anonymize instead of hard delete (อาจจำเป็นสำหรับ order history)
        $user->update([
            'name'     => 'Anonymous User #' . $user->id,
            'email'    => 'deleted_' . $user->id . '@anonymized.com',
            'phone'    => null,
            'address'  => null,
            'deleted_at' => now(),
        ]);

        // ลบข้อมูลส่วนตัว
        $user->personalData()->delete();
        $user->profilePicture()->delete();
        $user->sessions()->delete();
        $user->tokens()->delete();

        // Anonymize comments แต่เก็บไว้ (เพื่อ content integrity)
        $user->comments()->update([
            'author_name' => 'Deleted User',
            'author_email' => null,
        ]);
    }

    /**
     * Consent Management
     */
    public function recordConsent(User $user, string $purpose, bool $granted): void
    {
        $user->consents()->updateOrCreate(
            ['purpose' => $purpose],
            [
                'granted'    => $granted,
                'ip_address' => request()->ip(),
                'user_agent' => request()->userAgent(),
                'granted_at' => $granted ? now() : null,
                'revoked_at' => !$granted ? now() : null,
            ]
        );
    }

    /**
     * Data Retention - ลบข้อมูลเก่าตาม retention policy
     */
    public function applyRetentionPolicy(): void
    {
        // ลบ logs เก่ากว่า 2 ปี
        \App\Models\ActivityLog::where('created_at', '<', now()->subYears(2))->delete();

        // ลบ sessions เก่ากว่า 30 วัน
        \DB::table('sessions')->where('last_activity', '<', now()->subDays(30)->timestamp)->delete();

        // Anonymize orders เก่ากว่า 7 ปี
        \App\Models\Order::where('created_at', '<', now()->subYears(7))
            ->update([
                'customer_name'    => null,
                'customer_email'   => null,
                'customer_address' => null,
                'ip_address'       => null,
            ]);
    }
}
```

---

## ขั้นตอนที่ 1995: Penetration Testing Basics

```php
<?php

declare(strict_types=1);

// app/Security/SecurityAuditLogger.php
namespace App\Security;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class SecurityAuditLogger
{
    public function logSuspiciousActivity(Request $request, string $type, array $context = []): void
    {
        Log::channel('security')->warning("Suspicious activity: {$type}", array_merge([
            'ip'         => $request->ip(),
            'user_id'    => $request->user()?->id,
            'user_agent' => $request->userAgent(),
            'path'       => $request->getPathInfo(),
            'method'     => $request->method(),
            'timestamp'  => now()->toISOString(),
        ], $context));
    }

    public function detectSqlInjection(string $input): bool
    {
        $patterns = [
            '/(\bSELECT\b|\bINSERT\b|\bUPDATE\b|\bDELETE\b|\bDROP\b|\bUNION\b)/i',
            '/(-{2}|\/\*|\*\/)/',
            '/\b(OR|AND)\b.+[=<>]/i',
            '/(\'|")(\s*)(;|\bOR\b|\bAND\b)/i',
        ];

        foreach ($patterns as $pattern) {
            if (preg_match($pattern, $input)) {
                return true;
            }
        }

        return false;
    }

    public function detectXss(string $input): bool
    {
        $patterns = [
            '/<script[^>]*>.*?<\/script>/si',
            '/javascript:/i',
            '/on\w+\s*=/i',
            '/<iframe/i',
            '/eval\(/i',
        ];

        foreach ($patterns as $pattern) {
            if (preg_match($pattern, $input)) {
                return true;
            }
        }

        return false;
    }
}
```

---

## ขั้นตอนที่ 1996: Secrets Management

```php
<?php

declare(strict_types=1);

// app/Services/SecretsManager.php
namespace App\Services;

use Aws\SecretsManager\SecretsManagerClient;

class SecretsManager
{
    private SecretsManagerClient $client;

    public function __construct()
    {
        $this->client = new SecretsManagerClient([
            'version' => 'latest',
            'region'  => config('services.aws.region'),
        ]);
    }

    public function get(string $secretName): array
    {
        $cacheKey = "secret:{$secretName}";

        return cache()->remember($cacheKey, 3600, function () use ($secretName) {
            $result = $this->client->getSecretValue([
                'SecretId' => $secretName,
            ]);

            return json_decode($result['SecretString'], true);
        });
    }

    public function rotate(string $secretName, array $newValues): void
    {
        $this->client->putSecretValue([
            'SecretId'     => $secretName,
            'SecretString' => json_encode($newValues),
        ]);

        cache()->forget("secret:{$secretName}");
    }
}
```

---

## สรุป Part 71

| Tool/Practice | วัตถุประสงค์ |
|--------------|------------|
| composer audit | PHP dependency CVE scan |
| Psalm taint analysis | Static code analysis |
| Security Headers | Protect browsers |
| CSP | ป้องกัน XSS/injection |
| GDPR | Privacy compliance |
| Secret Management | ไม่ hardcode secrets |
| Audit Logging | Track suspicious activity |

ถัดไป → Part 72: Laravel Dusk E2E Testing
