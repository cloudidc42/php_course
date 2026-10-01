# Part 99: Final Project - Complete Backend
## ขั้นตอนที่ 2831-2860: Laravel SaaS Backend Implementation

Auth, Authorization, Subscription Management,
Team Management, Stripe Billing และ API Documentation

---

## ขั้นตอนที่ 2831: Authentication System

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\V1;

use App\Http\Requests\Auth\LoginRequest;
use App\Http\Requests\Auth\RegisterRequest;
use App\Models\User;
use App\Services\AuthService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class AuthController extends \App\Http\Controllers\Controller
{
    public function __construct(
        private readonly AuthService $authService
    ) {}

    public function register(RegisterRequest $request): JsonResponse
    {
        $result = $this->authService->register(
            name: $request->name,
            email: $request->email,
            password: $request->password,
            organizationName: $request->organization_name,
        );

        return response()->json([
            'message' => 'ลงทะเบียนสำเร็จ กรุณายืนยันอีเมล',
            'token' => $result['token'],
            'user' => $result['user'],
        ], 201);
    }

    public function login(LoginRequest $request): JsonResponse
    {
        $result = $this->authService->login(
            email: $request->email,
            password: $request->password,
            deviceName: $request->header('X-Device-Name', 'unknown'),
        );

        if (!$result) {
            return response()->json([
                'message' => 'อีเมลหรือรหัสผ่านไม่ถูกต้อง',
            ], 401);
        }

        return response()->json([
            'token' => $result['token'],
            'user' => $result['user'],
            'organizations' => $result['organizations'],
        ]);
    }

    public function me(Request $request): JsonResponse
    {
        $user = $request->user()->load('organizations');

        return response()->json([
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'avatar_url' => $user->avatar_url,
                'timezone' => $user->timezone,
                'locale' => $user->locale,
                'email_verified_at' => $user->email_verified_at,
                'organizations' => $user->organizations->map(fn ($org) => [
                    'id' => $org->id,
                    'name' => $org->name,
                    'slug' => $org->slug,
                    'plan' => $org->plan,
                    'role' => $org->pivot->role,
                ]),
            ],
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();
        return response()->json(['message' => 'ออกจากระบบแล้ว']);
    }
}
```

---

## ขั้นตอนที่ 2832: Auth Service

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Models\Organization;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;
use Illuminate\Support\Str;

class AuthService
{
    public function register(
        string $name,
        string $email,
        string $password,
        string $organizationName,
    ): array {
        return DB::transaction(function () use ($name, $email, $password, $organizationName): array {
            // สร้าง user
            $user = User::create([
                'name' => $name,
                'email' => $email,
                'password' => Hash::make($password),
            ]);

            // สร้าง organization
            $org = Organization::create([
                'name' => $organizationName,
                'slug' => $this->generateSlug($organizationName),
                'plan' => 'free',
                'trial_ends_at' => now()->addDays(14),
            ]);

            // เพิ่ม user เป็น owner
            $org->members()->attach($user->id, [
                'role' => 'owner',
                'joined_at' => now(),
            ]);

            // ส่ง email verification
            $user->sendEmailVerificationNotification();

            $token = $user->createToken('default', ['*'], now()->addDays(30));

            return [
                'token' => $token->plainTextToken,
                'user' => $user,
                'organization' => $org,
            ];
        });
    }

    public function login(string $email, string $password, string $deviceName): ?array
    {
        $user = User::where('email', $email)->first();

        if (!$user || !Hash::check($password, $user->password)) {
            return null;
        }

        // ลบ tokens เก่าของ device นี้
        $user->tokens()->where('name', $deviceName)->delete();

        $token = $user->createToken($deviceName, ['*'], now()->addDays(30));

        $organizations = $user->organizations()->with(['pivot'])->get()->map(fn ($org) => [
            'id' => $org->id,
            'name' => $org->name,
            'slug' => $org->slug,
            'plan' => $org->plan,
            'logo_url' => $org->logo_url,
            'role' => $org->pivot->role,
        ]);

        return [
            'token' => $token->plainTextToken,
            'user' => $user,
            'organizations' => $organizations,
        ];
    }

    private function generateSlug(string $name): string
    {
        $slug = Str::slug($name);
        $count = Organization::where('slug', 'like', "{$slug}%")->count();
        return $count > 0 ? "{$slug}-{$count}" : $slug;
    }
}
```

---

## ขั้นตอนที่ 2833: Project Management

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\V1;

use App\Http\Requests\Project\StoreProjectRequest;
use App\Http\Requests\Project\UpdateProjectRequest;
use App\Http\Resources\ProjectResource;
use App\Models\Organization;
use App\Models\Project;
use App\Services\ProjectService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\ResourceCollection;

class ProjectController extends \App\Http\Controllers\Controller
{
    public function __construct(
        private readonly ProjectService $projectService
    ) {}

    public function index(Request $request, Organization $org): ResourceCollection
    {
        $projects = Project::where('organization_id', $org->id)
            ->when($request->status, fn ($q) => $q->where('status', $request->status))
            ->when($request->search, fn ($q) => $q->where('name', 'like', "%{$request->search}%"))
            ->withCount(['tasks', 'tasks as completed_tasks_count' => fn ($q) => $q->where('status', 'done')])
            ->latest()
            ->paginate(20);

        return ProjectResource::collection($projects);
    }

    public function store(StoreProjectRequest $request, Organization $org): JsonResponse
    {
        $this->authorize('create', [Project::class, $org]);

        // ตรวจสอบ plan limit
        $currentCount = Project::where('organization_id', $org->id)->count();
        if (!app(FeatureFlagService::class)->isWithinLimit($org, 'projects', $currentCount)) {
            return response()->json([
                'message' => 'คุณถึงขีดจำกัดจำนวน projects ของแผน กรุณาอัพเกรด',
                'code' => 'PLAN_LIMIT_EXCEEDED',
                'upgrade_url' => route('billing.upgrade', $org),
            ], 422);
        }

        $project = $this->projectService->create($org, $request->user(), $request->validated());

        return response()->json(new ProjectResource($project), 201);
    }

    public function update(UpdateProjectRequest $request, Organization $org, Project $project): JsonResponse
    {
        $this->authorize('update', $project);

        $project = $this->projectService->update($project, $request->validated());

        return response()->json(new ProjectResource($project));
    }

    public function destroy(Organization $org, Project $project): JsonResponse
    {
        $this->authorize('delete', $project);

        $project->delete();

        return response()->json(null, 204);
    }
}
```

---

## ขั้นตอนที่ 2834: Stripe Subscription Service

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Models\Organization;
use App\Models\Plan;
use App\Models\Subscription;
use Stripe\StripeClient;

class SubscriptionService
{
    private StripeClient $stripe;

    public function __construct()
    {
        $this->stripe = new StripeClient(config('services.stripe.secret'));
    }

    public function createCheckoutSession(
        Organization $org,
        Plan $plan,
        string $billingCycle,
        string $successUrl,
        string $cancelUrl,
    ): string {
        $priceId = $billingCycle === 'yearly'
            ? $plan->stripe_yearly_price_id
            : $plan->stripe_monthly_price_id;

        // สร้าง/ดึง Stripe customer
        $customerId = $this->getOrCreateCustomer($org);

        $session = $this->stripe->checkout->sessions->create([
            'customer' => $customerId,
            'mode' => 'subscription',
            'line_items' => [[
                'price' => $priceId,
                'quantity' => 1,
            ]],
            'allow_promotion_codes' => true,
            'success_url' => $successUrl . '?session_id={CHECKOUT_SESSION_ID}',
            'cancel_url' => $cancelUrl,
            'metadata' => [
                'organization_id' => $org->id,
                'plan_id' => $plan->id,
                'billing_cycle' => $billingCycle,
            ],
            'subscription_data' => [
                'metadata' => [
                    'organization_id' => $org->id,
                ],
                'trial_period_days' => $org->trial_ends_at && $org->trial_ends_at->isFuture()
                    ? (int)now()->diffInDays($org->trial_ends_at)
                    : null,
            ],
        ]);

        return $session->url;
    }

    public function handleWebhook(array $payload, string $signature): void
    {
        $event = \Stripe\Webhook::constructEvent(
            json_encode($payload),
            $signature,
            config('services.stripe.webhook_secret')
        );

        match ($event->type) {
            'customer.subscription.created' => $this->handleSubscriptionCreated($event->data->object),
            'customer.subscription.updated' => $this->handleSubscriptionUpdated($event->data->object),
            'customer.subscription.deleted' => $this->handleSubscriptionDeleted($event->data->object),
            'invoice.payment_succeeded' => $this->handleInvoicePaid($event->data->object),
            'invoice.payment_failed' => $this->handleInvoicePaymentFailed($event->data->object),
            default => null,
        };
    }

    private function handleSubscriptionCreated(\Stripe\Subscription $stripeSub): void
    {
        $orgId = $stripeSub->metadata->organization_id;
        $planId = $stripeSub->metadata->plan_id ?? null;

        $org = Organization::findOrFail($orgId);
        $plan = $planId ? Plan::find($planId) : Plan::where('stripe_monthly_price_id', $stripeSub->items->data[0]->price->id)->first();

        Subscription::updateOrCreate(
            ['stripe_subscription_id' => $stripeSub->id],
            [
                'organization_id' => $org->id,
                'plan_id' => $plan->id,
                'status' => $stripeSub->status,
                'stripe_customer_id' => $stripeSub->customer,
                'current_period_start' => \Carbon\Carbon::createFromTimestamp($stripeSub->current_period_start),
                'current_period_end' => \Carbon\Carbon::createFromTimestamp($stripeSub->current_period_end),
            ]
        );

        $org->update(['plan' => $plan->slug]);
    }

    private function handleSubscriptionDeleted(\Stripe\Subscription $stripeSub): void
    {
        Subscription::where('stripe_subscription_id', $stripeSub->id)
            ->update(['status' => 'canceled', 'canceled_at' => now()]);

        $orgId = $stripeSub->metadata->organization_id ?? null;
        if ($orgId) {
            Organization::find($orgId)?->update(['plan' => 'free']);
        }
    }

    private function getOrCreateCustomer(Organization $org): string
    {
        $existing = Subscription::where('organization_id', $org->id)
            ->whereNotNull('stripe_customer_id')
            ->value('stripe_customer_id');

        if ($existing) return $existing;

        $customer = $this->stripe->customers->create([
            'name' => $org->name,
            'metadata' => ['organization_id' => $org->id],
        ]);

        return $customer->id;
    }

    private function handleSubscriptionUpdated(\Stripe\Subscription $sub): void
    {
        Subscription::where('stripe_subscription_id', $sub->id)->update([
            'status' => $sub->status,
            'current_period_end' => \Carbon\Carbon::createFromTimestamp($sub->current_period_end),
        ]);
    }

    private function handleInvoicePaid(\Stripe\Invoice $invoice): void
    {
        \App\Models\Invoice::updateOrCreate(
            ['stripe_invoice_id' => $invoice->id],
            [
                'status' => 'paid',
                'paid_at' => now(),
                'pdf_url' => $invoice->invoice_pdf,
            ]
        );
    }

    private function handleInvoicePaymentFailed(\Stripe\Invoice $invoice): void
    {
        // แจ้งเตือน organization owner
        $orgId = $invoice->subscription_details->metadata->organization_id ?? null;
        if (!$orgId) return;

        $org = Organization::find($orgId);
        $owner = $org?->members()->wherePivot('role', 'owner')->first();

        if ($owner) {
            $owner->notify(new \App\Notifications\PaymentFailedNotification($invoice));
        }
    }
}
```

---

## ขั้นตอนที่ 2835: Team Management

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\V1;

use App\Http\Resources\MemberResource;
use App\Models\Organization;
use App\Notifications\TeamInvitation;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class MemberController extends \App\Http\Controllers\Controller
{
    public function index(Organization $org): JsonResponse
    {
        $members = $org->members()
            ->with(['user'])
            ->orderBy('organization_members.role')
            ->orderBy('users.name')
            ->get()
            ->map(fn ($user) => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'avatar_url' => $user->avatar_url,
                'role' => $user->pivot->role,
                'joined_at' => $user->pivot->joined_at,
                'is_active' => $user->pivot->is_active,
            ]);

        return response()->json(['data' => $members]);
    }

    public function invite(Request $request, Organization $org): JsonResponse
    {
        $request->validate([
            'email' => ['required', 'email'],
            'role' => ['required', 'in:admin,member,viewer'],
        ]);

        // ตรวจสอบ plan limit
        $currentCount = $org->members()->where('is_active', true)->count();
        if (!app(FeatureFlagService::class)->isWithinLimit($org, 'members', $currentCount)) {
            return response()->json([
                'message' => 'ถึงขีดจำกัดสมาชิกของแผน',
                'code' => 'PLAN_LIMIT_EXCEEDED',
            ], 422);
        }

        // ตรวจสอบว่า email นี้อยู่ใน org แล้วหรือไม่
        $existingUser = \App\Models\User::where('email', $request->email)->first();

        if ($existingUser && $org->members()->where('user_id', $existingUser->id)->exists()) {
            return response()->json(['message' => 'ผู้ใช้นี้เป็นสมาชิกอยู่แล้ว'], 422);
        }

        // สร้าง invitation token
        $token = \Str::random(64);
        \App\Models\OrganizationInvitation::create([
            'organization_id' => $org->id,
            'email' => $request->email,
            'role' => $request->role,
            'token' => hash('sha256', $token),
            'invited_by' => $request->user()->id,
            'expires_at' => now()->addDays(7),
        ]);

        // ส่ง email
        \Notification::route('mail', $request->email)
            ->notify(new TeamInvitation($org, $request->role, $token));

        return response()->json(['message' => 'ส่งคำเชิญแล้ว']);
    }

    public function updateRole(Request $request, Organization $org, int $userId): JsonResponse
    {
        $request->validate(['role' => ['required', 'in:admin,member,viewer']]);

        // ป้องกันการเปลี่ยน role ของ owner
        $member = $org->members()->where('user_id', $userId)->first();
        if ($member?->pivot->role === 'owner') {
            return response()->json(['message' => 'ไม่สามารถเปลี่ยน role ของ owner ได้'], 403);
        }

        $org->members()->updateExistingPivot($userId, ['role' => $request->role]);

        return response()->json(['message' => 'อัพเดท role แล้ว']);
    }

    public function remove(Organization $org, int $userId): JsonResponse
    {
        // ตรวจสอบว่าไม่ใช่ owner
        $member = $org->members()->where('user_id', $userId)->first();
        if ($member?->pivot->role === 'owner') {
            return response()->json(['message' => 'ไม่สามารถลบ owner ออกได้'], 403);
        }

        $org->members()->detach($userId);

        return response()->json(null, 204);
    }
}
```

---

## ขั้นตอนที่ 2836: Real-time Events

```php
<?php
declare(strict_types=1);

namespace App\Events;

use App\Models\Task;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class TaskUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Task $task,
        public readonly string $action, // created, updated, deleted
        public readonly int $updatedBy,
    ) {}

    public function broadcastOn(): array
    {
        return [
            new PresenceChannel("org.{$this->task->organization_id}.project.{$this->task->project_id}"),
        ];
    }

    public function broadcastAs(): string
    {
        return 'task.' . $this->action;
    }

    public function broadcastWith(): array
    {
        return [
            'task' => [
                'id' => $this->task->id,
                'title' => $this->task->title,
                'status' => $this->task->status,
                'priority' => $this->task->priority,
                'assignee_id' => $this->task->assignee_id,
                'updated_at' => $this->task->updated_at->toISOString(),
            ],
            'updated_by' => $this->updatedBy,
            'action' => $this->action,
        ];
    }
}
```

---

## สรุปบทที่ 99

| หัวข้อ | Implementation | เทคโนโลยี |
|--------|---------------|---------|
| Auth | Register/Login/Token | Sanctum |
| Multi-tenant | Organization scope | Middleware |
| Projects/Tasks | CRUD + limits | Eloquent |
| Subscriptions | Stripe + webhooks | Stripe SDK |
| Team Management | Invite + roles | Notifications |
| Real-time | Broadcasting | Laravel Reverb |

ถัดไป → Part 100: Final Project Complete
