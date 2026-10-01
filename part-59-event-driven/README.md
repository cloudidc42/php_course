# Part 59: Event-Driven Architecture

## ขั้นตอนที่ 1631-1660: Event-Driven Architecture ใน PHP

Event-Driven Architecture แยก concerns ออกจากกัน ทำให้ระบบ scalable และ maintainable มากขึ้น

---

## ขั้นตอนที่ 1631: Domain Events พื้นฐาน

```php
<?php

declare(strict_types=1);

// app/Domain/Order/Events/OrderPlaced.php
namespace App\Domain\Order\Events;

use App\Domain\Order\ValueObjects\OrderId;
use App\Domain\Order\ValueObjects\Money;
use DateTimeImmutable;

final class OrderPlaced
{
    public function __construct(
        public readonly OrderId           $orderId,
        public readonly int               $customerId,
        public readonly Money             $total,
        public readonly array             $items,
        public readonly DateTimeImmutable $occurredAt,
    ) {}

    public static function raise(
        OrderId $orderId,
        int $customerId,
        Money $total,
        array $items,
    ): self {
        return new self(
            orderId:    $orderId,
            customerId: $customerId,
            total:      $total,
            items:      $items,
            occurredAt: new DateTimeImmutable(),
        );
    }
}

// app/Domain/Order/Events/OrderCancelled.php
final class OrderCancelled
{
    public function __construct(
        public readonly OrderId           $orderId,
        public readonly string            $reason,
        public readonly DateTimeImmutable $occurredAt,
    ) {}
}

// app/Domain/Order/Events/OrderShipped.php
final class OrderShipped
{
    public function __construct(
        public readonly OrderId           $orderId,
        public readonly string            $trackingNumber,
        public readonly string            $carrier,
        public readonly DateTimeImmutable $occurredAt,
    ) {}
}
```

---

## ขั้นตอนที่ 1632: Event Bus Implementation

```php
<?php

declare(strict_types=1);

// app/Infrastructure/EventBus/EventBusInterface.php
namespace App\Infrastructure\EventBus;

interface EventBusInterface
{
    public function publish(object $event): void;
    public function subscribe(string $eventClass, callable $handler): void;
}

// app/Infrastructure/EventBus/SynchronousEventBus.php
namespace App\Infrastructure\EventBus;

use Illuminate\Support\Facades\Log;
use Throwable;

class SynchronousEventBus implements EventBusInterface
{
    private array $handlers = [];

    public function publish(object $event): void
    {
        $eventClass = get_class($event);
        $handlers   = $this->handlers[$eventClass] ?? [];

        foreach ($handlers as $handler) {
            try {
                $handler($event);
            } catch (Throwable $e) {
                Log::error("Event handler failed for {$eventClass}: " . $e->getMessage(), [
                    'exception' => $e,
                    'event'     => $event,
                ]);
            }
        }
    }

    public function subscribe(string $eventClass, callable $handler): void
    {
        $this->handlers[$eventClass][] = $handler;
    }
}

// app/Infrastructure/EventBus/LaravelEventBus.php
// ใช้ Laravel's built-in event system
namespace App\Infrastructure\EventBus;

use Illuminate\Events\Dispatcher;

class LaravelEventBus implements EventBusInterface
{
    public function __construct(private Dispatcher $events) {}

    public function publish(object $event): void
    {
        $this->events->dispatch($event);
    }

    public function subscribe(string $eventClass, callable $handler): void
    {
        $this->events->listen($eventClass, $handler);
    }
}
```

---

## ขั้นตอนที่ 1633: Domain Model กับ Event Sourcing แนวคิด

```php
<?php

declare(strict_types=1);

// app/Domain/Order/Aggregates/Order.php
namespace App\Domain\Order\Aggregates;

use App\Domain\Order\Events\OrderPlaced;
use App\Domain\Order\Events\OrderCancelled;
use App\Domain\Order\Events\OrderShipped;
use App\Domain\Order\ValueObjects\OrderId;
use App\Domain\Order\ValueObjects\Money;
use DomainException;

final class Order
{
    private OrderId $id;
    private int $customerId;
    private string $status;
    private Money $total;
    private array $items = [];
    private array $domainEvents = [];

    private function __construct() {}

    public static function place(
        OrderId $orderId,
        int $customerId,
        array $items,
        Money $total,
    ): self {
        $order = new self();
        $order->apply(OrderPlaced::raise($orderId, $customerId, $total, $items));
        return $order;
    }

    public function cancel(string $reason): void
    {
        if (!in_array($this->status, ['pending', 'processing'])) {
            throw new DomainException("Cannot cancel order in status: {$this->status}");
        }

        $this->apply(new OrderCancelled($this->id, $reason, new \DateTimeImmutable()));
    }

    public function ship(string $trackingNumber, string $carrier): void
    {
        if ($this->status !== 'processing') {
            throw new DomainException("Can only ship orders in 'processing' status");
        }

        $this->apply(new OrderShipped($this->id, $trackingNumber, $carrier, new \DateTimeImmutable()));
    }

    private function apply(object $event): void
    {
        $this->domainEvents[] = $event;
        $this->when($event);
    }

    private function when(object $event): void
    {
        match (get_class($event)) {
            OrderPlaced::class    => $this->applyOrderPlaced($event),
            OrderCancelled::class => $this->applyOrderCancelled($event),
            OrderShipped::class   => $this->applyOrderShipped($event),
            default               => null,
        };
    }

    private function applyOrderPlaced(OrderPlaced $event): void
    {
        $this->id         = $event->orderId;
        $this->customerId = $event->customerId;
        $this->total      = $event->total;
        $this->items      = $event->items;
        $this->status     = 'pending';
    }

    private function applyOrderCancelled(OrderCancelled $event): void
    {
        $this->status = 'cancelled';
    }

    private function applyOrderShipped(OrderShipped $event): void
    {
        $this->status = 'shipped';
    }

    public function pullDomainEvents(): array
    {
        $events             = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }

    public function getStatus(): string { return $this->status; }
    public function getId(): OrderId    { return $this->id; }
    public function getTotal(): Money   { return $this->total; }
}
```

---

## ขั้นตอนที่ 1634: Outbox Pattern สำหรับ Reliable Event Publishing

```php
<?php

declare(strict_types=1);

// database/migrations/create_outbox_messages_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('outbox_messages', function (Blueprint $table) {
            $table->id();
            $table->string('aggregate_type');
            $table->string('aggregate_id');
            $table->string('event_type');
            $table->json('payload');
            $table->timestamp('created_at')->useCurrent();
            $table->timestamp('processed_at')->nullable();
            $table->integer('retry_count')->default(0);
            $table->text('last_error')->nullable();

            $table->index(['processed_at', 'created_at']);
        });
    }
};
```

```php
<?php

declare(strict_types=1);

// app/Infrastructure/Outbox/OutboxPublisher.php
namespace App\Infrastructure\Outbox;

use App\Models\OutboxMessage;
use Illuminate\Support\Facades\DB;

class OutboxPublisher
{
    /**
     * บันทึก domain events ลง outbox table ภายใน transaction เดียวกับ business data
     * นี่คือหัวใจของ Outbox Pattern - ไม่มี 2-phase commit
     */
    public function store(string $aggregateType, string $aggregateId, array $events): void
    {
        // This MUST run inside the same transaction as the business operation
        foreach ($events as $event) {
            OutboxMessage::create([
                'aggregate_type' => $aggregateType,
                'aggregate_id'   => $aggregateId,
                'event_type'     => get_class($event),
                'payload'        => json_encode($this->serializeEvent($event)),
            ]);
        }
    }

    private function serializeEvent(object $event): array
    {
        return array_map(
            fn ($value) => $value instanceof \DateTimeInterface ? $value->format(\DateTime::ISO8601) : $value,
            get_object_vars($event)
        );
    }
}

// app/Infrastructure/Outbox/OutboxPoller.php
namespace App\Infrastructure\Outbox;

use App\Models\OutboxMessage;
use Illuminate\Support\Facades\Log;

class OutboxPoller
{
    private const BATCH_SIZE = 100;
    private const MAX_RETRIES = 5;

    public function __construct(
        private \App\Infrastructure\EventBus\EventBusInterface $eventBus
    ) {}

    public function poll(): int
    {
        $processed = 0;

        OutboxMessage::where('processed_at', null)
            ->where('retry_count', '<', self::MAX_RETRIES)
            ->orderBy('created_at')
            ->limit(self::BATCH_SIZE)
            ->get()
            ->each(function (OutboxMessage $message) use (&$processed) {
                try {
                    $this->processMessage($message);
                    $processed++;
                } catch (\Throwable $e) {
                    $message->increment('retry_count');
                    $message->update(['last_error' => $e->getMessage()]);
                    Log::warning("Outbox message {$message->id} failed: " . $e->getMessage());
                }
            });

        return $processed;
    }

    private function processMessage(OutboxMessage $message): void
    {
        $eventClass = $message->event_type;
        $payload    = json_decode($message->payload, true);

        // Re-hydrate event from payload
        $event = $this->deserializeEvent($eventClass, $payload);

        // Publish to event bus
        $this->eventBus->publish($event);

        // Mark as processed
        $message->update(['processed_at' => now()]);
    }

    private function deserializeEvent(string $class, array $payload): object
    {
        // ใน production ควรใช้ proper deserializer
        return new $class(...$payload);
    }
}
```

---

## ขั้นตอนที่ 1635: Saga Pattern - Orchestration

```php
<?php

declare(strict_types=1);

// app/Domain/Order/Sagas/OrderFulfillmentSaga.php
namespace App\Domain\Order\Sagas;

use App\Domain\Order\Events\OrderPlaced;
use App\Domain\Order\Events\OrderCancelled;
use App\Domain\Payment\Events\PaymentProcessed;
use App\Domain\Payment\Events\PaymentFailed;
use App\Domain\Inventory\Events\InventoryReserved;
use App\Domain\Inventory\Events\InventoryReservationFailed;
use App\Domain\Shipping\Commands\CreateShipment;
use App\Domain\Payment\Commands\ProcessPayment;
use App\Domain\Inventory\Commands\ReserveInventory;
use App\Domain\Inventory\Commands\ReleaseInventory;

/**
 * Saga Orchestrator - ควบคุม workflow ของ order fulfillment
 * ถ้า step ใด fail จะทำ compensating transactions
 */
class OrderFulfillmentSaga
{
    private array $state = [];

    public function handle(OrderPlaced $event): void
    {
        $this->state[$event->orderId->value()] = [
            'order_id'  => $event->orderId->value(),
            'status'    => 'started',
            'steps'     => [],
        ];

        // Step 1: Reserve inventory
        dispatch(new ReserveInventory($event->orderId, $event->items));
    }

    public function handleInventoryReserved(InventoryReserved $event): void
    {
        $this->updateState($event->orderId->value(), 'inventory_reserved');

        // Step 2: Process payment
        dispatch(new ProcessPayment($event->orderId, $this->getOrderTotal($event->orderId)));
    }

    public function handleInventoryReservationFailed(InventoryReservationFailed $event): void
    {
        // Compensate: cancel order
        $this->cancelOrder($event->orderId, 'Inventory not available');
    }

    public function handlePaymentProcessed(PaymentProcessed $event): void
    {
        $this->updateState($event->orderId->value(), 'payment_processed');

        // Step 3: Create shipment
        dispatch(new CreateShipment($event->orderId));
    }

    public function handlePaymentFailed(PaymentFailed $event): void
    {
        // Compensate: release inventory
        dispatch(new ReleaseInventory($event->orderId));

        // Cancel order
        $this->cancelOrder($event->orderId, 'Payment failed: ' . $event->reason);
    }

    private function cancelOrder(object $orderId, string $reason): void
    {
        // Dispatch cancel command
        \App\Domain\Order\Commands\CancelOrder::dispatch($orderId, $reason);

        $this->updateState($orderId->value(), 'cancelled', ['reason' => $reason]);
    }

    private function updateState(string $orderId, string $step, array $data = []): void
    {
        $this->state[$orderId]['steps'][] = array_merge(['step' => $step], $data);
        $this->state[$orderId]['status']  = $step;
    }

    private function getOrderTotal(object $orderId): \App\Domain\Order\ValueObjects\Money
    {
        // Retrieve from storage
        return \App\Models\Order::findByOrderId($orderId)->total;
    }
}
```

---

## ขั้นตอนที่ 1636: Choreography-based Saga

```php
<?php

declare(strict_types=1);

// Choreography - แต่ละ service react ต่อ event ของตัวเอง ไม่มี central orchestrator

// app/Domain/Inventory/Listeners/ReserveInventoryOnOrderPlaced.php
namespace App\Domain\Inventory\Listeners;

use App\Domain\Order\Events\OrderPlaced;
use App\Domain\Inventory\Events\InventoryReserved;
use App\Domain\Inventory\Events\InventoryReservationFailed;
use App\Services\InventoryService;

class ReserveInventoryOnOrderPlaced
{
    public function __construct(private InventoryService $inventoryService) {}

    public function handle(OrderPlaced $event): void
    {
        $canReserve = $this->inventoryService->canReserve($event->items);

        if ($canReserve) {
            $this->inventoryService->reserve($event->orderId, $event->items);
            event(new InventoryReserved($event->orderId, $event->items));
        } else {
            event(new InventoryReservationFailed($event->orderId, 'Insufficient stock'));
        }
    }
}

// app/Domain/Payment/Listeners/ProcessPaymentOnInventoryReserved.php
namespace App\Domain\Payment\Listeners;

use App\Domain\Inventory\Events\InventoryReserved;
use App\Domain\Payment\Events\PaymentProcessed;
use App\Domain\Payment\Events\PaymentFailed;
use App\Services\PaymentService;
use App\Models\Order;

class ProcessPaymentOnInventoryReserved
{
    public function __construct(private PaymentService $paymentService) {}

    public function handle(InventoryReserved $event): void
    {
        $order = Order::findByOrderId($event->orderId);

        try {
            $charge = $this->paymentService->charge($order->customer, $order->total);
            event(new PaymentProcessed($event->orderId, $charge->id));
        } catch (\Exception $e) {
            event(new PaymentFailed($event->orderId, $e->getMessage()));
        }
    }
}

// app/Domain/Shipping/Listeners/CreateShipmentOnPaymentProcessed.php
namespace App\Domain\Shipping\Listeners;

use App\Domain\Payment\Events\PaymentProcessed;
use App\Domain\Shipping\Events\ShipmentCreated;

class CreateShipmentOnPaymentProcessed
{
    public function handle(PaymentProcessed $event): void
    {
        // Create shipment record
        $shipment = \App\Models\Shipment::create([
            'order_id' => $event->orderId->value(),
            'status'   => 'pending',
        ]);

        event(new ShipmentCreated($event->orderId, $shipment->id));
    }
}
```

---

## ขั้นตอนที่ 1637: Eventual Consistency และ Read Models

```php
<?php

declare(strict_types=1);

// app/Domain/Order/Projections/OrderSummaryProjection.php
namespace App\Domain\Order\Projections;

use App\Domain\Order\Events\OrderPlaced;
use App\Domain\Order\Events\OrderCancelled;
use App\Domain\Order\Events\OrderShipped;
use App\Models\OrderSummary;

/**
 * Read model ที่ถูก update โดย domain events
 * Eventual consistency: อาจ delay เล็กน้อย แต่ query เร็วมาก
 */
class OrderSummaryProjection
{
    public function onOrderPlaced(OrderPlaced $event): void
    {
        OrderSummary::create([
            'order_id'    => $event->orderId->value(),
            'customer_id' => $event->customerId,
            'total'       => $event->total->amount(),
            'currency'    => $event->total->currency(),
            'status'      => 'pending',
            'item_count'  => count($event->items),
            'placed_at'   => $event->occurredAt,
        ]);
    }

    public function onOrderCancelled(OrderCancelled $event): void
    {
        OrderSummary::where('order_id', $event->orderId->value())
            ->update([
                'status'       => 'cancelled',
                'cancelled_at' => $event->occurredAt,
            ]);
    }

    public function onOrderShipped(OrderShipped $event): void
    {
        OrderSummary::where('order_id', $event->orderId->value())
            ->update([
                'status'          => 'shipped',
                'tracking_number' => $event->trackingNumber,
                'shipped_at'      => $event->occurredAt,
            ]);
    }
}
```

---

## ขั้นตอนที่ 1638: Event Store

```php
<?php

declare(strict_types=1);

// app/Infrastructure/EventStore/EventStore.php
namespace App\Infrastructure\EventStore;

use App\Models\StoredEvent;
use Illuminate\Support\Collection;

class EventStore
{
    public function append(string $aggregateType, string $aggregateId, int $expectedVersion, array $events): void
    {
        $currentVersion = StoredEvent::where([
            'aggregate_type' => $aggregateType,
            'aggregate_id'   => $aggregateId,
        ])->max('version') ?? -1;

        if ($currentVersion !== $expectedVersion) {
            throw new \DomainException(
                "Optimistic concurrency conflict. Expected version {$expectedVersion}, got {$currentVersion}"
            );
        }

        foreach ($events as $i => $event) {
            StoredEvent::create([
                'aggregate_type' => $aggregateType,
                'aggregate_id'   => $aggregateId,
                'event_type'     => get_class($event),
                'payload'        => json_encode($event),
                'version'        => $expectedVersion + $i + 1,
                'occurred_at'    => now(),
            ]);
        }
    }

    public function getEvents(string $aggregateType, string $aggregateId, int $fromVersion = 0): Collection
    {
        return StoredEvent::where([
            'aggregate_type' => $aggregateType,
            'aggregate_id'   => $aggregateId,
        ])
            ->where('version', '>', $fromVersion)
            ->orderBy('version')
            ->get()
            ->map(fn ($stored) => $this->deserialize($stored));
    }

    private function deserialize(StoredEvent $stored): object
    {
        $class   = $stored->event_type;
        $payload = json_decode($stored->payload, true);
        return new $class(...$payload);
    }
}
```

---

## สรุป Part 59

| Concept | รายละเอียด |
|---------|-----------|
| Domain Events | Events ที่แทน business fact |
| Event Bus | Infrastructure สำหรับ publish/subscribe |
| Outbox Pattern | ป้องกัน lost events ด้วย DB transaction |
| Saga Orchestration | Central coordinator ควบคุม workflow |
| Saga Choreography | แต่ละ service react ต่อ events ตัวเอง |
| Eventual Consistency | Read models update แบบ async |
| Event Store | บันทึก events เป็น source of truth |

ถัดไป → Part 60: Laravel Cashier กับ Stripe
