# Part 72: Laravel Dusk

## ขั้นตอนที่ 2021-2050: End-to-End Testing ด้วย Laravel Dusk

Laravel Dusk ให้ browser automation สำหรับ E2E testing ที่ test JavaScript interactions ได้จริง

---

## ขั้นตอนที่ 2021: ติดตั้ง Laravel Dusk

```bash
# ติดตั้ง Dusk
composer require --dev laravel/dusk

# ติดตั้ง Chrome driver
php artisan dusk:install

# ดาวน์โหลด ChromeDriver
php artisan dusk:chrome-driver

# รัน tests
php artisan dusk

# รัน specific test
php artisan dusk --filter=LoginTest
```

```php
<?php

declare(strict_types=1);

// app/Providers/DuskServiceProvider.php
namespace App\Providers;

use Laravel\Dusk\Browser;
use Laravel\Dusk\DuskServiceProvider as ServiceProvider;

class DuskServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        parent::boot();

        // Custom Dusk macros
        Browser::macro('assertToastMessage', function (string $message) {
            /** @var Browser $this */
            return $this->waitForText($message, 5)
                        ->assertSee($message);
        });

        Browser::macro('waitForSpinnerToDisappear', function (int $timeout = 10) {
            /** @var Browser $this */
            return $this->waitUntilMissing('.loading-spinner', $timeout);
        });

        Browser::macro('fillVueInput', function (string $selector, string $value) {
            /** @var Browser $this */
            return $this->script("
                var element = document.querySelector('{$selector}');
                var nativeInputValueSetter = Object.getOwnPropertyDescriptor(window.HTMLInputElement.prototype, 'value').set;
                nativeInputValueSetter.call(element, '{$value}');
                element.dispatchEvent(new Event('input', { bubbles: true }));
            ");
        });
    }
}
```

---

## ขั้นตอนที่ 2022: Page Objects Pattern

```php
<?php

declare(strict_types=1);

// tests/Browser/Pages/LoginPage.php
namespace Tests\Browser\Pages;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Page;

class LoginPage extends Page
{
    public function url(): string
    {
        return '/login';
    }

    public function assert(Browser $browser): void
    {
        $browser->assertPathIs($this->url())
                ->assertSee('Login')
                ->assertPresent('@email-field')
                ->assertPresent('@password-field');
    }

    public function elements(): array
    {
        return [
            '@email-field'    => 'input[name="email"]',
            '@password-field' => 'input[name="password"]',
            '@login-button'   => 'button[type="submit"]',
            '@remember-me'    => 'input[name="remember"]',
            '@forgot-link'    => 'a.forgot-password',
            '@error-message'  => '.alert-danger',
        ];
    }

    // Custom page methods
    public function loginAs(Browser $browser, string $email, string $password): void
    {
        $browser->type('@email-field', $email)
                ->type('@password-field', $password)
                ->press('@login-button');
    }

    public function loginWithRemember(Browser $browser, string $email, string $password): void
    {
        $browser->type('@email-field', $email)
                ->type('@password-field', $password)
                ->check('@remember-me')
                ->press('@login-button');
    }
}
```

```php
<?php

declare(strict_types=1);

// tests/Browser/Pages/DashboardPage.php
namespace Tests\Browser\Pages;

use Laravel\Dusk\Browser;
use Laravel\Dusk\Page;

class DashboardPage extends Page
{
    public function url(): string
    {
        return '/dashboard';
    }

    public function assert(Browser $browser): void
    {
        $browser->assertPathIs($this->url())
                ->assertSee('Dashboard');
    }

    public function elements(): array
    {
        return [
            '@stats-cards'    => '.stats-overview',
            '@revenue-chart'  => '#revenue-chart',
            '@recent-orders'  => '.recent-orders-table',
            '@quick-actions'  => '.quick-actions',
            '@notification'   => '.notification-bell',
            '@user-menu'      => '.user-menu-dropdown',
        ];
    }

    public function openNotifications(Browser $browser): void
    {
        $browser->click('@notification')
                ->waitFor('.notifications-panel');
    }
}
```

---

## ขั้นตอนที่ 2023: Authentication Tests

```php
<?php

declare(strict_types=1);

// tests/Browser/Auth/LoginTest.php
namespace Tests\Browser\Auth;

use App\Models\User;
use Illuminate\Foundation\Testing\DatabaseTruncation;
use Laravel\Dusk\Browser;
use Tests\Browser\Pages\DashboardPage;
use Tests\Browser\Pages\LoginPage;
use Tests\DuskTestCase;

class LoginTest extends DuskTestCase
{
    use DatabaseTruncation;

    public function test_user_can_login(): void
    {
        $user = User::factory()->create([
            'email'    => 'test@example.com',
            'password' => bcrypt('password123'),
        ]);

        $this->browse(function (Browser $browser) {
            $browser->visit(new LoginPage)
                    ->loginAs('test@example.com', 'password123')
                    ->on(new DashboardPage)
                    ->assertSee('Welcome back')
                    ->assertAuthenticated();
        });
    }

    public function test_login_fails_with_invalid_credentials(): void
    {
        $this->browse(function (Browser $browser) {
            $browser->visit(new LoginPage)
                    ->loginAs('wrong@email.com', 'wrongpassword')
                    ->assertSee('credentials')  // Error message
                    ->assertPresent('@error-message')
                    ->assertPathIs('/login');    // Still on login page
        });
    }

    public function test_login_rate_limiting(): void
    {
        $this->browse(function (Browser $browser) {
            $loginPage = new LoginPage;
            $browser->visit($loginPage);

            // Attempt 5 failed logins
            for ($i = 0; $i < 5; $i++) {
                $browser->loginAs('bad@email.com', 'wrongpassword')
                        ->waitForText('credentials');
            }

            // 6th attempt should show rate limit message
            $browser->loginAs('bad@email.com', 'wrongpassword')
                    ->waitForText('Too many login attempts');
        });
    }

    public function test_remember_me_functionality(): void
    {
        $user = User::factory()->create();

        $this->browse(function (Browser $browser) use ($user) {
            $browser->visit(new LoginPage)
                    ->loginWithRemember($user->email, 'password')
                    ->on(new DashboardPage);

            // Check remember me cookie exists
            $cookies = $browser->driver->manage()->getCookies();
            $hasCookie = collect($cookies)->contains(fn ($c) => str_contains($c['name'], 'remember'));

            $this->assertTrue($hasCookie, 'Remember me cookie should exist');
        });
    }
}
```

---

## ขั้นตอนที่ 2024: Complex Form Testing

```php
<?php

declare(strict_types=1);

// tests/Browser/Products/CreateProductTest.php
namespace Tests\Browser\Products;

use App\Models\Category;
use App\Models\User;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class CreateProductTest extends DuskTestCase
{
    public function test_admin_can_create_product(): void
    {
        $admin    = User::factory()->admin()->create();
        $category = Category::factory()->create(['name' => 'Electronics']);

        $this->browse(function (Browser $browser) use ($admin, $category) {
            $browser->loginAs($admin)
                    ->visit('/admin/products/create')
                    ->assertSee('Create Product')

                    // กรอก form
                    ->type('name', 'Test Product 2024')
                    ->type('price', '999.99')
                    ->type('description', 'A great test product')

                    // เลือก category ด้วย Select2
                    ->click('.select2-selection')
                    ->waitFor('.select2-dropdown')
                    ->type('.select2-search__field', 'Elec')
                    ->waitForText('Electronics')
                    ->click('.select2-results__option:contains("Electronics")')

                    // Upload image
                    ->attach('featured_image', __DIR__ . '/fixtures/product.jpg')
                    ->waitFor('.image-preview')

                    // Tags input
                    ->type('#tags-input', 'new')
                    ->keys('#tags-input', '{enter}')
                    ->type('#tags-input', 'popular')
                    ->keys('#tags-input', '{enter}')

                    // Submit
                    ->press('Save Product')
                    ->waitForText('Product created successfully')

                    // ตรวจสอบ redirect
                    ->assertPathBeginsWith('/admin/products/')
                    ->assertSee('Test Product 2024');
        });
    }

    public function test_validation_errors_appear(): void
    {
        $admin = User::factory()->admin()->create();

        $this->browse(function (Browser $browser) use ($admin) {
            $browser->loginAs($admin)
                    ->visit('/admin/products/create')
                    ->press('Save Product')  // Submit empty form

                    // ตรวจสอบ validation errors
                    ->assertSee('The name field is required')
                    ->assertSee('The price field is required')
                    ->assertPresent('.invalid-feedback')

                    // Type invalid price
                    ->type('price', 'not-a-number')
                    ->press('Save Product')
                    ->assertSee('The price must be a number');
        });
    }
}
```

---

## ขั้นตอนที่ 2025: JavaScript Interaction Tests

```php
<?php

declare(strict_types=1);

// tests/Browser/Checkout/CheckoutTest.php
namespace Tests\Browser\Checkout;

use App\Models\Product;
use App\Models\User;
use Laravel\Dusk\Browser;
use Tests\DuskTestCase;

class CheckoutTest extends DuskTestCase
{
    public function test_complete_checkout_flow(): void
    {
        $user    = User::factory()->create();
        $product = Product::factory()->create([
            'price'          => 500,
            'stock_quantity' => 10,
        ]);

        $this->browse(function (Browser $browser) use ($user, $product) {
            $browser->loginAs($user)

                    // เพิ่ม product ไปยัง cart
                    ->visit("/products/{$product->slug}")
                    ->press('Add to Cart')
                    ->waitForText('Added to cart')

                    // ไปที่ cart
                    ->click('.cart-icon')
                    ->waitForLocation('/cart')
                    ->assertSee($product->name)
                    ->assertSee('฿500.00')

                    // ไป checkout
                    ->press('Checkout')
                    ->waitForLocation('/checkout')

                    // กรอก shipping info
                    ->type('[name="shipping_name"]', $user->name)
                    ->type('[name="shipping_address"]', '123 Sukhumvit Rd')
                    ->type('[name="shipping_city"]', 'Bangkok')
                    ->type('[name="shipping_postal_code"]', '10110')

                    // กรอก credit card ใน Stripe iframe
                    ->withinFrame('#stripe-card-element iframe', function (Browser $frame) {
                        $frame->type('[name="cardnumber"]', '4242424242424242')
                              ->type('[name="exp-date"]', '12/26')
                              ->type('[name="cvc"]', '123');
                    })

                    // Place order
                    ->press('Place Order')
                    ->waitForLocation('/orders/confirmation*', 30)
                    ->assertSee('Order Confirmed!')
                    ->assertSee($product->name);
        });
    }

    public function test_cart_quantity_update(): void
    {
        $user    = User::factory()->create();
        $product = Product::factory()->create(['price' => 100]);

        $this->browse(function (Browser $browser) use ($user, $product) {
            $browser->loginAs($user)
                    ->visit('/cart')

                    // เพิ่ม product ก่อน
                    ->script("
                        fetch('/api/cart', {
                            method: 'POST',
                            headers: {'Content-Type': 'application/json', 'X-CSRF-TOKEN': document.querySelector('meta[name=csrf-token]').content},
                            body: JSON.stringify({product_id: {$product->id}, quantity: 1})
                        }).then(() => location.reload());
                    ")
                    ->waitFor('.cart-item')

                    // เพิ่ม quantity
                    ->click('.quantity-increase')
                    ->waitForText('฿200.00')  // 2x price
                    ->assertInputValue('.quantity-input', '2')

                    // ลด quantity กลับ
                    ->click('.quantity-decrease')
                    ->waitForText('฿100.00')
                    ->assertInputValue('.quantity-input', '1');
        });
    }
}
```

---

## ขั้นตอนที่ 2026: CI Integration สำหรับ Browser Tests

```yaml
# .github/workflows/dusk.yml
name: Browser Tests

on:
  push:
    branches: [main]
  pull_request:

jobs:
  dusk:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'

      - name: Install PHP dependencies
        run: composer install --no-interaction

      - name: Install Node dependencies
        run: npm ci && npm run build

      - name: Setup environment
        run: |
          cp .env.dusk.ci .env
          php artisan key:generate
          php artisan migrate --seed

      - name: Start server
        run: php artisan serve &

      - name: Setup Chrome
        uses: browser-actions/setup-chrome@latest

      - name: Run Dusk tests
        run: php artisan dusk --env=ci
        env:
          APP_URL: http://localhost:8000

      - name: Upload screenshots
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: dusk-screenshots
          path: tests/Browser/screenshots

      - name: Upload console logs
        uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: dusk-console-logs
          path: tests/Browser/console
```

---

## สรุป Part 72

| Feature | รายละเอียด |
|---------|-----------|
| Page Objects | Reusable page interactions |
| Browser Macros | Custom browser methods |
| Form Testing | Complex form interactions |
| JavaScript Testing | SPA interactions |
| File Upload | Attach files in tests |
| Iframe Testing | Stripe, embedded content |
| CI Integration | Run in GitHub Actions |
| Screenshots | Capture on failure |

ถัดไป → Part 73: Composer Package Development
