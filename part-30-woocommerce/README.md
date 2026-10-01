# Part 30: WooCommerce Development
## ขั้นตอนที่ 741-770: E-Commerce ด้วย WooCommerce

---

## ขั้นตอนที่ 741: WooCommerce Setup & Hooks

```php
<?php
declare(strict_types=1);

// ตรวจสอบว่า WooCommerce active
if (!class_exists('WooCommerce')) {
    add_action('admin_notices', function(): void {
        echo '<div class="error"><p>' .
            esc_html__('This plugin requires WooCommerce.', 'my-plugin') .
            '</p></div>';
    });
    return;
}

// WooCommerce HPOS Compatibility
add_action('before_woocommerce_init', function(): void {
    if (class_exists(\Automattic\WooCommerce\Utilities\FeaturesUtil::class)) {
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
            'custom_order_tables',
            __FILE__,
            true
        );
    }
});

// Declare Cart & Checkout Blocks compatibility
add_action('before_woocommerce_init', function(): void {
    if (class_exists('\Automattic\WooCommerce\Utilities\FeaturesUtil')) {
        \Automattic\WooCommerce\Utilities\FeaturesUtil::declare_compatibility(
            'cart_checkout_blocks',
            __FILE__,
            true
        );
    }
});
```

---

## ขั้นตอนที่ 742: Custom Product Type

```php
<?php
declare(strict_types=1);

/**
 * Register custom product type: Subscription
 */
add_filter('product_type_selector', function(array $types): array {
    $types['subscription'] = __('Subscription Product', 'my-plugin');
    return $types;
});

// Load product class
add_filter('woocommerce_product_class', function(string $classname, string $product_type): string {
    return $product_type === 'subscription' ? WC_Product_Subscription::class : $classname;
}, 10, 2);

class WC_Product_Subscription extends WC_Product {
    public function __construct($product) {
        $this->product_type = 'subscription';
        parent::__construct($product);
    }
    
    public function get_type(): string {
        return 'subscription';
    }
    
    public function get_billing_period(): string {
        return $this->get_meta('_billing_period', true) ?: 'month';
    }
    
    public function get_billing_interval(): int {
        return absint($this->get_meta('_billing_interval', true)) ?: 1;
    }
    
    public function get_subscription_price_html(): string {
        $price = wc_price($this->get_price());
        $interval = $this->get_billing_interval();
        $period = $this->get_billing_period();
        
        return sprintf(
            __('%s / every %d %s', 'my-plugin'),
            $price,
            $interval,
            $period
        );
    }
}

// Add product type options in edit product page
add_action('woocommerce_product_options_general_product_data', function(): void {
    global $post;
    $product = wc_get_product($post->ID);
    
    if (!$product || $product->get_type() !== 'subscription') return;
    ?>
    <div class="options_group show_if_subscription">
        <?php
        woocommerce_wp_select([
            'id'      => '_billing_period',
            'label'   => __('Billing Period', 'my-plugin'),
            'options' => [
                'day'   => __('Daily', 'my-plugin'),
                'week'  => __('Weekly', 'my-plugin'),
                'month' => __('Monthly', 'my-plugin'),
                'year'  => __('Yearly', 'my-plugin'),
            ],
            'value'   => $product->get_meta('_billing_period') ?: 'month',
        ]);
        
        woocommerce_wp_text_input([
            'id'                => '_billing_interval',
            'label'             => __('Billing Interval', 'my-plugin'),
            'placeholder'       => '1',
            'type'              => 'number',
            'custom_attributes' => ['min' => '1', 'max' => '12'],
            'value'             => $product->get_meta('_billing_interval') ?: 1,
        ]);
        ?>
    </div>
    <?php
});

// Save product meta
add_action('woocommerce_process_product_meta_subscription', function(int $post_id): void {
    $product = wc_get_product($post_id);
    
    $billing_period   = sanitize_key($_POST['_billing_period'] ?? 'month');
    $billing_interval = min(12, max(1, absint($_POST['_billing_interval'] ?? 1)));
    
    $product->update_meta_data('_billing_period', $billing_period);
    $product->update_meta_data('_billing_interval', $billing_interval);
    $product->save();
});
```

---

## ขั้นตอนที่ 743: Cart & Order Hooks

```php
<?php
declare(strict_types=1);

/**
 * Add custom fee to cart
 */
add_action('woocommerce_cart_calculate_fees', function(WC_Cart $cart): void {
    if (is_admin() && !defined('DOING_AJAX')) return;
    
    $subtotal = $cart->get_subtotal();
    
    // Add handling fee for orders under minimum
    if ($subtotal < 500) {
        $cart->add_fee(__('Handling Fee', 'my-plugin'), 50.00, true);
    }
    
    // Add gift wrapping fee if selected
    if (WC()->session->get('gift_wrapping')) {
        $cart->add_fee(__('Gift Wrapping', 'my-plugin'), 99.00, false);
    }
});

// Gift wrapping option in cart
add_action('woocommerce_before_cart_totals', function(): void {
    $enabled = WC()->session->get('gift_wrapping', false);
    ?>
    <div class="gift-wrapping-option">
        <label>
            <input type="checkbox" id="gift_wrapping" <?= checked($enabled) ?>>
            <?= esc_html__('Add Gift Wrapping (+฿99)', 'my-plugin') ?>
        </label>
    </div>
    <?php
});

// Handle AJAX toggle
add_action('wp_ajax_toggle_gift_wrapping', 'handle_toggle_gift_wrapping');
add_action('wp_ajax_nopriv_toggle_gift_wrapping', 'handle_toggle_gift_wrapping');

function handle_toggle_gift_wrapping(): void {
    if (!wp_verify_nonce($_POST['nonce'] ?? '', 'woocommerce-cart')) {
        wp_send_json_error([], 403);
    }
    
    $enabled = filter_var($_POST['enabled'] ?? false, FILTER_VALIDATE_BOOLEAN);
    WC()->session->set('gift_wrapping', $enabled);
    
    WC()->cart->calculate_totals();
    
    wp_send_json_success([
        'total' => WC()->cart->get_total(),
    ]);
}

/**
 * Custom order status
 */
add_action('init', function(): void {
    register_post_status('wc-awaiting-shipment', [
        'label'                     => __('Awaiting Shipment', 'my-plugin'),
        'public'                    => true,
        'exclude_from_search'       => false,
        'show_in_admin_all_list'    => true,
        'show_in_admin_status_list' => true,
        /* translators: number of orders */
        'label_count'               => _n_noop('Awaiting Shipment <span class="count">(%s)</span>', 'Awaiting Shipment <span class="count">(%s)</span>', 'my-plugin'),
    ]);
});

add_filter('wc_order_statuses', function(array $statuses): array {
    $new_statuses = [];
    
    foreach ($statuses as $key => $status) {
        $new_statuses[$key] = $status;
        if ($key === 'wc-processing') {
            $new_statuses['wc-awaiting-shipment'] = __('Awaiting Shipment', 'my-plugin');
        }
    }
    
    return $new_statuses;
});

/**
 * Order meta - custom fields
 */
add_action('woocommerce_checkout_order_created', function(WC_Order $order): void {
    // Save delivery note from session
    $delivery_note = sanitize_textarea_field(
        WC()->session->get('delivery_note', '')
    );
    
    if ($delivery_note) {
        $order->update_meta_data('_delivery_note', $delivery_note);
        $order->save();
    }
});

// Show delivery note in admin
add_action('woocommerce_admin_order_data_after_billing_address', function(WC_Order $order): void {
    $note = $order->get_meta('_delivery_note');
    if (!$note) return;
    ?>
    <div class="delivery-note">
        <h4><?= esc_html__('Delivery Note:', 'my-plugin') ?></h4>
        <p><?= esc_html($note) ?></p>
    </div>
    <?php
});
```

---

## ขั้นตอนที่ 744: Custom Payment Gateway

```php
<?php
declare(strict_types=1);

add_filter('woocommerce_payment_gateways', function(array $gateways): array {
    $gateways[] = WC_Gateway_PromptPay::class;
    return $gateways;
});

class WC_Gateway_PromptPay extends WC_Payment_Gateway {
    public function __construct() {
        $this->id                 = 'promptpay';
        $this->icon               = plugins_url('assets/images/promptpay.png', __FILE__);
        $this->has_fields         = false;
        $this->method_title       = __('PromptPay', 'my-plugin');
        $this->method_description = __('รับชำระเงินผ่าน PromptPay QR Code', 'my-plugin');
        $this->supports           = ['products', 'refunds'];
        
        $this->init_form_fields();
        $this->init_settings();
        
        $this->title       = $this->get_option('title');
        $this->description = $this->get_option('description');
        $this->promptpay_id = $this->get_option('promptpay_id');
        $this->enabled     = $this->get_option('enabled');
        
        add_action('woocommerce_update_options_payment_gateways_' . $this->id, [$this, 'process_admin_options']);
        add_action('woocommerce_thankyou_' . $this->id, [$this, 'thankyou_page']);
        add_action('woocommerce_email_before_order_table', [$this, 'email_instructions'], 10, 3);
    }
    
    public function init_form_fields(): void {
        $this->form_fields = [
            'enabled' => [
                'title'   => __('Enable/Disable', 'my-plugin'),
                'type'    => 'checkbox',
                'label'   => __('Enable PromptPay', 'my-plugin'),
                'default' => 'yes',
            ],
            'title' => [
                'title'       => __('Title', 'my-plugin'),
                'type'        => 'text',
                'default'     => __('PromptPay', 'my-plugin'),
                'desc_tip'    => true,
                'description' => __('Payment title shown to customer.', 'my-plugin'),
            ],
            'description' => [
                'title'   => __('Description', 'my-plugin'),
                'type'    => 'textarea',
                'default' => __('โอนเงินผ่าน PromptPay แล้วส่งสลิปให้เรา', 'my-plugin'),
            ],
            'promptpay_id' => [
                'title'       => __('PromptPay ID', 'my-plugin'),
                'type'        => 'text',
                'description' => __('เบอร์โทรหรือเลขบัตรประชาชน', 'my-plugin'),
                'desc_tip'    => true,
            ],
        ];
    }
    
    public function process_payment(int $order_id): array {
        $order = wc_get_order($order_id);
        
        // Mark as pending (awaiting slip verification)
        $order->update_status('pending', __('Awaiting PromptPay payment.', 'my-plugin'));
        
        // Reduce stock
        wc_reduce_stock_levels($order_id);
        
        // Empty cart
        WC()->cart->empty_cart();
        
        return [
            'result'   => 'success',
            'redirect' => $this->get_return_url($order),
        ];
    }
    
    public function process_refund(int $order_id, float $amount = null, string $reason = ''): bool|\WP_Error {
        $order = wc_get_order($order_id);
        
        if (!$order) return new \WP_Error('invalid_order', 'Invalid order');
        
        // For manual gateways, refunds are manual
        $order->add_order_note(
            sprintf(
                __('Refund of %s requested. Reason: %s', 'my-plugin'),
                wc_price($amount),
                $reason ?: __('No reason provided', 'my-plugin')
            )
        );
        
        return true;
    }
    
    public function thankyou_page(int $order_id): void {
        if ($this->instructions) {
            echo wpautop(wptexturize(esc_html($this->instructions)));
        }
        ?>
        <div class="promptpay-instructions">
            <h3><?= esc_html__('Payment Instructions', 'my-plugin') ?></h3>
            <p><?= esc_html__('Please transfer the exact amount and send slip to:', 'my-plugin') ?></p>
            <img src="<?= esc_url(plugins_url('assets/images/qr-code.png', __FILE__)) ?>"
                 alt="PromptPay QR Code">
            <p><strong><?= esc_html($this->promptpay_id) ?></strong></p>
        </div>
        <?php
    }
}
```

---

## ขั้นตอนที่ 745: WooCommerce REST API Extension

```php
<?php
declare(strict_types=1);

/**
 * Extend WooCommerce REST API
 */

// Add custom field to product REST API response
add_filter('woocommerce_rest_prepare_product_object', function(
    \WP_REST_Response $response,
    WC_Product $product,
    \WP_REST_Request $request
): \WP_REST_Response {
    $data = $response->get_data();
    
    $data['custom_fields'] = [
        'material'    => $product->get_meta('_material'),
        'origin'      => $product->get_meta('_origin'),
        'warranty'    => $product->get_meta('_warranty_months') . ' months',
        'view_count'  => absint($product->get_meta('_view_count')),
    ];
    
    $response->set_data($data);
    return $response;
}, 10, 3);

// Make custom fields writable via REST API
add_filter('woocommerce_rest_product_schema', function(array $schema): array {
    $schema['properties']['custom_fields'] = [
        'description' => __('Custom product fields.', 'my-plugin'),
        'type'        => 'object',
        'context'     => ['view', 'edit'],
        'properties'  => [
            'material' => ['type' => 'string'],
            'origin'   => ['type' => 'string'],
        ],
    ];
    return $schema;
});

add_action('woocommerce_rest_insert_product_object', function(
    WC_Product $product,
    \WP_REST_Request $request,
    bool $creating
): void {
    $custom_fields = $request->get_param('custom_fields') ?? [];
    
    if (isset($custom_fields['material'])) {
        $product->update_meta_data('_material', sanitize_text_field($custom_fields['material']));
    }
    if (isset($custom_fields['origin'])) {
        $product->update_meta_data('_origin', sanitize_text_field($custom_fields['origin']));
    }
    
    $product->save();
}, 10, 3);
```

---

## 🎯 สรุป Part 30

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Custom Product Type | extend WC_Product |
| Cart Fees | add_fee, calculate_fees hook |
| Custom Order Status | register_post_status |
| Payment Gateway | WC_Payment_Gateway |
| REST API Extension | woocommerce_rest_prepare filters |
| HPOS Compatibility | FeaturesUtil::declare_compatibility |

**ถัดไป → Part 31: Drupal Basics**
