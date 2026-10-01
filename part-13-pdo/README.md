# Part 13: PDO & Prepared Statements
## ขั้นตอนที่ 301-330

---

## ขั้นตอนที่ 301: PDO พื้นฐาน

```php
<?php
declare(strict_types=1);

// PDO = PHP Data Objects - Database abstraction layer
// รองรับ: MySQL, PostgreSQL, SQLite, MSSQL, Oracle

// การเชื่อมต่อ MySQL
try {
    $dsn = "mysql:host=localhost;dbname=myapp;charset=utf8mb4";
    $username = "root";
    $password = "password";
    
    $options = [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,  // Throw exceptions
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,        // Fetch as array
        PDO::ATTR_EMULATE_PREPARES   => false,                   // Native prepares
        PDO::MYSQL_ATTR_INIT_COMMAND => "SET NAMES utf8mb4 COLLATE utf8mb4_unicode_ci",
        PDO::ATTR_TIMEOUT            => 5,
    ];
    
    $pdo = new PDO($dsn, $username, $password, $options);
    echo "Connected successfully!\n";
    
} catch (PDOException $e) {
    echo "Connection failed: " . $e->getMessage() . "\n";
    exit(1);
}

// SQLite (ไม่ต้องการ Server)
$sqlite = new PDO('sqlite:/tmp/test.db');
$sqlite->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

// PostgreSQL
// $pdo = new PDO("pgsql:host=localhost;dbname=myapp", "user", "pass");

// MSSQL
// $pdo = new PDO("sqlsrv:server=localhost;database=myapp", "user", "pass");
```

---

## ขั้นตอนที่ 302: DDL - Creating Tables

```php
<?php
// สร้างตาราง
function createTables(PDO $pdo): void {
    $sql = "
    CREATE TABLE IF NOT EXISTS users (
        id          INT AUTO_INCREMENT PRIMARY KEY,
        name        VARCHAR(100) NOT NULL,
        email       VARCHAR(150) UNIQUE NOT NULL,
        password    VARCHAR(255) NOT NULL,
        role        ENUM('admin', 'editor', 'user') DEFAULT 'user',
        active      TINYINT(1) DEFAULT 1,
        created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        INDEX idx_email (email),
        INDEX idx_role_active (role, active)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
    
    CREATE TABLE IF NOT EXISTS posts (
        id          INT AUTO_INCREMENT PRIMARY KEY,
        user_id     INT NOT NULL,
        title       VARCHAR(255) NOT NULL,
        slug        VARCHAR(255) UNIQUE NOT NULL,
        content     LONGTEXT,
        excerpt     TEXT,
        status      ENUM('draft', 'published', 'archived') DEFAULT 'draft',
        views       INT DEFAULT 0,
        published_at DATETIME NULL,
        created_at  DATETIME DEFAULT CURRENT_TIMESTAMP,
        updated_at  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
        INDEX idx_slug (slug),
        INDEX idx_status (status),
        FULLTEXT INDEX ft_content (title, content)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
    
    CREATE TABLE IF NOT EXISTS categories (
        id          INT AUTO_INCREMENT PRIMARY KEY,
        name        VARCHAR(100) NOT NULL,
        slug        VARCHAR(100) UNIQUE NOT NULL,
        parent_id   INT NULL,
        description TEXT,
        FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
    
    CREATE TABLE IF NOT EXISTS post_categories (
        post_id     INT NOT NULL,
        category_id INT NOT NULL,
        PRIMARY KEY (post_id, category_id),
        FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
        FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
    ";
    
    // Execute multiple statements
    $pdo->exec($sql);
    echo "Tables created successfully\n";
}
```

---

## ขั้นตอนที่ 303: Prepared Statements - INSERT

```php
<?php
// Prepared Statements ป้องกัน SQL Injection
class UserRepository {
    public function __construct(private PDO $pdo) {}
    
    // Create
    public function create(array $data): int {
        $stmt = $this->pdo->prepare("
            INSERT INTO users (name, email, password, role)
            VALUES (:name, :email, :password, :role)
        ");
        
        $stmt->execute([
            ':name' => $data['name'],
            ':email' => $data['email'],
            ':password' => password_hash($data['password'], PASSWORD_ARGON2ID),
            ':role' => $data['role'] ?? 'user'
        ]);
        
        return (int)$this->pdo->lastInsertId();
    }
    
    // Batch Insert
    public function createMany(array $users): int {
        $this->pdo->beginTransaction();
        
        try {
            $stmt = $this->pdo->prepare("
                INSERT INTO users (name, email, password) VALUES (?, ?, ?)
            ");
            
            $count = 0;
            foreach ($users as $user) {
                $stmt->execute([
                    $user['name'],
                    $user['email'],
                    password_hash($user['password'], PASSWORD_ARGON2ID)
                ]);
                $count++;
            }
            
            $this->pdo->commit();
            return $count;
            
        } catch (PDOException $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
    
    // Read - Single
    public function findById(int $id): ?array {
        $stmt = $this->pdo->prepare("
            SELECT id, name, email, role, active, created_at
            FROM users 
            WHERE id = ? AND active = 1
        ");
        $stmt->execute([$id]);
        $result = $stmt->fetch();
        return $result ?: null;
    }
    
    // Read - By Email
    public function findByEmail(string $email): ?array {
        $stmt = $this->pdo->prepare("
            SELECT * FROM users WHERE email = :email LIMIT 1
        ");
        $stmt->execute([':email' => $email]);
        return $stmt->fetch() ?: null;
    }
    
    // Read - All with Pagination
    public function findAll(
        int $page = 1, 
        int $perPage = 15,
        string $sortBy = 'id',
        string $sortDir = 'ASC'
    ): array {
        $offset = ($page - 1) * $perPage;
        $allowedSort = ['id', 'name', 'email', 'created_at'];
        $sortBy = in_array($sortBy, $allowedSort) ? $sortBy : 'id';
        $sortDir = strtoupper($sortDir) === 'DESC' ? 'DESC' : 'ASC';
        
        $stmt = $this->pdo->prepare("
            SELECT id, name, email, role, active, created_at
            FROM users
            ORDER BY $sortBy $sortDir
            LIMIT :limit OFFSET :offset
        ");
        
        $stmt->bindValue(':limit', $perPage, PDO::PARAM_INT);
        $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll();
    }
    
    // Count
    public function count(): int {
        return (int)$this->pdo->query("SELECT COUNT(*) FROM users")->fetchColumn();
    }
    
    // Update
    public function update(int $id, array $data): bool {
        $allowed = ['name', 'email', 'role', 'active'];
        $fields = array_intersect_key($data, array_flip($allowed));
        
        if (empty($fields)) return false;
        
        $setClauses = array_map(fn($k) => "$k = :$k", array_keys($fields));
        $setStr = implode(', ', $setClauses);
        
        $stmt = $this->pdo->prepare("
            UPDATE users SET $setStr WHERE id = :id
        ");
        
        $fields['id'] = $id;
        return $stmt->execute($fields) && $stmt->rowCount() > 0;
    }
    
    // Delete (Soft Delete)
    public function delete(int $id): bool {
        $stmt = $this->pdo->prepare("
            UPDATE users SET active = 0 WHERE id = :id
        ");
        return $stmt->execute([':id' => $id]) && $stmt->rowCount() > 0;
    }
    
    // Hard Delete
    public function forceDelete(int $id): bool {
        $stmt = $this->pdo->prepare("DELETE FROM users WHERE id = ?");
        return $stmt->execute([$id]) && $stmt->rowCount() > 0;
    }
    
    // Search
    public function search(string $query, int $limit = 20): array {
        $stmt = $this->pdo->prepare("
            SELECT id, name, email, role
            FROM users
            WHERE (name LIKE :query OR email LIKE :query)
            AND active = 1
            LIMIT :limit
        ");
        
        $stmt->bindValue(':query', "%$query%");
        $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
        $stmt->execute();
        
        return $stmt->fetchAll();
    }
}
```

---

## ขั้นตอนที่ 304: Advanced Queries

```php
<?php
class PostRepository {
    public function __construct(private PDO $pdo) {}
    
    // JOIN Query
    public function findWithAuthor(int $id): ?array {
        $stmt = $this->pdo->prepare("
            SELECT 
                p.id, p.title, p.content, p.status, p.views, p.published_at,
                u.name AS author_name, u.email AS author_email
            FROM posts p
            INNER JOIN users u ON p.user_id = u.id
            WHERE p.id = :id
        ");
        $stmt->execute([':id' => $id]);
        return $stmt->fetch() ?: null;
    }
    
    // Multiple JOINs
    public function findWithDetails(int $id): ?array {
        $stmt = $this->pdo->prepare("
            SELECT 
                p.*,
                u.name AS author_name,
                GROUP_CONCAT(c.name SEPARATOR ', ') AS categories,
                COUNT(DISTINCT com.id) AS comment_count
            FROM posts p
            INNER JOIN users u ON p.user_id = u.id
            LEFT JOIN post_categories pc ON p.id = pc.post_id
            LEFT JOIN categories c ON pc.category_id = c.id
            LEFT JOIN comments com ON p.id = com.post_id
            WHERE p.id = :id
            GROUP BY p.id
        ");
        $stmt->execute([':id' => $id]);
        return $stmt->fetch() ?: null;
    }
    
    // Dynamic WHERE Clauses
    public function findByFilters(array $filters): array {
        $conditions = ['1 = 1'];
        $params = [];
        
        if (!empty($filters['status'])) {
            $conditions[] = 'p.status = :status';
            $params[':status'] = $filters['status'];
        }
        
        if (!empty($filters['user_id'])) {
            $conditions[] = 'p.user_id = :user_id';
            $params[':user_id'] = $filters['user_id'];
        }
        
        if (!empty($filters['search'])) {
            $conditions[] = '(p.title LIKE :search OR p.content LIKE :search)';
            $params[':search'] = '%' . $filters['search'] . '%';
        }
        
        if (!empty($filters['date_from'])) {
            $conditions[] = 'p.published_at >= :date_from';
            $params[':date_from'] = $filters['date_from'];
        }
        
        if (!empty($filters['date_to'])) {
            $conditions[] = 'p.published_at <= :date_to';
            $params[':date_to'] = $filters['date_to'];
        }
        
        $where = implode(' AND ', $conditions);
        $page = $filters['page'] ?? 1;
        $perPage = $filters['per_page'] ?? 15;
        $offset = ($page - 1) * $perPage;
        
        $sql = "
            SELECT p.id, p.title, p.slug, p.status, p.views, p.published_at,
                   u.name AS author_name
            FROM posts p
            INNER JOIN users u ON p.user_id = u.id
            WHERE $where
            ORDER BY p.published_at DESC
            LIMIT :limit OFFSET :offset
        ";
        
        $stmt = $this->pdo->prepare($sql);
        
        foreach ($params as $key => $value) {
            $stmt->bindValue($key, $value);
        }
        $stmt->bindValue(':limit', $perPage, PDO::PARAM_INT);
        $stmt->bindValue(':offset', $offset, PDO::PARAM_INT);
        
        $stmt->execute();
        return $stmt->fetchAll();
    }
    
    // Aggregate Queries
    public function getStats(): array {
        $stmt = $this->pdo->query("
            SELECT 
                COUNT(*) AS total,
                SUM(CASE WHEN status = 'published' THEN 1 ELSE 0 END) AS published,
                SUM(CASE WHEN status = 'draft' THEN 1 ELSE 0 END) AS drafts,
                SUM(views) AS total_views,
                AVG(views) AS avg_views,
                MAX(views) AS max_views,
                MIN(published_at) AS first_post,
                MAX(published_at) AS last_post
            FROM posts
        ");
        return $stmt->fetch();
    }
    
    // Subquery
    public function getMostPopular(int $limit = 5): array {
        $stmt = $this->pdo->prepare("
            SELECT p.*, u.name AS author_name
            FROM posts p
            INNER JOIN users u ON p.user_id = u.id
            WHERE p.views = (
                SELECT MAX(views) FROM posts p2 WHERE p2.user_id = p.user_id
            )
            ORDER BY p.views DESC
            LIMIT :limit
        ");
        $stmt->bindValue(':limit', $limit, PDO::PARAM_INT);
        $stmt->execute();
        return $stmt->fetchAll();
    }
}
```

---

## ขั้นตอนที่ 305: Transactions

```php
<?php
class TransactionManager {
    public function __construct(private PDO $pdo) {}
    
    public function transfer(int $fromId, int $toId, float $amount): void {
        $this->pdo->beginTransaction();
        
        try {
            // ตรวจสอบยอดคงเหลือ (Pessimistic Lock)
            $stmt = $this->pdo->prepare("
                SELECT balance FROM accounts WHERE id = ? FOR UPDATE
            ");
            $stmt->execute([$fromId]);
            $fromAccount = $stmt->fetch();
            
            if (!$fromAccount || $fromAccount['balance'] < $amount) {
                throw new RuntimeException("Insufficient funds");
            }
            
            // หักเงิน
            $stmt = $this->pdo->prepare("
                UPDATE accounts SET balance = balance - ? WHERE id = ?
            ");
            $stmt->execute([$amount, $fromId]);
            
            // เพิ่มเงิน
            $stmt = $this->pdo->prepare("
                UPDATE accounts SET balance = balance + ? WHERE id = ?
            ");
            $stmt->execute([$amount, $toId]);
            
            // บันทึก Transaction
            $stmt = $this->pdo->prepare("
                INSERT INTO transactions (from_id, to_id, amount, type, status)
                VALUES (?, ?, ?, 'transfer', 'completed')
            ");
            $stmt->execute([$fromId, $toId, $amount]);
            
            $this->pdo->commit();
            
        } catch (\Exception $e) {
            $this->pdo->rollBack();
            throw $e;
        }
    }
    
    // Savepoints
    public function complexOperation(PDO $pdo): void {
        $pdo->beginTransaction();
        
        try {
            // Operation 1
            $pdo->exec("INSERT INTO table1 ...");
            
            // Savepoint
            $pdo->exec("SAVEPOINT sp1");
            
            try {
                // Risky Operation
                $pdo->exec("INSERT INTO table2 ...");
            } catch (\Exception $e) {
                // Rollback to savepoint only
                $pdo->exec("ROLLBACK TO SAVEPOINT sp1");
                // Continue with rest of transaction
            }
            
            // Operation 3
            $pdo->exec("INSERT INTO table3 ...");
            
            $pdo->commit();
            
        } catch (\Exception $e) {
            $pdo->rollBack();
            throw $e;
        }
    }
}
```

---

## ขั้นตอนที่ 306: Query Builder

```php
<?php
declare(strict_types=1);

class QueryBuilder {
    private string $table = '';
    private array $conditions = [];
    private array $params = [];
    private array $columns = ['*'];
    private array $joins = [];
    private array $orderBy = [];
    private array $groupBy = [];
    private array $having = [];
    private ?int $limit = null;
    private ?int $offset = null;
    private int $paramCounter = 0;
    
    public function __construct(private PDO $pdo) {}
    
    public function table(string $table): static {
        $this->table = $table;
        return $this;
    }
    
    public function select(string ...$columns): static {
        $this->columns = $columns;
        return $this;
    }
    
    public function where(string $column, string $operator, mixed $value): static {
        $param = ":p" . (++$this->paramCounter);
        $this->conditions[] = "$column $operator $param";
        $this->params[$param] = $value;
        return $this;
    }
    
    public function whereIn(string $column, array $values): static {
        $placeholders = [];
        foreach ($values as $value) {
            $param = ":p" . (++$this->paramCounter);
            $placeholders[] = $param;
            $this->params[$param] = $value;
        }
        $this->conditions[] = "$column IN (" . implode(', ', $placeholders) . ")";
        return $this;
    }
    
    public function whereLike(string $column, string $value): static {
        return $this->where($column, 'LIKE', "%$value%");
    }
    
    public function join(string $table, string $condition, string $type = 'INNER'): static {
        $this->joins[] = "$type JOIN $table ON $condition";
        return $this;
    }
    
    public function leftJoin(string $table, string $condition): static {
        return $this->join($table, $condition, 'LEFT');
    }
    
    public function orderBy(string $column, string $direction = 'ASC'): static {
        $this->orderBy[] = "$column $direction";
        return $this;
    }
    
    public function groupBy(string ...$columns): static {
        $this->groupBy = array_merge($this->groupBy, $columns);
        return $this;
    }
    
    public function having(string $condition): static {
        $this->having[] = $condition;
        return $this;
    }
    
    public function limit(int $limit): static {
        $this->limit = $limit;
        return $this;
    }
    
    public function offset(int $offset): static {
        $this->offset = $offset;
        return $this;
    }
    
    public function page(int $page, int $perPage = 15): static {
        $this->limit = $perPage;
        $this->offset = ($page - 1) * $perPage;
        return $this;
    }
    
    private function buildSQL(): string {
        $columns = implode(', ', $this->columns);
        $sql = "SELECT $columns FROM {$this->table}";
        
        foreach ($this->joins as $join) {
            $sql .= " $join";
        }
        
        if (!empty($this->conditions)) {
            $sql .= " WHERE " . implode(' AND ', $this->conditions);
        }
        
        if (!empty($this->groupBy)) {
            $sql .= " GROUP BY " . implode(', ', $this->groupBy);
        }
        
        if (!empty($this->having)) {
            $sql .= " HAVING " . implode(' AND ', $this->having);
        }
        
        if (!empty($this->orderBy)) {
            $sql .= " ORDER BY " . implode(', ', $this->orderBy);
        }
        
        if ($this->limit !== null) {
            $sql .= " LIMIT {$this->limit}";
        }
        
        if ($this->offset !== null) {
            $sql .= " OFFSET {$this->offset}";
        }
        
        return $sql;
    }
    
    public function get(): array {
        $sql = $this->buildSQL();
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($this->params);
        return $stmt->fetchAll();
    }
    
    public function first(): ?array {
        $this->limit(1);
        $results = $this->get();
        return $results[0] ?? null;
    }
    
    public function count(): int {
        $this->columns = ['COUNT(*) AS count'];
        $this->limit = null;
        $this->offset = null;
        $sql = $this->buildSQL();
        $stmt = $this->pdo->prepare($sql);
        $stmt->execute($this->params);
        return (int)$stmt->fetchColumn();
    }
    
    public function insert(array $data): int {
        $columns = implode(', ', array_keys($data));
        $placeholders = ':' . implode(', :', array_keys($data));
        
        $stmt = $this->pdo->prepare("
            INSERT INTO {$this->table} ($columns) VALUES ($placeholders)
        ");
        $stmt->execute($data);
        return (int)$this->pdo->lastInsertId();
    }
    
    public function update(array $data): int {
        $setClauses = array_map(fn($k) => "$k = :set_$k", array_keys($data));
        $setStr = implode(', ', $setClauses);
        
        $whereStr = empty($this->conditions) ? '' : 'WHERE ' . implode(' AND ', $this->conditions);
        
        $stmt = $this->pdo->prepare("UPDATE {$this->table} SET $setStr $whereStr");
        
        $params = [];
        foreach ($data as $key => $value) {
            $params["set_$key"] = $value;
        }
        foreach ($this->params as $key => $value) {
            $params[ltrim($key, ':')] = $value;
        }
        
        $stmt->execute($params);
        return $stmt->rowCount();
    }
    
    public function delete(): int {
        $whereStr = empty($this->conditions) ? '' : 'WHERE ' . implode(' AND ', $this->conditions);
        $stmt = $this->pdo->prepare("DELETE FROM {$this->table} $whereStr");
        $stmt->execute($this->params);
        return $stmt->rowCount();
    }
    
    public function toSQL(): string {
        return $this->buildSQL();
    }
}

// Usage
// $qb = new QueryBuilder($pdo);
// 
// $users = $qb->table('users')
//     ->select('id', 'name', 'email')
//     ->where('active', '=', 1)
//     ->where('role', '=', 'admin')
//     ->orderBy('name')
//     ->page(1, 10)
//     ->get();
//
// $qb->table('posts')
//     ->leftJoin('users', 'users.id = posts.user_id')
//     ->select('posts.*', 'users.name AS author')
//     ->where('posts.status', '=', 'published')
//     ->orderBy('posts.published_at', 'DESC')
//     ->limit(5)
//     ->get();
```

---

## ขั้นตอนที่ 307: Connection Pool และ Database Manager

```php
<?php
declare(strict_types=1);

class DatabaseManager {
    private static array $connections = [];
    private static array $configs = [];
    private static string $default = 'mysql';
    
    public static function addConnection(string $name, array $config): void {
        self::$configs[$name] = $config;
    }
    
    public static function connection(string $name = null): PDO {
        $name = $name ?? self::$default;
        
        if (!isset(self::$connections[$name])) {
            self::$connections[$name] = self::createConnection($name);
        }
        
        return self::$connections[$name];
    }
    
    private static function createConnection(string $name): PDO {
        $config = self::$configs[$name] ?? throw new RuntimeException("Connection '$name' not configured");
        
        $dsn = match($config['driver']) {
            'mysql' => "mysql:host={$config['host']};port={$config['port']};dbname={$config['database']};charset=utf8mb4",
            'pgsql' => "pgsql:host={$config['host']};port={$config['port']};dbname={$config['database']}",
            'sqlite' => "sqlite:{$config['database']}",
            default => throw new RuntimeException("Unsupported driver: {$config['driver']}")
        };
        
        $options = [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES => false,
        ];
        
        $pdo = new PDO($dsn, $config['username'] ?? null, $config['password'] ?? null, $options);
        
        // MySQL specific
        if ($config['driver'] === 'mysql') {
            $pdo->exec("SET time_zone = '+07:00'");
        }
        
        return $pdo;
    }
    
    public static function disconnect(string $name = null): void {
        if ($name) {
            unset(self::$connections[$name]);
        } else {
            self::$connections = [];
        }
    }
    
    public static function table(string $table, string $connection = null): QueryBuilder {
        return (new QueryBuilder(self::connection($connection)))->table($table);
    }
}

// Configuration
DatabaseManager::addConnection('mysql', [
    'driver' => 'mysql',
    'host' => 'localhost',
    'port' => 3306,
    'database' => 'myapp',
    'username' => 'root',
    'password' => 'password',
]);

DatabaseManager::addConnection('sqlite', [
    'driver' => 'sqlite',
    'database' => '/tmp/test.db',
]);

// Usage
// $db = DatabaseManager::connection();
// $users = DatabaseManager::table('users')->where('active', '=', 1)->get();
```

---

## 📝 แบบฝึกหัด Part 13

### แบบฝึกหัดที่ 1: User Authentication System
สร้าง Authentication ด้วย PDO:
- Register (hash password ด้วย Argon2ID)
- Login (verify password)
- Remember Me Token
- Password Reset

### แบบฝึกหัดที่ 2: Blog CRUD
สร้าง Blog System ด้วย PDO:
- Create/Read/Update/Delete Posts
- Categories
- Tags (Many-to-Many)
- Search with Full-text

### แบบฝึกหัดที่ 3: Report Generator
สร้าง Report ด้วย Complex Queries:
- Monthly Stats
- User Activity
- Top Content

---

## 🎯 สรุป Part 13

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| PDO Connection | DSN, Options, Exception Mode |
| Prepared Statements | Positional (?), Named (:param) |
| CRUD Operations | INSERT, SELECT, UPDATE, DELETE |
| Transactions | beginTransaction, commit, rollBack |
| Advanced Queries | JOIN, Subquery, Aggregate |
| Query Builder | Dynamic query building |
| Database Manager | Connection pooling |

**ถัดไป → Part 14: PHP Security**
