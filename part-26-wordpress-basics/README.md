# Part 26: WordPress Development พื้นฐาน
## ขั้นตอนที่ 631-660: WordPress Theme & Plugin Development

---

## ขั้นตอนที่ 631: WordPress คืออะไรและ Architecture

WordPress เป็น CMS (Content Management System) ที่ใช้งานมากที่สุดในโลก (~43% ของเว็บทั้งหมด)

### WordPress Architecture
```
WordPress Core
├── wp-admin/          # Admin Dashboard
├── wp-content/
│   ├── themes/        # Themes
│   ├── plugins/       # Plugins
│   └── uploads/       # Uploaded Media
├── wp-includes/       # Core PHP Libraries
├── wp-config.php      # Configuration
├── wp-login.php       # Login Page
└── index.php          # Entry Point
```

### WordPress Database Tables
```sql
wp_posts           -- Posts, Pages, Custom Post Types
wp_postmeta        -- Post Metadata
wp_users           -- Users
wp_usermeta        -- User Metadata
wp_terms           -- Tags, Categories
wp_term_taxonomy   -- Term Taxonomy
wp_term_relationships  -- Post-Term Relations
wp_options         -- Site Settings
wp_comments        -- Comments
wp_commentmeta     -- Comment Metadata
```

---

## ขั้นตอนที่ 632: WordPress Theme Development

### Theme File Structure
```
my-theme/
├── style.css              # Theme Header + Main CSS
├── index.php              # Main Template
├── functions.php          # Theme Functions
├── header.php             # Header Template
├── footer.php             # Footer Template
├── sidebar.php            # Sidebar Template
├── single.php             # Single Post
├── page.php               # Single Page
├── archive.php            # Archive (Category, Tag, Date)
├── search.php             # Search Results
├── 404.php                # 404 Error
├── front-page.php         # Homepage (static front page)
├── home.php               # Blog Index
├── category.php           # Category Archive
├── tag.php                # Tag Archive
├── author.php             # Author Archive
├── date.php               # Date Archive
├── attachment.php         # Attachment Page
├── comments.php           # Comments Template
├── searchform.php         # Search Form
├── screenshot.png         # Theme Preview (1200x900)
└── assets/
    ├── css/
    ├── js/
    └── images/
```

### style.css Theme Header
```css
/*
Theme Name: My Custom Theme
Theme URI: https://example.com/my-theme
Author: Your Name
Author URI: https://example.com
Description: A custom WordPress theme built for learning
Version: 1.0.0
Requires at least: 6.0
Requires PHP: 8.0
License: GNU General Public License v2 or later
License URI: https://www.gnu.org/licenses/gpl-2.0.html
Text Domain: my-theme
Tags: blog, custom-colors, custom-logo, custom-menu, two-columns
*/

/* Main Stylesheet */
*, *::before, *::after {
    box-sizing: border-box;
}

:root {
    --color-primary: #0073aa;
    --color-secondary: #005177;
    --color-text: #333;
    --color-bg: #fff;
    --font-size-base: 16px;
    --max-width: 1200px;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    font-size: var(--font-size-base);
    color: var(--color-text);
    background: var(--color-bg);
    margin: 0;
    padding: 0;
}

.container {
    max-width: var(--max-width);
    margin: 0 auto;
    padding: 0 20px;
}
```

---

## ขั้นตอนที่ 633: functions.php - Theme Setup

```php
<?php
// functions.php

if (!defined('ABSPATH')) exit;

// ========================
// Theme Setup
// ========================
function mytheme_setup(): void {
    // ภาษา
    load_theme_textdomain('my-theme', get_template_directory() . '/languages');
    
    // HTML5 Support
    add_theme_support('html5', [
        'search-form', 'comment-form', 'comment-list', 
        'gallery', 'caption', 'style', 'script'
    ]);
    
    // Post Thumbnails
    add_theme_support('post-thumbnails');
    set_post_thumbnail_size(1200, 628, true);
    add_image_size('my-theme-small', 400, 300, true);
    add_image_size('my-theme-medium', 800, 600, true);
    
    // Title Tag
    add_theme_support('title-tag');
    
    // Custom Logo
    add_theme_support('custom-logo', [
        'height' => 100,
        'width' => 300,
        'flex-height' => true,
        'flex-width' => true,
        'header-text' => ['site-title', 'site-description'],
    ]);
    
    // Custom Header
    add_theme_support('custom-header', [
        'default-image' => '',
        'default-text-color' => '000',
        'width' => 1920,
        'height' => 600,
        'flex-height' => true,
    ]);
    
    // Custom Background
    add_theme_support('custom-background');
    
    // Automatic Feed Links
    add_theme_support('automatic-feed-links');
    
    // WooCommerce
    add_theme_support('woocommerce');
    add_theme_support('wc-product-gallery-zoom');
    add_theme_support('wc-product-gallery-lightbox');
    
    // Block Editor (Gutenberg)
    add_theme_support('editor-styles');
    add_theme_support('responsive-embeds');
    add_theme_support('align-wide');
    
    // Menus
    register_nav_menus([
        'primary' => __('Primary Menu', 'my-theme'),
        'footer' => __('Footer Menu', 'my-theme'),
        'mobile' => __('Mobile Menu', 'my-theme'),
    ]);
    
    // Widget Areas (Sidebars)
    // จะ register ใน function แยก
}
add_action('after_setup_theme', 'mytheme_setup');

// ========================
// Register Sidebars
// ========================
function mytheme_widgets_init(): void {
    register_sidebar([
        'name'          => __('Main Sidebar', 'my-theme'),
        'id'            => 'sidebar-1',
        'description'   => __('Main sidebar widgets', 'my-theme'),
        'before_widget' => '<section id="%1$s" class="widget %2$s">',
        'after_widget'  => '</section>',
        'before_title'  => '<h2 class="widget-title">',
        'after_title'   => '</h2>',
    ]);
    
    register_sidebar([
        'name'          => __('Footer Widget Area', 'my-theme'),
        'id'            => 'footer-1',
        'before_widget' => '<div id="%1$s" class="footer-widget %2$s">',
        'after_widget'  => '</div>',
        'before_title'  => '<h3 class="footer-widget-title">',
        'after_title'   => '</h3>',
    ]);
}
add_action('widgets_init', 'mytheme_widgets_init');

// ========================
// Enqueue Scripts & Styles
// ========================
function mytheme_scripts(): void {
    $theme_version = wp_get_theme()->get('Version');
    
    // Main Stylesheet
    wp_enqueue_style(
        'my-theme-style',
        get_stylesheet_uri(),
        [],
        $theme_version
    );
    
    // Google Fonts
    wp_enqueue_style(
        'google-fonts',
        'https://fonts.googleapis.com/css2?family=Sarabun:wght@400;500;700&display=swap',
        [],
        null
    );
    
    // Bootstrap (Optional)
    wp_enqueue_style(
        'bootstrap',
        'https://cdnjs.cloudflare.com/ajax/libs/bootstrap/5.3.0/css/bootstrap.min.css',
        [],
        '5.3.0'
    );
    
    // Custom CSS
    wp_enqueue_style(
        'my-theme-custom',
        get_template_directory_uri() . '/assets/css/custom.css',
        ['my-theme-style'],
        $theme_version
    );
    
    // Navigation Script
    wp_enqueue_script(
        'my-theme-navigation',
        get_template_directory_uri() . '/assets/js/navigation.js',
        [],
        $theme_version,
        true  // In footer
    );
    
    // Main JS
    wp_enqueue_script(
        'my-theme-main',
        get_template_directory_uri() . '/assets/js/main.js',
        ['jquery'],
        $theme_version,
        true
    );
    
    // Pass PHP data to JavaScript
    wp_localize_script('my-theme-main', 'mythemeData', [
        'ajaxUrl' => admin_url('admin-ajax.php'),
        'nonce' => wp_create_nonce('mytheme_nonce'),
        'homeUrl' => home_url(),
        'theme' => [
            'name' => get_bloginfo('name'),
            'url' => get_template_directory_uri(),
        ],
        'i18n' => [
            'loading' => __('Loading...', 'my-theme'),
            'error' => __('An error occurred', 'my-theme'),
        ],
    ]);
    
    // Comments Script
    if (is_singular() && comments_open() && get_option('thread_comments')) {
        wp_enqueue_script('comment-reply');
    }
}
add_action('wp_enqueue_scripts', 'mytheme_scripts');

// ========================
// Custom Post Types
// ========================
function mytheme_register_post_types(): void {
    // Portfolio
    register_post_type('portfolio', [
        'labels' => [
            'name' => __('Portfolio', 'my-theme'),
            'singular_name' => __('Portfolio Item', 'my-theme'),
            'add_new_item' => __('Add New Portfolio Item', 'my-theme'),
            'edit_item' => __('Edit Portfolio Item', 'my-theme'),
        ],
        'public' => true,
        'has_archive' => true,
        'show_in_rest' => true,   // Gutenberg support
        'supports' => ['title', 'editor', 'thumbnail', 'excerpt'],
        'menu_icon' => 'dashicons-portfolio',
        'rewrite' => ['slug' => 'portfolio'],
        'menu_position' => 5,
    ]);
    
    // Testimonial
    register_post_type('testimonial', [
        'labels' => [
            'name' => __('Testimonials', 'my-theme'),
            'singular_name' => __('Testimonial', 'my-theme'),
        ],
        'public' => false,
        'show_ui' => true,
        'show_in_rest' => true,
        'supports' => ['title', 'editor', 'thumbnail'],
        'menu_icon' => 'dashicons-format-quote',
    ]);
}
add_action('init', 'mytheme_register_post_types');

// ========================
// Custom Taxonomies
// ========================
function mytheme_register_taxonomies(): void {
    // Portfolio Category
    register_taxonomy('portfolio_cat', 'portfolio', [
        'hierarchical' => true,
        'labels' => [
            'name' => __('Portfolio Categories', 'my-theme'),
            'singular_name' => __('Portfolio Category', 'my-theme'),
        ],
        'show_in_rest' => true,
        'rewrite' => ['slug' => 'portfolio-category'],
    ]);
}
add_action('init', 'mytheme_register_taxonomies');
```

---

## ขั้นตอนที่ 634: Template Files

### header.php
```php
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
    <meta charset="<?php bloginfo('charset'); ?>">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="profile" href="https://gmpg.org/xfn/11">
    <?php wp_head(); ?>
</head>
<body <?php body_class(); ?>>
    <?php wp_body_open(); ?>
    
    <div id="page" class="site">
        <header id="masthead" class="site-header">
            <div class="container">
                <div class="site-branding">
                    <?php if (has_custom_logo()): ?>
                        <?php the_custom_logo(); ?>
                    <?php else: ?>
                        <?php if (is_front_page() && is_home()): ?>
                            <h1 class="site-title">
                                <a href="<?php echo esc_url(home_url('/')); ?>" rel="home">
                                    <?php bloginfo('name'); ?>
                                </a>
                            </h1>
                        <?php else: ?>
                            <p class="site-title">
                                <a href="<?php echo esc_url(home_url('/')); ?>" rel="home">
                                    <?php bloginfo('name'); ?>
                                </a>
                            </p>
                        <?php endif; ?>
                        
                        <?php $description = get_bloginfo('description', 'display'); ?>
                        <?php if ($description || is_customize_preview()): ?>
                            <p class="site-description"><?php echo $description; ?></p>
                        <?php endif; ?>
                    <?php endif; ?>
                </div>
                
                <nav id="site-navigation" class="main-navigation">
                    <?php
                    wp_nav_menu([
                        'theme_location' => 'primary',
                        'menu_id' => 'primary-menu',
                        'container_class' => 'menu-container',
                        'items_wrap' => '<ul id="%1$s" class="%2$s">%3$s</ul>',
                    ]);
                    ?>
                </nav>
                
                <div class="header-search">
                    <?php get_search_form(); ?>
                </div>
            </div>
        </header>
        
        <main id="main" class="site-main">
```

### footer.php
```php
        </main><!-- #main -->
        
        <?php if (is_active_sidebar('footer-1')): ?>
        <div class="footer-widgets">
            <div class="container">
                <?php dynamic_sidebar('footer-1'); ?>
            </div>
        </div>
        <?php endif; ?>
        
        <footer id="colophon" class="site-footer">
            <div class="container">
                <div class="footer-info">
                    <p>
                        &copy; <?php echo date('Y'); ?> 
                        <a href="<?php echo esc_url(home_url('/')); ?>">
                            <?php bloginfo('name'); ?>
                        </a>
                        <?php
                        printf(
                            /* translators: %s: WordPress link */
                            esc_html__('Proudly powered by %s', 'my-theme'),
                            '<a href="https://wordpress.org/">WordPress</a>'
                        );
                        ?>
                    </p>
                </div>
                
                <nav class="footer-navigation">
                    <?php
                    wp_nav_menu([
                        'theme_location' => 'footer',
                        'container' => false,
                        'depth' => 1,
                        'fallback_cb' => false,
                    ]);
                    ?>
                </nav>
            </div>
        </footer>
    </div><!-- #page -->
    
    <?php wp_footer(); ?>
</body>
</html>
```

### index.php (Main Template)
```php
<?php get_header(); ?>

<div class="container">
    <div class="content-area">
        <?php if (have_posts()): ?>
            <?php if (is_home() && !is_front_page()): ?>
                <header>
                    <h1 class="page-title screen-reader-text">
                        <?php single_post_title(); ?>
                    </h1>
                </header>
            <?php endif; ?>
            
            <div class="posts-grid">
                <?php while (have_posts()): the_post(); ?>
                    <article id="post-<?php the_ID(); ?>" <?php post_class('post-card'); ?>>
                        <?php if (has_post_thumbnail()): ?>
                            <div class="post-thumbnail">
                                <a href="<?php the_permalink(); ?>">
                                    <?php the_post_thumbnail('my-theme-medium', [
                                        'alt' => the_title_attribute(['echo' => false])
                                    ]); ?>
                                </a>
                            </div>
                        <?php endif; ?>
                        
                        <div class="post-content">
                            <header class="entry-header">
                                <?php the_title('<h2 class="entry-title"><a href="' . esc_url(get_permalink()) . '">', '</a></h2>'); ?>
                                
                                <div class="entry-meta">
                                    <span class="posted-on">
                                        <time datetime="<?php echo get_the_date(DATE_W3C); ?>">
                                            <?php echo get_the_date(); ?>
                                        </time>
                                    </span>
                                    <span class="byline">
                                        <?php the_author_posts_link(); ?>
                                    </span>
                                    <span class="cat-links">
                                        <?php the_category(', '); ?>
                                    </span>
                                </div>
                            </header>
                            
                            <div class="entry-summary">
                                <?php the_excerpt(); ?>
                            </div>
                            
                            <footer class="entry-footer">
                                <a href="<?php the_permalink(); ?>" class="read-more">
                                    <?php _e('Read More', 'my-theme'); ?> 
                                    <span class="sr-only">"<?php the_title(); ?>"</span>
                                </a>
                                <?php the_tags('<span class="tags">', ', ', '</span>'); ?>
                            </footer>
                        </div>
                    </article>
                <?php endwhile; ?>
            </div>
            
            <?php the_posts_pagination([
                'prev_text' => '&larr; ' . __('Previous', 'my-theme'),
                'next_text' => __('Next', 'my-theme') . ' &rarr;',
                'mid_size' => 2,
            ]); ?>
            
        <?php else: ?>
            <p><?php _e('No posts found.', 'my-theme'); ?></p>
        <?php endif; ?>
    </div>
    
    <?php if (is_active_sidebar('sidebar-1')): ?>
        <aside id="secondary" class="widget-area">
            <?php dynamic_sidebar('sidebar-1'); ?>
        </aside>
    <?php endif; ?>
</div>

<?php get_footer(); ?>
```

---

## ขั้นตอนที่ 635: WordPress Hooks - Actions & Filters

```php
<?php
// Actions - ทำบางอย่างตอน Event เกิดขึ้น
// add_action( $hook, $callback, $priority, $accepted_args )

// Basic Action
add_action('wp_head', function() {
    echo '<meta name="custom" content="value">';
});

// With Priority (default 10, lower = earlier)
add_action('wp_head', 'add_custom_meta', 5);
add_action('wp_head', 'add_analytics', 20);

// In Class
class MyPlugin {
    public function __construct() {
        add_action('init', [$this, 'init']);
        add_action('wp_enqueue_scripts', [$this, 'enqueue']);
    }
    
    public function init(): void {
        // Register post types, etc.
    }
    
    public function enqueue(): void {
        wp_enqueue_style('my-plugin', plugin_dir_url(__FILE__) . 'style.css');
    }
}
new MyPlugin();

// Common Action Hooks
add_action('init', 'register_stuff');               // WordPress initializes
add_action('wp_enqueue_scripts', 'load_assets');    // Enqueue scripts/styles
add_action('wp_head', 'add_to_head');               // Inside <head>
add_action('wp_footer', 'add_to_footer');           // Before </body>
add_action('save_post', 'do_on_save', 10, 3);       // Post saved
add_action('delete_post', 'do_on_delete');          // Post deleted
add_action('publish_post', 'do_on_publish', 10, 2); // Post published
add_action('user_register', 'do_on_register');      // User registered
add_action('wp_login', 'do_on_login', 10, 2);       // User logged in
add_action('comment_post', 'do_on_comment', 10, 3); // Comment posted
add_action('admin_init', 'admin_setup');             // Admin initialized
add_action('admin_menu', 'add_menu_pages');          // Admin menu

// ========================
// FILTERS
// ========================
// Filters - แก้ไขข้อมูลก่อน Output
// add_filter( $hook, $callback, $priority, $accepted_args )

// Modify the_content
add_filter('the_content', function($content) {
    if (is_single()) {
        $content .= '<div class="share-buttons">' . get_share_buttons() . '</div>';
    }
    return $content;
});

// Modify Excerpt Length
add_filter('excerpt_length', fn() => 20);

// Modify Excerpt More
add_filter('excerpt_more', function($more) {
    return '... <a href="' . get_permalink() . '">' . __('Read More', 'my-theme') . '</a>';
});

// Modify Post Title
add_filter('the_title', function($title, $id) {
    if (get_post_type($id) === 'portfolio') {
        return '📁 ' . $title;
    }
    return $title;
}, 10, 2);

// Email Sender
add_filter('wp_mail_from', fn() => 'noreply@mysite.com');
add_filter('wp_mail_from_name', fn() => 'My Site');

// Login Redirect
add_filter('login_redirect', function($redirect_to, $request, $user) {
    if (isset($user->roles) && in_array('administrator', $user->roles)) {
        return admin_url();
    }
    return $redirect_to;
}, 10, 3);

// Remove Filters/Actions
remove_filter('the_content', 'wpautop');
remove_action('wp_head', 'wp_generator');
```

---

## ขั้นตอนที่ 636: WordPress Plugin Development

```php
<?php
/**
 * Plugin Name: My Awesome Plugin
 * Plugin URI: https://example.com/my-plugin
 * Description: A plugin built for learning WordPress development
 * Version: 1.0.0
 * Requires at least: 6.0
 * Requires PHP: 8.0
 * Author: Your Name
 * Author URI: https://example.com
 * License: GPL v2 or later
 * License URI: https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain: my-awesome-plugin
 * Domain Path: /languages
 */

// Security: ป้องกันการเข้าถึงโดยตรง
if (!defined('ABSPATH')) {
    exit;
}

// Plugin Constants
define('MY_PLUGIN_VERSION', '1.0.0');
define('MY_PLUGIN_DIR', plugin_dir_path(__FILE__));
define('MY_PLUGIN_URL', plugin_dir_url(__FILE__));
define('MY_PLUGIN_BASENAME', plugin_basename(__FILE__));

// ========================
// Main Plugin Class
// ========================
class MyAwesomePlugin {
    private static ?self $instance = null;
    
    public static function getInstance(): self {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
    
    private function __construct() {
        $this->init();
    }
    
    private function init(): void {
        // Activation/Deactivation/Uninstall
        register_activation_hook(MY_PLUGIN_DIR . 'my-plugin.php', [$this, 'activate']);
        register_deactivation_hook(MY_PLUGIN_DIR . 'my-plugin.php', [$this, 'deactivate']);
        
        // Hooks
        add_action('plugins_loaded', [$this, 'loadTextdomain']);
        add_action('init', [$this, 'registerPostTypes']);
        add_action('admin_menu', [$this, 'addAdminMenu']);
        add_action('wp_enqueue_scripts', [$this, 'enqueueAssets']);
        add_action('admin_enqueue_scripts', [$this, 'enqueueAdminAssets']);
        
        // AJAX
        add_action('wp_ajax_my_plugin_action', [$this, 'handleAjax']);
        add_action('wp_ajax_nopriv_my_plugin_action', [$this, 'handleAjax']); // For non-logged-in
        
        // REST API
        add_action('rest_api_init', [$this, 'registerRestRoutes']);
        
        // Shortcodes
        add_shortcode('my_plugin', [$this, 'renderShortcode']);
        
        // Filters
        add_filter('the_content', [$this, 'modifyContent']);
    }
    
    public function activate(): void {
        // Create Database Tables
        $this->createTables();
        
        // Set Default Options
        add_option('my_plugin_version', MY_PLUGIN_VERSION);
        add_option('my_plugin_settings', [
            'api_key' => '',
            'enable_feature' => false,
            'items_per_page' => 10,
        ]);
        
        // Create Roles
        add_role('content_manager', 'Content Manager', [
            'read' => true,
            'edit_posts' => true,
            'publish_posts' => true,
        ]);
        
        // Flush Rewrite Rules
        flush_rewrite_rules();
    }
    
    public function deactivate(): void {
        flush_rewrite_rules();
    }
    
    private function createTables(): void {
        global $wpdb;
        
        $table = $wpdb->prefix . 'my_plugin_data';
        $charset = $wpdb->get_charset_collate();
        
        $sql = "CREATE TABLE IF NOT EXISTS $table (
            id mediumint(9) NOT NULL AUTO_INCREMENT,
            user_id bigint(20) UNSIGNED NOT NULL,
            action varchar(100) NOT NULL,
            data longtext NOT NULL,
            created_at datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
            PRIMARY KEY (id),
            KEY user_id (user_id)
        ) $charset;";
        
        require_once ABSPATH . 'wp-admin/includes/upgrade.php';
        dbDelta($sql);
    }
    
    public function loadTextdomain(): void {
        load_plugin_textdomain(
            'my-awesome-plugin',
            false,
            MY_PLUGIN_DIR . 'languages/'
        );
    }
    
    public function addAdminMenu(): void {
        add_menu_page(
            __('My Plugin', 'my-awesome-plugin'),
            __('My Plugin', 'my-awesome-plugin'),
            'manage_options',
            'my-plugin',
            [$this, 'renderAdminPage'],
            'dashicons-star-filled',
            30
        );
        
        add_submenu_page(
            'my-plugin',
            __('Settings', 'my-awesome-plugin'),
            __('Settings', 'my-awesome-plugin'),
            'manage_options',
            'my-plugin-settings',
            [$this, 'renderSettingsPage']
        );
    }
    
    public function renderAdminPage(): void {
        if (!current_user_can('manage_options')) {
            wp_die(__('Unauthorized', 'my-awesome-plugin'));
        }
        ?>
        <div class="wrap">
            <h1><?php echo esc_html(get_admin_page_title()); ?></h1>
            <p><?php _e('Welcome to My Plugin Dashboard', 'my-awesome-plugin'); ?></p>
        </div>
        <?php
    }
    
    public function handleAjax(): void {
        // Verify Nonce (Security!)
        if (!wp_verify_nonce($_POST['nonce'] ?? '', 'my_plugin_nonce')) {
            wp_die(__('Security check failed', 'my-awesome-plugin'), 403);
        }
        
        // Check Capability
        if (!current_user_can('edit_posts')) {
            wp_send_json_error(['message' => 'Unauthorized'], 403);
        }
        
        $action = sanitize_text_field($_POST['action_type'] ?? '');
        $data = $this->sanitizeData($_POST['data'] ?? []);
        
        $result = $this->processAction($action, $data);
        
        if ($result['success']) {
            wp_send_json_success($result['data']);
        } else {
            wp_send_json_error($result['message']);
        }
    }
    
    public function registerRestRoutes(): void {
        register_rest_route('my-plugin/v1', '/items', [
            'methods' => WP_REST_Server::READABLE,
            'callback' => [$this, 'getItems'],
            'permission_callback' => fn() => current_user_can('read'),
            'args' => [
                'page' => [
                    'default' => 1,
                    'type' => 'integer',
                    'sanitize_callback' => 'absint',
                ],
                'per_page' => [
                    'default' => 10,
                    'type' => 'integer',
                    'sanitize_callback' => 'absint',
                ],
            ],
        ]);
        
        register_rest_route('my-plugin/v1', '/items/(?P<id>\d+)', [
            [
                'methods' => WP_REST_Server::READABLE,
                'callback' => [$this, 'getItem'],
                'permission_callback' => fn() => current_user_can('read'),
            ],
            [
                'methods' => WP_REST_Server::EDITABLE,
                'callback' => [$this, 'updateItem'],
                'permission_callback' => fn() => current_user_can('edit_posts'),
            ],
            [
                'methods' => WP_REST_Server::DELETABLE,
                'callback' => [$this, 'deleteItem'],
                'permission_callback' => fn() => current_user_can('delete_posts'),
            ],
        ]);
    }
    
    public function renderShortcode(array $atts, ?string $content = null): string {
        $atts = shortcode_atts([
            'type' => 'default',
            'count' => 5,
            'columns' => 3,
        ], $atts, 'my_plugin');
        
        ob_start();
        ?>
        <div class="my-plugin-shortcode columns-<?php echo esc_attr($atts['columns']); ?>">
            <?php /* Output content here */ ?>
        </div>
        <?php
        return ob_get_clean();
    }
    
    private function sanitizeData(mixed $data): array {
        if (!is_array($data)) return [];
        return array_map(fn($v) => sanitize_text_field($v), $data);
    }
    
    private function processAction(string $action, array $data): array {
        return ['success' => true, 'data' => []];
    }
    
    public function enqueueAssets(): void {
        wp_enqueue_style('my-plugin', MY_PLUGIN_URL . 'assets/css/frontend.css', [], MY_PLUGIN_VERSION);
        wp_enqueue_script('my-plugin', MY_PLUGIN_URL . 'assets/js/frontend.js', ['jquery'], MY_PLUGIN_VERSION, true);
        
        wp_localize_script('my-plugin', 'myPlugin', [
            'ajaxUrl' => admin_url('admin-ajax.php'),
            'nonce' => wp_create_nonce('my_plugin_nonce'),
        ]);
    }
    
    public function enqueueAdminAssets(string $hookSuffix): void {
        if (!in_array($hookSuffix, ['toplevel_page_my-plugin', 'my-plugin_page_my-plugin-settings'])) {
            return;
        }
        
        wp_enqueue_style('my-plugin-admin', MY_PLUGIN_URL . 'assets/css/admin.css', [], MY_PLUGIN_VERSION);
        wp_enqueue_script('my-plugin-admin', MY_PLUGIN_URL . 'assets/js/admin.js', ['jquery'], MY_PLUGIN_VERSION, true);
    }
    
    public function modifyContent(string $content): string {
        if (!is_single() || !is_main_query()) return $content;
        
        $post_id = get_the_ID();
        $custom_data = get_post_meta($post_id, '_my_plugin_data', true);
        
        if ($custom_data) {
            $content = '<div class="my-plugin-top">' . esc_html($custom_data) . '</div>' . $content;
        }
        
        return $content;
    }
    
    public function registerPostTypes(): void { /* ... */ }
    public function renderSettingsPage(): void { /* ... */ }
    public function getItems(WP_REST_Request $request): WP_REST_Response { return new WP_REST_Response([]); }
    public function getItem(WP_REST_Request $request): WP_REST_Response { return new WP_REST_Response([]); }
    public function updateItem(WP_REST_Request $request): WP_REST_Response { return new WP_REST_Response([]); }
    public function deleteItem(WP_REST_Request $request): WP_REST_Response { return new WP_REST_Response([]); }
}

// Initialize Plugin
MyAwesomePlugin::getInstance();

// Uninstall Hook (ในไฟล์ uninstall.php)
// register_uninstall_hook(__FILE__, 'my_plugin_uninstall');
```

---

## 🎯 สรุป Part 26

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| WordPress Architecture | Files, Database, Templates |
| Theme Development | style.css, functions.php, templates |
| Theme Setup | add_theme_support, register_nav_menus |
| Enqueue Assets | wp_enqueue_script, wp_localize_script |
| Actions & Filters | add_action, add_filter, remove_action |
| Plugin Development | Plugin header, activation, AJAX, REST API |
| Shortcodes | shortcode_atts, ob_start |
| Security | wp_nonce, sanitize, current_user_can |

**ถัดไป → Part 27: WordPress Theme ขั้นสูง**
