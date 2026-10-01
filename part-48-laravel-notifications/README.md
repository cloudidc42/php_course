# Part 48: Laravel Notifications

## ขั้นตอนที่ 1301-1330: ระบบ Notifications ใน Laravel

Notifications ช่วยส่งข้อความแจ้งเตือนผ่านช่องทางต่างๆ เช่น Email, SMS, Push, Slack และ Database

---

## ขั้นตอนที่ 1301: Email Notifications ด้วย Mailable

```php
<?php

declare(strict_types=1);

namespace App\Notifications;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class OrderConfirmed extends Notification implements ShouldQueue
{
    use Queueable;

    public string $queue = 'notifications';

    public function __construct(public readonly Order $order) {}

    public function via(object $notifiable): array
    {
        $channels = ['mail', 'database'];

        // เพิ่ม SMS ถ้าเบอร์โทรมี
        if ($notifiable->phone) {
            $channels[] = 'vonage';
        }

        return $channels;
    }

    public function toMail(object $notifiable): MailMessage
    {
        $trackingUrl = route('orders.track', $this->order->tracking_code);

        return (new MailMessage)
            ->subject("Order #{$this->order->id} Confirmed!")
            ->greeting("สวัสดี {$notifiable->name}!")
            ->line('ขอบคุณสำหรับการสั่งซื้อของคุณ')
            ->line("หมายเลขคำสั่งซื้อ: **{$this->order->id}**")
            ->line("ยอดรวม: **" . number_format($this->order->total, 2) . " บาท**")
            ->action('ติดตามสินค้า', $trackingUrl)
            ->line('หากมีคำถาม ติดต่อเราได้ที่ support@shop.com')
            ->salutation('ขอบคุณครับ/ค่ะ, ทีม Shop');
    }

    public function toDatabase(object $notifiable): array
    {
        return [
            'type'       => 'order_confirmed',
            'order_id'   => $this->order->id,
            'message'    => "คำสั่งซื้อ #{$this->order->id} ได้รับการยืนยันแล้ว",
            'action_url' => route('orders.show', $this->order->id),
        ];
    }

    public function toVonage(object $notifiable): \Illuminate\Notifications\Messages\VonageMessage
    {
        return (new \Illuminate\Notifications\Messages\VonageMessage)
            ->content("ยืนยันคำสั่งซื้อ #{$this->order->id} ยอด " . number_format($this->order->total, 0) . " บาท");
    }
}
```

---

## ขั้นตอนที่ 1302: Custom Mailable Classes

```php
<?php

declare(strict_types=1);

namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\Address;
use Illuminate\Mail\Mailables\Attachment;
use Illuminate\Mail\Mailables\Content;
use Illuminate\Mail\Mailables\Envelope;
use Illuminate\Queue\SerializesModels;

class OrderInvoiceMail extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(
        public readonly Order $order,
        public readonly string $pdfPath
    ) {}

    public function envelope(): Envelope
    {
        return new Envelope(
            from: new Address('billing@shop.com', 'Shop Billing'),
            replyTo: [new Address('support@shop.com', 'Support')],
            subject: "Invoice for Order #{$this->order->id}",
            tags: ['invoice', 'order'],
            metadata: ['order_id' => (string)$this->order->id],
        );
    }

    public function content(): Content
    {
        return new Content(
            view: 'emails.invoice',
            text: 'emails.invoice-text',
            with: [
                'orderNumber'   => $this->order->id,
                'customerName'  => $this->order->user->name,
                'total'         => $this->order->total,
                'items'         => $this->order->items,
                'invoiceDate'   => now()->format('d/m/Y'),
            ]
        );
    }

    public function attachments(): array
    {
        return [
            Attachment::fromPath($this->pdfPath)
                ->as("invoice-{$this->order->id}.pdf")
                ->withMime('application/pdf'),
        ];
    }
}
```

---

## ขั้นตอนที่ 1303: SMS ด้วย Twilio

```php
<?php

declare(strict_types=1);

namespace App\Channels;

use Illuminate\Notifications\Notification;
use Twilio\Rest\Client;

class TwilioChannel
{
    public function __construct(private Client $twilio) {}

    public function send(object $notifiable, Notification $notification): void
    {
        if (!method_exists($notification, 'toTwilio')) {
            return;
        }

        $message = $notification->toTwilio($notifiable);
        $phone   = $notifiable->routeNotificationFor('twilio', $notification);

        if (!$phone) {
            return;
        }

        $this->twilio->messages->create($phone, [
            'from' => config('services.twilio.from'),
            'body' => $message->content,
        ]);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Notifications;

use Illuminate\Notifications\Notification;
use App\Channels\TwilioChannel;

class OtpNotification extends Notification
{
    public function __construct(
        private readonly string $otp,
        private readonly int $expiryMinutes = 5
    ) {}

    public function via(object $notifiable): array
    {
        return [TwilioChannel::class];
    }

    public function toTwilio(object $notifiable): object
    {
        return new class($this->otp, $this->expiryMinutes) {
            public string $content;

            public function __construct(string $otp, int $minutes)
            {
                $this->content = "รหัส OTP ของคุณคือ {$otp} หมดอายุใน {$minutes} นาที";
            }
        };
    }
}
```

---

## ขั้นตอนที่ 1304: Slack Notifications

```php
<?php

declare(strict_types=1);

namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Notifications\Messages\SlackMessage;
use Illuminate\Notifications\Notification;

class NewOrderAlert extends Notification
{
    use Queueable;

    public function __construct(
        private readonly \App\Models\Order $order
    ) {}

    public function via(object $notifiable): array
    {
        return ['slack'];
    }

    public function toSlack(object $notifiable): SlackMessage
    {
        return (new SlackMessage)
            ->success()
            ->content("New Order Received! :shopping_bags:")
            ->attachment(function ($attachment) {
                $attachment
                    ->title("Order #{$this->order->id}", route('admin.orders.show', $this->order->id))
                    ->fields([
                        'Customer'  => $this->order->user->name,
                        'Total'     => number_format($this->order->total, 2) . ' THB',
                        'Items'     => $this->order->items->count(),
                        'Payment'   => ucfirst($this->order->payment_method),
                    ])
                    ->footer('Shop Admin')
                    ->timestamp($this->order->created_at);
            });
    }
}
```

---

## ขั้นตอนที่ 1305: Database Notifications

```php
<?php

declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Notifications\Notifiable;

class User extends Model
{
    use Notifiable;

    // Route notification for specific channels
    public function routeNotificationForMail(): string
    {
        return $this->email;
    }

    public function routeNotificationForVonage(): string
    {
        return $this->phone;
    }

    public function routeNotificationForSlack(): string
    {
        return config('services.slack.default_webhook');
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class NotificationController
{
    public function index(Request $request): JsonResponse
    {
        $notifications = $request->user()
            ->notifications()
            ->latest()
            ->paginate(20);

        return response()->json([
            'notifications' => $notifications->map(fn ($n) => [
                'id'          => $n->id,
                'type'        => $n->type,
                'data'        => $n->data,
                'read'        => !is_null($n->read_at),
                'read_at'     => $n->read_at?->toISOString(),
                'created_at'  => $n->created_at->diffForHumans(),
            ]),
            'unread_count'  => $request->user()->unreadNotifications()->count(),
        ]);
    }

    public function markAsRead(Request $request, string $id): JsonResponse
    {
        $notification = $request->user()
            ->notifications()
            ->findOrFail($id);

        $notification->markAsRead();

        return response()->json(['success' => true]);
    }

    public function markAllAsRead(Request $request): JsonResponse
    {
        $request->user()->unreadNotifications()->update(['read_at' => now()]);

        return response()->json(['success' => true]);
    }

    public function destroy(Request $request, string $id): JsonResponse
    {
        $request->user()
            ->notifications()
            ->where('id', $id)
            ->delete();

        return response()->json(['success' => true]);
    }
}
```

---

## ขั้นตอนที่ 1306: Push Notifications

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Google\Client;
use Google\Auth\CredentialsLoader;

class FcmService
{
    private string $projectId;

    public function __construct(private Client $googleClient)
    {
        $this->projectId = config('services.firebase.project_id');
    }

    public function send(string $deviceToken, string $title, string $body, array $data = []): array
    {
        $accessToken = $this->getAccessToken();

        $message = [
            'message' => [
                'token' => $deviceToken,
                'notification' => [
                    'title' => $title,
                    'body'  => $body,
                ],
                'data'    => array_map('strval', $data),
                'android' => [
                    'notification' => [
                        'sound'       => 'default',
                        'click_action'=> 'FLUTTER_NOTIFICATION_CLICK',
                    ],
                ],
                'apns' => [
                    'payload' => [
                        'aps' => [
                            'sound' => 'default',
                            'badge' => 1,
                        ],
                    ],
                ],
            ],
        ];

        $client = new \GuzzleHttp\Client();
        $response = $client->post(
            "https://fcm.googleapis.com/v1/projects/{$this->projectId}/messages:send",
            [
                'headers' => [
                    'Authorization' => "Bearer {$accessToken}",
                    'Content-Type'  => 'application/json',
                ],
                'json' => $message,
            ]
        );

        return json_decode($response->getBody()->getContents(), true);
    }

    public function sendToTopic(string $topic, string $title, string $body, array $data = []): array
    {
        // ส่งไปหลาย devices พร้อมกัน
        return $this->send("/topics/{$topic}", $title, $body, $data);
    }

    private function getAccessToken(): string
    {
        $credentialsPath = config('services.firebase.credentials_path');

        putenv("GOOGLE_APPLICATION_CREDENTIALS={$credentialsPath}");

        $this->googleClient->useApplicationDefaultCredentials();
        $this->googleClient->addScope('https://www.googleapis.com/auth/firebase.messaging');

        $token = $this->googleClient->fetchAccessTokenWithAssertion();
        return $token['access_token'];
    }
}
```

---

## ขั้นตอนที่ 1307: Notification Preferences

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
        Schema::create('notification_preferences', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->cascadeOnDelete();
            $table->string('notification_type');  // e.g., 'order_confirmed'
            $table->boolean('email')->default(true);
            $table->boolean('sms')->default(false);
            $table->boolean('push')->default(true);
            $table->boolean('slack')->default(false);
            $table->timestamps();

            $table->unique(['user_id', 'notification_type']);
        });
    }
};
```

```php
<?php

declare(strict_types=1);

namespace App\Notifications;

use App\Models\NotificationPreference;
use Illuminate\Notifications\Notification;

abstract class PreferenceAwareNotification extends Notification
{
    abstract protected function notificationType(): string;

    public function via(object $notifiable): array
    {
        $prefs = NotificationPreference::firstOrCreate(
            [
                'user_id'           => $notifiable->id,
                'notification_type' => $this->notificationType(),
            ],
            [
                'email' => true,
                'sms'   => false,
                'push'  => true,
            ]
        );

        $channels = [];

        if ($prefs->email) $channels[] = 'mail';
        if ($prefs->sms && $notifiable->phone) $channels[] = 'vonage';
        if ($prefs->push && $notifiable->device_token) $channels[] = \App\Channels\FcmChannel::class;
        if ($prefs->slack && $notifiable->slack_user_id) $channels[] = 'slack';

        // เพิ่ม database เสมอ
        $channels[] = 'database';

        return $channels;
    }
}
```

---

## สรุปบทที่ 48

| ช่องทาง | Package | ประโยชน์ |
|---------|---------|---------|
| Email | laravel/mail | HTML email พร้อม attachment |
| SMS | laravel/vonage, twilio | ข้อความสั้น OTP |
| Push | FCM/APNS | Mobile notifications |
| Slack | slack-webhook | Team alerts |
| Database | Built-in | In-app notifications |
| Preferences | Custom | User-controlled channels |

**ต่อไป**: Part 49 - WordPress Headless

---
