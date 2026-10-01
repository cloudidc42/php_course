# Part 21: Laravel Authentication & Authorization
## ขั้นตอนที่ 521-550: Auth, Gates, Policies

---

## ขั้นตอนที่ 521: Laravel Auth Setup

```bash
# ติดตั้ง Laravel Breeze (Simple auth)
composer require laravel/breeze --dev
php artisan breeze:install blade  # หรือ react, vue, api
php artisan migrate
npm install && npm run dev

# หรือ Laravel Fortify (headless)
composer require laravel/fortify
php artisan vendor:publish --provider="Laravel\Fortify\FortifyServiceProvider"
php artisan migrate

# หรือ Laravel Jetstream (Full featured)
composer require laravel/jetstream
php artisan jetstream:install livewire  # หรือ inertia
php artisan migrate
npm install && npm run dev
```

---

## ขั้นตอนที่ 522: Custom Authentication

```php
<?php
namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\RedirectResponse;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\Str;
use Illuminate\Validation\ValidationException;

class LoginController extends Controller {
    public function showForm() {
        return view('auth.login');
    }
    
    public function login(Request $request): RedirectResponse {
        $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
        ]);
        
        // Rate limiting
        $this->ensureIsNotRateLimited($request);
        
        $credentials = $request->only('email', 'password');
        $remember = $request->boolean('remember');
        
        if (!Auth::attempt($credentials, $remember)) {
            RateLimiter::hit($this->throttleKey($request), 300);
            
            throw ValidationException::withMessages([
                'email' => __('auth.failed'),
            ]);
        }
        
        RateLimiter::clear($this->throttleKey($request));
        
        $request->session()->regenerate();
        
        return redirect()->intended(route('dashboard'));
    }
    
    public function logout(Request $request): RedirectResponse {
        Auth::logout();
        
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        
        return redirect('/');
    }
    
    private function throttleKey(Request $request): string {
        return Str::transliterate(Str::lower($request->input('email')) . '|' . $request->ip());
    }
    
    private function ensureIsNotRateLimited(Request $request): void {
        if (!RateLimiter::tooManyAttempts($this->throttleKey($request), 5)) {
            return;
        }
        
        $seconds = RateLimiter::availableIn($this->throttleKey($request));
        
        throw ValidationException::withMessages([
            'email' => __('auth.throttle', ['seconds' => $seconds, 'minutes' => ceil($seconds / 60)]),
        ]);
    }
}

// RegisterController
class RegisterController extends Controller {
    public function store(Request $request): RedirectResponse {
        $validated = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users'],
            'password' => ['required', 'confirmed', \Illuminate\Validation\Rules\Password::defaults()],
        ]);
        
        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => bcrypt($validated['password']),
        ]);
        
        event(new \App\Events\UserRegistered($user));
        
        Auth::login($user);
        
        return redirect()->route('dashboard');
    }
}
```

---

## ขั้นตอนที่ 523: Gates & Policies

```php
<?php
// app/Providers/AppServiceProvider.php (Laravel 11)
// หรือ app/Providers/AuthServiceProvider.php (Laravel 10)

use Illuminate\Support\Facades\Gate;

public function boot(): void {
    // ========================
    // GATES - Simple authorization
    // ========================
    
    // Simple gate
    Gate::define('view-admin', fn(\App\Models\User $user) => $user->role === 'admin');
    
    // With additional args
    Gate::define('update-post', function (\App\Models\User $user, \App\Models\Post $post) {
        return $user->id === $post->user_id || $user->isAdmin();
    });
    
    // Before gate (ทุก gate check)
    Gate::before(function (\App\Models\User $user) {
        if ($user->isSuperAdmin()) return true; // Super admin bypasses all
    });
    
    // After gate
    Gate::after(function (\App\Models\User $user, string $ability, bool|null $result) {
        // Log authorization attempts
    });
    
    // ========================
    // POLICIES - Model-based authorization
    // ========================
    // php artisan make:policy PostPolicy --model=Post
    Gate::policy(\App\Models\Post::class, \App\Policies\PostPolicy::class);
}
```

```php
<?php
// app/Policies/PostPolicy.php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class PostPolicy {
    use HandlesAuthorization;
    
    // Before: called before all other methods
    public function before(User $user): ?bool {
        if ($user->isAdmin()) return true;
        return null; // Continue to specific method
    }
    
    public function viewAny(?User $user): bool {
        return true; // Anyone can view list
    }
    
    public function view(?User $user, Post $post): bool {
        if ($post->isPublished()) return true;
        return $user?->id === $post->user_id;
    }
    
    public function create(User $user): bool {
        return $user->role !== 'viewer';
    }
    
    public function update(User $user, Post $post): bool {
        return $user->id === $post->user_id;
    }
    
    public function delete(User $user, Post $post): bool {
        return $user->id === $post->user_id;
    }
    
    public function restore(User $user, Post $post): bool {
        return $user->id === $post->user_id;
    }
    
    public function forceDelete(User $user, Post $post): bool {
        return $user->isAdmin();
    }
    
    // Custom policy method
    public function publish(User $user, Post $post): bool {
        return $user->id === $post->user_id && !$post->isPublished();
    }
}
```

```php
<?php
// ========================
// Using Gates
// ========================

// In Controller
class AdminController extends Controller {
    public function index(): View {
        Gate::authorize('view-admin'); // Throws 403 if fails
        
        return view('admin.dashboard');
    }
}

// In Controller with $this->authorize (shorthand)
class PostController extends Controller {
    public function update(Request $request, Post $post): RedirectResponse {
        $this->authorize('update', $post); // Uses PostPolicy::update
        
        $post->update($request->validated());
        return back()->with('success', 'Updated!');
    }
    
    public function store(Request $request): RedirectResponse {
        $this->authorize('create', Post::class); // Policy for model class
        
        $post = Post::create($request->validated() + ['user_id' => auth()->id()]);
        return redirect()->route('posts.show', $post);
    }
}

// Check without throwing
if (Gate::allows('update-post', $post)) {
    // User can update
}

if (Gate::denies('view-admin')) {
    abort(403);
}

if (auth()->user()->can('publish', $post)) {
    // ...
}

if (auth()->user()->cannot('delete', $post)) {
    // ...
}

// In Blade
@can('update', $post)
    <a href="{{ route('posts.edit', $post) }}">Edit</a>
@endcan

@can('create', App\Models\Post::class)
    <a href="{{ route('posts.create') }}">New Post</a>
@endcan
```

---

## ขั้นตอนที่ 524: Multi-Guard Authentication

```php
<?php
// config/auth.php
return [
    'defaults' => ['guard' => 'web'],
    
    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],
        'api' => [
            'driver' => 'token', // หรือ sanctum
            'provider' => 'users',
            'hash' => true,
        ],
        'admin' => [
            'driver' => 'session',
            'provider' => 'admins',
        ],
    ],
    
    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\Models\User::class,
        ],
        'admins' => [
            'driver' => 'eloquent',
            'model' => App\Models\Admin::class,
        ],
    ],
    
    'passwords' => [
        'users' => [
            'provider' => 'users',
            'table' => 'password_reset_tokens',
            'expire' => 60,
            'throttle' => 60,
        ],
    ],
];

// Using Different Guards
Auth::guard('admin')->attempt($credentials);
Auth::guard('api')->user();

// In Routes
Route::middleware('auth:admin')->group(function () {
    Route::get('/admin', [AdminController::class, 'index']);
});

Route::middleware('auth:api,sanctum')->group(function () {
    Route::get('/api/profile', [ApiController::class, 'profile']);
});
```

---

## ขั้นตอนที่ 525: Laravel Sanctum (API Auth)

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

```php
<?php
// app/Models/User.php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable {
    use HasApiTokens;
    
    // ...
}

// ========================
// API Token Authentication
// ========================
// Login and get token
class AuthController extends Controller {
    public function login(Request $request): JsonResponse {
        $request->validate(['email' => 'required|email', 'password' => 'required']);
        
        if (!Auth::attempt($request->only('email', 'password'))) {
            return response()->json(['message' => 'Invalid credentials'], 401);
        }
        
        $user = $request->user();
        
        // Create token with abilities
        $token = $user->createToken('api-token', ['read', 'write'])->plainTextToken;
        
        return response()->json([
            'token' => $token,
            'token_type' => 'Bearer',
            'user' => $user,
        ]);
    }
    
    public function logout(Request $request): JsonResponse {
        $request->user()->currentAccessToken()->delete();
        // Or delete all tokens:
        // $request->user()->tokens()->delete();
        
        return response()->json(['message' => 'Logged out']);
    }
    
    public function profile(Request $request): JsonResponse {
        return response()->json($request->user());
    }
}

// ========================
// Check Token Abilities
// ========================
Route::middleware(['auth:sanctum'])->group(function () {
    Route::get('/profile', [AuthController::class, 'profile']);
    
    Route::middleware(['ability:write'])->group(function () {
        Route::post('/posts', [PostController::class, 'store']);
        Route::delete('/posts/{post}', [PostController::class, 'destroy']);
    });
});

// In Controller
if ($request->user()->tokenCan('write')) {
    // Has write ability
}

// ========================
// SPA Authentication (Cookie-based)
// ========================
// bootstrap/app.php
->withMiddleware(function (Middleware $middleware) {
    $middleware->statefulApi();  // Adds Sanctum SPA middleware
})

// CORS config: config/cors.php
'supports_credentials' => true,

// Frontend: First call /sanctum/csrf-cookie, then login
```

---

## ขั้นตอนที่ 526: Email Verification

```php
<?php
// app/Models/User.php
use Illuminate\Contracts\Auth\MustVerifyEmail;

class User extends Authenticatable implements MustVerifyEmail {
    // Laravel ส่ง verification email ให้อัตโนมัติหลัง register
}

// routes/web.php
Auth::routes(['verify' => true]);

// OR manually (Laravel 11):
Route::get('/email/verify', function () {
    return view('auth.verify-email');
})->middleware('auth')->name('verification.notice');

Route::get('/email/verify/{id}/{hash}', function (EmailVerificationRequest $request) {
    $request->fulfill();
    return redirect('/dashboard');
})->middleware(['auth', 'signed'])->name('verification.verify');

Route::post('/email/verification-notification', function (Request $request) {
    $request->user()->sendEmailVerificationNotification();
    return back()->with('message', 'Verification link sent!');
})->middleware(['auth', 'throttle:6,1'])->name('verification.send');

// Require verified email
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

// Custom Verification Email
class User extends Authenticatable implements MustVerifyEmail {
    public function sendEmailVerificationNotification(): void {
        $this->notify(new \App\Notifications\CustomVerifyEmail());
    }
}
```

---

## 🎯 สรุป Part 21

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Authentication | Auth::attempt, login, logout |
| Guards | web, api, custom guards |
| Gates | Simple ability checks |
| Policies | Model-based authorization |
| Sanctum | API tokens, SPA authentication |
| Email Verification | MustVerifyEmail, signed routes |

**ถัดไป → Part 22: Laravel Testing**
