# Part 73: Composer Package Development

## ขั้นตอนที่ 2051-2080: สร้าง Composer Packages

การสร้าง packages ที่ดีช่วยให้แชร์ code ระหว่างโปรเจค และ contribute to open source ได้

---

## ขั้นตอนที่ 2051: โครงสร้าง Composer Package

```bash
# สร้าง package ใหม่
mkdir my-company/awesome-package
cd my-company/awesome-package

# Initialize composer.json
composer init

# โครงสร้างมาตรฐาน
my-company/awesome-package/
├── src/
│   ├── AwesomePackage.php
│   ├── Facades/
│   │   └── Awesome.php
│   └── Providers/
│       └── AwesomeServiceProvider.php
├── config/
│   └── awesome.php
├── database/
│   └── migrations/
│       └── 2024_01_01_create_awesome_table.php
├── resources/
│   └── views/
│       └── awesome.blade.php
├── routes/
│   └── awesome.php
├── tests/
│   ├── Unit/
│   └── Feature/
├── .github/
│   └── workflows/
│       └── test.yml
├── composer.json
├── README.md
└── CHANGELOG.md
```

```json
{
    "name": "my-company/awesome-package",
    "description": "An awesome Laravel package",
    "type": "library",
    "license": "MIT",
    "authors": [
        {
            "name": "Your Name",
            "email": "you@example.com"
        }
    ],
    "require": {
        "php": "^8.2",
        "illuminate/support": "^10.0|^11.0"
    },
    "require-dev": {
        "orchestra/testbench": "^8.0|^9.0",
        "phpunit/phpunit": "^10.0|^11.0",
        "pestphp/pest": "^2.0"
    },
    "autoload": {
        "psr-4": {
            "MyCompany\\Awesome\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "MyCompany\\Awesome\\Tests\\": "tests/"
        }
    },
    "extra": {
        "laravel": {
            "providers": [
                "MyCompany\\Awesome\\Providers\\AwesomeServiceProvider"
            ],
            "aliases": {
                "Awesome": "MyCompany\\Awesome\\Facades\\Awesome"
            }
        }
    },
    "minimum-stability": "stable",
    "prefer-stable": true
}
```

---

## ขั้นตอนที่ 2052: Service Provider

```php
<?php

declare(strict_types=1);

// src/Providers/AwesomeServiceProvider.php
namespace MyCompany\Awesome\Providers;

use Illuminate\Support\ServiceProvider;
use MyCompany\Awesome\AwesomeManager;

class AwesomeServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // Merge config
        $this->mergeConfigFrom(
            __DIR__ . '/../../config/awesome.php',
            'awesome'
        );

        // Bind to container
        $this->app->singleton('awesome', function ($app) {
            return new AwesomeManager($app['config']['awesome']);
        });

        // Bind interface
        $this->app->bind(
            \MyCompany\Awesome\Contracts\AwesomeInterface::class,
            \MyCompany\Awesome\AwesomeManager::class
        );
    }

    public function boot(): void
    {
        // Publish config
        if ($this->app->runningInConsole()) {
            $this->publishes([
                __DIR__ . '/../../config/awesome.php' => config_path('awesome.php'),
            ], 'awesome-config');

            // Publish migrations
            $this->publishes([
                __DIR__ . '/../../database/migrations/' => database_path('migrations'),
            ], 'awesome-migrations');

            // Publish views
            $this->publishes([
                __DIR__ . '/../../resources/views' => resource_path('views/vendor/awesome'),
            ], 'awesome-views');

            // Register commands
            $this->commands([
                \MyCompany\Awesome\Commands\AwesomeCommand::class,
                \MyCompany\Awesome\Commands\InstallCommand::class,
            ]);
        }

        // Load views
        $this->loadViewsFrom(__DIR__ . '/../../resources/views', 'awesome');

        // Load translations
        $this->loadTranslationsFrom(__DIR__ . '/../../resources/lang', 'awesome');

        // Load routes
        $this->loadRoutesFrom(__DIR__ . '/../../routes/awesome.php');

        // Load migrations (optional - use publishable for user control)
        if (config('awesome.load_migrations', true)) {
            $this->loadMigrationsFrom(__DIR__ . '/../../database/migrations');
        }
    }
}
```

---

## ขั้นตอนที่ 2053: Facade

```php
<?php

declare(strict_types=1);

// src/Facades/Awesome.php
namespace MyCompany\Awesome\Facades;

use Illuminate\Support\Facades\Facade;

/**
 * @method static mixed process(array $data)
 * @method static bool validate(string $input)
 * @method static array getResults(string $query)
 *
 * @see \MyCompany\Awesome\AwesomeManager
 */
class Awesome extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'awesome';
    }
}
```

```php
<?php

declare(strict_types=1);

// src/AwesomeManager.php
namespace MyCompany\Awesome;

use MyCompany\Awesome\Contracts\AwesomeInterface;
use MyCompany\Awesome\Drivers\DriverInterface;

class AwesomeManager implements AwesomeInterface
{
    private array $drivers = [];

    public function __construct(private array $config) {}

    public function driver(string $name = null): DriverInterface
    {
        $name = $name ?? $this->config['default'];

        if (!isset($this->drivers[$name])) {
            $this->drivers[$name] = $this->createDriver($name);
        }

        return $this->drivers[$name];
    }

    private function createDriver(string $name): DriverInterface
    {
        $driverConfig = $this->config['drivers'][$name] ?? [];

        return match ($name) {
            'redis'    => new \MyCompany\Awesome\Drivers\RedisDriver($driverConfig),
            'database' => new \MyCompany\Awesome\Drivers\DatabaseDriver($driverConfig),
            'file'     => new \MyCompany\Awesome\Drivers\FileDriver($driverConfig),
            default    => throw new \InvalidArgumentException("Driver [{$name}] not supported."),
        };
    }

    public function process(array $data): mixed
    {
        return $this->driver()->process($data);
    }

    public function validate(string $input): bool
    {
        return $this->driver()->validate($input);
    }

    public function getResults(string $query): array
    {
        return $this->driver()->query($query);
    }
}
```

---

## ขั้นตอนที่ 2054: Testing with Orchestra Testbench

```php
<?php

declare(strict_types=1);

// tests/TestCase.php
namespace MyCompany\Awesome\Tests;

use MyCompany\Awesome\Providers\AwesomeServiceProvider;
use Orchestra\Testbench\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function getPackageProviders($app): array
    {
        return [
            AwesomeServiceProvider::class,
        ];
    }

    protected function getPackageAliases($app): array
    {
        return [
            'Awesome' => \MyCompany\Awesome\Facades\Awesome::class,
        ];
    }

    protected function getEnvironmentSetUp($app): void
    {
        // Setup database
        $app['config']->set('database.default', 'sqlite');
        $app['config']->set('database.connections.sqlite', [
            'driver'   => 'sqlite',
            'database' => ':memory:',
            'prefix'   => '',
        ]);

        // Package config
        $app['config']->set('awesome.default', 'file');
        $app['config']->set('awesome.drivers.file.path', sys_get_temp_dir());
    }

    protected function setUp(): void
    {
        parent::setUp();
        $this->loadMigrationsFrom(__DIR__ . '/../database/migrations');
    }
}
```

```php
<?php

declare(strict_types=1);

// tests/Unit/AwesomeManagerTest.php
namespace MyCompany\Awesome\Tests\Unit;

use MyCompany\Awesome\AwesomeManager;
use MyCompany\Awesome\Facades\Awesome;
use MyCompany\Awesome\Tests\TestCase;

class AwesomeManagerTest extends TestCase
{
    public function test_facade_is_accessible(): void
    {
        $this->assertInstanceOf(AwesomeManager::class, app('awesome'));
    }

    public function test_can_process_data(): void
    {
        $result = Awesome::process(['key' => 'value']);
        $this->assertIsArray($result);
    }

    public function test_validates_input(): void
    {
        $this->assertTrue(Awesome::validate('valid-input'));
        $this->assertFalse(Awesome::validate(''));
    }

    public function test_uses_correct_driver(): void
    {
        config(['awesome.default' => 'file']);

        $manager = app('awesome');
        $driver  = $manager->driver();

        $this->assertInstanceOf(\MyCompany\Awesome\Drivers\FileDriver::class, $driver);
    }
}
```

---

## ขั้นตอนที่ 2055: Semantic Versioning

```bash
# Semantic Versioning: MAJOR.MINOR.PATCH

# PATCH (1.0.0 -> 1.0.1): Bug fix, backward compatible
git tag v1.0.1

# MINOR (1.0.0 -> 1.1.0): New feature, backward compatible
git tag v1.1.0

# MAJOR (1.0.0 -> 2.0.0): Breaking changes
git tag v2.0.0

# Pre-release
git tag v2.0.0-beta.1
git tag v2.0.0-rc.1

# Push tags
git push origin --tags
```

```json
// composer.json version constraints
{
    "require": {
        "my-company/awesome": "^1.0",     // >= 1.0.0 < 2.0.0
        "my-company/awesome": "~1.2",     // >= 1.2.0 < 2.0.0
        "my-company/awesome": "1.2.*",    // >= 1.2.0 < 1.3.0
        "my-company/awesome": ">=1.0 <2", // Range
        "my-company/awesome": "1.2.3"     // Exact version
    }
}
```

---

## ขั้นตอนที่ 2056: Publishing to Packagist

```bash
# 1. สร้าง account บน packagist.org
# 2. Connect GitHub repo
# 3. Auto-update webhook

# GitHub Webhook สำหรับ auto-update
# Packagist URL: https://packagist.org/api/update-package
# Content-Type: application/json
# Secret: your-packagist-api-token

# package.json สำหรับ version management
{
    "scripts": {
        "release:patch": "npm version patch && git push --tags",
        "release:minor": "npm version minor && git push --tags",
        "release:major": "npm version major && git push --tags"
    }
}
```

---

## ขั้นตอนที่ 2057: Laravel Package Stubs

```php
<?php

declare(strict_types=1);

// src/Commands/InstallCommand.php
namespace MyCompany\Awesome\Commands;

use Illuminate\Console\Command;

class InstallCommand extends Command
{
    protected $signature   = 'awesome:install';
    protected $description = 'Install the Awesome package';

    public function handle(): int
    {
        $this->info('Installing Awesome package...');

        // Publish config
        $this->callSilent('vendor:publish', [
            '--provider' => "MyCompany\\Awesome\\Providers\\AwesomeServiceProvider",
            '--tag'      => 'awesome-config',
        ]);

        // Publish migrations
        $this->callSilent('vendor:publish', [
            '--provider' => "MyCompany\\Awesome\\Providers\\AwesomeServiceProvider",
            '--tag'      => 'awesome-migrations',
        ]);

        // Run migrations
        if ($this->confirm('Run migrations?', true)) {
            $this->call('migrate');
        }

        $this->info('Awesome package installed successfully!');
        $this->comment('Edit config/awesome.php to configure the package.');

        return self::SUCCESS;
    }
}
```

---

## สรุป Part 73

| ขั้นตอน | รายละเอียด |
|---------|-----------|
| composer.json | ระบุ dependencies, autoloading |
| Service Provider | Register + Boot package |
| Facade | Static API สำหรับ package |
| Testbench | Test package เหมือน Laravel app |
| Semantic Versioning | MAJOR.MINOR.PATCH |
| Packagist | Publish สำหรับ community |
| Extra.laravel | Auto-discover providers |

ถัดไป → Part 74: PHP Extensions และ Libraries
