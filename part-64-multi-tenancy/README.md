# Part 64: Multi-Tenancy

## ขั้นตอนที่ 1781-1810: Multi-Tenancy Architecture ใน PHP

Multi-tenancy ช่วยให้ application เดียวรองรับลูกค้าหลายราย (tenants) พร้อมการแยก data และ configuration

---

## ขั้นตอนที่ 1781: Single Database กับ tenant_id

```php
<?php

declare(strict_types=1);

// app/Models/Traits/BelongsToTenant.php
namespace App\Models\Traits;

use App\Models\Tenant;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Builder;

trait BelongsToTenant
{
    protected static function bootBelongsToTenant(): void
    {
        // Auto-set tenant_id เมื่อสร้าง record
        static::creating(function ($model) {
            if (app()->bound('tenant') && !$model->tenant_id) {
                $model->tenant_id = app('tenant')->id;
            }
        });

        // Auto-filter by tenant_id เมื่อ query
        static::addGlobalScope('tenant', function (Builder $builder) {
            if (app()->bound('tenant')) {
                $builder->where($builder->getModel()->getTable() . '.tenant_id', app('tenant')->id);
            }
        });
    }

    public function tenant(): BelongsTo
    {
        return $this->belongsTo(Tenant::class);
    }

    // Bypass tenant scope เมื่อต้องการ
    public static function withoutTenantScope(): Builder
    {
        return static::withoutGlobalScope('tenant');
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Models/Product.php
namespace App\Models;

use App\Models\Traits\BelongsToTenant;
use Illuminate\Database\Eloquent\Model;

class Product extends Model
{
    use BelongsToTenant;

    protected $fillable = ['name', 'price', 'description', 'tenant_id'];
}
```

---

## ขั้นตอนที่ 1782: Tenant Middleware

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/IdentifyTenant.php
namespace App\Http\Middleware;

use App\Models\Tenant;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class IdentifyTenant
{
    public function handle(Request $request, Closure $next): Response
    {
        $tenant = $this->resolveTenant($request);

        if (!$tenant) {
            abort(404, 'Tenant not found');
        }

        if (!$tenant->is_active) {
            abort(503, 'Tenant account is suspended');
        }

        // ผูก tenant กับ application container
        app()->instance('tenant', $tenant);
        app()->instance(Tenant::class, $tenant);

        // Set tenant config
        config(['app.name' => $tenant->name]);
        config(['mail.from.address' => $tenant->email]);

        return $next($request);
    }

    private function resolveTenant(Request $request): ?Tenant
    {
        // Subdomain-based: tenant1.myapp.com
        $subdomain = $this->getSubdomain($request->getHost());

        if ($subdomain) {
            return Tenant::where('subdomain', $subdomain)->first();
        }

        // Custom domain
        return Tenant::where('custom_domain', $request->getHost())->first();
    }

    private function getSubdomain(string $host): ?string
    {
        $appDomain = config('app.domain', 'myapp.com');
        $parts     = explode('.', $host);

        if (count($parts) >= 3 && str_ends_with($host, ".{$appDomain}")) {
            return $parts[0];
        }

        return null;
    }
}
```

---

## ขั้นตอนที่ 1783: Separate Databases per Tenant

```php
<?php

declare(strict_types=1);

// app/Services/TenantDatabaseManager.php
namespace App\Services;

use App\Models\Tenant;
use Illuminate\Support\Facades\DB;
use Illuminate\Database\ConnectionResolverInterface;

class TenantDatabaseManager
{
    public function __construct(private ConnectionResolverInterface $resolver) {}

    public function connect(Tenant $tenant): void
    {
        $config = $this->buildConfig($tenant);

        // Add dynamic connection
        config(["database.connections.tenant_{$tenant->id}" => $config]);

        // Set current tenant connection
        DB::purge('tenant');
        config(['database.connections.tenant' => $config]);
        DB::reconnect('tenant');
    }

    public function disconnect(): void
    {
        DB::purge('tenant');
    }

    private function buildConfig(Tenant $tenant): array
    {
        return [
            'driver'   => 'mysql',
            'host'     => $tenant->db_host ?? config('database.connections.mysql.host'),
            'port'     => $tenant->db_port ?? config('database.connections.mysql.port'),
            'database' => $tenant->db_name ?? "tenant_{$tenant->id}",
            'username' => $tenant->db_username ?? config('database.connections.mysql.username'),
            'password' => $tenant->db_password ?? config('database.connections.mysql.password'),
            'charset'  => 'utf8mb4',
            'prefix'   => '',
        ];
    }

    public function createDatabase(Tenant $tenant): void
    {
        $dbName = "tenant_{$tenant->id}";

        DB::statement("CREATE DATABASE IF NOT EXISTS `{$dbName}` CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci");

        $tenant->update(['db_name' => $dbName]);

        // Run migrations
        $this->connect($tenant);
        $this->runMigrations();
        $this->disconnect();
    }

    private function runMigrations(): void
    {
        \Artisan::call('migrate', [
            '--database' => 'tenant',
            '--path'     => 'database/migrations/tenant',
            '--force'    => true,
        ]);
    }

    public function deleteDatabase(Tenant $tenant): void
    {
        $dbName = $tenant->db_name ?? "tenant_{$tenant->id}";
        DB::statement("DROP DATABASE IF EXISTS `{$dbName}`");
    }
}
```

---

## ขั้นตอนที่ 1784: Laravel Tenancy Package

```bash
# ติดตั้ง stancl/tenancy
composer require stancl/tenancy

# Setup
php artisan tenancy:install
php artisan migrate
```

```php
<?php

declare(strict_types=1);

// app/Models/Tenant.php (stancl/tenancy)
namespace App\Models;

use Stancl\Tenancy\Database\Models\Tenant as BaseTenant;
use Stancl\Tenancy\Contracts\TenantWithDatabase;
use Stancl\Tenancy\Database\Concerns\HasDatabase;
use Stancl\Tenancy\Database\Concerns\HasDomains;

class Tenant extends BaseTenant implements TenantWithDatabase
{
    use HasDatabase, HasDomains;

    public static function getCustomColumns(): array
    {
        return [
            'id',
            'name',
            'plan',
            'is_active',
        ];
    }
}
```

```php
<?php

declare(strict_types=1);

// config/tenancy.php
return [
    'tenant_model' => \App\Models\Tenant::class,
    'id_generator' => \Stancl\Tenancy\UUIDGenerator::class,

    'domain_model' => \Stancl\Tenancy\Database\Models\Domain::class,

    'central_domains' => [
        'myapp.com',
        'www.myapp.com',
    ],

    'bootstrappers' => [
        \Stancl\Tenancy\Bootstrappers\DatabaseTenancyBootstrapper::class,
        \Stancl\Tenancy\Bootstrappers\CacheTenancyBootstrapper::class,
        \Stancl\Tenancy\Bootstrappers\FilesystemTenancyBootstrapper::class,
        \Stancl\Tenancy\Bootstrappers\QueueTenancyBootstrapper::class,
    ],

    'database' => [
        'central_connection' => env('DB_CONNECTION', 'mysql'),
        'template_tenant_connection' => null,
        'prefix' => 'tenant',
        'suffix' => '',
        'managers' => [
            'sqlite' => \Stancl\Tenancy\Database\TenantDatabaseManagers\SQLiteDatabaseManager::class,
            'mysql'  => \Stancl\Tenancy\Database\TenantDatabaseManagers\MySQLDatabaseManager::class,
            'pgsql'  => \Stancl\Tenancy\Database\TenantDatabaseManagers\PostgreSQLDatabaseManager::class,
        ],
    ],

    'cache' => [
        'tag_base' => 'tenant',
    ],

    'filesystem' => [
        'suffix_base'      => 'tenant',
        'disks'            => ['local', 'public'],
        'root_override'    => [
            'local'  => '%storage_path%/app/',
            'public' => '%storage_path%/app/public/',
        ],
        'override_storage_path' => null,
        'asset_helper_tenancy' => true,
    ],

    'redis' => [
        'prefix_base' => 'tenant',
        'prefixed_connections' => ['default'],
    ],
];
```

---

## ขั้นตอนที่ 1785: Tenant Routes

```php
<?php

declare(strict_types=1);

// routes/tenant.php
use Illuminate\Support\Facades\Route;

// Tenant-specific routes (accessible via subdomain)
Route::middleware([
    'web',
    \Stancl\Tenancy\Middleware\InitializeTenancyByDomain::class,
    \Stancl\Tenancy\Middleware\PreventAccessFromCentralDomains::class,
])->group(function () {
    Route::get('/', function () {
        return 'Hello from tenant: ' . tenant('name');
    });

    Route::prefix('api')->middleware('api')->group(function () {
        Route::apiResource('products', \App\Http\Controllers\Tenant\ProductController::class);
        Route::apiResource('orders', \App\Http\Controllers\Tenant\OrderController::class);
    });
});

// routes/web.php - Central domain routes
Route::middleware(['web'])->group(function () {
    Route::get('/', [\App\Http\Controllers\Central\HomeController::class, 'index']);
    Route::get('/pricing', [\App\Http\Controllers\Central\PricingController::class, 'index']);

    Route::middleware('auth')->group(function () {
        Route::post('/tenants', [\App\Http\Controllers\Central\TenantController::class, 'store']);
    });
});
```

---

## ขั้นตอนที่ 1786: Tenant Provisioning

```php
<?php

declare(strict_types=1);

// app/Actions/CreateTenant.php
namespace App\Actions;

use App\Models\Tenant;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Str;

class CreateTenant
{
    public function execute(array $data, User $owner): Tenant
    {
        return DB::transaction(function () use ($data, $owner) {
            // สร้าง tenant
            $tenant = Tenant::create([
                'id'        => Str::ulid(),
                'name'      => $data['company_name'],
                'plan'      => $data['plan'] ?? 'starter',
                'is_active' => true,
            ]);

            // สร้าง domain
            $tenant->createDomain([
                'domain' => $data['subdomain'] . '.' . config('app.domain'),
            ]);

            // Bootstrap tenant (สร้าง database, migrate, etc.)
            tenancy()->initialize($tenant);

            // สร้าง admin user ใน tenant database
            $tenantUser = User::create([
                'name'     => $owner->name,
                'email'    => $owner->email,
                'password' => $owner->password,
                'role'     => 'admin',
            ]);

            // ส่ง welcome email
            \Mail::to($owner->email)->send(new \App\Mail\TenantWelcome($tenant, $data['subdomain']));

            tenancy()->end();

            return $tenant;
        });
    }
}
```

---

## ขั้นตอนที่ 1787: Cross-Tenant Queries (Super Admin)

```php
<?php

declare(strict_types=1);

// app/Console/Commands/TenantReport.php
namespace App\Console\Commands;

use App\Models\Tenant;
use Illuminate\Console\Command;

class TenantReport extends Command
{
    protected $signature = 'tenant:report';

    public function handle(): int
    {
        $tenants = Tenant::all();

        $this->table(
            ['Tenant', 'Plan', 'Users', 'Orders', 'Revenue'],
            $tenants->map(function (Tenant $tenant) {
                tenancy()->initialize($tenant);

                $stats = [
                    $tenant->name,
                    $tenant->plan,
                    \App\Models\User::count(),
                    \App\Models\Order::count(),
                    '฿' . number_format(\App\Models\Order::sum('total'), 2),
                ];

                tenancy()->end();
                return $stats;
            })
        );

        return self::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1788: Tenant Isolation Security

```php
<?php

declare(strict_types=1);

// app/Policies/TenantPolicy.php
namespace App\Policies;

use App\Models\Tenant;
use App\Models\User;

class TenantPolicy
{
    public function view(User $user, Tenant $tenant): bool
    {
        // Super admin สามารถดูทุก tenant
        if ($user->isSuperAdmin()) {
            return true;
        }

        // User ทั่วไปดูได้แค่ tenant ตัวเอง
        return $user->tenant_id === $tenant->id;
    }

    public function update(User $user, Tenant $tenant): bool
    {
        if ($user->isSuperAdmin()) return true;

        return $user->tenant_id === $tenant->id
            && $user->hasRole('admin');
    }
}

// app/Http/Middleware/EnsureTenantIsolation.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class EnsureTenantIsolation
{
    public function handle(Request $request, Closure $next): mixed
    {
        // Ensure all queries are scoped to current tenant
        if (!app()->bound('tenant') && !$request->routeIs('central.*')) {
            abort(400, 'Tenant context required');
        }

        return $next($request);
    }
}
```

---

## สรุป Part 64

| Approach | Isolation | Complexity | Cost |
|---------|----------|------------|------|
| Single DB + tenant_id | Row-level | Low | Low |
| Separate Schemas | Schema-level | Medium | Medium |
| Separate Databases | DB-level | High | High |
| stancl/tenancy | Flexible | Medium-High | Medium |

| หัวข้อ | รายละเอียด |
|--------|-----------|
| BelongsToTenant Trait | Auto-scope queries |
| Subdomain Identification | tenant.app.com |
| DB per Tenant | Max isolation |
| stancl/tenancy | Package solution |
| Tenant Provisioning | Auto setup |
| Cross-tenant Admin | Super admin queries |

ถัดไป → Part 65: WordPress Multisite
