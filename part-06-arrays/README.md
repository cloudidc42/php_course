# Part 06: Arrays - Indexed, Associative, Multidimensional
## ขั้นตอนที่ 151-180

---

## ขั้นตอนที่ 151: Array Fundamentals

```php
<?php
declare(strict_types=1);

// ========================
// 1. Indexed Arrays
// ========================
$fruits = ['apple', 'banana', 'cherry'];
$numbers = [1, 2, 3, 4, 5];
$mixed = [1, 'two', 3.0, true, null];

// Old syntax
$colors = array('red', 'green', 'blue');

// Auto-increment keys
$arr = [];
$arr[] = 'first';   // key 0
$arr[] = 'second';  // key 1
$arr[] = 'third';   // key 2

// Manual keys
$sparse = [];
$sparse[5] = 'five';
$sparse[10] = 'ten';
$sparse[] = 'auto';  // key 11 (10+1)
print_r($sparse);  // [5=>'five', 10=>'ten', 11=>'auto']

// Range
$range = range(1, 10);
$evenRange = range(2, 20, 2);   // [2,4,6,...,20]
$letters = range('a', 'z');
$reverse = range(10, 1, -1);    // [10,9,8,...,1]

// ========================
// 2. Associative Arrays
// ========================
$person = [
    'name' => 'สมชาย',
    'age' => 25,
    'city' => 'Bangkok',
    'email' => 'somchai@example.com'
];

echo $person['name'] . "\n";

// isset vs array_key_exists
$data = ['key' => null];
var_dump(isset($data['key']));            // false (null = not set)
var_dump(array_key_exists('key', $data)); // true (key exists, value is null)

// Accessing Non-existent Key
echo $person['phone'] ?? 'N/A';  // N/A
// echo $person['phone'];         // Warning: Undefined array key

// ========================
// 3. Multidimensional Arrays
// ========================
$matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
];

echo $matrix[1][2] . "\n";  // 6

$students = [
    [
        'id' => 1,
        'name' => 'Alice',
        'scores' => [85, 90, 78, 92],
        'address' => ['city' => 'Bangkok', 'zip' => '10100']
    ],
    [
        'id' => 2,
        'name' => 'Bob',
        'scores' => [72, 88, 95, 80],
        'address' => ['city' => 'Chiang Mai', 'zip' => '50000']
    ],
];

echo $students[0]['name'] . "\n";            // Alice
echo $students[0]['address']['city'] . "\n"; // Bangkok
echo $students[1]['scores'][2] . "\n";       // 95

// Deep Access
$config = [
    'database' => [
        'connections' => [
            'mysql' => [
                'host' => 'localhost',
                'port' => 3306,
                'name' => 'mydb'
            ]
        ]
    ]
];

$host = $config['database']['connections']['mysql']['host'];
echo $host . "\n";

// Safe deep access
function arrayGet(array $arr, string $key, mixed $default = null): mixed {
    $keys = explode('.', $key);
    $current = $arr;
    foreach ($keys as $k) {
        if (!is_array($current) || !array_key_exists($k, $current)) {
            return $default;
        }
        $current = $current[$k];
    }
    return $current;
}

echo arrayGet($config, 'database.connections.mysql.host') . "\n";
echo arrayGet($config, 'database.connections.pgsql.host', 'N/A') . "\n";
```

---

## ขั้นตอนที่ 152: Array Functions ที่ใช้บ่อย

```php
<?php
// ========================
// Sorting Functions
// ========================
$numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5];

// sort - ascending, reset keys
sort($numbers);
print_r($numbers);

// rsort - descending, reset keys
rsort($numbers);
print_r($numbers);

// asort - ascending, preserve keys
$scores = ['Alice' => 85, 'Bob' => 92, 'Charlie' => 78];
asort($scores);
print_r($scores);  // Charlie, Alice, Bob (แต่ key ยังเดิม)

// arsort - descending, preserve keys
arsort($scores);
print_r($scores);  // Bob, Alice, Charlie

// ksort, krsort - sort by key
$data = ['c' => 3, 'a' => 1, 'b' => 2];
ksort($data);   // [a=>1, b=>2, c=>3]
krsort($data);  // [c=>3, b=>2, a=>1]

// Custom sort
$people = [
    ['name' => 'Charlie', 'age' => 30, 'salary' => 50000],
    ['name' => 'Alice', 'age' => 25, 'salary' => 60000],
    ['name' => 'Bob', 'age' => 35, 'salary' => 45000],
];

// Sort by age ascending
usort($people, fn($a, $b) => $a['age'] <=> $b['age']);

// Sort by multiple fields
usort($people, function($a, $b) {
    $salaryCompare = $b['salary'] <=> $a['salary']; // Desc
    if ($salaryCompare !== 0) return $salaryCompare;
    return strcmp($a['name'], $b['name']);          // Asc
});

// Natural sort (สำหรับ String ที่มีตัวเลข)
$files = ['file10.txt', 'file2.txt', 'file1.txt', 'file20.txt'];
sort($files);    // file1, file10, file2, file20 (ไม่ถูกต้อง)
natsort($files); // file1, file2, file10, file20 (ถูกต้อง)
natcasesort($files); // Case-insensitive

// ========================
// Searching Functions
// ========================
$fruits = ['apple', 'banana', 'cherry', 'apple', 'date'];

// in_array
var_dump(in_array('banana', $fruits));       // true
var_dump(in_array('grape', $fruits));        // false
var_dump(in_array('Apple', $fruits));        // false (case-sensitive)
var_dump(in_array('Apple', $fruits, true));  // false (strict)
var_dump(in_array(0, ['a', 'b']));           // true! (0 == "a")
var_dump(in_array(0, ['a', 'b'], true));     // false (strict fix)

// array_search - returns key
$key = array_search('cherry', $fruits);
echo $key . "\n";  // 2

$key = array_search('grape', $fruits);
var_dump($key);  // bool(false)

// array_keys - ได้ทุก key
print_r(array_keys($fruits));

// หา keys ที่มีค่าเจาะจง
print_r(array_keys($fruits, 'apple'));  // [0, 3]

// array_values - reset keys
$filtered = array_filter($fruits, fn($f) => strlen($f) > 5);
print_r($filtered);           // Original keys
print_r(array_values($filtered)); // Reset to 0,1,2...

// ========================
// Manipulation Functions
// ========================
$arr = [1, 2, 3, 4, 5];

// array_push, array_pop
array_push($arr, 6, 7);  // เพิ่มท้าย
$last = array_pop($arr);  // ลบท้าย, คืนค่า

// array_unshift, array_shift
array_unshift($arr, 0);  // เพิ่มหน้า
$first = array_shift($arr);  // ลบหน้า, คืนค่า

// array_splice - แทรก/ลบ ตรงกลาง
$arr = [1, 2, 3, 4, 5];
$removed = array_splice($arr, 1, 2, [20, 30, 40]);
// $arr = [1, 20, 30, 40, 4, 5]
// $removed = [2, 3]

// array_slice - ตัดส่วนหนึ่ง
$arr = [1, 2, 3, 4, 5];
$slice = array_slice($arr, 1, 3);           // [2, 3, 4]
$slicePreserve = array_slice($arr, 1, 3, true); // [1=>2, 2=>3, 3=>4]

// array_chunk - แบ่ง chunks
$arr = range(1, 10);
$chunks = array_chunk($arr, 3);
// [[1,2,3], [4,5,6], [7,8,9], [10]]

$chunksPreserve = array_chunk($arr, 3, true);
// [[0=>1,1=>2,2=>3], [3=>4,...], ...]

// ========================
// Combining Arrays
// ========================
$a = [1, 2, 3];
$b = [4, 5, 6];

// array_merge - รวม (reindex numeric keys)
$merged = array_merge($a, $b);
print_r($merged);  // [1,2,3,4,5,6]

// + Operator - union (ไม่ override existing keys)
$defaults = ['color' => 'red', 'size' => 'M', 'qty' => 1];
$custom = ['color' => 'blue', 'qty' => 5];
$result = $custom + $defaults;
// ['color' => 'blue', 'size' => 'M', 'qty' => 5]

// array_merge vs + with assoc
$a = ['a' => 1, 'b' => 2];
$b = ['b' => 3, 'c' => 4];
print_r(array_merge($a, $b));  // [a=>1, b=>3, c=>4] (b overridden)
print_r($a + $b);              // [a=>1, b=>2, c=>4] (b kept)

// array_combine - สร้าง assoc จาก 2 arrays
$keys = ['name', 'age', 'city'];
$values = ['Alice', 25, 'Bangkok'];
$combined = array_combine($keys, $values);
print_r($combined);  // ['name'=>'Alice', 'age'=>25, 'city'=>'Bangkok']

// array_zip_key ด้วย array_map
$zipped = array_map(null, $a, $b);

// ========================
// Set Operations
// ========================
$a = [1, 2, 3, 4, 5];
$b = [3, 4, 5, 6, 7];

// Intersection
print_r(array_intersect($a, $b));       // [3, 4, 5]
print_r(array_intersect_key($a, $b));   // By key

// Difference
print_r(array_diff($a, $b));      // [1, 2] (ใน a ไม่มีใน b)
print_r(array_diff($b, $a));      // [6, 7] (ใน b ไม่มีใน a)
print_r(array_diff_key($a, $b));  // By key

// Unique
$dup = [1, 2, 2, 3, 3, 3, 4];
print_r(array_unique($dup));  // [1, 2, 3, 4]
```

---

## ขั้นตอนที่ 153: Array Destructuring และ Pattern

```php
<?php
// List/Destructuring
[$a, $b, $c] = [1, 2, 3];
echo "$a $b $c\n";

// Skip elements
[, $second, , $fourth] = [1, 2, 3, 4];
echo "$second $fourth\n";  // 2 4

// Nested
[[$x, $y], [$z, $w]] = [[1, 2], [3, 4]];
echo "$x $y $z $w\n";  // 1 2 3 4

// Associative
['name' => $name, 'age' => $age] = ['name' => 'Alice', 'age' => 25];
echo "$name: $age\n";

// In foreach
$coords = [[1, 2], [3, 4], [5, 6]];
foreach ($coords as [$x, $y]) {
    echo "($x, $y)\n";
}

$records = [
    ['id' => 1, 'name' => 'Alice', 'score' => 90],
    ['id' => 2, 'name' => 'Bob', 'score' => 85],
];
foreach ($records as ['id' => $id, 'name' => $name, 'score' => $score]) {
    echo "$id: $name = $score\n";
}

// Spread in Array
$first = [1, 2, 3];
$second = [4, 5, 6];
$all = [...$first, ...$second];  // [1,2,3,4,5,6]

// String Keys Spread (PHP 8.1+)
$defaults = ['debug' => false, 'cache' => true, 'timeout' => 30];
$override = ['debug' => true];
$config = [...$defaults, ...$override];
// debug=true, cache=true, timeout=30

// Array Unpacking in const (PHP 8.1+)
class Config {
    const array DEFAULTS = ['timeout' => 30, 'retries' => 3];
}
```

---

## ขั้นตอนที่ 154: Advanced Array Patterns

```php
<?php
// Collection Class Pattern
class Collection {
    private array $items;
    
    public function __construct(array $items = []) {
        $this->items = $items;
    }
    
    public static function of(array $items): static {
        return new static($items);
    }
    
    // Immutable operations (return new instance)
    public function map(callable $fn): static {
        return new static(array_map($fn, $this->items));
    }
    
    public function filter(callable $fn): static {
        return new static(array_values(array_filter($this->items, $fn)));
    }
    
    public function reduce(callable $fn, mixed $initial = null): mixed {
        return array_reduce($this->items, $fn, $initial);
    }
    
    public function sort(callable $fn = null): static {
        $items = $this->items;
        $fn ? usort($items, $fn) : sort($items);
        return new static($items);
    }
    
    public function sortBy(string $key, bool $desc = false): static {
        $items = $this->items;
        usort($items, function($a, $b) use ($key, $desc) {
            $compare = $a[$key] <=> $b[$key];
            return $desc ? -$compare : $compare;
        });
        return new static($items);
    }
    
    public function groupBy(string|callable $key): static {
        $groups = [];
        foreach ($this->items as $item) {
            $groupKey = is_callable($key) ? $key($item) : $item[$key];
            $groups[$groupKey][] = $item;
        }
        return new static($groups);
    }
    
    public function first(callable $fn = null): mixed {
        if ($fn === null) return $this->items[0] ?? null;
        foreach ($this->items as $item) {
            if ($fn($item)) return $item;
        }
        return null;
    }
    
    public function last(callable $fn = null): mixed {
        $items = $fn ? array_values(array_filter($this->items, $fn)) : $this->items;
        return end($items) ?: null;
    }
    
    public function pluck(string $key): static {
        return $this->map(fn($item) => $item[$key] ?? null);
    }
    
    public function unique(): static {
        return new static(array_values(array_unique($this->items)));
    }
    
    public function chunk(int $size): static {
        return new static(array_chunk($this->items, $size));
    }
    
    public function flatten(int $depth = INF): static {
        return new static($this->flattenArray($this->items, $depth));
    }
    
    private function flattenArray(array $arr, int $depth): array {
        $result = [];
        foreach ($arr as $item) {
            if (is_array($item) && $depth > 0) {
                $result = array_merge($result, $this->flattenArray($item, $depth - 1));
            } else {
                $result[] = $item;
            }
        }
        return $result;
    }
    
    public function sum(string $key = null): float|int {
        return $key
            ? array_sum(array_column($this->items, $key))
            : array_sum($this->items);
    }
    
    public function avg(string $key = null): float {
        $count = count($this->items);
        return $count > 0 ? $this->sum($key) / $count : 0;
    }
    
    public function count(): int {
        return count($this->items);
    }
    
    public function all(): array {
        return $this->items;
    }
    
    public function toJson(): string {
        return json_encode($this->items, JSON_UNESCAPED_UNICODE);
    }
    
    public function each(callable $fn): static {
        foreach ($this->items as $key => $item) {
            $fn($item, $key);
        }
        return $this;
    }
    
    public function contains(mixed $value, callable $fn = null): bool {
        if ($fn) return (bool)$this->first($fn);
        return in_array($value, $this->items, true);
    }
    
    public function where(string $key, mixed $value): static {
        return $this->filter(fn($item) => ($item[$key] ?? null) === $value);
    }
    
    public function whereBetween(string $key, mixed $min, mixed $max): static {
        return $this->filter(fn($item) => ($item[$key] ?? null) >= $min 
                                       && ($item[$key] ?? null) <= $max);
    }
}

// Usage
$employees = Collection::of([
    ['name' => 'Alice', 'dept' => 'Engineering', 'salary' => 80000, 'age' => 30],
    ['name' => 'Bob', 'dept' => 'Marketing', 'salary' => 65000, 'age' => 35],
    ['name' => 'Charlie', 'dept' => 'Engineering', 'salary' => 95000, 'age' => 28],
    ['name' => 'Diana', 'dept' => 'HR', 'salary' => 55000, 'age' => 32],
    ['name' => 'Eve', 'dept' => 'Engineering', 'salary' => 75000, 'age' => 27],
]);

// Query-like operations
$result = $employees
    ->where('dept', 'Engineering')
    ->sortBy('salary', desc: true)
    ->pluck('name');

echo "Engineers by salary (desc): ";
$result->each(fn($name) => print("$name "));
echo "\n";

echo "Avg Engineering salary: " . 
    $employees->where('dept', 'Engineering')->avg('salary') . "\n";

$byDept = $employees->groupBy('dept');
echo "\nBy Department:\n";
foreach ($byDept->all() as $dept => $deptEmployees) {
    $count = count($deptEmployees);
    $avgSalary = array_sum(array_column($deptEmployees, 'salary')) / $count;
    echo "  $dept: $count employees, avg salary: " . number_format($avgSalary) . "\n";
}
```

---

## ขั้นตอนที่ 155: SPL Data Structures

```php
<?php
// PHP SPL (Standard PHP Library) Data Structures

// 1. SplStack (LIFO)
$stack = new SplStack();
$stack->push('first');
$stack->push('second');
$stack->push('third');

echo $stack->top() . "\n";   // third
echo $stack->pop() . "\n";   // third
echo $stack->count() . "\n"; // 2

// 2. SplQueue (FIFO)
$queue = new SplQueue();
$queue->enqueue('first');
$queue->enqueue('second');
$queue->enqueue('third');

echo $queue->dequeue() . "\n";  // first
echo $queue->count() . "\n";    // 2

// 3. SplMinHeap, SplMaxHeap
$heap = new SplMinHeap();
$heap->insert(5);
$heap->insert(1);
$heap->insert(3);

while (!$heap->isEmpty()) {
    echo $heap->extract() . " ";  // 1 3 5 (ascending)
}
echo "\n";

// 4. SplDoublyLinkedList
$list = new SplDoublyLinkedList();
$list->push('a');
$list->push('b');
$list->push('c');
$list->unshift('zero');  // เพิ่มหน้า

foreach ($list as $item) {
    echo $item . " ";  // zero a b c
}
echo "\n";

// 5. SplFixedArray - ประหยัด Memory กว่า Array ปกติ
$fixed = new SplFixedArray(5);
$fixed[0] = 'a';
$fixed[1] = 'b';
$fixed[4] = 'e';
echo $fixed->getSize() . "\n";  // 5

// 6. SplPriorityQueue
$pq = new SplPriorityQueue();
$pq->insert('low priority', 1);
$pq->insert('high priority', 10);
$pq->insert('medium priority', 5);

while (!$pq->isEmpty()) {
    echo $pq->extract() . "\n";
    // high priority (10)
    // medium priority (5)
    // low priority (1)
}
```

---

## ขั้นตอนที่ 156: Project - Shopping Cart System

```php
<?php
declare(strict_types=1);

class Product {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly float $price,
        public readonly string $category,
        public readonly int $stock = 0
    ) {}
}

class CartItem {
    public function __construct(
        public readonly Product $product,
        public int $quantity
    ) {}
    
    public function getSubtotal(): float {
        return $this->product->price * $this->quantity;
    }
}

class ShoppingCart {
    private array $items = [];
    private float $discountRate = 0.0;
    
    public function add(Product $product, int $quantity = 1): void {
        if ($quantity <= 0) throw new InvalidArgumentException("Quantity must be positive");
        if ($quantity > $product->stock) {
            throw new RuntimeException("Insufficient stock for {$product->name}");
        }
        
        $id = $product->id;
        if (isset($this->items[$id])) {
            $newQty = $this->items[$id]->quantity + $quantity;
            if ($newQty > $product->stock) {
                throw new RuntimeException("Insufficient stock");
            }
            $this->items[$id]->quantity = $newQty;
        } else {
            $this->items[$id] = new CartItem($product, $quantity);
        }
    }
    
    public function remove(int $productId): void {
        unset($this->items[$productId]);
    }
    
    public function updateQuantity(int $productId, int $quantity): void {
        if (!isset($this->items[$productId])) {
            throw new RuntimeException("Product not in cart");
        }
        if ($quantity <= 0) {
            $this->remove($productId);
            return;
        }
        $this->items[$productId]->quantity = $quantity;
    }
    
    public function setDiscount(float $rate): void {
        $this->discountRate = min(max($rate, 0), 1);
    }
    
    public function getSubtotal(): float {
        return array_sum(array_map(
            fn(CartItem $item) => $item->getSubtotal(),
            $this->items
        ));
    }
    
    public function getDiscount(): float {
        return $this->getSubtotal() * $this->discountRate;
    }
    
    public function getTotal(): float {
        return $this->getSubtotal() - $this->getDiscount();
    }
    
    public function getTotalItems(): int {
        return array_sum(array_column(
            array_map(fn($item) => ['qty' => $item->quantity], $this->items),
            'qty'
        ));
    }
    
    public function getItems(): array {
        return $this->items;
    }
    
    public function isEmpty(): bool {
        return empty($this->items);
    }
    
    public function clear(): void {
        $this->items = [];
        $this->discountRate = 0.0;
    }
    
    public function getSummary(): array {
        return [
            'items' => array_map(fn(CartItem $item) => [
                'id' => $item->product->id,
                'name' => $item->product->name,
                'price' => $item->product->price,
                'quantity' => $item->quantity,
                'subtotal' => $item->getSubtotal()
            ], $this->items),
            'subtotal' => $this->getSubtotal(),
            'discount' => $this->getDiscount(),
            'discount_rate' => $this->discountRate * 100 . '%',
            'total' => $this->getTotal(),
            'items_count' => $this->getTotalItems()
        ];
    }
}

// Test
$products = [
    new Product(1, 'PHP Book', 299.00, 'Books', 50),
    new Product(2, 'Laravel Course', 799.00, 'Courses', 100),
    new Product(3, 'MySQL Cheatsheet', 99.00, 'Books', 200),
    new Product(4, 'VS Code Pro', 499.00, 'Software', 30),
];

$cart = new ShoppingCart();

try {
    $cart->add($products[0], 2);    // 2 PHP Books
    $cart->add($products[1]);       // 1 Laravel Course
    $cart->add($products[2], 3);    // 3 MySQL Cheatsheets
    $cart->setDiscount(0.10);       // 10% discount
    
    $summary = $cart->getSummary();
    
    echo "=== Shopping Cart ===\n";
    foreach ($summary['items'] as $item) {
        printf("%-25s x%d  %10s\n",
            $item['name'],
            $item['quantity'],
            number_format($item['subtotal'], 2)
        );
    }
    echo str_repeat('-', 45) . "\n";
    printf("%-36s %10s\n", "Subtotal:", number_format($summary['subtotal'], 2));
    printf("%-36s %10s\n", "Discount ({$summary['discount_rate']}):", 
           '-' . number_format($summary['discount'], 2));
    printf("%-36s %10s\n", "Total:", number_format($summary['total'], 2));
    echo "\nTotal items: {$summary['items_count']}\n";
    
} catch (RuntimeException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}
```

---

## 🎯 สรุป Part 06

| หัวข้อ | Functions ที่ได้เรียน |
|--------|----------------------|
| Creation | [], array(), range() |
| Sorting | sort, usort, arsort, ksort, natsort |
| Searching | in_array, array_search, array_keys |
| Manipulation | push/pop, splice, slice, chunk |
| Combining | merge, +, combine, intersect, diff |
| Functional | map, filter, reduce, walk |
| Advanced | Collection Pattern, SPL Structures |

**ถัดไป → Part 07: String Functions & Manipulation**
