# Part 27: WordPress Advanced Theme Development
## ขั้นตอนที่ 651-680: สร้าง Theme ระดับมืออาชีพ

---

## ขั้นตอนที่ 651: Theme Architecture

```
mytheme/
├── assets/
│   ├── css/
│   │   ├── main.css
│   │   └── editor-style.css
│   ├── js/
│   │   ├── main.js
│   │   └── navigation.js
│   └── images/
├── inc/
│   ├── theme-setup.php
│   ├── customizer.php
│   ├── widgets.php
│   ├── template-tags.php
│   └── block-editor.php
├── template-parts/
│   ├── header/
│   │   ├── site-header.php
│   │   └── site-navigation.php
│   ├── content/
│   │   ├── content.php
│   │   ├── content-single.php
│   │   └── content-page.php
│   └── footer/
│       └── site-footer.php
├── templates/          # Custom page templates
│   ├── full-width.php
│   └── landing.php
├── functions.php
├── style.css           # Theme header
├── index.php
├── header.php
├── footer.php
├── single.php
├── page.php
├── archive.php
├── search.php
└── 404.php
```

---

## ขั้นตอนที่ 652: functions.php โครงสร้างที่ดี

```php
<?php
declare(strict_types=1);

// Prevent direct access
if (!defined('ABSPATH')) exit;

// Load core files
require get_template_directory() . '/inc/theme-setup.php';
require get_template_directory() . '/inc/customizer.php';
require get_template_directory() . '/inc/widgets.php';
require get_template_directory() . '/inc/template-tags.php';
require get_template_directory() . '/inc/block-editor.php';

// Theme version constant
define('MYTHEME_VERSION', wp_get_theme()->get('Version'));

/**
 * Theme Setup
 */
add_action('after_setup_theme', function(): void {
    // Translation support
    load_theme_textdomain('mytheme', get_template_directory() . '/languages');
    
    // HTML5 support
    add_theme_support('html5', [
        'search-form', 'comment-form', 'comment-list',
        'gallery', 'caption', 'style', 'script',
    ]);
    
    // WordPress features
    add_theme_support('post-thumbnails');
    add_theme_support('title-tag');
    add_theme_support('automatic-feed-links');
    add_theme_support('customize-selective-refresh-widgets');
    add_theme_support('wp-block-styles');
    add_theme_support('editor-styles');
    add_theme_support('responsive-embeds');
    add_theme_support('align-wide');
    
    // Custom logo
    add_theme_support('custom-logo', [
        'height'      => 100,
        'width'       => 300,
        'flex-height' => true,
        'flex-width'  => true,
    ]);
    
    // Post formats
    add_theme_support('post-formats', ['gallery', 'video', 'quote', 'aside', 'link']);
    
    // Image sizes
    add_image_size('hero', 1920, 600, true);
    add_image_size('card', 400, 300, true);
    add_image_size('square', 300, 300, true);
    
    // Navigation menus
    register_nav_menus([
        'primary'   => esc_html__('Primary Menu', 'mytheme'),
        'footer'    => esc_html__('Footer Menu', 'mytheme'),
        'social'    => esc_html__('Social Links', 'mytheme'),
    ]);
});

/**
 * Enqueue Scripts & Styles
 */
add_action('wp_enqueue_scripts', function(): void {
    // Main stylesheet
    wp_enqueue_style(
        'mytheme-style',
        get_stylesheet_uri(),
        [],
        MYTHEME_VERSION
    );
    
    // Additional CSS
    wp_enqueue_style(
        'mytheme-main',
        get_template_directory_uri() . '/assets/css/main.css',
        ['mytheme-style'],
        MYTHEME_VERSION
    );
    
    // Google Fonts
    wp_enqueue_style(
        'mytheme-fonts',
        'https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;700&display=swap',
        [],
        null
    );
    
    // Scripts
    wp_enqueue_script(
        'mytheme-navigation',
        get_template_directory_uri() . '/assets/js/navigation.js',
        [],
        MYTHEME_VERSION,
        true  // In footer
    );
    
    wp_enqueue_script(
        'mytheme-main',
        get_template_directory_uri() . '/assets/js/main.js',
        ['jquery'],
        MYTHEME_VERSION,
        true
    );
    
    // Localize script (pass PHP data to JS)
    wp_localize_script('mytheme-main', 'mythemeData', [
        'ajaxUrl'    => admin_url('admin-ajax.php'),
        'nonce'      => wp_create_nonce('mytheme-nonce'),
        'homeUrl'    => home_url('/'),
        'isLoggedIn' => is_user_logged_in(),
        'i18n'       => [
            'loading'     => __('กำลังโหลด...', 'mytheme'),
            'readMore'    => __('อ่านเพิ่มเติม', 'mytheme'),
            'noResults'   => __('ไม่พบผลลัพธ์', 'mytheme'),
        ],
    ]);
    
    // Comments script only on singular posts
    if (is_singular() && comments_open() && get_option('thread_comments')) {
        wp_enqueue_script('comment-reply');
    }
});

/**
 * Admin Styles
 */
add_action('admin_enqueue_scripts', function(): void {
    wp_enqueue_style(
        'mytheme-admin',
        get_template_directory_uri() . '/assets/css/admin.css',
        [],
        MYTHEME_VERSION
    );
});

add_editor_style('assets/css/editor-style.css');
```

---

## ขั้นตอนที่ 653: WordPress Customizer API

```php
<?php
// inc/customizer.php

declare(strict_types=1);

add_action('customize_register', function(WP_Customize_Manager $wp_customize): void {
    
    // ==============================
    // SECTION: Site Identity (existing)
    // ==============================
    $wp_customize->get_setting('blogname')->transport = 'postMessage';
    $wp_customize->get_setting('blogdescription')->transport = 'postMessage';
    
    // ==============================
    // PANEL: Theme Options
    // ==============================
    $wp_customize->add_panel('mytheme_options', [
        'title'       => __('Theme Options', 'mytheme'),
        'priority'    => 130,
        'description' => __('ตั้งค่า Theme ของคุณ', 'mytheme'),
    ]);
    
    // ==============================
    // SECTION: Colors
    // ==============================
    $wp_customize->add_section('mytheme_colors', [
        'title'    => __('Colors', 'mytheme'),
        'panel'    => 'mytheme_options',
        'priority' => 10,
    ]);
    
    // Primary Color
    $wp_customize->add_setting('primary_color', [
        'default'           => '#3490dc',
        'sanitize_callback' => 'sanitize_hex_color',
        'transport'         => 'postMessage',
    ]);
    
    $wp_customize->add_control(new WP_Customize_Color_Control($wp_customize, 'primary_color', [
        'label'   => __('Primary Color', 'mytheme'),
        'section' => 'mytheme_colors',
    ]));
    
    // Secondary Color
    $wp_customize->add_setting('secondary_color', [
        'default'           => '#ffed4a',
        'sanitize_callback' => 'sanitize_hex_color',
        'transport'         => 'postMessage',
    ]);
    
    $wp_customize->add_control(new WP_Customize_Color_Control($wp_customize, 'secondary_color', [
        'label'   => __('Secondary Color', 'mytheme'),
        'section' => 'mytheme_colors',
    ]));
    
    // ==============================
    // SECTION: Typography
    // ==============================
    $wp_customize->add_section('mytheme_typography', [
        'title' => __('Typography', 'mytheme'),
        'panel' => 'mytheme_options',
    ]);
    
    $wp_customize->add_setting('body_font_size', [
        'default'           => '16',
        'sanitize_callback' => 'absint',
        'transport'         => 'postMessage',
    ]);
    
    $wp_customize->add_control('body_font_size', [
        'label'       => __('Body Font Size (px)', 'mytheme'),
        'section'     => 'mytheme_typography',
        'type'        => 'range',
        'input_attrs' => [
            'min'  => 12,
            'max'  => 24,
            'step' => 1,
        ],
    ]);
    
    // ==============================
    // SECTION: Hero Section
    // ==============================
    $wp_customize->add_section('mytheme_hero', [
        'title'    => __('Hero Section', 'mytheme'),
        'panel'    => 'mytheme_options',
    ]);
    
    $wp_customize->add_setting('hero_background', [
        'default'           => '',
        'sanitize_callback' => 'absint',
    ]);
    
    $wp_customize->add_control(new WP_Customize_Media_Control($wp_customize, 'hero_background', [
        'label'     => __('Hero Background Image', 'mytheme'),
        'section'   => 'mytheme_hero',
        'mime_type' => 'image',
    ]));
    
    $wp_customize->add_setting('hero_title', [
        'default'           => __('Welcome to My Site', 'mytheme'),
        'sanitize_callback' => 'sanitize_text_field',
        'transport'         => 'postMessage',
    ]);
    
    $wp_customize->add_control('hero_title', [
        'label'   => __('Hero Title', 'mytheme'),
        'section' => 'mytheme_hero',
        'type'    => 'text',
    ]);
    
    // ==============================
    // SECTION: Footer
    // ==============================
    $wp_customize->add_section('mytheme_footer', [
        'title' => __('Footer', 'mytheme'),
        'panel' => 'mytheme_options',
    ]);
    
    $wp_customize->add_setting('footer_text', [
        'default'           => '',
        'sanitize_callback' => 'wp_kses_post',
    ]);
    
    $wp_customize->add_control('footer_text', [
        'label'   => __('Footer Text', 'mytheme'),
        'section' => 'mytheme_footer',
        'type'    => 'textarea',
    ]);
    
    $wp_customize->add_setting('show_scroll_top', [
        'default'           => true,
        'sanitize_callback' => 'rest_sanitize_boolean',
    ]);
    
    $wp_customize->add_control('show_scroll_top', [
        'label'   => __('Show Scroll to Top Button', 'mytheme'),
        'section' => 'mytheme_footer',
        'type'    => 'checkbox',
    ]);
});

// Output customizer CSS
add_action('wp_head', function(): void {
    $primary    = get_theme_mod('primary_color', '#3490dc');
    $secondary  = get_theme_mod('secondary_color', '#ffed4a');
    $font_size  = get_theme_mod('body_font_size', '16');
    ?>
    <style id="mytheme-customizer-css">
        :root {
            --color-primary: <?= esc_attr($primary) ?>;
            --color-secondary: <?= esc_attr($secondary) ?>;
            --font-size-base: <?= absint($font_size) ?>px;
        }
    </style>
    <?php
});

// Live preview (postMessage)
add_action('customize_preview_init', function(): void {
    wp_enqueue_script(
        'mytheme-customizer-preview',
        get_template_directory_uri() . '/assets/js/customizer-preview.js',
        ['customize-preview'],
        MYTHEME_VERSION,
        true
    );
});
```

---

## ขั้นตอนที่ 654: Custom Post Types & Taxonomies

```php
<?php
// inc/custom-post-types.php

declare(strict_types=1);

/**
 * Register Portfolio CPT
 */
add_action('init', function(): void {
    register_post_type('portfolio', [
        'labels' => [
            'name'               => __('Portfolios', 'mytheme'),
            'singular_name'      => __('Portfolio', 'mytheme'),
            'add_new'            => __('Add New Portfolio', 'mytheme'),
            'add_new_item'       => __('Add New Portfolio', 'mytheme'),
            'edit_item'          => __('Edit Portfolio', 'mytheme'),
            'view_item'          => __('View Portfolio', 'mytheme'),
            'search_items'       => __('Search Portfolios', 'mytheme'),
            'not_found'          => __('No portfolios found', 'mytheme'),
            'menu_name'          => __('Portfolio', 'mytheme'),
        ],
        'public'             => true,
        'publicly_queryable' => true,
        'show_ui'            => true,
        'show_in_menu'       => true,
        'show_in_rest'       => true,   // Gutenberg + REST API support
        'rest_base'          => 'portfolio',
        'query_var'          => true,
        'rewrite'            => ['slug' => 'portfolio', 'with_front' => false],
        'capability_type'    => 'post',
        'has_archive'        => true,
        'hierarchical'       => false,
        'menu_position'      => 5,
        'menu_icon'          => 'dashicons-portfolio',
        'supports'           => [
            'title', 'editor', 'thumbnail',
            'excerpt', 'custom-fields', 'page-attributes',
        ],
        'taxonomies'         => ['portfolio_category', 'portfolio_tag'],
    ]);
    
    // Portfolio Category Taxonomy
    register_taxonomy('portfolio_category', 'portfolio', [
        'labels' => [
            'name'              => __('Portfolio Categories', 'mytheme'),
            'singular_name'     => __('Portfolio Category', 'mytheme'),
            'search_items'      => __('Search Categories', 'mytheme'),
            'all_items'         => __('All Categories', 'mytheme'),
            'edit_item'         => __('Edit Category', 'mytheme'),
            'add_new_item'      => __('Add New Category', 'mytheme'),
            'menu_name'         => __('Categories', 'mytheme'),
        ],
        'hierarchical'      => true,
        'show_ui'           => true,
        'show_in_rest'      => true,
        'show_admin_column' => true,
        'query_var'         => true,
        'rewrite'           => ['slug' => 'portfolio-category'],
    ]);
    
    // Portfolio Tag (non-hierarchical)
    register_taxonomy('portfolio_tag', 'portfolio', [
        'labels'        => [
            'name'          => __('Portfolio Tags', 'mytheme'),
            'singular_name' => __('Portfolio Tag', 'mytheme'),
            'add_new_item'  => __('Add New Tag', 'mytheme'),
        ],
        'hierarchical'  => false,
        'show_ui'       => true,
        'show_in_rest'  => true,
        'rewrite'       => ['slug' => 'portfolio-tag'],
    ]);
});

// Add meta box for portfolio details
add_action('add_meta_boxes', function(): void {
    add_meta_box(
        'portfolio_details',
        __('Portfolio Details', 'mytheme'),
        'render_portfolio_details_meta_box',
        'portfolio',
        'normal',
        'high'
    );
});

function render_portfolio_details_meta_box(WP_Post $post): void {
    wp_nonce_field('portfolio_details_nonce', 'portfolio_nonce');
    
    $client_name = get_post_meta($post->ID, '_client_name', true);
    $project_url = get_post_meta($post->ID, '_project_url', true);
    $completion_date = get_post_meta($post->ID, '_completion_date', true);
    $technologies = get_post_meta($post->ID, '_technologies', true);
    ?>
    <div class="mytheme-meta-box">
        <p>
            <label for="client_name"><?= esc_html__('Client Name', 'mytheme') ?></label>
            <input type="text" id="client_name" name="client_name"
                   value="<?= esc_attr($client_name) ?>" class="widefat">
        </p>
        <p>
            <label for="project_url"><?= esc_html__('Project URL', 'mytheme') ?></label>
            <input type="url" id="project_url" name="project_url"
                   value="<?= esc_url($project_url) ?>" class="widefat">
        </p>
        <p>
            <label for="completion_date"><?= esc_html__('Completion Date', 'mytheme') ?></label>
            <input type="date" id="completion_date" name="completion_date"
                   value="<?= esc_attr($completion_date) ?>">
        </p>
        <p>
            <label for="technologies"><?= esc_html__('Technologies (comma separated)', 'mytheme') ?></label>
            <input type="text" id="technologies" name="technologies"
                   value="<?= esc_attr($technologies) ?>" class="widefat">
        </p>
    </div>
    <?php
}

add_action('save_post_portfolio', function(int $post_id): void {
    if (!isset($_POST['portfolio_nonce']) ||
        !wp_verify_nonce($_POST['portfolio_nonce'], 'portfolio_details_nonce')) {
        return;
    }
    
    if (defined('DOING_AUTOSAVE') && DOING_AUTOSAVE) return;
    if (!current_user_can('edit_post', $post_id)) return;
    
    $fields = [
        '_client_name'      => 'sanitize_text_field',
        '_project_url'      => 'esc_url_raw',
        '_completion_date'  => 'sanitize_text_field',
        '_technologies'     => 'sanitize_text_field',
    ];
    
    foreach ($fields as $meta_key => $sanitizer) {
        $form_key = ltrim($meta_key, '_');
        if (isset($_POST[$form_key])) {
            $value = call_user_func($sanitizer, $_POST[$form_key]);
            update_post_meta($post_id, $meta_key, $value);
        }
    }
});
```

---

## ขั้นตอนที่ 655: Walker Classes (Custom Menus)

```php
<?php
// inc/class-walker-nav-menu.php

declare(strict_types=1);

class Mytheme_Walker_Nav_Menu extends Walker_Nav_Menu {
    private int $depth_count = 0;
    
    public function start_lvl(
        string &$output,
        int $depth = 0,
        stdClass $args = null
    ): void {
        $indent = str_repeat("\t", $depth);
        $output .= "\n{$indent}<ul class=\"sub-menu dropdown-menu\" role=\"menu\">\n";
    }
    
    public function end_lvl(
        string &$output,
        int $depth = 0,
        stdClass $args = null
    ): void {
        $indent = str_repeat("\t", $depth);
        $output .= "{$indent}</ul>\n";
    }
    
    public function start_el(
        string &$output,
        WP_Post $data_object,
        int $depth = 0,
        stdClass $args = null,
        int $current_object_id = 0
    ): void {
        $indent = str_repeat("\t", $depth);
        
        $classes = empty($data_object->classes) ? [] : (array) $data_object->classes;
        $classes[] = 'menu-item-' . $data_object->ID;
        
        // Add Tailwind classes
        if ($depth === 0) {
            $classes[] = 'nav-item';
        } else {
            $classes[] = 'dropdown-item';
        }
        
        // Check for children
        $has_children = in_array('menu-item-has-children', $classes);
        if ($has_children) {
            $classes[] = 'has-dropdown';
        }
        
        $class_names = join(' ', apply_filters('nav_menu_css_class', array_filter($classes), $data_object, $args, $depth));
        $class_names = $class_names ? ' class="' . esc_attr($class_names) . '"' : '';
        
        $id = apply_filters('nav_menu_item_id', 'menu-item-' . $data_object->ID, $data_object, $args, $depth);
        $id = $id ? ' id="' . esc_attr($id) . '"' : '';
        
        $output .= $indent . '<li' . $id . $class_names . '>';
        
        $atts = [];
        $atts['title']  = !empty($data_object->attr_title) ? $data_object->attr_title : '';
        $atts['target'] = !empty($data_object->target) ? $data_object->target : '';
        $atts['rel']    = !empty($data_object->xfn) ? $data_object->xfn : '';
        $atts['href']   = !empty($data_object->url) ? $data_object->url : '';
        $atts['class']  = $depth === 0 ? 'nav-link' : 'dropdown-link';
        
        if ($has_children) {
            $atts['aria-haspopup'] = 'true';
            $atts['aria-expanded'] = 'false';
        }
        
        $atts = apply_filters('nav_menu_link_attributes', $atts, $data_object, $args, $depth);
        
        $attributes = '';
        foreach ($atts as $attr => $value) {
            if (!empty($value)) {
                $value = ('href' === $attr) ? esc_url($value) : esc_attr($value);
                $attributes .= ' ' . $attr . '="' . $value . '"';
            }
        }
        
        $title = apply_filters('the_title', $data_object->title, $data_object->ID);
        $title = apply_filters('nav_menu_item_title', $title, $data_object, $args, $depth);
        
        $item_output = $args->before ?? '';
        $item_output .= '<a' . $attributes . '>';
        $item_output .= ($args->link_before ?? '') . $title . ($args->link_after ?? '');
        
        if ($has_children && $depth === 0) {
            $item_output .= ' <span class="dropdown-arrow" aria-hidden="true">▼</span>';
        }
        
        $item_output .= '</a>';
        $item_output .= $args->after ?? '';
        
        $output .= apply_filters('walker_nav_menu_start_el', $item_output, $data_object, $depth, $args);
    }
    
    public function end_el(
        string &$output,
        WP_Post $data_object,
        int $depth = 0,
        stdClass $args = null
    ): void {
        $output .= "</li>\n";
    }
}

// Usage
wp_nav_menu([
    'theme_location' => 'primary',
    'menu_class'     => 'nav-menu',
    'container'      => 'nav',
    'container_class'=> 'site-navigation',
    'walker'         => new Mytheme_Walker_Nav_Menu(),
    'depth'          => 3,
    'fallback_cb'    => false,
]);
```

---

## ขั้นตอนที่ 656: AJAX in Themes

```php
<?php
// inc/ajax.php - AJAX handlers

declare(strict_types=1);

/**
 * Load More Posts
 */
add_action('wp_ajax_load_more_posts', 'mytheme_load_more_posts');
add_action('wp_ajax_nopriv_load_more_posts', 'mytheme_load_more_posts');

function mytheme_load_more_posts(): void {
    // Verify nonce
    if (!wp_verify_nonce($_POST['nonce'] ?? '', 'mytheme-nonce')) {
        wp_send_json_error(['message' => 'Invalid nonce'], 403);
    }
    
    $paged     = max(1, absint($_POST['page'] ?? 1));
    $post_type = sanitize_key($_POST['post_type'] ?? 'post');
    $per_page  = min(20, absint($_POST['per_page'] ?? 6));
    $category  = absint($_POST['category'] ?? 0);
    
    // Whitelist post types
    $allowed_post_types = ['post', 'portfolio', 'product'];
    if (!in_array($post_type, $allowed_post_types, true)) {
        wp_send_json_error(['message' => 'Invalid post type'], 400);
    }
    
    $args = [
        'post_type'      => $post_type,
        'post_status'    => 'publish',
        'posts_per_page' => $per_page,
        'paged'          => $paged,
        'no_found_rows'  => false,
        'orderby'        => 'date',
        'order'          => 'DESC',
    ];
    
    if ($category > 0) {
        $args['cat'] = $category;
    }
    
    $query = new WP_Query($args);
    
    ob_start();
    
    if ($query->have_posts()) {
        while ($query->have_posts()) {
            $query->the_post();
            get_template_part('template-parts/content/content-card');
        }
        wp_reset_postdata();
    }
    
    $html = ob_get_clean();
    
    wp_send_json_success([
        'html'      => $html,
        'found'     => $query->found_posts,
        'max_pages' => $query->max_num_pages,
        'current'   => $paged,
        'has_more'  => $paged < $query->max_num_pages,
    ]);
}

/**
 * Search Autocomplete
 */
add_action('wp_ajax_search_autocomplete', 'mytheme_search_autocomplete');
add_action('wp_ajax_nopriv_search_autocomplete', 'mytheme_search_autocomplete');

function mytheme_search_autocomplete(): void {
    if (!wp_verify_nonce($_GET['nonce'] ?? '', 'mytheme-nonce')) {
        wp_send_json_error([], 403);
    }
    
    $term = sanitize_text_field($_GET['term'] ?? '');
    
    if (mb_strlen($term) < 2) {
        wp_send_json_success([]);
    }
    
    $results = get_posts([
        's'              => $term,
        'post_type'      => ['post', 'page', 'portfolio'],
        'post_status'    => 'publish',
        'posts_per_page' => 5,
        'no_found_rows'  => true,
    ]);
    
    $suggestions = array_map(fn(WP_Post $post): array => [
        'id'        => $post->ID,
        'title'     => get_the_title($post),
        'url'       => get_permalink($post),
        'type'      => $post->post_type,
        'thumbnail' => get_the_post_thumbnail_url($post, 'thumbnail'),
    ], $results);
    
    wp_send_json_success($suggestions);
}
```

---

## 🎯 สรุป Part 27

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Theme Architecture | โครงสร้างไฟล์มืออาชีพ |
| functions.php | add_theme_support, enqueue |
| Customizer API | Panels, Sections, Settings, Controls |
| Custom Post Types | CPT, Taxonomies, Meta Boxes |
| Walker Classes | Custom navigation markup |
| AJAX | Load more posts, autocomplete |

**ถัดไป → Part 28: WordPress Plugin Development**
