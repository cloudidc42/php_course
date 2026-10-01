# Part 79: Laravel SPA Authentication
## ขั้นตอนที่ 2231-2260: กลยุทธ์การ Authentication สำหรับ SPA

การเลือกกลยุทธ์ authentication ที่เหมาะสมสำหรับ SPA
ครอบคลุม Sanctum, Token-based, Social Login และ Multi-device

---

## ขั้นตอนที่ 2231: Sanctum Cookie-based Auth

```php
<?php
declare(strict_types=1);

// ติดตั้ง Laravel Sanctum
// composer require laravel/sanctum
// php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

// config/sanctum.php - ส่วนสำคัญ
return [
    'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', sprintf(
        '%s%s',
        'localhost,localhost:3000,127.0.0.1,127.0.0.1:8000,::1',
        Sanctum::currentApplicationUrlWithPort()
    ))),
    'guard' => ['web'],
    'expiration' => null,
    'token_prefix' => env('SANCTUM_TOKEN_PREFIX', ''),
    'middleware' => [
        'authenticate_session' => Laravel\Sanctum\Http\Middleware\AuthenticateSession::class,
        'encrypt_cookies' => Illuminate\Cookie\Middleware\EncryptCookies::class,
        'validate_csrf_token' => Illuminate\Foundation\Http\Middleware\ValidateCsrfToken::class,
    ],
];
```

```javascript
// frontend/src/lib/csrf.ts
// ต้องเรียก CSRF cookie ก่อน login สำหรับ cookie-based auth
export async function initCsrf(): Promise<void> {
    await fetch(`${import.meta.env.VITE_API_URL}/sanctum/csrf-cookie`, {
        credentials: 'include',
    })
}

// ใช้ axios กับ withCredentials
import axios from 'axios'

const api = axios.create({
    baseURL: import.meta.env.VITE_API_URL,
    withCredentials: true, // สำคัญมาก!
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-Requested-With': 'XMLHttpRequest',
    },
})
```

---

## ขั้นตอนที่ 2232: Token-based Auth พร้อม Refresh Token

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\User;
use App\Models\RefreshToken;
use Carbon\Carbon;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;

class TokenAuthController extends Controller
{
    private const ACCESS_TOKEN_EXPIRY_MINUTES = 15;
    private const REFRESH_TOKEN_EXPIRY_DAYS = 30;

    public function login(Request $request): JsonResponse
    {
        $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
            'device_name' => ['required', 'string', 'max:255'],
        ]);

        $user = User::where('email', $request->email)->first();

        if (! $user || ! Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['ข้อมูลไม่ถูกต้อง'],
            ]);
        }

        // สร้าง access token (หมดอายุในไม่กี่นาที)
        $accessToken = $user->createToken(
            $request->device_name,
            ['*'],
            Carbon::now()->addMinutes(self::ACCESS_TOKEN_EXPIRY_MINUTES)
        )->plainTextToken;

        // สร้าง refresh token (หมดอายุในไม่กี่วัน)
        $refreshToken = $this->createRefreshToken($user, $request->device_name);

        return response()->json([
            'access_token' => $accessToken,
            'refresh_token' => $refreshToken->token,
            'token_type' => 'Bearer',
            'expires_in' => self::ACCESS_TOKEN_EXPIRY_MINUTES * 60,
            'user' => $user,
        ]);
    }

    public function refresh(Request $request): JsonResponse
    {
        $request->validate([
            'refresh_token' => ['required', 'string'],
        ]);

        $refreshToken = RefreshToken::where('token', $request->refresh_token)
            ->where('expires_at', '>', now())
            ->whereNull('revoked_at')
            ->with('user')
            ->first();

        if (! $refreshToken) {
            return response()->json(['message' => 'Invalid refresh token'], 401);
        }

        $user = $refreshToken->user;

        // Revoke old tokens สำหรับ device นี้
        $user->tokens()->where('name', $refreshToken->device_name)->delete();

        // สร้าง token ใหม่
        $accessToken = $user->createToken(
            $refreshToken->device_name,
            ['*'],
            Carbon::now()->addMinutes(self::ACCESS_TOKEN_EXPIRY_MINUTES)
        )->plainTextToken;

        // Rotate refresh token
        $refreshToken->update(['revoked_at' => now()]);
        $newRefreshToken = $this->createRefreshToken($user, $refreshToken->device_name);

        return response()->json([
            'access_token' => $accessToken,
            'refresh_token' => $newRefreshToken->token,
            'token_type' => 'Bearer',
            'expires_in' => self::ACCESS_TOKEN_EXPIRY_MINUTES * 60,
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        // Revoke current access token
        $request->user()->currentAccessToken()->delete();

        // Revoke all refresh tokens สำหรับ device นี้
        RefreshToken::where('user_id', $request->user()->id)
            ->where('device_name', $request->device_name ?? 'unknown')
            ->update(['revoked_at' => now()]);

        return response()->json(['message' => 'Logged out successfully']);
    }

    public function logoutAll(Request $request): JsonResponse
    {
        // Revoke ทุก token ของ user
        $request->user()->tokens()->delete();
        RefreshToken::where('user_id', $request->user()->id)
            ->update(['revoked_at' => now()]);

        return response()->json(['message' => 'Logged out from all devices']);
    }

    private function createRefreshToken(User $user, string $deviceName): RefreshToken
    {
        return RefreshToken::create([
            'user_id' => $user->id,
            'token' => hash('sha256', Str::random(64)),
            'device_name' => $deviceName,
            'expires_at' => Carbon::now()->addDays(self::REFRESH_TOKEN_EXPIRY_DAYS),
        ]);
    }
}
```

---

## ขั้นตอนที่ 2233: RefreshToken Model และ Migration

```php
<?php
declare(strict_types=1);

// database/migrations/xxxx_create_refresh_tokens_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('refresh_tokens', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('token', 64)->unique();
            $table->string('device_name');
            $table->string('ip_address', 45)->nullable();
            $table->text('user_agent')->nullable();
            $table->timestamp('last_used_at')->nullable();
            $table->timestamp('expires_at');
            $table->timestamp('revoked_at')->nullable();
            $table->timestamps();

            $table->index(['user_id', 'device_name']);
            $table->index('expires_at');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('refresh_tokens');
    }
};
```

```php
<?php
declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class RefreshToken extends Model
{
    protected $fillable = [
        'user_id',
        'token',
        'device_name',
        'ip_address',
        'user_agent',
        'last_used_at',
        'expires_at',
        'revoked_at',
    ];

    protected $casts = [
        'expires_at' => 'datetime',
        'revoked_at' => 'datetime',
        'last_used_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function isValid(): bool
    {
        return $this->expires_at > now() && is_null($this->revoked_at);
    }
}
```

---

## ขั้นตอนที่ 2234: Remember Me

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class SessionController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
            'remember' => ['boolean'],
        ]);

        $credentials = $request->only('email', 'password');
        $remember = $request->boolean('remember');

        if (! Auth::attempt($credentials, $remember)) {
            throw ValidationException::withMessages([
                'email' => ['อีเมลหรือรหัสผ่านไม่ถูกต้อง'],
            ]);
        }

        $request->session()->regenerate();

        // ถ้า remember me อายุ session จะยาวนานขึ้น
        if ($remember) {
            $request->session()->put('remember_expires', now()->addDays(30));
        }

        return response()->json([
            'user' => Auth::user(),
            'remember' => $remember,
        ]);
    }
}
```

---

## ขั้นตอนที่ 2235: Social Login กับ Socialite

```php
<?php
declare(strict_types=1);

// composer require laravel/socialite

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Str;
use Laravel\Socialite\Facades\Socialite;

class SocialAuthController extends Controller
{
    private const SUPPORTED_PROVIDERS = ['google', 'github', 'facebook'];

    public function redirect(string $provider): RedirectResponse
    {
        abort_if(! in_array($provider, self::SUPPORTED_PROVIDERS), 400, 'Provider ไม่รองรับ');

        return Socialite::driver($provider)
            ->stateless()
            ->redirect();
    }

    public function callback(string $provider, Request $request): JsonResponse
    {
        abort_if(! in_array($provider, self::SUPPORTED_PROVIDERS), 400, 'Provider ไม่รองรับ');

        try {
            $socialUser = Socialite::driver($provider)->stateless()->user();
        } catch (\Exception $e) {
            return response()->json(['message' => 'Social auth failed'], 422);
        }

        $user = $this->findOrCreateUser($socialUser, $provider);

        $token = $user->createToken('social-auth')->plainTextToken;

        return response()->json([
            'token' => $token,
            'user' => $user,
            'is_new_user' => $user->wasRecentlyCreated,
        ]);
    }

    private function findOrCreateUser(mixed $socialUser, string $provider): User
    {
        // หา social account ก่อน
        $socialAccount = \App\Models\SocialAccount::where('provider', $provider)
            ->where('provider_id', $socialUser->getId())
            ->first();

        if ($socialAccount) {
            return $socialAccount->user;
        }

        // หา user จาก email
        $user = User::where('email', $socialUser->getEmail())->first();

        if (! $user) {
            $user = User::create([
                'name' => $socialUser->getName(),
                'email' => $socialUser->getEmail(),
                'password' => bcrypt(Str::random(32)),
                'email_verified_at' => now(),
                'avatar' => $socialUser->getAvatar(),
            ]);
        }

        // สร้าง social account
        $user->socialAccounts()->create([
            'provider' => $provider,
            'provider_id' => $socialUser->getId(),
            'token' => $socialUser->token,
            'refresh_token' => $socialUser->refreshToken,
            'expires_at' => $socialUser->expiresIn
                ? now()->addSeconds($socialUser->expiresIn)
                : null,
        ]);

        return $user;
    }
}
```

---

## ขั้นตอนที่ 2236: Multi-device Session Management

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Laravel\Sanctum\PersonalAccessToken;

class SessionManagementController extends Controller
{
    /**
     * แสดง sessions ทั้งหมดของ user
     */
    public function index(Request $request): JsonResponse
    {
        $currentTokenId = $request->user()->currentAccessToken()->id;

        $sessions = $request->user()->tokens()
            ->orderByDesc('last_used_at')
            ->get()
            ->map(function (PersonalAccessToken $token) use ($currentTokenId) {
                return [
                    'id' => $token->id,
                    'name' => $token->name,
                    'is_current' => $token->id === $currentTokenId,
                    'last_used_at' => $token->last_used_at?->diffForHumans(),
                    'created_at' => $token->created_at->toDateString(),
                    'expires_at' => $token->expires_at?->toDateTimeString(),
                ];
            });

        return response()->json(['data' => $sessions]);
    }

    /**
     * ยกเลิก session เฉพาะ
     */
    public function revoke(Request $request, int $tokenId): JsonResponse
    {
        $token = $request->user()->tokens()->findOrFail($tokenId);
        $token->delete();

        return response()->json(['message' => 'Session revoked']);
    }

    /**
     * ยกเลิก sessions ทั้งหมด ยกเว้น current
     */
    public function revokeOthers(Request $request): JsonResponse
    {
        $currentTokenId = $request->user()->currentAccessToken()->id;

        $count = $request->user()->tokens()
            ->where('id', '!=', $currentTokenId)
            ->delete();

        return response()->json([
            'message' => "Revoked {$count} sessions",
            'revoked_count' => $count,
        ]);
    }
}
```

---

## ขั้นตอนที่ 2237: Frontend Token Management

```typescript
// src/lib/tokenManager.ts

interface TokenData {
    accessToken: string
    refreshToken: string
    expiresAt: number // Unix timestamp
}

class TokenManager {
    private readonly ACCESS_TOKEN_KEY = 'access_token'
    private readonly REFRESH_TOKEN_KEY = 'refresh_token'
    private readonly EXPIRES_AT_KEY = 'token_expires_at'

    setTokens(accessToken: string, refreshToken: string, expiresIn: number): void {
        localStorage.setItem(this.ACCESS_TOKEN_KEY, accessToken)
        localStorage.setItem(this.REFRESH_TOKEN_KEY, refreshToken)
        localStorage.setItem(
            this.EXPIRES_AT_KEY,
            String(Date.now() + expiresIn * 1000)
        )
    }

    getAccessToken(): string | null {
        return localStorage.getItem(this.ACCESS_TOKEN_KEY)
    }

    getRefreshToken(): string | null {
        return localStorage.getItem(this.REFRESH_TOKEN_KEY)
    }

    isAccessTokenExpired(): boolean {
        const expiresAt = localStorage.getItem(this.EXPIRES_AT_KEY)
        if (!expiresAt) return true
        return Date.now() > parseInt(expiresAt) - 60_000 // 1 minute buffer
    }

    clearTokens(): void {
        localStorage.removeItem(this.ACCESS_TOKEN_KEY)
        localStorage.removeItem(this.REFRESH_TOKEN_KEY)
        localStorage.removeItem(this.EXPIRES_AT_KEY)
    }

    hasValidTokens(): boolean {
        return !!this.getAccessToken() && !!this.getRefreshToken()
    }
}

export const tokenManager = new TokenManager()
```

---

## ขั้นตอนที่ 2238: Email Verification สำหรับ SPA

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Auth\Events\Verified;
use Illuminate\Foundation\Auth\EmailVerificationRequest;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class EmailVerificationController extends Controller
{
    public function send(Request $request): JsonResponse
    {
        if ($request->user()->hasVerifiedEmail()) {
            return response()->json(['message' => 'Email already verified']);
        }

        $request->user()->sendEmailVerificationNotification();

        return response()->json(['message' => 'Verification email sent']);
    }

    public function verify(EmailVerificationRequest $request): JsonResponse
    {
        if ($request->user()->hasVerifiedEmail()) {
            return response()->json(['message' => 'Email already verified']);
        }

        if ($request->user()->markEmailAsVerified()) {
            event(new Verified($request->user()));
        }

        return response()->json(['message' => 'Email verified successfully']);
    }
}
```

---

## สรุปบทที่ 79

| กลยุทธ์ | เหมาะกับ | ข้อดี |
|---------|---------|-------|
| Sanctum Cookie | Same-domain SPA | Auto CSRF protection |
| Sanctum Token | Mobile / Cross-domain | Flexible |
| Refresh Token | High security | Short-lived access |
| Social Login | Better UX | No password needed |
| Multi-device | Enterprise | Full control |
| Remember Me | User convenience | Persistent session |

ถัดไป → Part 80: PHP Data Structures และ Algorithms
