# Part 28: WordPress Plugin Development
## ขั้นตอนที่ 681-710: สร้าง Plugin ระดับมืออาชีพ

---

## ขั้นตอนที่ 681: Plugin Structure

```
my-plugin/
├── admin/
│   ├── class-admin.php
│   ├── partials/
│   │   ├── settings-page.php
│   │   └── dashboard-widget.php
│   ├── css/admin.css
│   └── js/admin.js
├── includes/
│   ├── class-plugin.php          # Main plugin class
│   ├── class-activator.php       # Activation hook
│   ├── class-deactivator.php     # Deactivation hook
│   ├── class-loader.php          # Hook registration
│   └── class-i18n.php            # Internationalization
├── public/
│   ├── class-public.php
│   ├── css/public.css
│   └── js/public.js
├── languages/
│   └── my-plugin.pot
├── my-plugin.php                 # Main plugin file
└── uninstall.php                 # Cleanup on uninstall
```

---

## ขั้นตอนที่ 682: Main Plugin File

```php
<?php
/**
 * Plugin Name:       My Awesome Plugin
 * Plugin URI:        https://example.com/my-plugin
 * Description:       A professional WordPress plugin
 * Version:           1.0.0
 * Requires at least: 6.0
 * Requires PHP:      8.1
 * Author:            Your Name
 * Author URI:        https://example.com
 * License:           GPL v2 or later
 * License URI:       https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain:       my-plugin
 * Domain Path:       /languages
 */

declare(strict_types=1);

// Security check
if (!defined('ABSPATH')) {
    exit;
}

// Plugin constants
define('MY_PLUGIN_VERSION',  '1.0.0');
define('MY_PLUGIN_BASENAME', plugin_basename(__FILE__));
define('MY_PLUGIN_DIR',      plugin_dir_path(__FILE__));
define('MY_PLUGIN_URL',      plugin_dir_url(__FILE__));
define('MY_PLUGIN_MIN_PHP',  '8.1');
define('MY_PLUGIN_MIN_WP',   '6.0');

// Check PHP version
if (version_compare(PHP_VERSION, MY_PLUGIN_MIN_PHP, '<')) {
    add_action('admin_notices', function(): void {
        printf(
            '<div class="error"><p>%s</p></div>',
            sprintf(
                esc_html__('My Plugin requires PHP %s or higher.', 'my-plugin'),
                MY_PLUGIN_MIN_PHP
            )
        );
    });
    return;
}

// Autoload
require_once MY_PLUGIN_DIR . 'vendor/autoload.php';

// Activation/Deactivation hooks
register_activation_hook(__FILE__, ['\MyPlugin\Activator', 'activate']);
register_deactivation_hook(__FILE__, ['\MyPlugin\Deactivator', 'deactivate']);

// Bootstrap
function my_plugin(): \MyPlugin\Plugin {
    return \MyPlugin\Plugin::get_instance();
}

my_plugin()->run();
```

---

## ขั้นตอนที่ 683: Main Plugin Class (Singleton)

```php
<?php
// includes/class-plugin.php

declare(strict_types=1);

namespace MyPlugin;

class Plugin {
    private static ?self $instance = null;
    private Loader $loader;
    
    private function __construct() {
        $this->loader = new Loader();
        $this->set_locale();
        $this->define_admin_hooks();
        $this->define_public_hooks();
    }
    
    public static function get_instance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    private function set_locale(): void {
        $i18n = new I18n();
        $this->loader->add_action('plugins_loaded', $i18n, 'load_plugin_textdomain');
    }
    
    private function define_admin_hooks(): void {
        $admin = new Admin\Admin(MY_PLUGIN_VERSION);
        
        $this->loader->add_action('admin_enqueue_scripts', $admin, 'enqueue_styles');
        $this->loader->add_action('admin_enqueue_scripts', $admin, 'enqueue_scripts');
        $this->loader->add_action('admin_menu', $admin, 'register_menus');
        $this->loader->add_action('admin_init', $admin, 'register_settings');
        
        // Dashboard widget
        $this->loader->add_action('wp_dashboard_setup', $admin, 'add_dashboard_widget');
        
        // Plugin action links
        $this->loader->add_filter(
            'plugin_action_links_' . MY_PLUGIN_BASENAME,
            $admin,
            'add_action_links'
        );
    }
    
    private function define_public_hooks(): void {
        $public = new PublicFacing\PublicFacing(MY_PLUGIN_VERSION);
        
        $this->loader->add_action('wp_enqueue_scripts', $public, 'enqueue_styles');
        $this->loader->add_action('wp_enqueue_scripts', $public, 'enqueue_scripts');
        
        // Register shortcodes
        $shortcodes = new Shortcodes();
        $this->loader->add_action('init', $shortcodes, 'register_all');
        
        // REST API
        $api = new Api\RestController();
        $this->loader->add_action('rest_api_init', $api, 'register_routes');
        
        // Custom post type
        $cpt = new PostTypes\ReviewPostType();
        $this->loader->add_action('init', $cpt, 'register');
    }
    
    public function run(): void {
        $this->loader->run();
    }
    
    public function get_loader(): Loader {
        return $this->loader;
    }
}
```

---

## ขั้นตอนที่ 684: Settings API

```php
<?php
// admin/class-admin.php

declare(strict_types=1);

namespace MyPlugin\Admin;

class Admin {
    private const SETTINGS_GROUP = 'my_plugin_settings';
    private const OPTION_NAME    = 'my_plugin_options';
    
    public function __construct(private string $version) {}
    
    public function register_menus(): void {
        add_menu_page(
            __('My Plugin', 'my-plugin'),
            __('My Plugin', 'my-plugin'),
            'manage_options',
            'my-plugin',
            [$this, 'render_main_page'],
            'dashicons-star-filled',
            80
        );
        
        add_submenu_page(
            'my-plugin',
            __('Settings', 'my-plugin'),
            __('Settings', 'my-plugin'),
            'manage_options',
            'my-plugin-settings',
            [$this, 'render_settings_page']
        );
        
        add_submenu_page(
            'my-plugin',
            __('Reports', 'my-plugin'),
            __('Reports', 'my-plugin'),
            'manage_options',
            'my-plugin-reports',
            [$this, 'render_reports_page']
        );
    }
    
    public function register_settings(): void {
        register_setting(
            self::SETTINGS_GROUP,
            self::OPTION_NAME,
            [
                'sanitize_callback' => [$this, 'sanitize_settings'],
                'default'           => $this->get_defaults(),
            ]
        );
        
        // Section: General
        add_settings_section(
            'my_plugin_general',
            __('General Settings', 'my-plugin'),
            fn() => echo '<p>' . esc_html__('Configure general settings.', 'my-plugin') . '</p>',
            'my-plugin-settings'
        );
        
        // Field: API Key
        add_settings_field(
            'api_key',
            __('API Key', 'my-plugin'),
            [$this, 'render_api_key_field'],
            'my-plugin-settings',
            'my_plugin_general'
        );
        
        // Field: Enable Feature
        add_settings_field(
            'enable_feature',
            __('Enable Feature', 'my-plugin'),
            [$this, 'render_enable_feature_field'],
            'my-plugin-settings',
            'my_plugin_general'
        );
        
        // Section: Display
        add_settings_section(
            'my_plugin_display',
            __('Display Options', 'my-plugin'),
            null,
            'my-plugin-settings'
        );
        
        add_settings_field(
            'items_per_page',
            __('Items Per Page', 'my-plugin'),
            [$this, 'render_items_per_page_field'],
            'my-plugin-settings',
            'my_plugin_display'
        );
    }
    
    public function sanitize_settings(array $input): array {
        $sanitized = [];
        
        $sanitized['api_key'] = sanitize_text_field($input['api_key'] ?? '');
        $sanitized['enable_feature'] = !empty($input['enable_feature']);
        $sanitized['items_per_page'] = min(100, max(1, absint($input['items_per_page'] ?? 10)));
        
        if (!empty($input['api_key']) && !preg_match('/^[a-zA-Z0-9_-]{32}$/', $sanitized['api_key'])) {
            add_settings_error(
                'my_plugin_options',
                'invalid_api_key',
                __('Invalid API key format.', 'my-plugin')
            );
        }
        
        return $sanitized;
    }
    
    public function render_api_key_field(): void {
        $options = get_option(self::OPTION_NAME, $this->get_defaults());
        printf(
            '<input type="text" id="api_key" name="%s[api_key]" value="%s" class="regular-text">',
            esc_attr(self::OPTION_NAME),
            esc_attr($options['api_key'])
        );
        echo '<p class="description">' . esc_html__('Enter your API key.', 'my-plugin') . '</p>';
    }
    
    public function render_enable_feature_field(): void {
        $options = get_option(self::OPTION_NAME, $this->get_defaults());
        printf(
            '<input type="checkbox" id="enable_feature" name="%s[enable_feature]" %s>',
            esc_attr(self::OPTION_NAME),
            checked($options['enable_feature'], true, false)
        );
        echo '<label for="enable_feature">' . esc_html__('Enable this feature', 'my-plugin') . '</label>';
    }
    
    public function render_items_per_page_field(): void {
        $options = get_option(self::OPTION_NAME, $this->get_defaults());
        printf(
            '<input type="number" id="items_per_page" name="%s[items_per_page]" value="%d" min="1" max="100" class="small-text">',
            esc_attr(self::OPTION_NAME),
            absint($options['items_per_page'])
        );
    }
    
    public function render_settings_page(): void {
        if (!current_user_can('manage_options')) {
            return;
        }
        
        include MY_PLUGIN_DIR . 'admin/partials/settings-page.php';
    }
    
    private function get_defaults(): array {
        return [
            'api_key'        => '',
            'enable_feature' => false,
            'items_per_page' => 10,
        ];
    }
    
    public function add_action_links(array $links): array {
        $plugin_links = [
            '<a href="' . admin_url('admin.php?page=my-plugin-settings') . '">' .
            esc_html__('Settings', 'my-plugin') . '</a>',
        ];
        return array_merge($plugin_links, $links);
    }
}
```

---

## ขั้นตอนที่ 685: Activator & Database Table

```php
<?php
// includes/class-activator.php

declare(strict_types=1);

namespace MyPlugin;

class Activator {
    public static function activate(): void {
        self::create_tables();
        self::set_defaults();
        self::schedule_cron();
        
        flush_rewrite_rules();
        
        // Set activation flag for redirect
        add_option('my_plugin_activated', true);
    }
    
    private static function create_tables(): void {
        global $wpdb;
        
        $charset_collate = $wpdb->get_charset_collate();
        $table_name      = $wpdb->prefix . 'my_plugin_reviews';
        
        $sql = "CREATE TABLE {$table_name} (
            id bigint(20) unsigned NOT NULL AUTO_INCREMENT,
            post_id bigint(20) unsigned NOT NULL,
            user_id bigint(20) unsigned NOT NULL,
            rating tinyint(1) unsigned NOT NULL DEFAULT 0,
            comment text NOT NULL,
            status varchar(20) NOT NULL DEFAULT 'pending',
            ip_address varchar(45) NOT NULL,
            created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            updated_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY post_id (post_id),
            KEY user_id (user_id),
            KEY status (status)
        ) {$charset_collate};";
        
        require_once ABSPATH . 'wp-admin/includes/upgrade.php';
        dbDelta($sql);
        
        // Save DB version for future migrations
        update_option('my_plugin_db_version', '1.0.0');
    }
    
    private static function set_defaults(): void {
        $defaults = [
            'api_key'        => '',
            'enable_feature' => false,
            'items_per_page' => 10,
        ];
        
        add_option('my_plugin_options', $defaults);
    }
    
    private static function schedule_cron(): void {
        if (!wp_next_scheduled('my_plugin_daily_cleanup')) {
            wp_schedule_event(time(), 'daily', 'my_plugin_daily_cleanup');
        }
    }
}

// Deactivator
class Deactivator {
    public static function deactivate(): void {
        wp_clear_scheduled_hook('my_plugin_daily_cleanup');
        flush_rewrite_rules();
    }
}

// uninstall.php (runs on plugin deletion)
// if (!defined('WP_UNINSTALL_PLUGIN')) exit;
// global $wpdb;
// delete_option('my_plugin_options');
// delete_option('my_plugin_db_version');
// $wpdb->query("DROP TABLE IF EXISTS {$wpdb->prefix}my_plugin_reviews");
```

---

## ขั้นตอนที่ 686: Shortcodes

```php
<?php
// includes/class-shortcodes.php

declare(strict_types=1);

namespace MyPlugin;

class Shortcodes {
    public function register_all(): void {
        add_shortcode('my_reviews', [$this, 'render_reviews']);
        add_shortcode('my_rating', [$this, 'render_rating']);
        add_shortcode('my_stats', [$this, 'render_stats']);
    }
    
    public function render_reviews(array $atts): string {
        $atts = shortcode_atts([
            'post_id'  => get_the_ID(),
            'limit'    => 5,
            'status'   => 'approved',
            'orderby'  => 'date',
        ], $atts, 'my_reviews');
        
        // Sanitize
        $post_id = absint($atts['post_id']);
        $limit   = min(50, absint($atts['limit']));
        $status  = sanitize_key($atts['status']);
        
        global $wpdb;
        $table = $wpdb->prefix . 'my_plugin_reviews';
        
        // phpcs:ignore WordPress.DB.PreparedSQL.InterpolatedNotPrepared
        $reviews = $wpdb->get_results(
            $wpdb->prepare(
                "SELECT * FROM {$table} WHERE post_id = %d AND status = %s ORDER BY created_at DESC LIMIT %d",
                $post_id, $status, $limit
            )
        );
        
        ob_start();
        include MY_PLUGIN_DIR . 'public/partials/reviews-list.php';
        return ob_get_clean();
    }
    
    public function render_rating(array $atts): string {
        $atts = shortcode_atts([
            'post_id' => get_the_ID(),
            'style'   => 'stars',
        ], $atts, 'my_rating');
        
        global $wpdb;
        $table = $wpdb->prefix . 'my_plugin_reviews';
        $post_id = absint($atts['post_id']);
        
        $result = $wpdb->get_row(
            $wpdb->prepare(
                "SELECT AVG(rating) as avg_rating, COUNT(*) as total FROM {$table} WHERE post_id = %d AND status = 'approved'",
                $post_id
            )
        );
        
        $avg = round((float) ($result->avg_rating ?? 0), 1);
        $total = absint($result->total ?? 0);
        
        return sprintf(
            '<div class="my-plugin-rating" data-rating="%s" data-total="%d">
                <span class="stars">%s</span>
                <span class="count">(%d %s)</span>
             </div>',
            esc_attr($avg),
            $total,
            esc_html(str_repeat('★', (int) round($avg)) . str_repeat('☆', 5 - (int) round($avg))),
            $total,
            esc_html(_n('review', 'reviews', $total, 'my-plugin'))
        );
    }
}
```

---

## 🎯 สรุป Part 28

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Plugin Structure | MVC-like structure |
| Main Plugin Class | Singleton, Loader pattern |
| Settings API | Sections, Fields, Sanitize |
| Activator | DB tables, default options |
| Shortcodes | shortcode_atts, ob_start |
| Security | nonce, sanitize, escape |

**ถัดไป → Part 29: WordPress REST API & Gutenberg Blocks**
