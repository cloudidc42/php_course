# Part 05: Functions พื้นฐานถึงขั้นสูง
## ขั้นตอนที่ 121-150

---

## ขั้นตอนที่ 121: Function Declaration และ Calling

```php
<?php
declare(strict_types=1);

// Basic Function
function greet(string $name): string {
    return "สวัสดี, $name!";
}

echo greet("สมชาย") . "\n";

// Function ที่ไม่ Return ค่า (void)
function printLine(string $text): void {
    echo $text . "\n";
}

printLine("Hello PHP!");

// Function ก่อนหรือหลัง Declaration ก็ได้ (Hoisting)
echo double(5) . "\n";  // 10 (ทำงานได้)

function double(int $n): int {
    return $n * 2;
}

// Recursive Function
function factorial(int $n): int {
    if ($n <= 1) return 1;
    return $n * factorial($n - 1);
}

echo factorial(5) . "\n";  // 120

// Fibonacci (Recursive + Memoization)
function fibonacci(int $n, array &$memo = []): int {
    if ($n <= 1) return $n;
    if (isset($memo[$n])) return $memo[$n];
    $memo[$n] = fibonacci($n - 1, $memo) + fibonacci($n - 2, $memo);
    return $memo[$n];
}

for ($i = 0; $i <= 10; $i++) {
    echo fibonacci($i) . " ";
}
echo "\n";  // 0 1 1 2 3 5 8 13 21 34 55
```

---

## ขั้นตอนที่ 122: Parameters และ Arguments

```php
<?php
declare(strict_types=1);

// Default Parameters
function createProfile(
    string $name,
    int $age = 0,
    string $city = 'Bangkok',
    bool $active = true
): array {
    return compact('name', 'age', 'city', 'active');
}

print_r(createProfile('Alice'));                    // ใช้ defaults
print_r(createProfile('Bob', 25));                 // age ระบุ
print_r(createProfile('Charlie', 30, 'Chiang Mai')); // city ระบุ

// Named Arguments (PHP 8.0+)
print_r(createProfile(
    name: 'Dave',
    active: false,
    age: 35
));  // ลำดับไม่สำคัญ

// Variadic Arguments (...)
function sum(int ...$numbers): int {
    return array_sum($numbers);
}

echo sum(1, 2, 3, 4, 5) . "\n";  // 15

$nums = [1, 2, 3, 4, 5];
echo sum(...$nums) . "\n";  // 15 (Spread)

// Mixed: Required + Variadic
function logMessage(string $level, string ...$messages): void {
    foreach ($messages as $msg) {
        echo "[$level] $msg\n";
    }
}

logMessage('INFO', 'Server started', 'Port: 8080', 'Debug mode: off');

// Type Declarations
function process(
    int $id,
    string $name,
    float $price,
    bool $active,
    array $tags,
    ?string $description = null  // Nullable
): array {
    return compact('id', 'name', 'price', 'active', 'tags', 'description');
}

// Return Type Declarations
function getUser(int $id): array {
    return ['id' => $id, 'name' => 'Test User'];
}

function findUser(int $id): ?array {  // Nullable return
    if ($id <= 0) return null;
    return ['id' => $id, 'name' => 'Found User'];
}

// Union Return Types (PHP 8.0+)
function divide(int $a, int $b): int|float {
    return $b === 0 ? 0 : $a / $b;
}

// Pass by Reference
function increment(int &$value, int $by = 1): void {
    $value += $by;
}

$counter = 0;
increment($counter);
increment($counter, 5);
echo $counter . "\n";  // 6

// Swap function
function swap(mixed &$a, mixed &$b): void {
    [$a, $b] = [$b, $a];
}

$x = "Hello";
$y = "World";
swap($x, $y);
echo "$x $y\n";  // World Hello
```

---

## ขั้นตอนที่ 123: Anonymous Functions (Closures)

```php
<?php
// Anonymous Function
$greet = function(string $name): string {
    return "Hello, $name!";
};

echo $greet("PHP") . "\n";

// Closure ใน Array
$operations = [
    'add' => fn($a, $b) => $a + $b,
    'sub' => fn($a, $b) => $a - $b,
    'mul' => fn($a, $b) => $a * $b,
    'div' => fn($a, $b) => $b != 0 ? $a / $b : null,
];

foreach ($operations as $name => $op) {
    echo "$name(10, 3) = " . $op(10, 3) . "\n";
}

// Closure กับ use (Capture Variables)
$prefix = "Hello";
$greeting = function(string $name) use ($prefix): string {
    return "$prefix, $name!";
};

echo $greeting("World") . "\n";

// Capture by Reference
$count = 0;
$increment = function() use (&$count): void {
    $count++;
};

$increment();
$increment();
$increment();
echo "Count: $count\n";  // 3

// Arrow Functions (PHP 7.4+) - auto-capture จาก parent scope
$multiplier = 3;
$multiply = fn($x) => $x * $multiplier;  // auto-capture $multiplier
echo $multiply(5) . "\n";  // 15

// Nested Arrow Functions
$addThenMultiply = fn($x) => fn($y) => ($x + $y) * $multiplier;
echo $addThenMultiply(2)(3) . "\n";  // 15 = (2+3) * 3

// Higher-Order Functions
function applyTwice(callable $fn, mixed $value): mixed {
    return $fn($fn($value));
}

echo applyTwice(fn($x) => $x * 2, 3) . "\n";  // 12

function compose(callable ...$fns): callable {
    return function($value) use ($fns) {
        foreach (array_reverse($fns) as $fn) {
            $value = $fn($value);
        }
        return $value;
    };
}

$process = compose(
    fn($x) => $x * 2,    // 3. ×2
    fn($x) => $x + 10,   // 2. +10
    fn($x) => $x ** 2    // 1. ²
);

echo $process(3) . "\n";  // 38 = ((3² + 10) * 2)

// Memoize Higher-Order Function
function memoize(callable $fn): callable {
    $cache = [];
    return function() use ($fn, &$cache) {
        $args = func_get_args();
        $key = serialize($args);
        if (!isset($cache[$key])) {
            $cache[$key] = $fn(...$args);
        }
        return $cache[$key];
    };
}

$expensiveCalc = memoize(function(int $n): int {
    echo "Computing for $n...\n";
    return $n * $n;
});

echo $expensiveCalc(5) . "\n";  // Computing for 5... 25
echo $expensiveCalc(5) . "\n";  // 25 (cached)
echo $expensiveCalc(6) . "\n";  // Computing for 6... 36
```

---

## ขั้นตอนที่ 124: Array Functions สำหรับ Functional Programming

```php
<?php
$numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// array_map - แปลงแต่ละ element
$doubled = array_map(fn($n) => $n * 2, $numbers);
print_r($doubled);

// Multiple Arrays
$sums = array_map(fn($a, $b) => $a + $b, [1, 2, 3], [10, 20, 30]);
print_r($sums);  // [11, 22, 33]

// array_filter - กรอง elements
$evens = array_filter($numbers, fn($n) => $n % 2 === 0);
print_r(array_values($evens));  // array_values reset keys

// array_filter ไม่มี callback = กรอง falsy values
$mixed = [0, 1, '', 'hello', null, false, true, [], [1]];
print_r(array_filter($mixed));  // [1, 'hello', true, [1]]

// array_reduce - รวมเป็นค่าเดียว
$sum = array_reduce($numbers, fn($carry, $item) => $carry + $item, 0);
echo "Sum: $sum\n";  // 55

$product = array_reduce($numbers, fn($carry, $item) => $carry * $item, 1);
echo "Product: $product\n";  // 3628800

// Chain Operations (Functional Style)
$result = array_reduce(
    array_filter(
        array_map(fn($n) => $n ** 2, $numbers),  // 1. ยกกำลัง 2
        fn($n) => $n > 25                          // 2. กรองค่า > 25
    ),
    fn($carry, $item) => $carry + $item,           // 3. รวม
    0
);
echo "Result: $result\n";  // 36+49+64+81+100 = 330

// usort, uasort, uksort
$people = [
    ['name' => 'Charlie', 'age' => 30],
    ['name' => 'Alice', 'age' => 25],
    ['name' => 'Bob', 'age' => 35],
];

// Sort by age
usort($people, fn($a, $b) => $a['age'] <=> $b['age']);
// Multi-key sort
usort($people, function($a, $b) {
    $ageCompare = $a['age'] <=> $b['age'];
    return $ageCompare !== 0 ? $ageCompare : strcmp($a['name'], $b['name']);
});

// array_walk
array_walk($people, function(&$person, $key) {
    $person['display'] = "#{$key}: {$person['name']} ({$person['age']})";
});

// array_column
$names = array_column($people, 'name');
$byName = array_column($people, null, 'name');

// array_unique, array_flip
$duplicates = [1, 2, 2, 3, 3, 3, 4];
print_r(array_unique($duplicates));  // [1, 2, 3, 4]

$flip = ['a' => 1, 'b' => 2, 'c' => 3];
print_r(array_flip($flip));  // [1 => 'a', 2 => 'b', 3 => 'c']
```

---

## ขั้นตอนที่ 125: Type Hints ขั้นสูง (PHP 8.x)

```php
<?php
declare(strict_types=1);

// Union Types (PHP 8.0+)
function formatInput(int|float|string $value): string {
    return match(true) {
        is_int($value) => "Integer: $value",
        is_float($value) => sprintf("Float: %.2f", $value),
        default => "String: $value"
    };
}

echo formatInput(42) . "\n";
echo formatInput(3.14) . "\n";
echo formatInput("PHP") . "\n";

// Intersection Types (PHP 8.1+)
interface Printable {
    public function print(): void;
}

interface Serializable {
    public function serialize(): string;
}

function process(Printable&Serializable $item): void {
    $item->print();
    echo $item->serialize();
}

// never return type (PHP 8.1+)
function abort(int $code, string $message): never {
    http_response_code($code);
    echo json_encode(['error' => $message]);
    exit;
}

// Fibers (PHP 8.1+) - Cooperative Multitasking
$fiber = new Fiber(function(): void {
    $value = Fiber::suspend('fiber');  // หยุดและส่งค่า
    echo "Got: $value\n";
    Fiber::suspend('done');
});

$result = $fiber->start();          // เริ่ม fiber
echo "Fiber returned: $result\n";  // fiber

$result = $fiber->resume('hello'); // ส่งค่าเข้า fiber
echo "Fiber returned: $result\n";  // done

// Readonly Properties (PHP 8.1+)
class ImmutablePoint {
    public function __construct(
        public readonly float $x,
        public readonly float $y,
        public readonly float $z = 0.0
    ) {}
    
    public function distanceTo(ImmutablePoint $other): float {
        return sqrt(
            ($this->x - $other->x) ** 2 +
            ($this->y - $other->y) ** 2 +
            ($this->z - $other->z) ** 2
        );
    }
    
    public function withX(float $x): static {
        return new static($x, $this->y, $this->z);
    }
}

$point = new ImmutablePoint(1.0, 2.0, 3.0);
// $point->x = 5.0;  // Error: readonly property

$newPoint = $point->withX(10.0);
echo $point->distanceTo($newPoint) . "\n";

// Enums (PHP 8.1+)
enum Suit: string {
    case Hearts = 'H';
    case Diamonds = 'D';
    case Clubs = 'C';
    case Spades = 'S';
    
    public function color(): string {
        return match($this) {
            self::Hearts, self::Diamonds => 'red',
            self::Clubs, self::Spades => 'black',
        };
    }
    
    public function label(): string {
        return match($this) {
            self::Hearts => '♥',
            self::Diamonds => '♦',
            self::Clubs => '♣',
            self::Spades => '♠',
        };
    }
}

$suit = Suit::Hearts;
echo $suit->value . "\n";   // H
echo $suit->name . "\n";    // Hearts
echo $suit->color() . "\n"; // red
echo $suit->label() . "\n"; // ♥

// Enum ใน match
$message = match($suit) {
    Suit::Hearts, Suit::Diamonds => "Red suit: {$suit->label()}",
    Suit::Clubs, Suit::Spades => "Black suit: {$suit->label()}",
};
echo $message . "\n";

// Enum cases()
$allSuits = Suit::cases();
foreach ($allSuits as $case) {
    echo "{$case->name}: {$case->value} {$case->label()}\n";
}

// Enum from() และ tryFrom()
$heartsSuit = Suit::from('H');    // Suit::Hearts
$unknown = Suit::tryFrom('X');    // null (ไม่ throw)
// $error = Suit::from('X');     // ValueError!
```

---

## ขั้นตอนที่ 126: First-Class Callable Syntax (PHP 8.1+)

```php
<?php
// ก่อน PHP 8.1
$fn = function(int $x): int { return strlen((string)$x); };
$fn = Closure::fromCallable('strlen');
$fn = [SomeClass::class, 'staticMethod'];

// PHP 8.1+ First-Class Callable
$strlen = strlen(...);  // Closure จาก built-in function
echo $strlen("Hello") . "\n";  // 5

$strtoupper = strtoupper(...);
$words = array_map($strtoupper, ['hello', 'world', 'php']);
print_r($words);  // [HELLO, WORLD, PHP]

// Method References
class MathHelper {
    public static function square(int $n): int {
        return $n * $n;
    }
    
    public function double(int $n): int {
        return $n * 2;
    }
}

$square = MathHelper::square(...);
$numbers = array_map($square, [1, 2, 3, 4, 5]);
print_r($numbers);  // [1, 4, 9, 16, 25]

$helper = new MathHelper();
$double = $helper->double(...);
$numbers = array_map($double, [1, 2, 3, 4, 5]);
print_r($numbers);  // [2, 4, 6, 8, 10]

// ใน Pipeline
$pipeline = [
    trim(...),
    strtolower(...),
    fn($s) => str_replace(' ', '-', $s),
];

$slug = array_reduce(
    $pipeline,
    fn($carry, $fn) => $fn($carry),
    "  Hello World PHP  "
);
echo $slug . "\n";  // hello-world-php
```

---

## ขั้นตอนที่ 127: Recursion Patterns

```php
<?php
declare(strict_types=1);

// 1. Tail Recursion (PHP ไม่ optimize แต่เป็น Pattern ที่ดี)
function tailFactorial(int $n, int $acc = 1): int {
    if ($n <= 1) return $acc;
    return tailFactorial($n - 1, $n * $acc);
}

echo tailFactorial(10) . "\n";  // 3628800

// 2. Tree Traversal
class TreeNode {
    public array $children = [];
    
    public function __construct(
        public readonly string $name,
        public readonly int $value = 0
    ) {}
    
    public function addChild(TreeNode $child): void {
        $this->children[] = $child;
    }
}

function buildTree(): TreeNode {
    $root = new TreeNode("root", 1);
    $child1 = new TreeNode("child1", 2);
    $child2 = new TreeNode("child2", 3);
    $grandchild1 = new TreeNode("grandchild1", 4);
    $grandchild2 = new TreeNode("grandchild2", 5);
    
    $child1->addChild($grandchild1);
    $child1->addChild($grandchild2);
    $root->addChild($child1);
    $root->addChild($child2);
    
    return $root;
}

// Depth-First Traversal
function dfs(TreeNode $node, int $depth = 0): void {
    echo str_repeat("  ", $depth) . $node->name . " ({$node->value})\n";
    foreach ($node->children as $child) {
        dfs($child, $depth + 1);
    }
}

// Sum all values
function sumTree(TreeNode $node): int {
    $sum = $node->value;
    foreach ($node->children as $child) {
        $sum += sumTree($child);
    }
    return $sum;
}

$tree = buildTree();
dfs($tree);
echo "Total: " . sumTree($tree) . "\n";

// 3. Flatten Nested Array
function flatten(array $arr): array {
    $result = [];
    foreach ($arr as $item) {
        if (is_array($item)) {
            $result = array_merge($result, flatten($item));
        } else {
            $result[] = $item;
        }
    }
    return $result;
}

$nested = [1, [2, 3], [4, [5, 6]], 7, [8, [9, [10]]]];
print_r(flatten($nested));  // [1,2,3,4,5,6,7,8,9,10]

// 4. Directory Scanner
function scanDirectory(string $dir, int $depth = 0): array {
    $result = [];
    if (!is_dir($dir)) return $result;
    
    foreach (scandir($dir) as $item) {
        if ($item === '.' || $item === '..') continue;
        $path = "$dir/$item";
        if (is_dir($path)) {
            $result[$item] = scanDirectory($path, $depth + 1);
        } else {
            $result[] = $item;
        }
    }
    
    return $result;
}

// 5. Tower of Hanoi
function hanoi(int $n, string $from, string $to, string $via): void {
    if ($n === 1) {
        echo "Move disk 1 from $from to $to\n";
        return;
    }
    hanoi($n - 1, $from, $via, $to);
    echo "Move disk $n from $from to $to\n";
    hanoi($n - 1, $via, $to, $from);
}

echo "Tower of Hanoi (3 disks):\n";
hanoi(3, 'A', 'C', 'B');
```

---

## ขั้นตอนที่ 128: Function Utilities

```php
<?php
// func_get_args, func_num_args
function oldStyleVariadic(): void {
    $args = func_get_args();
    $count = func_num_args();
    echo "Received $count arguments: " . implode(', ', $args) . "\n";
}

oldStyleVariadic(1, 2, 3, "four", 5.0);

// call_user_func, call_user_func_array
function myAdd(int $a, int $b): int {
    return $a + $b;
}

echo call_user_func('myAdd', 3, 4) . "\n";  // 7
echo call_user_func_array('myAdd', [3, 4]) . "\n";  // 7

// function_exists, method_exists
var_dump(function_exists('strlen'));       // true
var_dump(function_exists('myFunction'));   // false

// is_callable
var_dump(is_callable('strlen'));           // true
var_dump(is_callable([new stdClass(), 'method'])); // false
var_dump(is_callable(fn() => 1));         // true

// create_function (deprecated, ใช้ Closure แทน)
// ✅ แทนด้วย:
$add = fn($a, $b) => $a + $b;

// Currying
function curry(callable $fn): Closure {
    $arity = (new ReflectionFunction($fn))->getNumberOfParameters();
    
    $accumulator = function(array $args) use ($fn, $arity, &$accumulator): mixed {
        if (count($args) >= $arity) {
            return $fn(...$args);
        }
        return function() use ($args, $accumulator) {
            return $accumulator(array_merge($args, func_get_args()));
        };
    };
    
    return function() use ($accumulator) {
        return $accumulator(func_get_args());
    };
}

$curriedAdd = curry(fn($a, $b, $c) => $a + $b + $c);
$add5 = $curriedAdd(5);
$add5and3 = $add5(3);
echo $add5and3(2) . "\n";  // 10

// Partial Application
function partial(callable $fn, mixed ...$partial): Closure {
    return function() use ($fn, $partial) {
        $args = array_merge($partial, func_get_args());
        return $fn(...$args);
    };
}

$multiply = fn($a, $b) => $a * $b;
$double = partial($multiply, 2);
$triple = partial($multiply, 3);

echo $double(5) . "\n";  // 10
echo $triple(5) . "\n";  // 15

$numbers = [1, 2, 3, 4, 5];
print_r(array_map($double, $numbers));  // [2, 4, 6, 8, 10]
```

---

## ขั้นตอนที่ 129: Project - Math Library

```php
<?php
declare(strict_types=1);

/**
 * PHP Math Library - Functional Style
 */
class MathLib {
    // Statistics
    public static function mean(array $numbers): float {
        if (empty($numbers)) throw new InvalidArgumentException("Empty array");
        return array_sum($numbers) / count($numbers);
    }
    
    public static function median(array $numbers): float {
        if (empty($numbers)) throw new InvalidArgumentException("Empty array");
        sort($numbers);
        $count = count($numbers);
        $mid = (int)($count / 2);
        
        if ($count % 2 === 0) {
            return ($numbers[$mid - 1] + $numbers[$mid]) / 2;
        }
        return $numbers[$mid];
    }
    
    public static function mode(array $numbers): array {
        $counts = array_count_values($numbers);
        arsort($counts);
        $maxCount = reset($counts);
        return array_keys(array_filter($counts, fn($c) => $c === $maxCount));
    }
    
    public static function variance(array $numbers): float {
        $mean = self::mean($numbers);
        $squaredDiffs = array_map(fn($n) => ($n - $mean) ** 2, $numbers);
        return array_sum($squaredDiffs) / count($numbers);
    }
    
    public static function stdDev(array $numbers): float {
        return sqrt(self::variance($numbers));
    }
    
    // Number Theory
    public static function isPrime(int $n): bool {
        if ($n < 2) return false;
        if ($n < 4) return true;
        if ($n % 2 === 0 || $n % 3 === 0) return false;
        for ($i = 5; $i * $i <= $n; $i += 6) {
            if ($n % $i === 0 || $n % ($i + 2) === 0) return false;
        }
        return true;
    }
    
    public static function primes(int $limit): array {
        return array_filter(range(2, $limit), self::isPrime(...));
    }
    
    public static function gcd(int $a, int $b): int {
        while ($b !== 0) [$a, $b] = [$b, $a % $b];
        return abs($a);
    }
    
    public static function lcm(int $a, int $b): int {
        return abs($a * $b) / self::gcd($a, $b);
    }
    
    // Combinatorics
    public static function permutation(int $n, int $r): int {
        if ($r > $n) return 0;
        return (int)(factorial($n) / factorial($n - $r));
    }
    
    public static function combination(int $n, int $r): int {
        if ($r > $n) return 0;
        return (int)(factorial($n) / (factorial($r) * factorial($n - $r)));
    }
    
    // Functional Operations
    public static function pipe(mixed $value, callable ...$fns): mixed {
        return array_reduce($fns, fn($carry, $fn) => $fn($carry), $value);
    }
    
    public static function map(array $arr, callable $fn): array {
        return array_map($fn, $arr);
    }
    
    public static function filter(array $arr, callable $predicate): array {
        return array_values(array_filter($arr, $predicate));
    }
    
    public static function reduce(array $arr, callable $fn, mixed $initial = null): mixed {
        return array_reduce($arr, $fn, $initial);
    }
}

function factorial(int $n): int {
    return $n <= 1 ? 1 : $n * factorial($n - 1);
}

// Test
$data = [4, 8, 15, 16, 23, 42, 4, 8, 15, 4];

echo "Data: " . implode(', ', $data) . "\n";
echo "Mean: " . MathLib::mean($data) . "\n";
echo "Median: " . MathLib::median($data) . "\n";
echo "Mode: " . implode(', ', MathLib::mode($data)) . "\n";
echo "Std Dev: " . number_format(MathLib::stdDev($data), 2) . "\n";
echo "\nPrimes up to 50: " . implode(', ', MathLib::primes(50)) . "\n";
echo "GCD(48, 18): " . MathLib::gcd(48, 18) . "\n";
echo "LCM(4, 6): " . MathLib::lcm(4, 6) . "\n";
echo "C(5,2): " . MathLib::combination(5, 2) . "\n";

$result = MathLib::pipe(
    [1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
    fn($arr) => MathLib::filter($arr, fn($n) => $n % 2 === 0),
    fn($arr) => MathLib::map($arr, fn($n) => $n ** 2),
    fn($arr) => MathLib::reduce($arr, fn($carry, $n) => $carry + $n, 0)
);
echo "\nSum of squares of even numbers 1-10: $result\n";  // 220
```

---

## 📝 แบบฝึกหัด Part 05

### แบบฝึกหัดที่ 1: String Utilities Library
สร้าง Library ของ String Functions:
- `slugify()` - แปลง string เป็น URL slug
- `truncate()` - ตัด string ตามความยาว
- `wordWrap()` - ตัดคำตามความกว้าง
- `highlight()` - ไฮไลต์คำใน string

### แบบฝึกหัดที่ 2: Functional Pipeline
สร้าง Pipeline สำหรับ Data Processing:
```php
$result = pipeline($data)
    ->filter(fn($x) => $x > 0)
    ->map(fn($x) => $x * 2)
    ->reduce(fn($acc, $x) => $acc + $x, 0)
    ->get();
```

### แบบฝึกหัดที่ 3: Recursive JSON Builder
สร้าง Function สร้าง JSON ซ้อนกัน:
- รองรับ Array, Object, String, Number
- ใช้ Recursion
- แสดงผล Pretty Print

---

## 🎯 สรุป Part 05

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Function Basics | Declaration, Parameters, Return Types |
| Anonymous Functions | Closures, Arrow Functions |
| Higher-Order Functions | map, filter, reduce |
| Type Hints | Union, Intersection, Nullable |
| Recursion | Factorial, Tree, Flatten |
| Currying | Partial Application |
| Enums | PHP 8.1 Enums |

**ถัดไป → Part 06: Arrays ขั้นสูง**
