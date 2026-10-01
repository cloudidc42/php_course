# Part 24: Laravel API Resources & Advanced API
## ขั้นตอนที่ 611-630: สร้าง Production-Ready API

---

## ขั้นตอนที่ 611: API Resources

```php
<?php
// php artisan make:resource UserResource
// php artisan make:resource UserCollection --collection

namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource {
    public function toArray(Request $request): array {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            'role' => $this->role,
            'avatar' => $this->avatar_url, // from accessor
            'is_admin' => $this->isAdmin(),
            'created_at' => $this->created_at->toIso8601String(),
            
            // Conditional fields
            'phone' => $this->when($request->user()?->isAdmin(), $this->phone),
            'ssn_last4' => $this->when($request->user()?->id === $this->id, fn() => substr($this->ssn, -4)),
            
            // Conditional relationships
            'posts_count' => $this->whenCounted('posts'),
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'profile' => ProfileResource::make($this->whenLoaded('profile')),
            'latest_post' => PostResource::make($this->whenLoaded('latestPost')),
            
            // Merge attributes conditionally
            $this->mergeWhen($request->user()?->isAdmin(), [
                'last_login' => $this->last_login_at,
                'login_count' => $this->login_count,
                'ip_address' => $this->last_ip,
            ]),
            
            // Links (HATEOAS)
            'links' => [
                'self' => route('api.users.show', $this->id),
                'posts' => route('api.users.posts.index', $this->id),
                'avatar' => route('api.users.avatar', $this->id),
            ],
        ];
    }
    
    // Add custom headers
    public function withResponse(Request $request, \Illuminate\Http\JsonResponse $response): void {
        $response->header('X-User-Version', '1');
    }
    
    // Wrap with additional data
    public function with(Request $request): array {
        return [
            'meta' => [
                'version' => '1.0',
                'generated_at' => now()->toIso8601String(),
            ],
        ];
    }
}

// Collection Resource
class UserCollection extends \Illuminate\Http\Resources\Json\ResourceCollection {
    public $collects = UserResource::class;
    
    public function toArray(Request $request): array {
        return [
            'data' => $this->collection,
        ];
    }
    
    public function with(Request $request): array {
        return [
            'meta' => [
                'total' => $this->collection->count(),
            ],
        ];
    }
}

// Usage in Controller
public function index(): UserCollection {
    $users = User::with(['profile', 'posts'])
        ->withCount('posts')
        ->paginate(15);
    
    return UserCollection::make($users);
}

public function show(User $user): UserResource {
    $user->loadMissing(['profile', 'posts']);
    return UserResource::make($user);
}
```

---

## ขั้นตอนที่ 612: API Versioning Strategy

```php
<?php
// routes/api.php

use App\Http\Controllers\Api\V1;
use App\Http\Controllers\Api\V2;

Route::prefix('v1')->name('api.v1.')->group(function () {
    Route::apiResource('users', V1\UserController::class);
    Route::apiResource('posts', V1\PostController::class);
    
    Route::prefix('auth')->name('auth.')->group(function () {
        Route::post('login', [V1\AuthController::class, 'login'])->name('login');
        Route::post('register', [V1\AuthController::class, 'register'])->name('register');
        Route::post('refresh', [V1\AuthController::class, 'refresh'])->name('refresh');
        
        Route::middleware('auth:sanctum')->group(function () {
            Route::post('logout', [V1\AuthController::class, 'logout'])->name('logout');
            Route::get('me', [V1\AuthController::class, 'me'])->name('me');
        });
    });
});

Route::prefix('v2')->name('api.v2.')->group(function () {
    Route::apiResource('users', V2\UserController::class);
    // V2 might have different response format, new endpoints, etc.
});

// Accept-Version header strategy
Route::middleware(['api', 'api.version'])->group(function () {
    Route::get('/users', function (Request $request) {
        $version = $request->header('Accept-Version', 'v1');
        return match($version) {
            'v2' => V2\UserController::index(),
            default => V1\UserController::index(),
        };
    });
});
```

---

## ขั้นตอนที่ 613: API Error Handling

```php
<?php
namespace App\Exceptions;

use Illuminate\Foundation\Exceptions\Handler as ExceptionHandler;
use Illuminate\Http\Request;
use Illuminate\Validation\ValidationException;
use Illuminate\Auth\AuthenticationException;
use Symfony\Component\HttpKernel\Exception\HttpException;

class Handler extends ExceptionHandler {
    protected $dontReport = [
        ValidationException::class,
    ];
    
    public function register(): void {
        // Report exceptions to monitoring service
        $this->reportable(function (\Throwable $e) {
            if (app()->bound('sentry')) {
                app('sentry')->captureException($e);
            }
        });
        
        // Render exceptions as JSON for API requests
        $this->renderable(function (\Throwable $e, Request $request) {
            if ($request->expectsJson() || $request->is('api/*')) {
                return $this->renderApiException($e);
            }
        });
    }
    
    private function renderApiException(\Throwable $e): \Illuminate\Http\JsonResponse {
        return match(true) {
            $e instanceof ValidationException => response()->json([
                'status' => 'error',
                'message' => 'Validation failed',
                'errors' => $e->errors(),
                'code' => 422,
            ], 422),
            
            $e instanceof AuthenticationException => response()->json([
                'status' => 'error',
                'message' => 'Unauthenticated',
                'code' => 401,
            ], 401),
            
            $e instanceof \Illuminate\Auth\Access\AuthorizationException => response()->json([
                'status' => 'error',
                'message' => 'Unauthorized',
                'code' => 403,
            ], 403),
            
            $e instanceof \Illuminate\Database\Eloquent\ModelNotFoundException => response()->json([
                'status' => 'error',
                'message' => 'Resource not found',
                'code' => 404,
            ], 404),
            
            $e instanceof HttpException => response()->json([
                'status' => 'error',
                'message' => $e->getMessage() ?: 'HTTP Error',
                'code' => $e->getStatusCode(),
            ], $e->getStatusCode()),
            
            default => response()->json([
                'status' => 'error',
                'message' => app()->isProduction() ? 'Internal server error' : $e->getMessage(),
                'code' => 500,
                'trace' => app()->isProduction() ? null : $e->getTrace(),
            ], 500),
        };
    }
}
```

---

## ขั้นตอนที่ 614: API Rate Limiting

```php
<?php
// bootstrap/app.php (Laravel 11)
->withRouting(function (Router $router) {
    // Configure rate limits
    \RateLimiter::for('api', function (Request $request) {
        return $request->user()
            ? Limit::perMinute(120)->by($request->user()->id)
            : Limit::perMinute(30)->by($request->ip());
    });
    
    \RateLimiter::for('login', function (Request $request) {
        return [
            Limit::perMinute(5)->by($request->input('email')),
            Limit::perMinute(10)->by($request->ip()),
        ];
    });
    
    \RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->isAdmin()
            ? Limit::none()
            : Limit::perHour(20)->by($request->user()->id);
    });
})

// Custom Rate Limit Response
\RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)
        ->by($request->user()?->id ?: $request->ip())
        ->response(function (Request $request, array $headers) {
            return response()->json([
                'message' => 'Too many requests',
                'retry_after' => $headers['Retry-After'],
            ], 429, $headers);
        });
});

// In Routes
Route::middleware(['throttle:api'])->group(function () {
    Route::apiResource('posts', PostController::class);
});

Route::middleware(['throttle:login'])->group(function () {
    Route::post('/auth/login', [AuthController::class, 'login']);
});
```

---

## ขั้นตอนที่ 615: OpenAPI Documentation

```php
<?php
// composer require darkaonline/l5-swagger
// php artisan vendor:publish --provider "L5Swagger\L5SwaggerServiceProvider"

/**
 * @OA\Info(
 *     version="1.0.0",
 *     title="My API",
 *     description="API Documentation",
 *     @OA\Contact(email="api@example.com")
 * )
 * 
 * @OA\Server(url="https://api.example.com/v1")
 * 
 * @OA\SecurityScheme(
 *     securityScheme="bearerAuth",
 *     type="http",
 *     scheme="bearer",
 *     bearerFormat="JWT"
 * )
 */

class UserController extends Controller {
    /**
     * @OA\Get(
     *     path="/users",
     *     summary="List users",
     *     tags={"Users"},
     *     security={{"bearerAuth":{}}},
     *     @OA\Parameter(
     *         name="page",
     *         in="query",
     *         description="Page number",
     *         @OA\Schema(type="integer", default=1)
     *     ),
     *     @OA\Response(
     *         response=200,
     *         description="Success",
     *         @OA\JsonContent(
     *             @OA\Property(property="data", type="array",
     *                 @OA\Items(ref="#/components/schemas/User")
     *             ),
     *             @OA\Property(property="meta", ref="#/components/schemas/Pagination")
     *         )
     *     ),
     *     @OA\Response(response=401, description="Unauthenticated")
     * )
     */
    public function index(): UserCollection {
        return UserCollection::make(User::paginate(15));
    }
    
    /**
     * @OA\Post(
     *     path="/users",
     *     summary="Create user",
     *     tags={"Users"},
     *     security={{"bearerAuth":{}}},
     *     @OA\RequestBody(
     *         required=true,
     *         @OA\JsonContent(
     *             required={"name","email","password"},
     *             @OA\Property(property="name", type="string", example="John Doe"),
     *             @OA\Property(property="email", type="string", format="email"),
     *             @OA\Property(property="password", type="string", minLength=8)
     *         )
     *     ),
     *     @OA\Response(response=201, description="Created", @OA\JsonContent(ref="#/components/schemas/User")),
     *     @OA\Response(response=422, description="Validation Error")
     * )
     */
    public function store(StoreUserRequest $request): UserResource {
        $user = User::create($request->validated());
        return UserResource::make($user);
    }
}

/**
 * @OA\Schema(
 *     schema="User",
 *     @OA\Property(property="id", type="integer"),
 *     @OA\Property(property="name", type="string"),
 *     @OA\Property(property="email", type="string", format="email"),
 *     @OA\Property(property="created_at", type="string", format="date-time")
 * )
 */
class User extends Model {}
```

---

## ขั้นตอนที่ 616: Notifications & Broadcasting

```php
<?php
// php artisan make:notification OrderShipped

namespace App\Notifications;

use App\Models\Order;
use Illuminate\Notifications\Notification;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Messages\BroadcastMessage;

class OrderShipped extends Notification {
    public function __construct(
        private Order $order
    ) {}
    
    // Which channels to use
    public function via(object $notifiable): array {
        return ['mail', 'database', 'broadcast'];
    }
    
    // Email channel
    public function toMail(object $notifiable): MailMessage {
        return (new MailMessage)
            ->subject('คำสั่งซื้อของคุณถูกจัดส่งแล้ว')
            ->greeting('สวัสดีคุณ ' . $notifiable->name)
            ->line("คำสั่งซื้อ #{$this->order->id} ถูกจัดส่งแล้ว")
            ->action('ติดตามสินค้า', route('orders.track', $this->order))
            ->line('ขอบคุณที่ใช้บริการ!')
            ->salutation('ด้วยความนับถือ, ทีมงาน');
    }
    
    // Database channel
    public function toDatabase(object $notifiable): array {
        return [
            'order_id' => $this->order->id,
            'message' => "คำสั่งซื้อ #{$this->order->id} ถูกจัดส่งแล้ว",
            'url' => route('orders.show', $this->order),
        ];
    }
    
    // WebSocket broadcast
    public function toBroadcast(object $notifiable): BroadcastMessage {
        return new BroadcastMessage([
            'order_id' => $this->order->id,
            'message' => "Order #{$this->order->id} shipped",
            'status' => 'shipped',
        ]);
    }
    
    // WhatsApp/SMS (with custom channel)
    public function toWhatsapp(object $notifiable): string {
        return "คำสั่งซื้อ #{$this->order->id} ถูกจัดส่งแล้ว";
    }
}

// Send notification
$user->notify(new OrderShipped($order));

// To multiple users
\Notification::send($users, new OrderShipped($order));

// Read notifications in UI
$notifications = auth()->user()->notifications;
$unread = auth()->user()->unreadNotifications;
$unreadCount = auth()->user()->unreadNotifications()->count();

// Mark as read
auth()->user()->unreadNotifications()->update(['read_at' => now()]);
$notification->markAsRead();
```

---

## 🎯 สรุป Part 24

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| API Resources | toArray, conditional fields, links |
| Versioning | prefix, Accept header |
| Error Handling | Custom exception handler for API |
| Rate Limiting | RateLimiter::for, Limit::perMinute |
| OpenAPI Docs | @OA\* annotations |
| Notifications | mail, database, broadcast channels |

**ถัดไป → Part 25: Laravel Production Deployment**
