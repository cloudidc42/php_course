# Part 52: PHP Internals

## ขั้นตอนที่ 1421-1450: PHP ภายในและประสิทธิภาพขั้นสูง

บทนี้เจาะลึก PHP Opcache, JIT Compilation, PHP Extensions, Memory Model, Garbage Collection และ PHP 8.4

---

## ขั้นตอนที่ 1421: PHP Opcache Internals

Opcache แปลง PHP source code เป็น bytecode และ cache ไว้ใน memory

```ini
; php.ini - Opcache configuration
[opcache]
opcache.enable=1
opcache.enable_cli=1

; Memory ที่ใช้เก็บ opcache (MB)
opcache.memory_consumption=256

; จำนวน string ที่ intern
opcache.interned_strings_buffer=32

; จำนวน files สูงสุดที่ cache
opcache.max_accelerated_files=20000

; ตรวจสอบ file changes ทุกกี่วินาที (0 = ไม่ตรวจสอบ)
opcache.revalidate_freq=0

; ไม่ตรวจสอบ timestamps (production)
opcache.validate_timestamps=0

; ไม่ include comments ใน bytecode
opcache.save_comments=0

; Preload script (PHP 7.4+)
opcache.preload=/var/www/myapp/preload.php
opcache.preload_user=www-data

; JIT
opcache.jit_buffer_size=100M
opcache.jit=1255
```

```php
<?php

declare(strict_types=1);

// ตรวจสอบ opcache status
function getOpcacheStatus(): array
{
    if (!function_exists('opcache_get_status')) {
        return ['enabled' => false];
    }

    $status = opcache_get_status(false);

    if (!$status) {
        return ['enabled' => false];
    }

    $memory = $status['memory_usage'];
    $stats  = $status['opcache_statistics'];

    return [
        'enabled'          => true,
        'hit_rate'         => round($stats['opcache_hit_rate'], 2) . '%',
        'used_memory'      => round($memory['used_memory'] / 1024 / 1024, 2) . 'MB',
        'free_memory'      => round($memory['free_memory'] / 1024 / 1024, 2) . 'MB',
        'wasted_memory'    => round($memory['wasted_memory'] / 1024 / 1024, 2) . 'MB',
        'cached_files'     => $stats['num_cached_scripts'],
        'cache_full'       => $status['cache_full'],
        'hits'             => number_format($stats['hits']),
        'misses'           => number_format($stats['misses']),
    ];
}

// Preload script สำหรับ PHP 7.4+
// preload.php - โหลด classes ล่วงหน้าก่อนรับ request
```

```php
<?php

declare(strict_types=1);

// preload.php
$directory = __DIR__ . '/app';

$files = new RecursiveIteratorIterator(
    new RecursiveDirectoryIterator($directory, RecursiveDirectoryIterator::SKIP_DOTS),
    RecursiveIteratorIterator::SELF_FIRST
);

$count = 0;
foreach ($files as $file) {
    if ($file->isFile() && $file->getExtension() === 'php') {
        opcache_compile_file($file->getRealPath());
        $count++;
    }
}

echo "Preloaded {$count} files" . PHP_EOL;
```

---

## ขั้นตอนที่ 1422: JIT Compilation

```ini
; JIT modes
; 0 = disabled
; 1 = minimal JIT (tracing, fallback)
; 1205 = tracing JIT - fastest for web
; 1255 = function-level JIT with tracing
; 1235 = tracing JIT with inlining

opcache.jit=1255
opcache.jit_buffer_size=100M
```

```php
<?php

declare(strict_types=1);

// JIT มีประโยชน์มากสำหรับ CPU-intensive code
// ตัวอย่าง: mandelbrot set (CPU-bound)

function mandelbrot(int $size): int
{
    $count = 0;

    for ($row = 0; $row < $size; $row++) {
        for ($col = 0; $col < $size; $col++) {
            $c_re = ($col - $size / 2.0) * 4.0 / $size;
            $c_im = ($row - $size / 2.0) * 4.0 / $size;

            $x = 0.0;
            $y = 0.0;
            $iteration = 0;

            while ($x * $x + $y * $y <= 4 && $iteration < 100) {
                $x_new = $x * $x - $y * $y + $c_re;
                $y = 2 * $x * $y + $c_im;
                $x = $x_new;
                $iteration++;
            }

            if ($iteration < 100) {
                $count++;
            }
        }
    }

    return $count;
}

// Without JIT: ~2000ms
// With JIT enabled: ~300ms (6x faster!)
$start = microtime(true);
$result = mandelbrot(1000);
$elapsed = (microtime(true) - $start) * 1000;

echo "Result: {$result}, Time: " . round($elapsed) . "ms\n";
```

---

## ขั้นตอนที่ 1423: PHP Memory Model

```php
<?php

declare(strict_types=1);

// PHP uses copy-on-write (COW) for arrays and strings
$a = ['large' => range(1, 10000)]; // Allocated once
$b = $a;  // ยังไม่ copy - ชี้ไปที่เดิม (COW)

$b['new'] = 'value';  // ตอนนี้ copy

// ตรวจสอบ memory
echo "Memory: " . memory_get_usage(true) / 1024 . " KB\n";

// Memory management
function processLargeData(): void
{
    // ใช้ Generator แทน Array สำหรับ large datasets
    $data = generateData(1000000);

    foreach ($data as $item) {
        process($item);
    }
    // ไม่ต้อง unset - Generator ไม่ถือ data ทั้งหมดใน memory
}

function generateData(int $count): \Generator
{
    for ($i = 0; $i < $count; $i++) {
        yield ['id' => $i, 'value' => random_int(1, 100)];
    }
}

// Circular references - PHP GC ต้องจัดการ
class Node
{
    public ?Node $next     = null;
    public ?Node $previous = null;

    public function __construct(public string $value) {}

    public function __destruct()
    {
        // Destructor - เรียกเมื่อ GC เก็บ object
    }
}

$a = new Node('A');
$b = new Node('B');
$a->next = $b;
$b->previous = $a; // Circular reference!

// PHP GC จัดการ circular references ได้
// แต่ memory จะไม่ถูก free ทันที

// ล้าง circular reference แบบ manual
$a->next = null;
$b->previous = null;
unset($a, $b);
```

---

## ขั้นตอนที่ 1424: Garbage Collection

```php
<?php

declare(strict_types=1);

// ดู GC stats
$stats = gc_status();
echo "Runs: {$stats['runs']}\n";
echo "Collected: {$stats['collected']}\n";
echo "Roots: {$stats['roots']}\n";
echo "Threshold: {$stats['threshold']}\n";

// Enable/Disable GC
gc_enable();
gc_disable();

// Force GC
$collected = gc_collect_cycles();
echo "Collected {$collected} cycles\n";

// Memory-efficient processing
function processWithGcOptimization(array $ids): void
{
    // ปิด GC ระหว่าง loop เพื่อความเร็ว
    gc_disable();

    $gcRootCount = 0;
    foreach ($ids as $id) {
        $model = Model::find($id);
        processModel($model);
        unset($model);

        $gcRootCount++;

        // เรียก GC ทุก 1000 iterations
        if ($gcRootCount % 1000 === 0) {
            gc_enable();
            gc_collect_cycles();
            gc_disable();
        }
    }

    gc_enable();
    gc_collect_cycles();
}

function processModel(mixed $model): void
{
    // Process...
}
```

---

## ขั้นตอนที่ 1425: PHP 8.4 New Features

```php
<?php

declare(strict_types=1);

// PHP 8.4 - Property Hooks
class User
{
    public string $fullName {
        get => "{$this->firstName} {$this->lastName}";
        set (string $value) {
            $parts = explode(' ', $value, 2);
            $this->firstName = $parts[0];
            $this->lastName  = $parts[1] ?? '';
        }
    }

    public string $email {
        get => $this->email;
        set {
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new \InvalidArgumentException("Invalid email: {$value}");
            }
            $this->email = strtolower($value);
        }
    }

    public function __construct(
        public string $firstName,
        public string $lastName,
        string $email
    ) {
        $this->email = $email; // Triggers set hook
    }
}

$user = new User('John', 'Doe', 'JOHN@EXAMPLE.COM');
echo $user->fullName;   // "John Doe"
echo $user->email;      // "john@example.com"

$user->fullName = 'Jane Smith';
echo $user->firstName;  // "Jane"
echo $user->lastName;   // "Smith"
```

```php
<?php

declare(strict_types=1);

// PHP 8.4 - Asymmetric Visibility
class Order
{
    public private(set) int $id;

    // public read, protected write
    public protected(set) string $status = 'pending';

    public function __construct(int $id)
    {
        $this->id = $id;
    }

    public function complete(): void
    {
        $this->status = 'completed'; // OK - inside class
    }
}

class SpecialOrder extends Order
{
    public function markAsSpecial(): void
    {
        $this->status = 'special'; // OK - protected(set) allows subclass write
    }
}

$order = new Order(1);
echo $order->id;        // 1 - can read
// $order->id = 2;      // Error! private(set)
// $order->status = 'x'; // Error! protected(set)
```

```php
<?php

declare(strict_types=1);

// PHP 8.4 - New Array Functions
$numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3];

// array_find - หาค่าแรกที่ตรงเงื่อนไข
$firstLarge = array_find($numbers, fn ($n) => $n > 5);
echo $firstLarge; // 9

// array_find_key - หา key แรกที่ตรงเงื่อนไข
$firstLargeKey = array_find_key($numbers, fn ($n) => $n > 5);
echo $firstLargeKey; // 5

// array_any - ตรวจสอบว่ามีอย่างน้อย 1 ที่ตรง
$hasLarge = array_any($numbers, fn ($n) => $n > 8);
echo $hasLarge ? 'yes' : 'no'; // yes

// array_all - ตรวจสอบว่าทุกตัวตรง
$allPositive = array_all($numbers, fn ($n) => $n > 0);
echo $allPositive ? 'yes' : 'no'; // yes
```

```php
<?php

declare(strict_types=1);

// PHP 8.4 - new without parentheses
class Config
{
    public function get(string $key): mixed
    {
        return null;
    }
}

// Before 8.4
$value = (new Config())->get('debug');

// PHP 8.4 - new without parentheses
$value = new Config()->get('debug');

// PHP 8.4 - Lazy Objects
class ExpensiveService
{
    public function __construct()
    {
        // Simulates expensive initialization
        sleep(1);
    }

    public function doWork(): string
    {
        return 'done';
    }
}

// Object ไม่ถูก initialize จนกว่าจะมีการใช้งาน
$reflector = new \ReflectionClass(ExpensiveService::class);
$service = $reflector->newLazyGhost(function (ExpensiveService $instance) {
    // Initialize here
});

// Initialize ตอนนี้ เมื่อเข้าถึง property/method แรก
echo $service->doWork(); // "done" - initialize ตอนนี้
```

---

## ขั้นตอนที่ 1426: PHP Extensions Overview

```c
/* ตัวอย่าง PHP Extension ด้วย C (overview) */
/* hello_world.c */

#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "php_ini.h"
#include "ext/standard/info.h"

PHP_FUNCTION(hello_world)
{
    zend_string *name;

    ZEND_PARSE_PARAMETERS_START(1, 1)
        Z_PARAM_STR(name)
    ZEND_PARSE_PARAMETERS_END();

    php_printf("Hello, %s!\n", ZSTR_VAL(name));
}

static const zend_function_entry hello_world_functions[] = {
    PHP_FE(hello_world, NULL)
    PHP_FE_END
};

zend_module_entry hello_world_module_entry = {
    STANDARD_MODULE_HEADER,
    "hello_world",
    hello_world_functions,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    "1.0",
    STANDARD_MODULE_PROPERTIES
};

ZEND_GET_MODULE(hello_world)
```

```bash
# สร้าง extension skeleton
pecl create-ext hello_world

# Compile
phpize
./configure --enable-hello_world
make
make install

# Enable
echo "extension=hello_world.so" >> /etc/php/8.4/cli/php.ini
```

---

## สรุปบทที่ 52

| หัวข้อ | ความสำคัญ | ผลลัพธ์ |
|--------|-----------|---------|
| Opcache | สูงมาก | 2-10x faster |
| JIT | CPU-intensive code | 3-10x faster |
| Memory Model | COW, References | Memory efficiency |
| Garbage Collection | Circular refs | Prevent memory leaks |
| PHP 8.4 Features | Property hooks, Asymmetric visibility | Modern PHP |
| Extensions | C-level | Maximum performance |

**ต่อไป**: Part 53 - Laravel Telescope

---
