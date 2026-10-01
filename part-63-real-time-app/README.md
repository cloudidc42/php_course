# Part 63: Real-Time Application

## ขั้นตอนที่ 1751-1780: Building Real-Time Applications

Real-time applications ให้ประสบการณ์แบบ live โดยใช้ WebSockets, SSE หรือ Long Polling

---

## ขั้นตอนที่ 1751: WebSocket ด้วย Laravel Reverb

```bash
# ติดตั้ง Laravel Reverb (official WebSocket server)
composer require laravel/reverb

# ติดตั้ง
php artisan reverb:install

# รัน server
php artisan reverb:start --host=0.0.0.0 --port=8080

# ติดตั้ง Laravel Echo (frontend)
npm install laravel-echo pusher-js
```

```php
<?php

declare(strict_types=1);

// config/broadcasting.php
return [
    'default' => env('BROADCAST_DRIVER', 'reverb'),

    'connections' => [
        'reverb' => [
            'driver'   => 'reverb',
            'key'      => env('REVERB_APP_KEY'),
            'secret'   => env('REVERB_APP_SECRET'),
            'app_id'   => env('REVERB_APP_ID'),
            'options'  => [
                'host'   => env('REVERB_HOST', 'localhost'),
                'port'   => env('REVERB_PORT', 8080),
                'scheme' => env('REVERB_SCHEME', 'http'),
                'useTLS' => env('REVERB_SCHEME', 'http') === 'https',
            ],
        ],
    ],
];
```

---

## ขั้นตอนที่ 1752: Broadcasting Events

```php
<?php

declare(strict_types=1);

// app/Events/MessageSent.php
namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly Message $message,
        public readonly int $roomId,
    ) {}

    // Channel ที่จะ broadcast ไป
    public function broadcastOn(): array
    {
        return [
            new PrivateChannel("chat.{$this->roomId}"),
        ];
    }

    // ชื่อ event ที่ frontend จะรับ
    public function broadcastAs(): string
    {
        return 'message.sent';
    }

    // ข้อมูลที่ส่งไป
    public function broadcastWith(): array
    {
        return [
            'id'         => $this->message->id,
            'content'    => $this->message->content,
            'user_id'    => $this->message->user_id,
            'user'       => [
                'id'     => $this->message->user->id,
                'name'   => $this->message->user->name,
                'avatar' => $this->message->user->avatar_url,
            ],
            'created_at' => $this->message->created_at->toISOString(),
        ];
    }

    // Queue connection สำหรับ broadcast
    public function broadcastQueue(): string
    {
        return 'broadcasts';
    }
}
```

---

## ขั้นตอนที่ 1753: Presence Channels สำหรับ Online Users

```php
<?php

declare(strict_types=1);

// routes/channels.php
use App\Models\ChatRoom;
use Illuminate\Support\Facades\Broadcast;

// Private channel authorization
Broadcast::channel('chat.{roomId}', function ($user, $roomId) {
    return ChatRoom::find($roomId)?->members()->where('user_id', $user->id)->exists();
});

// Presence channel authorization + user info
Broadcast::channel('chat.room.{roomId}', function ($user, $roomId) {
    $room = ChatRoom::find($roomId);

    if (!$room?->members()->where('user_id', $user->id)->exists()) {
        return false;
    }

    // Return user info สำหรับ presence channel
    return [
        'id'     => $user->id,
        'name'   => $user->name,
        'avatar' => $user->avatar_url,
        'status' => 'online',
    ];
});
```

```php
<?php

declare(strict_types=1);

// app/Events/UserTyping.php
namespace App\Events;

use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserTyping implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public readonly int    $userId,
        public readonly string $userName,
        public readonly int    $roomId,
        public readonly bool   $isTyping,
    ) {}

    public function broadcastOn(): array
    {
        return [new PresenceChannel("chat.room.{$this->roomId}")];
    }

    public function broadcastAs(): string
    {
        return 'user.typing';
    }

    public function broadcastWith(): array
    {
        return [
            'user_id'   => $this->userId,
            'user_name' => $this->userName,
            'is_typing' => $this->isTyping,
        ];
    }
}
```

---

## ขั้นตอนที่ 1754: Chat Controller

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/ChatController.php
namespace App\Http\Controllers;

use App\Events\MessageSent;
use App\Events\UserTyping;
use App\Models\ChatRoom;
use App\Models\Message;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class ChatController extends Controller
{
    public function sendMessage(Request $request, int $roomId): JsonResponse
    {
        $request->validate([
            'content' => 'required|string|max:2000',
            'type'    => 'nullable|in:text,image,file',
        ]);

        $room = ChatRoom::findOrFail($roomId);

        // Verify user is member
        abort_unless(
            $room->members()->where('user_id', $request->user()->id)->exists(),
            403,
            'Not a member of this room'
        );

        $message = $room->messages()->create([
            'user_id' => $request->user()->id,
            'content' => $request->content,
            'type'    => $request->input('type', 'text'),
        ]);

        $message->load('user');

        // Broadcast to all members in room
        broadcast(new MessageSent($message, $roomId))->toOthers();

        return response()->json($message, 201);
    }

    public function markTyping(Request $request, int $roomId): JsonResponse
    {
        $request->validate(['is_typing' => 'required|boolean']);

        broadcast(new UserTyping(
            userId:   $request->user()->id,
            userName: $request->user()->name,
            roomId:   $roomId,
            isTyping: $request->boolean('is_typing'),
        ))->toOthers();

        return response()->json(['status' => 'ok']);
    }

    public function getMessages(Request $request, int $roomId): JsonResponse
    {
        $room = ChatRoom::findOrFail($roomId);

        $messages = $room->messages()
            ->with('user:id,name,avatar')
            ->latest()
            ->paginate(50);

        return response()->json($messages);
    }
}
```

---

## ขั้นตอนที่ 1755: Server-Sent Events (SSE)

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/SseController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\StreamedResponse;

class SseController extends Controller
{
    public function stream(Request $request): StreamedResponse
    {
        $userId = $request->user()->id;

        return response()->stream(function () use ($userId) {
            // Headers สำหรับ SSE
            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');
            header('X-Accel-Buffering: no'); // สำหรับ Nginx

            $lastEventId = 0;

            while (true) {
                // Check ถ้า client disconnect แล้ว
                if (connection_aborted()) {
                    break;
                }

                // ดึง notifications ใหม่
                $notifications = \App\Models\Notification::where('user_id', $userId)
                    ->where('id', '>', $lastEventId)
                    ->where('read_at', null)
                    ->orderBy('id')
                    ->limit(10)
                    ->get();

                foreach ($notifications as $notification) {
                    $this->sendSseEvent(
                        id:   $notification->id,
                        event: 'notification',
                        data: json_encode([
                            'id'      => $notification->id,
                            'type'    => $notification->type,
                            'message' => $notification->message,
                            'time'    => $notification->created_at->diffForHumans(),
                        ])
                    );

                    $lastEventId = $notification->id;
                }

                // Send heartbeat ทุก 30 วินาที
                $this->sendSseEvent(event: 'heartbeat', data: 'ping');

                ob_flush();
                flush();

                sleep(5); // Poll ทุก 5 วินาที
            }
        }, 200, [
            'Content-Type'  => 'text/event-stream',
            'Cache-Control' => 'no-cache',
            'Connection'    => 'keep-alive',
        ]);
    }

    private function sendSseEvent(?int $id = null, string $event = 'message', string $data = ''): void
    {
        if ($id !== null) {
            echo "id: {$id}\n";
        }
        echo "event: {$event}\n";
        echo "data: {$data}\n";
        echo "\n"; // ต้องมี blank line หลัง event
    }
}
```

---

## ขั้นตอนที่ 1756: Real-Time Dashboard

```php
<?php

declare(strict_types=1);

// app/Events/DashboardMetricUpdated.php
namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;

class DashboardMetricUpdated implements ShouldBroadcast
{
    use Dispatchable;

    public function __construct(
        public readonly string $metric,
        public readonly mixed  $value,
        public readonly array  $metadata = [],
    ) {}

    public function broadcastOn(): array
    {
        return [new Channel('dashboard')]; // Public channel
    }

    public function broadcastAs(): string
    {
        return 'metric.updated';
    }
}

// app/Console/Commands/BroadcastDashboardMetrics.php
namespace App\Console\Commands;

use App\Events\DashboardMetricUpdated;
use App\Models\Order;
use App\Models\User;
use Illuminate\Console\Command;

class BroadcastDashboardMetrics extends Command
{
    protected $signature = 'dashboard:broadcast';
    protected $description = 'Broadcast dashboard metrics in real-time';

    public function handle(): void
    {
        // Broadcast active users
        broadcast(new DashboardMetricUpdated(
            metric: 'active_users',
            value:  \Cache::get('active_users_count', 0),
        ));

        // Broadcast today's orders
        broadcast(new DashboardMetricUpdated(
            metric: 'today_orders',
            value:  Order::whereDate('created_at', today())->count(),
        ));

        // Broadcast revenue
        broadcast(new DashboardMetricUpdated(
            metric: 'today_revenue',
            value:  Order::whereDate('created_at', today())->sum('total'),
        ));
    }
}
```

---

## ขั้นตอนที่ 1757: Long Polling Fallback

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/LongPollingController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class LongPollingController extends Controller
{
    private const POLL_TIMEOUT = 30; // seconds
    private const POLL_INTERVAL = 1;  // seconds

    public function poll(Request $request): JsonResponse
    {
        $lastId  = (int) $request->input('last_id', 0);
        $userId  = $request->user()->id;
        $timeout = time() + self::POLL_TIMEOUT;

        // รอจนกว่าจะมี data ใหม่ หรือ timeout
        while (time() < $timeout) {
            $notifications = \App\Models\Notification::where('user_id', $userId)
                ->where('id', '>', $lastId)
                ->orderBy('id')
                ->get();

            if ($notifications->isNotEmpty()) {
                return response()->json([
                    'notifications' => $notifications,
                    'last_id'       => $notifications->last()->id,
                ]);
            }

            // ป้องกัน CPU spike
            usleep(self::POLL_INTERVAL * 1000000);
        }

        // Timeout - ส่ง empty response
        return response()->json([
            'notifications' => [],
            'last_id'       => $lastId,
        ]);
    }
}
```

---

## ขั้นตอนที่ 1758: Laravel Echo Frontend Setup

```javascript
// resources/js/echo.js
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key:         import.meta.env.VITE_REVERB_APP_KEY,
    wsHost:      import.meta.env.VITE_REVERB_HOST,
    wsPort:      import.meta.env.VITE_REVERB_PORT ?? 80,
    wssPort:     import.meta.env.VITE_REVERB_PORT ?? 443,
    forceTLS:    (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});

// Listen to public channel
Echo.channel('dashboard')
    .listen('.metric.updated', (data) => {
        console.log('Metric updated:', data);
        updateDashboardMetric(data.metric, data.value);
    });

// Listen to private channel
Echo.private(`chat.${roomId}`)
    .listen('.message.sent', (data) => {
        appendMessage(data);
    });

// Presence channel
Echo.join(`chat.room.${roomId}`)
    .here((users) => {
        // users ที่อยู่ใน channel ตอนนี้
        updateOnlineUsers(users);
    })
    .joining((user) => {
        addOnlineUser(user);
        showNotification(`${user.name} joined`);
    })
    .leaving((user) => {
        removeOnlineUser(user);
        showNotification(`${user.name} left`);
    })
    .listen('.user.typing', (data) => {
        if (data.is_typing) {
            showTypingIndicator(data.user_name);
        } else {
            hideTypingIndicator(data.user_name);
        }
    });
```

---

## สรุป Part 63

| Technology | Use Case | Pros | Cons |
|-----------|---------|------|------|
| WebSocket | Chat, collaboration | Bidirectional, low latency | Complex setup |
| SSE | Notifications, live feed | Simple, HTTP-based | Server → Client only |
| Long Polling | Simple real-time | Works everywhere | Higher server load |
| Presence Channels | Online users | User tracking | Requires auth |
| Broadcasting | Event push | Decoupled | Queue dependency |

ถัดไป → Part 64: Multi-Tenancy Architecture
