# Part 41: Laravel Advanced

## ขั้นตอนที่ 1091-1120: Laravel ขั้นสูง

บทนี้จะลงลึกเรื่อง Service Container, Artisan Commands, Events, Broadcasting, Horizon, Validation และ Form Requests

---

## ขั้นตอนที่ 1091: Laravel Service Container Deep Dive

Service Container คือ Dependency Injection Container ที่ทรงพลังของ Laravel

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Services\PaymentService;
use App\Gateways\StripeGateway;
use App\Gateways\PayPalGateway;
use App\Contracts\PaymentGatewayInterface;

class AppServiceProvider extends ServiceProvider
{
    // Bindings ที่จะถูก resolve ทุกครั้ง (transient)
    public array $bindings = [
        'App\Contracts\UserRepositoryInterface' => 'App\Repositories\EloquentUserRepository',
    ];

    // Singletons ที่จะถูก resolve ครั้งเดียว
    public array $singletons = [
        'App\Services\CacheService' => 'App\Services\RedisCacheService',
    ];

    public function register(): void
    {
        // Binding พื้นฐาน
        $this->app->bind(PaymentGatewayInterface::class, function ($app) {
            $gateway = config('payment.default');

            return match ($gateway) {
                'stripe' => new StripeGateway(
                    apiKey: config('services.stripe.secret'),
                    webhookSecret: config('services.stripe.webhook_secret')
                ),
                'paypal' => new PayPalGateway(
                    clientId: config('services.paypal.client_id'),
                    clientSecret: config('services.paypal.client_secret')
                ),
                default => throw new \InvalidArgumentException("Unknown gateway: {$gateway}"),
            };
        });

        // Singleton ที่ต้องการ configuration
        $this->app->singleton(PaymentService::class, function ($app) {
            return new PaymentService(
                gateway: $app->make(PaymentGatewayInterface::class),
                logger: $app->make('log'),
                retryAttempts: config('payment.retry_attempts', 3)
            );
        });

        // Named binding
        $this->app->bind('payment.stripe', fn ($app) => new StripeGateway(
            apiKey: config('services.stripe.secret'),
            webhookSecret: config('services.stripe.webhook_secret')
        ));

        // Contextual binding - class ต่างกัน ได้ instance ต่างกัน
        $this->app->when(\App\Http\Controllers\ReportController::class)
            ->needs(\App\Services\FileStorageService::class)
            ->give(fn () => new \App\Services\FileStorageService('reports'));

        $this->app->when(\App\Http\Controllers\MediaController::class)
            ->needs(\App\Services\FileStorageService::class)
            ->give(fn () => new \App\Services\FileStorageService('media'));
    }

    public function boot(): void
    {
        // Extending existing binding
        $this->app->extend(PaymentService::class, function ($service, $app) {
            return new \App\Services\CachedPaymentService(
                $service,
                $app->make('cache')
            );
        });
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Contracts\PaymentGatewayInterface;
use Psr\Log\LoggerInterface;

class PaymentService
{
    public function __construct(
        private PaymentGatewayInterface $gateway,
        private LoggerInterface $logger,
        private int $retryAttempts = 3
    ) {}

    public function charge(float $amount, string $token): array
    {
        $attempts = 0;

        while ($attempts < $this->retryAttempts) {
            try {
                $result = $this->gateway->charge($amount, $token);
                $this->logger->info('Payment successful', [
                    'amount'         => $amount,
                    'transaction_id' => $result['transaction_id'],
                ]);

                return $result;
            } catch (\Exception $e) {
                $attempts++;
                $this->logger->warning('Payment attempt failed', [
                    'attempt' => $attempts,
                    'error'   => $e->getMessage(),
                ]);

                if ($attempts >= $this->retryAttempts) {
                    throw new \App\Exceptions\PaymentFailedException(
                        'Payment failed after ' . $this->retryAttempts . ' attempts'
                    );
                }

                usleep(100000 * $attempts); // Exponential backoff
            }
        }

        throw new \App\Exceptions\PaymentFailedException('Unexpected error');
    }
}
```

---

## ขั้นตอนที่ 1092: Custom Artisan Commands

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use App\Models\User;
use App\Services\EmailService;
use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class SendDailyDigest extends Command
{
    protected $signature = 'digest:send
                            {--type=daily : Type of digest (daily/weekly/monthly)}
                            {--dry-run : Run without actually sending emails}
                            {--user= : Send only to specific user ID}';

    protected $description = 'Send daily digest emails to active users';

    public function __construct(private EmailService $emailService)
    {
        parent::__construct();
    }

    public function handle(): int
    {
        $type   = $this->option('type');
        $dryRun = $this->option('dry-run');
        $userId = $this->option('user');

        $this->info("Starting {$type} digest " . ($dryRun ? '(DRY RUN)' : ''));

        $query = User::active()->with(['posts', 'notifications']);

        if ($userId) {
            $query->where('id', $userId);
        }

        $users = $query->get();

        $this->output->progressStart($users->count());

        $sent    = 0;
        $failed  = 0;
        $skipped = 0;

        foreach ($users as $user) {
            try {
                if (!$user->wantsDigest($type)) {
                    $skipped++;
                    $this->output->progressAdvance();
                    continue;
                }

                if (!$dryRun) {
                    $this->emailService->sendDigest($user, $type);
                }

                $sent++;
            } catch (\Exception $e) {
                $failed++;
                $this->error("\nFailed for user {$user->id}: {$e->getMessage()}");
            }

            $this->output->progressAdvance();
        }

        $this->output->progressFinish();

        // แสดงตารางสรุป
        $this->table(
            ['Type', 'Count'],
            [
                ['Sent', $sent],
                ['Failed', $failed],
                ['Skipped', $skipped],
                ['Total', $users->count()],
            ]
        );

        return $failed > 0 ? Command::FAILURE : Command::SUCCESS;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Console\GeneratorCommand;

class MakeRepository extends GeneratorCommand
{
    protected $name = 'make:repository';
    protected $description = 'Create a new repository class';
    protected $type = 'Repository';

    protected function getStub(): string
    {
        return __DIR__ . '/stubs/repository.stub';
    }

    protected function getDefaultNamespace($rootNamespace): string
    {
        return $rootNamespace . '\Repositories';
    }
}
```

---

## ขั้นตอนที่ 1093: Laravel Events และ Listeners

```php
<?php

declare(strict_types=1);

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderPlaced
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Order $order,
        public readonly string $ipAddress,
        public readonly \DateTimeInterface $placedAt = new \DateTimeImmutable()
    ) {}
}
```

```php
<?php

declare(strict_types=1);

namespace App\Listeners;

use App\Events\OrderPlaced;
use App\Services\EmailService;
use App\Services\InventoryService;
use App\Services\FraudDetectionService;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class ProcessNewOrder implements ShouldQueue
{
    use InteractsWithQueue;

    public string $queue  = 'orders';
    public int $delay     = 0;
    public int $tries     = 3;
    public int $backoff   = 30; // seconds

    public function __construct(
        private EmailService $emailService,
        private InventoryService $inventoryService,
        private FraudDetectionService $fraudService
    ) {}

    public function handle(OrderPlaced $event): void
    {
        $order = $event->order;

        // ตรวจสอบการฉ้อโกง
        if ($this->fraudService->isSuspicious($order, $event->ipAddress)) {
            $order->flag('suspicious');
            return;
        }

        // อัพเดท inventory
        $this->inventoryService->decreaseStock($order->items);

        // ส่ง confirmation email
        $this->emailService->sendOrderConfirmation($order);
    }

    public function failed(OrderPlaced $event, \Throwable $exception): void
    {
        \Log::error('Failed to process order', [
            'order_id' => $event->order->id,
            'error'    => $exception->getMessage(),
        ]);

        // แจ้งเตือน admin
        \Notification::route('slack', config('slack.orders'))
            ->notify(new \App\Notifications\OrderProcessingFailed($event->order));
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        \App\Events\OrderPlaced::class => [
            \App\Listeners\ProcessNewOrder::class,
            \App\Listeners\SendOrderNotification::class,
            \App\Listeners\UpdateAnalytics::class,
        ],
        \App\Events\UserRegistered::class => [
            \App\Listeners\SendWelcomeEmail::class,
            \App\Listeners\AssignDefaultRole::class,
            \App\Listeners\CreateUserProfile::class,
        ],
    ];

    // Observer pattern
    public function boot(): void
    {
        \App\Models\Order::observe(\App\Observers\OrderObserver::class);
        \App\Models\User::observe(\App\Observers\UserObserver::class);
    }
}
```

---

## ขั้นตอนที่ 1094: Broadcasting ด้วย Pusher/Reverb

```php
<?php

declare(strict_types=1);

namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class NewChatMessage implements ShouldBroadcast
{
    use InteractsWithSockets, SerializesModels;

    public function __construct(public readonly Message $message) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('chat.' . $this->message->conversation_id),
        ];
    }

    public function broadcastAs(): string
    {
        return 'new.message';
    }

    public function broadcastWith(): array
    {
        return [
            'id'         => $this->message->id,
            'body'       => $this->message->body,
            'user'       => [
                'id'     => $this->message->user->id,
                'name'   => $this->message->user->name,
                'avatar' => $this->message->user->avatar_url,
            ],
            'created_at' => $this->message->created_at->toISOString(),
        ];
    }

    // ส่งเฉพาะ user ที่ไม่ใช่ผู้ส่ง
    public function broadcastToOthers(): bool
    {
        return true;
    }
}
```

```php
<?php

declare(strict_types=1);

// routes/channels.php
use Illuminate\Support\Facades\Broadcast;

// Private channel authorization
Broadcast::channel('chat.{conversationId}', function ($user, int $conversationId) {
    return $user->canAccessConversation($conversationId);
});

// Presence channel - ส่งข้อมูล user
Broadcast::channel('room.{roomId}', function ($user, int $roomId) {
    if ($user->canJoinRoom($roomId)) {
        return [
            'id'     => $user->id,
            'name'   => $user->name,
            'avatar' => $user->avatar_url,
        ];
    }

    return false;
});
```

---

## ขั้นตอนที่ 1095: Laravel Horizon

```php
<?php

declare(strict_types=1);

// config/horizon.php
return [
    'use'                 => 'default',
    'prefix'              => env('HORIZON_PREFIX', 'horizon:'),
    'middleware'          => ['web', 'auth', 'can:viewHorizon'],
    'waits'               => ['redis:default' => 60],
    'trim'                => [
        'recent'          => 60,
        'pending'         => 60,
        'completed'       => 60,
        'recent_failed'   => 10080,
        'failed'          => 10080,
        'monitored'       => 10080,
    ],
    'environments' => [
        'production' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue'      => ['default', 'emails', 'notifications'],
                'balance'    => 'auto',
                'processes'  => 10,
                'tries'      => 3,
                'timeout'    => 60,
            ],
            'supervisor-high' => [
                'connection' => 'redis',
                'queue'      => ['high'],
                'balance'    => 'simple',
                'processes'  => 5,
                'tries'      => 1,
                'timeout'    => 30,
            ],
        ],
        'local' => [
            'supervisor-1' => [
                'connection' => 'redis',
                'queue'      => ['default', 'emails'],
                'balance'    => 'simple',
                'processes'  => 3,
            ],
        ],
    ],
    'metrics' => [
        'trim_snapshots' => [
            'job'   => 24,
            'queue' => 24,
        ],
    ],
];
```

---

## ขั้นตอนที่ 1096: Custom Validation Rules

```php
<?php

declare(strict_types=1);

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use Illuminate\Translation\PotentiallyTranslatedString;

class ThaiPhoneNumber implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        // รูปแบบเบอร์โทรไทย: 0xxxxxxxxx (10 digits) หรือ +66xxxxxxxxx
        $pattern = '/^(\+66|0)[0-9]{9}$/';

        if (!preg_match($pattern, $value)) {
            $fail("The {$attribute} must be a valid Thai phone number.");
        }
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;
use App\Services\TaxIdVerificationService;

class ValidThaiTaxId implements ValidationRule
{
    public function __construct(
        private TaxIdVerificationService $verificationService
    ) {}

    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        // ตรวจสอบรูปแบบ
        if (!preg_match('/^[0-9]{13}$/', (string)$value)) {
            $fail('The tax ID must be 13 digits.');
            return;
        }

        // ตรวจสอบ checksum
        if (!$this->isValidChecksum((string)$value)) {
            $fail('The tax ID checksum is invalid.');
            return;
        }

        // ตรวจสอบกับ API ภาษี (optional - ช้า)
        // if (!$this->verificationService->verify($value)) {
        //     $fail('The tax ID could not be verified.');
        // }
    }

    private function isValidChecksum(string $taxId): bool
    {
        $sum = 0;
        for ($i = 0; $i < 12; $i++) {
            $sum += (int)$taxId[$i] * (13 - $i);
        }

        $checkDigit = (11 - ($sum % 11)) % 10;
        return $checkDigit === (int)$taxId[12];
    }
}
```

---

## ขั้นตอนที่ 1097: Form Requests

```php
<?php

declare(strict_types=1);

namespace App\Http\Requests;

use App\Rules\ThaiPhoneNumber;
use App\Rules\ValidThaiTaxId;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class CreateCompanyRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', \App\Models\Company::class);
    }

    public function rules(): array
    {
        return [
            'name'              => ['required', 'string', 'max:255', 'unique:companies,name'],
            'tax_id'            => ['required', new ValidThaiTaxId(app(\App\Services\TaxIdVerificationService::class))],
            'phone'             => ['required', new ThaiPhoneNumber()],
            'email'             => ['required', 'email:rfc,dns', 'unique:companies,email'],
            'address'           => ['required', 'string', 'max:500'],
            'province'          => ['required', 'string', 'exists:provinces,name'],
            'postal_code'       => ['required', 'regex:/^[0-9]{5}$/'],
            'established_year'  => ['required', 'integer', 'min:1900', 'max:' . date('Y')],
            'logo'              => ['nullable', 'image', 'max:2048', 'dimensions:min_width=100,min_height=100'],
            'categories'        => ['required', 'array', 'min:1', 'max:5'],
            'categories.*'      => ['exists:categories,id'],
        ];
    }

    public function messages(): array
    {
        return [
            'name.unique'              => 'ชื่อบริษัทนี้ถูกใช้งานแล้ว',
            'tax_id.required'          => 'กรุณากรอกเลขประจำตัวผู้เสียภาษี',
            'email.unique'             => 'อีเมลนี้ถูกลงทะเบียนแล้ว',
            'categories.min'           => 'กรุณาเลือกอย่างน้อย 1 หมวดหมู่',
            'categories.max'           => 'เลือกได้สูงสุด 5 หมวดหมู่',
            'established_year.max'     => 'ปีที่ก่อตั้งต้องไม่เกินปีปัจจุบัน',
        ];
    }

    public function attributes(): array
    {
        return [
            'name'             => 'ชื่อบริษัท',
            'tax_id'           => 'เลขประจำตัวผู้เสียภาษี',
            'phone'            => 'เบอร์โทรศัพท์',
            'email'            => 'อีเมล',
            'established_year' => 'ปีที่ก่อตั้ง',
        ];
    }

    protected function prepareForValidation(): void
    {
        // ทำความสะอาดข้อมูลก่อน validate
        $this->merge([
            'phone'   => preg_replace('/[^0-9+]/', '', $this->phone ?? ''),
            'tax_id'  => preg_replace('/[^0-9]/', '', $this->tax_id ?? ''),
            'name'    => trim($this->name ?? ''),
        ]);
    }

    protected function failedValidation(Validator $validator): void
    {
        // Custom error format
        throw new HttpResponseException(
            response()->json([
                'message' => 'Validation failed',
                'errors'  => $validator->errors()->toArray(),
            ], 422)
        );
    }

    public function passedValidation(): void
    {
        // หลัง validate สำเร็จ
        \Log::info('Company registration validated', [
            'name' => $this->name,
            'ip'   => $this->ip(),
        ]);
    }
}
```

---

## ขั้นตอนที่ 1098: Macros และ Extensions

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Support\Collection;
use Illuminate\Support\Str;
use Illuminate\Http\Request;
use Illuminate\Support\ServiceProvider;

class MacroServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Collection macro
        Collection::macro('toAssoc', function () {
            return $this->reduce(function ($carry, $item) {
                $carry[$item['key']] = $item['value'];
                return $carry;
            }, []);
        });

        Collection::macro('sumByField', function (string $field) {
            return $this->sum(fn ($item) => $item[$field] ?? 0);
        });

        // String macro
        Str::macro('thaiSlug', function (string $text): string {
            $text = transliterator_transliterate('Any-Latin; Latin-ASCII', $text);
            return Str::slug($text);
        });

        // Request macro
        Request::macro('isFromThailand', function () {
            $ip = request()->ip();
            // ตรวจสอบ IP range ไทย (simplified)
            return str_starts_with($ip, '202.') || str_starts_with($ip, '203.');
        });
    }
}
```

---

## สรุปบทที่ 41

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| Service Container | IoC Container | Dependency injection |
| Artisan Commands | Symfony Console | CLI automation |
| Events & Listeners | Laravel Events | Decoupled architecture |
| Broadcasting | Pusher/Reverb | Real-time updates |
| Horizon | Redis/Queue | Queue monitoring |
| Custom Validation | ValidationRule | Business-specific rules |
| Form Requests | HTTP Layer | Clean controller code |

**ต่อไป**: Part 42 - Laravel API

---
