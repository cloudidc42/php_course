# Part 75: Architecture Patterns

## ขั้นตอนที่ 2111-2140: Architecture Patterns ใน PHP

Architecture patterns ที่ดีทำให้ code maintainable, testable และ scalable ในระยะยาว

---

## ขั้นตอนที่ 2111: Hexagonal Architecture (Ports and Adapters)

```php
<?php

declare(strict_types=1);

// Domain Layer - ไม่รู้จัก infrastructure เลย

// src/Domain/Order/Ports/OrderRepositoryInterface.php (Port)
namespace App\Domain\Order\Ports;

use App\Domain\Order\Entities\Order;
use App\Domain\Order\ValueObjects\OrderId;

interface OrderRepositoryInterface
{
    public function findById(OrderId $id): ?Order;
    public function save(Order $order): void;
    public function delete(OrderId $id): void;
    public function findByCustomerId(int $customerId): array;
}

// src/Domain/Order/Ports/PaymentGatewayInterface.php (Port)
namespace App\Domain\Order\Ports;

use App\Domain\Order\ValueObjects\Money;

interface PaymentGatewayInterface
{
    public function charge(string $customerId, Money $amount): string; // returns transaction ID
    public function refund(string $transactionId, Money $amount): void;
}

// src/Domain/Order/Ports/NotificationServiceInterface.php (Port)
namespace App\Domain\Order\Ports;

use App\Domain\Order\Entities\Order;

interface NotificationServiceInterface
{
    public function notifyOrderPlaced(Order $order): void;
    public function notifyOrderShipped(Order $order, string $trackingNumber): void;
}
```

```php
<?php

declare(strict_types=1);

// Domain Use Case
// src/Domain/Order/UseCases/PlaceOrderUseCase.php
namespace App\Domain\Order\UseCases;

use App\Domain\Order\Entities\Order;
use App\Domain\Order\Ports\OrderRepositoryInterface;
use App\Domain\Order\Ports\PaymentGatewayInterface;
use App\Domain\Order\Ports\NotificationServiceInterface;
use App\Domain\Order\ValueObjects\OrderId;
use App\Domain\Order\ValueObjects\Money;

class PlaceOrderUseCase
{
    public function __construct(
        private OrderRepositoryInterface    $orderRepository,
        private PaymentGatewayInterface     $paymentGateway,
        private NotificationServiceInterface $notificationService,
    ) {}

    public function execute(PlaceOrderCommand $command): Order
    {
        // Pure business logic - ไม่รู้จัก Eloquent, Stripe, Mail
        $order = Order::create(
            id:         OrderId::generate(),
            customerId: $command->customerId,
            items:      $command->items,
            total:      Money::fromAmount($command->totalAmount, 'THB'),
        );

        // Charge via port (adapter handles Stripe details)
        $transactionId = $this->paymentGateway->charge(
            $command->stripeCustomerId,
            $order->getTotal()
        );

        $order->markAsPaid($transactionId);

        // Save via port (adapter handles Eloquent details)
        $this->orderRepository->save($order);

        // Notify via port (adapter handles Mail/SMS details)
        $this->notificationService->notifyOrderPlaced($order);

        return $order;
    }
}
```

```php
<?php

declare(strict_types=1);

// Adapters - Infrastructure implementations of Ports

// src/Infrastructure/Adapters/EloquentOrderRepository.php (Adapter)
namespace App\Infrastructure\Adapters;

use App\Domain\Order\Entities\Order;
use App\Domain\Order\Ports\OrderRepositoryInterface;
use App\Domain\Order\ValueObjects\OrderId;
use App\Models\Order as EloquentOrder;

class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function findById(OrderId $id): ?Order
    {
        $eloquentOrder = EloquentOrder::find($id->value());

        if (!$eloquentOrder) return null;

        return $this->toDomainEntity($eloquentOrder);
    }

    public function save(Order $order): void
    {
        EloquentOrder::updateOrCreate(
            ['id' => $order->getId()->value()],
            $this->toEloquentData($order)
        );
    }

    public function delete(OrderId $id): void
    {
        EloquentOrder::destroy($id->value());
    }

    public function findByCustomerId(int $customerId): array
    {
        return EloquentOrder::where('customer_id', $customerId)
            ->get()
            ->map(fn ($o) => $this->toDomainEntity($o))
            ->toArray();
    }

    private function toDomainEntity(EloquentOrder $eloquent): Order
    {
        return Order::reconstitute(
            id:            OrderId::from($eloquent->id),
            customerId:    $eloquent->customer_id,
            total:         \App\Domain\Order\ValueObjects\Money::fromAmount($eloquent->total, 'THB'),
            status:        $eloquent->status,
            transactionId: $eloquent->transaction_id,
        );
    }

    private function toEloquentData(Order $order): array
    {
        return [
            'customer_id'    => $order->getCustomerId(),
            'total'          => $order->getTotal()->amount(),
            'status'         => $order->getStatus(),
            'transaction_id' => $order->getTransactionId(),
        ];
    }
}

// src/Infrastructure/Adapters/StripePaymentGateway.php (Adapter)
namespace App\Infrastructure\Adapters;

use App\Domain\Order\Ports\PaymentGatewayInterface;
use App\Domain\Order\ValueObjects\Money;
use Stripe\PaymentIntent;
use Stripe\StripeClient;

class StripePaymentGateway implements PaymentGatewayInterface
{
    private StripeClient $stripe;

    public function __construct()
    {
        $this->stripe = new StripeClient(config('cashier.secret'));
    }

    public function charge(string $customerId, Money $amount): string
    {
        $intent = $this->stripe->paymentIntents->create([
            'amount'   => (int) ($amount->amount() * 100), // to satangs
            'currency' => strtolower($amount->currency()),
            'customer' => $customerId,
            'confirm'  => true,
        ]);

        return $intent->id;
    }

    public function refund(string $transactionId, Money $amount): void
    {
        $this->stripe->refunds->create([
            'payment_intent' => $transactionId,
            'amount'         => (int) ($amount->amount() * 100),
        ]);
    }
}
```

---

## ขั้นตอนที่ 2112: Clean Architecture

```php
<?php

declare(strict_types=1);

/**
 * Clean Architecture layers:
 * 1. Entities (innermost) - Business rules
 * 2. Use Cases - Application business rules
 * 3. Interface Adapters - Controllers, Presenters, Gateways
 * 4. Frameworks & Drivers (outermost) - Laravel, MySQL, etc.
 *
 * Dependency rule: source code dependencies only point INWARD
 */

// Layer 1: Entities
// src/Domain/User/Entity/User.php
namespace App\Domain\User\Entity;

use App\Domain\User\ValueObject\Email;
use App\Domain\User\ValueObject\UserId;

final class User
{
    private function __construct(
        private readonly UserId $id,
        private Email           $email,
        private string          $name,
        private bool            $isActive,
    ) {}

    public static function create(Email $email, string $name): self
    {
        return new self(
            id:       UserId::generate(),
            email:    $email,
            name:     $name,
            isActive: true,
        );
    }

    public function changeEmail(Email $newEmail): void
    {
        if ($newEmail->equals($this->email)) {
            throw new \DomainException('New email must be different');
        }

        $this->email = $newEmail;
    }

    public function deactivate(): void
    {
        if (!$this->isActive) {
            throw new \DomainException('User is already inactive');
        }
        $this->isActive = false;
    }

    public function getId(): UserId     { return $this->id; }
    public function getEmail(): Email   { return $this->email; }
    public function getName(): string   { return $this->name; }
    public function isActive(): bool    { return $this->isActive; }
}

// Layer 2: Use Cases
// src/Application/User/ChangeEmail/ChangeEmailUseCase.php
namespace App\Application\User\ChangeEmail;

use App\Domain\User\Port\UserRepositoryInterface;
use App\Domain\User\ValueObject\Email;
use App\Domain\User\ValueObject\UserId;

class ChangeEmailUseCase
{
    public function __construct(
        private UserRepositoryInterface $userRepository
    ) {}

    public function execute(ChangeEmailRequest $request): ChangeEmailResponse
    {
        $user = $this->userRepository->findById(UserId::from($request->userId));

        if (!$user) {
            throw new \DomainException("User {$request->userId} not found");
        }

        $newEmail = Email::from($request->newEmail);
        $user->changeEmail($newEmail);

        $this->userRepository->save($user);

        return new ChangeEmailResponse(
            userId:   $user->getId()->value(),
            newEmail: $user->getEmail()->value(),
        );
    }
}
```

---

## ขั้นตอนที่ 2113: Vertical Slice Architecture

```php
<?php

declare(strict_types=1);

/**
 * Vertical Slice: แบ่งตาม feature ไม่ใช่ layer
 * app/Features/CreateOrder/ มีทุกอย่างที่ต้องการ
 */

// app/Features/CreateOrder/CreateOrderCommand.php
namespace App\Features\CreateOrder;

final readonly class CreateOrderCommand
{
    public function __construct(
        public int    $customerId,
        public array  $items,
        public string $paymentMethodId,
        public array  $shippingAddress,
    ) {}
}

// app/Features/CreateOrder/CreateOrderHandler.php
namespace App\Features\CreateOrder;

use App\Events\OrderCreated;
use App\Models\Order;
use App\Models\OrderItem;
use App\Models\Product;
use Illuminate\Support\Facades\DB;

class CreateOrderHandler
{
    public function handle(CreateOrderCommand $command): CreateOrderResult
    {
        return DB::transaction(function () use ($command): CreateOrderResult {
            // Validate stock
            foreach ($command->items as $item) {
                $product = Product::findOrFail($item['product_id']);

                if ($product->stock_quantity < $item['quantity']) {
                    throw new \DomainException(
                        "Insufficient stock for: {$product->name}"
                    );
                }
            }

            // Calculate total
            $total = collect($command->items)->sum(function ($item) {
                return Product::find($item['product_id'])->price * $item['quantity'];
            });

            // Create order
            $order = Order::create([
                'customer_id'      => $command->customerId,
                'total'            => $total,
                'status'           => 'pending',
                'shipping_address' => $command->shippingAddress,
            ]);

            // Create items
            foreach ($command->items as $item) {
                $product = Product::find($item['product_id']);

                OrderItem::create([
                    'order_id'   => $order->id,
                    'product_id' => $product->id,
                    'quantity'   => $item['quantity'],
                    'unit_price' => $product->price,
                ]);

                // Decrement stock
                $product->decrement('stock_quantity', $item['quantity']);
            }

            event(new OrderCreated($order));

            return new CreateOrderResult(
                orderId:      $order->id,
                orderNumber:  $order->order_number,
                total:        $total,
            );
        });
    }
}

// app/Features/CreateOrder/CreateOrderController.php
namespace App\Features\CreateOrder;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class CreateOrderController
{
    public function __construct(private CreateOrderHandler $handler) {}

    public function __invoke(Request $request): JsonResponse
    {
        $request->validate([
            'items'                     => 'required|array|min:1',
            'items.*.product_id'        => 'required|integer|exists:products,id',
            'items.*.quantity'          => 'required|integer|min:1',
            'payment_method_id'         => 'required|string',
            'shipping_address'          => 'required|array',
            'shipping_address.name'     => 'required|string',
            'shipping_address.address'  => 'required|string',
            'shipping_address.city'     => 'required|string',
        ]);

        $command = new CreateOrderCommand(
            customerId:      $request->user()->id,
            items:           $request->items,
            paymentMethodId: $request->payment_method_id,
            shippingAddress: $request->shipping_address,
        );

        $result = $this->handler->handle($command);

        return response()->json([
            'order_id'     => $result->orderId,
            'order_number' => $result->orderNumber,
            'total'        => $result->total,
        ], 201);
    }
}
```

---

## ขั้นตอนที่ 2114: Modular Monolith

```php
<?php

declare(strict_types=1);

/**
 * Modular Monolith: modules ที่มี clear boundaries
 * แต่ deploy เป็น single application
 *
 * app/Modules/
 *   ├── Catalog/      (Products, Categories)
 *   ├── Orders/       (Orders, OrderItems)
 *   ├── Payments/     (Payments, Invoices)
 *   ├── Customers/    (Users, Addresses)
 *   └── Notifications/ (Email, SMS, Push)
 */

// app/Modules/Catalog/CatalogModule.php
namespace App\Modules\Catalog;

use Illuminate\Support\ServiceProvider;

class CatalogModule extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            \App\Modules\Catalog\Contracts\ProductRepositoryInterface::class,
            \App\Modules\Catalog\Repositories\EloquentProductRepository::class
        );
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(__DIR__ . '/database/migrations');
        $this->loadRoutesFrom(__DIR__ . '/routes.php');
        $this->loadViewsFrom(__DIR__ . '/resources/views', 'catalog');
    }
}

// Module contracts สำหรับ cross-module communication
// app/Modules/Catalog/Contracts/ProductRepositoryInterface.php
namespace App\Modules\Catalog\Contracts;

interface ProductRepositoryInterface
{
    public function findById(int $id): ?array;
    public function decrementStock(int $productId, int $quantity): void;
    public function checkAvailability(int $productId, int $quantity): bool;
}

// Orders module ใช้ Catalog interface (ไม่ depend on implementation)
// app/Modules/Orders/Services/OrderService.php
namespace App\Modules\Orders\Services;

use App\Modules\Catalog\Contracts\ProductRepositoryInterface;

class OrderService
{
    public function __construct(
        private ProductRepositoryInterface $productRepository
    ) {}

    public function validateAndPlaceOrder(array $items): void
    {
        foreach ($items as $item) {
            if (!$this->productRepository->checkAvailability($item['product_id'], $item['quantity'])) {
                throw new \DomainException("Product {$item['product_id']} not available");
            }
        }

        // Place order logic...

        // Update inventory (through contract, not direct DB)
        foreach ($items as $item) {
            $this->productRepository->decrementStock($item['product_id'], $item['quantity']);
        }
    }
}
```

---

## ขั้นตอนที่ 2115: Dependency Inversion ใน Practice

```php
<?php

declare(strict_types=1);

// BAD: High-level depends on Low-level
class OrderService_BAD
{
    private \App\Models\Order $orderModel;    // Depends on Eloquent
    private \Stripe\StripeClient $stripe;    // Depends on Stripe SDK
    private \Illuminate\Mail\Mailer $mailer; // Depends on Laravel Mailer

    public function placeOrder(array $data): void
    {
        // Tightly coupled - hard to test, hard to change
        $this->stripe->paymentIntents->create([...]);
        $this->orderModel->create($data);
        $this->mailer->send(...);
    }
}

// GOOD: Dependency Inversion Principle
interface OrderStorageInterface
{
    public function persist(array $data): int;
}

interface PaymentProcessorInterface
{
    public function process(float $amount, string $currency): string;
}

interface OrderNotifierInterface
{
    public function notifyCreated(int $orderId, string $customerEmail): void;
}

class OrderService_GOOD
{
    public function __construct(
        private OrderStorageInterface    $storage,
        private PaymentProcessorInterface $payment,
        private OrderNotifierInterface    $notifier,
    ) {}

    public function placeOrder(array $data): int
    {
        $transactionId = $this->payment->process($data['total'], 'THB');

        $orderId = $this->storage->persist(array_merge($data, [
            'transaction_id' => $transactionId,
        ]));

        $this->notifier->notifyCreated($orderId, $data['customer_email']);

        return $orderId;
    }
}

// Testing is easy! Just swap implementations
class InMemoryOrderStorage implements OrderStorageInterface
{
    private array $orders = [];

    public function persist(array $data): int
    {
        $id = count($this->orders) + 1;
        $this->orders[$id] = $data;
        return $id;
    }
}

class FakePaymentProcessor implements PaymentProcessorInterface
{
    public function process(float $amount, string $currency): string
    {
        return 'fake_transaction_' . time();
    }
}
```

---

## ขั้นตอนที่ 2116: Module Boundary Examples

```php
<?php

declare(strict_types=1);

// app/Providers/ModuleServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class ModuleServiceProvider extends ServiceProvider
{
    private array $modules = [
        \App\Modules\Catalog\CatalogModule::class,
        \App\Modules\Orders\OrdersModule::class,
        \App\Modules\Payments\PaymentsModule::class,
        \App\Modules\Customers\CustomersModule::class,
        \App\Modules\Notifications\NotificationsModule::class,
    ];

    public function register(): void
    {
        foreach ($this->modules as $module) {
            $this->app->register($module);
        }
    }
}

// Cross-module event approach (loose coupling)
// Orders module fires event, Notifications module listens
// No direct dependency between modules

// app/Modules/Orders/Events/OrderPlaced.php
namespace App\Modules\Orders\Events;

final class OrderPlaced
{
    public function __construct(
        public readonly int    $orderId,
        public readonly int    $customerId,
        public readonly float  $total,
        public readonly string $customerEmail,
    ) {}
}

// app/Modules/Notifications/Listeners/SendOrderConfirmation.php
namespace App\Modules\Notifications\Listeners;

use App\Modules\Orders\Events\OrderPlaced;

class SendOrderConfirmation
{
    public function handle(OrderPlaced $event): void
    {
        // Notifications module handles this without knowing Order internals
        \Mail::to($event->customerEmail)->send(
            new \App\Modules\Notifications\Mail\OrderConfirmation($event->orderId, $event->total)
        );
    }
}
```

---

## สรุป Part 75

| Pattern | ข้อดี | เมื่อใช้ |
|---------|-------|---------|
| Hexagonal | Testable, replaceable adapters | Complex business logic |
| Clean Architecture | Clear layer separation | Enterprise apps |
| Vertical Slice | Feature-focused | Medium-size teams |
| Modular Monolith | Bounded contexts | กำลัง grow, ยังไม่ต้องการ microservices |
| DIP | Loose coupling | ทุก application |

---

## สรุปหลักสูตร Parts 56-75

| Part | หัวข้อ | ขั้นตอน |
|------|--------|---------|
| 56 | Laravel Filament Admin | 1541-1570 |
| 57 | Elasticsearch Search | 1571-1600 |
| 58 | Advanced Caching | 1601-1630 |
| 59 | Event-Driven Architecture | 1631-1660 |
| 60 | Laravel Cashier + Stripe | 1661-1690 |
| 61 | PHP-FPM + Nginx | 1691-1720 |
| 62 | Laravel Scout | 1721-1750 |
| 63 | Real-Time Applications | 1751-1780 |
| 64 | Multi-Tenancy | 1781-1810 |
| 65 | WordPress Multisite | 1811-1840 |
| 66 | Drupal Advanced | 1841-1870 |
| 67 | PHP Concurrency | 1871-1900 |
| 68 | API Gateway | 1901-1930 |
| 69 | Database Scaling | 1931-1960 |
| 70 | Advanced CI/CD | 1961-1990 |
| 71 | Security Audit | 1991-2020 |
| 72 | Laravel Dusk E2E | 2021-2050 |
| 73 | Composer Packages | 2051-2080 |
| 74 | PHP Extensions | 2081-2110 |
| 75 | Architecture Patterns | 2111-2140 |

ถัดไป → Part 76: Microservices กับ PHP
