# Part 83: Laravel Localization
## ขั้นตอนที่ 2351-2380: Multi-language Application

การทำ Web Application รองรับหลายภาษา
ครอบคลุม Translation files, Dynamic DB translations, Date/Number formatting

---

## ขั้นตอนที่ 2351: ตั้งค่า Localization

```php
<?php
declare(strict_types=1);

// config/app.php
return [
    'locale' => env('APP_LOCALE', 'th'),
    'fallback_locale' => env('APP_FALLBACK_LOCALE', 'en'),
    'faker_locale' => env('APP_FAKER_LOCALE', 'th_TH'),
    'supported_locales' => ['th', 'en', 'zh', 'ja'],
];
```

```php
<?php
declare(strict_types=1);

// app/Http/Middleware/SetLocale.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SetLocale
{
    public function handle(Request $request, Closure $next): Response
    {
        $locale = $this->determineLocale($request);

        app()->setLocale($locale);
        \Carbon\Carbon::setLocale($locale);

        return $next($request);
    }

    private function determineLocale(Request $request): string
    {
        $supported = config('app.supported_locales', ['th', 'en']);

        // 1. Check user preference (authenticated users)
        if ($request->user()?->locale) {
            $locale = $request->user()->locale;
            if (in_array($locale, $supported)) {
                return $locale;
            }
        }

        // 2. Check session
        if ($request->session()->has('locale')) {
            $locale = $request->session()->get('locale');
            if (in_array($locale, $supported)) {
                return $locale;
            }
        }

        // 3. Check request parameter
        if ($request->has('lang')) {
            $locale = $request->get('lang');
            if (in_array($locale, $supported)) {
                $request->session()->put('locale', $locale);
                return $locale;
            }
        }

        // 4. Check Accept-Language header
        $acceptLanguage = $request->header('Accept-Language', '');
        foreach (explode(',', $acceptLanguage) as $lang) {
            $locale = substr(trim($lang), 0, 2);
            if (in_array($locale, $supported)) {
                return $locale;
            }
        }

        return config('app.locale', 'th');
    }
}
```

---

## ขั้นตอนที่ 2352: Translation Files

```php
<?php
declare(strict_types=1);

// lang/th/messages.php
return [
    'welcome' => 'ยินดีต้อนรับ',
    'goodbye' => 'ลาก่อน',
    'auth' => [
        'login' => 'เข้าสู่ระบบ',
        'logout' => 'ออกจากระบบ',
        'register' => 'สมัครสมาชิก',
        'failed' => 'ข้อมูลไม่ถูกต้อง',
        'password' => 'รหัสผ่านไม่ถูกต้อง',
        'throttle' => 'พยายามเข้าระบบมากเกินไป กรุณาลองใหม่ใน :seconds วินาที',
    ],
    'posts' => [
        'created' => 'สร้างบทความสำเร็จ',
        'updated' => 'อัพเดตบทความสำเร็จ',
        'deleted' => 'ลบบทความสำเร็จ',
        'not_found' => 'ไม่พบบทความ',
        'count' => ':count บทความ',
        'published' => '{0} ยังไม่มีบทความ|{1} :count บทความ|[2,*] :count บทความ',
    ],
    'time' => [
        'just_now' => 'เมื่อกี้',
        'minutes_ago' => ':count นาทีที่แล้ว',
        'hours_ago' => ':count ชั่วโมงที่แล้ว',
        'days_ago' => ':count วันที่แล้ว',
    ],
];
```

```php
<?php
declare(strict_types=1);

// lang/en/messages.php
return [
    'welcome' => 'Welcome',
    'goodbye' => 'Goodbye',
    'auth' => [
        'login' => 'Login',
        'logout' => 'Logout',
        'register' => 'Register',
        'failed' => 'These credentials do not match our records.',
        'password' => 'The provided password is incorrect.',
        'throttle' => 'Too many login attempts. Please try again in :seconds seconds.',
    ],
    'posts' => [
        'created' => 'Post created successfully',
        'updated' => 'Post updated successfully',
        'deleted' => 'Post deleted successfully',
        'not_found' => 'Post not found',
        'count' => ':count posts',
        'published' => '{0} No posts|{1} :count post|[2,*] :count posts',
    ],
    'time' => [
        'just_now' => 'Just now',
        'minutes_ago' => ':count minutes ago',
        'hours_ago' => ':count hours ago',
        'days_ago' => ':count days ago',
    ],
];
```

---

## ขั้นตอนที่ 2353: Dynamic Translations กับ Database

```php
<?php
declare(strict_types=1);

// database/migrations/xxxx_create_translations_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('translations', function (Blueprint $table) {
            $table->id();
            $table->string('locale', 10);
            $table->string('group');
            $table->string('key');
            $table->text('value')->nullable();
            $table->timestamps();

            $table->unique(['locale', 'group', 'key']);
            $table->index(['locale', 'group']);
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('translations');
    }
};
```

```php
<?php
declare(strict_types=1);

namespace App\Services;

use App\Models\Translation;
use Illuminate\Support\Facades\Cache;

class TranslationService
{
    private const CACHE_TTL = 3600; // 1 hour

    public function get(string $key, string $locale, array $replace = []): string
    {
        [$group, $item] = $this->parseKey($key);

        $translations = $this->getGroupTranslations($group, $locale);
        $value = $translations[$item] ?? $this->getFallback($key, $locale, $replace);

        return $this->replaceParameters($value ?? $key, $replace);
    }

    private function getGroupTranslations(string $group, string $locale): array
    {
        return Cache::remember(
            "translations.{$locale}.{$group}",
            self::CACHE_TTL,
            fn () => Translation::where('locale', $locale)
                ->where('group', $group)
                ->pluck('value', 'key')
                ->toArray()
        );
    }

    public function set(string $group, string $key, string $locale, string $value): void
    {
        Translation::updateOrCreate(
            ['locale' => $locale, 'group' => $group, 'key' => $key],
            ['value' => $value]
        );

        Cache::forget("translations.{$locale}.{$group}");
    }

    public function sync(string $locale, string $group, array $translations): void
    {
        foreach ($translations as $key => $value) {
            $this->set($group, $key, $locale, $value);
        }
    }

    public function getMissingTranslations(string $locale): array
    {
        $allKeys = Translation::where('locale', config('app.fallback_locale'))
            ->get(['group', 'key'])
            ->pluck('key', 'group')
            ->toArray();

        $existingKeys = Translation::where('locale', $locale)
            ->get(['group', 'key'])
            ->groupBy('group')
            ->map(fn ($group) => $group->pluck('key')->toArray())
            ->toArray();

        $missing = [];
        foreach ($allKeys as $group => $keys) {
            foreach ((array)$keys as $key) {
                if (! in_array($key, $existingKeys[$group] ?? [])) {
                    $missing[] = "{$group}.{$key}";
                }
            }
        }

        return $missing;
    }

    private function parseKey(string $key): array
    {
        $parts = explode('.', $key, 2);
        return count($parts) === 2 ? $parts : ['general', $key];
    }

    private function getFallback(string $key, string $locale, array $replace): ?string
    {
        $fallback = config('app.fallback_locale');
        if ($locale !== $fallback) {
            [$group, $item] = $this->parseKey($key);
            $fallbackTranslations = $this->getGroupTranslations($group, $fallback);
            $value = $fallbackTranslations[$item] ?? null;
            if ($value) return $this->replaceParameters($value, $replace);
        }
        return null;
    }

    private function replaceParameters(string $string, array $replace): string
    {
        foreach ($replace as $key => $value) {
            $string = str_replace(":{$key}", $value, $string);
        }
        return $string;
    }
}
```

---

## ขั้นตอนที่ 2354: Translatable Models

```php
<?php
declare(strict_types=1);

// composer require spatie/laravel-translatable

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Spatie\Translatable\HasTranslations;

class Category extends Model
{
    use HasTranslations;

    public array $translatable = ['name', 'description', 'slug'];

    protected $fillable = ['name', 'description', 'slug', 'parent_id'];
}

// การใช้งาน
$category = Category::create([
    'name' => [
        'th' => 'เทคโนโลยี',
        'en' => 'Technology',
        'zh' => '技术',
    ],
    'description' => [
        'th' => 'บทความเกี่ยวกับเทคโนโลยี',
        'en' => 'Articles about technology',
    ],
]);

app()->setLocale('th');
echo $category->name; // เทคโนโลยี

app()->setLocale('en');
echo $category->name; // Technology

// แก้ไข translation
$category->setTranslation('name', 'ja', 'テクノロジー');
$category->save();

// ดึงทุก translations
$allNames = $category->getTranslations('name');
// ['th' => 'เทคโนโลยี', 'en' => 'Technology', ...]
```

---

## ขั้นตอนที่ 2355: Number และ Date Formatting

```php
<?php
declare(strict_types=1);

namespace App\Helpers;

use Carbon\Carbon;
use NumberFormatter;

class LocaleHelper
{
    public static function formatNumber(
        float|int $number,
        string $locale = null,
        int $decimals = 2
    ): string {
        $locale ??= app()->getLocale();

        $formatter = new NumberFormatter($locale, NumberFormatter::DECIMAL);
        $formatter->setAttribute(NumberFormatter::FRACTION_DIGITS, $decimals);

        return $formatter->format($number);
    }

    public static function formatCurrency(
        float|int $amount,
        string $currency = 'THB',
        string $locale = null
    ): string {
        $locale ??= app()->getLocale();

        $formatter = new NumberFormatter($locale, NumberFormatter::CURRENCY);
        return $formatter->formatCurrency($amount / 100, $currency);
    }

    public static function formatDate(
        Carbon|string $date,
        string $format = 'long',
        string $locale = null
    ): string {
        $locale ??= app()->getLocale();
        $carbon = $date instanceof Carbon ? $date : Carbon::parse($date);

        return match ($format) {
            'short' => $carbon->locale($locale)->isoFormat('D/M/YYYY'),
            'medium' => $carbon->locale($locale)->isoFormat('D MMM YYYY'),
            'long' => $carbon->locale($locale)->isoFormat('D MMMM YYYY'),
            'full' => $carbon->locale($locale)->isoFormat('dddd D MMMM YYYY'),
            'relative' => $carbon->locale($locale)->diffForHumans(),
            default => $carbon->locale($locale)->isoFormat($format),
        };
    }

    public static function formatBuddhistYear(Carbon $date): string
    {
        // แปลงเป็น พ.ศ.
        $be = $date->year + 543;
        return $date->locale('th')->isoFormat('D MMMM') . ' ' . $be;
    }

    public static function parseThaiDate(string $date): Carbon
    {
        // แปลง พ.ศ. เป็น ค.ศ.
        preg_match('/(\d{1,2})\/(\d{1,2})\/(\d{4})/', $date, $matches);
        if ($matches && (int)$matches[3] > 2500) {
            $year = (int)$matches[3] - 543;
            return Carbon::createFromDate($year, (int)$matches[2], (int)$matches[1]);
        }
        return Carbon::parse($date);
    }
}
```

---

## ขั้นตอนที่ 2356: Locale-based Routing

```php
<?php
declare(strict_types=1);

// routes/web.php
use Illuminate\Support\Facades\Route;

// Locale-prefixed routes
Route::prefix('{locale}')
    ->where(['locale' => '[a-z]{2}'])
    ->middleware('setLocale')
    ->group(function () {
        Route::get('/', [\App\Http\Controllers\HomeController::class, 'index'])->name('home');
        Route::get('/posts', [\App\Http\Controllers\PostController::class, 'index'])->name('posts.index');
        Route::get('/posts/{post:slug}', [\App\Http\Controllers\PostController::class, 'show'])->name('posts.show');
    });

// Redirect root to locale
Route::get('/', function () {
    $locale = session('locale', config('app.locale'));
    return redirect("/{$locale}");
});
```

```php
<?php
declare(strict_types=1);

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class LocaleFromRoute
{
    public function handle(Request $request, Closure $next): Response
    {
        $locale = $request->route('locale');
        $supported = config('app.supported_locales', ['th', 'en']);

        if (! in_array($locale, $supported)) {
            abort(404);
        }

        app()->setLocale($locale);
        session(['locale' => $locale]);

        return $next($request);
    }
}
```

---

## สรุปบทที่ 83

| หัวข้อ | เครื่องมือ | ตัวอย่าง |
|--------|----------|---------|
| File Translations | lang/*.php | __('messages.welcome') |
| DB Translations | Translation model + Cache | Dynamic content |
| Model Translations | spatie/translatable | $post->title in locale |
| Date Formatting | Carbon + isoFormat | พ.ศ./ค.ศ. |
| Number Formatting | NumberFormatter | ₿1,234.56 |
| Route Localization | Prefix + middleware | /th/posts |

ถัดไป → Part 84: WordPress E-commerce
