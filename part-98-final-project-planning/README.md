# Part 98: Final Project - Planning & Architecture
## ขั้นตอนที่ 2801-2830: SaaS Platform Design

Database Design, Multi-tenant Architecture,
API Design, Subscription Billing และ Security Planning

---

## ขั้นตอนที่ 2801: SaaS Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     SAAS PLATFORM                       │
│                   "TeamFlow Pro"                        │
├─────────────────────────────────────────────────────────┤
│  Frontend (Vue 3 + TypeScript)                          │
│  ├── Marketing site (SSR - Nuxt 3)                      │
│  ├── App (SPA - Vue 3 + Inertia)                        │
│  └── Admin Panel (Vue 3)                                │
├─────────────────────────────────────────────────────────┤
│  API Layer (Laravel 11)                                  │
│  ├── REST API (v1, v2)                                  │
│  ├── WebSocket (Laravel Echo + Reverb)                  │
│  └── Webhook System                                     │
├─────────────────────────────────────────────────────────┤
│  Business Logic                                         │
│  ├── Multi-tenant with Row-level isolation              │
│  ├── RBAC (Spatie Permission)                           │
│  ├── Subscription Billing (Stripe)                      │
│  └── Feature Flags                                      │
├─────────────────────────────────────────────────────────┤
│  Infrastructure                                         │
│  ├── PostgreSQL (primary data)                          │
│  ├── Redis (cache + queues + sessions)                  │
│  ├── S3 (file storage)                                  │
│  └── CloudFront (CDN)                                   │
└─────────────────────────────────────────────────────────┘
```

---

## ขั้นตอนที่ 2802: Multi-tenant Database Design

```php
<?php
declare(strict_types=1);

/**
 * Multi-tenancy Strategy: Row-level isolation
 * ทุก table มี tenant_id เพื่อแยกข้อมูล
 *
 * Tenant = Organization (บริษัท/ทีม)
 * User สามารถอยู่ใน หลาย Tenant ได้
 */

// database/migrations/0001_create_core_tables.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // Organizations (Tenants)
        Schema::create('organizations', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('slug')->unique();
            $table->string('domain')->nullable()->unique(); // custom domain
            $table->string('logo_url')->nullable();
            $table->string('plan')->default('free'); // free, starter, pro, enterprise
            $table->json('settings')->nullable();
            $table->timestamp('trial_ends_at')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();
            $table->softDeletes();

            $table->index(['slug', 'is_active']);
        });

        // Users
        Schema::create('users', function (Blueprint $table) {
            $table->id();
            $table->string('name');
            $table->string('email')->unique();
            $table->string('password');
            $table->string('avatar_url')->nullable();
            $table->string('timezone')->default('Asia/Bangkok');
            $table->string('locale')->default('th');
            $table->timestamp('email_verified_at')->nullable();
            $table->rememberToken();
            $table->timestamps();
            $table->softDeletes();
        });

        // Organization Members (pivot with roles)
        Schema::create('organization_members', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('role')->default('member'); // owner, admin, member, viewer
            $table->json('permissions')->nullable(); // granular permissions
            $table->timestamp('invited_at')->nullable();
            $table->timestamp('joined_at')->nullable();
            $table->boolean('is_active')->default(true);
            $table->timestamps();

            $table->unique(['organization_id', 'user_id']);
            $table->index(['organization_id', 'role']);
        });

        // Teams within organizations
        Schema::create('teams', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
            $table->string('name');
            $table->string('description')->nullable();
            $table->string('color', 7)->default('#3B82F6');
            $table->timestamps();
            $table->softDeletes();

            $table->index('organization_id');
        });

        // Projects
        Schema::create('projects', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
            $table->foreignId('team_id')->nullable()->constrained()->nullOnDelete();
            $table->foreignId('created_by')->constrained('users');
            $table->string('name');
            $table->string('key', 10)->comment('Short project key like PROJ');
            $table->text('description')->nullable();
            $table->string('status')->default('active');
            $table->timestamp('due_date')->nullable();
            $table->json('settings')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->unique(['organization_id', 'key']);
            $table->index(['organization_id', 'status']);
        });

        // Tasks
        Schema::create('tasks', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
            $table->foreignId('project_id')->constrained()->cascadeOnDelete();
            $table->foreignId('parent_id')->nullable()->constrained('tasks')->nullOnDelete();
            $table->foreignId('created_by')->constrained('users');
            $table->foreignId('assignee_id')->nullable()->constrained('users')->nullOnDelete();
            $table->string('title');
            $table->text('description')->nullable();
            $table->string('status')->default('todo'); // todo, in_progress, in_review, done
            $table->string('priority')->default('medium'); // low, medium, high, urgent
            $table->integer('story_points')->nullable();
            $table->integer('order')->default(0);
            $table->timestamp('due_date')->nullable();
            $table->timestamp('completed_at')->nullable();
            $table->timestamps();
            $table->softDeletes();

            $table->index(['organization_id', 'project_id', 'status']);
            $table->index(['assignee_id', 'status']);
        });
    }
};
```

---

## ขั้นตอนที่ 2803: Subscription Plans Design

```php
<?php
declare(strict_types=1);

// database/migrations/0002_create_subscription_tables.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        // Subscription Plans (stored in DB for flexibility)
        Schema::create('plans', function (Blueprint $table) {
            $table->id();
            $table->string('name'); // Free, Starter, Pro, Enterprise
            $table->string('slug')->unique();
            $table->string('stripe_monthly_price_id')->nullable();
            $table->string('stripe_yearly_price_id')->nullable();
            $table->integer('price_monthly')->default(0); // สตางค์
            $table->integer('price_yearly')->default(0);
            $table->json('features'); // Feature list
            $table->json('limits'); // { members: 5, projects: 3, storage_gb: 1 }
            $table->boolean('is_active')->default(true);
            $table->integer('sort_order')->default(0);
            $table->timestamps();
        });

        // Subscriptions
        Schema::create('subscriptions', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained()->cascadeOnDelete();
            $table->string('stripe_subscription_id')->unique()->nullable();
            $table->string('stripe_customer_id')->nullable();
            $table->foreignId('plan_id')->constrained();
            $table->string('status'); // active, past_due, canceled, trialing
            $table->string('billing_cycle')->default('monthly'); // monthly, yearly
            $table->integer('quantity')->default(1); // จำนวน seats
            $table->timestamp('trial_ends_at')->nullable();
            $table->timestamp('current_period_start')->nullable();
            $table->timestamp('current_period_end')->nullable();
            $table->timestamp('canceled_at')->nullable();
            $table->json('metadata')->nullable();
            $table->timestamps();

            $table->index(['organization_id', 'status']);
        });

        // Invoices
        Schema::create('invoices', function (Blueprint $table) {
            $table->id();
            $table->foreignId('organization_id')->constrained();
            $table->foreignId('subscription_id')->nullable()->constrained();
            $table->string('stripe_invoice_id')->unique()->nullable();
            $table->string('number')->unique(); // INV-2024-000001
            $table->string('status'); // draft, open, paid, void, uncollectible
            $table->integer('subtotal'); // สตางค์
            $table->integer('tax')->default(0);
            $table->integer('total');
            $table->string('currency', 3)->default('THB');
            $table->timestamp('due_date')->nullable();
            $table->timestamp('paid_at')->nullable();
            $table->string('pdf_url')->nullable();
            $table->timestamps();
        });
    }
};
```

---

## ขั้นตอนที่ 2804: API Design Principles

```php
<?php
declare(strict_types=1);

/**
 * API Design Guidelines สำหรับ TeamFlow Pro
 *
 * Base URL: https://api.teamflow.pro/v1
 *
 * Authentication: Bearer token (Sanctum)
 * Format: JSON
 * Pagination: cursor-based
 */

// routes/api.php
use App\Http\Controllers\Api\V1;
use Illuminate\Support\Facades\Route;

Route::prefix('v1')->group(function () {
    // Public routes
    Route::post('/auth/login', [V1\AuthController::class, 'login']);
    Route::post('/auth/register', [V1\AuthController::class, 'register']);
    Route::post('/auth/forgot-password', [V1\AuthController::class, 'forgotPassword']);
    Route::post('/auth/reset-password', [V1\AuthController::class, 'resetPassword']);

    // Authenticated routes
    Route::middleware('auth:sanctum')->group(function () {
        // Auth
        Route::post('/auth/logout', [V1\AuthController::class, 'logout']);
        Route::get('/auth/me', [V1\AuthController::class, 'me']);

        // Organizations
        Route::apiResource('organizations', V1\OrganizationController::class);
        Route::post('/organizations/{org}/invite', [V1\OrganizationController::class, 'invite']);

        // Tenant-scoped routes
        Route::prefix('org/{org}')->middleware('tenant')->group(function () {
            Route::apiResource('projects', V1\ProjectController::class);
            Route::apiResource('projects.tasks', V1\TaskController::class)->shallow();
            Route::apiResource('teams', V1\TeamController::class);
            Route::apiResource('members', V1\MemberController::class);

            // Subscription
            Route::get('/subscription', [V1\SubscriptionController::class, 'show']);
            Route::post('/subscription/upgrade', [V1\SubscriptionController::class, 'upgrade']);
            Route::delete('/subscription', [V1\SubscriptionController::class, 'cancel']);
        });
    });
});
```

---

## ขั้นตอนที่ 2805: Feature Flag System

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Models\Organization;
use Illuminate\Support\Facades\Cache;

class FeatureFlagService
{
    private array $planFeatures = [
        'free' => [
            'max_members' => 5,
            'max_projects' => 3,
            'max_storage_gb' => 1,
            'features' => ['basic_tasks', 'comments'],
        ],
        'starter' => [
            'max_members' => 25,
            'max_projects' => 20,
            'max_storage_gb' => 10,
            'features' => ['basic_tasks', 'comments', 'time_tracking', 'integrations'],
        ],
        'pro' => [
            'max_members' => 100,
            'max_projects' => -1, // unlimited
            'max_storage_gb' => 100,
            'features' => ['basic_tasks', 'comments', 'time_tracking', 'integrations', 'analytics', 'custom_fields', 'automations'],
        ],
        'enterprise' => [
            'max_members' => -1,
            'max_projects' => -1,
            'max_storage_gb' => -1,
            'features' => ['*'], // all features
            'extras' => ['sso', 'audit_log', 'priority_support', 'custom_contracts'],
        ],
    ];

    public function can(Organization $org, string $feature): bool
    {
        return Cache::remember(
            "org:{$org->id}:feature:{$feature}",
            300,
            function () use ($org, $feature): bool {
                $plan = $this->planFeatures[$org->plan] ?? $this->planFeatures['free'];

                // Enterprise มีทุก features
                if ($plan['features'] === ['*']) {
                    return true;
                }

                return in_array($feature, $plan['features']);
            }
        );
    }

    public function getLimit(Organization $org, string $limit): int
    {
        $plan = $this->planFeatures[$org->plan] ?? $this->planFeatures['free'];
        return $plan[$limit] ?? 0;
    }

    public function isWithinLimit(Organization $org, string $resource, int $current): bool
    {
        $limitKey = "max_{$resource}";
        $limit = $this->getLimit($org, $limitKey);
        return $limit === -1 || $current < $limit;
    }
}
```

---

## ขั้นตอนที่ 2806: Tenant Middleware

```php
<?php
declare(strict_types=1);

namespace App\Http\Middleware;

use App\Models\Organization;
use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class TenantMiddleware
{
    public function handle(Request $request, Closure $next): Response
    {
        $orgSlug = $request->route('org');

        $organization = Organization::where('slug', $orgSlug)
            ->where('is_active', true)
            ->first();

        if (!$organization) {
            return response()->json(['message' => 'Organization not found'], 404);
        }

        $user = $request->user();
        $member = $organization->members()
            ->where('user_id', $user->id)
            ->where('is_active', true)
            ->first();

        if (!$member) {
            return response()->json(['message' => 'Access denied'], 403);
        }

        // ตั้ง current tenant ใน context
        app()->instance('tenant', $organization);
        app()->instance('tenant.member', $member);

        $request->attributes->set('organization', $organization);
        $request->attributes->set('member', $member);

        return $next($request);
    }
}
```

---

## สรุปบทที่ 98

| หัวข้อ | เทคนิค | เหตุผล |
|--------|--------|--------|
| Multi-tenancy | Row-level isolation | Simple, scalable |
| API Design | RESTful + versioning | Maintainable |
| Feature Flags | Plan-based features | Flexible pricing |
| Database | PostgreSQL + indexes | Performance |
| Subscriptions | Stripe integration | Reliable billing |
| Architecture | Service + Repository | Testable |

ถัดไป → Part 99: Final Project Backend
