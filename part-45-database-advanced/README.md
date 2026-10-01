# Part 45: Database Advanced

## ขั้นตอนที่ 1211-1240: ฐานข้อมูลขั้นสูง

บทนี้ครอบคลุม Database Indexing, Query Optimization, Full-text Search, Migrations, Soft Deletes, Audit Trails และ Multi-tenancy

---

## ขั้นตอนที่ 1211: Database Indexing Strategies

Index คือโครงสร้างข้อมูลที่ช่วยให้ค้นหาเร็วขึ้น แต่ทำให้ write ช้าลง

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->foreignId('product_id')->constrained();
            $table->string('status', 20)->default('pending');
            $table->decimal('total', 10, 2);
            $table->string('email')->index();        // Single column index
            $table->string('reference_code')->unique(); // Unique index
            $table->json('metadata')->nullable();
            $table->timestamp('completed_at')->nullable();
            $table->timestamps();
            $table->softDeletes();

            // Composite index - ลำดับสำคัญมาก (ใช้บ่อยสุดขึ้นก่อน)
            $table->index(['user_id', 'status', 'created_at'], 'orders_user_status_date');

            // Index สำหรับ reporting queries
            $table->index(['status', 'completed_at'], 'orders_status_completed');

            // Partial index ใน PostgreSQL
            // $table->rawIndex("(status) WHERE status = 'pending'", 'orders_pending_status');
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

```sql
-- ตรวจสอบ index ที่มีอยู่
SHOW INDEX FROM orders;

-- วิเคราะห์ query plan
EXPLAIN SELECT * FROM orders
WHERE user_id = 1
  AND status = 'completed'
ORDER BY created_at DESC
LIMIT 10;

-- ผลลัพธ์ที่ดี: type=ref, key=orders_user_status_date, rows ต่ำ

-- Covering Index - ข้อมูลทุกอย่างอยู่ใน index
EXPLAIN SELECT user_id, status, created_at
FROM orders
WHERE user_id = 1
  AND status = 'pending';
-- ผลลัพธ์ที่ดี: Extra=Using index (ไม่ต้องอ่าน table เพิ่ม)
```

---

## ขั้นตอนที่ 1212: Query Optimization ด้วย EXPLAIN

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\DB;

class QueryAnalyzer
{
    public function analyzeQuery(string $sql, array $bindings = []): array
    {
        $explain = DB::select("EXPLAIN FORMAT=JSON {$sql}", $bindings);

        return json_decode($explain[0]->EXPLAIN, true);
    }

    public function getSlowQueries(int $threshold = 1000): array
    {
        return DB::select(
            'SELECT * FROM information_schema.PROCESSLIST 
             WHERE TIME > ? AND COMMAND != "Sleep"
             ORDER BY TIME DESC',
            [$threshold]
        );
    }

    public function enableQueryLog(): void
    {
        DB::enableQueryLog();
    }

    public function getQueryLog(): array
    {
        return DB::getQueryLog();
    }

    public function findNPlusOne(): void
    {
        $queries = DB::getQueryLog();
        $patterns = [];

        foreach ($queries as $query) {
            $normalized = preg_replace('/\d+/', '?', $query['query']);
            $patterns[$normalized] = ($patterns[$normalized] ?? 0) + 1;
        }

        // หา queries ที่ถูกเรียกซ้ำ > 5 ครั้ง
        $suspicious = array_filter($patterns, fn ($count) => $count > 5);

        if (!empty($suspicious)) {
            logger()->warning('Possible N+1 queries detected', $suspicious);
        }
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Builder;

class Order extends Model
{
    // Scope ที่ optimized
    public function scopeWithUserAndItems(Builder $query): Builder
    {
        return $query->with([
            'user:id,name,email',     // Select เฉพาะที่ต้องการ
            'items' => function ($q) {
                $q->with('product:id,name,price')
                  ->select(['id', 'order_id', 'product_id', 'quantity', 'price']);
            },
        ]);
    }

    // Avoid N+1 ด้วย eager loading
    public function scopeForDashboard(Builder $query): Builder
    {
        return $query->withUserAndItems()
            ->withCount('items')
            ->withSum('items', 'price');
    }

    // Chunked processing สำหรับ large datasets
    public static function processAll(callable $callback): void
    {
        static::query()
            ->where('status', 'pending')
            ->chunkById(100, function ($orders) use ($callback) {
                foreach ($orders as $order) {
                    $callback($order);
                }
            });
    }
}
```

---

## ขั้นตอนที่ 1213: Full-text Search ใน MySQL

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\DB;

return new class extends Migration
{
    public function up(): void
    {
        Schema::table('posts', function (Blueprint $table) {
            // Full-text index
            $table->fullText(['title', 'body'], 'posts_fulltext');
        });

        // Custom FULLTEXT configuration สำหรับภาษาไทย
        DB::statement('ALTER TABLE posts ADD FULLTEXT INDEX posts_content (title, body) WITH PARSER ngram');
    }

    public function down(): void
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->dropFullText('posts_fulltext');
        });
    }
};
```

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    // Boolean mode search
    public function scopeSearch(Builder $query, string $keyword): Builder
    {
        return $query->whereRaw(
            'MATCH(title, body) AGAINST(? IN BOOLEAN MODE)',
            [$this->prepareSearchKeyword($keyword)]
        );
    }

    // Natural language search พร้อม relevance score
    public function scopeSearchWithRelevance(Builder $query, string $keyword): Builder
    {
        return $query->selectRaw(
            '*, MATCH(title, body) AGAINST(? IN NATURAL LANGUAGE MODE) AS relevance',
            [$keyword]
        )
        ->whereRaw(
            'MATCH(title, body) AGAINST(? IN NATURAL LANGUAGE MODE)',
            [$keyword]
        )
        ->orderByDesc('relevance');
    }

    private function prepareSearchKeyword(string $keyword): string
    {
        $words = array_filter(explode(' ', trim($keyword)));
        $prepared = array_map(fn ($word) => "+{$word}*", $words);
        return implode(' ', $prepared);
    }
}
```

```php
<?php

declare(strict_types=1);

// การใช้งาน full-text search
$posts = Post::searchWithRelevance('PHP performance')
    ->where('status', 'published')
    ->with('author:id,name')
    ->paginate(15);

// ผลลัพธ์จะเรียงตาม relevance ที่สูงสุดก่อน
```

---

## ขั้นตอนที่ 1214: Database Migration Best Practices

```php
<?php

declare(strict_types=1);

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;
use Illuminate\Support\Facades\DB;

// Zero-downtime migration สำหรับ large tables
return new class extends Migration
{
    // ปิด transactions สำหรับ DDL statements บาง DB engines
    public bool $withinTransaction = false;

    public function up(): void
    {
        // เพิ่ม column ด้วย default value (ไม่ lock table นาน)
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)
                ->default('Asia/Bangkok')
                ->after('email');
        });

        // Backfill data แบบ chunk (ไม่ lock table)
        DB::table('users')
            ->whereNull('timezone')
            ->chunkById(500, function ($users) {
                $ids = $users->pluck('id')->toArray();

                DB::table('users')
                    ->whereIn('id', $ids)
                    ->update(['timezone' => 'Asia/Bangkok']);
            });

        // หลัง backfill - make NOT NULL
        Schema::table('users', function (Blueprint $table) {
            $table->string('timezone', 50)
                ->default('Asia/Bangkok')
                ->nullable(false)
                ->change();
        });
    }

    public function down(): void
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('timezone');
        });
    }
};
```

---

## ขั้นตอนที่ 1215: Soft Deletes และ Audit Trails

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    protected $dates = ['deleted_at', 'published_at'];

    public function trashPost(): void
    {
        $this->delete(); // Sets deleted_at timestamp
    }

    public function restore(): bool
    {
        return parent::restore(); // Clears deleted_at
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Models\Concerns;

use App\Models\AuditLog;
use Illuminate\Database\Eloquent\Model;

trait Auditable
{
    public static function bootAuditable(): void
    {
        static::created(function (Model $model) {
            AuditLog::record('created', $model);
        });

        static::updated(function (Model $model) {
            AuditLog::record('updated', $model, $model->getChanges());
        });

        static::deleted(function (Model $model) {
            AuditLog::record('deleted', $model);
        });

        static::restored(function (Model $model) {
            AuditLog::record('restored', $model);
        });
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class AuditLog extends Model
{
    protected $fillable = [
        'auditable_type',
        'auditable_id',
        'event',
        'old_values',
        'new_values',
        'user_id',
        'ip_address',
        'user_agent',
    ];

    protected $casts = [
        'old_values' => 'array',
        'new_values' => 'array',
    ];

    public static function record(string $event, Model $model, array $changes = []): void
    {
        static::create([
            'auditable_type' => get_class($model),
            'auditable_id'   => $model->getKey(),
            'event'          => $event,
            'old_values'     => $event === 'updated'
                ? array_intersect_key($model->getOriginal(), $changes)
                : null,
            'new_values'     => $changes ?: $model->getAttributes(),
            'user_id'        => auth()->id(),
            'ip_address'     => request()->ip(),
            'user_agent'     => request()->userAgent(),
        ]);
    }

    public function auditable()
    {
        return $this->morphTo();
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

---

## ขั้นตอนที่ 1216: Multi-tenancy กับ Separate Schemas

```php
<?php

declare(strict_types=1);

namespace App\MultiTenancy;

use Illuminate\Support\Facades\DB;

class TenantManager
{
    private ?string $currentTenant = null;

    public function setTenant(string $tenantId): void
    {
        $this->currentTenant = $tenantId;

        // เปลี่ยน database connection ตาม tenant
        config(['database.connections.tenant.database' => "tenant_{$tenantId}"]);
        DB::purge('tenant');
        DB::reconnect('tenant');
    }

    public function getCurrentTenant(): ?string
    {
        return $this->currentTenant;
    }

    public function runForTenant(string $tenantId, callable $callback): mixed
    {
        $previous = $this->currentTenant;

        try {
            $this->setTenant($tenantId);
            return $callback();
        } finally {
            if ($previous) {
                $this->setTenant($previous);
            } else {
                $this->currentTenant = null;
            }
        }
    }

    public function createTenantDatabase(string $tenantId): void
    {
        $dbName = "tenant_{$tenantId}";

        DB::statement("CREATE DATABASE IF NOT EXISTS `{$dbName}` 
                       CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");

        // Run migrations สำหรับ tenant ใหม่
        $this->setTenant($tenantId);
        \Artisan::call('migrate', [
            '--database' => 'tenant',
            '--path'     => 'database/migrations/tenant',
            '--force'    => true,
        ]);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Middleware;

use App\MultiTenancy\TenantManager;
use App\Models\Tenant;
use Closure;
use Illuminate\Http\Request;

class IdentifyTenant
{
    public function __construct(private TenantManager $tenantManager) {}

    public function handle(Request $request, Closure $next): mixed
    {
        // ระบุ tenant จาก subdomain
        $host = $request->getHost();
        $subdomain = explode('.', $host)[0];

        $tenant = Tenant::where('subdomain', $subdomain)->first();

        if (!$tenant) {
            return response()->json(['error' => 'Tenant not found'], 404);
        }

        $this->tenantManager->setTenant($tenant->id);

        // แนบ tenant ไปกับ request
        $request->merge(['tenant' => $tenant]);

        return $next($request);
    }
}
```

---

## ขั้นตอนที่ 1217: Database Transactions ขั้นสูง

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Illuminate\Support\Facades\DB;

class TransferService
{
    public function transfer(int $fromAccountId, int $toAccountId, float $amount): void
    {
        // Transaction with locking
        DB::transaction(function () use ($fromAccountId, $toAccountId, $amount) {
            // Lock rows ป้องกัน race condition
            $fromAccount = DB::table('accounts')
                ->where('id', $fromAccountId)
                ->lockForUpdate()   // Pessimistic locking
                ->first();

            $toAccount = DB::table('accounts')
                ->where('id', $toAccountId)
                ->lockForUpdate()
                ->first();

            if ($fromAccount->balance < $amount) {
                throw new \DomainException('Insufficient funds');
            }

            DB::table('accounts')
                ->where('id', $fromAccountId)
                ->decrement('balance', $amount);

            DB::table('accounts')
                ->where('id', $toAccountId)
                ->increment('balance', $amount);

            DB::table('transactions')->insert([
                'from_account_id' => $fromAccountId,
                'to_account_id'   => $toAccountId,
                'amount'          => $amount,
                'created_at'      => now(),
                'updated_at'      => now(),
            ]);
        }, 3); // retry 3 times on deadlock
    }

    // Optimistic locking ด้วย version column
    public function updateWithOptimisticLock(int $accountId, float $amount, int $version): bool
    {
        $updated = DB::table('accounts')
            ->where('id', $accountId)
            ->where('version', $version)
            ->update([
                'balance' => DB::raw("balance + {$amount}"),
                'version' => DB::raw('version + 1'),
            ]);

        if ($updated === 0) {
            throw new \RuntimeException('Concurrent modification detected. Please retry.');
        }

        return true;
    }
}
```

---

## สรุปบทที่ 45

| หัวข้อ | เทคนิค | ผลลัพธ์ |
|--------|--------|---------|
| Indexing | Composite, covering | Query 10x-100x faster |
| EXPLAIN | EXPLAIN FORMAT=JSON | Identify slow queries |
| Full-text Search | MATCH...AGAINST | Relevance-based search |
| Migrations | Zero-downtime, chunk | No app downtime |
| Soft Deletes | deleted_at | Recoverable data |
| Audit Trails | Auditable trait | Full change history |
| Multi-tenancy | Separate databases | Data isolation |
| Transactions | lockForUpdate, optimistic | Prevent race conditions |

**ต่อไป**: Part 46 - Laravel Queues

---
