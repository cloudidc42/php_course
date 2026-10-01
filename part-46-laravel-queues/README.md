# Part 46: Laravel Queues

## ขั้นตอนที่ 1241-1270: Queue System ขั้นสูง

Queue ช่วยให้ทำงานหนักๆ แบบ asynchronous เพื่อไม่ให้ผู้ใช้รอนาน

---

## ขั้นตอนที่ 1241: Job Classes และการ Dispatch

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use App\Models\Order;
use App\Services\InvoiceService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class GenerateInvoice implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // จำนวนครั้งที่ retry ถ้า fail
    public int $tries = 3;

    // Timeout ในวินาที
    public int $timeout = 60;

    // Backoff strategy
    public array $backoff = [10, 30, 60]; // seconds

    // Queue ที่จะใช้
    public string $queue = 'invoices';

    public function __construct(
        public readonly Order $order,
        public readonly string $format = 'pdf'
    ) {}

    public function handle(InvoiceService $invoiceService): void
    {
        $invoice = $invoiceService->generate($this->order, $this->format);

        // ส่งอีเมล invoice
        $this->order->user->notify(
            new \App\Notifications\InvoiceReady($invoice)
        );
    }

    public function failed(\Throwable $exception): void
    {
        \Log::error('Failed to generate invoice', [
            'order_id' => $this->order->id,
            'error'    => $exception->getMessage(),
            'attempt'  => $this->attempts(),
        ]);

        // แจ้ง admin
        \Notification::route('slack', config('slack.alerts'))
            ->notify(new \App\Notifications\JobFailed(
                job: 'GenerateInvoice',
                context: ['order_id' => $this->order->id],
                error: $exception->getMessage()
            ));
    }

    // กำหนด unique key เพื่อป้องกัน duplicate jobs
    public function uniqueId(): string
    {
        return "invoice:order:{$this->order->id}:{$this->format}";
    }

    public function uniqueFor(): int
    {
        return 600; // 10 minutes
    }
}
```

```php
<?php

declare(strict_types=1);

// การ dispatch Jobs แบบต่างๆ

// ส่งทันที
GenerateInvoice::dispatch($order);

// ส่งด้วย delay 5 นาที
GenerateInvoice::dispatch($order)->delay(now()->addMinutes(5));

// ส่งไป queue ที่ต้องการ
GenerateInvoice::dispatch($order)->onQueue('high-priority');

// ส่งไป connection อื่น
GenerateInvoice::dispatch($order)->onConnection('redis');

// Conditional dispatch
GenerateInvoice::dispatchIf($order->isPaid(), $order);
GenerateInvoice::dispatchUnless($order->isFree(), $order);

// Dispatch after response (ส่งหลัง HTTP response)
GenerateInvoice::dispatchAfterResponse($order);
```

---

## ขั้นตอนที่ 1242: Queue Priorities และ Delays

```php
<?php

declare(strict_types=1);

// config/queue.php
return [
    'default' => env('QUEUE_CONNECTION', 'redis'),

    'connections' => [
        'redis' => [
            'driver'     => 'redis',
            'connection' => 'default',
            'queue'      => ['default'],
            'retry_after' => 90,
            'block_for'  => null,
            'after_commit' => true,
        ],
    ],
];
```

```bash
# รัน worker สำหรับ queue priority
php artisan queue:work redis --queue=critical,high,default,low

# รัน worker ด้วย memory limit
php artisan queue:work --memory=512

# รัน worker แบบ daemon
php artisan queue:work --daemon --sleep=3 --tries=3 --timeout=90
```

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

// Critical job - ส่งไป queue critical
class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public string $queue    = 'critical';
    public int $tries       = 1;     // Payment ไม่ควร retry โดยอัตโนมัติ
    public int $timeout     = 30;

    public function __construct(
        public readonly string $paymentIntentId,
        public readonly float $amount
    ) {}

    public function handle(): void
    {
        // Process payment...
    }
}

// Low priority job
class SendNewsletterBatch implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public string $queue    = 'low';
    public int $tries       = 2;
    public int $timeout     = 300; // 5 minutes

    public function __construct(
        public readonly array $userIds,
        public readonly string $campaignId
    ) {}

    public function handle(): void
    {
        // Send newsletter...
    }
}
```

---

## ขั้นตอนที่ 1243: Failed Jobs Handling

```php
<?php

declare(strict_types=1);

// ดู failed jobs
// php artisan queue:failed

// Retry failed job ทั้งหมด
// php artisan queue:retry all

// Retry specific job
// php artisan queue:retry 5

// Flush failed jobs
// php artisan queue:flush
```

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class RetryFailedJobs extends Command
{
    protected $signature = 'jobs:retry-failed
                            {--queue= : Queue name to retry}
                            {--hours=24 : Retry jobs failed in last N hours}
                            {--class= : Specific job class to retry}';

    protected $description = 'Retry failed jobs with filtering';

    public function handle(): int
    {
        $query = DB::table('failed_jobs');

        if ($queue = $this->option('queue')) {
            $query->where('queue', $queue);
        }

        if ($hours = $this->option('hours')) {
            $query->where('failed_at', '>=', now()->subHours($hours));
        }

        if ($class = $this->option('class')) {
            $query->where('payload', 'like', "%{$class}%");
        }

        $failedJobs = $query->get();

        $this->info("Found {$failedJobs->count()} failed jobs");

        $bar = $this->output->createProgressBar($failedJobs->count());
        $bar->start();

        $retried = 0;
        $errors  = 0;

        foreach ($failedJobs as $job) {
            try {
                $this->call('queue:retry', ['id' => [$job->uuid]]);
                $retried++;
            } catch (\Exception $e) {
                $errors++;
                $this->error("\nFailed to retry job {$job->uuid}: {$e->getMessage()}");
            }

            $bar->advance();
        }

        $bar->finish();
        $this->newLine();

        $this->table(
            ['Metric', 'Count'],
            [
                ['Total found', $failedJobs->count()],
                ['Retried', $retried],
                ['Errors', $errors],
            ]
        );

        return Command::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1244: Job Chaining และ Batching

```php
<?php

declare(strict_types=1);

use Illuminate\Support\Facades\Bus;
use App\Jobs\{ProcessPayment, GenerateInvoice, SendConfirmationEmail, UpdateInventory};

// Job Chain - รัน sequential
Bus::chain([
    new ProcessPayment($order->payment_intent_id, $order->total),
    new UpdateInventory($order->items),
    new GenerateInvoice($order),
    new SendConfirmationEmail($order),
])->onQueue('orders')
  ->catch(function (\Throwable $e) use ($order) {
      // ถ้า job ใดใน chain fail
      $order->update(['status' => 'failed']);
      \Log::error('Order processing chain failed', [
          'order_id' => $order->id,
          'error'    => $e->getMessage(),
      ]);
  })
  ->dispatch();
```

```php
<?php

declare(strict_types=1);

use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;
use App\Jobs\ProcessUserReport;
use App\Models\User;

// Job Batch - รัน parallel
$userIds = User::active()->pluck('id');

$batch = Bus::batch(
    $userIds->map(fn ($id) => new ProcessUserReport($id))->all()
)
->name('Generate Monthly Reports')
->allowFailures()   // ให้บาง job fail ได้โดยไม่ยกเลิก batch
->onQueue('reports')
->then(function (Batch $batch) {
    // ทุก job สำเร็จ
    \Log::info("Batch completed: {$batch->totalJobs} reports generated");
})
->catch(function (Batch $batch, \Throwable $e) {
    // มี job fail
    \Log::warning("Batch had failures: {$batch->failedJobs} failed");
})
->finally(function (Batch $batch) {
    // เรียกเสมอไม่ว่าจะสำเร็จหรือไม่
    cache()->forget('report:generating');
})
->dispatch();

// บันทึก batch ID เพื่อ track progress
cache()->put('report:batch_id', $batch->id);

// ตรวจสอบ progress
$batch = Bus::findBatch($batch->id);
echo "Progress: {$batch->progress()}%";
echo "Pending: {$batch->pendingJobs}";
echo "Failed: {$batch->failedJobs}";
```

---

## ขั้นตอนที่ 1245: Rate Limiting Jobs

```php
<?php

declare(strict_types=1);

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Queue\Middleware\RateLimited;
use Illuminate\Queue\Middleware\WithoutOverlapping;

class SendSms implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(
        public readonly string $phone,
        public readonly string $message
    ) {}

    // Middleware สำหรับ rate limit และ prevent overlap
    public function middleware(): array
    {
        return [
            // ส่ง SMS ได้ไม่เกิน 100 ต่อนาที
            new RateLimited('sms-per-minute'),

            // ป้องกันการส่ง SMS ซ้ำไปยังเบอร์เดียวกันพร้อมกัน
            (new WithoutOverlapping("sms:{$this->phone}"))
                ->expireAfter(30)
                ->releaseAfter(5),
        ];
    }

    public function handle(): void
    {
        app(\App\Services\SmsService::class)->send($this->phone, $this->message);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Providers;

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        RateLimiter::for('sms-per-minute', function (object $job) {
            return Limit::perMinute(100);
        });

        RateLimiter::for('email-per-hour', function (object $job) {
            return Limit::perHour(1000)->by($job->email ?? 'default');
        });
    }
}
```

---

## ขั้นตอนที่ 1246: Laravel Horizon Configuration

```php
<?php

declare(strict_types=1);

// app/Providers/HorizonServiceProvider.php
namespace App\Providers;

use Laravel\Horizon\Horizon;
use Laravel\Horizon\HorizonApplicationServiceProvider;

class HorizonServiceProvider extends HorizonApplicationServiceProvider
{
    public function boot(): void
    {
        parent::boot();

        // ใครสามารถดู Horizon Dashboard
        Horizon::auth(function ($request) {
            return $request->user()?->isAdmin() ?? false;
        });

        // Email เมื่อ wait time เกิน threshold
        Horizon::routeMailNotificationsTo('devops@example.com');

        // Slack notification
        Horizon::routeSlackNotificationsTo(
            config('services.slack.horizon_webhook'),
            '#alerts'
        );

        // SMS notification
        Horizon::routeSmsNotificationsTo('+66812345678');

        // Trim jobs เก็บไว้ 48 ชั่วโมง
        Horizon::night();  // Enable night mode
    }
}
```

---

## ขั้นตอนที่ 1247: Queue Monitoring

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Queue;

class MonitorQueues extends Command
{
    protected $signature = 'queues:monitor';
    protected $description = 'Monitor queue health';

    public function handle(): int
    {
        $queues = ['critical', 'high', 'default', 'low', 'emails', 'reports'];

        $this->table(
            ['Queue', 'Size', 'Status'],
            collect($queues)->map(function ($queue) {
                $size   = Queue::size($queue);
                $status = match (true) {
                    $size > 1000 => '<error>OVERLOADED</error>',
                    $size > 500  => '<comment>HIGH</comment>',
                    $size > 100  => '<info>MODERATE</info>',
                    default      => '<info>HEALTHY</info>',
                };

                return [$queue, $size, $status];
            })->all()
        );

        // Alert ถ้า queue ใดมี backlog สูง
        collect($queues)->each(function ($queue) {
            $size = Queue::size($queue);

            if ($size > 1000) {
                \Notification::route('slack', config('slack.devops'))
                    ->notify(new \App\Notifications\QueueBacklogAlert($queue, $size));
            }
        });

        return Command::SUCCESS;
    }
}
```

---

## สรุปบทที่ 46

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| Job Classes | ShouldQueue | Async processing |
| Priorities | Queue names | Critical jobs first |
| Failed Jobs | failed_jobs table | Retry mechanism |
| Job Chaining | Bus::chain | Sequential workflow |
| Job Batching | Bus::batch | Parallel processing |
| Rate Limiting | RateLimited middleware | API rate compliance |
| Horizon | Laravel Horizon | Visual monitoring |

**ต่อไป**: Part 47 - PHP CLI

---
