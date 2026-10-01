# Part 60: Laravel Cashier กับ Stripe

## ขั้นตอนที่ 1661-1690: การจัดการ Subscription และ Payment

Laravel Cashier ให้ interface ที่ expressive สำหรับ Stripe's subscription billing services

---

## ขั้นตอนที่ 1661: ติดตั้ง Laravel Cashier

```bash
# ติดตั้ง Cashier
composer require laravel/cashier

# Publish migration
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate

# Publish config
php artisan vendor:publish --tag="cashier-config"
```

```php
<?php

declare(strict_types=1);

// app/Models/User.php
namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Cashier\Billable;

class User extends Authenticatable
{
    use Billable;

    protected $fillable = ['name', 'email', 'password'];

    protected $casts = [
        'email_verified_at' => 'datetime',
        'trial_ends_at'     => 'datetime',
    ];
}
```

```php
<?php

declare(strict_types=1);

// .env configuration
// STRIPE_KEY=pk_test_xxxx
// STRIPE_SECRET=sk_test_xxxx
// STRIPE_WEBHOOK_SECRET=whsec_xxxx
// CASHIER_CURRENCY=thb
// CASHIER_CURRENCY_LOCALE=th_TH
```

---

## ขั้นตอนที่ 1662: สร้าง Stripe Products และ Prices

```php
<?php

declare(strict_types=1);

// app/Console/Commands/SetupStripeProducts.php
namespace App\Console\Commands;

use Illuminate\Console\Command;
use Stripe\Stripe;
use Stripe\Product;
use Stripe\Price;

class SetupStripeProducts extends Command
{
    protected $signature = 'stripe:setup';
    protected $description = 'Create Stripe products and prices';

    public function handle(): int
    {
        Stripe::setApiKey(config('cashier.secret'));

        // สร้าง Starter Plan
        $starter = Product::create([
            'name'        => 'Starter Plan',
            'description' => 'Perfect for small businesses',
        ]);

        Price::create([
            'product'     => $starter->id,
            'unit_amount' => 29900, // ฿299 in satangs
            'currency'    => 'thb',
            'recurring'   => ['interval' => 'month'],
            'lookup_key'  => 'starter_monthly',
        ]);

        Price::create([
            'product'     => $starter->id,
            'unit_amount' => 299000, // ฿2990/year
            'currency'    => 'thb',
            'recurring'   => ['interval' => 'year'],
            'lookup_key'  => 'starter_yearly',
        ]);

        // สร้าง Pro Plan
        $pro = Product::create([
            'name'        => 'Pro Plan',
            'description' => 'For growing teams',
        ]);

        Price::create([
            'product'     => $pro->id,
            'unit_amount' => 99900,
            'currency'    => 'thb',
            'recurring'   => ['interval' => 'month'],
            'lookup_key'  => 'pro_monthly',
        ]);

        $this->info('Stripe products and prices created!');
        return self::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1663: สร้าง Subscription

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/SubscriptionController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Laravel\Cashier\Exceptions\IncompletePayment;

class SubscriptionController extends Controller
{
    public function createCheckoutSession(Request $request): JsonResponse
    {
        $request->validate([
            'price_id' => 'required|string',
            'trial'    => 'nullable|boolean',
        ]);

        /** @var User $user */
        $user = $request->user();

        $checkoutBuilder = $user->newSubscription('default', $request->price_id)
            ->checkout([
                'success_url' => route('subscription.success') . '?session_id={CHECKOUT_SESSION_ID}',
                'cancel_url'  => route('subscription.cancel'),
                'metadata'    => [
                    'user_id'  => $user->id,
                    'price_id' => $request->price_id,
                ],
            ]);

        if ($request->boolean('trial')) {
            $checkoutBuilder->trialDays(14);
        }

        $session = $checkoutBuilder;

        return response()->json(['url' => $session->url]);
    }

    public function subscribe(Request $request): JsonResponse
    {
        $request->validate([
            'payment_method' => 'required|string',
            'price_id'       => 'required|string',
        ]);

        /** @var User $user */
        $user = $request->user();

        try {
            // เพิ่ม payment method ก่อน
            $user->addPaymentMethod($request->payment_method);

            // สร้าง subscription
            $subscription = $user->newSubscription('default', $request->price_id)
                ->trialDays(14)
                ->create($request->payment_method);

            return response()->json([
                'subscription' => $subscription,
                'message'      => 'Subscription created successfully',
            ]);
        } catch (IncompletePayment $e) {
            return response()->json([
                'payment_intent'        => $e->payment->id,
                'redirect_to_payment'   => route('cashier.payment', $e->payment->id),
            ], 402);
        }
    }

    public function cancel(Request $request): JsonResponse
    {
        /** @var User $user */
        $user = $request->user();

        // Cancel at end of billing period (grace period)
        $user->subscription('default')->cancel();

        return response()->json(['message' => 'Subscription will be cancelled at end of billing period']);
    }

    public function cancelNow(Request $request): JsonResponse
    {
        /** @var User $user */
        $user = $request->user();

        // Cancel immediately
        $user->subscription('default')->cancelNow();

        return response()->json(['message' => 'Subscription cancelled immediately']);
    }

    public function resume(Request $request): JsonResponse
    {
        /** @var User $user */
        $user = $request->user();

        if ($user->subscription('default')->onGracePeriod()) {
            $user->subscription('default')->resume();
            return response()->json(['message' => 'Subscription resumed']);
        }

        return response()->json(['message' => 'Subscription cannot be resumed'], 400);
    }

    public function swap(Request $request): JsonResponse
    {
        $request->validate(['price_id' => 'required|string']);

        /** @var User $user */
        $user = $request->user();

        // Swap to different plan (prorate by default)
        $user->subscription('default')->swap($request->price_id);

        return response()->json(['message' => 'Plan changed successfully']);
    }

    public function swapAndInvoice(Request $request): JsonResponse
    {
        $request->validate(['price_id' => 'required|string']);

        /** @var User $user */
        $user = $request->user();

        // Swap และ invoice ทันที
        $user->subscription('default')->swapAndInvoice($request->price_id);

        return response()->json(['message' => 'Plan changed and invoiced']);
    }
}
```

---

## ขั้นตอนที่ 1664: One-Time Charges

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/ChargeController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class ChargeController extends Controller
{
    public function charge(Request $request): JsonResponse
    {
        $request->validate([
            'amount'         => 'required|integer|min:100',
            'payment_method' => 'required|string',
            'description'    => 'nullable|string',
        ]);

        /** @var User $user */
        $user = $request->user();

        try {
            $payment = $user->charge(
                $request->amount, // in satangs
                $request->payment_method,
                [
                    'description' => $request->description ?? 'One-time charge',
                    'metadata'    => [
                        'user_id' => $user->id,
                    ],
                ]
            );

            return response()->json([
                'payment_id' => $payment->id,
                'status'     => $payment->status,
                'amount'     => $payment->amount,
            ]);
        } catch (\Exception $e) {
            return response()->json(['error' => $e->getMessage()], 422);
        }
    }

    public function createInvoiceCharge(Request $request): JsonResponse
    {
        $request->validate([
            'items' => 'required|array',
            'items.*.description' => 'required|string',
            'items.*.amount'      => 'required|integer',
            'items.*.quantity'    => 'nullable|integer',
        ]);

        /** @var User $user */
        $user = $request->user();

        // สร้าง invoice
        $invoice = $user->tab('My Invoice', [
            'metadata' => ['order_id' => $request->order_id],
        ]);

        // เพิ่ม line items
        foreach ($request->items as $item) {
            $user->invoiceFor($item['description'], $item['amount'], [
                'quantity' => $item['quantity'] ?? 1,
            ]);
        }

        // Invoice และ charge ทันที
        $invoice = $user->invoice();

        return response()->json([
            'invoice_id' => $invoice->id,
            'total'      => $invoice->total(),
            'pdf_url'    => route('cashier.invoice', [$user->id, $invoice->id]),
        ]);
    }
}
```

---

## ขั้นตอนที่ 1665: Webhook Handling

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/WebhookController.php
namespace App\Http\Controllers;

use Laravel\Cashier\Http\Controllers\WebhookController as CashierController;
use Stripe\Subscription as StripeSubscription;

class WebhookController extends CashierController
{
    /**
     * Handle subscription created
     */
    protected function handleCustomerSubscriptionCreated(array $payload): void
    {
        $subscription = $payload['data']['object'];
        $customerId   = $subscription['customer'];

        $user = \App\Models\User::where('stripe_id', $customerId)->first();

        if ($user) {
            // Provision user features based on plan
            $priceId = $subscription['items']['data'][0]['price']['id'];
            $this->provisionFeatures($user, $priceId);

            // Send welcome email
            \Mail::to($user)->send(new \App\Mail\SubscriptionWelcome($user));
        }

        parent::handleCustomerSubscriptionCreated($payload);
    }

    /**
     * Handle subscription deleted (cancelled)
     */
    protected function handleCustomerSubscriptionDeleted(array $payload): void
    {
        $subscription = $payload['data']['object'];
        $customerId   = $subscription['customer'];

        $user = \App\Models\User::where('stripe_id', $customerId)->first();

        if ($user) {
            // Revoke user features
            $this->revokeFeatures($user);

            // Send cancellation email
            \Mail::to($user)->send(new \App\Mail\SubscriptionCancelled($user));
        }

        parent::handleCustomerSubscriptionDeleted($payload);
    }

    /**
     * Handle payment failure
     */
    protected function handleInvoicePaymentFailed(array $payload): void
    {
        $invoice    = $payload['data']['object'];
        $customerId = $invoice['customer'];

        $user = \App\Models\User::where('stripe_id', $customerId)->first();

        if ($user) {
            \Mail::to($user)->send(new \App\Mail\PaymentFailed($user, $invoice));

            \Log::warning("Payment failed for user {$user->id}", [
                'invoice_id' => $invoice['id'],
                'amount'     => $invoice['amount_due'],
            ]);
        }

        parent::handleInvoicePaymentFailed($payload);
    }

    protected function handleInvoicePaymentSucceeded(array $payload): void
    {
        $invoice    = $payload['data']['object'];
        $customerId = $invoice['customer'];

        $user = \App\Models\User::where('stripe_id', $customerId)->first();

        if ($user && $invoice['billing_reason'] === 'subscription_cycle') {
            \Mail::to($user)->send(new \App\Mail\InvoicePaid($user));
        }

        parent::handleInvoicePaymentSucceeded($payload);
    }

    private function provisionFeatures(mixed $user, string $priceId): void
    {
        $features = config("plans.{$priceId}.features", []);
        $user->grantFeatures($features);
    }

    private function revokeFeatures(mixed $user): void
    {
        $user->revokeAllFeatures();
    }
}
```

---

## ขั้นตอนที่ 1666: Trial Periods และ Grace Periods

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/CheckSubscription.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckSubscription
{
    public function handle(Request $request, Closure $next, string $plan = 'default'): Response
    {
        $user = $request->user();

        if (!$user) {
            return redirect()->route('login');
        }

        // Check subscription status
        if ($user->subscribed($plan)) {
            return $next($request);
        }

        // Check trial
        if ($user->onTrial()) {
            return $next($request);
        }

        // On grace period - still allow but show warning
        if ($user->subscription($plan)?->onGracePeriod()) {
            session()->flash('subscription_warning',
                'Your subscription has been cancelled. Access ends on ' .
                $user->subscription($plan)->ends_at->format('d M Y')
            );
            return $next($request);
        }

        return redirect()->route('subscription.plans')
            ->with('error', 'Please subscribe to access this feature');
    }
}

// routes/web.php ตัวอย่าง
Route::middleware(['auth', 'subscribed'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/reports', [ReportController::class, 'index']);
});

// Feature-based middleware
Route::middleware(['auth', 'feature:advanced_analytics'])->group(function () {
    Route::get('/analytics', [AnalyticsController::class, 'advanced']);
});
```

---

## ขั้นตอนที่ 1667: Invoices และ Receipts

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/InvoiceController.php
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class InvoiceController extends Controller
{
    public function index(Request $request)
    {
        /** @var User $user */
        $user = $request->user();

        $invoices = $user->invoices(true); // true = include pending

        return view('billing.invoices', compact('invoices'));
    }

    public function download(Request $request, string $invoiceId): Response
    {
        /** @var User $user */
        $user = $request->user();

        return $user->downloadInvoice($invoiceId, [
            'vendor'  => config('app.name'),
            'product' => 'Subscription Plan',
            'street'  => '123 Sukhumvit Road',
            'location' => 'Bangkok, 10110',
            'phone'   => '+66-2-xxx-xxxx',
            'url'     => config('app.url'),
        ], 'invoice-' . $invoiceId . '.pdf');
    }

    public function upcoming(Request $request)
    {
        /** @var User $user */
        $user = $request->user();

        if ($user->subscribed()) {
            $upcomingInvoice = $user->upcomingInvoice();
            return response()->json([
                'amount'       => $upcomingInvoice?->total(),
                'next_payment' => $upcomingInvoice?->date(),
            ]);
        }

        return response()->json(['message' => 'No active subscription']);
    }
}
```

---

## ขั้นตอนที่ 1668: Cashier กับ Multiple Subscriptions

```php
<?php

declare(strict_types=1);

// ผู้ใช้สามารถมีหลาย subscription พร้อมกัน
namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\Request;

class MultiSubscriptionController extends Controller
{
    public function subscribeToAddon(Request $request): \Illuminate\Http\JsonResponse
    {
        $request->validate(['addon' => 'required|string']);

        /** @var User $user */
        $user = $request->user();

        $addonPrice = config("addons.{$request->addon}.price_id");

        // สร้าง subscription แยกสำหรับ addon
        $user->newSubscription("addon_{$request->addon}", $addonPrice)->create(
            $user->defaultPaymentMethod()->id
        );

        return response()->json(['message' => 'Addon activated']);
    }

    public function status(Request $request): \Illuminate\Http\JsonResponse
    {
        /** @var User $user */
        $user = $request->user();

        return response()->json([
            'main_plan'         => $user->subscribed('default'),
            'on_trial'          => $user->onTrial('default'),
            'on_grace_period'   => $user->subscription('default')?->onGracePeriod(),
            'ends_at'           => $user->subscription('default')?->ends_at,
            'trial_ends_at'     => $user->subscription('default')?->trial_ends_at,
            'addons'            => [
                'storage'   => $user->subscribed('addon_storage'),
                'analytics' => $user->subscribed('addon_analytics'),
            ],
            'payment_method'    => $user->defaultPaymentMethod()?->card,
        ]);
    }
}
```

---

## สรุป Part 60

| Feature | รายละเอียด |
|---------|-----------|
| Subscription | สร้าง, ยกเลิก, เปลี่ยนแผน |
| Trial Periods | ทดลองใช้ก่อนเสียเงิน |
| Grace Periods | ยังใช้งานได้หลังยกเลิก |
| One-time Charges | ชำระครั้งเดียว |
| Webhooks | รับ events จาก Stripe |
| Invoices | ดาวน์โหลด PDF |
| Multiple Subscriptions | หลาย subscription พร้อมกัน |

ถัดไป → Part 61: PHP-FPM และ Nginx Configuration
