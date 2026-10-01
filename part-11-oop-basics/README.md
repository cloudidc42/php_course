# Part 11: OOP พื้นฐาน - Classes, Objects
## ขั้นตอนที่ 241-270

---

## ขั้นตอนที่ 241: Classes และ Objects พื้นฐาน

```php
<?php
declare(strict_types=1);

// Class Declaration
class BankAccount {
    // Properties
    private string $owner;
    private float $balance;
    private array $transactions = [];
    private static int $totalAccounts = 0;
    
    // Constants
    const MIN_BALANCE = 0.0;
    const MAX_WITHDRAWAL = 50000.0;
    
    // Constructor
    public function __construct(string $owner, float $initialBalance = 0.0) {
        if ($initialBalance < 0) {
            throw new InvalidArgumentException("Initial balance cannot be negative");
        }
        $this->owner = $owner;
        $this->balance = $initialBalance;
        self::$totalAccounts++;
        
        $this->logTransaction('OPEN', $initialBalance);
    }
    
    // Destructor
    public function __destruct() {
        // Cleanup เมื่อ Object ถูก destroy
    }
    
    // Methods
    public function deposit(float $amount): void {
        if ($amount <= 0) {
            throw new InvalidArgumentException("Deposit amount must be positive");
        }
        $this->balance += $amount;
        $this->logTransaction('DEPOSIT', $amount);
    }
    
    public function withdraw(float $amount): void {
        if ($amount <= 0) {
            throw new InvalidArgumentException("Withdrawal amount must be positive");
        }
        if ($amount > self::MAX_WITHDRAWAL) {
            throw new RuntimeException("Exceeds maximum withdrawal limit");
        }
        if ($amount > $this->balance) {
            throw new RuntimeException("Insufficient funds");
        }
        $this->balance -= $amount;
        $this->logTransaction('WITHDRAW', -$amount);
    }
    
    public function transfer(BankAccount $target, float $amount): void {
        $this->withdraw($amount);
        $target->deposit($amount);
    }
    
    // Getters (Accessors)
    public function getOwner(): string {
        return $this->owner;
    }
    
    public function getBalance(): float {
        return $this->balance;
    }
    
    public function getTransactions(): array {
        return $this->transactions;
    }
    
    // Static Method
    public static function getTotalAccounts(): int {
        return self::$totalAccounts;
    }
    
    // Private Helper
    private function logTransaction(string $type, float $amount): void {
        $this->transactions[] = [
            'type' => $type,
            'amount' => $amount,
            'balance' => $this->balance,
            'timestamp' => date('Y-m-d H:i:s')
        ];
    }
    
    // Magic Methods
    public function __toString(): string {
        return sprintf(
            "Account[%s, Balance: %.2f]",
            $this->owner,
            $this->balance
        );
    }
}

// Creating Objects
$account1 = new BankAccount("Alice", 1000.00);
$account2 = new BankAccount("Bob");

$account1->deposit(500.00);
$account1->withdraw(200.00);
$account1->transfer($account2, 300.00);

echo $account1 . "\n";  // Account[Alice, Balance: 1000.00]
echo $account2 . "\n";  // Account[Bob, Balance: 300.00]

echo "Total Accounts: " . BankAccount::getTotalAccounts() . "\n";  // 2

// Print Transactions
foreach ($account1->getTransactions() as $tx) {
    printf("[%s] %s: %.2f (Balance: %.2f)\n",
        $tx['timestamp'], $tx['type'], $tx['amount'], $tx['balance']
    );
}
```

---

## ขั้นตอนที่ 242: Access Modifiers และ Encapsulation

```php
<?php
class Person {
    // public - เข้าถึงได้จากทุกที่
    public string $name;
    
    // protected - เข้าถึงได้จาก class นี้และ subclass
    protected int $age;
    
    // private - เข้าถึงได้เฉพาะ class นี้เท่านั้น
    private string $ssn;
    
    // readonly - ตั้งค่าได้ครั้งเดียว (PHP 8.1+)
    public readonly string $id;
    
    public function __construct(
        string $name, 
        int $age, 
        string $ssn,
        string $id
    ) {
        $this->name = $name;
        $this->age = $age;
        $this->ssn = $ssn;
        $this->id = $id;
    }
    
    public function getAge(): int {
        return $this->age;
    }
    
    public function getSSN(): string {
        // Mask SSN
        return 'XXX-XX-' . substr($this->ssn, -4);
    }
    
    public function isAdult(): bool {
        return $this->age >= 18;
    }
}

// Constructor Promotion (PHP 8.0+)
class Employee {
    public function __construct(
        public readonly int $id,
        public string $name,
        protected float $salary,
        private string $taxId,
        public string $department = 'General'
    ) {}
    
    public function getSalary(): float {
        return $this->salary;
    }
    
    public function raise(float $percent): void {
        $this->salary *= (1 + $percent / 100);
    }
    
    public function getAnnualCost(): float {
        return $this->salary * 12 * 1.3; // +30% benefits
    }
}

$emp = new Employee(1, 'Alice', 80000, 'TX-12345', 'Engineering');
echo $emp->name . "\n";          // Alice
echo $emp->getSalary() . "\n";   // 80000
$emp->raise(10);
echo $emp->getSalary() . "\n";   // 88000
// echo $emp->salary;             // Error: protected
// echo $emp->taxId;              // Error: private
// $emp->id = 2;                  // Error: readonly
```

---

## ขั้นตอนที่ 243: Inheritance

```php
<?php
// Base Class
abstract class Shape {
    protected string $color;
    
    public function __construct(string $color = 'black') {
        $this->color = $color;
    }
    
    // Abstract method - ต้อง implement ใน subclass
    abstract public function area(): float;
    abstract public function perimeter(): float;
    
    // Concrete method - ใช้ร่วมกัน
    public function describe(): string {
        return sprintf(
            "%s: area=%.2f, perimeter=%.2f, color=%s",
            get_class($this),
            $this->area(),
            $this->perimeter(),
            $this->color
        );
    }
    
    // Template Method Pattern
    public function draw(): void {
        echo "Drawing " . get_class($this) . "...\n";
        $this->beforeDraw();
        $this->doDraw();
        $this->afterDraw();
    }
    
    protected function beforeDraw(): void { /* hook */ }
    protected function doDraw(): void { /* default */ }
    protected function afterDraw(): void { /* hook */ }
}

class Circle extends Shape {
    public function __construct(
        private float $radius,
        string $color = 'black'
    ) {
        parent::__construct($color);
    }
    
    public function area(): float {
        return M_PI * $this->radius ** 2;
    }
    
    public function perimeter(): float {
        return 2 * M_PI * $this->radius;
    }
    
    public function getRadius(): float {
        return $this->radius;
    }
    
    protected function doDraw(): void {
        echo "○ (radius: {$this->radius})\n";
    }
}

class Rectangle extends Shape {
    public function __construct(
        private float $width,
        private float $height,
        string $color = 'black'
    ) {
        parent::__construct($color);
    }
    
    public function area(): float {
        return $this->width * $this->height;
    }
    
    public function perimeter(): float {
        return 2 * ($this->width + $this->height);
    }
    
    protected function doDraw(): void {
        echo "□ ({$this->width}x{$this->height})\n";
    }
}

class Square extends Rectangle {
    public function __construct(float $side, string $color = 'black') {
        parent::__construct($side, $side, $color);
    }
    
    public function getSide(): float {
        return $this->area() ** 0.5; // sqrt(area)
    }
}

// Polymorphism
$shapes = [
    new Circle(5, 'red'),
    new Rectangle(4, 6, 'blue'),
    new Square(3, 'green'),
    new Circle(2.5),
];

foreach ($shapes as $shape) {
    echo $shape->describe() . "\n";
    $shape->draw();
}

// instanceof
foreach ($shapes as $shape) {
    if ($shape instanceof Circle) {
        echo "Circle with radius: " . $shape->getRadius() . "\n";
    }
}

// Late Static Binding
class Model {
    protected static string $table = 'models';
    
    public static function getTable(): string {
        return static::$table;  // late static binding
    }
    
    public static function find(int $id): static {
        echo "SELECT * FROM " . static::$table . " WHERE id = $id\n";
        return new static();  // returns correct subclass type
    }
}

class UserModel extends Model {
    protected static string $table = 'users';
}

class PostModel extends Model {
    protected static string $table = 'posts';
}

echo UserModel::getTable() . "\n";  // users
echo PostModel::getTable() . "\n";  // posts

UserModel::find(1);   // SELECT * FROM users WHERE id = 1
PostModel::find(5);   // SELECT * FROM posts WHERE id = 5
```

---

## ขั้นตอนที่ 244: Interfaces และ Traits

```php
<?php
// Interfaces - กำหนด Contract
interface Drawable {
    public function draw(): string;
    public function getColor(): string;
}

interface Resizable {
    public function resize(float $factor): static;
}

interface Exportable {
    public function toArray(): array;
    public function toJson(): string;
}

// Interface สามารถ extend interface อื่นได้
interface Shape extends Drawable, Resizable, Exportable {
    public function area(): float;
    public function perimeter(): float;
}

// Traits - Code Reuse
trait JsonExportable {
    public function toJson(): string {
        return json_encode($this->toArray(), JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
    }
}

trait Timestampable {
    private ?string $createdAt = null;
    private ?string $updatedAt = null;
    
    public function initTimestamps(): void {
        $this->createdAt = date('Y-m-d H:i:s');
        $this->updatedAt = date('Y-m-d H:i:s');
    }
    
    public function touch(): void {
        $this->updatedAt = date('Y-m-d H:i:s');
    }
    
    public function getCreatedAt(): ?string {
        return $this->createdAt;
    }
    
    public function getUpdatedAt(): ?string {
        return $this->updatedAt;
    }
}

trait Colorable {
    private string $color = 'black';
    
    public function getColor(): string {
        return $this->color;
    }
    
    public function setColor(string $color): static {
        $clone = clone $this;
        $clone->color = $color;
        return $clone;
    }
    
    public function withColor(string $color): static {
        return $this->setColor($color);
    }
}

// Class ที่ใช้ Interface + Traits
class Circle implements Shape {
    use JsonExportable, Timestampable, Colorable;
    
    public function __construct(
        private float $radius
    ) {
        $this->initTimestamps();
    }
    
    public function area(): float {
        return M_PI * $this->radius ** 2;
    }
    
    public function perimeter(): float {
        return 2 * M_PI * $this->radius;
    }
    
    public function draw(): string {
        return "Drawing Circle (r={$this->radius}, color={$this->color})";
    }
    
    public function resize(float $factor): static {
        $clone = clone $this;
        $clone->radius *= $factor;
        return $clone;
    }
    
    public function toArray(): array {
        return [
            'type' => 'circle',
            'radius' => $this->radius,
            'area' => round($this->area(), 2),
            'perimeter' => round($this->perimeter(), 2),
            'color' => $this->color,
        ];
    }
}

$circle = new Circle(5.0);
$redCircle = $circle->withColor('red');
$bigRedCircle = $redCircle->resize(2);

echo $circle->draw() . "\n";
echo $redCircle->draw() . "\n";
echo $bigRedCircle->draw() . "\n";
echo $bigRedCircle->toJson() . "\n";

// Trait Conflict Resolution
trait A {
    public function hello(): string {
        return "Hello from A";
    }
}

trait B {
    public function hello(): string {
        return "Hello from B";
    }
}

class C {
    use A, B {
        A::hello insteadof B;  // ใช้ A::hello แทน B::hello
        B::hello as helloB;    // rename B::hello เป็น helloB
    }
}

$c = new C();
echo $c->hello() . "\n";   // Hello from A
echo $c->helloB() . "\n";  // Hello from B

// Abstract Trait
trait Singleton {
    private static ?self $instance = null;
    
    private function __construct() {}
    private function __clone() {}
    
    public static function getInstance(): static {
        if (static::$instance === null) {
            static::$instance = new static();
        }
        return static::$instance;
    }
}

class Database {
    use Singleton;
    
    private \PDO $pdo;
    
    public function connect(string $dsn): void {
        // $this->pdo = new \PDO($dsn);
        echo "Connected to: $dsn\n";
    }
}

$db1 = Database::getInstance();
$db2 = Database::getInstance();
var_dump($db1 === $db2);  // true - same instance
```

---

## ขั้นตอนที่ 245: Magic Methods

```php
<?php
class MagicClass {
    private array $data = [];
    private array $methods = [];
    
    // __get, __set, __isset, __unset
    public function __get(string $name): mixed {
        echo "__get called: $name\n";
        return $this->data[$name] ?? null;
    }
    
    public function __set(string $name, mixed $value): void {
        echo "__set called: $name = $value\n";
        $this->data[$name] = $value;
    }
    
    public function __isset(string $name): bool {
        echo "__isset called: $name\n";
        return isset($this->data[$name]);
    }
    
    public function __unset(string $name): void {
        echo "__unset called: $name\n";
        unset($this->data[$name]);
    }
    
    // __call, __callStatic
    public function __call(string $name, array $args): mixed {
        echo "__call: $name(" . implode(', ', $args) . ")\n";
        if (isset($this->methods[$name])) {
            return ($this->methods[$name])(...$args);
        }
        throw new BadMethodCallException("Method $name not found");
    }
    
    public static function __callStatic(string $name, array $args): mixed {
        echo "__callStatic: $name(" . implode(', ', $args) . ")\n";
        return null;
    }
    
    // __toString
    public function __toString(): string {
        return json_encode($this->data);
    }
    
    // __invoke
    public function __invoke(mixed ...$args): string {
        return "Invoked with: " . implode(', ', $args);
    }
    
    // __clone
    public function __clone(): void {
        echo "Object cloned\n";
        // Deep copy data
        $this->data = array_map(
            fn($v) => is_object($v) ? clone $v : $v,
            $this->data
        );
    }
    
    // __debugInfo
    public function __debugInfo(): array {
        return [
            'data_count' => count($this->data),
            'keys' => array_keys($this->data),
        ];
    }
    
    // Serialization
    public function __serialize(): array {
        return ['data' => $this->data];
    }
    
    public function __unserialize(array $data): void {
        $this->data = $data['data'];
    }
    
    public function addMethod(string $name, callable $fn): void {
        $this->methods[$name] = $fn;
    }
}

$obj = new MagicClass();
$obj->name = "PHP";         // __set
echo $obj->name . "\n";    // __get
var_dump(isset($obj->name)); // __isset
unset($obj->name);           // __unset

$obj->nonExistentMethod(1, 2, 3);  // __call

echo $obj . "\n";  // __toString
echo $obj("a", "b", "c") . "\n";   // __invoke

$clone = clone $obj;  // __clone

// Dynamic Methods
$obj->addMethod('calculate', fn($a, $b) => $a + $b);
echo $obj->calculate(3, 4) . "\n";  // 7

// Serialization
$serialized = serialize($obj);
$unserialized = unserialize($serialized);

// var_dump shows __debugInfo result
var_dump($obj);
```

---

## ขั้นตอนที่ 246: Project - E-Commerce ORM

```php
<?php
declare(strict_types=1);

// Simple ORM-like Pattern

abstract class BaseModel {
    protected static string $table = '';
    protected static array $fillable = [];
    protected static array $guarded = ['id'];
    protected static array $casts = [];
    
    protected array $attributes = [];
    protected array $original = [];
    protected bool $exists = false;
    
    public function __construct(array $attributes = []) {
        $this->fill($attributes);
        $this->original = $this->attributes;
    }
    
    public function fill(array $attributes): static {
        foreach ($attributes as $key => $value) {
            if ($this->isFillable($key)) {
                $this->setAttribute($key, $value);
            }
        }
        return $this;
    }
    
    protected function isFillable(string $key): bool {
        if (in_array($key, static::$guarded)) return false;
        return empty(static::$fillable) || in_array($key, static::$fillable);
    }
    
    public function setAttribute(string $key, mixed $value): void {
        if (isset(static::$casts[$key])) {
            $value = $this->castAttribute($key, $value);
        }
        $this->attributes[$key] = $value;
    }
    
    protected function castAttribute(string $key, mixed $value): mixed {
        return match(static::$casts[$key]) {
            'int', 'integer' => (int)$value,
            'float', 'double' => (float)$value,
            'bool', 'boolean' => (bool)$value,
            'string' => (string)$value,
            'array' => is_string($value) ? json_decode($value, true) : (array)$value,
            'json' => is_string($value) ? json_decode($value, true) : json_encode($value),
            default => $value
        };
    }
    
    public function getAttribute(string $key): mixed {
        return $this->attributes[$key] ?? null;
    }
    
    public function __get(string $key): mixed {
        return $this->getAttribute($key);
    }
    
    public function __set(string $key, mixed $value): void {
        $this->setAttribute($key, $value);
    }
    
    public function __isset(string $key): bool {
        return isset($this->attributes[$key]);
    }
    
    public function isDirty(string $key = null): bool {
        if ($key) return ($this->attributes[$key] ?? null) !== ($this->original[$key] ?? null);
        return $this->attributes !== $this->original;
    }
    
    public function getDirty(): array {
        return array_diff_assoc($this->attributes, $this->original);
    }
    
    public function toArray(): array {
        return $this->attributes;
    }
    
    public function toJson(): string {
        return json_encode($this->toArray(), JSON_UNESCAPED_UNICODE);
    }
    
    public function __toString(): string {
        return $this->toJson();
    }
}

// Concrete Models
class Product extends BaseModel {
    protected static string $table = 'products';
    protected static array $fillable = ['name', 'price', 'category', 'stock', 'description'];
    protected static array $casts = [
        'price' => 'float',
        'stock' => 'int',
        'active' => 'bool',
    ];
    
    public function isInStock(): bool {
        return $this->getAttribute('stock') > 0;
    }
    
    public function applyDiscount(float $percent): float {
        $price = $this->getAttribute('price');
        return $price * (1 - $percent / 100);
    }
}

class Order extends BaseModel {
    protected static string $table = 'orders';
    protected static array $fillable = ['customer_id', 'status', 'items', 'total'];
    protected static array $casts = [
        'customer_id' => 'int',
        'total' => 'float',
        'items' => 'array',
    ];
    
    private array $orderItems = [];
    
    public function addItem(Product $product, int $quantity): void {
        $this->orderItems[] = [
            'product_id' => $product->getAttribute('id'),
            'name' => $product->getAttribute('name'),
            'price' => $product->getAttribute('price'),
            'quantity' => $quantity,
            'subtotal' => $product->getAttribute('price') * $quantity
        ];
        $this->recalculateTotal();
    }
    
    private function recalculateTotal(): void {
        $total = array_sum(array_column($this->orderItems, 'subtotal'));
        $this->setAttribute('total', $total);
        $this->setAttribute('items', $this->orderItems);
    }
    
    public function getItems(): array {
        return $this->orderItems;
    }
}

// Test
$product1 = new Product([
    'id' => 1,
    'name' => 'PHP 8.3 Complete Guide',
    'price' => '299.99',  // จะถูก cast เป็น float
    'category' => 'Books',
    'stock' => '50'       // จะถูก cast เป็น int
]);

$product2 = new Product([
    'id' => 2,
    'name' => 'Laravel Masterclass',
    'price' => '799',
    'stock' => '100'
]);

echo "Product 1: $product1\n";
echo "In Stock: " . ($product1->isInStock() ? 'Yes' : 'No') . "\n";
echo "Discounted Price: " . $product1->applyDiscount(10) . "\n";

$order = new Order(['customer_id' => 1, 'status' => 'pending']);
$order->addItem($product1, 2);
$order->addItem($product2, 1);

echo "\nOrder: $order\n";
echo "Total: " . number_format($order->total, 2) . "\n";
echo "Items: " . count($order->getItems()) . "\n";

// isDirty
$product1->setAttribute('price', 350.00);
var_dump($product1->isDirty('price'));  // true
print_r($product1->getDirty());
```

---

## 🎯 สรุป Part 11

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Classes & Objects | Constructor, Destructor, Methods |
| Access Modifiers | public, protected, private, readonly |
| Constructor Promotion | PHP 8.0+ syntax |
| Inheritance | extends, parent::, abstract |
| Interfaces | Contracts, Multiple interfaces |
| Traits | Code reuse, Conflict resolution |
| Magic Methods | __get, __set, __call, __toString |
| Late Static Binding | static:: vs self:: |

**ถัดไป → Part 12: OOP ขั้นสูง - Design Patterns**
