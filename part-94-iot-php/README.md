# Part 94: IoT กับ PHP
## ขั้นตอนที่ 2681-2710: MQTT, Sensor Data และ Real-time Dashboards

การพัฒนา IoT backend ด้วย PHP, MQTT protocol,
Time-series storage และ Real-time monitoring

---

## ขั้นตอนที่ 2681: MQTT Client กับ PHP

```bash
composer require php-mqtt/client
```

```php
<?php
declare(strict_types=1);

namespace App\Services\IoT;

use PhpMqtt\Client\MqttClient;
use PhpMqtt\Client\ConnectionSettings;
use Illuminate\Support\Facades\Log;

class MQTTService
{
    private MqttClient $client;

    public function __construct()
    {
        $settings = (new ConnectionSettings())
            ->setUsername(config('iot.mqtt.username'))
            ->setPassword(config('iot.mqtt.password'))
            ->setConnectTimeout(5)
            ->setKeepAliveInterval(60)
            ->setLastWillTopic('devices/status')
            ->setLastWillMessage('offline')
            ->setLastWillQualityOfService(1);

        $this->client = new MqttClient(
            host: config('iot.mqtt.host', 'localhost'),
            port: (int)config('iot.mqtt.port', 1883),
            clientId: config('iot.mqtt.client_id', 'laravel-iot-' . uniqid()),
        );

        $this->client->connect($settings);
    }

    public function subscribe(string $topic, callable $callback, int $qos = 0): void
    {
        $this->client->subscribe($topic, function (string $topic, string $message) use ($callback) {
            $data = json_decode($message, true) ?? ['raw' => $message];
            $callback($topic, $data);
        }, $qos);
    }

    public function publish(string $topic, array|string $message, int $qos = 0): void
    {
        $payload = is_array($message) ? json_encode($message) : $message;
        $this->client->publish($topic, $payload, $qos);
    }

    public function listen(int $seconds = 0): void
    {
        $this->client->loop($seconds > 0, $seconds * 1000);
    }

    public function disconnect(): void
    {
        $this->client->disconnect();
    }
}
```

---

## ขั้นตอนที่ 2682: Artisan Command สำหรับ MQTT Listener

```php
<?php
declare(strict_types=1);

namespace App\Console\Commands;

use App\Services\IoT\MQTTService;
use App\Services\IoT\SensorDataProcessor;
use Illuminate\Console\Command;

class MQTTListenerCommand extends Command
{
    protected $signature = 'iot:listen 
                            {--topic=sensors/# : MQTT topic to subscribe}
                            {--qos=0 : Quality of Service level}';

    protected $description = 'Listen to MQTT topics for IoT sensor data';

    public function __construct(
        private readonly MQTTService $mqtt,
        private readonly SensorDataProcessor $processor,
    ) {
        parent::__construct();
    }

    public function handle(): int
    {
        $topic = $this->option('topic');
        $qos = (int)$this->option('qos');

        $this->info("Subscribing to topic: {$topic}");

        $this->mqtt->subscribe($topic, function (string $topic, array $data) {
            $this->processMessage($topic, $data);
        }, $qos);

        $this->info('Listening for messages... (Ctrl+C to stop)');

        // Loop indefinitely
        $this->mqtt->listen();

        return self::SUCCESS;
    }

    private function processMessage(string $topic, array $data): void
    {
        try {
            $this->processor->process($topic, $data);
            $this->line(sprintf(
                "[%s] %s: %s",
                now()->format('H:i:s'),
                $topic,
                json_encode($data)
            ));
        } catch (\Throwable $e) {
            $this->error("Error processing message: {$e->getMessage()}");
        }
    }
}
```

---

## ขั้นตอนที่ 2683: Sensor Data Processor

```php
<?php
declare(strict_types=1);

namespace App\Services\IoT;

use App\Events\IoT\SensorAlertTriggered;
use App\Models\IoT\Device;
use App\Models\IoT\SensorReading;
use Illuminate\Support\Facades\Cache;

class SensorDataProcessor
{
    private array $alertThresholds = [];

    public function process(string $topic, array $data): void
    {
        // Parse topic: sensors/{device_id}/{sensor_type}
        $parts = explode('/', $topic);
        if (count($parts) < 3) return;

        $deviceId = $parts[1];
        $sensorType = $parts[2];

        // บันทึก reading
        $reading = $this->storeReading($deviceId, $sensorType, $data);

        // อัพเดต device status
        $this->updateDeviceStatus($deviceId, $data);

        // ตรวจสอบ alerts
        $this->checkAlerts($reading);

        // Broadcast ไปยัง WebSocket
        broadcast(new \App\Events\IoT\SensorDataReceived($reading));
    }

    private function storeReading(string $deviceId, string $sensorType, array $data): SensorReading
    {
        $device = Device::where('device_id', $deviceId)->firstOrFail();

        return SensorReading::create([
            'device_id' => $device->id,
            'sensor_type' => $sensorType,
            'value' => $data['value'] ?? null,
            'unit' => $data['unit'] ?? null,
            'raw_data' => $data,
            'recorded_at' => isset($data['timestamp'])
                ? \Carbon\Carbon::createFromTimestamp($data['timestamp'])
                : now(),
            'battery_level' => $data['battery'] ?? null,
            'signal_strength' => $data['rssi'] ?? null,
        ]);
    }

    private function updateDeviceStatus(string $deviceId, array $data): void
    {
        $device = Device::where('device_id', $deviceId)->first();
        if (!$device) return;

        $device->update([
            'last_seen_at' => now(),
            'status' => 'online',
            'battery_level' => $data['battery'] ?? $device->battery_level,
        ]);

        // Cache online status
        Cache::put("device:status:{$deviceId}", 'online', 300); // 5 minutes
    }

    private function checkAlerts(SensorReading $reading): void
    {
        $thresholds = $this->getThresholds($reading->device_id, $reading->sensor_type);

        foreach ($thresholds as $threshold) {
            $triggered = match ($threshold['operator']) {
                '>' => $reading->value > $threshold['value'],
                '<' => $reading->value < $threshold['value'],
                '>=' => $reading->value >= $threshold['value'],
                '<=' => $reading->value <= $threshold['value'],
                '==' => $reading->value == $threshold['value'],
                default => false,
            };

            if ($triggered) {
                event(new SensorAlertTriggered($reading, $threshold));
            }
        }
    }

    private function getThresholds(int $deviceId, string $sensorType): array
    {
        return Cache::remember(
            "iot:thresholds:{$deviceId}:{$sensorType}",
            3600,
            fn () => \App\Models\IoT\AlertThreshold::where([
                'device_id' => $deviceId,
                'sensor_type' => $sensorType,
                'is_active' => true,
            ])->get()->toArray()
        );
    }
}
```

---

## ขั้นตอนที่ 2684: Time-series Data Storage

```php
<?php
declare(strict_types=1);

// database/migrations/xxxx_create_sensor_readings_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('sensor_readings', function (Blueprint $table) {
            $table->id();
            $table->foreignId('device_id')->constrained('iot_devices');
            $table->string('sensor_type', 50);
            $table->decimal('value', 15, 6)->nullable();
            $table->string('unit', 20)->nullable();
            $table->json('raw_data')->nullable();
            $table->tinyInteger('battery_level')->nullable();
            $table->smallInteger('signal_strength')->nullable();
            $table->timestamp('recorded_at')->useCurrent();

            // Partitioned by month for performance
            $table->index(['device_id', 'sensor_type', 'recorded_at']);
            $table->index('recorded_at');
        });
    }
};
```

```php
<?php
declare(strict_types=1);

namespace App\Services\IoT;

use App\Models\IoT\SensorReading;
use Illuminate\Support\Collection;

class TimeSeriesAnalytics
{
    public function getAggregated(
        int $deviceId,
        string $sensorType,
        string $startDate,
        string $endDate,
        string $interval = 'hour'
    ): Collection {
        $dateFormat = match ($interval) {
            'minute' => '%Y-%m-%d %H:%i',
            'hour' => '%Y-%m-%d %H:00',
            'day' => '%Y-%m-%d',
            'week' => '%Y-%u',
            'month' => '%Y-%m',
            default => '%Y-%m-%d %H:00',
        };

        return SensorReading::selectRaw("
            DATE_FORMAT(recorded_at, '{$dateFormat}') as time_bucket,
            AVG(value) as avg_value,
            MIN(value) as min_value,
            MAX(value) as max_value,
            COUNT(*) as reading_count,
            STD(value) as std_deviation
        ")
        ->where('device_id', $deviceId)
        ->where('sensor_type', $sensorType)
        ->whereBetween('recorded_at', [$startDate, $endDate])
        ->groupBy('time_bucket')
        ->orderBy('time_bucket')
        ->get();
    }

    public function detectAnomalies(int $deviceId, string $sensorType, int $windowMinutes = 60): array
    {
        $readings = SensorReading::where('device_id', $deviceId)
            ->where('sensor_type', $sensorType)
            ->where('recorded_at', '>=', now()->subMinutes($windowMinutes))
            ->orderBy('recorded_at')
            ->pluck('value')
            ->toArray();

        if (count($readings) < 10) return [];

        $mean = array_sum($readings) / count($readings);
        $variance = array_sum(array_map(fn ($x) => pow($x - $mean, 2), $readings)) / count($readings);
        $stdDev = sqrt($variance);

        $anomalies = [];
        foreach ($readings as $i => $value) {
            $zScore = abs(($value - $mean) / ($stdDev ?: 1));
            if ($zScore > 3) { // 3 standard deviations
                $anomalies[] = [
                    'index' => $i,
                    'value' => $value,
                    'z_score' => round($zScore, 2),
                ];
            }
        }

        return $anomalies;
    }

    public function calculateStatistics(int $deviceId, string $sensorType, int $hours = 24): array
    {
        $readings = SensorReading::where('device_id', $deviceId)
            ->where('sensor_type', $sensorType)
            ->where('recorded_at', '>=', now()->subHours($hours))
            ->pluck('value')
            ->filter()
            ->toArray();

        if (empty($readings)) {
            return [];
        }

        sort($readings);
        $count = count($readings);

        return [
            'count' => $count,
            'min' => min($readings),
            'max' => max($readings),
            'mean' => round(array_sum($readings) / $count, 4),
            'median' => $readings[intdiv($count, 2)],
            'p95' => $readings[(int)($count * 0.95)],
            'p99' => $readings[(int)($count * 0.99)],
        ];
    }
}
```

---

## ขั้นตอนที่ 2685: Real-time Dashboard API

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Models\IoT\Device;
use App\Services\IoT\TimeSeriesAnalytics;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;

class IoTDashboardController extends \App\Http\Controllers\Controller
{
    public function __construct(
        private readonly TimeSeriesAnalytics $analytics
    ) {}

    public function overview(): JsonResponse
    {
        $data = Cache::remember('iot:dashboard:overview', 30, function () {
            $devices = Device::withCount([
                'readings as readings_today' => fn ($q) => $q->whereDate('recorded_at', today()),
                'alerts as active_alerts' => fn ($q) => $q->where('resolved_at', null),
            ])
            ->get(['id', 'name', 'device_id', 'type', 'status', 'last_seen_at', 'battery_level']);

            return [
                'total_devices' => $devices->count(),
                'online_devices' => $devices->where('status', 'online')->count(),
                'offline_devices' => $devices->where('status', 'offline')->count(),
                'low_battery_devices' => $devices->where('battery_level', '<', 20)->count(),
                'total_readings_today' => $devices->sum('readings_today'),
                'active_alerts' => $devices->sum('active_alerts'),
                'devices' => $devices,
            ];
        });

        return response()->json($data);
    }

    public function deviceChart(Request $request, int $deviceId): JsonResponse
    {
        $request->validate([
            'sensor_type' => ['required', 'string'],
            'interval' => ['in:minute,hour,day'],
            'hours' => ['integer', 'min:1', 'max:720'],
        ]);

        $hours = $request->hours ?? 24;
        $interval = $request->interval ?? 'hour';

        $data = $this->analytics->getAggregated(
            deviceId: $deviceId,
            sensorType: $request->sensor_type,
            startDate: now()->subHours($hours)->toDateTimeString(),
            endDate: now()->toDateTimeString(),
            interval: $interval
        );

        return response()->json([
            'device_id' => $deviceId,
            'sensor_type' => $request->sensor_type,
            'interval' => $interval,
            'data' => $data,
            'statistics' => $this->analytics->calculateStatistics($deviceId, $request->sensor_type, $hours),
        ]);
    }

    public function sendCommand(Request $request, string $deviceId): JsonResponse
    {
        $request->validate([
            'command' => ['required', 'string'],
            'payload' => ['array'],
        ]);

        $device = Device::where('device_id', $deviceId)->firstOrFail();

        if ($device->status !== 'online') {
            return response()->json(['error' => 'Device is offline'], 422);
        }

        // ส่ง command ผ่าน MQTT
        app(\App\Services\IoT\MQTTService::class)->publish(
            "devices/{$deviceId}/commands",
            [
                'command' => $request->command,
                'payload' => $request->payload ?? [],
                'timestamp' => now()->timestamp,
                'command_id' => \Str::uuid(),
            ],
            qos: 1 // At least once delivery
        );

        return response()->json(['message' => 'Command sent', 'device_id' => $deviceId]);
    }
}
```

---

## สรุปบทที่ 94

| หัวข้อ | เทคโนโลยี | Use Case |
|--------|----------|---------|
| MQTT | php-mqtt/client | Sensor data ingestion |
| Time-series | MySQL + partitioning | Historical data |
| Analytics | PHP calculations | Aggregation, anomaly |
| Real-time | Laravel Broadcasting | Dashboard updates |
| Alerting | Events + Notifications | Threshold alerts |
| Commands | MQTT publish | Device control |

ถัดไป → Part 95: Machine Learning กับ PHP
