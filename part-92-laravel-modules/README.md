# Part 92: Laravel Modules
## ขั้นตอนที่ 2621-2650: Modular Laravel Applications

การสร้าง Laravel application แบบ Modular ด้วย nwidart/laravel-modules
ทำให้แต่ละ feature เป็น module ที่แยกจากกันได้

---

## ขั้นตอนที่ 2621: ติดตั้ง Laravel Modules

```bash
composer require nwidart/laravel-modules
php artisan vendor:publish --provider="Nwidart\Modules\LaravelModulesServiceProvider"

# สร้าง module ใหม่
php artisan module:make Blog
php artisan module:make Shop
php artisan module:make Auth

# โครงสร้าง module
# Modules/
# └── Blog/
#     ├── app/
#     │   ├── Http/Controllers/
#     │   ├── Models/
#     │   ├── Providers/
#     │   └── Services/
#     ├── config/
#     ├── database/
#     │   ├── factories/
#     │   ├── migrations/
#     │   └── seeders/
#     ├── resources/
#     │   ├── assets/
#     │   ├── lang/
#     │   └── views/
#     ├── routes/
#     │   ├── api.php
#     │   └── web.php
#     ├── tests/
#     ├── composer.json
#     └── module.json
```

---

## ขั้นตอนที่ 2622: Module Service Provider

```php
<?php
declare(strict_types=1);

namespace Modules\Blog\Providers;

use Illuminate\Support\ServiceProvider;
use Modules\Blog\Contracts\PostRepositoryInterface;
use Modules\Blog\Repositories\EloquentPostRepository;
use Modules\Blog\Services\PostService;
use Nwidart\Modules\Traits\PathNamespace;

class BlogServiceProvider extends ServiceProvider
{
    use PathNamespace;

    protected string $moduleName = 'Blog';
    protected string $moduleNameLower = 'blog';

    public function register(): void
    {
        $this->app->register(RouteServiceProvider::class);
        $this->app->register(EventServiceProvider::class);

        // Bind interfaces to implementations
        $this->app->bind(
            PostRepositoryInterface::class,
            EloquentPostRepository::class
        );

        // Register services
        $this->app->singleton(PostService::class, function ($app) {
            return new PostService(
                $app->make(PostRepositoryInterface::class),
                $app->make('events'),
            );
        });
    }

    public function boot(): void
    {
        $this->registerTranslations();
        $this->registerConfig();
        $this->registerViews();
        $this->loadMigrationsFrom(module_path($this->moduleName, 'database/migrations'));
    }

    protected function registerConfig(): void
    {
        $this->publishes([
            module_path($this->moduleName, 'config/config.php') => config_path($this->moduleNameLower . '.php'),
        ], 'config');

        $this->mergeConfigFrom(
            module_path($this->moduleName, 'config/config.php'),
            $this->moduleNameLower
        );
    }

    protected function registerViews(): void
    {
        $viewPath = resource_path('views/modules/' . $this->moduleNameLower);
        $sourcePath = module_path($this->moduleName, 'resources/views');

        $this->publishes([$sourcePath => $viewPath], ['views', $this->moduleNameLower . '-module-views']);
        $this->loadViewsFrom(array_merge($this->getPublishableViewPaths(), [$sourcePath]), $this->moduleNameLower);
    }

    protected function registerTranslations(): void
    {
        $langPath = resource_path('lang/modules/' . $this->moduleNameLower);
        if (is_dir($langPath)) {
            $this->loadTranslationsFrom($langPath, $this->moduleNameLower);
        } else {
            $this->loadTranslationsFrom(module_path($this->moduleName, 'resources/lang'), $this->moduleNameLower);
        }
    }

    private function getPublishableViewPaths(): array
    {
        $paths = [];
        foreach (config('view.paths') as $path) {
            if (is_dir($path . '/modules/' . $this->moduleNameLower)) {
                $paths[] = $path . '/modules/' . $this->moduleNameLower;
            }
        }
        return $paths;
    }
}
```

---

## ขั้นตอนที่ 2623: Blog Module - Post Repository

```php
<?php
declare(strict_types=1);

namespace Modules\Blog\Contracts;

use Illuminate\Pagination\LengthAwarePaginator;
use Modules\Blog\Models\Post;

interface PostRepositoryInterface
{
    public function paginate(int $perPage = 15, array $filters = []): LengthAwarePaginator;
    public function findBySlug(string $slug): ?Post;
    public function findById(int $id): ?Post;
    public function create(array $data): Post;
    public function update(Post $post, array $data): Post;
    public function delete(Post $post): bool;
    public function getPopular(int $limit = 10): \Illuminate\Support\Collection;
}
```

```php
<?php
declare(strict_types=1);

namespace Modules\Blog\Repositories;

use Illuminate\Pagination\LengthAwarePaginator;
use Modules\Blog\Contracts\PostRepositoryInterface;
use Modules\Blog\Models\Post;

class EloquentPostRepository implements PostRepositoryInterface
{
    public function paginate(int $perPage = 15, array $filters = []): LengthAwarePaginator
    {
        return Post::query()
            ->with(['author:id,name', 'categories:id,name,slug'])
            ->withCount('comments')
            ->when(isset($filters['status']), fn ($q) => $q->where('status', $filters['status']))
            ->when(isset($filters['search']), fn ($q) => $q->search($filters['search']))
            ->when(isset($filters['category']), fn ($q) => $q->inCategory($filters['category']))
            ->latest()
            ->paginate($perPage);
    }

    public function findBySlug(string $slug): ?Post
    {
        return Post::where('slug', $slug)
            ->with(['author', 'categories', 'comments.user'])
            ->first();
    }

    public function findById(int $id): ?Post
    {
        return Post::find($id);
    }

    public function create(array $data): Post
    {
        return Post::create($data);
    }

    public function update(Post $post, array $data): Post
    {
        $post->update($data);
        return $post->fresh();
    }

    public function delete(Post $post): bool
    {
        return (bool)$post->delete();
    }

    public function getPopular(int $limit = 10): \Illuminate\Support\Collection
    {
        return Post::published()
            ->orderByDesc('views')
            ->limit($limit)
            ->get(['id', 'title', 'slug', 'views']);
    }
}
```

---

## ขั้นตอนที่ 2624: Shop Module

```php
<?php
declare(strict_types=1);

namespace Modules\Shop\Services;

use Modules\Shop\Models\Cart;
use Modules\Shop\Models\Order;
use Modules\Shop\Models\Product;
use Modules\Shop\Events\OrderPlaced;
use Illuminate\Support\Facades\DB;

class CartService
{
    public function addItem(Cart $cart, int $productId, int $quantity = 1): Cart
    {
        $product = Product::findOrFail($productId);

        if ($product->stock < $quantity) {
            throw new \RuntimeException("สินค้าไม่เพียงพอ สต็อกเหลือ {$product->stock} ชิ้น");
        }

        $existingItem = $cart->items()->where('product_id', $productId)->first();

        if ($existingItem) {
            $existingItem->increment('quantity', $quantity);
        } else {
            $cart->items()->create([
                'product_id' => $productId,
                'quantity' => $quantity,
                'price' => $product->price,
            ]);
        }

        return $cart->fresh(['items.product']);
    }

    public function checkout(Cart $cart, array $orderData): Order
    {
        return DB::transaction(function () use ($cart, $orderData) {
            // ตรวจสอบ stock อีกครั้ง
            foreach ($cart->items as $item) {
                if ($item->product->stock < $item->quantity) {
                    throw new \RuntimeException(
                        "สินค้า {$item->product->name} มีสต็อกไม่เพียงพอ"
                    );
                }
            }

            // สร้าง order
            $order = Order::create([
                'user_id' => $cart->user_id,
                'subtotal' => $cart->subtotal,
                'shipping' => $orderData['shipping_cost'] ?? 0,
                'total' => $cart->subtotal + ($orderData['shipping_cost'] ?? 0),
                'status' => 'pending',
                'shipping_address' => $orderData['shipping_address'],
            ]);

            // สร้าง order items และลด stock
            foreach ($cart->items as $item) {
                $order->items()->create([
                    'product_id' => $item->product_id,
                    'quantity' => $item->quantity,
                    'price' => $item->price,
                    'total' => $item->price * $item->quantity,
                ]);

                $item->product->decrement('stock', $item->quantity);
            }

            // ล้าง cart
            $cart->items()->delete();
            $cart->update(['coupon_id' => null]);

            event(new OrderPlaced($order));

            return $order;
        });
    }

    public function applyCoupon(Cart $cart, string $code): array
    {
        $coupon = \Modules\Shop\Models\Coupon::where('code', $code)
            ->where('is_active', true)
            ->where(fn ($q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', now()))
            ->first();

        if (!$coupon) {
            throw new \RuntimeException('Coupon ไม่ถูกต้องหรือหมดอายุ');
        }

        $discount = match ($coupon->type) {
            'percentage' => $cart->subtotal * ($coupon->value / 100),
            'fixed' => min($coupon->value, $cart->subtotal),
            default => 0,
        };

        $cart->update(['coupon_id' => $coupon->id, 'discount' => $discount]);

        return [
            'discount' => $discount,
            'total' => $cart->subtotal - $discount,
        ];
    }
}
```

---

## ขั้นตอนที่ 2625: Inter-module Communication

```php
<?php
declare(strict_types=1);

// การสื่อสารระหว่าง modules ด้วย Events

// Modules/Shop/Events/OrderPlaced.php
namespace Modules\Shop\Events;

use Modules\Shop\Models\Order;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderPlaced
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Order $order
    ) {}
}

// Modules/Notification/Listeners/SendOrderConfirmation.php
namespace Modules\Notification\Listeners;

use Modules\Shop\Events\OrderPlaced;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendOrderConfirmation implements ShouldQueue
{
    public function handle(OrderPlaced $event): void
    {
        $order = $event->order;
        $user = $order->user;

        // ส่ง email confirmation
        \Mail::to($user->email)->send(
            new \Modules\Notification\Mail\OrderConfirmation($order)
        );
    }
}

// Modules/Inventory/Listeners/UpdateInventory.php
namespace Modules\Inventory\Listeners;

use Modules\Shop\Events\OrderPlaced;

class UpdateInventory
{
    public function handle(OrderPlaced $event): void
    {
        foreach ($event->order->items as $item) {
            \Modules\Inventory\Models\InventoryLog::create([
                'product_id' => $item->product_id,
                'type' => 'sale',
                'quantity' => -$item->quantity,
                'reference' => 'order_' . $event->order->id,
            ]);
        }
    }
}
```

---

## ขั้นตอนที่ 2626: Module Testing

```php
<?php
declare(strict_types=1);

namespace Modules\Blog\Tests\Feature;

use Modules\Blog\Models\Post;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class PostTest extends TestCase
{
    use RefreshDatabase;

    public function test_can_list_published_posts(): void
    {
        Post::factory()->published()->count(5)->create();
        Post::factory()->draft()->count(3)->create();

        $response = $this->getJson('/api/blog/posts');

        $response->assertOk()
                 ->assertJsonCount(5, 'data')
                 ->assertJsonStructure([
                     'data' => [['id', 'title', 'slug', 'excerpt']],
                     'meta' => ['total', 'per_page'],
                 ]);
    }

    public function test_can_create_post_as_author(): void
    {
        $user = \App\Models\User::factory()->create();
        $user->assignRole('author');

        $data = [
            'title' => 'Test Post',
            'content' => str_repeat('Lorem ipsum ', 50),
            'status' => 'published',
        ];

        $response = $this->actingAs($user, 'sanctum')
                         ->postJson('/api/blog/posts', $data);

        $response->assertCreated()
                 ->assertJsonPath('data.title', 'Test Post');

        $this->assertDatabaseHas('posts', ['title' => 'Test Post']);
    }

    public function test_unauthorized_user_cannot_create_post(): void
    {
        $response = $this->postJson('/api/blog/posts', [
            'title' => 'Test',
            'content' => str_repeat('Lorem ipsum ', 50),
        ]);

        $response->assertUnauthorized();
    }
}
```

---

## สรุปบทที่ 92

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|----------|---------|
| Module Structure | nwidart/laravel-modules | Organized code |
| Service Provider | register/boot | Module initialization |
| Repository Pattern | Interface + Eloquent | Decoupled data layer |
| Inter-module Events | Laravel Events | Loose coupling |
| Module Testing | PHPUnit | Isolated tests |
| Lazy Loading | ServiceProvider | Performance |

ถัดไป → Part 93: Blockchain กับ PHP
