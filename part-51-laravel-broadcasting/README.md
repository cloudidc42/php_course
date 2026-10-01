# Part 51: Laravel Broadcasting

## ขั้นตอนที่ 1391-1420: Real-time Broadcasting ด้วย Laravel

Broadcasting ช่วยให้ส่งข้อมูลแบบ real-time ไปยัง browser ผ่าน WebSockets

---

## ขั้นตอนที่ 1391: WebSockets กับ Laravel Echo

```javascript
// ติดตั้ง frontend packages
// npm install laravel-echo pusher-js

// resources/js/bootstrap.js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: import.meta.env.VITE_PUSHER_APP_KEY,
    cluster: import.meta.env.VITE_PUSHER_APP_CLUSTER,
    forceTLS: true,
});

// Subscribe to public channel
Echo.channel('announcements')
    .listen('NewAnnouncement', (data) => {
        console.log('New announcement:', data.message);
        showNotification(data.message);
    });

// Subscribe to private channel
Echo.private(`orders.${userId}`)
    .listen('OrderStatusUpdated', (data) => {
        updateOrderStatus(data.order_id, data.status);
    });

// Subscribe to presence channel
Echo.join('chat.room.1')
    .here((users) => {
        // users already in channel
        updateOnlineUsers(users);
    })
    .joining((user) => {
        addOnlineUser(user);
    })
    .leaving((user) => {
        removeOnlineUser(user);
    })
    .listen('NewChatMessage', (data) => {
        appendMessage(data);
    });
```

---

## ขั้นตอนที่ 1392: Pusher Integration

```php
<?php

declare(strict_types=1);

// config/broadcasting.php
return [
    'default' => env('BROADCAST_DRIVER', 'pusher'),

    'connections' => [
        'pusher' => [
            'driver'  => 'pusher',
            'key'     => env('PUSHER_APP_KEY'),
            'secret'  => env('PUSHER_APP_SECRET'),
            'app_id'  => env('PUSHER_APP_ID'),
            'options' => [
                'cluster'   => env('PUSHER_APP_CLUSTER', 'ap1'),
                'useTLS'    => true,
                'host'      => env('PUSHER_HOST'),
                'port'      => env('PUSHER_PORT', 443),
                'scheme'    => env('PUSHER_SCHEME', 'https'),
                'encrypted' => true,
            ],
        ],

        'ably' => [
            'driver' => 'ably',
            'key'    => env('ABLY_KEY'),
        ],

        'redis' => [
            'driver'     => 'redis',
            'connection' => 'default',
        ],

        'log' => [
            'driver' => 'log',
        ],

        'null' => [
            'driver' => 'null',
        ],
    ],
];
```

```php
<?php

declare(strict_types=1);

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Queue\SerializesModels;

// ShouldBroadcastNow = ส่งทันที ไม่ผ่าน queue
class SystemAnnouncement implements ShouldBroadcastNow
{
    use InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly string $title,
        public readonly string $message,
        public readonly string $type = 'info'
    ) {}

    public function broadcastOn(): array
    {
        return [new Channel('announcements')];
    }

    public function broadcastAs(): string
    {
        return 'announcement';
    }

    public function broadcastWith(): array
    {
        return [
            'title'     => $this->title,
            'message'   => $this->message,
            'type'      => $this->type,
            'timestamp' => now()->toISOString(),
        ];
    }
}
```

---

## ขั้นตอนที่ 1393: Laravel Reverb (Self-hosted WebSockets)

```bash
# ติดตั้ง Laravel Reverb
composer require laravel/reverb

# Publish config
php artisan reverb:install

# รัน Reverb server
php artisan reverb:start --host=0.0.0.0 --port=8080
```

```php
<?php

declare(strict_types=1);

// config/reverb.php
return [
    'servers' => [
        'reverb' => [
            'host'    => env('REVERB_SERVER_HOST', '0.0.0.0'),
            'port'    => env('REVERB_SERVER_PORT', 8080),
            'scheme'  => env('REVERB_SCHEME', 'http'),
            'options' => [
                'tls' => [],
            ],
        ],
    ],
    'apps' => [
        'provider' => 'config',
        'apps'     => [
            [
                'key'              => env('REVERB_APP_KEY'),
                'secret'           => env('REVERB_APP_SECRET'),
                'app_id'           => env('REVERB_APP_ID'),
                'options'          => [
                    'host'         => env('REVERB_HOST', 'localhost'),
                    'port'         => env('REVERB_PORT', 443),
                    'scheme'       => env('REVERB_SCHEME', 'https'),
                    'useTLS'       => env('REVERB_SCHEME', 'https') === 'https',
                ],
                'allowed_origins'  => ['*'],
                'ping_interval'    => env('REVERB_APP_PING_INTERVAL', 60),
                'activity_timeout' => env('REVERB_APP_ACTIVITY_TIMEOUT', 30),
                'max_message_size' => env('REVERB_APP_MAX_MESSAGE_SIZE', 10000),
            ],
        ],
    ],
];
```

---

## ขั้นตอนที่ 1394: Private และ Presence Channels

```php
<?php

declare(strict_types=1);

// routes/channels.php
use Illuminate\Support\Facades\Broadcast;
use App\Models\Conversation;
use App\Models\Room;

// Private channel - ตรวจสอบ user
Broadcast::channel('user.{userId}', function ($user, int $userId) {
    return $user->id === $userId;
});

// Private channel กับ role check
Broadcast::channel('admin.dashboard', function ($user) {
    return $user->hasRole('admin');
});

// Presence channel - return user data
Broadcast::channel('room.{roomId}', function ($user, int $roomId) {
    $room = Room::find($roomId);

    if (!$room || !$room->members->contains($user->id)) {
        return false;
    }

    return [
        'id'     => $user->id,
        'name'   => $user->name,
        'avatar' => $user->avatar_url,
        'status' => 'online',
    ];
});

// Channel สำหรับ conversation
Broadcast::channel('conversation.{conversationId}', function ($user, int $conversationId) {
    $conversation = Conversation::find($conversationId);

    if (!$conversation) {
        return false;
    }

    return $conversation->participants->contains('id', $user->id);
});
```

---

## ขั้นตอนที่ 1395: Real-time Chat Application

```php
<?php

declare(strict_types=1);

namespace App\Http\Controllers;

use App\Events\NewChatMessage;
use App\Models\Conversation;
use App\Models\Message;
use Illuminate\Http\Request;

class ChatController
{
    public function sendMessage(Request $request, int $conversationId): \Illuminate\Http\JsonResponse
    {
        $validated = $request->validate([
            'body'         => ['required', 'string', 'max:5000'],
            'attachments'  => ['nullable', 'array', 'max:5'],
            'attachments.*' => ['file', 'max:10240'],
        ]);

        $conversation = Conversation::findOrFail($conversationId);

        // ตรวจสอบสิทธิ์
        $this->authorize('participate', $conversation);

        $message = Message::create([
            'conversation_id' => $conversationId,
            'user_id'         => auth()->id(),
            'body'            => $validated['body'],
            'type'            => 'text',
        ]);

        $message->load('user:id,name,avatar');

        // Broadcast ไปยัง participants ทุกคน
        broadcast(new NewChatMessage($message))->toOthers();

        return response()->json([
            'message' => $message,
        ], 201);
    }

    public function getMessages(int $conversationId): \Illuminate\Http\JsonResponse
    {
        $conversation = Conversation::findOrFail($conversationId);
        $this->authorize('view', $conversation);

        $messages = Message::where('conversation_id', $conversationId)
            ->with('user:id,name,avatar')
            ->latest()
            ->paginate(50);

        return response()->json($messages);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Queue\SerializesModels;

class NewChatMessage implements ShouldBroadcast
{
    use InteractsWithSockets, SerializesModels;

    public string $queue = 'broadcasting';

    public function __construct(public readonly Message $message) {}

    public function broadcastOn(): array
    {
        return [
            new PrivateChannel('conversation.' . $this->message->conversation_id),
        ];
    }

    public function broadcastAs(): string
    {
        return 'message.sent';
    }

    public function broadcastWith(): array
    {
        return [
            'id'              => $this->message->id,
            'conversation_id' => $this->message->conversation_id,
            'body'            => $this->message->body,
            'type'            => $this->message->type,
            'user'            => [
                'id'     => $this->message->user->id,
                'name'   => $this->message->user->name,
                'avatar' => $this->message->user->avatar,
            ],
            'created_at'      => $this->message->created_at->toISOString(),
        ];
    }
}
```

---

## ขั้นตอนที่ 1396: Real-time Notifications ด้วย Broadcasting

```php
<?php

declare(strict_types=1);

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcastNow;
use Illuminate\Queue\SerializesModels;

class UserNotification implements ShouldBroadcastNow
{
    use InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly User $user,
        public readonly string $title,
        public readonly string $body,
        public readonly string $type = 'info',
        public readonly ?string $actionUrl = null
    ) {}

    public function broadcastOn(): array
    {
        return [new PrivateChannel("user.{$this->user->id}")];
    }

    public function broadcastAs(): string
    {
        return 'notification';
    }

    public function broadcastWith(): array
    {
        return [
            'title'      => $this->title,
            'body'       => $this->body,
            'type'       => $this->type,
            'action_url' => $this->actionUrl,
            'timestamp'  => now()->toISOString(),
        ];
    }
}
```

```javascript
// resources/js/notifications.js
class NotificationManager {
    constructor(userId) {
        this.userId = userId;
        this.channel = null;
        this.unreadCount = 0;
        this.handlers = {};
    }

    connect() {
        this.channel = Echo.private(`user.${this.userId}`)
            .listen('.notification', (data) => {
                this.handleNotification(data);
            })
            .error((error) => {
                console.error('Broadcasting error:', error);
            });

        return this;
    }

    handleNotification(data) {
        this.unreadCount++;
        this.updateBadge();

        // แสดง toast notification
        this.showToast(data.title, data.body, data.type);

        // เรียก handlers ที่ลงทะเบียนไว้
        if (this.handlers[data.type]) {
            this.handlers[data.type](data);
        }
    }

    on(type, handler) {
        this.handlers[type] = handler;
        return this;
    }

    updateBadge() {
        const badge = document.getElementById('notification-badge');
        if (badge) {
            badge.textContent = this.unreadCount;
            badge.style.display = this.unreadCount > 0 ? 'block' : 'none';
        }
    }

    showToast(title, body, type = 'info') {
        // ใช้ library เช่น Toastr หรือ custom
        console.log(`[${type.toUpperCase()}] ${title}: ${body}`);
    }

    disconnect() {
        if (this.channel) {
            this.channel.stopListening('.notification');
            Echo.leave(`user.${this.userId}`);
        }
    }
}

// การใช้งาน
const notifications = new NotificationManager(window.userId)
    .connect()
    .on('order_status', (data) => {
        updateOrderStatusUI(data);
    });
```

---

## สรุปบทที่ 51

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|-----------|---------|
| Laravel Echo | Laravel Echo + Pusher-JS | Frontend WebSocket client |
| Pusher | Cloud WebSocket service | ไม่ต้อง manage server |
| Reverb | Self-hosted WebSockets | ลดค่าใช้จ่าย |
| Private Channels | Channel auth | Secure per-user events |
| Presence Channels | User tracking | Online status |
| Real-time Chat | Broadcast + Echo | Low-latency messaging |
| Real-time Notifications | Private channels | Instant alerts |

**ต่อไป**: Part 52 - PHP Internals

---
