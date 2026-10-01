# Part 50: Monitoring & Logging

## ขั้นตอนที่ 1361-1390: การ Monitor และ Log แอปพลิเคชัน

การ logging และ monitoring ที่ดีช่วยให้ตรวจพบและแก้ไขปัญหาได้รวดเร็ว

---

## ขั้นตอนที่ 1361: Monolog Channels และ Handlers

```php
<?php

declare(strict_types=1);

// config/logging.php
return [
    'default' => env('LOG_CHANNEL', 'stack'),

    'channels' => [
        'stack' => [
            'driver'            => 'stack',
            'channels'          => ['daily', 'slack'],
            'ignore_exceptions' => false,
        ],

        'single' => [
            'driver' => 'single',
            'path'   => storage_path('logs/laravel.log'),
            'level'  => 'debug',
        ],

        'daily' => [
            'driver' => 'daily',
            'path'   => storage_path('logs/laravel.log'),
            'level'  => env('LOG_LEVEL', 'debug'),
            'days'   => 14,
        ],

        'slack' => [
            'driver'   => 'slack',
            'url'      => env('LOG_SLACK_WEBHOOK_URL'),
            'username' => 'Laravel Log',
            'emoji'    => ':boom:',
            'level'    => 'critical',
        ],

        'papertrail' => [
            'driver'       => 'monolog',
            'level'        => 'debug',
            'handler'      => \Monolog\Handler\SyslogUdpHandler::class,
            'handler_with' => [
                'host' => env('PAPERTRAIL_URL'),
                'port' => env('PAPERTRAIL_PORT'),
            ],
        ],

        // JSON structured logging
        'json' => [
            'driver'    => 'monolog',
            'level'     => 'debug',
            'handler'   => \Monolog\Handler\StreamHandler::class,
            'formatter' => \Monolog\Formatter\JsonFormatter::class,
            'with'      => [
                'stream' => storage_path('logs/json.log'),
            ],
        ],

        // Separate channel สำหรับ payments
        'payments' => [
            'driver'    => 'daily',
            'path'      => storage_path('logs/payments.log'),
            'level'     => 'info',
            'days'      => 90,
            'formatter' => \Monolog\Formatter\JsonFormatter::class,
        ],

        // Null channel สำหรับ testing
        'null' => [
            'driver'  => 'monolog',
            'handler' => \Monolog\Handler\NullHandler::class,
        ],
    ],
];
```

---

## ขั้นตอนที่ 1362: Structured Logging (JSON)

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\Log;

class PaymentLogger
{
    public function logPaymentAttempt(string $orderId, float $amount, string $gateway): void
    {
        Log::channel('payments')->info('Payment attempt', [
            'event'      => 'payment.attempt',
            'order_id'   => $orderId,
            'amount'     => $amount,
            'currency'   => 'THB',
            'gateway'    => $gateway,
            'user_id'    => auth()->id(),
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
            'timestamp'  => now()->toISOString(),
        ]);
    }

    public function logPaymentSuccess(
        string $orderId,
        string $transactionId,
        float $amount
    ): void {
        Log::channel('payments')->info('Payment successful', [
            'event'          => 'payment.success',
            'order_id'       => $orderId,
            'transaction_id' => $transactionId,
            'amount'         => $amount,
            'timestamp'      => now()->toISOString(),
        ]);
    }

    public function logPaymentFailure(
        string $orderId,
        string $reason,
        string $errorCode = ''
    ): void {
        Log::channel('payments')->error('Payment failed', [
            'event'      => 'payment.failed',
            'order_id'   => $orderId,
            'reason'     => $reason,
            'error_code' => $errorCode,
            'timestamp'  => now()->toISOString(),
        ]);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Logging;

use Monolog\Formatter\JsonFormatter;
use Monolog\LogRecord;

class CustomJsonFormatter extends JsonFormatter
{
    public function format(LogRecord $record): string
    {
        // เพิ่ม fields มาตรฐาน
        $record = $record->with(extra: array_merge($record->extra, [
            'app'         => config('app.name'),
            'environment' => config('app.env'),
            'version'     => config('app.version', '1.0.0'),
            'request_id'  => request()->header('X-Request-ID', uniqid()),
        ]));

        return parent::format($record);
    }
}
```

---

## ขั้นตอนที่ 1363: Log Aggregation กับ ELK Stack

```yaml
# docker-compose.yml สำหรับ ELK Stack (development)
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - esdata:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/log/nginx:/var/log/nginx:ro
      - /var/www/myapp/storage/logs:/app/logs:ro
    depends_on:
      - logstash

volumes:
  esdata:
```

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /app/logs/*.log
    json.keys_under_root: true
    json.add_error_key: true
    fields:
      app: myapp
      environment: production

output.logstash:
  hosts: ["logstash:5044"]

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
```

```conf
# logstash/pipeline/laravel.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [app] == "myapp" {
    date {
      match => ["timestamp", "ISO8601"]
      target => "@timestamp"
    }

    mutate {
      remove_field => ["timestamp", "beat", "input"]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "myapp-logs-%{+YYYY.MM.dd}"
  }
}
```

---

## ขั้นตอนที่ 1364: Error Tracking ด้วย Sentry

```php
<?php

declare(strict_types=1);

// config/sentry.php
return [
    'dsn'                  => env('SENTRY_LARAVEL_DSN'),
    'release'              => trim(exec('git log --pretty="%h" -n1 HEAD')),
    'environment'          => app()->environment(),
    'breadcrumbs'          => [
        'logs'             => true,
        'cache'            => true,
        'livewire'         => true,
        'sql_queries'      => true,
        'sql_bindings'     => true,
        'queue_info'       => true,
        'command_info'     => true,
    ],
    'tracing'              => [
        'queue_job_transactions' => true,
        'queue_jobs'             => true,
        'sql_queries'            => true,
        'sql_origin'             => true,
        'http_client_requests'   => true,
    ],
    'traces_sample_rate'   => env('SENTRY_TRACES_SAMPLE_RATE', 0.1),
    'profiles_sample_rate' => env('SENTRY_PROFILES_SAMPLE_RATE', 0.1),
    'ignore_exceptions'    => [
        \Illuminate\Auth\AuthenticationException::class,
        \Illuminate\Validation\ValidationException::class,
        \Symfony\Component\HttpKernel\Exception\NotFoundHttpException::class,
    ],
];
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Sentry\State\Scope;

class SentryContext
{
    public function handle(Request $request, Closure $next): mixed
    {
        if ($user = $request->user()) {
            \Sentry\configureScope(function (Scope $scope) use ($user, $request) {
                $scope->setUser([
                    'id'    => $user->id,
                    'email' => $user->email,
                    'name'  => $user->name,
                ]);

                $scope->setContext('request', [
                    'url'        => $request->fullUrl(),
                    'method'     => $request->method(),
                    'ip'         => $request->ip(),
                    'user_agent' => $request->userAgent(),
                ]);

                $scope->setTag('route', $request->route()?->getName() ?? 'unknown');
            });
        }

        return $next($request);
    }
}
```

---

## ขั้นตอนที่ 1365: Application Performance Monitoring (APM)

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class PerformanceMonitor
{
    public function handle(Request $request, Closure $next): mixed
    {
        $startTime   = microtime(true);
        $startMemory = memory_get_usage(true);

        $response = $next($request);

        $duration     = (microtime(true) - $startTime) * 1000; // ms
        $memoryDelta  = memory_get_usage(true) - $startMemory;
        $peakMemory   = memory_get_peak_usage(true);

        // เพิ่ม headers สำหรับ debugging
        $response->headers->set('X-Response-Time', round($duration, 2) . 'ms');
        $response->headers->set('X-Memory-Usage', round($peakMemory / 1024 / 1024, 2) . 'MB');

        // Log slow requests
        if ($duration > 1000) { // > 1 second
            Log::channel('performance')->warning('Slow request detected', [
                'url'          => $request->fullUrl(),
                'method'       => $request->method(),
                'duration_ms'  => round($duration, 2),
                'memory_mb'    => round($peakMemory / 1024 / 1024, 2),
                'user_id'      => $request->user()?->id,
                'route'        => $request->route()?->getName(),
                'db_queries'   => count(\DB::getQueryLog()),
            ]);
        }

        // Push metrics to time-series database
        if (config('monitoring.metrics_enabled')) {
            app(\App\Services\MetricsService::class)->record([
                'request_duration' => $duration,
                'memory_usage'     => $peakMemory,
                'route'            => $request->route()?->getName() ?? 'unknown',
                'status_code'      => $response->getStatusCode(),
            ]);
        }

        return $response;
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Services;

class MetricsService
{
    public function record(array $metrics): void
    {
        // ส่งไป Prometheus/StatsD/InfluxDB
        if (config('monitoring.driver') === 'prometheus') {
            $this->recordToPrometheus($metrics);
        }
    }

    private function recordToPrometheus(array $metrics): void
    {
        // ใช้ promphp/prometheus_client_php
        $registry = app(\Prometheus\CollectorRegistry::class);

        $histogram = $registry->getOrRegisterHistogram(
            'app',
            'request_duration_milliseconds',
            'Request duration in ms',
            ['route', 'status_code'],
            [50, 100, 200, 500, 1000, 2000, 5000]
        );

        $histogram->observe(
            $metrics['request_duration'],
            [$metrics['route'], (string)$metrics['status_code']]
        );
    }
}
```

---

## ขั้นตอนที่ 1366: Alerting และ Dashboards

```php
<?php

declare(strict_types=1);

namespace App\Console\Commands;

use Illuminate\Console\Command;

class CheckSystemHealth extends Command
{
    protected $signature = 'health:check';
    protected $description = 'Check system health metrics';

    public function handle(): int
    {
        $checks = [
            'Database'     => $this->checkDatabase(),
            'Redis'        => $this->checkRedis(),
            'Queue'        => $this->checkQueue(),
            'Disk Space'   => $this->checkDiskSpace(),
            'Error Rate'   => $this->checkErrorRate(),
        ];

        $failed = array_filter($checks, fn ($c) => !$c['healthy']);

        $this->table(
            ['Component', 'Status', 'Details'],
            array_map(fn ($name, $check) => [
                $name,
                $check['healthy'] ? '<info>OK</info>' : '<error>FAILED</error>',
                $check['message'],
            ], array_keys($checks), $checks)
        );

        if (!empty($failed)) {
            $this->triggerAlerts($failed);
            return Command::FAILURE;
        }

        return Command::SUCCESS;
    }

    private function checkDatabase(): array
    {
        try {
            \DB::select('SELECT 1');
            $connections = \DB::select('SHOW STATUS WHERE Variable_name = "Threads_connected"');
            $count = $connections[0]->Value ?? 0;

            return [
                'healthy' => true,
                'message' => "Connected ({$count} active connections)",
            ];
        } catch (\Exception $e) {
            return [
                'healthy' => false,
                'message' => $e->getMessage(),
            ];
        }
    }

    private function checkRedis(): array
    {
        try {
            \Redis::ping();
            $info   = \Redis::info();
            $memory = round($info['used_memory'] / 1024 / 1024, 2);

            return [
                'healthy' => true,
                'message' => "Connected (Memory: {$memory}MB)",
            ];
        } catch (\Exception $e) {
            return [
                'healthy' => false,
                'message' => $e->getMessage(),
            ];
        }
    }

    private function checkDiskSpace(): array
    {
        $diskFree  = disk_free_space('/');
        $diskTotal = disk_total_space('/');
        $percent   = round(($diskFree / $diskTotal) * 100, 1);

        return [
            'healthy' => $percent > 10,
            'message' => "Free: {$percent}% (" . round($diskFree / 1024 / 1024 / 1024, 1) . "GB)",
        ];
    }

    private function checkQueue(): array
    {
        try {
            $backlog = \DB::table('jobs')->count();

            return [
                'healthy' => $backlog < 1000,
                'message' => "{$backlog} jobs pending",
            ];
        } catch (\Exception $e) {
            return [
                'healthy' => false,
                'message' => $e->getMessage(),
            ];
        }
    }

    private function checkErrorRate(): array
    {
        $errorCount = \DB::table('failed_jobs')
            ->where('failed_at', '>=', now()->subHour())
            ->count();

        return [
            'healthy' => $errorCount < 10,
            'message' => "{$errorCount} failures in last hour",
        ];
    }

    private function triggerAlerts(array $failedChecks): void
    {
        $message = "System health check failed:\n";
        foreach ($failedChecks as $name => $check) {
            $message .= "- {$name}: {$check['message']}\n";
        }

        \Notification::route('slack', config('slack.alerts'))
            ->notify(new \App\Notifications\SystemHealthAlert($message));
    }
}
```

---

## สรุปบทที่ 50

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| Monolog | Laravel Logging | Multiple channels |
| Structured Logs | JSON Formatter | Machine-readable |
| ELK Stack | Elasticsearch/Kibana | Log aggregation |
| Sentry | sentry/sentry-laravel | Error tracking |
| APM | Middleware + Metrics | Performance monitoring |
| Health Checks | Custom command | Proactive alerting |

**ต่อไป**: Part 51 - Laravel Broadcasting

---
