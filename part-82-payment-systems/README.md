# Part 82: Payment Systems Integration
## ขั้นตอนที่ 2321-2350: ระบบชำระเงินครบวงจรสำหรับไทย

การรวม Payment Gateway ต่าง ๆ ที่ใช้งานในประเทศไทย
ครอบคลุม PromptPay, KBank, SCB และการจัดการ refund

---

## ขั้นตอนที่ 2321: Payment Architecture

```php
<?php
declare(strict_types=1);

// Payment Gateway Interface
namespace App\Contracts;

interface PaymentGateway
{
    public function charge(array $data): PaymentResult;
    public function refund(string $transactionId, int $amount): RefundResult;
    public function verify(string $transactionId): VerificationResult;
    public function getStatus(string $transactionId): string;
}

// Payment Result Value Objects
namespace App\ValueObjects;

readonly class PaymentResult
{
    public function __construct(
        public bool $success,
        public string $transactionId,
        public int $amount,
        public string $currency,
        public string $status,
        public ?string $redirectUrl = null,
        public ?string $qrCode = null,
        public ?string $errorMessage = null,
        public array $metadata = [],
    ) {}
}

readonly class RefundResult
{
    public function __construct(
        public bool $success,
        public string $refundId,
        public int $amount,
        public string $status,
        public ?string $errorMessage = null,
    ) {}
}
```

---

## ขั้นตอนที่ 2322: Payment Model และ Migration

```php
<?php
declare(strict_types=1);

// database/migrations/xxxx_create_payments_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('payments', function (Blueprint $table) {
            $table->id();
            $table->string('reference')->unique(); // รหัสอ้างอิงของเรา
            $table->foreignId('user_id')->constrained();
            $table->nullableMorphs('payable'); // Order, Subscription, etc.
            $table->string('gateway'); // promptpay, kbank, scb, stripe
            $table->string('gateway_transaction_id')->nullable()->index();
            $table->unsignedBigInteger('amount'); // สตางค์ (100 = 1 บาท)
            $table->string('currency', 3)->default('THB');
            $table->string('status')->default('pending');
            // pending, processing, completed, failed, refunded, cancelled
            $table->string('payment_method')->nullable(); // qr, card, banking
            $table->text('description')->nullable();
            $table->json('metadata')->nullable();
            $table->json('gateway_response')->nullable();
            $table->timestamp('paid_at')->nullable();
            $table->timestamp('failed_at')->nullable();
            $table->timestamp('refunded_at')->nullable();
            $table->timestamps();

            $table->index(['status', 'gateway']);
            $table->index('paid_at');
        });

        Schema::create('payment_refunds', function (Blueprint $table) {
            $table->id();
            $table->foreignId('payment_id')->constrained()->cascadeOnDelete();
            $table->string('refund_reference')->unique();
            $table->string('gateway_refund_id')->nullable();
            $table->unsignedBigInteger('amount');
            $table->string('reason');
            $table->string('status')->default('pending');
            $table->json('gateway_response')->nullable();
            $table->foreignId('initiated_by')->constrained('users');
            $table->timestamp('refunded_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('payment_refunds');
        Schema::dropIfExists('payments');
    }
};
```

---

## ขั้นตอนที่ 2323: PromptPay QR Code

```php
<?php
declare(strict_types=1);

namespace App\Services\Payment;

use App\Contracts\PaymentGateway;
use App\Models\Payment;
use App\ValueObjects\PaymentResult;
use App\ValueObjects\RefundResult;
use App\ValueObjects\VerificationResult;
use Illuminate\Support\Str;

class PromptPayGateway implements PaymentGateway
{
    // PromptPay EMV QR Code Standard
    private const EMV_MERCHANT_ACCOUNT = '29';
    private const BOT_AID = 'A000000677010111';

    public function charge(array $data): PaymentResult
    {
        $reference = $data['reference'] ?? Str::random(20);
        $amount = $data['amount']; // in satang
        $phoneOrId = config('services.promptpay.id'); // เบอร์โทรหรือเลขบัตร

        $qrString = $this->generateQRString(
            id: $phoneOrId,
            amount: $amount / 100, // convert to THB
            reference: $reference
        );

        return new PaymentResult(
            success: true,
            transactionId: $reference,
            amount: $amount,
            currency: 'THB',
            status: 'pending',
            qrCode: $qrString,
        );
    }

    private function generateQRString(string $id, float $amount, string $reference): string
    {
        // Normalize PromptPay ID
        $normalizedId = $this->normalizeId($id);

        // Build EMV QR Code fields
        $merchantAccount = $this->buildTLV('00', '01') // Version
            . $this->buildTLV('01', $this->BOT_AID)
            . $this->buildTLV('02', $normalizedId);

        $payload = $this->buildTLV('00', '01') // Payload Format Indicator
            . $this->buildTLV('01', '12')      // Point of Initiation: Dynamic
            . $this->buildTLV('29', $merchantAccount) // Merchant Account
            . $this->buildTLV('53', '764')     // Transaction Currency (THB)
            . $this->buildTLV('54', number_format($amount, 2, '.', '')) // Amount
            . $this->buildTLV('58', 'TH')      // Country Code
            . $this->buildTLV('62', $this->buildTLV('05', $reference)); // Additional Data

        // Add CRC
        $payload .= '6304';
        $crc = $this->crc16($payload);
        $payload .= strtoupper(str_pad(dechex($crc), 4, '0', STR_PAD_LEFT));

        return $payload;
    }

    private function normalizeId(string $id): string
    {
        // เบอร์โทร 0812345678 -> 0066812345678
        if (preg_match('/^0[0-9]{9}$/', $id)) {
            return '0066' . substr($id, 1);
        }
        // เลขบัตรประชาชน/เลขบริษัท
        return $id;
    }

    private function buildTLV(string $tag, string $value): string
    {
        return $tag . str_pad(strlen($value), 2, '0', STR_PAD_LEFT) . $value;
    }

    private function crc16(string $data): int
    {
        $crc = 0xFFFF;
        for ($i = 0; $i < strlen($data); $i++) {
            $crc ^= ord($data[$i]) << 8;
            for ($j = 0; $j < 8; $j++) {
                if ($crc & 0x8000) {
                    $crc = ($crc << 1) ^ 0x1021;
                } else {
                    $crc <<= 1;
                }
                $crc &= 0xFFFF;
            }
        }
        return $crc;
    }

    public function refund(string $transactionId, int $amount): RefundResult
    {
        // PromptPay ไม่รองรับ auto refund ต้องทำ manual
        return new RefundResult(
            success: false,
            refundId: '',
            amount: $amount,
            status: 'manual_required',
            errorMessage: 'PromptPay requires manual refund processing',
        );
    }

    public function verify(string $transactionId): VerificationResult
    {
        // Verify ผ่าน webhook หรือ polling
        $payment = Payment::where('reference', $transactionId)->first();

        return new VerificationResult(
            verified: $payment?->status === 'completed',
            status: $payment?->status ?? 'unknown',
        );
    }

    public function getStatus(string $transactionId): string
    {
        return Payment::where('reference', $transactionId)->value('status') ?? 'unknown';
    }
}
```

---

## ขั้นตอนที่ 2324: KBank Payment

```php
<?php
declare(strict_types=1);

namespace App\Services\Payment;

use App\Contracts\PaymentGateway;
use App\ValueObjects\PaymentResult;
use App\ValueObjects\RefundResult;
use App\ValueObjects\VerificationResult;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Str;

class KBankGateway implements PaymentGateway
{
    private string $apiKey;
    private string $secretKey;
    private string $merchantId;
    private string $baseUrl;

    public function __construct()
    {
        $this->apiKey = config('services.kbank.api_key');
        $this->secretKey = config('services.kbank.secret_key');
        $this->merchantId = config('services.kbank.merchant_id');
        $this->baseUrl = config('services.kbank.sandbox')
            ? 'https://apigateway.kasikornbank.com/v2'
            : 'https://apigateway.kasikornbank.com/v2';
    }

    public function charge(array $data): PaymentResult
    {
        $reference = $data['reference'] ?? Str::uuid()->toString();
        $token = $data['token']; // Token จาก KBank Payment.js

        $response = Http::withHeaders([
            'x-api-key' => $this->apiKey,
            'Content-Type' => 'application/json',
        ])->post("{$this->baseUrl}/payments/tokens/charge", [
            'amount' => $data['amount'],
            'currency' => 'THB',
            'description' => $data['description'] ?? 'Payment',
            'sourceOfFund' => 'EP',
            'paymentToken' => $token,
            'mid' => $this->merchantId,
            'orderID' => $reference,
        ]);

        if ($response->successful() && $response->json('status.code') === '00') {
            return new PaymentResult(
                success: true,
                transactionId: $response->json('tranID'),
                amount: $data['amount'],
                currency: 'THB',
                status: 'completed',
                metadata: [
                    'reference' => $reference,
                    'approval_code' => $response->json('approvalCode'),
                    'card_mask' => $response->json('cardInfo.cardNumber'),
                ],
            );
        }

        return new PaymentResult(
            success: false,
            transactionId: $reference,
            amount: $data['amount'],
            currency: 'THB',
            status: 'failed',
            errorMessage: $response->json('status.description', 'Payment failed'),
        );
    }

    public function refund(string $transactionId, int $amount): RefundResult
    {
        $refundRef = 'REF-' . Str::random(10);

        $response = Http::withHeaders([
            'x-api-key' => $this->apiKey,
            'Content-Type' => 'application/json',
        ])->post("{$this->baseUrl}/payments/{$transactionId}/refunds", [
            'amount' => $amount,
            'currency' => 'THB',
            'referenceOrder' => $refundRef,
        ]);

        if ($response->successful()) {
            return new RefundResult(
                success: true,
                refundId: $response->json('refundTranID', $refundRef),
                amount: $amount,
                status: 'completed',
            );
        }

        return new RefundResult(
            success: false,
            refundId: $refundRef,
            amount: $amount,
            status: 'failed',
            errorMessage: $response->json('message', 'Refund failed'),
        );
    }

    public function verify(string $transactionId): VerificationResult
    {
        $response = Http::withHeaders([
            'x-api-key' => $this->apiKey,
        ])->get("{$this->baseUrl}/payments/{$transactionId}");

        return new VerificationResult(
            verified: $response->json('status.code') === '00',
            status: $response->json('tranStatus', 'unknown'),
        );
    }

    public function getStatus(string $transactionId): string
    {
        $result = $this->verify($transactionId);
        return $result->status;
    }
}
```

---

## ขั้นตอนที่ 2325: Payment Service

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Contracts\PaymentGateway;
use App\Events\PaymentCompleted;
use App\Events\PaymentFailed;
use App\Models\Payment;
use App\Services\Payment\KBankGateway;
use App\Services\Payment\PromptPayGateway;
use App\Services\Payment\SCBGateway;
use Illuminate\Support\Str;

class PaymentService
{
    private array $gateways = [];

    public function __construct()
    {
        $this->gateways = [
            'promptpay' => new PromptPayGateway(),
            'kbank' => new KBankGateway(),
            'scb' => new SCBGateway(),
        ];
    }

    public function gateway(string $name): PaymentGateway
    {
        return $this->gateways[$name] ?? throw new \InvalidArgumentException("Gateway {$name} not supported");
    }

    public function createPayment(array $data): Payment
    {
        return Payment::create([
            'reference' => 'PAY-' . strtoupper(Str::random(12)),
            'user_id' => $data['user_id'],
            'payable_type' => $data['payable_type'] ?? null,
            'payable_id' => $data['payable_id'] ?? null,
            'gateway' => $data['gateway'],
            'amount' => $data['amount'],
            'currency' => $data['currency'] ?? 'THB',
            'description' => $data['description'] ?? null,
            'metadata' => $data['metadata'] ?? null,
        ]);
    }

    public function processPayment(Payment $payment, array $gatewayData = []): Payment
    {
        $gateway = $this->gateway($payment->gateway);

        $result = $gateway->charge(array_merge($gatewayData, [
            'reference' => $payment->reference,
            'amount' => $payment->amount,
            'currency' => $payment->currency,
            'description' => $payment->description,
        ]));

        if ($result->success) {
            $payment->update([
                'status' => $result->status,
                'gateway_transaction_id' => $result->transactionId,
                'gateway_response' => $result->metadata,
                'paid_at' => $result->status === 'completed' ? now() : null,
            ]);

            if ($result->status === 'completed') {
                event(new PaymentCompleted($payment));
            }
        } else {
            $payment->update([
                'status' => 'failed',
                'failed_at' => now(),
                'gateway_response' => ['error' => $result->errorMessage],
            ]);

            event(new PaymentFailed($payment));
        }

        return $payment->fresh();
    }

    public function refund(Payment $payment, int $amount, string $reason, int $initiatedBy): bool
    {
        if ($payment->status !== 'completed') {
            throw new \RuntimeException('ไม่สามารถคืนเงินได้ เนื่องจากการชำระเงินยังไม่สำเร็จ');
        }

        $gateway = $this->gateway($payment->gateway);
        $result = $gateway->refund($payment->gateway_transaction_id, $amount);

        $refund = $payment->refunds()->create([
            'refund_reference' => 'REF-' . strtoupper(Str::random(10)),
            'gateway_refund_id' => $result->refundId,
            'amount' => $amount,
            'reason' => $reason,
            'status' => $result->success ? 'completed' : 'failed',
            'initiated_by' => $initiatedBy,
            'refunded_at' => $result->success ? now() : null,
        ]);

        if ($result->success) {
            $totalRefunded = $payment->refunds()->where('status', 'completed')->sum('amount');
            if ($totalRefunded >= $payment->amount) {
                $payment->update(['status' => 'refunded', 'refunded_at' => now()]);
            }
        }

        return $result->success;
    }
}
```

---

## ขั้นตอนที่ 2326: Webhook Handler

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers;

use App\Models\Payment;
use App\Services\PaymentService;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Illuminate\Support\Facades\Log;

class PaymentWebhookController extends \App\Http\Controllers\Controller
{
    public function __construct(
        private readonly PaymentService $paymentService
    ) {}

    public function kbank(Request $request): Response
    {
        // Verify webhook signature
        $signature = $request->header('X-KBank-Signature');
        if (! $this->verifyKBankSignature($request->getContent(), $signature)) {
            Log::warning('Invalid KBank webhook signature', ['ip' => $request->ip()]);
            return response('Unauthorized', 401);
        }

        $payload = $request->json()->all();
        $transactionId = $payload['tranID'] ?? null;
        $status = $payload['tranStatus'] ?? null;

        if ($transactionId && $status === 'Approved') {
            $payment = Payment::where('gateway_transaction_id', $transactionId)->first();

            if ($payment && $payment->status === 'pending') {
                $payment->update([
                    'status' => 'completed',
                    'paid_at' => now(),
                    'gateway_response' => $payload,
                ]);

                event(new \App\Events\PaymentCompleted($payment));
            }
        }

        return response('OK', 200);
    }

    public function scb(Request $request): Response
    {
        $payload = $request->json()->all();

        Log::info('SCB Webhook received', $payload);

        $transRef = $payload['transRef'] ?? null;
        $amount = $payload['amount'] ?? 0;

        if ($transRef) {
            $payment = Payment::where('reference', $transRef)->first();

            if ($payment && $payment->status === 'pending') {
                if ((int)($amount * 100) === $payment->amount) {
                    $payment->update([
                        'status' => 'completed',
                        'gateway_transaction_id' => $payload['txnId'] ?? null,
                        'paid_at' => now(),
                        'gateway_response' => $payload,
                    ]);

                    event(new \App\Events\PaymentCompleted($payment));
                }
            }
        }

        return response('OK', 200);
    }

    private function verifyKBankSignature(string $payload, ?string $signature): bool
    {
        if (! $signature) return false;

        $expectedSignature = hash_hmac(
            'sha256',
            $payload,
            config('services.kbank.webhook_secret')
        );

        return hash_equals($expectedSignature, $signature);
    }
}
```

---

## สรุปบทที่ 82

| Gateway | วิธีชำระ | Auto Refund |
|---------|---------|------------|
| PromptPay | QR Code | ไม่รองรับ |
| KBank | Card/QR | รองรับ |
| SCB Easy Pay | QR/Banking | รองรับ |
| Stripe | International | รองรับ |
| 2C2P | Multi-method | รองรับ |
| Omise | Card/Promptpay | รองรับ |

ถัดไป → Part 83: Laravel Localization
