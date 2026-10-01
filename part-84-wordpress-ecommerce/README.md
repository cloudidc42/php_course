# Part 84: WordPress E-commerce
## ขั้นตอนที่ 2381-2410: WooCommerce Advanced Customization

การปรับแต่ง WooCommerce ขั้นสูง ครอบคลุม Custom Product Types,
Digital Downloads, Subscriptions, Booking และ Inventory Management

---

## ขั้นตอนที่ 2381: Custom Product Type

```php
<?php
declare(strict_types=1);

// plugins/my-custom-products/my-custom-products.php

/**
 * Plugin Name: My Custom Products
 * Description: Custom WooCommerce product types
 * Version: 1.0.0
 */

if (!defined('ABSPATH')) exit;

// Register custom product type
add_filter('product_type_selector', function(array $types): array {
    $types['bundle'] = __('Bundle Product', 'my-custom-products');
    $types['subscription'] = __('Subscription Product', 'my-custom-products');
    return $types;
});

// Load custom product class
add_filter('woocommerce_product_class', function(string $classname, string $product_type): string {
    return match ($product_type) {
        'bundle' => 'WC_Product_Bundle',
        'subscription' => 'WC_Product_Subscription_Custom',
        default => $classname,
    };
}, 10, 2);

class WC_Product_Bundle extends \WC_Product
{
    public string $product_type = 'bundle';

    public function __construct(int|\WC_Product $product = 0)
    {
        $this->supports[] = 'ajax_add_to_cart';
        parent::__construct($product);
    }

    public function get_bundled_items(): array
    {
        $items = $this->get_meta('_bundled_items', true);
        return is_array($items) ? $items : [];
    }

    public function set_bundled_items(array $items): void
    {
        $this->update_meta_data('_bundled_items', $items);
    }

    public function calculate_bundle_price(): float
    {
        $total = 0.0;
        foreach ($this->get_bundled_items() as $item) {
            $product = wc_get_product($item['product_id']);
            if ($product) {
                $discount = (float)($item['discount'] ?? 0);
                $price = (float)$product->get_price();
                $total += $price * (1 - $discount / 100) * (int)$item['quantity'];
            }
        }
        return $total;
    }
}
```

---

## ขั้นตอนที่ 2382: Digital Downloads

```php
<?php
declare(strict_types=1);

// เพิ่มฟีเจอร์ secure download
class WC_Secure_Download_Manager
{
    public function __construct()
    {
        add_filter('woocommerce_download_file_redirect', [$this, 'secureRedirect'], 10, 2);
        add_action('woocommerce_download_product', [$this, 'logDownload'], 10, 5);
        add_filter('woocommerce_customer_available_downloads', [$this, 'addExpiryInfo'], 10, 2);
    }

    public function secureRedirect(bool $redirect, string $file_path): bool
    {
        // ใช้ signed URL แทน direct redirect
        return false;
    }

    public function generateSignedUrl(int $product_id, int $user_id, string $file_hash): string
    {
        $expiry = time() + (24 * 3600); // 24 hours
        $token = hash_hmac('sha256', "{$product_id}:{$user_id}:{$file_hash}:{$expiry}", AUTH_KEY);

        return add_query_arg([
            'download' => 1,
            'product' => $product_id,
            'uid' => $user_id,
            'hash' => $file_hash,
            'exp' => $expiry,
            'token' => $token,
        ], home_url('/'));
    }

    public function logDownload(int $product_id, int $user_id, string $email, string $order_key, string $download_id): void
    {
        global $wpdb;
        $wpdb->insert(
            $wpdb->prefix . 'download_logs',
            [
                'product_id' => $product_id,
                'user_id' => $user_id,
                'email' => $email,
                'order_key' => $order_key,
                'download_id' => $download_id,
                'ip_address' => $_SERVER['REMOTE_ADDR'],
                'user_agent' => $_SERVER['HTTP_USER_AGENT'],
                'downloaded_at' => current_time('mysql'),
            ],
            ['%d', '%d', '%s', '%s', '%s', '%s', '%s', '%s']
        );
    }

    public function addExpiryInfo(array $downloads, int $user_id): array
    {
        foreach ($downloads as &$download) {
            $order = wc_get_order($download['order_id']);
            if ($order) {
                $order_date = $order->get_date_created();
                $expiry_days = (int)get_option('woocommerce_downloads_expiry_days', 365);
                $expiry = $order_date->modify("+{$expiry_days} days");
                $download['expiry_date'] = $expiry->format('Y-m-d');
                $download['days_remaining'] = max(0, (int)$expiry->diff(new \DateTime())->days);
            }
        }
        return $downloads;
    }
}

new WC_Secure_Download_Manager();
```

---

## ขั้นตอนที่ 2383: Custom Checkout Fields

```php
<?php
declare(strict_types=1);

class WC_Custom_Checkout
{
    public function __construct()
    {
        add_filter('woocommerce_checkout_fields', [$this, 'addCustomFields']);
        add_action('woocommerce_checkout_update_order_meta', [$this, 'saveCustomFields']);
        add_action('woocommerce_admin_order_data_after_billing_address', [$this, 'displayCustomFields']);
        add_filter('woocommerce_email_order_meta_fields', [$this, 'addCustomFieldsToEmail'], 10, 3);
    }

    public function addCustomFields(array $fields): array
    {
        // เพิ่ม Tax ID (เลขผู้เสียภาษี)
        $fields['billing']['billing_tax_id'] = [
            'label' => __('Tax ID / เลขผู้เสียภาษี', 'woocommerce'),
            'placeholder' => '0123456789012',
            'required' => false,
            'class' => ['form-row-wide'],
            'clear' => true,
            'maxlength' => 13,
            'type' => 'text',
            'validate' => ['thai-tax-id'],
        ];

        // เพิ่ม Delivery Time Preference
        $fields['order']['delivery_time'] = [
            'label' => __('เวลาจัดส่งที่ต้องการ', 'woocommerce'),
            'type' => 'select',
            'options' => [
                '' => __('-- เลือกเวลา --', 'woocommerce'),
                'morning' => __('เช้า 9:00-12:00', 'woocommerce'),
                'afternoon' => __('บ่าย 12:00-17:00', 'woocommerce'),
                'evening' => __('เย็น 17:00-20:00', 'woocommerce'),
            ],
            'required' => false,
        ];

        return $fields;
    }

    public function saveCustomFields(int $order_id): void
    {
        $fields = ['billing_tax_id', 'delivery_time'];

        foreach ($fields as $field) {
            if (isset($_POST[$field])) {
                update_post_meta($order_id, '_' . $field, sanitize_text_field($_POST[$field]));
            }
        }
    }

    public function displayCustomFields(\WC_Order $order): void
    {
        $tax_id = get_post_meta($order->get_id(), '_billing_tax_id', true);
        $delivery_time = get_post_meta($order->get_id(), '_delivery_time', true);

        if ($tax_id) {
            echo '<p><strong>' . __('Tax ID:', 'woocommerce') . '</strong> ' . esc_html($tax_id) . '</p>';
        }
        if ($delivery_time) {
            echo '<p><strong>' . __('Delivery Time:', 'woocommerce') . '</strong> ' . esc_html($delivery_time) . '</p>';
        }
    }

    public function addCustomFieldsToEmail(array $fields, bool $sent_to_admin, \WC_Order $order): array
    {
        $tax_id = get_post_meta($order->get_id(), '_billing_tax_id', true);
        if ($tax_id) {
            $fields['tax_id'] = [
                'label' => __('Tax ID', 'woocommerce'),
                'value' => $tax_id,
            ];
        }
        return $fields;
    }
}

new WC_Custom_Checkout();
```

---

## ขั้นตอนที่ 2384: Inventory Management

```php
<?php
declare(strict_types=1);

class WC_Advanced_Inventory
{
    public function __construct()
    {
        add_action('woocommerce_product_set_stock', [$this, 'onStockChange'], 10, 1);
        add_action('woocommerce_reduce_order_stock', [$this, 'onOrderReduceStock'], 10, 1);
        add_filter('woocommerce_product_needs_shipping', [$this, 'checkInventoryBeforeShipping'], 10, 2);
        add_action('init', [$this, 'scheduleLowStockCheck']);
    }

    public function onStockChange(\WC_Product $product): void
    {
        $stock = $product->get_stock_quantity();
        $low_threshold = (int)get_option('woocommerce_notify_low_stock_amount', 2);

        if ($stock !== null && $stock <= $low_threshold) {
            $this->sendLowStockAlert($product, $stock);
        }

        // Log stock change
        $this->logStockChange($product->get_id(), $stock, 'manual_update');
    }

    public function onOrderReduceStock(\WC_Order $order): void
    {
        foreach ($order->get_items() as $item) {
            $product = $item->get_product();
            if ($product && $product->managing_stock()) {
                $this->logStockChange(
                    $product->get_id(),
                    $product->get_stock_quantity(),
                    'order_' . $order->get_id(),
                    $item->get_quantity()
                );
            }
        }
    }

    private function sendLowStockAlert(\WC_Product $product, int $stock): void
    {
        $admin_email = get_option('admin_email');
        $subject = sprintf(
            __('[%s] สินค้าใกล้หมด: %s', 'woocommerce'),
            get_bloginfo('name'),
            $product->get_name()
        );

        $message = sprintf(
            __("สินค้า %s มีสต็อกเหลือเพียง %d ชิ้น\nSKU: %s\nดูสินค้า: %s", 'woocommerce'),
            $product->get_name(),
            $stock,
            $product->get_sku(),
            admin_url('post.php?post=' . $product->get_id() . '&action=edit')
        );

        wp_mail($admin_email, $subject, $message);
    }

    private function logStockChange(int $product_id, ?int $new_stock, string $reason, int $quantity_changed = 0): void
    {
        global $wpdb;
        $wpdb->insert(
            $wpdb->prefix . 'stock_logs',
            [
                'product_id' => $product_id,
                'new_stock' => $new_stock,
                'quantity_changed' => $quantity_changed,
                'reason' => $reason,
                'created_by' => get_current_user_id(),
                'created_at' => current_time('mysql'),
            ],
            ['%d', '%d', '%d', '%s', '%d', '%s']
        );
    }

    public function scheduleLowStockCheck(): void
    {
        if (! wp_next_scheduled('wc_low_stock_check')) {
            wp_schedule_event(time(), 'daily', 'wc_low_stock_check');
        }
        add_action('wc_low_stock_check', [$this, 'dailyLowStockReport']);
    }

    public function dailyLowStockReport(): void
    {
        $low_threshold = (int)get_option('woocommerce_notify_low_stock_amount', 2);

        $args = [
            'post_type' => 'product',
            'posts_per_page' => -1,
            'meta_query' => [
                [
                    'key' => '_manage_stock',
                    'value' => 'yes',
                ],
                [
                    'key' => '_stock',
                    'value' => $low_threshold,
                    'compare' => '<=',
                    'type' => 'NUMERIC',
                ],
            ],
        ];

        $products = get_posts($args);

        if (! empty($products)) {
            $report = "รายงานสินค้าใกล้หมดประจำวัน\n\n";
            foreach ($products as $post) {
                $product = wc_get_product($post->ID);
                $report .= sprintf(
                    "- %s (SKU: %s) เหลือ %d ชิ้น\n",
                    $product->get_name(),
                    $product->get_sku(),
                    $product->get_stock_quantity()
                );
            }

            wp_mail(get_option('admin_email'), 'รายงานสต็อกสินค้า', $report);
        }
    }
}

new WC_Advanced_Inventory();
```

---

## ขั้นตอนที่ 2385: WooCommerce REST API Extension

```php
<?php
declare(strict_types=1);

class WC_Custom_REST_API
{
    public function __construct()
    {
        add_action('rest_api_init', [$this, 'registerRoutes']);
        add_filter('woocommerce_rest_product_object_query', [$this, 'addCustomFilters'], 10, 2);
    }

    public function registerRoutes(): void
    {
        register_rest_route('wc/v3', '/custom/inventory-report', [
            'methods' => \WP_REST_Server::READABLE,
            'callback' => [$this, 'getInventoryReport'],
            'permission_callback' => function () {
                return current_user_can('manage_woocommerce');
            },
        ]);

        register_rest_route('wc/v3', '/custom/bulk-update-prices', [
            'methods' => \WP_REST_Server::EDITABLE,
            'callback' => [$this, 'bulkUpdatePrices'],
            'permission_callback' => function () {
                return current_user_can('manage_woocommerce');
            },
            'args' => [
                'products' => [
                    'required' => true,
                    'type' => 'array',
                    'items' => [
                        'type' => 'object',
                        'properties' => [
                            'id' => ['type' => 'integer'],
                            'regular_price' => ['type' => 'string'],
                            'sale_price' => ['type' => 'string'],
                        ],
                    ],
                ],
            ],
        ]);
    }

    public function getInventoryReport(\WP_REST_Request $request): \WP_REST_Response
    {
        global $wpdb;

        $low_stock = (int)get_option('woocommerce_notify_low_stock_amount', 2);

        $results = $wpdb->get_results($wpdb->prepare("
            SELECT p.ID, p.post_title,
                   pm_stock.meta_value AS stock_quantity,
                   pm_sku.meta_value AS sku,
                   pm_price.meta_value AS price
            FROM {$wpdb->posts} p
            LEFT JOIN {$wpdb->postmeta} pm_stock ON p.ID = pm_stock.post_id AND pm_stock.meta_key = '_stock'
            LEFT JOIN {$wpdb->postmeta} pm_sku ON p.ID = pm_sku.post_id AND pm_sku.meta_key = '_sku'
            LEFT JOIN {$wpdb->postmeta} pm_price ON p.ID = pm_price.post_id AND pm_price.meta_key = '_regular_price'
            WHERE p.post_type = 'product'
            AND p.post_status = 'publish'
            AND CAST(pm_stock.meta_value AS SIGNED) <= %d
            ORDER BY CAST(pm_stock.meta_value AS SIGNED) ASC
        ", $low_stock));

        return rest_ensure_response([
            'low_stock_threshold' => $low_stock,
            'products_at_risk' => count($results),
            'products' => $results,
        ]);
    }

    public function bulkUpdatePrices(\WP_REST_Request $request): \WP_REST_Response
    {
        $products = $request->get_param('products');
        $updated = [];
        $errors = [];

        foreach ($products as $item) {
            $product = wc_get_product($item['id']);
            if (! $product) {
                $errors[] = ['id' => $item['id'], 'error' => 'Product not found'];
                continue;
            }

            if (isset($item['regular_price'])) {
                $product->set_regular_price($item['regular_price']);
            }
            if (isset($item['sale_price'])) {
                $product->set_sale_price($item['sale_price']);
            }

            $product->save();
            $updated[] = $item['id'];
        }

        return rest_ensure_response([
            'updated' => $updated,
            'errors' => $errors,
        ]);
    }

    public function addCustomFilters(array $args, \WP_REST_Request $request): array
    {
        if ($request->get_param('low_stock')) {
            $threshold = (int)get_option('woocommerce_notify_low_stock_amount', 2);
            $args['meta_query'][] = [
                'key' => '_stock',
                'value' => $threshold,
                'compare' => '<=',
                'type' => 'NUMERIC',
            ];
        }
        return $args;
    }
}

new WC_Custom_REST_API();
```

---

## สรุปบทที่ 84

| หัวข้อ | Hook/Filter | ประโยชน์ |
|--------|-----------|---------|
| Custom Product Type | product_type_selector | Bundle/Subscription |
| Digital Downloads | woocommerce_download_file_redirect | Secure download |
| Custom Checkout | woocommerce_checkout_fields | เพิ่ม Tax ID |
| Inventory | woocommerce_product_set_stock | Low stock alert |
| REST API | rest_api_init | Custom endpoints |
| Email Templates | woocommerce_email_* | Custom emails |

ถัดไป → Part 85: Drupal Commerce
