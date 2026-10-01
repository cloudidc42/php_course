# Part 47: PHP CLI

## ขั้นตอนที่ 1271-1300: PHP Command Line Interface

PHP ไม่ได้ใช้แค่สำหรับ web เท่านั้น ยังสามารถใช้สร้าง CLI tools ที่ทรงพลังได้

---

## ขั้นตอนที่ 1271: PHP CLI Scripts พื้นฐาน

```php
#!/usr/bin/env php
<?php

declare(strict_types=1);

// ตรวจสอบว่าเรียกจาก CLI เท่านั้น
if (PHP_SAPI !== 'cli') {
    die('This script must be run from the command line.' . PHP_EOL);
}

// รับ arguments
$options = getopt('', ['env:', 'dry-run', 'verbose', 'limit:']);

$env     = $options['env']   ?? 'production';
$dryRun  = isset($options['dry-run']);
$verbose = isset($options['verbose']);
$limit   = (int)($options['limit'] ?? 100);

// รับ positional arguments
$args = array_slice($argv, 1);
$command = $args[0] ?? 'help';

// Signal handling
pcntl_signal(SIGTERM, function () {
    echo PHP_EOL . "Received SIGTERM. Shutting down gracefully..." . PHP_EOL;
    exit(0);
});

pcntl_signal(SIGINT, function () {
    echo PHP_EOL . "Interrupted. Cleaning up..." . PHP_EOL;
    exit(0);
});

// Output helpers
function info(string $msg): void
{
    echo "\033[32m[INFO]\033[0m {$msg}" . PHP_EOL;
}

function warn(string $msg): void
{
    echo "\033[33m[WARN]\033[0m {$msg}" . PHP_EOL;
}

function error(string $msg): void
{
    echo "\033[31m[ERROR]\033[0m {$msg}" . PHP_EOL;
}

// Main logic
match ($command) {
    'run'   => runCommand($env, $dryRun, $limit),
    'check' => checkCommand($verbose),
    'help'  => showHelp(),
    default => error("Unknown command: {$command}") ?? showHelp(),
};

function runCommand(string $env, bool $dryRun, int $limit): void
{
    info("Running in {$env} environment");
    $dryRun && warn("DRY RUN MODE - no changes will be made");
    // ...
}

function showHelp(): void
{
    echo <<<HELP
Usage: script.php [command] [options]

Commands:
  run     Run the main process
  check   Check system health
  help    Show this help

Options:
  --env=<env>     Environment (default: production)
  --dry-run       Run without making changes
  --verbose       Verbose output
  --limit=<n>     Limit processing to N items

HELP;
}
```

---

## ขั้นตอนที่ 1272: Symfony Console Component

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(
    name: 'data:export',
    description: 'Export data to various formats',
)]
class ExportDataCommand extends Command
{
    protected function configure(): void
    {
        $this
            ->addArgument('entity', InputArgument::REQUIRED, 'Entity to export (users, orders, products)')
            ->addOption('format', 'f', InputOption::VALUE_OPTIONAL, 'Export format (csv, json, xlsx)', 'csv')
            ->addOption('from', null, InputOption::VALUE_OPTIONAL, 'Start date (Y-m-d)')
            ->addOption('to', null, InputOption::VALUE_OPTIONAL, 'End date (Y-m-d)')
            ->addOption('limit', 'l', InputOption::VALUE_OPTIONAL, 'Max records', 1000)
            ->addOption('output', 'o', InputOption::VALUE_OPTIONAL, 'Output file path');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);

        $entity = $input->getArgument('entity');
        $format = $input->getOption('format');
        $limit  = (int)$input->getOption('limit');
        $from   = $input->getOption('from');
        $to     = $input->getOption('to');

        $io->title('Data Export Tool');
        $io->text([
            "Entity: <info>{$entity}</info>",
            "Format: <info>{$format}</info>",
            "Limit:  <info>{$limit}</info>",
        ]);

        // Confirm before large exports
        if ($limit > 10000) {
            if (!$io->confirm("Exporting more than 10,000 records. Continue?", false)) {
                $io->warning('Export cancelled');
                return Command::SUCCESS;
            }
        }

        try {
            $io->section('Fetching data...');
            $data = $this->fetchData($entity, $from, $to, $limit);
            $io->success("Found {$data->count()} records");

            $io->section('Exporting...');
            $outputPath = $this->export($data, $format, $input->getOption('output'));
            $io->success("Exported to: {$outputPath}");

            return Command::SUCCESS;
        } catch (\Exception $e) {
            $io->error($e->getMessage());
            return Command::FAILURE;
        }
    }

    private function fetchData(string $entity, ?string $from, ?string $to, int $limit): object
    {
        // ... fetch from database
        return collect([]);
    }

    private function export(object $data, string $format, ?string $outputPath): string
    {
        $path = $outputPath ?? "export_" . date('Y-m-d_H-i-s') . ".{$format}";
        // ... export logic
        return $path;
    }
}
```

---

## ขั้นตอนที่ 1273: Progress Bars, Tables, Questions

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Helper\ProgressBar;
use Symfony\Component\Console\Helper\Table;
use Symfony\Component\Console\Helper\TableSeparator;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;
use Symfony\Component\Console\Question\ChoiceQuestion;
use Symfony\Component\Console\Question\ConfirmationQuestion;
use Symfony\Component\Console\Question\Question;
use Symfony\Component\Console\Style\SymfonyStyle;

class InteractiveCommand extends Command
{
    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = new SymfonyStyle($input, $output);
        $helper = $this->getHelper('question');

        // Text question
        $nameQuestion = new Question('Enter your name: ');
        $nameQuestion->setValidator(function ($value) {
            if (empty(trim($value))) {
                throw new \RuntimeException('Name cannot be empty');
            }
            return $value;
        });
        $name = $helper->ask($input, $output, $nameQuestion);

        // Choice question
        $environmentQuestion = new ChoiceQuestion(
            'Select environment:',
            ['development', 'staging', 'production'],
            0 // default
        );
        $environment = $helper->ask($input, $output, $environmentQuestion);

        // Confirmation
        $confirmQuestion = new ConfirmationQuestion(
            "Deploy to <comment>{$environment}</comment>? [y/N] ",
            false
        );

        if (!$helper->ask($input, $output, $confirmQuestion)) {
            $io->info('Deployment cancelled');
            return Command::SUCCESS;
        }

        // Progress bar
        $items = range(1, 100);
        $progressBar = new ProgressBar($output, count($items));
        $progressBar->setFormat(' %current%/%max% [%bar%] %percent:3s%% %elapsed:6s%/%estimated:-6s% %memory:6s%');
        $progressBar->start();

        foreach ($items as $item) {
            usleep(50000); // Simulate work
            $progressBar->advance();
        }

        $progressBar->finish();
        $output->writeln('');

        // Table
        $table = new Table($output);
        $table->setHeaders(['ID', 'Name', 'Status', 'Created']);
        $table->addRows([
            [1, 'Item One', '<info>Active</info>', '2025-01-01'],
            [2, 'Item Two', '<comment>Pending</comment>', '2025-01-02'],
            new TableSeparator(),
            ['', '', '<info>Total: 2</info>', ''],
        ]);
        $table->render();

        return Command::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1274: Cron Jobs ด้วย PHP

```php
<?php

declare(strict_types=1);

// app/Console/Kernel.php - Laravel Scheduler
namespace App\Console;

use Illuminate\Console\Scheduling\Schedule;
use Illuminate\Foundation\Console\Kernel as ConsoleKernel;

class Kernel extends ConsoleKernel
{
    protected function schedule(Schedule $schedule): void
    {
        // ทุกนาที
        $schedule->command('queue:prune-failed --hours=168')
            ->everyMinute()
            ->withoutOverlapping();

        // ทุกชั่วโมง
        $schedule->command('cache:prune-stale-tags')
            ->hourly();

        // ทุกวัน ตี 2 (เวลาไทย UTC+7)
        $schedule->command('backup:run')
            ->dailyAt('02:00')
            ->timezone('Asia/Bangkok')
            ->onOneServer()    // รันแค่ server เดียวใน cluster
            ->emailOutputOnFailure('devops@example.com');

        // ทุกวันจันทร์ เวลา 8 โมงเช้า
        $schedule->command('report:weekly')
            ->weeklyOn(1, '08:00');

        // ทุกวันสิ้นเดือน
        $schedule->command('billing:generate-invoices')
            ->monthlyOn(1, '00:00');

        // Custom frequency
        $schedule->command('analytics:aggregate')
            ->cron('*/15 * * * *')  // ทุก 15 นาที
            ->runInBackground();

        // Closure
        $schedule->call(function () {
            \DB::table('sessions')
                ->where('last_activity', '<', now()->subHours(24))
                ->delete();
        })->hourly();

        // Queue jobs
        $schedule->job(new \App\Jobs\CleanupTempFiles())
            ->daily()
            ->onQueue('maintenance');
    }
}
```

```cron
# crontab entry - รัน Laravel scheduler ทุกนาที
* * * * * cd /var/www/myapp && php artisan schedule:run >> /dev/null 2>&1
```

---

## ขั้นตอนที่ 1275: Long-running Daemons

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Illuminate\Console\Command;

class WebhookProcessor extends Command
{
    protected $signature = 'webhooks:process {--memory=128 : Memory limit in MB}';
    protected $description = 'Long-running webhook processor daemon';

    private bool $shouldStop = false;
    private int $processedCount = 0;

    public function handle(): int
    {
        // Memory limit
        $memoryLimit = (int)$this->option('memory');
        ini_set('memory_limit', $memoryLimit . 'M');

        $this->info("Starting webhook processor daemon (PID: " . getmypid() . ")");

        // Register signal handlers
        pcntl_async_signals(true);

        pcntl_signal(SIGTERM, function () {
            $this->warn("SIGTERM received. Will stop after current job.");
            $this->shouldStop = true;
        });

        pcntl_signal(SIGINT, function () {
            $this->warn("SIGINT received. Stopping...");
            $this->shouldStop = true;
        });

        // Main loop
        while (!$this->shouldStop) {
            try {
                $webhook = $this->fetchNextWebhook();

                if (!$webhook) {
                    // ไม่มีงาน - พัก 1 วินาที
                    sleep(1);
                    continue;
                }

                $this->processWebhook($webhook);
                $this->processedCount++;

                if ($this->processedCount % 100 === 0) {
                    $this->info("Processed {$this->processedCount} webhooks");
                }

                // ตรวจสอบ memory
                if ($this->isMemoryExceeded($memoryLimit)) {
                    $this->warn("Memory limit approaching. Restarting...");
                    break;
                }
            } catch (\Exception $e) {
                $this->error("Error processing webhook: {$e->getMessage()}");
                sleep(5); // รอก่อน retry
            }
        }

        $this->info("Daemon stopped. Processed {$this->processedCount} webhooks total.");
        return Command::SUCCESS;
    }

    private function fetchNextWebhook(): ?object
    {
        return \DB::table('webhooks')
            ->where('status', 'pending')
            ->where('retry_at', '<=', now())
            ->orderBy('created_at')
            ->first();
    }

    private function processWebhook(object $webhook): void
    {
        \DB::table('webhooks')->where('id', $webhook->id)->update(['status' => 'processing']);

        try {
            $processor = app(\App\Services\WebhookProcessorService::class);
            $processor->process($webhook);

            \DB::table('webhooks')->where('id', $webhook->id)->update([
                'status'       => 'completed',
                'completed_at' => now(),
            ]);
        } catch (\Exception $e) {
            $attempts = $webhook->attempts + 1;

            \DB::table('webhooks')->where('id', $webhook->id)->update([
                'status'     => $attempts >= 3 ? 'failed' : 'pending',
                'attempts'   => $attempts,
                'last_error' => $e->getMessage(),
                'retry_at'   => now()->addMinutes(5 * $attempts),
            ]);

            throw $e;
        }
    }

    private function isMemoryExceeded(int $limitMb): bool
    {
        $current = memory_get_usage(true);
        $limit   = $limitMb * 1024 * 1024;

        return $current > ($limit * 0.9); // 90% ของ limit
    }
}
```

---

## ขั้นตอนที่ 1276: Signal Handling

```php
<?php

declare(strict_types=1);

// Signal handling สำหรับ graceful shutdown

namespace App\Console;

class GracefulShutdown
{
    private bool $shuttingDown = false;
    private array $cleanupCallbacks = [];

    public function __construct()
    {
        pcntl_async_signals(true);

        pcntl_signal(SIGTERM, [$this, 'handleShutdown']);
        pcntl_signal(SIGINT, [$this, 'handleShutdown']);
        pcntl_signal(SIGHUP, [$this, 'handleReload']);
    }

    public function onShutdown(callable $callback): void
    {
        $this->cleanupCallbacks[] = $callback;
    }

    public function isShuttingDown(): bool
    {
        return $this->shuttingDown;
    }

    public function handleShutdown(int $signal): void
    {
        $signalName = match ($signal) {
            SIGTERM => 'SIGTERM',
            SIGINT  => 'SIGINT',
            default => "Signal {$signal}",
        };

        echo PHP_EOL . "Received {$signalName}. Shutting down gracefully..." . PHP_EOL;
        $this->shuttingDown = true;

        // รัน cleanup callbacks
        foreach ($this->cleanupCallbacks as $callback) {
            try {
                $callback();
            } catch (\Exception $e) {
                echo "Cleanup error: {$e->getMessage()}" . PHP_EOL;
            }
        }
    }

    public function handleReload(int $signal): void
    {
        echo "Received SIGHUP. Reloading configuration..." . PHP_EOL;
        // Reload config...
    }
}

// การใช้งาน
$shutdown = new GracefulShutdown();

$db = new PDO('mysql:host=localhost;dbname=myapp', 'user', 'pass');

// ลงทะเบียน cleanup
$shutdown->onShutdown(function () use ($db) {
    echo "Closing database connection..." . PHP_EOL;
    $db = null;
});

$shutdown->onShutdown(function () {
    echo "Flushing logs..." . PHP_EOL;
    // flush logs
});

// Main loop
while (!$shutdown->isShuttingDown()) {
    // Process work...
    sleep(1);
}
```

---

## สรุปบทที่ 47

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| CLI Scripts | PHP SAPI | Simple automation |
| Symfony Console | symfony/console | Rich CLI tools |
| Progress Bars | ProgressBar | User feedback |
| Tables | Table helper | Structured output |
| Cron Jobs | Laravel Scheduler | Scheduled tasks |
| Daemons | Long-running process | Background services |
| Signal Handling | pcntl_signal | Graceful shutdown |

**ต่อไป**: Part 48 - Laravel Notifications

---
