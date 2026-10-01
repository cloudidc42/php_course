# Part 43: Design Patterns

## ขั้นตอนที่ 1151-1180: Design Patterns ใน PHP

บทนี้ครอบคลุม Repository Pattern, Service Layer, CQRS, Event Sourcing, DDD และ Value Objects

---

## ขั้นตอนที่ 1151: Repository Pattern กับ Eloquent

Repository Pattern แยก business logic ออกจาก data access layer

```php
<?php

declare(strict_types=1);

namespace App\Contracts\Repositories;

use App\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

interface UserRepositoryInterface
{
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    public function create(array $data): User;
    public function update(int $id, array $data): User;
    public function delete(int $id): bool;
    public function paginate(int $perPage = 15, array $filters = []): LengthAwarePaginator;
    public function findActive(): \Illuminate\Database\Eloquent\Collection;
}
```

```php
<?php

declare(strict_types=1);

namespace App\Repositories;

use App\Contracts\Repositories\UserRepositoryInterface;
use App\Models\User;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;
use Illuminate\Database\Eloquent\Collection;

class EloquentUserRepository implements UserRepositoryInterface
{
    public function __construct(private User $model) {}

    public function findById(int $id): ?User
    {
        return $this->model->find($id);
    }

    public function findByEmail(string $email): ?User
    {
        return $this->model->where('email', $email)->first();
    }

    public function create(array $data): User
    {
        return $this->model->create($data);
    }

    public function update(int $id, array $data): User
    {
        $user = $this->findById($id);

        if (!$user) {
            throw new \App\Exceptions\UserNotFoundException("User {$id} not found");
        }

        $user->update($data);
        return $user->refresh();
    }

    public function delete(int $id): bool
    {
        $user = $this->findById($id);

        if (!$user) {
            throw new \App\Exceptions\UserNotFoundException("User {$id} not found");
        }

        return $user->delete();
    }

    public function paginate(int $perPage = 15, array $filters = []): LengthAwarePaginator
    {
        return $this->model->newQuery()
            ->when(isset($filters['role']), fn ($q) => $q->whereRole($filters['role']))
            ->when(isset($filters['search']), fn ($q) => $q->search($filters['search']))
            ->when(isset($filters['active']), fn ($q) => $q->where('is_active', $filters['active']))
            ->latest()
            ->paginate($perPage);
    }

    public function findActive(): Collection
    {
        return $this->model->where('is_active', true)->get();
    }
}
```

```php
<?php

declare(strict_types=1);

// Cached Repository Decorator
namespace App\Repositories;

use App\Contracts\Repositories\UserRepositoryInterface;
use App\Models\User;
use Illuminate\Contracts\Cache\Repository as Cache;

class CachedUserRepository implements UserRepositoryInterface
{
    private const CACHE_TTL = 3600; // 1 hour

    public function __construct(
        private UserRepositoryInterface $repository,
        private Cache $cache
    ) {}

    public function findById(int $id): ?User
    {
        return $this->cache->remember(
            "user:{$id}",
            self::CACHE_TTL,
            fn () => $this->repository->findById($id)
        );
    }

    public function create(array $data): User
    {
        $user = $this->repository->create($data);
        $this->cache->put("user:{$user->id}", $user, self::CACHE_TTL);
        return $user;
    }

    public function update(int $id, array $data): User
    {
        $user = $this->repository->update($id, $data);
        $this->cache->put("user:{$id}", $user, self::CACHE_TTL);
        $this->cache->forget("users:list");
        return $user;
    }

    public function delete(int $id): bool
    {
        $result = $this->repository->delete($id);
        $this->cache->forget("user:{$id}");
        $this->cache->forget("users:list");
        return $result;
    }

    public function findByEmail(string $email): ?User
    {
        return $this->repository->findByEmail($email);
    }

    public function paginate(int $perPage = 15, array $filters = []): \Illuminate\Contracts\Pagination\LengthAwarePaginator
    {
        return $this->repository->paginate($perPage, $filters);
    }

    public function findActive(): \Illuminate\Database\Eloquent\Collection
    {
        return $this->cache->remember(
            'users:active',
            self::CACHE_TTL,
            fn () => $this->repository->findActive()
        );
    }
}
```

---

## ขั้นตอนที่ 1152: Service Layer Pattern

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Contracts\Repositories\UserRepositoryInterface;
use App\Events\UserRegistered;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;

class UserService
{
    public function __construct(
        private UserRepositoryInterface $userRepository,
        private \App\Services\ProfileService $profileService,
        private \App\Services\RoleService $roleService
    ) {}

    public function register(array $data): User
    {
        return DB::transaction(function () use ($data) {
            // Hash password
            $data['password'] = Hash::make($data['password']);

            // Create user
            $user = $this->userRepository->create($data);

            // Assign default role
            $this->roleService->assignDefaultRole($user);

            // Create profile
            $this->profileService->createDefault($user);

            // Fire event
            event(new UserRegistered($user));

            return $user;
        });
    }

    public function updateProfile(int $userId, array $data): User
    {
        return DB::transaction(function () use ($userId, $data) {
            $user = $this->userRepository->update($userId, $data);

            if (isset($data['avatar'])) {
                $this->profileService->updateAvatar($user, $data['avatar']);
            }

            return $user;
        });
    }

    public function deactivate(int $userId): void
    {
        DB::transaction(function () use ($userId) {
            $this->userRepository->update($userId, ['is_active' => false]);

            // Cancel active subscriptions
            $user = $this->userRepository->findById($userId);
            $user->subscriptions()->active()->each->cancel();

            // Revoke tokens
            $user->tokens()->delete();
        });
    }
}
```

---

## ขั้นตอนที่ 1153: CQRS Pattern

```php
<?php

declare(strict_types=1);

// COMMAND
namespace App\Commands;

final class CreateOrderCommand
{
    public function __construct(
        public readonly int $userId,
        public readonly array $items,
        public readonly string $shippingAddress,
        public readonly string $paymentMethod
    ) {}
}

// QUERY
namespace App\Queries;

final class GetUserOrdersQuery
{
    public function __construct(
        public readonly int $userId,
        public readonly string $status = 'all',
        public readonly int $perPage = 15
    ) {}
}
```

```php
<?php

declare(strict_types=1);

namespace App\Handlers\Commands;

use App\Commands\CreateOrderCommand;
use App\Models\Order;
use App\Events\OrderPlaced;
use Illuminate\Support\Facades\DB;

class CreateOrderHandler
{
    public function handle(CreateOrderCommand $command): Order
    {
        return DB::transaction(function () use ($command) {
            $order = Order::create([
                'user_id'          => $command->userId,
                'shipping_address' => $command->shippingAddress,
                'payment_method'   => $command->paymentMethod,
                'status'           => 'pending',
            ]);

            foreach ($command->items as $item) {
                $order->items()->create([
                    'product_id' => $item['product_id'],
                    'quantity'   => $item['quantity'],
                    'price'      => $item['price'],
                ]);
            }

            $order->calculateTotal();
            $order->save();

            event(new OrderPlaced($order));

            return $order;
        });
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Handlers\Queries;

use App\Queries\GetUserOrdersQuery;
use App\Models\Order;
use Illuminate\Contracts\Pagination\LengthAwarePaginator;

class GetUserOrdersHandler
{
    public function handle(GetUserOrdersQuery $query): LengthAwarePaginator
    {
        return Order::with(['items.product', 'payments'])
            ->where('user_id', $query->userId)
            ->when(
                $query->status !== 'all',
                fn ($q) => $q->where('status', $query->status)
            )
            ->latest()
            ->paginate($query->perPage);
    }
}
```

```php
<?php

declare(strict_types=1);

// Command Bus
namespace App\Bus;

use Illuminate\Container\Container;

class CommandBus
{
    private array $handlers = [];

    public function __construct(private Container $container) {}

    public function register(string $command, string $handler): void
    {
        $this->handlers[$command] = $handler;
    }

    public function dispatch(object $command): mixed
    {
        $commandClass = get_class($command);

        if (!isset($this->handlers[$commandClass])) {
            throw new \RuntimeException("No handler for command: {$commandClass}");
        }

        $handler = $this->container->make($this->handlers[$commandClass]);
        return $handler->handle($command);
    }
}
```

---

## ขั้นตอนที่ 1154: Event Sourcing Basics

```php
<?php

declare(strict_types=1);

namespace App\EventSourcing;

// Domain Event
abstract class DomainEvent
{
    public readonly string $eventId;
    public readonly \DateTimeImmutable $occurredAt;

    public function __construct()
    {
        $this->eventId    = bin2hex(random_bytes(16));
        $this->occurredAt = new \DateTimeImmutable();
    }

    abstract public function eventName(): string;
    abstract public function payload(): array;
}
```

```php
<?php

declare(strict_types=1);

namespace App\EventSourcing\Events;

use App\EventSourcing\DomainEvent;

class BankAccountOpened extends DomainEvent
{
    public function __construct(
        public readonly string $accountId,
        public readonly string $ownerName,
        public readonly float $initialBalance
    ) {
        parent::__construct();
    }

    public function eventName(): string
    {
        return 'bank_account.opened';
    }

    public function payload(): array
    {
        return [
            'account_id'      => $this->accountId,
            'owner_name'      => $this->ownerName,
            'initial_balance' => $this->initialBalance,
        ];
    }
}

class MoneyDeposited extends DomainEvent
{
    public function __construct(
        public readonly string $accountId,
        public readonly float $amount,
        public readonly string $description = ''
    ) {
        parent::__construct();
    }

    public function eventName(): string
    {
        return 'bank_account.money_deposited';
    }

    public function payload(): array
    {
        return [
            'account_id'  => $this->accountId,
            'amount'      => $this->amount,
            'description' => $this->description,
        ];
    }
}

class MoneyWithdrawn extends DomainEvent
{
    public function __construct(
        public readonly string $accountId,
        public readonly float $amount,
        public readonly string $description = ''
    ) {
        parent::__construct();
    }

    public function eventName(): string
    {
        return 'bank_account.money_withdrawn';
    }

    public function payload(): array
    {
        return [
            'account_id'  => $this->accountId,
            'amount'      => $this->amount,
            'description' => $this->description,
        ];
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\EventSourcing;

// Aggregate Root
class BankAccount
{
    private float $balance     = 0.0;
    private bool $isOpen       = false;
    private array $events      = [];
    private string $accountId;

    public static function open(string $ownerName, float $initialBalance): self
    {
        $account = new self();
        $account->apply(new Events\BankAccountOpened(
            accountId: bin2hex(random_bytes(8)),
            ownerName: $ownerName,
            initialBalance: $initialBalance
        ));

        return $account;
    }

    public function deposit(float $amount, string $description = ''): void
    {
        if (!$this->isOpen) {
            throw new \DomainException('Account is not open');
        }

        if ($amount <= 0) {
            throw new \InvalidArgumentException('Deposit amount must be positive');
        }

        $this->apply(new Events\MoneyDeposited($this->accountId, $amount, $description));
    }

    public function withdraw(float $amount, string $description = ''): void
    {
        if (!$this->isOpen) {
            throw new \DomainException('Account is not open');
        }

        if ($amount > $this->balance) {
            throw new \DomainException('Insufficient funds');
        }

        $this->apply(new Events\MoneyWithdrawn($this->accountId, $amount, $description));
    }

    private function apply(DomainEvent $event): void
    {
        $this->handle($event);
        $this->events[] = $event;
    }

    private function handle(DomainEvent $event): void
    {
        match (get_class($event)) {
            Events\BankAccountOpened::class => $this->handleOpened($event),
            Events\MoneyDeposited::class    => $this->handleDeposited($event),
            Events\MoneyWithdrawn::class    => $this->handleWithdrawn($event),
        };
    }

    private function handleOpened(Events\BankAccountOpened $event): void
    {
        $this->accountId = $event->accountId;
        $this->balance   = $event->initialBalance;
        $this->isOpen    = true;
    }

    private function handleDeposited(Events\MoneyDeposited $event): void
    {
        $this->balance += $event->amount;
    }

    private function handleWithdrawn(Events\MoneyWithdrawn $event): void
    {
        $this->balance -= $event->amount;
    }

    public function getBalance(): float
    {
        return $this->balance;
    }

    public function getRecordedEvents(): array
    {
        return $this->events;
    }

    public function clearEvents(): void
    {
        $this->events = [];
    }

    // Reconstruct from events
    public static function reconstruct(array $events): self
    {
        $account = new self();

        foreach ($events as $event) {
            $account->handle($event);
        }

        return $account;
    }
}
```

---

## ขั้นตอนที่ 1155: Domain Driven Design - Value Objects

```php
<?php

declare(strict_types=1);

namespace App\Domain\ValueObjects;

// Value Object - immutable, no identity
final class Money
{
    public function __construct(
        private readonly int $amount,     // stored in cents
        private readonly string $currency
    ) {
        if ($amount < 0) {
            throw new \InvalidArgumentException('Money amount cannot be negative');
        }

        if (strlen($currency) !== 3) {
            throw new \InvalidArgumentException('Currency must be ISO 4217 code (3 chars)');
        }
    }

    public static function of(float $amount, string $currency): self
    {
        return new self((int)round($amount * 100), strtoupper($currency));
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function subtract(self $other): self
    {
        $this->assertSameCurrency($other);

        if ($this->amount < $other->amount) {
            throw new \DomainException('Cannot subtract: result would be negative');
        }

        return new self($this->amount - $other->amount, $this->currency);
    }

    public function multiply(float $factor): self
    {
        return new self((int)round($this->amount * $factor), $this->currency);
    }

    public function isGreaterThan(self $other): bool
    {
        $this->assertSameCurrency($other);
        return $this->amount > $other->amount;
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount && $this->currency === $other->currency;
    }

    public function getAmount(): float
    {
        return $this->amount / 100;
    }

    public function getCurrency(): string
    {
        return $this->currency;
    }

    public function format(): string
    {
        return number_format($this->getAmount(), 2) . ' ' . $this->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException(
                "Cannot operate on different currencies: {$this->currency} and {$other->currency}"
            );
        }
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Domain\ValueObjects;

final class Email
{
    private readonly string $value;

    public function __construct(string $email)
    {
        $normalized = strtolower(trim($email));

        if (!filter_var($normalized, FILTER_VALIDATE_EMAIL)) {
            throw new \InvalidArgumentException("Invalid email address: {$email}");
        }

        $this->value = $normalized;
    }

    public function value(): string
    {
        return $this->value;
    }

    public function domain(): string
    {
        return substr($this->value, strpos($this->value, '@') + 1);
    }

    public function localPart(): string
    {
        return substr($this->value, 0, strpos($this->value, '@'));
    }

    public function equals(self $other): bool
    {
        return $this->value === $other->value;
    }

    public function __toString(): string
    {
        return $this->value;
    }
}
```

---

## ขั้นตอนที่ 1156: Entities และ Aggregates

```php
<?php

declare(strict_types=1);

namespace App\Domain\Entities;

use App\Domain\ValueObjects\Money;
use App\Domain\ValueObjects\Email;

// Entity - has identity
class Product
{
    private string $id;

    public function __construct(
        string $id,
        private string $name,
        private Money $price,
        private int $stockQuantity,
        private bool $isAvailable = true
    ) {
        if (empty(trim($name))) {
            throw new \InvalidArgumentException('Product name cannot be empty');
        }

        $this->id = $id;
    }

    public function getId(): string
    {
        return $this->id;
    }

    public function updatePrice(Money $newPrice): void
    {
        $this->price = $newPrice;
    }

    public function restock(int $quantity): void
    {
        if ($quantity <= 0) {
            throw new \InvalidArgumentException('Restock quantity must be positive');
        }

        $this->stockQuantity += $quantity;
        $this->isAvailable = true;
    }

    public function decreaseStock(int $quantity): void
    {
        if ($quantity > $this->stockQuantity) {
            throw new \DomainException("Insufficient stock for product {$this->id}");
        }

        $this->stockQuantity -= $quantity;

        if ($this->stockQuantity === 0) {
            $this->isAvailable = false;
        }
    }

    public function isAvailable(): bool
    {
        return $this->isAvailable && $this->stockQuantity > 0;
    }

    public function getPrice(): Money
    {
        return $this->price;
    }

    public function getStockQuantity(): int
    {
        return $this->stockQuantity;
    }
}
```

---

## สรุปบทที่ 43

| Pattern | เมื่อใช้ | ประโยชน์ |
|---------|---------|---------|
| Repository | Data access abstraction | Swap storage easily |
| Service Layer | Business logic | Reusable, testable |
| CQRS | Complex read/write | Scalability, clarity |
| Event Sourcing | Audit trail needed | Full history |
| Value Objects | Immutable concepts | No identity bugs |
| Entities | Domain objects | Clear identity |

**ต่อไป**: Part 44 - Laravel Livewire

---
