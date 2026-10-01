# Part 96: Mobile App Backend
## ขั้นตอนที่ 2741-2770: Laravel Backend สำหรับ Mobile Apps

Push Notifications, Mobile Authentication, Offline Sync
Binary Responses และ App Versioning

---

## ขั้นตอนที่ 2741: Push Notifications กับ FCM

```bash
composer require kreait/firebase-php
```

```php
<?php
declare(strict_types=1);

namespace App\Services\Mobile;

use Kreait\Firebase\Factory;
use Kreait\Firebase\Messaging\CloudMessage;
use Kreait\Firebase\Messaging\Notification;
use Kreait\Firebase\Messaging\MulticastSendReport;

class PushNotificationService
{
    private \Kreait\Firebase\Contract\Messaging $messaging;

    public function __construct()
    {
        $factory = (new Factory())->withServiceAccount(
            config('services.firebase.credentials_file')
        );
        $this->messaging = $factory->createMessaging();
    }

    public function sendToDevice(string $deviceToken, array $data): bool
    {
        $message = CloudMessage::withTarget('token', $deviceToken)
            ->withNotification(
                Notification::create($data['title'], $data['body'])
            )
            ->withData(array_map('strval', $data['payload'] ?? []))
            ->withAndroidConfig([
                'priority' => 'high',
                'notification' => [
                    'sound' => 'default',
                    'channel_id' => $data['channel'] ?? 'default',
                ],
            ])
            ->withApnsConfig([
                'headers' => ['apns-priority' => '10'],
                'payload' => [
                    'aps' => [
                        'sound' => 'default',
                        'badge' => $data['badge'] ?? 1,
                    ],
                ],
            ]);

        try {
            $this->messaging->send($message);
            return true;
        } catch (\Throwable $e) {
            \Log::error('Push notification failed', [
                'token' => substr($deviceToken, 0, 10) . '...',
                'error' => $e->getMessage(),
            ]);
            return false;
        }
    }

    public function sendToMultipleDevices(array $tokens, array $data): MulticastSendReport
    {
        $message = CloudMessage::new()
            ->withNotification(Notification::create($data['title'], $data['body']))
            ->withData(array_map('strval', $data['payload'] ?? []));

        $report = $this->messaging->sendMulticast($message, $tokens);

        // Log failed tokens
        if ($report->hasFailures()) {
            foreach ($report->failures()->getItems() as $failure) {
                \Log::warning('Push notification failed for token', [
                    'token' => substr($failure->target()->value(), 0, 10) . '...',
                    'error' => $failure->error()->getMessage(),
                ]);
            }
        }

        return $report;
    }

    public function sendToTopic(string $topic, array $data): bool
    {
        $message = CloudMessage::withTarget('topic', $topic)
            ->withNotification(Notification::create($data['title'], $data['body']))
            ->withData(array_map('strval', $data['payload'] ?? []));

        try {
            $this->messaging->send($message);
            return true;
        } catch (\Throwable $e) {
            return false;
        }
    }

    public function subscribeToTopic(array $tokens, string $topic): void
    {
        $this->messaging->subscribeToTopic($topic, $tokens);
    }
}
```

---

## ขั้นตอนที่ 2742: Device Token Management

```php
<?php
declare(strict_types=1);

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class DeviceToken extends Model
{
    protected $fillable = [
        'user_id', 'token', 'platform', 'app_version',
        'os_version', 'device_model', 'is_active', 'last_used_at',
    ];

    protected $casts = [
        'is_active' => 'boolean',
        'last_used_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\Mobile;

use App\Models\DeviceToken;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class DeviceController extends \App\Http\Controllers\Controller
{
    public function register(Request $request): JsonResponse
    {
        $request->validate([
            'token' => ['required', 'string'],
            'platform' => ['required', 'in:ios,android'],
            'app_version' => ['required', 'string'],
            'os_version' => ['nullable', 'string'],
            'device_model' => ['nullable', 'string'],
        ]);

        $device = DeviceToken::updateOrCreate(
            [
                'user_id' => $request->user()->id,
                'token' => $request->token,
            ],
            [
                'platform' => $request->platform,
                'app_version' => $request->app_version,
                'os_version' => $request->os_version,
                'device_model' => $request->device_model,
                'is_active' => true,
                'last_used_at' => now(),
            ]
        );

        return response()->json(['message' => 'Device registered', 'device_id' => $device->id]);
    }

    public function unregister(Request $request): JsonResponse
    {
        DeviceToken::where('user_id', $request->user()->id)
            ->where('token', $request->token)
            ->update(['is_active' => false]);

        return response()->json(['message' => 'Device unregistered']);
    }
}
```

---

## ขั้นตอนที่ 2743: Mobile Authentication

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\Mobile;

use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class MobileAuthController extends \App\Http\Controllers\Controller
{
    public function login(Request $request): JsonResponse
    {
        $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required'],
            'device_name' => ['required', 'string'],
            'platform' => ['required', 'in:ios,android'],
            'push_token' => ['nullable', 'string'],
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json([
                'message' => 'อีเมลหรือรหัสผ่านไม่ถูกต้อง',
                'code' => 'INVALID_CREDENTIALS',
            ], 401);
        }

        if ($user->is_banned) {
            return response()->json([
                'message' => 'บัญชีถูกระงับ',
                'code' => 'ACCOUNT_BANNED',
            ], 403);
        }

        // สร้าง token
        $token = $user->createToken(
            $request->device_name,
            ['mobile'],
            now()->addDays(30)
        );

        // บันทึก push token
        if ($request->push_token) {
            $user->deviceTokens()->updateOrCreate(
                ['token' => $request->push_token],
                [
                    'platform' => $request->platform,
                    'is_active' => true,
                    'last_used_at' => now(),
                ]
            );
        }

        return response()->json([
            'token' => $token->plainTextToken,
            'expires_at' => now()->addDays(30)->toISOString(),
            'user' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
                'avatar' => $user->avatar_url,
                'role' => $user->role,
            ],
        ]);
    }

    public function refreshToken(Request $request): JsonResponse
    {
        $user = $request->user();
        $currentToken = $user->currentAccessToken();

        // สร้าง token ใหม่
        $newToken = $user->createToken(
            $currentToken->name,
            $currentToken->abilities,
            now()->addDays(30)
        );

        // ลบ token เก่า
        $currentToken->delete();

        return response()->json([
            'token' => $newToken->plainTextToken,
            'expires_at' => now()->addDays(30)->toISOString(),
        ]);
    }
}
```

---

## ขั้นตอนที่ 2744: Offline Sync Strategy

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api\Mobile;

use App\Models\Post;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class SyncController extends \App\Http\Controllers\Controller
{
    /**
     * Delta sync - ส่งเฉพาะข้อมูลที่เปลี่ยนแปลง
     */
    public function sync(Request $request): JsonResponse
    {
        $request->validate([
            'last_sync_at' => ['nullable', 'date'],
            'device_id' => ['required', 'string'],
        ]);

        $lastSync = $request->last_sync_at
            ? \Carbon\Carbon::parse($request->last_sync_at)
            : now()->subYear();

        // ดึงข้อมูลที่เปลี่ยนแปลงตั้งแต่ last sync
        $posts = Post::where('updated_at', '>', $lastSync)
            ->select(['id', 'title', 'slug', 'excerpt', 'status', 'updated_at', 'deleted_at'])
            ->withTrashed()
            ->limit(100)
            ->get();

        return response()->json([
            'synced_at' => now()->toISOString(),
            'created' => $posts->filter(fn ($p) => !$p->trashed() && $p->created_at > $lastSync),
            'updated' => $posts->filter(fn ($p) => !$p->trashed() && $p->created_at <= $lastSync),
            'deleted' => $posts->filter(fn ($p) => $p->trashed())->pluck('id'),
            'has_more' => $posts->count() === 100,
        ]);
    }

    /**
     * Batch update จาก offline changes
     */
    public function pushChanges(Request $request): JsonResponse
    {
        $request->validate([
            'changes' => ['required', 'array'],
            'changes.*.type' => ['required', 'in:create,update,delete'],
            'changes.*.entity' => ['required', 'string'],
            'changes.*.data' => ['required', 'array'],
            'changes.*.client_id' => ['required', 'string'],
        ]);

        $results = [];
        $conflicts = [];

        foreach ($request->changes as $change) {
            try {
                $result = $this->processChange($change, $request->user());
                $results[] = [
                    'client_id' => $change['client_id'],
                    'status' => 'success',
                    'server_id' => $result['id'] ?? null,
                ];
            } catch (\Exception $e) {
                $conflicts[] = [
                    'client_id' => $change['client_id'],
                    'error' => $e->getMessage(),
                    'code' => 'CONFLICT',
                ];
            }
        }

        return response()->json([
            'results' => $results,
            'conflicts' => $conflicts,
            'processed_at' => now()->toISOString(),
        ]);
    }

    private function processChange(array $change, \App\Models\User $user): array
    {
        return match ($change['entity']) {
            'post' => $this->processPostChange($change, $user),
            default => throw new \InvalidArgumentException("Unknown entity: {$change['entity']}"),
        };
    }

    private function processPostChange(array $change, \App\Models\User $user): array
    {
        return match ($change['type']) {
            'create' => Post::create(array_merge($change['data'], ['user_id' => $user->id]))->toArray(),
            'update' => tap(Post::findOrFail($change['data']['id']), fn ($p) => $p->update($change['data']))->toArray(),
            'delete' => tap(Post::findOrFail($change['data']['id']), fn ($p) => $p->delete())->toArray(),
            default => throw new \InvalidArgumentException("Unknown change type"),
        };
    }
}
```

---

## ขั้นตอนที่ 2745: App Version Management

```php
<?php
declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CheckAppVersion
{
    private const MIN_SUPPORTED_VERSION = '2.0.0';
    private const LATEST_VERSION = '3.5.0';
    private const FORCE_UPDATE_VERSIONS = ['1.x', '2.0', '2.1'];

    public function handle(Request $request, Closure $next): Response
    {
        $appVersion = $request->header('X-App-Version');
        $platform = $request->header('X-Platform');

        if (!$appVersion) {
            return $next($request);
        }

        // ตรวจสอบว่าต้อง force update หรือไม่
        if ($this->shouldForceUpdate($appVersion, $platform)) {
            return response()->json([
                'error' => 'UPDATE_REQUIRED',
                'message' => 'กรุณาอัพเดท app เป็นเวอร์ชั่นล่าสุด',
                'current_version' => $appVersion,
                'latest_version' => self::LATEST_VERSION,
                'update_url' => $this->getUpdateUrl($platform),
                'force_update' => true,
            ], 426); // Upgrade Required
        }

        // แจ้งเตือน optional update
        $response = $next($request);

        if (version_compare($appVersion, self::LATEST_VERSION, '<')) {
            $response->headers->set('X-Latest-Version', self::LATEST_VERSION);
            $response->headers->set('X-Update-Available', 'true');
        }

        return $response;
    }

    private function shouldForceUpdate(string $version, ?string $platform): bool
    {
        // Version ต่ำกว่า minimum ต้อง force update
        if (version_compare($version, self::MIN_SUPPORTED_VERSION, '<')) {
            return true;
        }

        // Check force update list
        foreach (self::FORCE_UPDATE_VERSIONS as $forceVersion) {
            if (str_starts_with($version, $forceVersion)) {
                return true;
            }
        }

        return false;
    }

    private function getUpdateUrl(?string $platform): string
    {
        return match ($platform) {
            'ios' => 'https://apps.apple.com/app/id000000000',
            'android' => 'https://play.google.com/store/apps/details?id=com.example.app',
            default => 'https://app.example.com/update',
        };
    }
}
```

---

## สรุปบทที่ 96

| หัวข้อ | เครื่องมือ | ประโยชน์ |
|--------|----------|---------|
| Push Notifications | kreait/firebase-php | FCM/APNs |
| Device Management | DeviceToken model | Track devices |
| Mobile Auth | Sanctum + abilities | Token-based |
| Offline Sync | Delta sync API | Work offline |
| Push to batch | Batch changes API | Conflict resolution |
| App Versioning | Middleware | Force update |

ถัดไป → Part 97: WordPress Enterprise
