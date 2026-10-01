# Part 03: Operators ทุกประเภท
## ขั้นตอนที่ 61-90: Operators in PHP

---

## ขั้นตอนที่ 61: Arithmetic Operators

```php
<?php
declare(strict_types=1);

// Basic Arithmetic
$a = 10;
$b = 3;

echo $a + $b . "\n";   // 13 (Addition)
echo $a - $b . "\n";   // 7  (Subtraction)
echo $a * $b . "\n";   // 30 (Multiplication)
echo $a / $b . "\n";   // 3.3333... (Division)
echo $a % $b . "\n";   // 1  (Modulo)
echo $a ** $b . "\n";  // 1000 (Exponentiation - PHP 5.6+)

// Integer Division
echo intdiv(10, 3) . "\n";   // 3
echo intdiv(-10, 3) . "\n";  // -3

// Modulo กับตัวเลข Negative
echo 10 % 3 . "\n";   // 1
echo -10 % 3 . "\n";  // -1 (สัญลักษณ์ตาม Dividend)
echo 10 % -3 . "\n";  // 1 (สัญลักษณ์ตาม Dividend)

// Floating Point Division
echo 7 / 2 . "\n";    // 3.5
echo 7 % 2 . "\n";    // 1

// Increment / Decrement
$x = 5;
echo $x++ . "\n";  // 5 (Post-increment: return แล้วค่อย increment)
echo $x . "\n";    // 6

echo ++$x . "\n";  // 7 (Pre-increment: increment แล้วค่อย return)
echo $x . "\n";    // 7

echo $x-- . "\n";  // 7 (Post-decrement)
echo $x . "\n";    // 6

echo --$x . "\n";  // 5 (Pre-decrement)

// ++ -- กับ String
$str = 'a';
echo ++$str . "\n";  // b
echo ++$str . "\n";  // c

$str = 'z';
echo ++$str . "\n";  // aa

$str = 'Az';
echo ++$str . "\n";  // Ba

$str = 'Zz';
echo ++$str . "\n";  // AAa

// *** -- ไม่ทำงานกับ String! ***
$str = 'b';
$str--;             // ไม่เปลี่ยนแปลง
echo $str . "\n";  // b (ยังคงเป็น b)
```

---

## ขั้นตอนที่ 62: Comparison Operators

```php
<?php
// Equal (==) - เปรียบเทียบค่า (Type Coercion)
var_dump(1 == 1);      // true
var_dump(1 == "1");    // true (coercion)
var_dump(1 == true);   // true (coercion)
var_dump(0 == false);  // true (coercion)
var_dump(0 == null);   // true (coercion) ← อันตราย!
var_dump("" == null);  // true (coercion)
var_dump("php" == true); // true (non-empty string = true)
var_dump("0" == false);  // true

// Identical (===) - เปรียบเทียบค่า + Type
var_dump(1 === 1);      // true
var_dump(1 === "1");    // false ✓ (int vs string)
var_dump(1 === true);   // false ✓
var_dump(0 === false);  // false ✓
var_dump(null === false); // false ✓

// Not Equal (!=, <>)
var_dump(1 != 2);   // true
var_dump(1 <> 2);   // true (เหมือน !=)
var_dump(1 != "1"); // false (coercion)

// Not Identical (!==)
var_dump(1 !== "1"); // true ✓

// Less/Greater
var_dump(1 < 2);   // true
var_dump(2 > 1);   // true
var_dump(1 <= 1);  // true
var_dump(1 >= 2);  // false

// Spaceship Operator (<=> ) PHP 7+
// Returns -1, 0, 1
echo (1 <=> 2) . "\n";  // -1 (1 น้อยกว่า 2)
echo (2 <=> 2) . "\n";  // 0  (เท่ากัน)
echo (3 <=> 2) . "\n";  // 1  (3 มากกว่า 2)

echo ("a" <=> "b") . "\n";  // -1
echo ("b" <=> "a") . "\n";  // 1

// ใช้ใน usort
$numbers = [3, 1, 4, 1, 5, 9, 2, 6];
usort($numbers, fn($a, $b) => $a <=> $b);
print_r($numbers);  // [1, 1, 2, 3, 4, 5, 6, 9]

// Sort Objects
$people = [
    ['name' => 'Charlie', 'age' => 30],
    ['name' => 'Alice', 'age' => 25],
    ['name' => 'Bob', 'age' => 35],
];

usort($people, fn($a, $b) => $a['age'] <=> $b['age']);
foreach ($people as $p) {
    echo "{$p['name']}: {$p['age']}\n";
}
// Alice: 25, Charlie: 30, Bob: 35

// String Comparison
// PHP compares strings character by character (ASCII values)
var_dump("abc" == "abc");  // true
var_dump("abc" < "abd");   // true (c < d)
var_dump("abc" > "ab");    // true (longer)
var_dump("10" > "9");      // false! (string comparison: "1" < "9")
var_dump(10 > 9);          // true  (numeric comparison)

// ใช้ strcmp() สำหรับ String comparison ที่ชัดเจน
echo strcmp("abc", "abc") . "\n";  // 0
echo strcmp("abc", "abd") . "\n";  // -1
echo strcmp("abd", "abc") . "\n";  // 1

// Case-insensitive
echo strcasecmp("ABC", "abc") . "\n";  // 0
```

---

## ขั้นตอนที่ 63: Logical Operators

```php
<?php
// Logical AND (&&, and)
var_dump(true && true);   // true
var_dump(true && false);  // false
var_dump(false && true);  // false
var_dump(false && false); // false

// Logical OR (||, or)
var_dump(true || false);  // true
var_dump(false || false); // false

// Logical NOT (!, not)
var_dump(!true);   // false
var_dump(!false);  // true
var_dump(!!true);  // true (double negation)

// Logical XOR (xor)
var_dump(true xor true);   // false
var_dump(true xor false);  // true
var_dump(false xor false); // false

// ความแตกต่าง && vs and (Precedence)
$a = true;
$b = false;

$result1 = $a && $b;  // $result1 = (true && false) = false
$result2 = $a and $b; // ($result2 = true) and false = true!

var_dump($result1);  // bool(false)
var_dump($result2);  // bool(true) ← แปลก!

// แนะนำใช้ && และ || เสมอ เพราะ Precedence ชัดเจนกว่า

// Short-circuit Evaluation
function logAndReturn(string $label, bool $value): bool {
    echo "$label evaluating...\n";
    return $value;
}

// && - ถ้าซ้ายเป็น false ไม่ evaluate ขวา
$result = logAndReturn("Left", false) && logAndReturn("Right", true);
// แสดงแค่ "Left evaluating..."

echo "\n";

// || - ถ้าซ้ายเป็น true ไม่ evaluate ขวา
$result = logAndReturn("Left", true) || logAndReturn("Right", false);
// แสดงแค่ "Left evaluating..."

echo "\n";

// ประโยชน์ของ Short-circuit
$user = null;
// PHP 7 style
$name = $user && $user->name ? $user->name : 'Guest';

// PHP 8 style (Nullsafe + Null Coalescing)
$name = $user?->name ?? 'Guest';

// การใช้ && สำหรับ Guard Clause
function readFile(string $path): string|false {
    return file_exists($path) && is_readable($path) 
        ? file_get_contents($path)
        : false;
}

// การใช้ || สำหรับ Default Value
function getConfig(string $key): mixed {
    static $config = ['debug' => false, 'version' => '1.0'];
    return $config[$key] ?? null;
}

$debug = getConfig('debug') || false;
```

---

## ขั้นตอนที่ 64: String Operators

```php
<?php
// Concatenation (.)
$firstName = "สมชาย";
$lastName = "ใจดี";
$fullName = $firstName . " " . $lastName;
echo $fullName . "\n";  // สมชาย ใจดี

// Concatenation Assignment (.=)
$message = "สวัสดี";
$message .= " คุณ";
$message .= $firstName;
$message .= "!";
echo $message . "\n";  // สวัสดี คุณสมชาย!

// String Repetition (ใช้ str_repeat)
$line = str_repeat("-", 30);
echo $line . "\n";  // ------------------------------

$stars = str_repeat("* ", 5);
echo $stars . "\n";  // * * * * *

// String in Double Quotes - Variable Interpolation
$name = "PHP";
$version = 8.3;

echo "ฉันชอบ $name!\n";
echo "PHP $version มาแล้ว!\n";
echo "PHP {$name}Developer\n";  // ใช้ {} เพื่อกั้น variable
echo "PHP ${name}!\n";          // หรือใช้ ${}

// Array Access ใน Double Quotes
$colors = ['red', 'green', 'blue'];
echo "สีแรกคือ {$colors[0]}\n";
echo "สีแรกคือ $colors[0]\n";  // ก็ได้เช่นกัน (แต่ไม่แนะนำ)

$person = ['name' => 'สมชาย'];
echo "ชื่อ: {$person['name']}\n";  // ต้องใช้ {} สำหรับ associative

// Object Property ใน Double Quotes
$obj = new stdClass();
$obj->name = "PHP";
echo "Language: {$obj->name}\n";

// String Comparison Operators (ใช้ Function แทน)
$str1 = "Hello";
$str2 = "World";

echo strcmp($str1, $str2) . "\n";     // < 0 (H < W)
echo strtolower($str1) . "\n";        // hello
echo strtoupper($str1) . "\n";        // HELLO
echo strlen($str1) . "\n";            // 5
echo strrev($str1) . "\n";            // olleH
echo str_repeat($str1, 3) . "\n";     // HelloHelloHello
echo str_word_count($str1) . "\n";    // 1
```

---

## ขั้นตอนที่ 65: Bitwise Operators

```php
<?php
// Bitwise AND (&)
echo (0b1010 & 0b1100) . "\n";  // 8 = 0b1000
// 1010
// 1100
// ----
// 1000

// Bitwise OR (|)
echo (0b1010 | 0b1100) . "\n";  // 14 = 0b1110

// Bitwise XOR (^)
echo (0b1010 ^ 0b1100) . "\n";  // 6 = 0b0110

// Bitwise NOT (~)
echo (~0b1010) . "\n";  // -11 (Two's complement)

// Left Shift (<<)
echo (1 << 4) . "\n";   // 16 (1 * 2^4)
echo (5 << 2) . "\n";   // 20 (5 * 4)

// Right Shift (>>)
echo (16 >> 4) . "\n";  // 1 (16 / 2^4)
echo (20 >> 2) . "\n";  // 5 (20 / 4)

// Practical: Permissions System (เหมือน Linux chmod)
const READ    = 0b100;  // 4
const WRITE   = 0b010;  // 2
const EXECUTE = 0b001;  // 1

// กำหนด permission
$userPerms = READ | WRITE;  // 6 = 0b110
$adminPerms = READ | WRITE | EXECUTE;  // 7 = 0b111
$guestPerms = READ;  // 4 = 0b100

// ตรวจสอบ permission
function hasPermission(int $userPerms, int $requiredPerm): bool {
    return ($userPerms & $requiredPerm) === $requiredPerm;
}

var_dump(hasPermission($userPerms, READ));    // true
var_dump(hasPermission($userPerms, WRITE));   // true
var_dump(hasPermission($userPerms, EXECUTE)); // false

// เพิ่ม permission
$userPerms |= EXECUTE;  // เพิ่ม EXECUTE
var_dump(hasPermission($userPerms, EXECUTE)); // true

// ลบ permission
$userPerms &= ~EXECUTE;  // ลบ EXECUTE
var_dump(hasPermission($userPerms, EXECUTE)); // false

// Toggle permission
$userPerms ^= EXECUTE;   // Toggle (เพิ่ม/ลบ)

// Flags Pattern
class UserRole {
    const NONE     = 0;
    const READ     = 1 << 0;  // 1
    const CREATE   = 1 << 1;  // 2
    const UPDATE   = 1 << 2;  // 4
    const DELETE   = 1 << 3;  // 8
    const ADMIN    = 1 << 4;  // 16
    const ALL      = self::READ | self::CREATE | self::UPDATE | self::DELETE;
    
    private int $permissions;
    
    public function __construct(int $permissions = self::NONE) {
        $this->permissions = $permissions;
    }
    
    public function can(int $permission): bool {
        return ($this->permissions & $permission) === $permission;
    }
    
    public function grant(int $permission): void {
        $this->permissions |= $permission;
    }
    
    public function revoke(int $permission): void {
        $this->permissions &= ~$permission;
    }
}

$role = new UserRole(UserRole::READ | UserRole::CREATE);
var_dump($role->can(UserRole::READ));    // true
var_dump($role->can(UserRole::DELETE));  // false
$role->grant(UserRole::DELETE);
var_dump($role->can(UserRole::DELETE));  // true
```

---

## ขั้นตอนที่ 66: Assignment Operators

```php
<?php
// Basic Assignment
$x = 10;

// Compound Assignment
$x += 5;   echo $x . "\n";  // 15 (x = x + 5)
$x -= 3;   echo $x . "\n";  // 12 (x = x - 3)
$x *= 2;   echo $x . "\n";  // 24 (x = x * 2)
$x /= 4;   echo $x . "\n";  // 6  (x = x / 4)
$x %= 4;   echo $x . "\n";  // 2  (x = x % 4)
$x **= 3;  echo $x . "\n";  // 8  (x = x ** 3)

// String
$str = "Hello";
$str .= " World";
echo $str . "\n";  // Hello World

// Bitwise
$flags = 0b1010;
$flags &= 0b1100;  echo decbin($flags) . "\n";  // 1000
$flags = 0b1010;
$flags |= 0b1100;  echo decbin($flags) . "\n";  // 1110
$flags = 0b1010;
$flags ^= 0b1100;  echo decbin($flags) . "\n";  // 110
$flags = 1;
$flags <<= 3;      echo $flags . "\n";           // 8
$flags >>= 2;      echo $flags . "\n";           // 2

// Null Coalescing Assignment (PHP 7.4+)
$data = [];
$data['count'] ??= 0;
echo $data['count'] . "\n";  // 0

$data['count'] ??= 100;  // ไม่เปลี่ยน เพราะมีค่าแล้ว (0)
echo $data['count'] . "\n";  // 0

// Chained Assignment
$a = $b = $c = 10;
echo "$a $b $c\n";  // 10 10 10

// Destructuring Assignment
$coordinates = [10.5, 20.3, 5.7];
[$x, $y, $z] = $coordinates;
echo "x=$x, y=$y, z=$z\n";

// Skip elements
[, $middle, ] = [1, 2, 3];
echo "Middle: $middle\n";  // 2

// With keys
$person = ['name' => 'Alice', 'age' => 30, 'city' => 'Bangkok'];
['name' => $name, 'age' => $age] = $person;
echo "$name is $age years old\n";
```

---

## ขั้นตอนที่ 67: Conditional/Ternary Operators

```php
<?php
// Ternary Operator (? :)
$age = 20;
$status = $age >= 18 ? "ผู้ใหญ่" : "เยาวชน";
echo $status . "\n";  // ผู้ใหญ่

// Nested Ternary (ไม่แนะนำ)
$score = 75;
$grade = $score >= 90 ? 'A' 
       : ($score >= 80 ? 'B' 
       : ($score >= 70 ? 'C' 
       : ($score >= 60 ? 'D' : 'F')));
echo "Grade: $grade\n";  // Grade: C

// ใช้ match expression แทน (PHP 8.0+)
$grade = match(true) {
    $score >= 90 => 'A',
    $score >= 80 => 'B',
    $score >= 70 => 'C',
    $score >= 60 => 'D',
    default => 'F'
};
echo "Grade: $grade\n";

// Elvis Operator (?:) - Short Ternary
$name = null;
$displayName = $name ?: "ไม่ระบุชื่อ";
echo $displayName . "\n";  // ไม่ระบุชื่อ

$name = "สมชาย";
$displayName = $name ?: "ไม่ระบุชื่อ";
echo $displayName . "\n";  // สมชาย

// Null Coalescing (??)
$config = ['debug' => false];
$debug = $config['debug'] ?? true;  // false (ค่ามีอยู่)
$host = $config['host'] ?? 'localhost';  // 'localhost' (ค่าไม่มี)

// ความต่าง ?: กับ ??
$value = 0;
echo ($value ?: "default") . "\n";   // "default" (0 เป็น falsy)
echo ($value ?? "default") . "\n";   // "0" (?? ตรวจสอบแค่ null)

$value = null;
echo ($value ?: "default") . "\n";   // "default"
echo ($value ?? "default") . "\n";   // "default"

$value = "";
echo ($value ?: "default") . "\n";   // "default" ("" เป็น falsy)
echo ($value ?? "default") . "\n";   // "" (ไม่ใช่ null)

// Match Expression (PHP 8.0+) - ดีกว่า switch
$lang = 'PHP';
$description = match($lang) {
    'PHP' => 'Server-side scripting',
    'JavaScript', 'TypeScript' => 'Frontend/Backend',
    'Python' => 'Data Science, AI',
    'Go', 'Rust' => 'Systems Programming',
    default => 'Unknown language'
};
echo $description . "\n";

// Match ใช้ Strict Comparison (===)
$value = "1";
echo match($value) {
    1 => "integer 1",       // ไม่ match (strict)
    "1" => "string '1'",    // match!
    true => "boolean true",
    default => "no match"
} . "\n";  // string '1'

// Match กับ No Argument
$x = 5;
$result = match(true) {
    $x < 0 => "negative",
    $x === 0 => "zero",
    $x < 10 => "small positive",
    $x < 100 => "medium positive",
    default => "large positive"
};
echo $result . "\n";  // small positive
```

---

## ขั้นตอนที่ 68: Type Operators

```php
<?php
// instanceof - ตรวจสอบ Type ของ Object
class Animal {
    public string $name;
    public function __construct(string $name) {
        $this->name = $name;
    }
}

class Dog extends Animal {
    public function bark(): string {
        return "Woof!";
    }
}

class Cat extends Animal {
    public function meow(): string {
        return "Meow!";
    }
}

interface Trainable {
    public function train(string $command): bool;
}

class TrainedDog extends Dog implements Trainable {
    public function train(string $command): bool {
        echo "Training: $command\n";
        return true;
    }
}

$dog = new TrainedDog("Rex");

var_dump($dog instanceof Dog);       // true
var_dump($dog instanceof Animal);    // true (inheritance)
var_dump($dog instanceof Trainable); // true (interface)
var_dump($dog instanceof Cat);       // false

// ใช้ instanceof ใน Type Checking
function makeSound(Animal $animal): string {
    if ($animal instanceof Dog) {
        return $animal->bark();
    }
    if ($animal instanceof Cat) {
        return $animal->meow();
    }
    return "...";
}

// PHP 8.0+ - is_a() function
var_dump(is_a($dog, 'Animal'));   // true
var_dump(is_a($dog, 'Dog'));      // true

// get_class(), get_parent_class()
echo get_class($dog) . "\n";         // TrainedDog
echo get_parent_class($dog) . "\n";  // Dog

// is_* Functions
var_dump(is_int(42));         // true
var_dump(is_float(3.14));     // true
var_dump(is_string("PHP"));   // true
var_dump(is_bool(true));      // true
var_dump(is_null(null));      // true
var_dump(is_array([1,2,3]));  // true
var_dump(is_object($dog));    // true
var_dump(is_callable('strlen')); // true
var_dump(is_numeric("42"));   // true
var_dump(is_numeric("42.5")); // true
var_dump(is_numeric("0x1A")); // false (PHP 7+)
```

---

## ขั้นตอนที่ 69: Error Control Operator (@)

```php
<?php
// @ Operator - Suppress errors (ไม่แนะนำใน Production)

// ตัวอย่าง: เปิดไฟล์ที่อาจไม่มี
$file = @fopen("nonexistent.txt", "r");  // ไม่แสดง Warning
if ($file === false) {
    echo "ไม่สามารถเปิดไฟล์ได้\n";
}

// *** ไม่แนะนำใช้ @ เพราะ:
// 1. ซ่อน Error ทำให้ Debug ยาก
// 2. ทำให้ Code ช้าลง
// 3. PHP 8 มี @ ที่ทำงานแตกต่างกับ PHP 7

// ✅ วิธีที่ดีกว่า: ใช้ try/catch
try {
    if (!file_exists("nonexistent.txt")) {
        throw new RuntimeException("ไม่พบไฟล์");
    }
    $content = file_get_contents("nonexistent.txt");
} catch (RuntimeException $e) {
    echo "Error: " . $e->getMessage() . "\n";
}

// ✅ หรือตรวจสอบก่อน
$path = "nonexistent.txt";
if (file_exists($path) && is_readable($path)) {
    $content = file_get_contents($path);
} else {
    $content = "";
}
```

---

## ขั้นตอนที่ 70: Operator Precedence & Associativity

```php
<?php
// Operator Precedence (สูงไปต่ำ)
// 1. clone, new
// 2. ** (right to left)
// 3. ++, --, ~, (cast), @
// 4. instanceof
// 5. !
// 6. *, /, %
// 7. +, -, .
// 8. <<, >>
// 9. <, <=, >, >=
// 10. ==, !=, ===, !==, <=>
// 11. &
// 12. ^
// 13. |
// 14. &&
// 15. ||
// 16. ??
// 17. ? :
// 18. =, +=, -=, ..., ??=
// 19. yield from, yield
// 20. print
// 21. and
// 22. xor
// 23. or

// ตัวอย่าง Precedence
echo 2 + 3 * 4 . "\n";      // 14 (ไม่ใช่ 20)
echo (2 + 3) * 4 . "\n";    // 20 (ใช้ () ควบคุม)

echo 2 ** 3 ** 2 . "\n";    // 512 (right to left: 2 ** (3 ** 2) = 2 ** 9)
echo (2 ** 3) ** 2 . "\n";  // 64

// เปรียบเทียบ && vs and
$a = true;
$b = $a && false;    // $b = false (ถูกต้อง)
$c = $a and false;   // $c = true  (ผิด! = binds tighter)

// Concatenation กับ +
echo 1 + 2 . " hello\n";   // "3 hello" (+ ก่อน .)
echo 1 . 2 + 3 . "\n";     // "15" (. ก่อน + ??? ไม่ใช่!)
// จริงๆ แล้ว: PHP deprecated การผสม . กับ + ในหนึ่ง expression
// ใช้ () เสมอเพื่อความชัดเจน

// ตัวอย่างที่ดี
$price = 100;
$tax = 0.07;
$total = $price + ($price * $tax);
echo "Total: " . $total . "\n";  // Total: 107

// Associativity
// Left Associative: ส่วนใหญ่ (+, -, *, /)
echo 10 - 3 - 2 . "\n";   // 5 = (10 - 3) - 2

// Right Associative: =, **
$a = $b = $c = 5;  // c=5, b=5, a=5 (right to left)
echo "$a $b $c\n";  // 5 5 5

echo 2 ** 3 ** 2 . "\n";   // 512 = 2 ** (3 ** 2) = 2 ** 9
echo (2 ** 3) ** 2 . "\n"; // 64

// Non-associative: <, >, <=, >=, ==, !=, ===, !==
// $a < $b < $c  // ❌ Parse error ใน PHP
// ต้องใช้: ($a < $b) && ($b < $c)

// Practical: Complex Expression
$items = [1, 2, 3, 4, 5];
$total = array_sum($items);
$count = count($items);
$average = $count > 0 ? $total / $count : 0;
$isAboveAverage = fn($x) => $x > $average;
$aboveAverage = array_filter($items, $isAboveAverage);

echo "Average: $average\n";
echo "Above average: " . implode(", ", $aboveAverage) . "\n";
```

---

## ขั้นตอนที่ 71: Spread Operator (...)

```php
<?php
// Spread Operator - PHP 5.6+

// 1. ใน Function Arguments
function sum(int ...$numbers): int {
    return array_sum($numbers);
}

echo sum(1, 2, 3, 4, 5) . "\n";  // 15

$nums = [1, 2, 3, 4, 5];
echo sum(...$nums) . "\n";  // 15

// 2. Array Spread (PHP 7.4+)
$first = [1, 2, 3];
$second = [4, 5, 6];
$merged = [...$first, ...$second];
print_r($merged);  // [1, 2, 3, 4, 5, 6]

// เพิ่มตรงกลาง
$withMiddle = [...$first, 99, ...$second];
print_r($withMiddle);  // [1, 2, 3, 99, 4, 5, 6]

// 3. String Keys (PHP 8.1+)
$defaults = ['color' => 'red', 'size' => 'medium'];
$custom = ['color' => 'blue', ...$defaults]; // Override defaults
// ค่า color จะเป็น 'red' เพราะ $defaults spread ทีหลัง
print_r($custom);

$overridden = [...$defaults, 'color' => 'blue']; // Override after
// ค่า color จะเป็น 'blue'
print_r($overridden);

// 4. Named Arguments (PHP 8.0+)
function createUser(
    string $name,
    int $age = 0,
    string $email = '',
    bool $active = true
): array {
    return compact('name', 'age', 'email', 'active');
}

// ไม่ต้องใส่ตามลำดับ
$user = createUser(
    age: 25,
    name: 'สมชาย',
    email: 'somchai@example.com'
);
print_r($user);

// Named + Positional
$user2 = createUser('สมหญิง', age: 30, active: false);
print_r($user2);
```

---

## ขั้นตอนที่ 72: Project - Expression Evaluator

```php
<?php
declare(strict_types=1);

class Calculator {
    private array $history = [];
    
    public function calculate(float $a, string $op, float $b): float|string {
        $result = match($op) {
            '+' => $a + $b,
            '-' => $a - $b,
            '*' => $a * $b,
            '/' => $b != 0 ? $a / $b : throw new DivisionByZeroError("หารด้วย 0"),
            '%' => $b != 0 ? fmod($a, $b) : throw new DivisionByZeroError("หารด้วย 0"),
            '**' => $a ** $b,
            default => throw new InvalidArgumentException("ไม่รู้จักตัวดำเนินการ: $op")
        };
        
        $this->history[] = [
            'expression' => "$a $op $b",
            'result' => $result,
            'timestamp' => time()
        ];
        
        return $result;
    }
    
    public function getHistory(): array {
        return $this->history;
    }
    
    public function clearHistory(): void {
        $this->history = [];
    }
}

// Test
$calc = new Calculator();

try {
    echo $calc->calculate(10, '+', 5) . "\n";    // 15
    echo $calc->calculate(10, '-', 3) . "\n";    // 7
    echo $calc->calculate(4, '**', 3) . "\n";    // 64
    echo $calc->calculate(10, '/', 0) . "\n";    // Exception!
} catch (DivisionByZeroError $e) {
    echo "Error: " . $e->getMessage() . "\n";
}

// Show history
foreach ($calc->getHistory() as $record) {
    echo "{$record['expression']} = {$record['result']}\n";
}
```

---

## 📝 แบบฝึกหัด Part 03

### แบบฝึกหัดที่ 1: Operator Quiz
เขียนโปรแกรมทดสอบ Operators:
- คำถาม 10 ข้อ
- ให้ผู้ใช้ตอบ
- แสดงคะแนน

### แบบฝึกหัดที่ 2: Permission System
สร้างระบบ Permission ด้วย Bitwise:
- Admin, Editor, Author, Viewer
- CRUD permissions
- ตรวจสอบและแสดงผล

### แบบฝึกหัดที่ 3: Expression Parser
สร้างโปรแกรมรับ Expression เช่น "10 + 5 * 2" และคำนวณ

---

## 🎯 สรุป Part 03

| Operator | ประเภท |
|----------|--------|
| + - * / % ** | Arithmetic |
| == === != !== < > <= >= <=> | Comparison |
| && \|\| ! and or xor | Logical |
| . .= | String |
| & \| ^ ~ << >> | Bitwise |
| = += -= .= ??= | Assignment |
| ?: ?? | Conditional |
| instanceof | Type |
| ... | Spread |

**ถัดไป → Part 04: Control Flow**
