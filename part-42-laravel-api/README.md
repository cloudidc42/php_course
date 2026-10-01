# Part 42: Laravel API

## ขั้นตอนที่ 1121-1150: การสร้าง API ด้วย Laravel

บทนี้ครอบคลุม API Resources, Versioning, Sanctum Authentication, Rate Limiting, OpenAPI Documentation และ API Testing

---

## ขั้นตอนที่ 1121: API Resources และ Transformations

API Resources ช่วยควบคุม format ของ JSON response

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'         => $this->id,
            'name'       => $this->name,
            'email'      => $this->email,
            'avatar_url' => $this->avatar_url,
            'role'       => $this->role->name,

            // Conditional fields
            'phone'      => $this->when(
                $request->user()?->can('viewPhone', $this->resource),
                $this->phone
            ),

            // Merge conditionally
            $this->mergeWhen($request->user()?->id === $this->id, [
                'two_factor_enabled' => $this->two_factor_enabled,
                'api_token_expires'  => $this->api_token_expires_at?->toISOString(),
            ]),

            // Relationships (loaded only when available)
            'posts'      => PostResource::collection($this->whenLoaded('posts')),
            'company'    => new CompanyResource($this->whenLoaded('company')),

            // Pivot data
            'joined_at'  => $this->whenPivotLoaded('team_user', function () {
                return $this->pivot->joined_at->toDateString();
            }),

            // Computed fields
            'full_address' => $this->when(
                $this->address !== null,
                fn () => "{$this->address}, {$this->city}, {$this->province}"
            ),

            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }

    public function with(Request $request): array
    {
        return [
            'meta' => [
                'version' => '1.0',
            ],
        ];
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    public string $collects = UserResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
        ];
    }

    public function with(Request $request): array
    {
        return [
            'meta' => [
                'total'       => $this->resource->total(),
                'per_page'    => $this->resource->perPage(),
                'current_page'=> $this->resource->currentPage(),
                'last_page'   => $this->resource->lastPage(),
            ],
            'links' => [
                'first' => $this->resource->url(1),
                'last'  => $this->resource->url($this->resource->lastPage()),
                'prev'  => $this->resource->previousPageUrl(),
                'next'  => $this->resource->nextPageUrl(),
            ],
        ];
    }
}
```

---

## ขั้นตอนที่ 1122: API Versioning Strategies

```php
<?php

declare(strict_types=1);

// routes/api.php - URL-based versioning
use Illuminate\Support\Facades\Route;

// Version 1
Route::prefix('v1')->name('api.v1.')->group(function () {
    Route::apiResource('users', \App\Http\Controllers\Api\V1\UserController::class);
    Route::apiResource('posts', \App\Http\Controllers\Api\V1\PostController::class);
});

// Version 2 - new endpoints, different format
Route::prefix('v2')->name('api.v2.')->group(function () {
    Route::apiResource('users', \App\Http\Controllers\Api\V2\UserController::class);
    Route::apiResource('posts', \App\Http\Controllers\Api\V2\PostController::class);
    Route::apiResource('media', \App\Http\Controllers\Api\V2\MediaController::class);
});
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api\V1;

use App\Http\Controllers\Controller;
use App\Http\Resources\V1\UserResource;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\AnonymousResourceCollection;

class UserController extends Controller
{
    public function index(Request $request): AnonymousResourceCollection
    {
        $users = User::with(['role', 'company'])
            ->filter($request->validated())
            ->paginate($request->get('per_page', 15));

        return UserResource::collection($users);
    }

    public function show(User $user): UserResource
    {
        $user->load(['role', 'company', 'posts' => fn ($q) => $q->latest()->limit(5)]);

        return new UserResource($user);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

// Header-based versioning middleware
class ApiVersion
{
    public function handle(Request $request, Closure $next): mixed
    {
        $version = $request->header('Api-Version', '1');

        // Set version จาก header
        $request->merge(['api_version' => (int)$version]);

        $response = $next($request);
        $response->headers->set('Api-Version', $version);

        return $response;
    }
}
```

---

## ขั้นตอนที่ 1123: Laravel Sanctum สำหรับ SPA Authentication

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use App\Http\Requests\Auth\LoginRequest;
use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\ValidationException;

class SanctumAuthController extends Controller
{
    public function login(LoginRequest $request): JsonResponse
    {
        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        if ($user->is_locked) {
            return response()->json([
                'message' => 'Account is locked. Contact support.',
            ], 403);
        }

        // ลบ tokens เก่า (optional)
        $user->tokens()->where('name', 'api')->delete();

        // สร้าง token ใหม่พร้อม abilities
        $token = $user->createToken('api', [
            'read',
            'write',
            ...$user->isAdmin() ? ['admin'] : [],
        ]);

        return response()->json([
            'token' => $token->plainTextToken,
            'user'  => [
                'id'    => $user->id,
                'name'  => $user->name,
                'email' => $user->email,
            ],
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        // ลบ current token
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out successfully']);
    }

    public function logoutAll(Request $request): JsonResponse
    {
        // ลบ tokens ทั้งหมด
        $request->user()->tokens()->delete();

        return response()->json(['message' => 'Logged out from all devices']);
    }
}
```

```php
<?php

declare(strict_types=1);

// Checking token abilities
namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;

class PostController
{
    public function store(Request $request)
    {
        // ตรวจสอบ ability ของ token
        if (!$request->user()->tokenCan('write')) {
            abort(403, 'Token does not have write ability');
        }

        // ...
    }
}
```

---

## ขั้นตอนที่ 1124: API Rate Limiting ขั้นสูง

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Foundation\Support\Providers\RouteServiceProvider as ServiceProvider;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

class RouteServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->configureRateLimiting();
        parent::boot();
    }

    protected function configureRateLimiting(): void
    {
        // Standard API limit
        RateLimiter::for('api', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(60)->by($request->user()->id)
                : Limit::perMinute(20)->by($request->ip());
        });

        // Auth endpoints - strict limit
        RateLimiter::for('auth', function (Request $request) {
            return [
                Limit::perMinute(5)->by('auth:' . $request->ip()),
                Limit::perHour(20)->by('auth-hourly:' . $request->ip()),
            ];
        });

        // Premium users get higher limits
        RateLimiter::for('premium', function (Request $request) {
            if ($request->user()?->isPremium()) {
                return Limit::perMinute(300)->by($request->user()->id);
            }

            return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
        });

        // Search - expensive operation
        RateLimiter::for('search', function (Request $request) {
            return [
                Limit::perMinute(10)->by('search:' . ($request->user()?->id ?: $request->ip())),
                Limit::perDay(1000)->by('search-daily:' . ($request->user()?->id ?: $request->ip())),
            ];
        });
    }
}
```

---

## ขั้นตอนที่ 1125: OpenAPI/Swagger Documentation

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Http\Resources\UserResource;
use App\Models\User;
use Illuminate\Http\Request;

/**
 * @OA\Info(
 *     version="1.0.0",
 *     title="My API",
 *     description="My Laravel API Documentation",
 *     @OA\Contact(email="admin@example.com")
 * )
 *
 * @OA\Server(
 *     url="https://api.example.com/v1",
 *     description="Production"
 * )
 *
 * @OA\SecurityScheme(
 *     securityScheme="bearerAuth",
 *     type="http",
 *     scheme="bearer",
 *     bearerFormat="JWT"
 * )
 */
class UserController
{
    /**
     * @OA\Get(
     *     path="/users",
     *     operationId="listUsers",
     *     tags={"Users"},
     *     summary="Get all users",
     *     security={{"bearerAuth":{}}},
     *     @OA\Parameter(
     *         name="per_page",
     *         in="query",
     *         required=false,
     *         @OA\Schema(type="integer", default=15, minimum=1, maximum=100)
     *     ),
     *     @OA\Parameter(
     *         name="search",
     *         in="query",
     *         required=false,
     *         @OA\Schema(type="string")
     *     ),
     *     @OA\Response(
     *         response=200,
     *         description="Success",
     *         @OA\JsonContent(
     *             @OA\Property(property="data", type="array",
     *                 @OA\Items(ref="#/components/schemas/User")
     *             ),
     *             @OA\Property(property="meta", ref="#/components/schemas/PaginationMeta")
     *         )
     *     ),
     *     @OA\Response(response=401, description="Unauthenticated"),
     *     @OA\Response(response=403, description="Unauthorized"),
     *     @OA\Response(response=429, description="Too Many Requests")
     * )
     */
    public function index(Request $request)
    {
        $users = User::query()
            ->when($request->search, fn ($q, $s) => $q->search($s))
            ->paginate($request->get('per_page', 15));

        return UserResource::collection($users);
    }

    /**
     * @OA\Post(
     *     path="/users",
     *     operationId="createUser",
     *     tags={"Users"},
     *     summary="Create a new user",
     *     security={{"bearerAuth":{}}},
     *     @OA\RequestBody(
     *         required=true,
     *         @OA\JsonContent(ref="#/components/schemas/CreateUserRequest")
     *     ),
     *     @OA\Response(
     *         response=201,
     *         description="User created",
     *         @OA\JsonContent(
     *             @OA\Property(property="data", ref="#/components/schemas/User")
     *         )
     *     ),
     *     @OA\Response(response=422, description="Validation error")
     * )
     */
    public function store(\App\Http\Requests\CreateUserRequest $request)
    {
        $user = User::create($request->validated());
        return new UserResource($user);
    }
}
```

```php
<?php

declare(strict_types=1);

/**
 * @OA\Schema(
 *     schema="User",
 *     required={"id", "name", "email"},
 *     @OA\Property(property="id", type="integer", example=1),
 *     @OA\Property(property="name", type="string", example="John Doe"),
 *     @OA\Property(property="email", type="string", format="email", example="john@example.com"),
 *     @OA\Property(property="created_at", type="string", format="date-time"),
 * )
 *
 * @OA\Schema(
 *     schema="PaginationMeta",
 *     @OA\Property(property="total", type="integer", example=100),
 *     @OA\Property(property="per_page", type="integer", example=15),
 *     @OA\Property(property="current_page", type="integer", example=1),
 *     @OA\Property(property="last_page", type="integer", example=7),
 * )
 */
class ApiSchemas {}
```

---

## ขั้นตอนที่ 1126: API Testing ด้วย HTTP Tests

```php
<?php

declare(strict_types=1);

namespace Tests\Feature\Api;

use App\Models\User;
use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

class PostApiTest extends TestCase
{
    use RefreshDatabase;

    private User $user;
    private User $admin;

    protected function setUp(): void
    {
        parent::setUp();
        $this->user  = User::factory()->create();
        $this->admin = User::factory()->admin()->create();
    }

    public function test_can_list_posts(): void
    {
        Post::factory()->count(20)->create(['user_id' => $this->user->id]);

        Sanctum::actingAs($this->user, ['read']);

        $response = $this->getJson('/api/v1/posts?per_page=10');

        $response->assertStatus(200)
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'title', 'body', 'created_at'],
                ],
                'meta' => ['total', 'per_page', 'current_page'],
                'links' => ['next', 'prev'],
            ])
            ->assertJsonCount(10, 'data')
            ->assertJsonPath('meta.total', 20);
    }

    public function test_can_create_post(): void
    {
        Sanctum::actingAs($this->user, ['write']);

        $response = $this->postJson('/api/v1/posts', [
            'title'  => 'My New Post',
            'body'   => 'Content of the post',
            'status' => 'draft',
        ]);

        $response->assertStatus(201)
            ->assertJsonPath('data.title', 'My New Post')
            ->assertJsonPath('data.user.id', $this->user->id);

        $this->assertDatabaseHas('posts', [
            'title'   => 'My New Post',
            'user_id' => $this->user->id,
        ]);
    }

    public function test_cannot_create_post_without_write_ability(): void
    {
        Sanctum::actingAs($this->user, ['read']); // read only

        $response = $this->postJson('/api/v1/posts', [
            'title' => 'New Post',
            'body'  => 'Content',
        ]);

        $response->assertStatus(403);
    }

    public function test_can_update_own_post(): void
    {
        $post = Post::factory()->create(['user_id' => $this->user->id]);

        Sanctum::actingAs($this->user, ['write']);

        $response = $this->putJson("/api/v1/posts/{$post->id}", [
            'title' => 'Updated Title',
            'body'  => 'Updated content',
        ]);

        $response->assertStatus(200)
            ->assertJsonPath('data.title', 'Updated Title');
    }

    public function test_cannot_update_others_post(): void
    {
        $otherUser = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $otherUser->id]);

        Sanctum::actingAs($this->user, ['write']);

        $response = $this->putJson("/api/v1/posts/{$post->id}", [
            'title' => 'Trying to update',
            'body'  => 'Body',
        ]);

        $response->assertStatus(403);
    }

    public function test_api_rate_limiting(): void
    {
        Sanctum::actingAs($this->user);

        // Send 61 requests (limit is 60)
        for ($i = 0; $i < 61; $i++) {
            $response = $this->getJson('/api/v1/posts');
        }

        $response->assertStatus(429)
            ->assertHeader('Retry-After')
            ->assertJsonPath('message', fn ($msg) => str_contains($msg, 'Too Many'));
    }
}
```

---

## ขั้นตอนที่ 1127: API Error Handling

```php
<?php

declare(strict_types=1);

namespace App\Exceptions;

use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;
use Illuminate\Validation\ValidationException;
use Illuminate\Auth\AuthenticationException;

class Handler extends ExceptionHandler
{
    public function render($request, \Throwable $e): mixed
    {
        if ($request->expectsJson()) {
            return $this->renderApiException($request, $e);
        }

        return parent::render($request, $e);
    }

    private function renderApiException(Request $request, \Throwable $e): JsonResponse
    {
        if ($e instanceof ValidationException) {
            return response()->json([
                'message' => 'The given data was invalid.',
                'errors'  => $e->errors(),
            ], 422);
        }

        if ($e instanceof AuthenticationException) {
            return response()->json([
                'message' => 'Unauthenticated.',
            ], 401);
        }

        if ($e instanceof HttpExceptionInterface) {
            return response()->json([
                'message' => $e->getMessage() ?: 'HTTP Error',
            ], $e->getStatusCode());
        }

        // Production: hide details
        if (config('app.env') === 'production') {
            return response()->json([
                'message' => 'An unexpected error occurred.',
            ], 500);
        }

        return response()->json([
            'message' => $e->getMessage(),
            'file'    => $e->getFile(),
            'line'    => $e->getLine(),
            'trace'   => $e->getTrace(),
        ], 500);
    }
}
```

---

## สรุปบทที่ 42

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| API Resources | JsonResource | Consistent response format |
| API Versioning | Route prefix | Backward compatibility |
| Sanctum | Token-based auth | SPA/Mobile auth |
| Rate Limiting | RateLimiter | Prevent abuse |
| OpenAPI/Swagger | darkaonline/l5-swagger | Auto documentation |
| HTTP Testing | TestCase | Integration testing |
| Error Handling | Exception Handler | Consistent errors |

**ต่อไป**: Part 43 - Design Patterns

---
