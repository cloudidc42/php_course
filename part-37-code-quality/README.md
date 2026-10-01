# Part 37: PHP Static Analysis & Code Quality
## ขั้นตอนที่ 971-1000: มาตรฐานโค้ดระดับโลก

---

## ขั้นตอนที่ 971: PHPStan (Static Analysis)

```bash
# ติดตั้ง
composer require --dev phpstan/phpstan
composer require --dev phpstan/extension-installer
composer require --dev larastan/larastan  # สำหรับ Laravel
```

```neon
# phpstan.neon

includes:
    - vendor/larastan/larastan/extension.neon

parameters:
    level: 9  # 0-9, ยิ่งสูงยิ่ง strict
    paths:
        - app
        - tests
    excludePaths:
        - app/Http/Controllers/Auth  # ถ้า generated code
    
    # Type checking for array shapes
    checkGenericClassInNonGenericObjectType: false
    
    # Check missing return types
    checkMissingIterableValueType: false
    
    # Laravel-specific
    checkModelProperties: true
    checkPhpDocMissingReturn: true
    
    # Ignore certain errors
    ignoreErrors:
        - '#Call to an undefined method Illuminate\\Database\\Eloquent\\Builder#'
    
    # Universal object crates (bypass type checking)
    universalObjectCratesClasses:
        - stdClass
```

```php
<?php
declare(strict_types=1);

// Example: PHPStan Level 9 compliant code

// ❌ PHPStan Level 5+ error: missing return type
function getUser($id) {
    return User::find($id);
}

// ✅ PHPStan level 9: explicit types
function getUser(int $id): ?User {
    return User::find($id);
}

// ❌ PHPStan error: array access without type
function processItems(array $items): float {
    $total = 0;
    foreach ($items as $item) {
        $total += $item['price']; // No guarantee 'price' exists
    }
    return $total;
}

// ✅ PHPStan: typed array with PHPDoc
/**
 * @param array<int, array{price: float, name: string}> $items
 */
function processItems(array $items): float {
    $total = 0.0;
    foreach ($items as $item) {
        $total += $item['price'];
    }
    return $total;
}

// ✅ Even better: use DTOs
class OrderItem {
    public function __construct(
        public readonly string $name,
        public readonly float $price,
        public readonly int $quantity = 1,
    ) {}
    
    public function total(): float {
        return $this->price * $this->quantity;
    }
}

function processItems2(array $items): float {
    return array_sum(array_map(fn(OrderItem $item) => $item->total(), $items));
}

// Generics with PHPDoc
/**
 * @template T of object
 * @param class-string<T> $class
 * @param int $id
 * @return T|null
 */
function findEntity(string $class, int $id): ?object {
    return $class::find($id);
}

$user = findEntity(User::class, 1); // PHPStan knows: User|null

// Run PHPStan
// vendor/bin/phpstan analyse
// vendor/bin/phpstan analyse --level=max
// vendor/bin/phpstan analyse app --generate-baseline  # Create baseline for existing code
```

---

## ขั้นตอนที่ 972: PHP CS Fixer (Code Style)

```php
<?php
// .php-cs-fixer.dist.php

$finder = PhpCsFixer\Finder::create()
    ->in(__DIR__ . '/app')
    ->in(__DIR__ . '/tests')
    ->exclude(['bootstrap', 'storage', 'public'])
    ->name('*.php')
    ->notName('*.blade.php');

return (new PhpCsFixer\Config())
    ->setRules([
        '@PSR12'                                    => true,
        '@PHP82Migration'                           => true,
        
        // Arrays
        'array_syntax'                              => ['syntax' => 'short'],
        'array_indentation'                         => true,
        'trailing_comma_in_multiline'               => ['elements' => ['arrays', 'parameters', 'arguments']],
        
        // Imports
        'ordered_imports'                           => ['sort_algorithm' => 'alpha'],
        'no_unused_imports'                         => true,
        'global_namespace_import'                   => ['import_classes' => true],
        
        // Strings
        'single_quote'                              => true,
        'explicit_string_variable'                  => true,
        
        // Statements
        'declare_strict_types'                      => true,
        'strict_comparison'                         => true,
        'strict_param'                              => true,
        
        // Classes
        'class_attributes_separation'               => ['elements' => ['method' => 'one', 'property' => 'one']],
        'ordered_class_elements'                    => ['order' => ['use_trait', 'constant', 'property', 'construct', 'destruct', 'magic', 'phpunit', 'method_public', 'method_protected', 'method_private']],
        'final_class'                               => false,
        'self_static_accessor'                      => true,
        
        // Functions
        'return_type_declaration'                   => ['space_before' => 'none'],
        'nullable_type_declaration_for_default_null_value' => true,
        'phpdoc_to_param_type'                     => true,
        'phpdoc_to_return_type'                    => true,
        
        // PHPDoc
        'phpdoc_align'                              => ['align' => 'left'],
        'phpdoc_order'                              => true,
        'no_empty_phpdoc'                           => true,
        'phpdoc_no_useless_inheritdoc'              => true,
        
        // Modern PHP
        'modernize_types_casting'                   => true,
        'get_class_to_class_keyword'                => true,
        'use_arrow_functions'                       => true,
    ])
    ->setFinder($finder)
    ->setCacheFile('/tmp/.php-cs-fixer.cache');
```

---

## ขั้นตอนที่ 973: Psalm (Another Static Analyzer)

```xml
<!-- psalm.xml -->
<?xml version="1.0"?>
<psalm
  errorLevel="1"
  resolveFromConfigFile="true"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="https://getpsalm.org/schema/config"
  xsi:schemaLocation="https://getpsalm.org/schema/config vendor/vimeo/psalm/config.xsd"
>
    <projectFiles>
        <directory name="app" />
        <ignoreFiles>
            <directory name="vendor" />
        </ignoreFiles>
    </projectFiles>
    
    <issueHandlers>
        <PropertyNotSetInConstructor errorLevel="suppress" />
    </issueHandlers>
    
    <plugins>
        <pluginClass class="Psalm\LaravelPlugin\Plugin"/>
    </plugins>
</psalm>
```

```php
<?php
declare(strict_types=1);

// Psalm-specific annotations

/**
 * @psalm-immutable
 */
final class Money {
    public function __construct(
        /** @psalm-readonly */
        public readonly int $amount,    // amount in satang (สตางค์)
        /** @psalm-readonly */
        public readonly string $currency,
    ) {}
    
    /** @psalm-pure */
    public function add(Money $other): self {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException('Currency mismatch');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }
    
    /** @psalm-pure */
    public function toFloat(): float {
        return $this->amount / 100;
    }
    
    public function __toString(): string {
        return number_format($this->toFloat(), 2) . ' ' . $this->currency;
    }
}

// Template types
/**
 * @template T
 * @psalm-immutable
 */
class Optional {
    private bool $hasValue;
    
    /** @psalm-var T|null */
    private mixed $value;
    
    private function __construct(mixed $value, bool $hasValue) {
        $this->value = $value;
        $this->hasValue = $hasValue;
    }
    
    /** @psalm-return static<null> */
    public static function empty(): static {
        return new static(null, false);
    }
    
    /**
     * @template U
     * @psalm-param U $value
     * @psalm-return static<U>
     */
    public static function of(mixed $value): static {
        return new static($value, true);
    }
    
    public function isPresent(): bool {
        return $this->hasValue;
    }
    
    /** @psalm-return T */
    public function get(): mixed {
        if (!$this->hasValue) {
            throw new \RuntimeException('No value present');
        }
        return $this->value;
    }
    
    /**
     * @template U
     * @psalm-param callable(T): U $mapper
     * @psalm-return static<U>
     */
    public function map(callable $mapper): static {
        if (!$this->hasValue) return static::empty();
        return static::of($mapper($this->value));
    }
}
```

---

## ขั้นตอนที่ 974: Rector (Automated Refactoring)

```bash
# ติดตั้ง
composer require rector/rector --dev

# rector.php
```

```php
<?php
// rector.php

use Rector\Config\RectorConfig;
use Rector\Set\ValueObject\LevelSetList;
use Rector\Set\ValueObject\SetList;
use Rector\TypeDeclaration\Rector\Property\TypedPropertyFromStrictConstructorRector;
use Rector\PHP80\Rector\Switch_\ChangeSwitchToMatchRector;
use Rector\PHP81\Rector\Array_\FirstClassCallableRector;
use Rector\PHP82\Rector\Class_\ReadOnlyClassRector;

return RectorConfig::configure()
    ->withPaths([__DIR__ . '/app', __DIR__ . '/tests'])
    ->withSkip([__DIR__ . '/app/Http/Controllers/Auth'])
    ->withSets([
        LevelSetList::UP_TO_PHP_83,
        SetList::CODE_QUALITY,
        SetList::DEAD_CODE,
        SetList::TYPE_DECLARATION,
    ])
    ->withRules([
        TypedPropertyFromStrictConstructorRector::class,
        ChangeSwitchToMatchRector::class,
        FirstClassCallableRector::class,
        ReadOnlyClassRector::class,
    ]);

// Run Rector
// vendor/bin/rector process --dry-run  # Preview changes
// vendor/bin/rector process            # Apply changes
```

```php
<?php
// Before Rector (PHP 7.x style)
class OldCode {
    private $name;
    private $email;
    
    public function __construct($name, $email) {
        $this->name = $name;
        $this->email = $email;
    }
    
    public function getName() {
        return $this->name;
    }
    
    public function process($value) {
        switch ($value) {
            case 'a': return 1;
            case 'b': return 2;
            default: return 0;
        }
    }
}

// After Rector (PHP 8.3 style)
final readonly class NewCode {
    public function __construct(
        private string $name,
        private string $email,
    ) {}
    
    public function getName(): string {
        return $this->name;
    }
    
    public function process(string $value): int {
        return match($value) {
            'a' => 1,
            'b' => 2,
            default => 0,
        };
    }
}
```

---

## ขั้นตอนที่ 975: Makefile สำหรับ Development

```makefile
# Makefile

.PHONY: help install test lint fix analyse build deploy

help: ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

install: ## Install dependencies
	composer install
	npm install

test: ## Run tests
	php artisan test --parallel

test-coverage: ## Run tests with coverage
	php artisan test --coverage --min=80

lint: ## Check code style
	vendor/bin/php-cs-fixer fix --dry-run --diff

fix: ## Fix code style
	vendor/bin/php-cs-fixer fix
	vendor/bin/rector process

analyse: ## Run static analysis
	vendor/bin/phpstan analyse --no-progress
	vendor/bin/psalm --no-progress

qa: lint analyse test ## Run all quality checks

build: ## Build assets
	npm run build

migrate-fresh: ## Fresh migration with seeds
	php artisan migrate:fresh --seed

ci: ## CI pipeline locally
	@make lint
	@make analyse
	@make test

docker-up: ## Start Docker environment
	docker-compose up -d

docker-down: ## Stop Docker environment
	docker-compose down

docker-logs: ## Show logs
	docker-compose logs -f app
```

---

## ขั้นตอนที่ 976: Pre-commit Hooks

```bash
# .git/hooks/pre-commit

#!/bin/bash
set -e

echo "Running pre-commit checks..."

# PHP syntax check
FILES=$(git diff --cached --name-only --diff-filter=ACM | grep '\.php$')
if [ -n "$FILES" ]; then
    for FILE in $FILES; do
        php -l "$FILE"
    done
fi

# Code style
if ! ./vendor/bin/php-cs-fixer fix --dry-run --diff --quiet; then
    echo "❌ Code style issues found. Run 'make fix' to fix."
    exit 1
fi

# Static analysis
if ! ./vendor/bin/phpstan analyse --no-progress --quiet; then
    echo "❌ PHPStan errors found."
    exit 1
fi

# Tests (fast unit tests only)
if ! php artisan test --testsuite=Unit --quiet; then
    echo "❌ Unit tests failed."
    exit 1
fi

echo "✅ Pre-commit checks passed!"
```

```bash
# Install using Husky (for npm projects) or directly
chmod +x .git/hooks/pre-commit

# Or use CaptainHook (PHP)
# composer require captainhook/captainhook --dev
# vendor/bin/captainhook install
```

---

## 🎯 สรุป Part 37

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| PHPStan | Level 0-9, type annotations, baseline |
| PHP CS Fixer | PSR-12, PHP 8.x rules, auto-fix |
| Psalm | Immutable, pure functions, generics |
| Rector | Auto-upgrade PHP 7 → 8.3 code |
| Makefile | Development workflow automation |
| Pre-commit Hooks | Quality gates before commit |

**ถัดไป → Part 38: Kubernetes & Cloud-Native PHP**
