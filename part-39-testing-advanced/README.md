# Part 39: Advanced Testing

## ขั้นตอนที่ 1031-1060: การทดสอบขั้นสูงด้วย PHPUnit, Pest และ TDD

การทดสอบซอฟต์แวร์ขั้นสูงเป็นหัวใจสำคัญของการพัฒนาที่มีคุณภาพ บทนี้จะครอบคลุม Data Providers, Mock Objects, Pest PHP Framework, TDD และ Mutation Testing

---

## ขั้นตอนที่ 1031: PHPUnit Data Providers

Data Providers ช่วยให้เราทดสอบฟังก์ชันเดียวกันกับ input หลายชุดโดยไม่ต้องเขียน test ซ้ำ

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use App\Services\Calculator;
use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\DataProvider;

class CalculatorTest extends TestCase
{
    private Calculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new Calculator();
    }

    #[DataProvider('additionProvider')]
    public function test_addition(int $a, int $b, int $expected): void
    {
        $result = $this->calculator->add($a, $b);
        $this->assertSame($expected, $result);
    }

    public static function additionProvider(): array
    {
        return [
            'positive numbers'  => [2, 3, 5],
            'negative numbers'  => [-1, -1, -2],
            'mixed signs'       => [-5, 10, 5],
            'zeros'             => [0, 0, 0],
            'large numbers'     => [PHP_INT_MAX - 1, 1, PHP_INT_MAX],
        ];
    }

    #[DataProvider('divisionProvider')]
    public function test_division(float $a, float $b, float $expected): void
    {
        $result = $this->calculator->divide($a, $b);
        $this->assertEqualsWithDelta($expected, $result, 0.0001);
    }

    public static function divisionProvider(): array
    {
        return [
            'whole division'    => [10.0, 2.0, 5.0],
            'decimal result'    => [10.0, 3.0, 3.3333],
            'divide by one'     => [5.0, 1.0, 5.0],
        ];
    }

    public function test_division_by_zero_throws_exception(): void
    {
        $this->expectException(\DivisionByZeroError::class);
        $this->expectExceptionMessage('Cannot divide by zero');

        $this->calculator->divide(10.0, 0.0);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Services;

class Calculator
{
    public function add(int $a, int $b): int
    {
        return $a + $b;
    }

    public function divide(float $a, float $b): float
    {
        if ($b === 0.0) {
            throw new \DivisionByZeroError('Cannot divide by zero');
        }

        return $a / $b;
    }
}
```

---

## ขั้นตอนที่ 1032: Mock Objects และ Test Doubles

Mock Objects ช่วยให้เราทดสอบ class โดยไม่ต้องพึ่งพา dependencies จริง

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use App\Repositories\UserRepository;
use App\Services\UserService;
use App\Models\User;
use App\Mail\WelcomeEmail;
use Illuminate\Mail\Mailer;
use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\MockObject\MockObject;

class UserServiceTest extends TestCase
{
    private UserService $userService;
    private UserRepository&MockObject $userRepository;
    private Mailer&MockObject $mailer;

    protected function setUp(): void
    {
        $this->userRepository = $this->createMock(UserRepository::class);
        $this->mailer = $this->createMock(Mailer::class);

        $this->userService = new UserService(
            $this->userRepository,
            $this->mailer
        );
    }

    public function test_create_user_saves_to_repository(): void
    {
        $userData = [
            'name'  => 'John Doe',
            'email' => 'john@example.com',
        ];

        $expectedUser = new User($userData);

        $this->userRepository
            ->expects($this->once())
            ->method('create')
            ->with($this->equalTo($userData))
            ->willReturn($expectedUser);

        $this->mailer
            ->expects($this->once())
            ->method('send')
            ->with($this->isInstanceOf(WelcomeEmail::class));

        $result = $this->userService->createUser($userData);

        $this->assertSame($expectedUser, $result);
    }

    public function test_find_user_returns_null_when_not_found(): void
    {
        $this->userRepository
            ->expects($this->once())
            ->method('findById')
            ->with(999)
            ->willReturn(null);

        $result = $this->userService->findUser(999);

        $this->assertNull($result);
    }

    public function test_update_user_throws_exception_for_nonexistent_user(): void
    {
        $this->userRepository
            ->method('findById')
            ->willReturn(null);

        $this->expectException(\App\Exceptions\UserNotFoundException::class);

        $this->userService->updateUser(999, ['name' => 'New Name']);
    }
}
```

---

## ขั้นตอนที่ 1033: Stub, Spy และ Fake

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use App\Services\PaymentService;
use App\Gateways\PaymentGatewayInterface;
use PHPUnit\Framework\TestCase;

class PaymentServiceTest extends TestCase
{
    // Stub - กำหนดค่าที่จะ return
    public function test_process_payment_with_stub(): void
    {
        $gateway = $this->createStub(PaymentGatewayInterface::class);
        $gateway->method('charge')->willReturn([
            'success'        => true,
            'transaction_id' => 'txn_123',
        ]);

        $service = new PaymentService($gateway);
        $result = $service->processPayment(100.00, 'card_token');

        $this->assertTrue($result['success']);
        $this->assertEquals('txn_123', $result['transaction_id']);
    }

    // Spy - ตรวจสอบว่า method ถูกเรียกกี่ครั้ง
    public function test_failed_payment_retries_three_times(): void
    {
        $gateway = $this->createMock(PaymentGatewayInterface::class);
        $gateway->expects($this->exactly(3))
            ->method('charge')
            ->willThrowException(new \RuntimeException('Network error'));

        $service = new PaymentService($gateway, maxRetries: 3);

        $this->expectException(\App\Exceptions\PaymentFailedException::class);
        $service->processPayment(100.00, 'card_token');
    }

    // Callback-based stub
    public function test_different_results_on_multiple_calls(): void
    {
        $gateway = $this->createMock(PaymentGatewayInterface::class);
        $gateway->method('charge')
            ->willReturnOnConsecutiveCalls(
                ['success' => false, 'error' => 'Insufficient funds'],
                ['success' => false, 'error' => 'Card declined'],
                ['success' => true, 'transaction_id' => 'txn_456'],
            );

        $service = new PaymentService($gateway, maxRetries: 3);
        $result = $service->processPayment(100.00, 'card_token');

        $this->assertTrue($result['success']);
    }
}
```

---

## ขั้นตอนที่ 1034: Pest PHP Testing Framework

Pest เป็น testing framework ที่ทันสมัย มี syntax ที่สวยงามกว่า PHPUnit

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Services\AuthService;

// การใช้ describe/it syntax
describe('AuthService', function () {
    beforeEach(function () {
        $this->authService = app(AuthService::class);
    });

    it('authenticates valid credentials', function () {
        $user = User::factory()->create([
            'email'    => 'test@example.com',
            'password' => bcrypt('password123'),
        ]);

        $result = $this->authService->authenticate(
            'test@example.com',
            'password123'
        );

        expect($result)->toBeInstanceOf(User::class);
        expect($result->id)->toBe($user->id);
    });

    it('rejects invalid password', function () {
        User::factory()->create([
            'email'    => 'test@example.com',
            'password' => bcrypt('correct_password'),
        ]);

        expect(fn () => $this->authService->authenticate(
            'test@example.com',
            'wrong_password'
        ))->toThrow(\App\Exceptions\InvalidCredentialsException::class);
    });

    it('locks account after 5 failed attempts', function () {
        $user = User::factory()->create([
            'email'    => 'test@example.com',
            'password' => bcrypt('password'),
        ]);

        // พยายาม login ผิด 5 ครั้ง
        for ($i = 0; $i < 5; $i++) {
            try {
                $this->authService->authenticate('test@example.com', 'wrong');
            } catch (\Exception $e) {
                // ข้าม exception
            }
        }

        expect($user->fresh()->is_locked)->toBeTrue();
    });
});
```

```php
<?php

declare(strict_types=1);

// Higher-order tests with Pest
test('user can be created')
    ->expect(fn () => User::factory()->create())
    ->toBeInstanceOf(User::class);

// Dataset (เทียบเท่า Data Providers)
it('validates email format', function (string $email, bool $isValid) {
    $validator = validator(['email' => $email], ['email' => 'email']);

    expect($validator->passes())->toBe($isValid);
})->with([
    ['valid@email.com', true],
    ['invalid-email', false],
    ['missing@domain', false],
    ['@nodomain.com', false],
    ['user+tag@example.co.th', true],
]);
```

---

## ขั้นตอนที่ 1035: Feature Tests และ Integration Tests

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class UserRegistrationTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_register_with_valid_data(): void
    {
        $response = $this->postJson('/api/register', [
            'name'                  => 'John Doe',
            'email'                 => 'john@example.com',
            'password'              => 'SecurePass123!',
            'password_confirmation' => 'SecurePass123!',
        ]);

        $response->assertStatus(201)
            ->assertJsonStructure([
                'data' => [
                    'id',
                    'name',
                    'email',
                    'created_at',
                ],
                'token',
            ]);

        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com',
            'name'  => 'John Doe',
        ]);
    }

    public function test_registration_fails_with_duplicate_email(): void
    {
        User::factory()->create(['email' => 'existing@example.com']);

        $response = $this->postJson('/api/register', [
            'name'                  => 'Jane Doe',
            'email'                 => 'existing@example.com',
            'password'              => 'SecurePass123!',
            'password_confirmation' => 'SecurePass123!',
        ]);

        $response->assertStatus(422)
            ->assertJsonValidationErrors(['email']);
    }

    public function test_authenticated_user_can_get_profile(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)
            ->getJson('/api/profile');

        $response->assertStatus(200)
            ->assertJson([
                'data' => [
                    'id'    => $user->id,
                    'email' => $user->email,
                ],
            ]);
    }
}
```

---

## ขั้นตอนที่ 1036: Test-Driven Development (TDD)

TDD = เขียน test ก่อน → เขียน code ให้ผ่าน → Refactor

```php
<?php

declare(strict_types=1);

// ขั้นที่ 1: เขียน test ก่อน (RED)
namespace Tests\Unit;

use App\Services\ShoppingCart;
use App\Models\Product;
use PHPUnit\Framework\TestCase;

class ShoppingCartTest extends TestCase
{
    private ShoppingCart $cart;

    protected function setUp(): void
    {
        $this->cart = new ShoppingCart();
    }

    /** @test */
    public function empty_cart_has_zero_total(): void
    {
        $this->assertEquals(0.0, $this->cart->total());
        $this->assertCount(0, $this->cart->items());
    }

    /** @test */
    public function can_add_item_to_cart(): void
    {
        $product = new Product('PHP Book', 29.99);
        $this->cart->add($product, 1);

        $this->assertCount(1, $this->cart->items());
        $this->assertEqualsWithDelta(29.99, $this->cart->total(), 0.001);
    }

    /** @test */
    public function adding_same_product_increases_quantity(): void
    {
        $product = new Product('PHP Book', 29.99);
        $this->cart->add($product, 1);
        $this->cart->add($product, 2);

        $this->assertCount(1, $this->cart->items());
        $this->assertEqualsWithDelta(89.97, $this->cart->total(), 0.001);
    }

    /** @test */
    public function can_apply_percentage_discount(): void
    {
        $product = new Product('PHP Book', 100.00);
        $this->cart->add($product, 1);
        $this->cart->applyDiscount(20); // 20% discount

        $this->assertEqualsWithDelta(80.00, $this->cart->total(), 0.001);
    }

    /** @test */
    public function can_remove_item(): void
    {
        $product = new Product('PHP Book', 29.99);
        $this->cart->add($product, 2);
        $this->cart->remove($product->id());

        $this->assertCount(0, $this->cart->items());
        $this->assertEquals(0.0, $this->cart->total());
    }
}
```

```php
<?php

declare(strict_types=1);

// ขั้นที่ 2: Implementation (GREEN)
namespace App\Services;

use App\Models\Product;

class ShoppingCart
{
    private array $items = [];
    private float $discountPercentage = 0.0;

    public function add(Product $product, int $quantity): void
    {
        $productId = $product->id();

        if (isset($this->items[$productId])) {
            $this->items[$productId]['quantity'] += $quantity;
        } else {
            $this->items[$productId] = [
                'product'  => $product,
                'quantity' => $quantity,
            ];
        }
    }

    public function remove(string $productId): void
    {
        unset($this->items[$productId]);
    }

    public function items(): array
    {
        return $this->items;
    }

    public function applyDiscount(float $percentage): void
    {
        $this->discountPercentage = $percentage;
    }

    public function total(): float
    {
        $subtotal = array_reduce(
            $this->items,
            fn (float $carry, array $item) =>
                $carry + ($item['product']->price() * $item['quantity']),
            0.0
        );

        if ($this->discountPercentage > 0) {
            $discount = $subtotal * ($this->discountPercentage / 100);
            return $subtotal - $discount;
        }

        return $subtotal;
    }
}
```

---

## ขั้นตอนที่ 1037: Mutation Testing ด้วย Infection PHP

Mutation Testing ตรวจสอบว่า test ของเรา "แข็งแกร่ง" พอหรือไม่

```bash
# ติดตั้ง Infection
composer require --dev infection/infection

# รัน mutation testing
./vendor/bin/infection --threads=4 --min-msi=80 --min-covered-msi=90
```

```php
<?php

declare(strict_types=1);

namespace App\Services;

// Code ที่จะถูก mutate
class PriceCalculator
{
    public function calculateDiscount(float $price, float $percentage): float
    {
        if ($percentage < 0 || $percentage > 100) {
            throw new \InvalidArgumentException('Percentage must be between 0 and 100');
        }

        return $price * (1 - $percentage / 100);
    }

    public function applyTax(float $price, float $taxRate): float
    {
        return $price * (1 + $taxRate / 100);
    }

    public function calculateBulkDiscount(int $quantity, float $unitPrice): float
    {
        if ($quantity >= 100) {
            return $unitPrice * 0.70; // 30% discount
        }

        if ($quantity >= 50) {
            return $unitPrice * 0.80; // 20% discount
        }

        if ($quantity >= 10) {
            return $unitPrice * 0.90; // 10% discount
        }

        return $unitPrice;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace Tests\Unit;

use App\Services\PriceCalculator;
use PHPUnit\Framework\TestCase;
use PHPUnit\Framework\Attributes\DataProvider;

// Test ที่ครอบคลุม mutations ทุกประเภท
class PriceCalculatorTest extends TestCase
{
    private PriceCalculator $calculator;

    protected function setUp(): void
    {
        $this->calculator = new PriceCalculator();
    }

    #[DataProvider('discountProvider')]
    public function test_calculate_discount(
        float $price,
        float $percentage,
        float $expected
    ): void {
        $result = $this->calculator->calculateDiscount($price, $percentage);
        $this->assertEqualsWithDelta($expected, $result, 0.001);
    }

    public static function discountProvider(): array
    {
        return [
            'no discount'    => [100.00, 0.0, 100.00],
            '10% discount'   => [100.00, 10.0, 90.00],
            '50% discount'   => [100.00, 50.0, 50.00],
            '100% discount'  => [100.00, 100.0, 0.00],
        ];
    }

    public function test_negative_percentage_throws(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->calculator->calculateDiscount(100.00, -1.0);
    }

    public function test_over_100_percentage_throws(): void
    {
        $this->expectException(\InvalidArgumentException::class);
        $this->calculator->calculateDiscount(100.00, 101.0);
    }

    #[DataProvider('bulkDiscountProvider')]
    public function test_bulk_discount_boundaries(
        int $quantity,
        float $unitPrice,
        float $expected
    ): void {
        $result = $this->calculator->calculateBulkDiscount($quantity, $unitPrice);
        $this->assertEqualsWithDelta($expected, $result, 0.001);
    }

    public static function bulkDiscountProvider(): array
    {
        return [
            'below 10'     => [9, 100.00, 100.00],
            'exactly 10'   => [10, 100.00, 90.00],
            'below 50'     => [49, 100.00, 90.00],
            'exactly 50'   => [50, 100.00, 80.00],
            'below 100'    => [99, 100.00, 80.00],
            'exactly 100'  => [100, 100.00, 70.00],
            'above 100'    => [200, 100.00, 70.00],
        ];
    }
}
```

---

## ขั้นตอนที่ 1038: Contract Testing

Contract Testing ตรวจสอบว่า API client และ server ตกลงกันได้ถูกต้อง

```php
<?php

declare(strict_types=1);

namespace Tests\Contract;

use PHPUnit\Framework\TestCase;

// Consumer Contract Test
class UserApiContractTest extends TestCase
{
    private string $baseUrl = 'http://localhost:8080';

    public function test_get_user_contract(): void
    {
        $expectedSchema = [
            'type' => 'object',
            'required' => ['id', 'name', 'email', 'created_at'],
            'properties' => [
                'id'         => ['type' => 'integer'],
                'name'       => ['type' => 'string'],
                'email'      => ['type' => 'string', 'format' => 'email'],
                'created_at' => ['type' => 'string', 'format' => 'date-time'],
            ],
        ];

        // ทดสอบว่า response ตรงกับ contract
        $response = $this->makeApiCall('GET', '/api/users/1');

        $this->assertJsonMatchesSchema($response, $expectedSchema);
    }

    public function test_create_user_contract(): void
    {
        $requestBody = [
            'name'     => 'Test User',
            'email'    => 'test@contract.com',
            'password' => 'SecurePass123!',
        ];

        $response = $this->makeApiCall('POST', '/api/users', $requestBody);

        $this->assertEquals(201, $response['status']);
        $this->assertArrayHasKey('id', $response['data']);
        $this->assertEquals($requestBody['email'], $response['data']['email']);
    }

    private function makeApiCall(string $method, string $path, array $body = []): array
    {
        // Simplified HTTP client
        $ch = curl_init("{$this->baseUrl}{$path}");
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, ['Content-Type: application/json']);

        if ($method === 'POST') {
            curl_setopt($ch, CURLOPT_POST, true);
            curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
        }

        $responseBody = curl_exec($ch);
        $statusCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
        curl_close($ch);

        return [
            'status' => $statusCode,
            'data'   => json_decode($responseBody, true),
        ];
    }

    private function assertJsonMatchesSchema(array $data, array $schema): void
    {
        foreach ($schema['required'] ?? [] as $field) {
            $this->assertArrayHasKey($field, $data['data'], "Missing required field: {$field}");
        }
    }
}
```

---

## ขั้นตอนที่ 1039: Test Coverage และ Code Quality

```xml
<!-- phpunit.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
         stopOnFailure="false">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">./tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">./tests/Feature</directory>
        </testsuite>
        <testsuite name="Integration">
            <directory suffix="Test.php">./tests/Integration</directory>
        </testsuite>
    </testsuites>

    <coverage>
        <report>
            <html outputDirectory="coverage/html"/>
            <clover outputFile="coverage/clover.xml"/>
            <text/>
        </report>
        <include>
            <directory suffix=".php">./app</directory>
        </include>
        <exclude>
            <directory>./app/Http/Middleware</directory>
            <file>./app/Console/Kernel.php</file>
        </exclude>
    </coverage>

    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="SESSION_DRIVER" value="array"/>
        <env name="QUEUE_DRIVER" value="sync"/>
    </php>
</phpunit>
```

---

## ขั้นตอนที่ 1040: Pest Expectations API ขั้นสูง

```php
<?php

declare(strict_types=1);

use App\Models\User;
use App\Models\Post;

// Pest Expectations
it('has comprehensive expectations', function () {
    $user = User::factory()->create([
        'name'  => 'John Doe',
        'email' => 'john@example.com',
        'age'   => 25,
    ]);

    // Type checks
    expect($user)->toBeInstanceOf(User::class);
    expect($user->id)->toBeInt();
    expect($user->name)->toBeString();
    expect($user->email)->toBeString();

    // Value checks
    expect($user->name)->toBe('John Doe');
    expect($user->email)->toContain('@');
    expect($user->age)->toBeGreaterThan(18);
    expect($user->age)->toBeBetween(18, 65);

    // Null checks
    expect($user->deleted_at)->toBeNull();
    expect($user->id)->not->toBeNull();

    // Array checks
    $userArray = $user->toArray();
    expect($userArray)->toHaveKeys(['id', 'name', 'email']);
    expect($userArray)->not->toHaveKey('password');

    // Snapshot testing
    expect($user->name)->toMatchSnapshot();
});

// Custom expectations
expect()->extend('toBeValidEmail', function () {
    $pattern = '/^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/';
    expect($this->value)->toMatch($pattern);

    return $this;
});

it('validates email format with custom expectation', function () {
    expect('john@example.com')->toBeValidEmail();
    expect('invalid-email')->not->toBeValidEmail();
});
```

---

## ขั้นตอนที่ 1041: Testing Queued Jobs และ Events

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Jobs\SendWelcomeEmail;
use App\Events\UserRegistered;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Queue;
use Tests\TestCase;

class UserRegistrationEventTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_registration_fires_event(): void
    {
        Event::fake();

        $user = User::factory()->create();

        Event::assertDispatched(UserRegistered::class, function ($event) use ($user) {
            return $event->user->id === $user->id;
        });
    }

    public function test_registration_queues_welcome_email(): void
    {
        Queue::fake();

        $this->postJson('/api/register', [
            'name'                  => 'Test User',
            'email'                 => 'test@example.com',
            'password'              => 'Password123!',
            'password_confirmation' => 'Password123!',
        ]);

        Queue::assertPushed(SendWelcomeEmail::class, function ($job) {
            return $job->email === 'test@example.com';
        });

        Queue::assertPushedOn('emails', SendWelcomeEmail::class);
    }

    public function test_no_extra_jobs_are_queued(): void
    {
        Queue::fake();

        User::factory()->create();

        Queue::assertNothingPushed();
    }
}
```

---

## ขั้นตอนที่ 1042: Database Testing

```php
<?php

declare(strict_types=1);

namespace Tests\Feature;

use App\Models\User;
use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PostTest extends TestCase
{
    use RefreshDatabase;

    public function test_can_create_post(): void
    {
        $user = User::factory()->create();

        $post = Post::factory()->create([
            'user_id' => $user->id,
            'title'   => 'Test Post',
        ]);

        $this->assertDatabaseHas('posts', [
            'user_id' => $user->id,
            'title'   => 'Test Post',
        ]);

        $this->assertDatabaseCount('posts', 1);
    }

    public function test_deleting_user_soft_deletes(): void
    {
        $user = User::factory()->create();
        $user->delete();

        $this->assertSoftDeleted('users', ['id' => $user->id]);
        $this->assertDatabaseMissing('users', [
            'id'         => $user->id,
            'deleted_at' => null,
        ]);
    }

    public function test_user_posts_relationship(): void
    {
        $user = User::factory()
            ->has(Post::factory()->count(5))
            ->create();

        $this->assertCount(5, $user->posts);

        $user->posts->each(function (Post $post) use ($user) {
            $this->assertEquals($user->id, $post->user_id);
        });
    }
}
```

---

## สรุปบทที่ 39

| หัวข้อ | เครื่องมือ | ความสำคัญ |
|--------|-----------|-----------|
| Data Providers | PHPUnit | ทดสอบ input หลายชุด |
| Mock Objects | PHPUnit MockBuilder | แยก dependencies |
| Pest Framework | Pest | Syntax ที่อ่านง่าย |
| TDD | PHPUnit/Pest | พัฒนาจาก test |
| Mutation Testing | Infection PHP | ตรวจสอบ test quality |
| Contract Testing | Custom/Pact | API compatibility |
| Feature Tests | Laravel Testing | Integration testing |

**ต่อไป**: Part 40 - Advanced Security

---
