# Part 54: API Integration

## ขั้นตอนที่ 1481-1510: การเชื่อมต่อ External APIs

บทนี้ครอบคลุม Guzzle HTTP Client, Retry Middleware, Circuit Breaker, Webhook Handling, Stripe Payment และ OAuth Integration

---

## ขั้นตอนที่ 1481: Guzzle HTTP Client ขั้นสูง

```php
<?php

declare(strict_types=1);

namespace App\Http\Clients;

use GuzzleHttp\Client;
use GuzzleHttp\HandlerStack;
use GuzzleHttp\Middleware;
use GuzzleHttp\Psr7\Request;
use GuzzleHttp\Psr7\Response;
use GuzzleHttp\Exception\ConnectException;
use GuzzleHttp\Exception\RequestException;
use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ResponseInterface;

class HttpClientFactory
{
    public function create(array $config = []): Client
    {
        $stack = HandlerStack::create();

        // Add retry middleware
        $stack->push($this->retryMiddleware());

        // Add logging middleware
        $stack->push($this->loggingMiddleware());

        // Add request ID middleware
        $stack->push(Middleware::mapRequest(function (RequestInterface $request) {
            return $request->withHeader('X-Request-ID', bin2hex(random_bytes(8)));
        }));

        return new Client(array_merge([
            'handler'  => $stack,
            'timeout'  => 30,
            'connect_timeout' => 10,
            'http_errors' => true,
            'headers' => [
                'Accept'       => 'application/json',
                'Content-Type' => 'application/json',
                'User-Agent'   => 'MyApp/1.0',
            ],
        ], $config));
    }

    private function retryMiddleware(): callable
    {
        $maxRetries = 3;

        return Middleware::retry(
            // Decide whether to retry
            function (
                int $retries,
                RequestInterface $request,
                ?ResponseInterface $response = null,
                ?\RuntimeException $exception = null
            ) use ($maxRetries): bool {
                if ($retries >= $maxRetries) {
                    return false;
                }

                // Retry on connection error
                if ($exception instanceof ConnectException) {
                    return true;
                }

                // Retry on 429 (rate limit) and 5xx
                if ($response) {
                    $statusCode = $response->getStatusCode();
                    return $statusCode === 429 || $statusCode >= 500;
                }

                return false;
            },
            // Delay between retries
            function (int $retries, ?ResponseInterface $response = null): int {
                // Exponential backoff: 1s, 2s, 4s...
                $delay = 1000 * (2 ** ($retries - 1));

                // ถ้ามี Retry-After header ให้ใช้ค่านั้น
                if ($response && $response->hasHeader('Retry-After')) {
                    $delay = (int)$response->getHeaderLine('Retry-After') * 1000;
                }

                return $delay;
            }
        );
    }

    private function loggingMiddleware(): callable
    {
        return function (callable $handler) {
            return function (RequestInterface $request, array $options) use ($handler) {
                $startTime = microtime(true);

                return $handler($request, $options)->then(
                    function (ResponseInterface $response) use ($request, $startTime) {
                        $duration = (microtime(true) - $startTime) * 1000;

                        \Log::channel('api_calls')->info('API Request', [
                            'method'      => $request->getMethod(),
                            'url'         => (string)$request->getUri(),
                            'status'      => $response->getStatusCode(),
                            'duration_ms' => round($duration, 2),
                        ]);

                        return $response;
                    },
                    function (\Exception $e) use ($request) {
                        \Log::channel('api_calls')->error('API Request Failed', [
                            'method' => $request->getMethod(),
                            'url'    => (string)$request->getUri(),
                            'error'  => $e->getMessage(),
                        ]);

                        throw $e;
                    }
                );
            };
        };
    }
}
```

---

## ขั้นตอนที่ 1482: Circuit Breaker Pattern

```php
<?php

declare(strict_types=1);

namespace App\Patterns;

use Illuminate\Contracts\Cache\Repository as Cache;

class CircuitBreaker
{
    private const STATE_CLOSED   = 'closed';    // Normal operation
    private const STATE_OPEN     = 'open';      // Rejecting requests
    private const STATE_HALF_OPEN = 'half_open'; // Testing recovery

    public function __construct(
        private Cache $cache,
        private string $name,
        private int $failureThreshold = 5,
        private int $timeout          = 60,  // seconds before trying again
        private int $successThreshold = 2    // successes to close circuit
    ) {}

    public function call(callable $callable): mixed
    {
        $state = $this->getState();

        if ($state === self::STATE_OPEN) {
            // Check if timeout has passed
            if (!$this->isTimeoutExpired()) {
                throw new \RuntimeException(
                    "Circuit breaker '{$this->name}' is OPEN. Service unavailable."
                );
            }

            $this->transitionTo(self::STATE_HALF_OPEN);
        }

        try {
            $result = $callable();
            $this->recordSuccess();
            return $result;
        } catch (\Exception $e) {
            $this->recordFailure();
            throw $e;
        }
    }

    private function recordSuccess(): void
    {
        $state = $this->getState();

        if ($state === self::STATE_HALF_OPEN) {
            $successCount = $this->incrementCounter('successes');

            if ($successCount >= $this->successThreshold) {
                $this->transitionTo(self::STATE_CLOSED);
                $this->resetCounters();
            }
        } elseif ($state === self::STATE_CLOSED) {
            $this->resetCounters();
        }
    }

    private function recordFailure(): void
    {
        $failures = $this->incrementCounter('failures');

        if ($failures >= $this->failureThreshold) {
            $this->transitionTo(self::STATE_OPEN);
            $this->cache->put(
                $this->key('opened_at'),
                time(),
                now()->addSeconds($this->timeout * 2)
            );
        }
    }

    private function transitionTo(string $state): void
    {
        $this->cache->put(
            $this->key('state'),
            $state,
            now()->addHour()
        );
    }

    private function getState(): string
    {
        return $this->cache->get($this->key('state'), self::STATE_CLOSED);
    }

    private function isTimeoutExpired(): bool
    {
        $openedAt = $this->cache->get($this->key('opened_at'), 0);
        return (time() - $openedAt) >= $this->timeout;
    }

    private function incrementCounter(string $type): int
    {
        $key = $this->key($type);
        $count = $this->cache->get($key, 0) + 1;
        $this->cache->put($key, $count, now()->addMinutes(5));
        return $count;
    }

    private function resetCounters(): void
    {
        $this->cache->forget($this->key('failures'));
        $this->cache->forget($this->key('successes'));
    }

    private function key(string $suffix): string
    {
        return "circuit_breaker:{$this->name}:{$suffix}";
    }

    public function getStatus(): array
    {
        return [
            'name'     => $this->name,
            'state'    => $this->getState(),
            'failures' => $this->cache->get($this->key('failures'), 0),
        ];
    }
}
```

---

## ขั้นตอนที่ 1483: Webhook Handling

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\Response;

class WebhookController
{
    public function stripe(Request $request): Response
    {
        $payload   = $request->getContent();
        $signature = $request->header('Stripe-Signature');
        $secret    = config('services.stripe.webhook_secret');

        // ตรวจสอบ signature
        try {
            $event = \Stripe\Webhook::constructEvent($payload, $signature, $secret);
        } catch (\Stripe\Exception\SignatureVerificationException $e) {
            \Log::warning('Invalid Stripe webhook signature', [
                'error' => $e->getMessage(),
                'ip'    => $request->ip(),
            ]);

            return response('Invalid signature', 400);
        }

        // บันทึก webhook event
        $webhookEvent = \App\Models\WebhookEvent::create([
            'provider'   => 'stripe',
            'event_type' => $event->type,
            'event_id'   => $event->id,
            'payload'    => json_decode($payload, true),
            'status'     => 'pending',
        ]);

        // Process async ผ่าน queue
        \App\Jobs\ProcessStripeWebhook::dispatch($webhookEvent);

        return response('OK', 200);
    }

    public function github(Request $request): Response
    {
        $signature = $request->header('X-Hub-Signature-256');
        $payload   = $request->getContent();
        $secret    = config('services.github.webhook_secret');

        // Verify signature
        $expectedSignature = 'sha256=' . hash_hmac('sha256', $payload, $secret);

        if (!hash_equals($expectedSignature, $signature ?? '')) {
            return response('Invalid signature', 403);
        }

        $event = $request->header('X-GitHub-Event');

        match ($event) {
            'push'         => $this->handleGithubPush($request),
            'pull_request' => $this->handleGithubPr($request),
            'release'      => $this->handleGithubRelease($request),
            default        => null,
        };

        return response('OK', 200);
    }

    private function handleGithubPush(Request $request): void
    {
        $data   = $request->json()->all();
        $branch = str_replace('refs/heads/', '', $data['ref'] ?? '');

        if ($branch === 'main') {
            \App\Jobs\DeployApplication::dispatch();
        }
    }

    private function handleGithubPr(Request $request): void
    {
        $data = $request->json()->all();

        if ($data['action'] === 'closed' && $data['pull_request']['merged']) {
            \App\Jobs\TriggerCI::dispatch($data['pull_request']['head']['sha']);
        }
    }

    private function handleGithubRelease(Request $request): void
    {
        $data = $request->json()->all();

        if ($data['action'] === 'published') {
            \App\Jobs\NotifySlackRelease::dispatch($data['release']['tag_name']);
        }
    }
}
```

---

## ขั้นตอนที่ 1484: Stripe Payment Integration

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Stripe\Exception\ApiErrorException;
use Stripe\StripeClient;

class StripePaymentService
{
    private StripeClient $stripe;

    public function __construct()
    {
        $this->stripe = new StripeClient(config('services.stripe.secret'));
    }

    public function createPaymentIntent(float $amount, string $currency = 'thb'): array
    {
        try {
            $intent = $this->stripe->paymentIntents->create([
                'amount'              => (int)($amount * 100), // Stripe ใช้ satang
                'currency'            => $currency,
                'automatic_payment_methods' => [
                    'enabled' => true,
                ],
                'metadata'            => [
                    'app'         => config('app.name'),
                    'environment' => config('app.env'),
                ],
            ]);

            return [
                'client_secret' => $intent->client_secret,
                'payment_intent_id' => $intent->id,
            ];
        } catch (ApiErrorException $e) {
            throw new \App\Exceptions\PaymentException(
                "Failed to create payment: {$e->getMessage()}",
                $e->getCode(),
                $e
            );
        }
    }

    public function createSubscription(
        string $customerId,
        string $priceId,
        array $options = []
    ): array {
        try {
            $subscription = $this->stripe->subscriptions->create([
                'customer'       => $customerId,
                'items'          => [['price' => $priceId]],
                'payment_behavior'=> 'default_incomplete',
                'expand'         => ['latest_invoice.payment_intent'],
                ...$options,
            ]);

            return [
                'subscription_id' => $subscription->id,
                'client_secret'   => $subscription->latest_invoice
                    ?->payment_intent
                    ?->client_secret,
                'status'          => $subscription->status,
            ];
        } catch (ApiErrorException $e) {
            throw new \App\Exceptions\PaymentException($e->getMessage());
        }
    }

    public function createCustomer(string $email, string $name): string
    {
        $customer = $this->stripe->customers->create([
            'email'    => $email,
            'name'     => $name,
            'metadata' => ['app' => config('app.name')],
        ]);

        return $customer->id;
    }

    public function refund(string $paymentIntentId, ?int $amountCents = null): array
    {
        $params = ['payment_intent' => $paymentIntentId];

        if ($amountCents !== null) {
            $params['amount'] = $amountCents;
        }

        $refund = $this->stripe->refunds->create($params);

        return [
            'refund_id' => $refund->id,
            'status'    => $refund->status,
            'amount'    => $refund->amount / 100,
        ];
    }
}
```

---

## ขั้นตอนที่ 1485: Third-party OAuth Integration

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers\Auth;

use App\Models\User;
use Illuminate\Http\RedirectResponse;
use Illuminate\Http\Request;
use Laravel\Socialite\Facades\Socialite;

class SocialAuthController
{
    public function redirect(string $provider): RedirectResponse
    {
        $this->validateProvider($provider);

        return Socialite::driver($provider)
            ->scopes($this->getScopesForProvider($provider))
            ->redirect();
    }

    public function callback(Request $request, string $provider): RedirectResponse
    {
        $this->validateProvider($provider);

        try {
            $socialUser = Socialite::driver($provider)->user();
        } catch (\Exception $e) {
            return redirect('/login')->withErrors([
                'social' => 'Authentication failed. Please try again.',
            ]);
        }

        // หา user จาก social profile หรือ email
        $user = User::where("social_{$provider}_id", $socialUser->getId())
            ->orWhere('email', $socialUser->getEmail())
            ->first();

        if (!$user) {
            // สร้าง user ใหม่
            $user = User::create([
                'name'                  => $socialUser->getName(),
                'email'                 => $socialUser->getEmail(),
                'avatar'                => $socialUser->getAvatar(),
                "social_{$provider}_id" => $socialUser->getId(),
                'password'              => bcrypt(bin2hex(random_bytes(16))),
                'email_verified_at'     => now(),
            ]);
        } else {
            // อัพเดท social ID ถ้ายังไม่มี
            $user->update(["social_{$provider}_id" => $socialUser->getId()]);
        }

        auth()->login($user, remember: true);

        return redirect()->intended('/dashboard');
    }

    private function validateProvider(string $provider): void
    {
        if (!in_array($provider, ['google', 'facebook', 'github', 'line'])) {
            abort(404, 'Unknown OAuth provider');
        }
    }

    private function getScopesForProvider(string $provider): array
    {
        return match ($provider) {
            'google'   => ['email', 'profile'],
            'facebook' => ['email', 'public_profile'],
            'github'   => ['user:email'],
            'line'     => ['profile', 'email'],
            default    => [],
        };
    }
}
```

---

## สรุปบทที่ 54

| หัวข้อ | เครื่องมือ | ความสำคัญ |
|--------|-----------|---------|
| Guzzle HTTP | guzzlehttp/guzzle | External API calls |
| Retry Middleware | Guzzle Middleware | Resilient requests |
| Circuit Breaker | Custom + Cache | Prevent cascade failures |
| Webhook Handling | HMAC Verification | Secure callbacks |
| Stripe Payment | stripe-php | Payment processing |
| OAuth Integration | Laravel Socialite | Social login |

**ต่อไป**: Part 55 - Laravel Octane

---
