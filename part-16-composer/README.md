# Part 16: Composer & Package Management
## ขั้นตอนที่ 401-420: จัดการ Dependencies อย่างมืออาชีพ

---

## ขั้นตอนที่ 401: Composer คืออะไร

```bash
# ติดตั้ง Composer
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer --version

# Composer Commands ที่ใช้บ่อย
composer init                    # เริ่มต้น project ใหม่
composer require vendor/package  # ติดตั้ง package
composer require vendor/package:^2.0  # กำหนด version
composer require --dev phpunit/phpunit  # dev dependency
composer remove vendor/package   # ลบ package
composer install                 # ติดตั้งจาก composer.lock
composer update                  # อัปเดต packages
composer update vendor/package   # อัปเดต package เดียว
composer dump-autoload           # regenerate autoload
composer dump-autoload -o        # optimized (production)
composer show                    # แสดง installed packages
composer show -t                 # tree view
composer outdated                # packages ที่มี update
composer validate                # ตรวจสอบ composer.json
composer diagnose                # ตรวจสอบปัญหา
composer self-update             # อัปเดต composer เอง
composer search keyword          # ค้นหา package
composer create-project          # สร้าง project จาก template
```

---

## ขั้นตอนที่ 402: composer.json

```json
{
    "name": "mycompany/myapp",
    "description": "My awesome PHP application",
    "type": "project",
    "license": "MIT",
    "minimum-stability": "stable",
    "prefer-stable": true,
    "require": {
        "php": "^8.2",
        "ext-pdo": "*",
        "ext-json": "*",
        "ext-mbstring": "*",
        "vlucas/phpdotenv": "^5.6",
        "ramsey/uuid": "^4.7",
        "nesbot/carbon": "^3.0",
        "league/flysystem": "^3.0",
        "monolog/monolog": "^3.0",
        "guzzlehttp/guzzle": "^7.8",
        "symfony/cache": "^7.0",
        "firebase/php-jwt": "^6.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "mockery/mockery": "^1.6",
        "fakerphp/faker": "^1.23",
        "phpstan/phpstan": "^1.10",
        "squizlabs/php_codesniffer": "^3.8",
        "friendsofphp/php-cs-fixer": "^3.0"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/",
            "App\\Database\\": "database/"
        },
        "files": [
            "src/helpers.php"
        ]
    },
    "autoload-dev": {
        "psr-4": {
            "Tests\\": "tests/"
        }
    },
    "scripts": {
        "test": "phpunit",
        "test:coverage": "phpunit --coverage-html coverage",
        "cs": "phpcs --standard=PSR12 src/",
        "cs:fix": "php-cs-fixer fix src/",
        "stan": "phpstan analyse src/ --level=8",
        "check": [
            "@cs",
            "@stan",
            "@test"
        ],
        "post-install-cmd": [
            "@php -r \"if (file_exists('.env.example') && !file_exists('.env')) { copy('.env.example', '.env'); echo '.env created from .env.example'; }\""
        ]
    },
    "config": {
        "optimize-autoloader": true,
        "sort-packages": true,
        "allow-plugins": {
            "php-http/discovery": true
        }
    },
    "extra": {
        "branch-alias": {
            "dev-main": "1.0-dev"
        }
    }
}
```

---

## ขั้นตอนที่ 403: PSR-4 Autoloading

```
Directory Structure:
myapp/
├── src/
│   ├── Controllers/
│   │   └── UserController.php
│   ├── Models/
│   │   └── User.php
│   ├── Services/
│   │   └── UserService.php
│   ├── Repositories/
│   │   └── UserRepository.php
│   └── helpers.php
├── tests/
│   └── UserServiceTest.php
├── composer.json
└── index.php
```

```php
<?php
// src/Models/User.php
namespace App\Models;

class User {
    public function __construct(
        public readonly int $id,
        public readonly string $name,
        public readonly string $email,
    ) {}
}

// src/Controllers/UserController.php
namespace App\Controllers;

use App\Models\User;
use App\Services\UserService;

class UserController {
    public function __construct(
        private UserService $userService
    ) {}
    
    public function index(): array {
        return $this->userService->getAllUsers();
    }
}

// src/Services/UserService.php
namespace App\Services;

use App\Models\User;
use App\Repositories\UserRepository;

class UserService {
    public function __construct(
        private UserRepository $users
    ) {}
    
    public function getAllUsers(): array {
        return $this->users->findAll();
    }
    
    public function createUser(string $name, string $email): User {
        // Business logic here
        $user = new User(0, $name, $email);
        return $this->users->save($user);
    }
}

// index.php
require 'vendor/autoload.php';

use App\Controllers\UserController;
use App\Services\UserService;
use App\Repositories\PdoUserRepository;

$pdo = new PDO('...');
$controller = new UserController(
    new UserService(
        new PdoUserRepository($pdo)
    )
);

$users = $controller->index();
```

---

## ขั้นตอนที่ 404: Popular PHP Packages

```php
<?php
declare(strict_types=1);
require 'vendor/autoload.php';

// ========================
// Carbon - Date/Time
// ========================
use Carbon\Carbon;

$now = Carbon::now('Asia/Bangkok');
echo $now->format('Y-m-d H:i:s');  // 2024-01-15 14:30:00
echo $now->toIso8601String();       // 2024-01-15T14:30:00+07:00

$birthday = Carbon::parse('1990-05-20');
echo $birthday->age;                // ตามอายุจริง
echo $birthday->diffForHumans();    // 33 years ago

$future = Carbon::now()->addDays(7)->addHours(3);
echo $future->format('d M Y');

// Thai locale
Carbon::setLocale('th');
echo Carbon::now()->diffForHumans(); // "ขณะนี้"

// ========================
// Guzzle - HTTP Client
// ========================
use GuzzleHttp\Client;
use GuzzleHttp\Exception\RequestException;

$client = new Client([
    'base_uri' => 'https://api.example.com/v1/',
    'timeout' => 10.0,
    'headers' => [
        'Accept' => 'application/json',
        'Authorization' => 'Bearer ' . $token,
    ],
]);

// GET
$response = $client->get('users', [
    'query' => ['page' => 1, 'per_page' => 20],
]);
$users = json_decode($response->getBody(), true);

// POST
$response = $client->post('users', [
    'json' => ['name' => 'สมชาย', 'email' => 'somchai@example.com'],
]);
$created = json_decode($response->getBody(), true);

// With error handling
try {
    $response = $client->get('users/999');
    $user = json_decode($response->getBody(), true);
} catch (RequestException $e) {
    if ($e->hasResponse()) {
        $status = $e->getResponse()->getStatusCode();
        echo "Error {$status}";
    }
}

// ========================
// Monolog - Logging
// ========================
use Monolog\Logger;
use Monolog\Handler\StreamHandler;
use Monolog\Handler\RotatingFileHandler;
use Monolog\Formatter\JsonFormatter;

$log = new Logger('app');

// File handler with rotation
$handler = new RotatingFileHandler(
    '/var/log/app/app.log',
    30, // keep 30 days
    Logger::INFO
);
$handler->setFormatter(new JsonFormatter());
$log->pushHandler($handler);

// Console handler
$log->pushHandler(new StreamHandler('php://stderr', Logger::DEBUG));

$log->info('User logged in', ['user_id' => 42, 'ip' => '127.0.0.1']);
$log->warning('High memory usage', ['memory' => memory_get_usage(true)]);
$log->error('Database connection failed', ['exception' => $e]);

// ========================
// Ramsey UUID
// ========================
use Ramsey\Uuid\Uuid;

$uuid = Uuid::uuid4()->toString();     // 550e8400-e29b-41d4-a716-446655440000
$uuid7 = Uuid::uuid7()->toString();    // Time-sorted UUID (v7)
$isValid = Uuid::isValid($uuid);

// ========================
// vlucas/phpdotenv - Environment
// ========================
use Dotenv\Dotenv;

$dotenv = Dotenv::createImmutable(__DIR__);
$dotenv->load();
$dotenv->required(['DB_HOST', 'DB_NAME', 'DB_USER', 'DB_PASS']);
$dotenv->required('APP_ENV')->allowedValues(['local', 'staging', 'production']);

$dbHost = $_ENV['DB_HOST'];

// ========================
// league/flysystem - File Storage
// ========================
use League\Flysystem\Filesystem;
use League\Flysystem\Local\LocalFilesystemAdapter;
use League\Flysystem\AwsS3V3\AwsS3V3Adapter; // needs league/flysystem-aws-s3-v3

// Local
$adapter = new LocalFilesystemAdapter('/var/www/storage');
$filesystem = new Filesystem($adapter);

$filesystem->write('uploads/photo.jpg', $fileContents);
$filesystem->copy('uploads/photo.jpg', 'backups/photo.jpg');
$filesystem->delete('uploads/photo.jpg');
$contents = $filesystem->read('uploads/photo.jpg');
$listing = $filesystem->listContents('uploads');

// Switch to S3 without changing business logic
// $adapter = new AwsS3V3Adapter($s3Client, 'my-bucket');
// $filesystem = new Filesystem($adapter);
// Same methods work!

// ========================
// Faker - Test Data
// ========================
use Faker\Factory;

$faker = Factory::create('th_TH');

echo $faker->name();          // ชื่อไทย
echo $faker->email();         // test123@example.com
echo $faker->phoneNumber();   // 081-234-5678
echo $faker->address();       // ที่อยู่ไทย
echo $faker->paragraph();     // Lorem ipsum

// Seeder
function seedUsers(\PDO $pdo, int $count = 100): void {
    $faker = Factory::create('th_TH');
    $pm = new PasswordManager();
    
    $stmt = $pdo->prepare("INSERT INTO users (name, email, password) VALUES (?, ?, ?)");
    
    for ($i = 0; $i < $count; $i++) {
        $stmt->execute([
            $faker->name(),
            $faker->unique()->safeEmail(),
            $pm->hash('password123'),
        ]);
    }
    
    echo "Seeded {$count} users\n";
}
```

---

## ขั้นตอนที่ 405: สร้าง Package ของตัวเอง

```php
<?php
// composer.json ของ package
{
    "name": "myname/thai-helpers",
    "description": "Thai language utilities for PHP",
    "type": "library",
    "license": "MIT",
    "keywords": ["thai", "utilities", "helpers"],
    "authors": [{"name": "Your Name", "email": "you@example.com"}],
    "require": {"php": "^8.1"},
    "autoload": {
        "psr-4": {"ThaiHelpers\\": "src/"},
        "files": ["src/helpers.php"]
    },
    "autoload-dev": {
        "psr-4": {"ThaiHelpers\\Tests\\": "tests/"}
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0"
    }
}
```

```php
<?php
declare(strict_types=1);
// src/ThaiNumber.php
namespace ThaiHelpers;

class ThaiNumber {
    private const DIGITS = ['ศูนย์', 'หนึ่ง', 'สอง', 'สาม', 'สี่', 'ห้า', 'หก', 'เจ็ด', 'แปด', 'เก้า'];
    private const PLACES = ['', 'สิบ', 'ร้อย', 'พัน', 'หมื่น', 'แสน', 'ล้าน'];
    
    public static function toText(float $amount): string {
        if ($amount < 0) return 'ลบ' . static::toText(abs($amount));
        if ($amount === 0.0) return 'ศูนย์บาทถ้วน';
        
        $baht = (int) floor($amount);
        $satang = round(($amount - $baht) * 100);
        
        $result = static::convertWholeNumber($baht) . 'บาท';
        
        if ($satang > 0) {
            $result .= static::convertWholeNumber((int) $satang) . 'สตางค์';
        } else {
            $result .= 'ถ้วน';
        }
        
        return $result;
    }
    
    private static function convertWholeNumber(int $n): string {
        if ($n === 0) return '';
        if ($n < 10) return self::DIGITS[$n];
        if ($n < 100) {
            $result = '';
            $tens = (int) floor($n / 10);
            $ones = $n % 10;
            
            if ($tens > 1) $result .= self::DIGITS[$tens];
            $result .= 'สิบ';
            if ($ones === 1) $result .= 'เอ็ด';
            elseif ($ones > 0) $result .= self::DIGITS[$ones];
            
            return $result;
        }
        
        if ($n >= 1000000) {
            return static::convertWholeNumber((int) floor($n / 1000000)) . 'ล้าน'
                . static::convertWholeNumber($n % 1000000);
        }
        
        // Other places
        $places = [100000 => 'แสน', 10000 => 'หมื่น', 1000 => 'พัน', 100 => 'ร้อย'];
        $result = '';
        
        foreach ($places as $place => $name) {
            if ($n >= $place) {
                $result .= self::DIGITS[(int)floor($n / $place)] . $name;
                $n %= $place;
            }
        }
        
        if ($n > 0) $result .= static::convertWholeNumber($n);
        return $result;
    }
}

// Test
echo ThaiNumber::toText(1234.50); // หนึ่งพันสองร้อยสามสิบสี่บาทห้าสิบสตางค์
echo ThaiNumber::toText(1000000); // หนึ่งล้านบาทถ้วน
```

---

## 🎯 สรุป Part 16

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Composer | init, require, install, update, scripts |
| composer.json | dependencies, autoload, scripts |
| PSR-4 | Namespace mapping, directory structure |
| Popular Packages | Carbon, Guzzle, Monolog, Faker, Dotenv |
| Create Package | composer.json, namespace, publish to Packagist |

**ถัดไป → Part 18: Laravel Routing & Controllers**
