# Part 69: Database Scaling

## ขั้นตอนที่ 1931-1960: Database Scaling Strategies

เมื่อข้อมูลเพิ่มขึ้น การ scale database อย่างถูกต้องเป็นสิ่งสำคัญ

---

## ขั้นตอนที่ 1931: Read Replicas ใน Laravel

```php
<?php

declare(strict_types=1);

// config/database.php - Read/Write separation
return [
    'connections' => [
        'mysql' => [
            'driver' => 'mysql',

            // Write connection (primary)
            'write' => [
                'host' => [
                    env('DB_HOST', '127.0.0.1'),
                ],
            ],

            // Read connections (replicas)
            'read' => [
                'host' => [
                    env('DB_READ_HOST_1', '127.0.0.1'),
                    env('DB_READ_HOST_2', '127.0.0.1'),
                    env('DB_READ_HOST_3', '127.0.0.1'),
                ],
            ],

            'sticky'    => true,  // ใช้ write connection หลังจากเขียนใน request เดียวกัน
            'database'  => env('DB_DATABASE', 'forge'),
            'username'  => env('DB_USERNAME', 'forge'),
            'password'  => env('DB_PASSWORD', ''),
            'charset'   => 'utf8mb4',
            'collation' => 'utf8mb4_unicode_ci',
        ],
    ],
];
```

```php
<?php

declare(strict_types=1);

// การใช้งาน - Laravel จะเลือก connection อัตโนมัติ
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Order extends Model
{
    // SELECT จะใช้ read replica
    public function getRecentOrders(): \Illuminate\Database\Eloquent\Collection
    {
        return static::where('created_at', '>=', now()->subDays(30))
            ->orderBy('created_at', 'desc')
            ->get();
    }

    // INSERT/UPDATE/DELETE จะใช้ write
    public function placeOrder(array $data): static
    {
        return static::create($data);
    }

    // Force ใช้ write connection
    public function getCriticalData(int $id): ?static
    {
        return static::onWriteConnection()->find($id);
    }
}
```

---

## ขั้นตอนที่ 1932: CQRS กับ Separate Read/Write Databases

```php
<?php

declare(strict_types=1);

// app/Infrastructure/CQRS/ReadRepository.php
namespace App\Infrastructure\CQRS;

use Illuminate\Support\Facades\DB;

abstract class ReadRepository
{
    protected function getConnection(): \Illuminate\Database\Connection
    {
        return DB::connection('mysql_read');
    }

    protected function query(string $table): \Illuminate\Database\Query\Builder
    {
        return $this->getConnection()->table($table);
    }
}

// app/Infrastructure/CQRS/WriteRepository.php
abstract class WriteRepository
{
    protected function getConnection(): \Illuminate\Database\Connection
    {
        return DB::connection('mysql_write');
    }
}

// app/Repositories/Read/ProductReadRepository.php
namespace App\Repositories\Read;

use App\Infrastructure\CQRS\ReadRepository;

class ProductReadRepository extends ReadRepository
{
    public function findById(int $id): ?array
    {
        return $this->query('products')
            ->join('categories', 'products.category_id', '=', 'categories.id')
            ->select([
                'products.*',
                'categories.name as category_name',
            ])
            ->where('products.id', $id)
            ->first()?->toArray();
    }

    public function searchWithFilters(array $filters, int $page = 1, int $perPage = 20): array
    {
        $query = $this->query('products')
            ->where('products.status', 'active');

        if (!empty($filters['category_id'])) {
            $query->where('products.category_id', $filters['category_id']);
        }

        if (!empty($filters['min_price'])) {
            $query->where('products.price', '>=', $filters['min_price']);
        }

        if (!empty($filters['max_price'])) {
            $query->where('products.price', '<=', $filters['max_price']);
        }

        $total   = $query->count();
        $items   = $query->offset(($page - 1) * $perPage)->limit($perPage)->get();

        return [
            'total'    => $total,
            'page'     => $page,
            'per_page' => $perPage,
            'items'    => $items->toArray(),
        ];
    }
}

// app/Repositories/Write/ProductWriteRepository.php
namespace App\Repositories\Write;

use App\Infrastructure\CQRS\WriteRepository;
use App\Models\Product;

class ProductWriteRepository extends WriteRepository
{
    public function save(Product $product): Product
    {
        $product->save();
        return $product;
    }

    public function update(int $id, array $data): bool
    {
        return Product::where('id', $id)->update($data) > 0;
    }

    public function delete(int $id): bool
    {
        return Product::destroy($id) > 0;
    }
}
```

---

## ขั้นตอนที่ 1933: Database Partitioning

```sql
-- MySQL Table Partitioning ตาม date range
CREATE TABLE orders (
    id         BIGINT UNSIGNED AUTO_INCREMENT,
    user_id    INT UNSIGNED NOT NULL,
    total      DECIMAL(10, 2) NOT NULL,
    status     VARCHAR(20) NOT NULL,
    created_at DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)
)
PARTITION BY RANGE (YEAR(created_at) * 100 + MONTH(created_at)) (
    PARTITION p202401 VALUES LESS THAN (202402),
    PARTITION p202402 VALUES LESS THAN (202403),
    PARTITION p202403 VALUES LESS THAN (202404),
    PARTITION p202404 VALUES LESS THAN (202405),
    PARTITION p202405 VALUES LESS THAN (202406),
    PARTITION p202406 VALUES LESS THAN (202407),
    PARTITION p202407 VALUES LESS THAN (202408),
    PARTITION p202408 VALUES LESS THAN (202409),
    PARTITION p202409 VALUES LESS THAN (202410),
    PARTITION p202410 VALUES LESS THAN (202411),
    PARTITION p202411 VALUES LESS THAN (202412),
    PARTITION p202412 VALUES LESS THAN (202501),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- PostgreSQL Table Partitioning
CREATE TABLE events (
    id         BIGSERIAL,
    user_id    INTEGER NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    payload    JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- สร้าง partitions ทีละเดือน
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Auto create partitions ด้วย pg_partman
SELECT partman.create_parent(
    p_parent_table  => 'public.events',
    p_control       => 'created_at',
    p_interval      => 'monthly',
    p_start_partition => '2024-01-01'
);
```

```php
<?php

declare(strict_types=1);

// app/Console/Commands/CreateMonthlyPartition.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class CreateMonthlyPartition extends Command
{
    protected $signature   = 'db:create-partition {table} {month?}';
    protected $description = 'Create monthly partition for a table';

    public function handle(): int
    {
        $table     = $this->argument('table');
        $month     = $this->argument('month') ? new \DateTime($this->argument('month')) : now()->addMonth();
        $startDate = $month->format('Y-m-01');
        $endDate   = $month->format('Y-m-t');
        $endDate   = date('Y-m-d', strtotime($endDate . ' +1 day'));
        $partition = $table . '_' . $month->format('Y_m');

        try {
            DB::statement("
                CREATE TABLE {$partition} PARTITION OF {$table}
                FOR VALUES FROM ('{$startDate}') TO ('{$endDate}')
            ");

            $this->info("Created partition: {$partition}");
        } catch (\Exception $e) {
            $this->error("Failed: " . $e->getMessage());
        }

        return self::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1934: Connection Pooling กับ PgBouncer

```ini
; /etc/pgbouncer/pgbouncer.ini
[databases]
myapp = host=127.0.0.1 port=5432 dbname=myapp

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432

auth_type   = md5
auth_file   = /etc/pgbouncer/userlist.txt

; Transaction pooling (ดีที่สุดสำหรับ PHP-FPM)
pool_mode = transaction

; จำนวน connections ต่อ pool
max_client_conn   = 1000
default_pool_size = 20
min_pool_size     = 5
reserve_pool_size = 5
reserve_pool_timeout = 3

; Timeouts
server_connect_timeout = 15
server_idle_timeout    = 600
client_idle_timeout    = 0
query_timeout          = 0

; Logging
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid
```

---

## ขั้นตอนที่ 1935: Database Sharding

```php
<?php

declare(strict_types=1);

// app/Services/ShardingManager.php
namespace App\Services;

use Illuminate\Database\DatabaseManager;

class ShardingManager
{
    private int $shardCount;
    private array $shardConnections;

    public function __construct(private DatabaseManager $db)
    {
        $this->shardCount       = config('sharding.count', 4);
        $this->shardConnections = config('sharding.connections', []);
    }

    /**
     * กำหนด shard จาก entity ID
     */
    public function getShardId(int $entityId): int
    {
        return $entityId % $this->shardCount;
    }

    /**
     * ดึง connection สำหรับ entity
     */
    public function getConnection(int $entityId): \Illuminate\Database\Connection
    {
        $shardId = $this->getShardId($entityId);
        return $this->db->connection("shard_{$shardId}");
    }

    /**
     * Query ทุก shards
     */
    public function queryAllShards(callable $queryBuilder): \Illuminate\Support\Collection
    {
        $results = collect();

        for ($i = 0; $i < $this->shardCount; $i++) {
            $connection = $this->db->connection("shard_{$i}");
            $shardResults = $queryBuilder($connection);
            $results = $results->concat($shardResults);
        }

        return $results;
    }

    /**
     * Insert ไปยัง shard ที่ถูกต้อง
     */
    public function insert(int $shardKey, string $table, array $data): bool
    {
        return $this->getConnection($shardKey)->table($table)->insert($data);
    }
}

// config/sharding.php
return [
    'count'       => 4,
    'connections' => ['shard_0', 'shard_1', 'shard_2', 'shard_3'],
];

// config/database.php - Shard connections
'shard_0' => [
    'driver'   => 'mysql',
    'host'     => env('SHARD_0_HOST', '127.0.0.1'),
    'database' => 'myapp_shard_0',
    'username' => env('DB_USERNAME'),
    'password' => env('DB_PASSWORD'),
],
'shard_1' => [
    'driver'   => 'mysql',
    'host'     => env('SHARD_1_HOST', '127.0.0.1'),
    'database' => 'myapp_shard_1',
    'username' => env('DB_USERNAME'),
    'password' => env('DB_PASSWORD'),
],
```

---

## ขั้นตอนที่ 1936: Zero-Downtime Migrations

```php
<?php

declare(strict_types=1);

// database/migrations/2024_01_01_add_new_column_safely.php
// เทคนิค: expand-contract pattern

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

// Phase 1: Expand - เพิ่ม column ใหม่ (backward compatible)
return new class extends Migration
{
    public function up(): void
    {
        // ไม่ลบ column เก่าก่อน
        Schema::table('users', function (Blueprint $table) {
            // เพิ่ม column ใหม่ที่ nullable หรือมี default
            $table->string('display_name')->nullable()->after('name');
            $table->index('display_name');
        });

        // Backfill ข้อมูล - ต้องทำแบบ batch เพื่อไม่ lock table
        $this->backfillDisplayName();
    }

    private function backfillDisplayName(): void
    {
        \DB::table('users')
            ->whereNull('display_name')
            ->orderBy('id')
            ->chunkById(1000, function ($users) {
                foreach ($users as $user) {
                    \DB::table('users')
                        ->where('id', $user->id)
                        ->update(['display_name' => $user->name]);
                }
            });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('display_name');
        });
    }
};
```

```php
<?php

declare(strict_types=1);

// app/Console/Commands/MigrateWithLock.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class SafeMigrate extends Command
{
    protected $signature   = 'migrate:safe {--step=} {--pretend}';
    protected $description = 'Run migrations with safety checks';

    public function handle(): int
    {
        // Check replication lag ก่อน migrate
        $lag = $this->getReplicationLag();

        if ($lag > 5) { // seconds
            $this->error("Replication lag too high: {$lag}s. Aborting.");
            return self::FAILURE;
        }

        // Check active connections
        $activeConns = $this->getActiveConnections();
        if ($activeConns > 100) {
            $this->warn("High active connections: {$activeConns}. Proceed carefully.");

            if (!$this->confirm('Continue?')) {
                return self::FAILURE;
            }
        }

        // Run migration
        $this->call('migrate', [
            '--step'    => $this->option('step'),
            '--pretend' => $this->option('pretend'),
            '--force'   => true,
        ]);

        $this->info('Migration completed successfully.');
        return self::SUCCESS;
    }

    private function getReplicationLag(): float
    {
        $result = DB::select('SHOW SLAVE STATUS');
        return $result[0]?->Seconds_Behind_Master ?? 0;
    }

    private function getActiveConnections(): int
    {
        $result = DB::selectOne('SHOW STATUS WHERE Variable_name = "Threads_connected"');
        return (int) $result?->Value;
    }
}
```

---

## สรุป Part 69

| Strategy | Use Case | เมื่อใช้ |
|---------|---------|---------|
| Read Replicas | Read-heavy workloads | > 80% reads |
| CQRS | Complex queries | Separate read/write models |
| Partitioning | Time-series data | > 100M rows |
| PgBouncer | PHP-FPM + PostgreSQL | > 1000 clients |
| Sharding | Massive scale | > 1B rows |
| Zero-downtime Migrations | Production deploys | ต้องการ uptime 100% |

ถัดไป → Part 70: Advanced CI/CD Pipelines
