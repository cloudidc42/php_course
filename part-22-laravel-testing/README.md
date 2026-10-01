# Part 22: Laravel Testing
## ขั้นตอนที่ 551-580: Test-Driven Development

---

## ขั้นตอนที่ 551: PHPUnit & Pest Setup

```bash
# PHPUnit (built-in กับ Laravel)
php artisan make:test UserTest
php artisan make:test UserTest --unit
php artisan test
php artisan test --filter=UserTest
php artisan test --group=auth
php artisan test --parallel  # Run in parallel

# Pest (Modern testing framework)
composer require pestphp/pest --dev --with-all-dependencies
composer require pestphp/pest-plugin-laravel --dev
php artisan pest:install
php artisan pest:test UserTest
./vendor/bin/pest
./vendor/bin/pest --filter=UserTest
```

---

## ขั้นตอนที่ 552: Feature Tests

```php
<?php
// tests/Feature/UserTest.php

namespace Tests\Feature;

use App\Models\User;
use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Illuminate\Foundation\Testing\WithFaker;
use Tests\TestCase;

class UserTest extends TestCase {
    use RefreshDatabase; // Reset DB ทุก test
    // use DatabaseTransactions; // Rollback แทน
    
    // ========================
    // Authentication Tests
    // ========================
    public function test_user_can_login(): void {
        $user = User::factory()->create([
            'password' => bcrypt('password123'),
        ]);
        
        $response = $this->post('/login', [
            'email' => $user->email,
            'password' => 'password123',
        ]);
        
        $response->assertRedirect('/dashboard');
        $this->assertAuthenticatedAs($user);
    }
    
    public function test_login_with_wrong_password_fails(): void {
        $user = User::factory()->create();
        
        $response = $this->post('/login', [
            'email' => $user->email,
            'password' => 'wrongpassword',
        ]);
        
        $response->assertSessionHasErrors('email');
        $this->assertGuest();
    }
    
    public function test_user_can_logout(): void {
        $user = User::factory()->create();
        
        $response = $this->actingAs($user)->post('/logout');
        
        $response->assertRedirect('/');
        $this->assertGuest();
    }
    
    // ========================
    // CRUD Tests
    // ========================
    public function test_user_list_is_accessible(): void {
        $users = User::factory(5)->create();
        $admin = User::factory()->admin()->create();
        
        $response = $this->actingAs($admin)->get('/admin/users');
        
        $response->assertOk();
        $response->assertSee($users->first()->name);
        $response->assertViewIs('admin.users.index');
        $response->assertViewHas('users');
    }
    
    public function test_user_can_be_created(): void {
        $admin = User::factory()->admin()->create();
        
        $response = $this->actingAs($admin)->post('/admin/users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'SecurePass123!',
            'role' => 'user',
        ]);
        
        $response->assertRedirect();
        $response->assertSessionHas('success');
        
        $this->assertDatabaseHas('users', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
        ]);
    }
    
    public function test_user_creation_requires_valid_email(): void {
        $admin = User::factory()->admin()->create();
        
        $response = $this->actingAs($admin)->post('/admin/users', [
            'name' => 'John',
            'email' => 'not-an-email',
            'password' => 'password',
        ]);
        
        $response->assertSessionHasErrors('email');
        $this->assertDatabaseCount('users', 1); // Only admin
    }
    
    public function test_user_can_be_deleted(): void {
        $admin = User::factory()->admin()->create();
        $user = User::factory()->create();
        
        $response = $this->actingAs($admin)->delete("/admin/users/{$user->id}");
        
        $response->assertRedirect();
        $this->assertModelMissing($user);
        // OR:
        $this->assertDatabaseMissing('users', ['id' => $user->id]);
    }
    
    // ========================
    // API Tests
    // ========================
    public function test_api_returns_users(): void {
        $user = User::factory()->create();
        User::factory(9)->create();
        
        $response = $this->actingAs($user, 'sanctum')
            ->getJson('/api/users');
        
        $response->assertOk()
            ->assertJsonStructure([
                'data' => [
                    '*' => ['id', 'name', 'email']
                ],
                'meta' => ['total', 'page', 'per_page']
            ])
            ->assertJsonPath('meta.total', 10);
    }
    
    public function test_api_requires_authentication(): void {
        $response = $this->getJson('/api/users');
        $response->assertUnauthorized();
    }
    
    public function test_api_creates_user(): void {
        $admin = User::factory()->admin()->create();
        
        $response = $this->actingAs($admin, 'sanctum')
            ->postJson('/api/users', [
                'name' => 'New User',
                'email' => 'new@example.com',
                'password' => 'SecurePass123!',
            ]);
        
        $response->assertCreated()
            ->assertJsonPath('data.name', 'New User')
            ->assertJsonPath('data.email', 'new@example.com');
        
        $this->assertDatabaseHas('users', ['email' => 'new@example.com']);
    }
}
```

---

## ขั้นตอนที่ 553: Unit Tests

```php
<?php
// tests/Unit/Services/UserServiceTest.php

namespace Tests\Unit\Services;

use App\Models\User;
use App\Repositories\UserRepository;
use App\Services\UserService;
use Mockery;
use Tests\TestCase;

class UserServiceTest extends TestCase {
    private UserService $service;
    private UserRepository $repository;
    
    protected function setUp(): void {
        parent::setUp();
        
        // Mock the repository
        $this->repository = Mockery::mock(UserRepository::class);
        $this->service = new UserService($this->repository);
    }
    
    protected function tearDown(): void {
        Mockery::close();
        parent::tearDown();
    }
    
    public function test_get_all_users_returns_paginated_results(): void {
        $users = User::factory(5)->make(); // make() = no DB
        
        $this->repository
            ->shouldReceive('findAll')
            ->once()
            ->with(1, 15)
            ->andReturn($users);
        
        $result = $this->service->getAllUsers(page: 1, perPage: 15);
        
        $this->assertCount(5, $result);
    }
    
    public function test_create_user_hashes_password(): void {
        $userData = [
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => 'plaintext',
        ];
        
        $this->repository
            ->shouldReceive('create')
            ->once()
            ->withArgs(function(array $data) {
                return $data['name'] === 'Test User'
                    && $data['email'] === 'test@example.com'
                    && password_verify('plaintext', $data['password']);
            })
            ->andReturn(new User($userData));
        
        $user = $this->service->createUser($userData);
        $this->assertInstanceOf(User::class, $user);
    }
    
    public function test_create_user_with_duplicate_email_throws_exception(): void {
        $this->repository
            ->shouldReceive('findByEmail')
            ->once()
            ->andReturn(new User()); // Email exists
        
        $this->expectException(\App\Exceptions\EmailAlreadyExistsException::class);
        
        $this->service->createUser([
            'name' => 'Test',
            'email' => 'existing@example.com',
            'password' => 'pass',
        ]);
    }
}
```

---

## ขั้นตอนที่ 554: Factories & Seeders

```php
<?php
// database/factories/UserFactory.php

namespace Database\Factories;

use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;
use Illuminate\Support\Str;

class UserFactory extends Factory {
    protected $model = User::class;
    
    public function definition(): array {
        return [
            'name' => fake('th_TH')->name(),
            'email' => fake()->unique()->safeEmail(),
            'email_verified_at' => now(),
            'password' => bcrypt('password'),
            'remember_token' => Str::random(10),
            'role' => 'user',
            'active' => true,
            'created_at' => fake()->dateTimeBetween('-1 year', 'now'),
        ];
    }
    
    // States
    public function admin(): static {
        return $this->state(fn(array $attrs) => ['role' => 'admin']);
    }
    
    public function inactive(): static {
        return $this->state(['active' => false]);
    }
    
    public function unverified(): static {
        return $this->state(['email_verified_at' => null]);
    }
    
    public function withPosts(int $count = 3): static {
        return $this->has(Post::factory($count), 'posts');
    }
}

// PostFactory
class PostFactory extends Factory {
    public function definition(): array {
        return [
            'title' => fake()->sentence(6),
            'slug' => fn(array $attrs) => Str::slug($attrs['title']),
            'body' => fake()->paragraphs(5, true),
            'excerpt' => fake()->paragraph(),
            'user_id' => User::factory(),
            'published_at' => fake()->optional(0.7)->dateTimeThisYear(),
            'status' => fake()->randomElement(['draft', 'published', 'archived']),
        ];
    }
    
    public function published(): static {
        return $this->state([
            'status' => 'published',
            'published_at' => fake()->dateTimeBetween('-6 months', 'now'),
        ]);
    }
    
    public function draft(): static {
        return $this->state(['status' => 'draft', 'published_at' => null]);
    }
    
    public function forUser(User $user): static {
        return $this->state(['user_id' => $user->id]);
    }
}

// Usage in tests
$user = User::factory()->create();
$admin = User::factory()->admin()->create();
$users = User::factory(10)->create();
$userWithPosts = User::factory()->withPosts(5)->create();

// Seeders
class DatabaseSeeder extends Seeder {
    public function run(): void {
        // Create admin
        User::factory()->admin()->create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
        ]);
        
        // Create users with posts
        User::factory(20)
            ->has(Post::factory(5)->published())
            ->create();
        
        // Call other seeders
        $this->call([
            CategorySeeder::class,
            TagSeeder::class,
        ]);
    }
}

// Run: php artisan db:seed
// Or: php artisan migrate:fresh --seed
```

---

## ขั้นตอนที่ 555: Pest Testing (Modern)

```php
<?php
// tests/Feature/UserTest.php (Pest style)

use App\Models\User;
use App\Models\Post;

// Basic test
test('users can be listed', function () {
    $admin = User::factory()->admin()->create();
    User::factory(5)->create();
    
    $this->actingAs($admin)
        ->get('/admin/users')
        ->assertOk()
        ->assertViewIs('admin.users.index');
});

// Using it()
it('requires authentication to access admin', function () {
    $this->get('/admin/users')
        ->assertRedirect('/login');
});

// Datasets (Data Provider)
dataset('invalid emails', [
    'not-an-email',
    '@nodomain.com',
    'no-at-sign',
    '',
    'spaces in@email.com',
]);

it('validates email format', function (string $email) {
    $admin = User::factory()->admin()->create();
    
    $this->actingAs($admin)
        ->post('/admin/users', ['name' => 'Test', 'email' => $email])
        ->assertSessionHasErrors('email');
})->with('invalid emails');

// Expectations
test('user model has correct structure', function () {
    $user = User::factory()->create([
        'name' => 'Test User',
        'email' => 'test@example.com',
        'role' => 'admin',
    ]);
    
    expect($user)
        ->name->toBe('Test User')
        ->email->toBe('test@example.com')
        ->role->toBe('admin')
        ->isAdmin()->toBeTrue()
        ->created_at->not->toBeNull();
});

// Snapshot testing
test('api response matches snapshot', function () {
    $user = User::factory()->create(['name' => 'Test User']);
    
    $response = $this->actingAs($user, 'sanctum')
        ->getJson("/api/users/{$user->id}");
    
    expect($response->json('data'))->toMatchSnapshot();
});

// Group tests
describe('UserController', function () {
    beforeEach(function () {
        $this->admin = User::factory()->admin()->create();
    });
    
    it('can create user', function () {
        $response = $this->actingAs($this->admin)
            ->post('/admin/users', User::factory()->make()->toArray() + ['password' => 'pass123!']);
        
        $response->assertRedirect();
    });
    
    it('can delete user', function () {
        $user = User::factory()->create();
        
        $this->actingAs($this->admin)
            ->delete("/admin/users/{$user->id}")
            ->assertRedirect();
        
        $this->assertModelMissing($user);
    });
});
```

---

## ขั้นตอนที่ 556: Mocking & HTTP Tests

```php
<?php
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Storage;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Notification;

class OrderTest extends TestCase {
    public function test_order_sends_confirmation_email(): void {
        Mail::fake();
        
        $user = User::factory()->create();
        $order = Order::factory()->for($user)->create();
        
        // Process order
        (new OrderService())->process($order);
        
        Mail::assertSent(\App\Mail\OrderConfirmation::class, function ($mail) use ($user, $order) {
            return $mail->hasTo($user->email)
                && $mail->order->id === $order->id;
        });
        
        Mail::assertSent(\App\Mail\OrderConfirmation::class, 1); // Sent once
        Mail::assertNotSent(\App\Mail\OrderFailed::class);
    }
    
    public function test_payment_job_is_queued(): void {
        Queue::fake();
        
        $order = Order::factory()->create();
        (new OrderService())->pay($order, amount: 1500.00);
        
        Queue::assertPushed(\App\Jobs\ProcessPayment::class, function ($job) use ($order) {
            return $job->order->id === $order->id;
        });
    }
    
    public function test_file_upload(): void {
        Storage::fake('public');
        
        $user = User::factory()->create();
        $file = \Illuminate\Http\UploadedFile::fake()->image('avatar.jpg', 300, 300);
        
        $this->actingAs($user)
            ->post('/profile/avatar', ['avatar' => $file])
            ->assertRedirect();
        
        Storage::disk('public')->assertExists('avatars/' . $file->hashName());
    }
    
    public function test_external_api_call(): void {
        Http::fake([
            'api.payment.com/*' => Http::response([
                'status' => 'success',
                'transaction_id' => 'TX_12345',
            ], 200),
            'api.sms.com/*' => Http::response(['sent' => true], 200),
        ]);
        
        $result = (new PaymentGateway())->charge(1000.00, 'card_token');
        
        $this->assertTrue($result['success']);
        Http::assertSent(fn($req) => $req->url() === 'https://api.payment.com/charge');
    }
    
    public function test_notification_is_sent(): void {
        Notification::fake();
        
        $user = User::factory()->create();
        $user->notify(new \App\Notifications\NewMessage('Hello!'));
        
        Notification::assertSentTo($user, \App\Notifications\NewMessage::class, function ($notification) {
            return $notification->message === 'Hello!';
        });
    }
    
    public function test_event_is_dispatched(): void {
        Event::fake();
        
        $user = User::factory()->create();
        (new UserService())->registerUser($user);
        
        Event::assertDispatched(\App\Events\UserRegistered::class, function ($event) use ($user) {
            return $event->user->id === $user->id;
        });
    }
}
```

---

## 🎯 สรุป Part 22

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| PHPUnit | TestCase, RefreshDatabase, assertX |
| Feature Tests | actingAs, HTTP assertions |
| Unit Tests | Mockery, isolated testing |
| Factories | Factory states, relationships |
| Pest | Modern syntax, datasets, describe |
| Fakes | Mail, Queue, Storage, Http, Event |

**ถัดไป → Part 23: Laravel Advanced (Queue, Events, Cache)**
