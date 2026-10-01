# Part 02: Variables, Data Types, Constants
## ขั้นตอนที่ 31-60: ตัวแปรและชนิดข้อมูล

---

## 📋 สิ่งที่จะได้เรียนในบทนี้
- Variables: การประกาศและใช้งาน
- Data Types ทั้ง 10 ประเภท
- Type Juggling และ Type Coercion
- Constants: define() และ const
- Superglobals
- Variable Variables
- Strict Types

---

## ขั้นตอนที่ 31: Variables พื้นฐาน

### กฎการตั้งชื่อ Variable

```php
<?php
// ✅ ถูกต้อง
$name = "PHP";
$_name = "underscore";
$name123 = "number";
$camelCase = "camelCase";
$PascalCase = "PascalCase";
$snake_case = "snake_case";

// ❌ ผิด
// $123abc = "starts with number";
// $my-var = "dash not allowed";
// $my var = "space not allowed";

// Case-sensitive!
$Name = "PHP";
$name = "php";
var_dump($Name === $name);  // bool(false)

// Variable เริ่มต้นด้วย $ ตามด้วย [a-zA-Z_][a-zA-Z0-9_]*
// ตัวอย่างที่ดี:
$userId = 1;
$first_name = "สมชาย";
$isLoggedIn = true;
$MAX_SIZE = 100;

// ตัวอย่างที่ควรหลีกเลี่ยง:
$a = 1;      // ไม่สื่อความหมาย
$temp = 2;   // ไม่ชัดเจน
$data = [];  // กว้างเกินไป
```

### Variable Assignment

```php
<?php
// Simple Assignment
$x = 10;

// Reference Assignment (&)
$a = 10;
$b = &$a;  // $b เป็น Reference ของ $a
$a = 20;
echo $b;   // 20 - เปลี่ยนตามด้วย!

$b = 30;
echo $a;   // 30 - เปลี่ยนตามด้วยเช่นกัน

// Unset reference
unset($b);  // ลบ $b แต่ $a ยังมีอยู่
echo $a;   // 30

// List Assignment
list($x, $y, $z) = [1, 2, 3];
echo "$x, $y, $z\n";  // 1, 2, 3

// Short syntax (PHP 7.1+)
[$first, $second] = ["PHP", "Laravel"];
echo "$first, $second\n";  // PHP, Laravel

// Swap variables
$a = "Hello";
$b = "World";
[$a, $b] = [$b, $a];
echo "$a, $b\n";  // World, Hello

// Skip elements
[, $second, , $fourth] = [1, 2, 3, 4];
echo "$second, $fourth\n";  // 2, 4

// Nested
[[$a, $b], [$c, $d]] = [[1, 2], [3, 4]];
echo "$a $b $c $d\n";  // 1 2 3 4
```

---

## ขั้นตอนที่ 32: Data Types ทั้ง 10 ประเภท

PHP มี Data Types แบ่งเป็น 3 กลุ่ม:

### กลุ่มที่ 1: Scalar Types (4 ประเภท)

```php
<?php
// 1. Boolean
$isTrue = true;
$isFalse = false;
$fromInt = (bool)1;      // true
$fromString = (bool)"";  // false

// Falsy values ใน PHP:
// false, 0, 0.0, "", "0", [], null

var_dump((bool)0);       // bool(false)
var_dump((bool)0.0);     // bool(false)
var_dump((bool)"");      // bool(false)
var_dump((bool)"0");     // bool(false)
var_dump((bool)[]);      // bool(false)
var_dump((bool)null);    // bool(false)

// Truthy: ทุกอย่างที่ไม่ใช่ Falsy
var_dump((bool)1);       // bool(true)
var_dump((bool)-1);      // bool(true)
var_dump((bool)"false"); // bool(true) - string "false" เป็น truthy!
var_dump((bool)[0]);     // bool(true) - array มี element


// 2. Integer
$decimal = 42;
$negative = -42;
$hex = 0x2A;        // 42 (Hexadecimal)
$octal = 052;       // 42 (Octal)
$binary = 0b101010; // 42 (Binary)
$underscore = 1_000_000; // PHP 7.4+ - underscore separator

echo "$decimal, $hex, $octal, $binary, $underscore\n";
// 42, 42, 42, 42, 1000000

// Integer limits
echo PHP_INT_MAX . "\n";  // 9223372036854775807 (64-bit)
echo PHP_INT_MIN . "\n";  // -9223372036854775808
echo PHP_INT_SIZE . "\n"; // 8 bytes

// Overflow -> Float
$big = PHP_INT_MAX + 1;
var_dump($big);  // float(9.2233720368548E+18)


// 3. Float (Double)
$pi = 3.14159265358979;
$scientific = 1.2e5;  // 120000
$negative_exp = 1.5e-3;  // 0.0015

echo PHP_FLOAT_EPSILON . "\n";  // 2.2204460492503E-16
echo PHP_FLOAT_MAX . "\n";      // 1.7976931348623E+308
echo PHP_FLOAT_MIN . "\n";      // 2.2250738585072E-308

// Float Precision Issues!
$a = 0.1;
$b = 0.2;
$sum = $a + $b;
echo $sum . "\n";           // 0.3
var_dump($sum == 0.3);      // bool(false) !!!

// วิธีเปรียบเทียบ Float
var_dump(abs($sum - 0.3) < PHP_FLOAT_EPSILON);  // bool(true)

// bcmath สำหรับ Precision
echo bcadd("0.1", "0.2", 10) . "\n";  // 0.3000000000
echo bcmul("0.1", "3", 10) . "\n";    // 0.3000000000


// 4. String
$single = 'Single quote: ไม่ Parse Variables';
$double = "Double quote: Parse $pi "; // Parse ตัวแปร
$heredoc = <<<EOT
Heredoc: Parse $pi
หลายบรรทัด
EOT;
$nowdoc = <<<'EOT'
Nowdoc: ไม่ Parse $pi
เหมือน Single Quote
EOT;

echo $single . "\n";
echo $double . "\n";
echo $heredoc . "\n";
echo $nowdoc . "\n";

// String Indexing
$str = "Hello";
echo $str[0] . "\n";   // H
echo $str[-1] . "\n";  // o (PHP 7.1+)
echo $str[1] . "\n";   // e

// String ใน PHP เป็น Byte String (ไม่ใช่ Unicode)
$thai = "สวัสดี";
echo strlen($thai) . "\n";       // 18 (bytes, ไม่ใช่ตัวอักษร!)
echo mb_strlen($thai) . "\n";    // 6 (ตัวอักษรที่ถูกต้อง)
```

### กลุ่มที่ 2: Compound Types (4 ประเภท)

```php
<?php
// 5. Array
$indexed = [1, 2, 3, "four", 5.0];
$associative = ["name" => "PHP", "version" => 8];
$mixed = [0 => "zero", "one" => 1, 2, "three" => 3];

// Multi-dimensional
$matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
];

echo $matrix[1][2] . "\n";  // 6

// Array Functions พื้นฐาน
echo count($indexed) . "\n";  // 5
echo sizeof($indexed) . "\n"; // 5 (เหมือนกัน)

// ตรวจสอบ
var_dump(is_array($indexed));  // bool(true)


// 6. Object
class Person {
    public string $name;
    public int $age;
    
    public function __construct(string $name, int $age) {
        $this->name = $name;
        $this->age = $age;
    }
    
    public function greet(): string {
        return "สวัสดี ฉันชื่อ {$this->name} อายุ {$this->age} ปี";
    }
}

$person = new Person("สมชาย", 25);
echo $person->greet() . "\n";
var_dump(is_object($person));  // bool(true)

// stdClass - Generic Object
$obj = new stdClass();
$obj->name = "PHP";
$obj->version = 8;
echo $obj->name . "\n";  // PHP

// Object จาก Array
$data = ["name" => "PHP", "version" => 8];
$obj2 = (object)$data;
echo $obj2->name . "\n";  // PHP


// 7. Callable
// Function ที่สามารถถูกเรียกได้

// Function name as string
function double(int $n): int { return $n * 2; }
$callable = 'double';
echo $callable(5) . "\n";  // 10

// Anonymous function
$triple = function(int $n): int { return $n * 3; };
echo $triple(5) . "\n";  // 15

// Arrow function (PHP 7.4+)
$quadruple = fn(int $n) => $n * 4;
echo $quadruple(5) . "\n";  // 20

// Array callable [object/class, method]
$callable2 = [$person, 'greet'];
echo $callable2() . "\n";

// Invokable object
class Multiplier {
    public function __invoke(int $x, int $factor): int {
        return $x * $factor;
    }
}
$mul = new Multiplier();
echo $mul(6, 7) . "\n";  // 42


// 8. Iterable (PHP 7.1+)
// Array หรือ Object ที่ implement Traversable
function processItems(iterable $items): void {
    foreach ($items as $item) {
        echo $item . "\n";
    }
}

processItems([1, 2, 3]);

// Generator เป็น Iterable ด้วย
function generateNumbers(): Generator {
    yield 1;
    yield 2;
    yield 3;
}
processItems(generateNumbers());
```

### กลุ่มที่ 3: Special Types (2 ประเภท)

```php
<?php
// 9. NULL
$null1 = null;
$null2 = NULL;  // Case-insensitive
$undefined = null;  // ตัวแปรที่ไม่มีค่าจะเป็น null

var_dump(is_null($null1));   // bool(true)
var_dump($null1 === null);   // bool(true)
var_dump(isset($null1));     // bool(false) - isset() returns false for null
var_dump(isset($undefined)); // bool(false)

$value = null;
$value ??= "default";  // Null coalescing assignment
echo $value . "\n";  // "default"


// 10. Resource
// Pointer ไปยัง External Resource (file, database, image)

$file = fopen("test.txt", "w");
var_dump($file);   // resource(5) of type (stream)
var_dump(is_resource($file)); // bool(true)

fwrite($file, "Hello PHP");
fclose($file);

var_dump(is_resource($file)); // bool(false) - ปิดแล้ว

// Database connection เป็น Resource เช่นกัน
// $conn = mysqli_connect(...);
// var_dump($conn); // resource(6) of type (mysqli)
```

---

## ขั้นตอนที่ 33: Type Juggling และ Type Coercion

```php
<?php
// PHP ทำ Type Juggling อัตโนมัติ

// Arithmetic Operations
$result = "10" + 5;       // int(15) - String แปลงเป็น int
$result = "10.5" + 5;     // float(15.5) - String แปลงเป็น float
$result = "10 cats" + 5;  // int(15) + Warning (PHP 8)
$result = "cats" + 5;     // int(5) + Warning

// String Concatenation
$result = 10 . 5;         // string "105"
$result = 10.5 . 5;       // string "10.55"

// Comparison (==) vs Strict Comparison (===)
var_dump(0 == false);     // bool(true) !
var_dump(0 == null);      // bool(true) !
var_dump("" == false);    // bool(true) !
var_dump("" == null);     // bool(true) !
var_dump(null == false);  // bool(true)
var_dump("1" == 1);       // bool(true) - String "1" == int 1
var_dump("01" == 1);      // bool(true)
var_dump("10" == "1e1");  // bool(true) !!! "1e1" = 10.0

// PHP 8 เปลี่ยนแปลง: String vs Number Comparison
// PHP 7: "0" == false → true
// PHP 8: ถ้า string ไม่ใช่ numeric string จะไม่แปลง
var_dump(0 == "foo");     // PHP 7: true, PHP 8: false !
var_dump(0 == "");        // PHP 7: true, PHP 8: false !

// Strict Comparison (===) ตรวจสอบ Type ด้วย
var_dump(0 === false);    // bool(false) ✓
var_dump("1" === 1);      // bool(false) ✓
var_dump(1 === 1);        // bool(true) ✓
var_dump("PHP" === "PHP"); // bool(true) ✓

// !! สำคัญมาก: ใช้ === เสมอถ้าไม่แน่ใจ !!

// Type Coercion ใน Function Calls
function add(int $a, int $b): int {
    return $a + $b;
}

// Without strict_types
echo add(1.9, 2.9) . "\n";  // 3 (แปลง float เป็น int โดย truncate)
echo add("1", "2") . "\n";  // 3 (แปลง string เป็น int)

// With strict_types=1 (ต้องอยู่บรรทัดแรกของไฟล์)
// declare(strict_types=1);
// add(1.9, 2.9);  // TypeError!
// add("1", "2"); // TypeError!
```

---

## ขั้นตอนที่ 34: Constants

```php
<?php
// 1. define() - Global scope
define('MAX_SIZE', 100);
define('APP_NAME', 'My PHP App');
define('DEBUG', true);
define('COLORS', ['red', 'green', 'blue']);  // Array constant (PHP 7+)

echo MAX_SIZE . "\n";
echo APP_NAME . "\n";
echo COLORS[0] . "\n";

// ไม่มี $ นำหน้า!
// ไม่สามารถเปลี่ยนค่าได้
// MAX_SIZE = 200;  // ❌ Error


// 2. const - Class/Namespace scope (หรือ Global)
const VERSION = '1.0.0';
const DB_HOST = 'localhost';

// const ต่างจาก define:
// - const กำหนดได้ที่ top-level ของ file หรือ class
// - const ไม่สามารถใช้ใน function, if, loop ได้
// - define() ใช้ได้ทุกที่

// const ใน Class
class Config {
    const MAX_ATTEMPTS = 3;
    const ALLOWED_TYPES = ['jpg', 'png', 'gif'];
    
    // PHP 8.3: Typed class constants
    const string APP_NAME = 'My App';
    const int VERSION = 8;
    const array FEATURES = ['oop', 'closure', 'generator'];
}

echo Config::MAX_ATTEMPTS . "\n";
echo Config::APP_NAME . "\n";
print_r(Config::FEATURES);


// 3. Magic Constants (ค่าที่เปลี่ยนตาม Context)
echo __LINE__ . "\n";       // เลขบรรทัดปัจจุบัน
echo __FILE__ . "\n";       // Path ของไฟล์ปัจจุบัน (absolute)
echo __DIR__ . "\n";        // Directory ของไฟล์ปัจจุบัน
echo __FUNCTION__ . "\n";   // ชื่อ Function ปัจจุบัน
echo __CLASS__ . "\n";      // ชื่อ Class ปัจจุบัน
echo __TRAIT__ . "\n";      // ชื่อ Trait ปัจจุบัน
echo __METHOD__ . "\n";     // ชื่อ Class::Method ปัจจุบัน
echo __NAMESPACE__ . "\n";  // Namespace ปัจจุบัน

// ใช้ใน Real Code
function loadFile(string $filename): string {
    $path = __DIR__ . '/' . $filename;
    if (!file_exists($path)) {
        throw new RuntimeException("ไม่พบไฟล์: $path");
    }
    return file_get_contents($path);
}


// 4. PHP Predefined Constants
echo PHP_EOL;           // \n (Linux) หรือ \r\n (Windows)
echo PHP_MAXPATHLEN . "\n";  // ความยาว path สูงสุด
echo DIRECTORY_SEPARATOR . "\n"; // / (Linux) หรือ \ (Windows)
echo PATH_SEPARATOR . "\n";  // : (Linux) หรือ ; (Windows)

// Error Constants
echo E_ALL . "\n";       // 32767
echo E_ERROR . "\n";     // 1
echo E_WARNING . "\n";   // 2
echo E_NOTICE . "\n";    // 8

// Math Constants
echo M_PI . "\n";     // 3.14159...
echo M_E . "\n";      // 2.71828...
echo M_SQRT2 . "\n";  // 1.41421...
echo INF . "\n";      // INF
echo NAN . "\n";      // NAN
var_dump(is_nan(NAN));   // bool(true)
var_dump(is_infinite(INF)); // bool(true)
var_dump(is_finite(42.0));  // bool(true)
```

---

## ขั้นตอนที่ 35: Superglobals

```php
<?php
// Superglobals เข้าถึงได้ทุก Scope

// 1. $_SERVER - ข้อมูล Server และ Request
echo $_SERVER['PHP_SELF'] . "\n";      // /path/to/script.php
echo $_SERVER['SERVER_NAME'] . "\n";   // localhost
echo $_SERVER['HTTP_HOST'] . "\n";     // localhost:8080
echo $_SERVER['REQUEST_METHOD'] . "\n"; // GET, POST, PUT, etc.
echo $_SERVER['REQUEST_URI'] . "\n";   // /page?param=value
echo $_SERVER['DOCUMENT_ROOT'] . "\n"; // /var/www/html
echo $_SERVER['REMOTE_ADDR'] . "\n";   // IP ของ Client
echo $_SERVER['HTTP_USER_AGENT'] . "\n"; // Browser info
echo $_SERVER['SCRIPT_FILENAME'] . "\n"; // Full path ของ script

// ตรวจสอบ HTTPS
$isHttps = isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off';
$isHttps = $isHttps || (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) 
                        && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https');


// 2. $_GET - Query String Parameters
// URL: /search?q=php&page=1&category=tutorial
$query = $_GET['q'] ?? '';
$page = (int)($_GET['page'] ?? 1);
$category = $_GET['category'] ?? 'all';

// ตรวจสอบก่อนใช้งาน
$search = filter_input(INPUT_GET, 'q', FILTER_SANITIZE_SPECIAL_CHARS);


// 3. $_POST - Form Data
// <form method="POST">
$username = $_POST['username'] ?? '';
$password = $_POST['password'] ?? '';

// Validate
$username = filter_var($_POST['username'] ?? '', FILTER_SANITIZE_SPECIAL_CHARS);
$email = filter_var($_POST['email'] ?? '', FILTER_VALIDATE_EMAIL);


// 4. $_FILES - Uploaded Files
if (isset($_FILES['avatar'])) {
    $file = $_FILES['avatar'];
    /*
    Array
    (
        [name] => photo.jpg       // ชื่อไฟล์ต้นฉบับ
        [type] => image/jpeg      // MIME type
        [tmp_name] => /tmp/phpXXX // Temp path
        [error] => 0              // Error code
        [size] => 12345           // ขนาดไฟล์ bytes
    )
    */
    
    if ($file['error'] === UPLOAD_ERR_OK) {
        $uploadDir = '/var/www/uploads/';
        $filename = basename($file['name']);
        move_uploaded_file($file['tmp_name'], $uploadDir . $filename);
    }
}


// 5. $_COOKIE
// ตั้งค่า Cookie
setcookie('user_pref', 'dark_mode', [
    'expires' => time() + (86400 * 30), // 30 วัน
    'path' => '/',
    'domain' => 'example.com',
    'secure' => true,     // HTTPS only
    'httponly' => true,   // ป้องกัน JavaScript อ่าน
    'samesite' => 'Strict' // ป้องกัน CSRF
]);

$preference = $_COOKIE['user_pref'] ?? 'light_mode';


// 6. $_SESSION
session_start();

$_SESSION['user_id'] = 1;
$_SESSION['username'] = 'john';
$_SESSION['logged_in'] = true;

// ตรวจสอบ Login
function isLoggedIn(): bool {
    return isset($_SESSION['logged_in']) && $_SESSION['logged_in'] === true;
}

// Logout
function logout(): void {
    $_SESSION = [];
    if (ini_get('session.use_cookies')) {
        $params = session_get_cookie_params();
        setcookie(session_name(), '', time() - 42000,
            $params['path'], $params['domain'],
            $params['secure'], $params['httponly']
        );
    }
    session_destroy();
}


// 7. $_REQUEST - GET + POST + COOKIE (ไม่แนะนำใช้)
// $value = $_REQUEST['key'];  // อันตราย!
// ควรใช้ $_GET, $_POST แยกกันชัดเจนกว่า


// 8. $_ENV - Environment Variables
echo $_ENV['HOME'] ?? 'ไม่มี HOME' . "\n";
echo getenv('PATH') . "\n";

// ตั้งค่าใน .env file แล้วโหลดด้วย dotenv
// $db_host = $_ENV['DB_HOST'];
// $db_pass = $_ENV['DB_PASSWORD'];


// 9. $GLOBALS - ตัวแปร Global ทั้งหมด
$globalVar = "I'm global";

function testGlobal(): void {
    // ไม่สามารถเข้าถึง $globalVar โดยตรง
    echo $GLOBALS['globalVar'] . "\n";  // ✓
}

testGlobal();
```

---

## ขั้นตอนที่ 36: Variable Variables

```php
<?php
// Variable Variables - ตัวแปรที่ชื่อเป็นตัวแปร

$varName = 'hello';
$$varName = 'world';
echo $hello . "\n";  // world
echo $$varName . "\n";  // world

// Practical use: Dynamic property access
$field = 'name';
$data = ['name' => 'PHP', 'version' => 8];
echo $data[$field] . "\n";  // PHP

// Variable Variables ใน Loop
$colors = ['red', 'green', 'blue'];
foreach ($colors as $color) {
    $$color = "I am {$color}";
}
echo $red . "\n";    // I am red
echo $green . "\n";  // I am green

// ระวัง: Variable Variables อาจทำให้ Code อ่านยาก
// แนะนำให้ใช้ Array หรือ Object แทน
```

---

## ขั้นตอนที่ 37: Null Safety (PHP 8.0+)

```php
<?php
// Nullsafe Operator (?->)
class User {
    public ?Address $address = null;
}

class Address {
    public ?City $city = null;
}

class City {
    public string $name = 'Bangkok';
    
    public function getZipCode(): string {
        return '10100';
    }
}

$user = new User();

// PHP 7 - ต้องตรวจสอบทุกขั้น
if ($user !== null && $user->address !== null && $user->address->city !== null) {
    echo $user->address->city->name;
}

// PHP 8 - Nullsafe Operator
echo $user?->address?->city?->name ?? 'ไม่มีข้อมูล';
echo $user?->address?->city?->getZipCode() ?? 'ไม่มีรหัสไปรษณีย์';

// Null Coalescing (??)
$config = [];
$host = $config['database']['host'] ?? 'localhost';
$port = $config['database']['port'] ?? 3306;

// Null Coalescing Assignment (??=)
$data = [];
$data['hits'] ??= 0;
$data['hits']++;
echo $data['hits'] . "\n";  // 1

// Nullsafe + Method Chaining
$zipCode = $user?->address?->city?->getZipCode();
```

---

## ขั้นตอนที่ 38: Strict Types

```php
<?php
declare(strict_types=1);  // ต้องอยู่บรรทัดแรก!

// เมื่อเปิด strict_types PHP จะไม่แปลง Type อัตโนมัติ

function add(int $a, int $b): int {
    return $a + $b;
}

echo add(1, 2) . "\n";  // 3 ✓

// add("1", "2");  // TypeError: must be int
// add(1.5, 2.5);  // TypeError: must be int

// Typed Properties (PHP 7.4+)
class Product {
    public int $id;
    public string $name;
    public float $price;
    public bool $inStock = true;
    public ?string $description = null;
    
    public function __construct(int $id, string $name, float $price) {
        $this->id = $id;
        $this->name = $name;
        $this->price = $price;
    }
}

$product = new Product(1, "PHP Book", 299.99);
// $product->price = "expensive";  // TypeError!

// Union Types (PHP 8.0+)
function processInput(int|string $input): string {
    if (is_int($input)) {
        return "Number: $input";
    }
    return "String: $input";
}

echo processInput(42) . "\n";     // Number: 42
echo processInput("PHP") . "\n";  // String: PHP

// Intersection Types (PHP 8.1+)
interface Stringable {
    public function __toString(): string;
}

interface Countable {
    public function count(): int;
}

// function process(Stringable&Countable $obj): string
// ต้อง implement ทั้ง 2 interfaces

// Never Return Type (PHP 8.1+)
function throwException(string $message): never {
    throw new Exception($message);
    // ไม่มี return - function จะ throw เสมอ
}

// Enum (PHP 8.1+)
enum Status: string {
    case Active = 'active';
    case Inactive = 'inactive';
    case Pending = 'pending';
    
    public function label(): string {
        return match($this) {
            Status::Active => 'ใช้งาน',
            Status::Inactive => 'ไม่ใช้งาน',
            Status::Pending => 'รอดำเนินการ',
        };
    }
}

$status = Status::Active;
echo $status->value . "\n";   // active
echo $status->label() . "\n"; // ใช้งาน
echo $status->name . "\n";    // Active

// Typed class constants (PHP 8.3)
class AppConfig {
    const string VERSION = '1.0.0';
    const int MAX_USERS = 1000;
    const float TAX_RATE = 0.07;
    const bool DEBUG = false;
    const array ALLOWED_ORIGINS = ['example.com', 'api.example.com'];
}
```

---

## ขั้นตอนที่ 39: Variable Scopes

```php
<?php
// Global Scope
$globalVar = "ฉันอยู่ใน Global Scope";

function testScope(): void {
    // ไม่สามารถเข้าถึง $globalVar โดยตรง
    // echo $globalVar;  // Warning: Undefined variable

    // วิธีที่ 1: ใช้ global keyword
    global $globalVar;
    echo $globalVar . "\n";  // ✓

    // วิธีที่ 2: ใช้ $GLOBALS
    echo $GLOBALS['globalVar'] . "\n";  // ✓
}

testScope();

// Local Scope
function outer(): void {
    $outerVar = "ฉันอยู่ใน outer()";
    
    function inner(): void {
        // ไม่สามารถเข้าถึง $outerVar
        // echo $outerVar;  // Warning!
    }
    
    inner();
}

// Closure Scope
$x = 10;
$closure = function() use ($x) {  // Capture by value
    echo $x . "\n";
};

$x = 20;
$closure();  // 10 (ค่าตอน capture)

$closureRef = function() use (&$x) {  // Capture by reference
    echo $x . "\n";
};

$x = 30;
$closureRef();  // 30 (ค่าปัจจุบัน)

// Static Variables
function counter(): int {
    static $count = 0;  // Initialize ครั้งแรกเท่านั้น
    return ++$count;
}

echo counter() . "\n";  // 1
echo counter() . "\n";  // 2
echo counter() . "\n";  // 3

// Static ใน Class
class Singleton {
    private static ?self $instance = null;
    private static int $callCount = 0;
    
    private function __construct() {}
    
    public static function getInstance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    public static function getCallCount(): int {
        self::$callCount++;
        return self::$callCount;
    }
}
```

---

## ขั้นตอนที่ 40: Project - Data Type Showcase

```php
<?php
// data_types.php - แสดงชนิดข้อมูลทั้งหมด

declare(strict_types=1);

class DataTypeShowcase {
    public function demo(): void {
        $this->showScalars();
        $this->showArrays();
        $this->showObjects();
        $this->showSpecial();
    }
    
    private function showScalars(): void {
        echo "=== Scalar Types ===\n\n";
        
        // Boolean
        $bool = true;
        echo "Boolean: " . ($bool ? 'true' : 'false') . "\n";
        echo "Type: " . gettype($bool) . "\n\n";
        
        // Integer
        $int = PHP_INT_MAX;
        echo "Integer: " . number_format($int) . "\n";
        echo "Type: " . gettype($int) . "\n";
        echo "Size: " . PHP_INT_SIZE . " bytes\n\n";
        
        // Float
        $float = M_PI;
        echo "Float (PI): " . $float . "\n";
        echo "Type: " . gettype($float) . "\n";
        printf("Formatted: %.10f\n\n", $float);
        
        // String
        $str = "สวัสดี PHP!";
        echo "String: $str\n";
        echo "Length (bytes): " . strlen($str) . "\n";
        echo "Length (chars): " . mb_strlen($str) . "\n";
        echo "Type: " . gettype($str) . "\n\n";
    }
    
    private function showArrays(): void {
        echo "=== Array Types ===\n\n";
        
        // Indexed
        $indexed = range(1, 5);
        echo "Indexed: ";
        print_r($indexed);
        
        // Associative
        $assoc = [
            'name' => 'PHP',
            'version' => 8.3,
            'type' => 'Server-side'
        ];
        echo "\nAssociative: ";
        print_r($assoc);
        
        // Multidimensional
        $multi = array_chunk(range(1, 9), 3);
        echo "\nMultidimensional (3x3 Matrix):";
        foreach ($multi as $row) {
            echo "\n  " . implode(" ", $row);
        }
        echo "\n\n";
    }
    
    private function showObjects(): void {
        echo "=== Object Types ===\n\n";
        
        $obj = new stdClass();
        $obj->name = "Dynamic Object";
        $obj->created = date('Y-m-d');
        $obj->version = 1.0;
        
        echo "Object:\n";
        print_r($obj);
        echo "Type: " . gettype($obj) . "\n";
        echo "Class: " . get_class($obj) . "\n\n";
    }
    
    private function showSpecial(): void {
        echo "=== Special Types ===\n\n";
        
        // NULL
        $null = null;
        echo "NULL value: ";
        var_dump($null);
        
        // Resource
        $file = fopen('/dev/null', 'r');
        echo "Resource type: " . get_resource_type($file) . "\n";
        fclose($file);
        
        echo "\n";
    }
}

$showcase = new DataTypeShowcase();
$showcase->demo();
```

---

## 📝 แบบฝึกหัด Part 02

### แบบฝึกหัดที่ 1: Type Detective
เขียน function `detectType($value)` ที่แสดงรายละเอียดของ value:
- Type
- Length/Size (ถ้ามี)
- Truthy/Falsy
- JSON representation

### แบบฝึกหัดที่ 2: Temperature Converter
สร้างโปรแกรมแปลงอุณหภูมิ:
- Celsius ↔ Fahrenheit ↔ Kelvin
- ใช้ Constants กำหนดสูตรแปลง
- รับค่าจาก Form

### แบบฝึกหัดที่ 3: Config System
สร้าง Configuration System:
- ใช้ Constants สำหรับ App Config
- ใช้ Array สำหรับ Dynamic Config
- รองรับ Environment Variables

---

## 🎯 สรุป Part 02

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Variables | การตั้งชื่อ, Assignment, Reference |
| Scalar Types | bool, int, float, string |
| Compound Types | array, object, callable, iterable |
| Special Types | null, resource |
| Type Juggling | Type Coercion, == vs === |
| Constants | define(), const, Magic Constants |
| Superglobals | $_GET, $_POST, $_SESSION, $_SERVER |
| Strict Types | declare(strict_types=1) |

**ถัดไป → Part 03: Operators ทุกประเภท**
