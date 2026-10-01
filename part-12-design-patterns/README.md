# Part 12: Design Patterns ขั้นสูง
## ขั้นตอนที่ 271-300: Patterns ที่ใช้ในโปรแกรมจริง

---

## ขั้นตอนที่ 271: Creational Patterns

### Singleton
```php
<?php
declare(strict_types=1);

class Database {
    private static ?self $instance = null;
    private \PDO $pdo;
    
    private function __construct() {
        $this->pdo = new \PDO(
            'mysql:host=localhost;dbname=mydb;charset=utf8mb4',
            'user', 'password',
            [\PDO::ATTR_ERRMODE => \PDO::ERRMODE_EXCEPTION]
        );
    }
    
    // ป้องกันการ clone
    private function __clone() {}
    public function __wakeup(): never { throw new \RuntimeException("Cannot unserialize singleton"); }
    
    public static function getInstance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    public function getConnection(): \PDO {
        return $this->pdo;
    }
}

$db = Database::getInstance();
$pdo = $db->getConnection();
```

### Factory Method
```php
<?php
declare(strict_types=1);

interface Logger {
    public function log(string $level, string $message): void;
}

class FileLogger implements Logger {
    public function __construct(private string $filePath) {}
    
    public function log(string $level, string $message): void {
        $entry = sprintf("[%s] [%s] %s\n", date('Y-m-d H:i:s'), strtoupper($level), $message);
        file_put_contents($this->filePath, $entry, FILE_APPEND);
    }
}

class DatabaseLogger implements Logger {
    public function __construct(private \PDO $pdo) {}
    
    public function log(string $level, string $message): void {
        $stmt = $this->pdo->prepare("INSERT INTO logs (level, message, created_at) VALUES (?, ?, NOW())");
        $stmt->execute([$level, $message]);
    }
}

class SlackLogger implements Logger {
    public function __construct(private string $webhookUrl) {}
    
    public function log(string $level, string $message): void {
        // Send to Slack webhook
    }
}

class NullLogger implements Logger {
    public function log(string $level, string $message): void {}
}

// Factory
class LoggerFactory {
    public static function create(string $type, array $config = []): Logger {
        return match($type) {
            'file' => new FileLogger($config['path'] ?? '/tmp/app.log'),
            'database' => new DatabaseLogger($config['pdo'] ?? throw new \InvalidArgumentException('PDO required')),
            'slack' => new SlackLogger($config['webhook'] ?? ''),
            'null' => new NullLogger(),
            default => throw new \InvalidArgumentException("Unknown logger type: {$type}"),
        };
    }
}

// Usage
$logger = LoggerFactory::create('file', ['path' => '/var/log/app.log']);
$logger->log('info', 'Application started');
```

### Abstract Factory
```php
<?php
declare(strict_types=1);

// Products
interface Button {
    public function render(): string;
}

interface Checkbox {
    public function render(): string;
}

// Bootstrap UI
class BootstrapButton implements Button {
    public function __construct(private string $label, private string $type = 'primary') {}
    
    public function render(): string {
        return "<button class=\"btn btn-{$this->type}\">{$this->label}</button>";
    }
}

class BootstrapCheckbox implements Checkbox {
    public function __construct(private string $label) {}
    
    public function render(): string {
        return "<div class=\"form-check\"><input class=\"form-check-input\" type=\"checkbox\"><label class=\"form-check-label\">{$this->label}</label></div>";
    }
}

// Tailwind UI
class TailwindButton implements Button {
    public function __construct(private string $label, private string $color = 'blue') {}
    
    public function render(): string {
        return "<button class=\"bg-{$this->color}-500 text-white px-4 py-2 rounded\">{$this->label}</button>";
    }
}

class TailwindCheckbox implements Checkbox {
    public function __construct(private string $label) {}
    
    public function render(): string {
        return "<label class=\"flex items-center\"><input type=\"checkbox\" class=\"mr-2\">{$this->label}</label>";
    }
}

// Abstract Factory
interface UIFactory {
    public function createButton(string $label): Button;
    public function createCheckbox(string $label): Checkbox;
}

class BootstrapFactory implements UIFactory {
    public function createButton(string $label): Button {
        return new BootstrapButton($label);
    }
    
    public function createCheckbox(string $label): Checkbox {
        return new BootstrapCheckbox($label);
    }
}

class TailwindFactory implements UIFactory {
    public function createButton(string $label): Button {
        return new TailwindButton($label);
    }
    
    public function createCheckbox(string $label): Checkbox {
        return new TailwindCheckbox($label);
    }
}

// Client code
function renderForm(UIFactory $factory): string {
    $button = $factory->createButton('Submit');
    $checkbox = $factory->createCheckbox('I agree to terms');
    
    return $checkbox->render() . "\n" . $button->render();
}

$framework = getenv('UI_FRAMEWORK') === 'tailwind' ? new TailwindFactory() : new BootstrapFactory();
echo renderForm($framework);
```

### Builder
```php
<?php
declare(strict_types=1);

class QueryBuilder {
    private string $table = '';
    private array $selects = ['*'];
    private array $wheres = [];
    private array $bindings = [];
    private array $joins = [];
    private array $orderBys = [];
    private array $groupBys = [];
    private ?string $having = null;
    private ?int $limit = null;
    private int $offset = 0;
    
    public function table(string $table): self {
        $clone = clone $this;
        $clone->table = $table;
        return $clone;
    }
    
    public function select(string ...$columns): self {
        $clone = clone $this;
        $clone->selects = $columns;
        return $clone;
    }
    
    public function where(string $column, string $operator, mixed $value): self {
        $clone = clone $this;
        $clone->wheres[] = "{$column} {$operator} ?";
        $clone->bindings[] = $value;
        return $clone;
    }
    
    public function whereIn(string $column, array $values): self {
        $clone = clone $this;
        $placeholders = implode(',', array_fill(0, count($values), '?'));
        $clone->wheres[] = "{$column} IN ({$placeholders})";
        $clone->bindings = array_merge($clone->bindings, $values);
        return $clone;
    }
    
    public function join(string $table, string $on): self {
        $clone = clone $this;
        $clone->joins[] = "JOIN {$table} ON {$on}";
        return $clone;
    }
    
    public function leftJoin(string $table, string $on): self {
        $clone = clone $this;
        $clone->joins[] = "LEFT JOIN {$table} ON {$on}";
        return $clone;
    }
    
    public function orderBy(string $column, string $direction = 'ASC'): self {
        $clone = clone $this;
        $clone->orderBys[] = "{$column} {$direction}";
        return $clone;
    }
    
    public function groupBy(string ...$columns): self {
        $clone = clone $this;
        $clone->groupBys = $columns;
        return $clone;
    }
    
    public function having(string $condition): self {
        $clone = clone $this;
        $clone->having = $condition;
        return $clone;
    }
    
    public function limit(int $limit): self {
        $clone = clone $this;
        $clone->limit = $limit;
        return $clone;
    }
    
    public function offset(int $offset): self {
        $clone = clone $this;
        $clone->offset = $offset;
        return $clone;
    }
    
    public function build(): array {
        $sql = 'SELECT ' . implode(', ', $this->selects);
        $sql .= ' FROM ' . $this->table;
        
        foreach ($this->joins as $join) {
            $sql .= " {$join}";
        }
        
        if (!empty($this->wheres)) {
            $sql .= ' WHERE ' . implode(' AND ', $this->wheres);
        }
        
        if (!empty($this->groupBys)) {
            $sql .= ' GROUP BY ' . implode(', ', $this->groupBys);
        }
        
        if ($this->having) {
            $sql .= " HAVING {$this->having}";
        }
        
        if (!empty($this->orderBys)) {
            $sql .= ' ORDER BY ' . implode(', ', $this->orderBys);
        }
        
        if ($this->limit !== null) {
            $sql .= " LIMIT {$this->limit}";
        }
        
        if ($this->offset > 0) {
            $sql .= " OFFSET {$this->offset}";
        }
        
        return ['sql' => $sql, 'bindings' => $this->bindings];
    }
    
    public function toSql(): string {
        return $this->build()['sql'];
    }
}

// Usage
$query = (new QueryBuilder())
    ->table('users u')
    ->select('u.id', 'u.name', 'u.email', 'COUNT(o.id) as order_count')
    ->leftJoin('orders o', 'u.id = o.user_id')
    ->where('u.active', '=', 1)
    ->whereIn('u.role', ['admin', 'manager'])
    ->groupBy('u.id')
    ->having('order_count > 5')
    ->orderBy('order_count', 'DESC')
    ->limit(10)
    ->offset(20);

$result = $query->build();
echo $result['sql'];
// SELECT u.id, u.name, u.email, COUNT(o.id) as order_count FROM users u LEFT JOIN orders o ON u.id = o.user_id WHERE u.active = ? AND u.role IN (?,?) GROUP BY u.id HAVING order_count > 5 ORDER BY order_count DESC LIMIT 10 OFFSET 20
```

---

## ขั้นตอนที่ 272: Structural Patterns

### Decorator
```php
<?php
declare(strict_types=1);

interface Cache {
    public function get(string $key): mixed;
    public function set(string $key, mixed $value, int $ttl = 3600): bool;
    public function delete(string $key): bool;
    public function has(string $key): bool;
}

class ArrayCache implements Cache {
    private array $store = [];
    
    public function get(string $key): mixed {
        $item = $this->store[$key] ?? null;
        if (!$item) return null;
        if ($item['expires'] < time()) {
            $this->delete($key);
            return null;
        }
        return $item['value'];
    }
    
    public function set(string $key, mixed $value, int $ttl = 3600): bool {
        $this->store[$key] = ['value' => $value, 'expires' => time() + $ttl];
        return true;
    }
    
    public function delete(string $key): bool {
        unset($this->store[$key]);
        return true;
    }
    
    public function has(string $key): bool {
        return $this->get($key) !== null;
    }
}

// Decorator: Logging Cache
class LoggingCache implements Cache {
    public function __construct(
        private Cache $cache,
        private Logger $logger
    ) {}
    
    public function get(string $key): mixed {
        $value = $this->cache->get($key);
        $this->logger->log('debug', $value !== null ? "Cache HIT: {$key}" : "Cache MISS: {$key}");
        return $value;
    }
    
    public function set(string $key, mixed $value, int $ttl = 3600): bool {
        $this->logger->log('debug', "Cache SET: {$key} (TTL: {$ttl}s)");
        return $this->cache->set($key, $value, $ttl);
    }
    
    public function delete(string $key): bool {
        $this->logger->log('debug', "Cache DELETE: {$key}");
        return $this->cache->delete($key);
    }
    
    public function has(string $key): bool {
        return $this->cache->has($key);
    }
}

// Decorator: Serializing Cache (for complex types)
class SerializingCache implements Cache {
    public function __construct(private Cache $cache) {}
    
    public function get(string $key): mixed {
        $value = $this->cache->get($key);
        return $value !== null ? unserialize($value) : null;
    }
    
    public function set(string $key, mixed $value, int $ttl = 3600): bool {
        return $this->cache->set($key, serialize($value), $ttl);
    }
    
    public function delete(string $key): bool { return $this->cache->delete($key); }
    public function has(string $key): bool { return $this->cache->has($key); }
}

// Stack Decorators
$cache = new LoggingCache(
    new SerializingCache(new ArrayCache()),
    new FileLogger('/var/log/cache.log')
);

$cache->set('user:1', ['id' => 1, 'name' => 'สมชาย']);
$user = $cache->get('user:1');
```

### Repository Pattern
```php
<?php
declare(strict_types=1);

// Domain Entity
class User {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly string $email,
        public readonly string $role = 'user',
        public readonly bool $active = true,
        public readonly \DateTimeImmutable $createdAt = new \DateTimeImmutable(),
    ) {}
}

// Repository Interface
interface UserRepository {
    public function findById(int $id): ?User;
    public function findByEmail(string $email): ?User;
    /** @return User[] */
    public function findAll(int $page = 1, int $perPage = 20): array;
    public function findByRole(string $role): array;
    public function save(User $user): User;
    public function delete(int $id): bool;
    public function count(): int;
}

// PDO Implementation
class PdoUserRepository implements UserRepository {
    public function __construct(private \PDO $pdo) {}
    
    public function findById(int $id): ?User {
        $stmt = $this->pdo->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$id]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $row ? $this->hydrate($row) : null;
    }
    
    public function findByEmail(string $email): ?User {
        $stmt = $this->pdo->prepare("SELECT * FROM users WHERE email = ?");
        $stmt->execute([$email]);
        $row = $stmt->fetch(\PDO::FETCH_ASSOC);
        return $row ? $this->hydrate($row) : null;
    }
    
    public function findAll(int $page = 1, int $perPage = 20): array {
        $offset = ($page - 1) * $perPage;
        $stmt = $this->pdo->prepare("SELECT * FROM users LIMIT ? OFFSET ?");
        $stmt->execute([$perPage, $offset]);
        return array_map([$this, 'hydrate'], $stmt->fetchAll(\PDO::FETCH_ASSOC));
    }
    
    public function findByRole(string $role): array {
        $stmt = $this->pdo->prepare("SELECT * FROM users WHERE role = ?");
        $stmt->execute([$role]);
        return array_map([$this, 'hydrate'], $stmt->fetchAll(\PDO::FETCH_ASSOC));
    }
    
    public function save(User $user): User {
        if ($user->id === 0) {
            return $this->insert($user);
        }
        return $this->update($user);
    }
    
    private function insert(User $user): User {
        $stmt = $this->pdo->prepare(
            "INSERT INTO users (name, email, role, active) VALUES (?, ?, ?, ?)"
        );
        $stmt->execute([$user->name, $user->email, $user->role, $user->active]);
        $id = (int) $this->pdo->lastInsertId();
        return new User($id, $user->name, $user->email, $user->role, $user->active);
    }
    
    private function update(User $user): User {
        $stmt = $this->pdo->prepare(
            "UPDATE users SET name = ?, email = ?, role = ?, active = ? WHERE id = ?"
        );
        $stmt->execute([$user->name, $user->email, $user->role, $user->active, $user->id]);
        return $user;
    }
    
    public function delete(int $id): bool {
        $stmt = $this->pdo->prepare("DELETE FROM users WHERE id = ?");
        return $stmt->execute([$id]);
    }
    
    public function count(): int {
        return (int) $this->pdo->query("SELECT COUNT(*) FROM users")->fetchColumn();
    }
    
    private function hydrate(array $row): User {
        return new User(
            (int) $row['id'],
            $row['name'],
            $row['email'],
            $row['role'],
            (bool) $row['active'],
            new \DateTimeImmutable($row['created_at']),
        );
    }
}

// In-Memory Implementation (for testing)
class InMemoryUserRepository implements UserRepository {
    private array $users = [];
    private int $nextId = 1;
    
    public function findById(int $id): ?User {
        return $this->users[$id] ?? null;
    }
    
    public function findByEmail(string $email): ?User {
        foreach ($this->users as $user) {
            if ($user->email === $email) return $user;
        }
        return null;
    }
    
    public function findAll(int $page = 1, int $perPage = 20): array {
        $all = array_values($this->users);
        return array_slice($all, ($page - 1) * $perPage, $perPage);
    }
    
    public function findByRole(string $role): array {
        return array_values(array_filter($this->users, fn($u) => $u->role === $role));
    }
    
    public function save(User $user): User {
        if ($user->id === 0) {
            $newUser = new User($this->nextId++, $user->name, $user->email, $user->role, $user->active);
            $this->users[$newUser->id] = $newUser;
            return $newUser;
        }
        $this->users[$user->id] = $user;
        return $user;
    }
    
    public function delete(int $id): bool {
        unset($this->users[$id]);
        return true;
    }
    
    public function count(): int {
        return count($this->users);
    }
}
```

---

## ขั้นตอนที่ 273: Behavioral Patterns

### Observer
```php
<?php
declare(strict_types=1);

interface EventInterface {
    public function getName(): string;
    public function getPayload(): array;
}

interface EventListener {
    public function handle(EventInterface $event): void;
}

class Event implements EventInterface {
    public function __construct(
        private string $name,
        private array $payload = [],
    ) {}
    
    public function getName(): string { return $this->name; }
    public function getPayload(): array { return $this->payload; }
}

class EventDispatcher {
    private array $listeners = [];
    
    public function listen(string $event, EventListener|callable $listener): void {
        $this->listeners[$event][] = $listener;
    }
    
    public function dispatch(EventInterface $event): void {
        $listeners = $this->listeners[$event->getName()] ?? [];
        
        foreach ($listeners as $listener) {
            if (is_callable($listener)) {
                $listener($event);
            } else {
                $listener->handle($event);
            }
        }
    }
}

// Listeners
class SendWelcomeEmail implements EventListener {
    public function handle(EventInterface $event): void {
        $user = $event->getPayload()['user'];
        echo "Sending welcome email to {$user['email']}\n";
    }
}

class CreateUserProfile implements EventListener {
    public function handle(EventInterface $event): void {
        $user = $event->getPayload()['user'];
        echo "Creating profile for user #{$user['id']}\n";
    }
}

// Usage
$dispatcher = new EventDispatcher();
$dispatcher->listen('user.registered', new SendWelcomeEmail());
$dispatcher->listen('user.registered', new CreateUserProfile());
$dispatcher->listen('user.registered', function(EventInterface $event) {
    error_log("New user: " . $event->getPayload()['user']['email']);
});

$dispatcher->dispatch(new Event('user.registered', [
    'user' => ['id' => 1, 'email' => 'somchai@example.com'],
]));
```

### Strategy
```php
<?php
declare(strict_types=1);

interface PaymentStrategy {
    public function pay(float $amount): array;
    public function refund(string $transactionId, float $amount): bool;
}

class CreditCardPayment implements PaymentStrategy {
    public function __construct(
        private string $cardNumber,
        private string $expiry,
        private string $cvv,
    ) {}
    
    public function pay(float $amount): array {
        // Process credit card payment
        return [
            'success' => true,
            'transaction_id' => 'CC_' . uniqid(),
            'amount' => $amount,
            'method' => 'credit_card',
        ];
    }
    
    public function refund(string $transactionId, float $amount): bool {
        return true;
    }
}

class PayPalPayment implements PaymentStrategy {
    public function __construct(private string $email) {}
    
    public function pay(float $amount): array {
        return [
            'success' => true,
            'transaction_id' => 'PP_' . uniqid(),
            'amount' => $amount,
            'method' => 'paypal',
        ];
    }
    
    public function refund(string $transactionId, float $amount): bool {
        return true;
    }
}

class PromptPayPayment implements PaymentStrategy {
    public function __construct(private string $phone) {}
    
    public function pay(float $amount): array {
        return [
            'success' => true,
            'transaction_id' => 'PP_TH_' . uniqid(),
            'amount' => $amount,
            'method' => 'promptpay',
        ];
    }
    
    public function refund(string $transactionId, float $amount): bool {
        return false; // PromptPay ไม่รองรับ refund อัตโนมัติ
    }
}

class PaymentProcessor {
    private PaymentStrategy $strategy;
    
    public function setStrategy(PaymentStrategy $strategy): self {
        $this->strategy = $strategy;
        return $this;
    }
    
    public function pay(float $amount): array {
        $result = $this->strategy->pay($amount);
        
        if ($result['success']) {
            $this->logTransaction($result);
        }
        
        return $result;
    }
    
    private function logTransaction(array $transaction): void {
        // Log to database
    }
}

// Usage
$processor = new PaymentProcessor();

// Strategy can be changed at runtime
$processor->setStrategy(new PromptPayPayment('0812345678'));
$result = $processor->pay(1500.00);

if ($result['success']) {
    echo "Payment successful! Transaction: {$result['transaction_id']}\n";
}
```

### Command
```php
<?php
declare(strict_types=1);

interface Command {
    public function execute(): void;
    public function undo(): void;
}

class TextEditor {
    private string $content = '';
    private array $history = [];
    private array $redoStack = [];
    
    public function executeCommand(Command $command): void {
        $command->execute();
        $this->history[] = $command;
        $this->redoStack = []; // Clear redo after new command
    }
    
    public function undo(): void {
        if (empty($this->history)) return;
        
        $command = array_pop($this->history);
        $command->undo();
        $this->redoStack[] = $command;
    }
    
    public function redo(): void {
        if (empty($this->redoStack)) return;
        
        $command = array_pop($this->redoStack);
        $command->execute();
        $this->history[] = $command;
    }
    
    public function getContent(): string { return $this->content; }
    public function setContent(string $content): void { $this->content = $content; }
    public function insertAt(int $pos, string $text): void {
        $this->content = substr($this->content, 0, $pos) . $text . substr($this->content, $pos);
    }
    public function deleteAt(int $pos, int $length): string {
        $deleted = substr($this->content, $pos, $length);
        $this->content = substr($this->content, 0, $pos) . substr($this->content, $pos + $length);
        return $deleted;
    }
}

class InsertCommand implements Command {
    public function __construct(
        private TextEditor $editor,
        private int $position,
        private string $text,
    ) {}
    
    public function execute(): void {
        $this->editor->insertAt($this->position, $this->text);
    }
    
    public function undo(): void {
        $this->editor->deleteAt($this->position, mb_strlen($this->text));
    }
}

class DeleteCommand implements Command {
    private string $deletedText = '';
    
    public function __construct(
        private TextEditor $editor,
        private int $position,
        private int $length,
    ) {}
    
    public function execute(): void {
        $this->deletedText = $this->editor->deleteAt($this->position, $this->length);
    }
    
    public function undo(): void {
        $this->editor->insertAt($this->position, $this->deletedText);
    }
}

// Usage
$editor = new TextEditor();
$editor->setContent('Hello World');

$editor->executeCommand(new InsertCommand($editor, 5, ', PHP'));
echo $editor->getContent(); // Hello, PHP World

$editor->executeCommand(new DeleteCommand($editor, 0, 6));
echo $editor->getContent(); // PHP World

$editor->undo();
echo $editor->getContent(); // Hello, PHP World

$editor->undo();
echo $editor->getContent(); // Hello World
```

---

## 🎯 สรุป Part 12

| Pattern | การใช้งาน |
|---------|-----------|
| Singleton | Database connection, Logger |
| Factory | Object creation based on type |
| Abstract Factory | UI component families |
| Builder | Complex object construction (QueryBuilder) |
| Decorator | Add features without modifying class |
| Repository | Data access abstraction |
| Observer | Event system |
| Strategy | Interchangeable algorithms |
| Command | Undo/Redo, queueable actions |

**ถัดไป → Part 14: PHP Security**
