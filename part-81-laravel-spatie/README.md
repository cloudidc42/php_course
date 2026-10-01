# Part 81: Spatie Laravel Packages
## ขั้นตอนที่ 2291-2320: Spatie Permission, Media Library, Activity Log

Spatie เป็น package ที่ได้รับความนิยมสูงสุดใน Laravel ecosystem
ครอบคลุมการจัดการ roles/permissions, media, logs และอื่น ๆ

---

## ขั้นตอนที่ 2291: Spatie Laravel Permission

```bash
composer require spatie/laravel-permission
php artisan vendor:publish --provider="Spatie\Permission\PermissionServiceProvider"
php artisan migrate
```

```php
<?php
declare(strict_types=1);

// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Spatie\Permission\Traits\HasRoles;

class User extends Authenticatable
{
    use HasRoles;

    // ...
}
```

```php
<?php
declare(strict_types=1);

// database/seeders/RolePermissionSeeder.php
namespace Database\Seeders;

use Illuminate\Database\Seeder;
use Spatie\Permission\Models\Permission;
use Spatie\Permission\Models\Role;

class RolePermissionSeeder extends Seeder
{
    public function run(): void
    {
        // สร้าง permissions
        $permissions = [
            // Posts
            'posts.view', 'posts.create', 'posts.update', 'posts.delete',
            'posts.publish', 'posts.manage',
            // Users
            'users.view', 'users.create', 'users.update', 'users.delete',
            // Settings
            'settings.view', 'settings.update',
            // Media
            'media.upload', 'media.delete',
        ];

        foreach ($permissions as $permission) {
            Permission::firstOrCreate(['name' => $permission]);
        }

        // สร้าง roles และกำหนด permissions
        $admin = Role::firstOrCreate(['name' => 'admin']);
        $admin->syncPermissions(Permission::all());

        $editor = Role::firstOrCreate(['name' => 'editor']);
        $editor->syncPermissions([
            'posts.view', 'posts.create', 'posts.update', 'posts.publish',
            'media.upload',
        ]);

        $author = Role::firstOrCreate(['name' => 'author']);
        $author->syncPermissions([
            'posts.view', 'posts.create', 'posts.update',
            'media.upload',
        ]);

        $viewer = Role::firstOrCreate(['name' => 'viewer']);
        $viewer->syncPermissions(['posts.view']);
    }
}
```

---

## ขั้นตอนที่ 2292: การใช้งาน Roles และ Permissions

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Admin;

use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Spatie\Permission\Models\Role;
use Spatie\Permission\Models\Permission;

class RoleManagementController extends \App\Http\Controllers\Controller
{
    public function assignRole(Request $request, User $user): JsonResponse
    {
        $request->validate([
            'role' => ['required', 'string', 'exists:roles,name'],
        ]);

        $user->syncRoles([$request->role]);

        return response()->json([
            'message' => "กำหนด role '{$request->role}' ให้ {$user->name} เรียบร้อย",
            'roles' => $user->getRoleNames(),
        ]);
    }

    public function givePermission(Request $request, User $user): JsonResponse
    {
        $request->validate([
            'permissions' => ['required', 'array'],
            'permissions.*' => ['exists:permissions,name'],
        ]);

        $user->givePermissionTo($request->permissions);

        return response()->json([
            'message' => 'กำหนด permissions เรียบร้อย',
            'permissions' => $user->getAllPermissions()->pluck('name'),
        ]);
    }

    public function revokePermission(Request $request, User $user): JsonResponse
    {
        $request->validate([
            'permission' => ['required', 'string', 'exists:permissions,name'],
        ]);

        $user->revokePermissionTo($request->permission);

        return response()->json([
            'message' => "ถอน permission '{$request->permission}' เรียบร้อย",
        ]);
    }

    public function getRoles(): JsonResponse
    {
        $roles = Role::with('permissions:id,name')
            ->withCount('users')
            ->get();

        return response()->json(['data' => $roles]);
    }

    public function createRole(Request $request): JsonResponse
    {
        $request->validate([
            'name' => ['required', 'string', 'unique:roles,name'],
            'permissions' => ['array'],
            'permissions.*' => ['exists:permissions,name'],
        ]);

        $role = Role::create(['name' => $request->name]);

        if ($request->has('permissions')) {
            $role->syncPermissions($request->permissions);
        }

        return response()->json(['data' => $role->load('permissions')], 201);
    }
}
```

---

## ขั้นตอนที่ 2293: Permission Middleware และ Gates

```php
<?php
declare(strict_types=1);

// routes/api.php - ใช้ middleware
Route::middleware(['auth:sanctum'])->group(function () {
    // ใช้ role middleware
    Route::middleware('role:admin')->group(function () {
        Route::apiResource('users', UserController::class);
        Route::apiResource('roles', RoleController::class);
    });

    // ใช้ permission middleware
    Route::middleware('permission:posts.publish')->group(function () {
        Route::post('/posts/{post}/publish', [PostController::class, 'publish']);
    });

    // ใช้ role or permission
    Route::middleware('role_or_permission:admin|posts.manage')->group(function () {
        Route::delete('/posts/{post}', [PostController::class, 'destroy']);
    });
});
```

```php
<?php
declare(strict_types=1);

// app/Providers/AppServiceProvider.php
namespace App\Providers;

use Illuminate\Support\Facades\Gate;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Super admin bypasses all permission checks
        Gate::before(function ($user, $ability) {
            if ($user->hasRole('super-admin')) {
                return true;
            }
        });

        // Custom gate
        Gate::define('manage-post', function ($user, $post) {
            return $user->id === $post->user_id
                || $user->hasPermissionTo('posts.manage');
        });
    }
}
```

---

## ขั้นตอนที่ 2294: Spatie Media Library

```bash
composer require spatie/laravel-medialibrary
php artisan vendor:publish --provider="Spatie\MediaLibrary\MediaLibraryServiceProvider" --tag="medialibrary-migrations"
php artisan migrate
```

```php
<?php
declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Spatie\MediaLibrary\HasMedia;
use Spatie\MediaLibrary\InteractsWithMedia;
use Spatie\MediaLibrary\MediaCollections\Models\Media;
use Spatie\Image\Enums\Fit;

class Post extends Model implements HasMedia
{
    use InteractsWithMedia;

    public function registerMediaCollections(): void
    {
        $this->addMediaCollection('featured')
             ->singleFile() // มีได้แค่ไฟล์เดียว
             ->acceptsMimeTypes(['image/jpeg', 'image/png', 'image/webp']);

        $this->addMediaCollection('gallery')
             ->acceptsMimeTypes(['image/jpeg', 'image/png', 'image/webp', 'image/gif']);

        $this->addMediaCollection('attachments')
             ->acceptsMimeTypes(['application/pdf', 'application/msword']);
    }

    public function registerMediaConversions(?Media $media = null): void
    {
        $this->addMediaConversion('thumb')
             ->fit(Fit::Crop, 300, 300)
             ->performOnCollections('featured', 'gallery');

        $this->addMediaConversion('medium')
             ->fit(Fit::Max, 800, 600)
             ->performOnCollections('featured', 'gallery');

        $this->addMediaConversion('webp')
             ->format('webp')
             ->quality(80)
             ->performOnCollections('featured', 'gallery');
    }
}
```

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Spatie\MediaLibrary\MediaCollections\Exceptions\FileDoesNotExist;

class MediaController extends \App\Http\Controllers\Controller
{
    public function uploadFeatured(Request $request, Post $post): JsonResponse
    {
        $request->validate([
            'image' => ['required', 'image', 'max:5120', 'mimes:jpeg,png,webp'],
        ]);

        $this->authorize('update', $post);

        $media = $post->addMediaFromRequest('image')
                      ->usingFileName("{$post->slug}-featured.webp")
                      ->withCustomProperties([
                          'uploaded_by' => $request->user()->id,
                      ])
                      ->toMediaCollection('featured');

        return response()->json([
            'id' => $media->id,
            'original' => $media->getFullUrl(),
            'thumb' => $media->getFullUrl('thumb'),
            'medium' => $media->getFullUrl('medium'),
        ]);
    }

    public function uploadGallery(Request $request, Post $post): JsonResponse
    {
        $request->validate([
            'images' => ['required', 'array', 'max:10'],
            'images.*' => ['image', 'max:5120'],
        ]);

        $uploaded = [];
        foreach ($request->file('images') as $image) {
            $media = $post->addMedia($image)
                          ->toMediaCollection('gallery');

            $uploaded[] = [
                'id' => $media->id,
                'url' => $media->getFullUrl(),
                'thumb' => $media->getFullUrl('thumb'),
            ];
        }

        return response()->json(['data' => $uploaded]);
    }

    public function deleteMedia(Post $post, int $mediaId): JsonResponse
    {
        $this->authorize('update', $post);

        $media = $post->getMedia()->find($mediaId);

        if (! $media) {
            return response()->json(['message' => 'Media not found'], 404);
        }

        $media->delete();

        return response()->json(['message' => 'Media deleted']);
    }
}
```

---

## ขั้นตอนที่ 2295: Spatie Activity Log

```bash
composer require spatie/laravel-activitylog
php artisan vendor:publish --provider="Spatie\Activitylog\ActivitylogServiceProvider" --tag="activitylog-migrations"
php artisan migrate
```

```php
<?php
declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Spatie\Activitylog\LogOptions;
use Spatie\Activitylog\Traits\LogsActivity;

class Post extends Model
{
    use LogsActivity;

    public function getActivitylogOptions(): LogOptions
    {
        return LogOptions::defaults()
            ->logOnly(['title', 'content', 'status', 'slug'])
            ->logOnlyDirty()
            ->dontSubmitEmptyLogs()
            ->setDescriptionForEvent(fn (string $eventName) => "Post was {$eventName}")
            ->useLogName('posts');
    }
}
```

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Models\User;
use Illuminate\Support\Facades\Auth;
use Spatie\Activitylog\Models\Activity;

class ActivityService
{
    public function logLogin(User $user, string $ip): void
    {
        activity('auth')
            ->causedBy($user)
            ->withProperties([
                'ip' => $ip,
                'user_agent' => request()->userAgent(),
            ])
            ->log('User logged in');
    }

    public function logPayment(User $user, array $paymentData): void
    {
        activity('payments')
            ->causedBy($user)
            ->withProperties([
                'amount' => $paymentData['amount'],
                'currency' => $paymentData['currency'],
                'transaction_id' => $paymentData['transaction_id'],
            ])
            ->log('Payment completed');
    }

    public function getUserActivity(User $user, int $limit = 50): \Illuminate\Pagination\LengthAwarePaginator
    {
        return Activity::causedBy($user)
            ->latest()
            ->paginate($limit);
    }

    public function getRecentActivity(string $logName = null, int $days = 7): \Illuminate\Support\Collection
    {
        return Activity::query()
            ->when($logName, fn ($q) => $q->where('log_name', $logName))
            ->where('created_at', '>=', now()->subDays($days))
            ->with('causer')
            ->latest()
            ->get();
    }
}
```

---

## ขั้นตอนที่ 2296: Spatie Query Builder

```bash
composer require spatie/laravel-query-builder
```

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Spatie\QueryBuilder\AllowedFilter;
use Spatie\QueryBuilder\AllowedInclude;
use Spatie\QueryBuilder\AllowedSort;
use Spatie\QueryBuilder\QueryBuilder;

class PostQueryController extends \App\Http\Controllers\Controller
{
    public function index(Request $request): JsonResponse
    {
        $posts = QueryBuilder::for(Post::class)
            ->allowedFilters([
                AllowedFilter::exact('status'),
                AllowedFilter::partial('title'),
                AllowedFilter::scope('search'),
                AllowedFilter::exact('categories.id', 'category'),
                AllowedFilter::callback('published_after', function ($query, $date) {
                    $query->where('published_at', '>=', $date);
                }),
            ])
            ->allowedIncludes([
                AllowedInclude::relationship('author'),
                AllowedInclude::relationship('categories'),
                AllowedInclude::relationship('comments'),
                AllowedInclude::count('comments'),
                AllowedInclude::count('likes'),
            ])
            ->allowedSorts([
                AllowedSort::field('created_at'),
                AllowedSort::field('title'),
                AllowedSort::custom('popular', new \App\Sorts\PopularSort()),
                AllowedSort::custom('trending', new \App\Sorts\TrendingSort()),
            ])
            ->allowedFields([
                'id', 'title', 'slug', 'excerpt', 'status', 'created_at',
            ])
            ->defaultSort('-created_at')
            ->paginate($request->per_page ?? 15)
            ->appends($request->query());

        return response()->json($posts);
    }
}
```

---

## ขั้นตอนที่ 2297: Spatie Laravel Backup

```bash
composer require spatie/laravel-backup
php artisan vendor:publish --provider="Spatie\Backup\BackupServiceProvider"
```

```php
<?php
declare(strict_types=1);

// config/backup.php (สำคัญส่วน)
return [
    'backup' => [
        'name' => env('APP_NAME', 'laravel-backup'),
        'source' => [
            'files' => [
                'include' => [base_path()],
                'exclude' => [
                    base_path('vendor'),
                    base_path('node_modules'),
                    storage_path('logs'),
                ],
                'follow_links' => false,
                'ignore_unreadable_directories' => false,
                'relative_path' => null,
            ],
            'databases' => ['mysql'],
        ],
        'database_dump_compressor' => \Spatie\DbDumper\Compressors\GzipCompressor::class,
        'destination' => [
            'filename_prefix' => '',
            'disks' => ['s3'],
        ],
        'password' => env('BACKUP_ARCHIVE_PASSWORD'),
    ],

    'notifications' => [
        'notifiable' => \Spatie\Backup\Notifications\Notifiable::class,
        'mail' => [
            'to' => env('BACKUP_NOTIFICATION_EMAIL', 'admin@example.com'),
        ],
    ],

    'cleanup' => [
        'strategy' => \Spatie\Backup\Tasks\Cleanup\Strategies\DefaultStrategy::class,
        'default_strategy' => [
            'keep_all_backups_for_days' => 7,
            'keep_daily_backups_for_days' => 16,
            'keep_weekly_backups_for_weeks' => 8,
            'keep_monthly_backups_for_months' => 4,
            'delete_oldest_backups_when_using_more_megabytes_than' => 5000,
        ],
    ],
];
```

---

## สรุปบทที่ 81

| Package | ฟีเจอร์ | การติดตั้ง |
|---------|---------|-----------|
| laravel-permission | Roles & Permissions | HasRoles trait |
| laravel-medialibrary | File management | HasMedia interface |
| laravel-activitylog | Audit trail | LogsActivity trait |
| laravel-query-builder | Advanced filtering | QueryBuilder::for() |
| laravel-backup | Automated backups | Artisan command |
| laravel-sluggable | Auto slug | HasSlug trait |

ถัดไป → Part 82: Payment Systems Integration
