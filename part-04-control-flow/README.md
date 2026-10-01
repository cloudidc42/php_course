# Part 04: Control Flow
## ขั้นตอนที่ 91-120: if, switch, loops

---

## ขั้นตอนที่ 91: if, elseif, else

```php
<?php
declare(strict_types=1);

// Basic if
$age = 20;
if ($age >= 18) {
    echo "ผู้ใหญ่\n";
}

// if-else
if ($age >= 18) {
    echo "ผู้ใหญ่\n";
} else {
    echo "เยาวชน\n";
}

// if-elseif-else
$score = 75;
if ($score >= 90) {
    $grade = 'A';
} elseif ($score >= 80) {
    $grade = 'B';
} elseif ($score >= 70) {
    $grade = 'C';
} elseif ($score >= 60) {
    $grade = 'D';
} else {
    $grade = 'F';
}
echo "Grade: $grade\n";  // Grade: C

// Alternative Syntax (สำหรับ Template)
$isLoggedIn = true;
?>
<?php if ($isLoggedIn): ?>
    <p>ยินดีต้อนรับ, คุณ!</p>
<?php else: ?>
    <p><a href="/login">เข้าสู่ระบบ</a></p>
<?php endif; ?>

<?php
// Nested if
$user = ['role' => 'admin', 'active' => true];
if ($user['active']) {
    if ($user['role'] === 'admin') {
        echo "Admin Panel\n";
    } else {
        echo "User Dashboard\n";
    }
} else {
    echo "Account Disabled\n";
}

// Early Return Pattern (ดีกว่า Nested if)
function processUser(array $user): string {
    if (!$user['active']) {
        return "Account Disabled";
    }
    
    if ($user['role'] !== 'admin') {
        return "User Dashboard";
    }
    
    return "Admin Panel";
}

echo processUser($user) . "\n";

// Ternary ใน Template
$count = 5;
echo "มี $count " . ($count == 1 ? "รายการ" : "รายการ") . "\n";
echo $count > 0 ? "มีข้อมูล" : "ไม่มีข้อมูล";
echo "\n";
```

---

## ขั้นตอนที่ 92: switch และ match

```php
<?php
// Switch Statement
$day = 'Monday';
switch ($day) {
    case 'Monday':
    case 'Tuesday':
    case 'Wednesday':
    case 'Thursday':
    case 'Friday':
        echo "วันทำงาน\n";
        break;
    case 'Saturday':
    case 'Sunday':
        echo "วันหยุด\n";
        break;
    default:
        echo "ไม่รู้จักวัน\n";
}

// Switch ใช้ == (Loose comparison)
switch ("0") {
    case 0:   echo "integer 0\n"; break;  // Match! ("0" == 0)
    case "0": echo "string 0\n"; break;
}

// Fall-through
$num = 3;
switch ($num) {
    case 1:
        echo "One\n";
    case 2:
        echo "Two\n";
    case 3:
        echo "Three\n";  // จะ fall-through ต่อ
    case 4:
        echo "Four\n";
        break;
    case 5:
        echo "Five\n";
}
// Output: Three, Four (fall-through ถึง break)

// Match Expression (PHP 8.0+)
$status = 'active';
$message = match($status) {
    'active' => 'ใช้งานอยู่',
    'inactive' => 'ปิดใช้งาน',
    'pending' => 'รอการอนุมัติ',
    'banned' => 'ถูกแบน',
    default => 'ไม่รู้จักสถานะ'
};
echo $message . "\n";

// Match ไม่มี Fall-through
// Match ใช้ === (Strict comparison)
$value = "1";
echo match($value) {
    1 => "int 1",      // ไม่ match
    "1" => "str '1'",  // match
    true => "true",
    default => "other"
} . "\n";  // str '1'

// Match หลาย Conditions
$lang = 'TypeScript';
$type = match($lang) {
    'PHP', 'Python', 'Ruby' => 'Backend',
    'JavaScript', 'TypeScript' => 'Frontend/Backend',
    'Swift', 'Kotlin' => 'Mobile',
    'C', 'C++', 'Rust' => 'Systems',
    default => 'Other'
};
echo "$lang: $type\n";

// Match ไม่มี Default → UnhandledMatchError
try {
    $result = match('unknown') {
        'a' => 1,
        'b' => 2,
    };
} catch (\UnhandledMatchError $e) {
    echo "UnhandledMatchError!\n";
}

// Match ใช้งาน Expression
$x = 5;
$category = match(true) {
    $x < 0   => 'negative',
    $x === 0 => 'zero',
    $x < 10  => 'small',
    $x < 100 => 'medium',
    default  => 'large'
};
echo "Category: $category\n";
```

---

## ขั้นตอนที่ 93: Loops - while, do-while

```php
<?php
// while loop
$i = 1;
while ($i <= 5) {
    echo $i . " ";
    $i++;
}
echo "\n";  // 1 2 3 4 5

// ระวัง: Infinite Loop
// while (true) { ... }  // จะวนไม่สิ้นสุด!

// do-while - ทำอย่างน้อย 1 ครั้ง
$i = 1;
do {
    echo $i . " ";
    $i++;
} while ($i <= 5);
echo "\n";  // 1 2 3 4 5

// do-while กับ Input Validation
do {
    $input = readline("กรอกตัวเลข 1-10: ");
    $num = (int)$input;
} while ($num < 1 || $num > 10);
echo "คุณกรอก: $num\n";

// while กับ Database Results
// while ($row = $stmt->fetch(PDO::FETCH_ASSOC)) {
//     echo $row['name'] . "\n";
// }

// Nested while
$i = 1;
while ($i <= 3) {
    $j = 1;
    while ($j <= 3) {
        echo "($i,$j) ";
        $j++;
    }
    echo "\n";
    $i++;
}

// while กับ break/continue
$i = 0;
while (true) {  // Infinite loop ที่ควบคุมด้วย break
    $i++;
    if ($i % 2 === 0) continue;  // ข้าม even numbers
    if ($i > 10) break;          // หยุดเมื่อเกิน 10
    echo $i . " ";               // แสดง odd numbers
}
echo "\n";  // 1 3 5 7 9
```

---

## ขั้นตอนที่ 94: Loops - for, foreach

```php
<?php
// for loop
for ($i = 0; $i < 5; $i++) {
    echo $i . " ";
}
echo "\n";  // 0 1 2 3 4

// Reverse
for ($i = 5; $i > 0; $i--) {
    echo $i . " ";
}
echo "\n";  // 5 4 3 2 1

// Multiple Variables
for ($i = 0, $j = 10; $i < 5; $i++, $j -= 2) {
    echo "i=$i, j=$j\n";
}

// Empty sections (infinite loop)
// for (;;) { ... }

// Nested for (สร้าง Multiplication Table)
echo "ตารางสูตรคูณ:\n";
for ($i = 1; $i <= 5; $i++) {
    for ($j = 1; $j <= 5; $j++) {
        printf("%4d", $i * $j);
    }
    echo "\n";
}

// foreach - วน Array/Object
$fruits = ['apple', 'banana', 'cherry'];

// Value only
foreach ($fruits as $fruit) {
    echo $fruit . "\n";
}

// Key + Value
foreach ($fruits as $index => $fruit) {
    echo "$index: $fruit\n";
}

// Associative Array
$person = [
    'name' => 'สมชาย',
    'age' => 25,
    'city' => 'Bangkok'
];

foreach ($person as $key => $value) {
    echo "$key: $value\n";
}

// Nested foreach
$students = [
    ['name' => 'Alice', 'scores' => [85, 90, 78]],
    ['name' => 'Bob', 'scores' => [72, 88, 95]],
    ['name' => 'Charlie', 'scores' => [90, 85, 92]],
];

foreach ($students as $student) {
    $avg = array_sum($student['scores']) / count($student['scores']);
    printf("%s: avg = %.1f\n", $student['name'], $avg);
}

// foreach กับ Reference
$prices = [100, 200, 300, 400];
foreach ($prices as &$price) {
    $price *= 1.07;  // เพิ่ม 7% VAT
}
unset($price);  // *** สำคัญ: ต้อง unset reference หลัง loop!

print_r($prices);

// list() ใน foreach
$rows = [[1, 'Alice', 25], [2, 'Bob', 30], [3, 'Charlie', 35]];
foreach ($rows as [$id, $name, $age]) {
    echo "ID: $id, Name: $name, Age: $age\n";
}

// Destructuring with Keys (PHP 7.1+)
$people = [
    ['name' => 'Alice', 'age' => 25],
    ['name' => 'Bob', 'age' => 30],
];
foreach ($people as ['name' => $name, 'age' => $age]) {
    echo "$name is $age years old\n";
}

// Object Traversal
class NumberRange implements Iterator {
    private int $current;
    
    public function __construct(
        private int $start,
        private int $end
    ) {
        $this->current = $start;
    }
    
    public function current(): int { return $this->current; }
    public function key(): int { return $this->current - $this->start; }
    public function next(): void { $this->current++; }
    public function rewind(): void { $this->current = $this->start; }
    public function valid(): bool { return $this->current <= $this->end; }
}

$range = new NumberRange(1, 5);
foreach ($range as $key => $value) {
    echo "$key: $value\n";
}
```

---

## ขั้นตอนที่ 95: break, continue, return

```php
<?php
// break - หยุด loop
for ($i = 0; $i < 10; $i++) {
    if ($i == 5) break;
    echo $i . " ";
}
echo "\n";  // 0 1 2 3 4

// break กับ Level Number
for ($i = 0; $i < 3; $i++) {
    for ($j = 0; $j < 3; $j++) {
        if ($i == 1 && $j == 1) break 2;  // หยุด 2 ระดับ
        echo "($i,$j) ";
    }
}
echo "\n";  // (0,0) (0,1) (0,2) (1,0)

// continue - ข้าม Iteration
for ($i = 0; $i < 10; $i++) {
    if ($i % 2 === 0) continue;  // ข้าม even
    echo $i . " ";
}
echo "\n";  // 1 3 5 7 9

// continue กับ Level Number
for ($i = 0; $i < 3; $i++) {
    for ($j = 0; $j < 3; $j++) {
        if ($j == 1) continue 2;  // ข้าม ไปที่ outer loop
        echo "($i,$j) ";
    }
}
echo "\n";  // (0,0) (1,0) (2,0)

// return ใน Loop
function findFirst(array $items, callable $predicate): mixed {
    foreach ($items as $item) {
        if ($predicate($item)) {
            return $item;  // Return ทันทีที่พบ
        }
    }
    return null;
}

$numbers = [1, 5, 3, 7, 2, 8, 4];
$firstEven = findFirst($numbers, fn($n) => $n % 2 === 0);
echo "First even: $firstEven\n";  // 2

// Labeled Loop (PHP ไม่มี label แต่ใช้ break n)
$found = false;
for ($i = 0; $i < 5; $i++) {
    for ($j = 0; $j < 5; $j++) {
        if ($i * $j > 10) {
            $found = true;
            break 2;
        }
    }
}
echo $found ? "Found!" : "Not found" . "\n";
```

---

## ขั้นตอนที่ 96: Exception Handling

```php
<?php
declare(strict_types=1);

// try-catch-finally
function divide(float $a, float $b): float {
    if ($b == 0) {
        throw new DivisionByZeroError("ไม่สามารถหารด้วย 0");
    }
    return $a / $b;
}

try {
    echo divide(10, 2) . "\n";   // 5
    echo divide(10, 0) . "\n";   // Exception!
} catch (DivisionByZeroError $e) {
    echo "Error: " . $e->getMessage() . "\n";
    echo "File: " . $e->getFile() . "\n";
    echo "Line: " . $e->getLine() . "\n";
} finally {
    echo "This always runs\n";  // ทำเสมอ
}

// Multiple catch
function riskyOperation(int $type): string {
    return match($type) {
        1 => throw new InvalidArgumentException("Invalid argument"),
        2 => throw new RuntimeException("Runtime error"),
        3 => throw new \OverflowException("Overflow"),
        default => "Success"
    };
}

try {
    echo riskyOperation(2) . "\n";
} catch (InvalidArgumentException $e) {
    echo "Invalid: " . $e->getMessage() . "\n";
} catch (RuntimeException $e) {
    echo "Runtime: " . $e->getMessage() . "\n";
} catch (\Exception $e) {
    echo "General: " . $e->getMessage() . "\n";
}

// Catch Multiple (PHP 8.0+)
try {
    echo riskyOperation(1) . "\n";
} catch (InvalidArgumentException | RuntimeException $e) {
    echo "Caught: " . $e->getMessage() . "\n";
}

// Custom Exceptions
class ValidationException extends RuntimeException {
    private array $errors;
    
    public function __construct(array $errors) {
        parent::__construct("Validation failed");
        $this->errors = $errors;
    }
    
    public function getErrors(): array {
        return $this->errors;
    }
}

class UserNotFoundException extends RuntimeException {
    public function __construct(int $id) {
        parent::__construct("User #$id not found", 404);
    }
}

// Exception Chaining
try {
    try {
        throw new \PDOException("Database connection failed");
    } catch (\PDOException $e) {
        throw new RuntimeException("Service unavailable", 503, $e);
    }
} catch (RuntimeException $e) {
    echo $e->getMessage() . "\n";
    echo "Caused by: " . $e->getPrevious()->getMessage() . "\n";
}

// Throwable Interface (PHP 7+)
// Exception หรือ Error ทั้งหมด implement Throwable
function safeOperation(callable $fn): mixed {
    try {
        return $fn();
    } catch (\Throwable $e) {
        error_log("Error: " . $e->getMessage());
        return null;
    }
}

$result = safeOperation(fn() => 10 / 2);  // 5
$result = safeOperation(fn() => throw new \Error("Test"));  // null
```

---

## ขั้นตอนที่ 97: goto Statement (ไม่แนะนำ)

```php
<?php
// goto - ข้ามไปยัง Label (ไม่แนะนำ!)
// มีข้อจำกัดมาก: ใช้ได้เฉพาะในบางกรณี

$i = 0;
start:
echo $i . " ";
$i++;
if ($i < 5) goto start;
echo "\n";  // 0 1 2 3 4

// *** ไม่ควรใช้ goto ใน Production Code ***
// ใช้ while, for, function แทน
```

---

## ขั้นตอนที่ 98: Alternative Syntax for HTML Templates

```php
<?php
$items = ['Apple', 'Banana', 'Cherry'];
$showList = true;
$count = count($items);
?>
<!DOCTYPE html>
<html>
<body>

<?php if ($showList): ?>
    <h2>รายการผลไม้ (<?= $count ?> รายการ)</h2>
    <?php if ($count > 0): ?>
        <ul>
        <?php foreach ($items as $index => $item): ?>
            <li><?= $index + 1 ?>. <?= htmlspecialchars($item) ?></li>
        <?php endforeach; ?>
        </ul>
    <?php else: ?>
        <p>ไม่มีรายการ</p>
    <?php endif; ?>
<?php endif; ?>

<?php for ($i = 1; $i <= 3; $i++): ?>
    <p>Loop <?= $i ?></p>
<?php endfor; ?>

<?php
$day = date('N'); // 1=Monday, 7=Sunday
switch ($day):
    case 1:
    case 2:
    case 3:
    case 4:
    case 5:
?>
    <p>วันทำงาน</p>
<?php
        break;
    case 6:
    case 7:
?>
    <p>วันหยุดสุดสัปดาห์</p>
<?php endswitch; ?>

<?php while (false): ?>
    <?php /* เนื้อหาข้างในจะไม่แสดง */ ?>
<?php endwhile; ?>

</body>
</html>
```

---

## ขั้นตอนที่ 99: Generators

```php
<?php
// Generator - สร้าง Iterable แบบ Lazy Evaluation

// ปัญหา: สร้าง Array ขนาดใหญ่
function rangeArray(int $start, int $end): array {
    $result = [];
    for ($i = $start; $i <= $end; $i++) {
        $result[] = $i;
    }
    return $result;  // ใช้ Memory มาก!
}

// แก้ด้วย Generator
function rangeGenerator(int $start, int $end): Generator {
    for ($i = $start; $i <= $end; $i++) {
        yield $i;  // ส่งค่าและหยุดรอ
    }
}

// ใช้งาน (ประหยัด Memory)
foreach (rangeGenerator(1, 1000000) as $num) {
    if ($num > 5) break;
    echo $num . " ";
}
echo "\n";

// Generator ส่ง Key-Value
function indexedGenerator(array $items): Generator {
    foreach ($items as $key => $item) {
        yield $key => $item;
    }
}

// Generator กับ send()
function accumulator(): Generator {
    $total = 0;
    while (true) {
        $value = yield $total;  // ส่งออก $total, รับ $value
        if ($value === null) break;
        $total += $value;
    }
}

$gen = accumulator();
$gen->current();     // Initialize
echo $gen->send(10) . "\n";  // 10
echo $gen->send(20) . "\n";  // 30
echo $gen->send(30) . "\n";  // 60

// yield from - Delegation
function innerGenerator(): Generator {
    yield 1;
    yield 2;
    yield 3;
}

function outerGenerator(): Generator {
    yield 0;
    yield from innerGenerator();  // Delegate
    yield from [4, 5, 6];         // Array ก็ได้
    yield 7;
}

foreach (outerGenerator() as $value) {
    echo $value . " ";
}
echo "\n";  // 0 1 2 3 4 5 6 7

// Fibonacci Generator
function fibonacci(): Generator {
    [$a, $b] = [0, 1];
    while (true) {
        yield $a;
        [$a, $b] = [$b, $a + $b];
    }
}

$fib = fibonacci();
$results = [];
for ($i = 0; $i < 10; $i++) {
    $results[] = $fib->current();
    $fib->next();
}
echo implode(', ', $results) . "\n";  // 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

---

## ขั้นตอนที่ 100: Project - Task Manager

```php
<?php
declare(strict_types=1);

// Task Manager ใช้ Control Flow ทุกประเภท

class Task {
    public readonly int $id;
    private static int $counter = 0;
    
    public function __construct(
        public string $title,
        public string $priority = 'medium',
        public string $status = 'pending',
        public ?string $dueDate = null
    ) {
        self::$counter++;
        $this->id = self::$counter;
    }
    
    public function isOverdue(): bool {
        if ($this->dueDate === null) return false;
        return strtotime($this->dueDate) < time();
    }
    
    public function getPriorityLevel(): int {
        return match($this->priority) {
            'high' => 3,
            'medium' => 2,
            'low' => 1,
            default => 0
        };
    }
}

class TaskManager {
    private array $tasks = [];
    
    public function addTask(Task $task): void {
        $this->tasks[$task->id] = $task;
    }
    
    public function getTasks(
        string $status = 'all',
        string $priority = 'all',
        string $sortBy = 'id'
    ): array {
        // Filter
        $filtered = array_filter($this->tasks, function(Task $task) use ($status, $priority) {
            if ($status !== 'all' && $task->status !== $status) return false;
            if ($priority !== 'all' && $task->priority !== $priority) return false;
            return true;
        });
        
        // Sort
        $sorted = array_values($filtered);
        usort($sorted, function(Task $a, Task $b) use ($sortBy) {
            return match($sortBy) {
                'priority' => $b->getPriorityLevel() <=> $a->getPriorityLevel(),
                'title' => strcmp($a->title, $b->title),
                'dueDate' => ($a->dueDate ?? '') <=> ($b->dueDate ?? ''),
                default => $a->id <=> $b->id
            };
        });
        
        return $sorted;
    }
    
    public function complete(int $id): bool {
        if (!isset($this->tasks[$id])) return false;
        $this->tasks[$id]->status = 'completed';
        return true;
    }
    
    public function getStats(): array {
        $stats = ['total' => 0, 'pending' => 0, 'completed' => 0, 'overdue' => 0];
        
        foreach ($this->tasks as $task) {
            $stats['total']++;
            
            switch ($task->status) {
                case 'pending': $stats['pending']++; break;
                case 'completed': $stats['completed']++; break;
            }
            
            if ($task->isOverdue() && $task->status !== 'completed') {
                $stats['overdue']++;
            }
        }
        
        return $stats;
    }
    
    public function generateReport(): Generator {
        $stats = $this->getStats();
        
        yield "=== รายงาน Task Manager ===";
        yield "Total: {$stats['total']} tasks";
        yield "Pending: {$stats['pending']}";
        yield "Completed: {$stats['completed']}";
        yield "Overdue: {$stats['overdue']}";
        yield "";
        yield "=== Tasks ===";
        
        foreach ($this->getTasks(sortBy: 'priority') as $task) {
            $status = match($task->status) {
                'completed' => '✓',
                'pending' => '○',
                default => '?'
            };
            
            $overdue = $task->isOverdue() && $task->status !== 'completed' ? ' [OVERDUE!]' : '';
            
            yield "$status [{$task->priority}] #{$task->id} {$task->title}{$overdue}";
        }
    }
}

// Test
$manager = new TaskManager();
$manager->addTask(new Task("เขียน Unit Tests", 'high', 'pending', '2024-01-01'));
$manager->addTask(new Task("อัพเดท Documentation", 'medium', 'pending', '2024-12-31'));
$manager->addTask(new Task("Fix Bug #123", 'high', 'completed'));
$manager->addTask(new Task("Review PR #456", 'low', 'pending'));
$manager->addTask(new Task("Deploy to Production", 'high', 'pending', '2024-02-01'));

$manager->complete(3);

foreach ($manager->generateReport() as $line) {
    echo $line . "\n";
}
```

---

## 📝 แบบฝึกหัด Part 04

### แบบฝึกหัดที่ 1: FizzBuzz Extended
เขียน FizzBuzz 1-100:
- Fizz ถ้าหาร 3 ลงตัว
- Buzz ถ้าหาร 5 ลงตัว
- FizzBuzz ถ้าหาร 15 ลงตัว
- Boom ถ้าหาร 7 ลงตัว

### แบบฝึกหัดที่ 2: Grade Calculator
รับคะแนนหลายวิชา คำนวณ GPA:
- Grade A = 4.0, B = 3.0, C = 2.0, D = 1.0, F = 0
- แสดงผลสรุป

### แบบฝึกหัดที่ 3: Number Pyramid
สร้าง Pattern ด้วย Loop:
```
    1
   121
  12321
 1234321
123454321
```

---

## 🎯 สรุป Part 04

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| if/else | Conditional execution |
| switch/match | Multi-way branching |
| while/do-while | Condition-based loops |
| for/foreach | Counter/Collection loops |
| break/continue | Loop control |
| Exceptions | try/catch/finally |
| Generators | Lazy evaluation |

**ถัดไป → Part 05: Functions**
