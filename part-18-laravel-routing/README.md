# Part 18: Laravel Routing & Controllers
## ขั้นตอนที่ 451-490: การจัดการ Routes และ Controllers

---

## ขั้นตอนที่ 451: Routes พื้นฐาน

```php
<?php
// routes/web.php

use Illuminate\Support\Facades\Route;

// Basic Routes
Route::get('/', fn() => view('welcome'));
Route::get('/about', fn() => 'About page');
Route::post('/contact', [ContactController::class, 'store']);
Route::put('/users/{id}', [UserController::class, 'update']);
Route::patch('/users/{id}', [UserController::class, 'update']);
Route::delete('/users/{id}', [UserController::class, 'destroy']);
Route::options('/users', fn() => response('', 200));

// Multiple Methods
Route::match(['get', 'post'], '/form', [FormController::class, 'handle']);
Route::any('/webhook', [WebhookController::class, 'handle']);

// Redirect
Route::redirect('/old-page', '/new-page', 301);
Route::permanentRedirect('/old', '/new');

// View
Route::view('/welcome', 'welcome', ['name' => 'Taylor']);

// ========================
// Route Parameters
// ========================
// Required
Route::get('/users/{id}', fn(string $id) => "User: {$id}");

// Optional
Route::get('/posts/{post?}', fn(?string $post = null) => $post ?? 'All posts');

// Constraints
Route::get('/users/{id}', [UserController::class, 'show'])
    ->where('id', '[0-9]+');

Route::get('/posts/{slug}', [PostController::class, 'show'])
    ->where('slug', '[a-z0-9-]+');

// Multiple constraints
Route::get('/users/{id}/posts/{postId}', fn() => '...')
    ->whereNumber('id')
    ->whereAlpha('slug')
    ->whereAlphaNumeric('token')
    ->whereUuid('uuid');

// Global constraints in RouteServiceProvider
// Route::pattern('id', '[0-9]+');

// ========================
// Named Routes
// ========================
Route::get('/users/{id}', [UserController::class, 'show'])->name('users.show');
Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');

// Generate URLs
$url = route('users.show', ['id' => 1]);         // /users/1
$url = route('users.show', ['id' => 1], false);  // Relative path

// Redirect to named route
return redirect()->route('users.show', ['id' => 1]);
return to_route('dashboard');  // Shorthand
```

---

## ขั้นตอนที่ 452: Route Groups & Middleware

```php
<?php
// routes/web.php

use App\Http\Controllers\Admin\UserController as AdminUserController;

// ========================
// Middleware Groups
// ========================
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::get('/profile', [ProfileController::class, 'show'])->name('profile');
    Route::put('/profile', [ProfileController::class, 'update'])->name('profile.update');
});

// ========================
// Prefix Groups
// ========================
Route::prefix('admin')->group(function () {
    Route::get('/users', [AdminUserController::class, 'index'])->name('admin.users.index');
    Route::get('/posts', [AdminPostController::class, 'index'])->name('admin.posts.index');
});

// ========================
// Combine prefix + middleware + name
// ========================
Route::prefix('admin')
    ->middleware(['auth', 'role:admin'])
    ->name('admin.')
    ->group(function () {
        Route::resource('users', AdminUserController::class);
        Route::resource('posts', AdminPostController::class);
        Route::get('/dashboard', [AdminDashboardController::class, 'index'])->name('dashboard');
    });

// ========================
// Resource Routes
// ========================
Route::resource('posts', PostController::class);
// Generates:
// GET    /posts             posts.index
// GET    /posts/create      posts.create
// POST   /posts             posts.store
// GET    /posts/{post}      posts.show
// GET    /posts/{post}/edit posts.edit
// PUT    /posts/{post}      posts.update
// DELETE /posts/{post}      posts.destroy

// Partial resource
Route::resource('photos', PhotoController::class)->only(['index', 'show']);
Route::resource('photos', PhotoController::class)->except(['destroy']);

// Nested resource
Route::resource('posts.comments', PostCommentController::class);
// /posts/{post}/comments/{comment}

// Shallow nesting
Route::resource('posts.comments', PostCommentController::class)->shallow();

// ========================
// API Resource Routes
// ========================
// routes/api.php
Route::apiResource('users', UserController::class);
// Same as resource but without create/edit (no view routes)

// Versioning
Route::prefix('v1')->group(function () {
    Route::apiResource('users', Api\V1\UserController::class);
    Route::apiResource('posts', Api\V1\PostController::class);
});

Route::prefix('v2')->group(function () {
    Route::apiResource('users', Api\V2\UserController::class);
});
```

---

## ขั้นตอนที่ 453: Controllers

```php
<?php
namespace App\Http\Controllers;

use App\Models\User;
use App\Http\Requests\StoreUserRequest;
use App\Http\Requests\UpdateUserRequest;
use App\Http\Resources\UserResource;
use App\Http\Resources\UserCollection;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Http\JsonResponse;
use Illuminate\View\View;
use Illuminate\Http\RedirectResponse;

class UserController extends Controller {
    public function index(Request $request): View {
        $users = User::query()
            ->when($request->filled('search'), fn($q) => $q->where('name', 'like', "%{$request->search}%"))
            ->when($request->filled('role'), fn($q) => $q->where('role', $request->role))
            ->orderBy($request->get('sort', 'created_at'), $request->get('direction', 'desc'))
            ->paginate(15)
            ->withQueryString();
        
        return view('users.index', compact('users'));
    }
    
    public function create(): View {
        return view('users.create');
    }
    
    public function store(StoreUserRequest $request): RedirectResponse {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => bcrypt($request->password),
        ]);
        
        return redirect()
            ->route('users.show', $user)
            ->with('success', 'User created successfully!');
    }
    
    public function show(User $user): View {
        // Route model binding: id ใน URL → User object
        return view('users.show', compact('user'));
    }
    
    public function edit(User $user): View {
        return view('users.edit', compact('user'));
    }
    
    public function update(UpdateUserRequest $request, User $user): RedirectResponse {
        $user->update($request->validated());
        
        return redirect()
            ->route('users.show', $user)
            ->with('success', 'Updated successfully!');
    }
    
    public function destroy(User $user): RedirectResponse {
        $user->delete();
        
        return redirect()
            ->route('users.index')
            ->with('success', 'User deleted!');
    }
}

// ========================
// API Controller
// ========================
namespace App\Http\Controllers\Api;

class UserController extends Controller {
    public function index(Request $request): JsonResponse {
        $users = User::paginate(15);
        return response()->json(UserCollection::make($users));
    }
    
    public function show(User $user): JsonResponse {
        return response()->json(UserResource::make($user));
    }
    
    public function store(StoreUserRequest $request): JsonResponse {
        $user = User::create($request->validated() + ['password' => bcrypt($request->password)]);
        return response()->json(UserResource::make($user), 201);
    }
    
    public function update(UpdateUserRequest $request, User $user): JsonResponse {
        $user->update($request->validated());
        return response()->json(UserResource::make($user->fresh()));
    }
    
    public function destroy(User $user): Response {
        $user->delete();
        return response()->noContent();
    }
}
```

---

## ขั้นตอนที่ 454: Form Requests

```php
<?php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rules\Password;

class StoreUserRequest extends FormRequest {
    // ตรวจสอบ Authorization
    public function authorize(): bool {
        // true = ทุกคนทำได้
        // ตรวจสอบ policy:
        // return $this->user()->can('create', User::class);
        return true;
    }
    
    // Validation Rules
    public function rules(): array {
        return [
            'name' => ['required', 'string', 'min:2', 'max:100'],
            'email' => ['required', 'email', 'unique:users,email'],
            'password' => ['required', Password::defaults()],
            'role' => ['required', 'in:admin,editor,user'],
            'avatar' => ['nullable', 'image', 'max:2048'],  // 2MB
        ];
    }
    
    // Custom error messages
    public function messages(): array {
        return [
            'name.required' => 'กรุณากรอกชื่อ',
            'email.unique' => 'Email นี้ถูกใช้แล้ว',
            'password.required' => 'กรุณากรอกรหัสผ่าน',
        ];
    }
    
    // Custom attribute names
    public function attributes(): array {
        return [
            'name' => 'ชื่อ',
            'email' => 'อีเมล',
            'password' => 'รหัสผ่าน',
        ];
    }
    
    // Modify data before validation
    protected function prepareForValidation(): void {
        $this->merge([
            'email' => strtolower($this->email ?? ''),
            'name' => trim($this->name ?? ''),
        ]);
    }
    
    // After validation
    public function passedValidation(): void {
        // Handle file upload, etc.
    }
}

class UpdateUserRequest extends FormRequest {
    public function authorize(): bool {
        $user = User::findOrFail($this->route('user'));
        return $this->user()->can('update', $user);
    }
    
    public function rules(): array {
        return [
            'name' => ['sometimes', 'string', 'min:2', 'max:100'],
            'email' => ['sometimes', 'email', "unique:users,email,{$this->route('user')}"],
        ];
    }
}
```

---

## ขั้นตอนที่ 455: Route Model Binding

```php
<?php
// routes/web.php

// Implicit Binding (by primary key)
Route::get('/users/{user}', [UserController::class, 'show']);
// {user} → User::findOrFail($id)

// Binding by different column
Route::get('/users/{user:email}', [UserController::class, 'showByEmail']);
// {user:email} → User::where('email', $email)->firstOrFail()

Route::get('/posts/{post:slug}', [PostController::class, 'show']);

// Explicit Binding (in RouteServiceProvider)
public function boot(): void {
    Route::model('user', User::class);
    // OR custom resolution:
    Route::bind('user', function (string $value) {
        return User::where('uuid', $value)->firstOrFail();
    });
}

// Scoped Bindings
Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    // Post ต้องเป็นของ $user
    // เพิ่ม ->scopeBindings() หรือ parent_id
})->scopeBindings();

// ========================
// Route Caching (Production)
// ========================
// php artisan route:cache
// php artisan route:clear
// php artisan route:list
// php artisan route:list --name=users
// php artisan route:list --path=api
```

---

## ขั้นตอนที่ 456: Middleware

```php
<?php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

// สร้าง: php artisan make:middleware CheckRole
class CheckRole {
    public function handle(Request $request, Closure $next, string ...$roles): Response {
        if (!$request->user()) {
            return redirect()->route('login');
        }
        
        if (!in_array($request->user()->role, $roles)) {
            abort(403, 'Unauthorized');
        }
        
        return $next($request);
    }
}

// Rate Limiting Middleware
class ThrottleRequests {
    public function handle(Request $request, Closure $next, int $maxAttempts = 60): Response {
        // Built-in: just use 'throttle:60,1' in route
        return $next($request);
    }
}

// bootstrap/app.php (Laravel 11)
->withMiddleware(function (Middleware $middleware) {
    // Append global middleware
    $middleware->append(SecurityHeaders::class);
    
    // Prepend
    $middleware->prepend(TrustProxies::class);
    
    // Alias (ใช้ใน routes)
    $middleware->alias([
        'role' => CheckRole::class,
        'admin' => EnsureIsAdmin::class,
        'json' => AcceptJson::class,
    ]);
    
    // Groups
    $middleware->appendToGroup('web', [
        ShareDataWithViews::class,
    ]);
    
    // API group
    $middleware->group('api', [
        'throttle:api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ]);
})

// ใช้ใน Routes
Route::get('/admin', [AdminController::class, 'index'])
    ->middleware(['auth', 'role:admin,super-admin']);
    
// Built-in Middleware
// auth         - ต้อง login
// auth:api     - API auth
// guest        - ต้องไม่ login
// throttle:60,1 - 60 requests per 1 minute
// verified     - ต้องยืนยัน email
// signed       - ต้องมี signed URL
// cache.headers - Cache control headers
```

---

## 🎯 สรุป Part 18

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Basic Routes | GET, POST, PUT, PATCH, DELETE |
| Route Groups | middleware, prefix, name |
| Resource Routes | 7 CRUD routes อัตโนมัติ |
| Controllers | CRUD methods, Route binding |
| Form Requests | authorize(), rules(), messages() |
| Route Model Binding | Auto-inject Model จาก URL |
| Middleware | Custom, Alias, Groups |

**ถัดไป → Part 19: Laravel MVC & Blade Templates**
