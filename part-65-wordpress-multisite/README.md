# Part 65: WordPress Multisite

## ขั้นตอนที่ 1811-1840: WordPress Multisite Network

WordPress Multisite ช่วยให้บริหารหลาย sites จาก installation เดียว

---

## ขั้นตอนที่ 1811: Setup Multisite Network

```php
<?php

declare(strict_types=1);

// wp-config.php - เพิ่มก่อน "That's all, stop editing!"
define('WP_ALLOW_MULTISITE', true);

// หลังจาก network setup ใน admin
define('MULTISITE', true);
define('SUBDOMAIN_INSTALL', true); // subdomain mode
define('DOMAIN_CURRENT_SITE', 'mynetwork.com');
define('PATH_CURRENT_SITE', '/');
define('SITE_ID_CURRENT_SITE', 1);
define('BLOG_ID_CURRENT_SITE', 1);
```

```nginx
# Nginx config สำหรับ Multisite (Subdomain)
server {
    listen 80;
    server_name mynetwork.com *.mynetwork.com;

    root /var/www/wordpress;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    # Uploaded files
    location ~* ^/files/$ {
        try_files $uri $uri/ /wp-includes/ms-files.php?file=$uri last;
    }

    location ~* ^/([_0-9a-zA-Z-]+/)?files/(.+) {
        try_files /wp-content/blogs.dir/$blogid/files/$2 /wp-includes/ms-files.php?file=$2 last;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
```

---

## ขั้นตอนที่ 1812: Network Admin Functions

```php
<?php

declare(strict_types=1);

// functions สำหรับ Network admin (ใช้ใน mu-plugins หรือ network plugin)

/**
 * สร้าง site ใหม่ใน network
 */
function create_network_site(string $domain, string $path, string $title, int $userId): int|WP_Error
{
    // ตรวจสอบว่า domain ถูกต้อง
    if (!is_subdomain_install() && !empty($domain)) {
        return new WP_Error('invalid_domain', 'Custom domain not supported in path mode');
    }

    $result = wpmu_create_blog($domain, $path, $title, $userId, [
        'public' => 1,
    ]);

    if (is_wp_error($result)) {
        return $result;
    }

    $blogId = $result;

    // Setup default options สำหรับ site ใหม่
    switch_to_blog($blogId);

    update_option('blogdescription', 'A new site in our network');
    update_option('posts_per_page', 10);
    update_option('time_format', 'H:i');
    update_option('date_format', 'd/m/Y');
    update_option('timezone_string', 'Asia/Bangkok');

    // ติดตั้ง plugins สำหรับ site นี้
    activate_plugin('woocommerce/woocommerce.php');

    restore_current_blog();

    return $blogId;
}

/**
 * ดึงสถิติของทุก sites ใน network
 */
function get_network_stats(): array
{
    global $wpdb;

    $sites = get_sites(['number' => 0]);
    $stats = [];

    foreach ($sites as $site) {
        switch_to_blog($site->blog_id);

        $stats[] = [
            'blog_id'    => $site->blog_id,
            'domain'     => $site->domain,
            'path'       => $site->path,
            'registered' => $site->registered,
            'post_count' => wp_count_posts()->publish,
            'user_count' => count(get_users(['blog_id' => $site->blog_id])),
            'last_post'  => get_posts(['numberposts' => 1, 'post_status' => 'publish'])[0]->post_date ?? null,
        ];

        restore_current_blog();
    }

    return $stats;
}
```

---

## ขั้นตอนที่ 1813: Must-Use Plugins (mu-plugins)

```php
<?php

declare(strict_types=1);

// wp-content/mu-plugins/network-setup.php
// Must-Use plugins โหลดอัตโนมัติในทุก sites

if (!defined('ABSPATH')) exit;

/**
 * กำหนด plugins ที่ต้อง active ในทุก site
 */
add_filter('option_active_plugins', function (array $plugins): array {
    $required = [
        'akismet/akismet.php',
        'wordpress-seo/wp-seo.php',
    ];

    return array_unique(array_merge($plugins, $required));
});

/**
 * Override site options based on network settings
 */
add_filter('pre_option_blogname', function (mixed $value): string {
    $blogId = get_current_blog_id();
    $networkOption = get_network_option(null, "site_{$blogId}_name");
    return $networkOption ?: $value;
});

/**
 * Limit upload size per site
 */
add_filter('upload_size_limit', function (int $size): int {
    $siteLimit = get_option('site_upload_limit', 64 * 1024 * 1024); // 64MB default
    return min($size, $siteLimit);
});

/**
 * Shared media ระหว่าง sites
 */
add_action('init', function (): void {
    if (get_current_blog_id() !== 1) {
        add_filter('wp_get_attachment_url', function (string $url, int $postId): string {
            // Check ถ้า attachment นี้อยู่ใน main site
            $mainSiteId   = get_main_site_id();
            $isMainSiteMedia = get_post_meta($postId, '_network_shared', true);

            if ($isMainSiteMedia) {
                switch_to_blog($mainSiteId);
                $mainUrl = wp_get_attachment_url($postId);
                restore_current_blog();
                return $mainUrl ?? $url;
            }

            return $url;
        }, 10, 2);
    }
});
```

---

## ขั้นตอนที่ 1814: Network-wide User Management

```php
<?php

declare(strict_types=1);

// ฟังก์ชันจัดการ users ทั่ว network

/**
 * เพิ่ม user ไปยังทุก sites ใน network
 */
function add_user_to_all_sites(int $userId, string $role = 'subscriber'): void
{
    $sites = get_sites(['number' => 0]);

    foreach ($sites as $site) {
        add_user_to_blog($site->blog_id, $userId, $role);
    }
}

/**
 * ดึง user's sites
 */
function get_user_sites(int $userId): array
{
    $blogs = get_blogs_of_user($userId);

    return array_map(function ($blog) use ($userId) {
        return [
            'blog_id'  => $blog->userblog_id,
            'domain'   => $blog->domain,
            'siteurl'  => $blog->siteurl,
            'blogname' => $blog->blogname,
            'role'     => get_user_role_on_blog($userId, $blog->userblog_id),
        ];
    }, $blogs);
}

function get_user_role_on_blog(int $userId, int $blogId): string
{
    switch_to_blog($blogId);
    $user  = new WP_User($userId);
    $roles = $user->roles;
    restore_current_blog();
    return $roles[0] ?? 'subscriber';
}

/**
 * Sync user profile ไปทุก sites
 */
function sync_user_across_network(int $userId): void
{
    $userMeta = get_user_meta($userId);
    $sites    = get_sites(['number' => 0]);

    foreach ($sites as $site) {
        if ($site->blog_id === 1) continue;

        switch_to_blog($site->blog_id);

        // Sync profile fields
        $user = get_user_by('id', $userId);
        if ($user) {
            wp_update_user([
                'ID'           => $userId,
                'display_name' => get_user_meta($userId, 'display_name', true),
            ]);
        }

        restore_current_blog();
    }
}
```

---

## ขั้นตอนที่ 1815: Custom Domain Mapping

```php
<?php

declare(strict_types=1);

// wp-content/mu-plugins/domain-mapping.php
// Custom domain mapping สำหรับ sites

add_action('ms_loaded', function (): void {
    if (is_admin()) return;

    $currentDomain = $_SERVER['HTTP_HOST'];
    $mappedBlogId  = get_blog_id_from_domain($currentDomain);

    if ($mappedBlogId && $mappedBlogId !== get_current_blog_id()) {
        switch_to_blog($mappedBlogId);
    }
});

function get_blog_id_from_domain(string $domain): ?int
{
    global $wpdb;

    $blogId = wp_cache_get("domain_map_{$domain}", 'domain_mapping');

    if ($blogId === false) {
        $blogId = $wpdb->get_var($wpdb->prepare(
            "SELECT blog_id FROM {$wpdb->site} WHERE domain = %s LIMIT 1",
            $domain
        ));

        if (!$blogId) {
            // Check custom domain table
            $blogId = $wpdb->get_var($wpdb->prepare(
                "SELECT blog_id FROM {$wpdb->base_prefix}domain_mapping WHERE domain = %s LIMIT 1",
                $domain
            ));
        }

        wp_cache_set("domain_map_{$domain}", $blogId ?? 0, 'domain_mapping', 3600);
    }

    return $blogId ?: null;
}

// AJAX endpoint สำหรับ add domain mapping
add_action('wp_ajax_add_domain_mapping', function (): void {
    check_ajax_referer('domain_mapping_nonce');

    if (!current_user_can('manage_network')) {
        wp_send_json_error('Insufficient permissions');
    }

    $domain = sanitize_text_field($_POST['domain'] ?? '');
    $blogId = (int) ($_POST['blog_id'] ?? 0);

    if (empty($domain) || !$blogId) {
        wp_send_json_error('Invalid data');
    }

    global $wpdb;
    $wpdb->replace("{$wpdb->base_prefix}domain_mapping", [
        'blog_id' => $blogId,
        'domain'  => $domain,
        'active'  => 1,
    ]);

    wp_cache_delete("domain_map_{$domain}", 'domain_mapping');

    wp_send_json_success(['message' => 'Domain mapped successfully']);
});
```

---

## ขั้นตอนที่ 1816: Multisite REST API

```php
<?php

declare(strict_types=1);

// เพิ่ม REST API endpoints สำหรับ network admin

add_action('rest_api_init', function (): void {

    // Get all sites
    register_rest_route('network/v1', '/sites', [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'api_get_network_sites',
        'permission_callback' => fn () => current_user_can('manage_network'),
        'args'                => [
            'per_page' => [
                'type'    => 'integer',
                'default' => 10,
                'minimum' => 1,
                'maximum' => 100,
            ],
            'page'     => [
                'type'    => 'integer',
                'default' => 1,
                'minimum' => 1,
            ],
        ],
    ]);

    // Create new site
    register_rest_route('network/v1', '/sites', [
        'methods'             => WP_REST_Server::CREATABLE,
        'callback'            => 'api_create_network_site',
        'permission_callback' => fn () => current_user_can('manage_network'),
        'args'                => [
            'subdomain' => [
                'required' => true,
                'type'     => 'string',
                'sanitize_callback' => 'sanitize_title',
            ],
            'title'     => [
                'required'          => true,
                'type'              => 'string',
                'sanitize_callback' => 'sanitize_text_field',
            ],
            'admin_email' => [
                'required'          => true,
                'type'              => 'string',
                'format'            => 'email',
                'sanitize_callback' => 'sanitize_email',
            ],
        ],
    ]);
});

function api_get_network_sites(WP_REST_Request $request): WP_REST_Response
{
    $perPage = $request->get_param('per_page');
    $page    = $request->get_param('page');

    $sites = get_sites([
        'number' => $perPage,
        'offset' => ($page - 1) * $perPage,
    ]);

    $total = get_sites(['count' => true]);

    $data = array_map(function ($site) {
        return [
            'id'         => $site->blog_id,
            'domain'     => $site->domain,
            'path'       => $site->path,
            'registered' => $site->registered,
            'last_updated' => $site->last_updated,
            'public'     => $site->public,
            'archived'   => $site->archived,
            'mature'     => $site->mature,
            'spam'       => $site->spam,
            'deleted'    => $site->deleted,
        ];
    }, $sites);

    $response = new WP_REST_Response($data, 200);
    $response->header('X-WP-Total', $total);
    $response->header('X-WP-TotalPages', ceil($total / $perPage));

    return $response;
}

function api_create_network_site(WP_REST_Request $request): WP_REST_Response|WP_Error
{
    $subdomain  = $request->get_param('subdomain');
    $title      = $request->get_param('title');
    $adminEmail = $request->get_param('admin_email');

    // Find or create admin user
    $adminUser = get_user_by('email', $adminEmail);
    if (!$adminUser) {
        $userId = wpmu_create_user($subdomain, wp_generate_password(), $adminEmail);
        if (is_wp_error($userId)) return $userId;
    } else {
        $userId = $adminUser->ID;
    }

    $domain = $subdomain . '.' . DOMAIN_CURRENT_SITE;
    $result = wpmu_create_blog($domain, '/', $title, $userId);

    if (is_wp_error($result)) {
        return $result;
    }

    return new WP_REST_Response([
        'id'     => $result,
        'domain' => $domain,
        'title'  => $title,
        'url'    => "https://{$domain}/",
    ], 201);
}
```

---

## ขั้นตอนที่ 1817: Network-wide Customizer Settings

```php
<?php

declare(strict_types=1);

// Network-wide settings ที่ override ได้ per-site

function get_network_setting(string $key, mixed $default = null): mixed
{
    $networkSettings = get_site_option('network_global_settings', []);
    $siteOverride    = get_option("site_setting_{$key}");

    // Site-specific override มีความสำคัญกว่า network default
    if ($siteOverride !== false) {
        return $siteOverride;
    }

    return $networkSettings[$key] ?? $default;
}

// ใช้งาน
$primaryColor = get_network_setting('primary_color', '#0066cc');
$logoUrl      = get_network_setting('logo_url', get_template_directory_uri() . '/images/logo.png');
$footerText   = get_network_setting('footer_text', '&copy; ' . date('Y') . ' Our Network');

add_action('customize_register', function (WP_Customize_Manager $wp_customize): void {
    $wp_customize->add_section('network_settings', [
        'title'    => 'Network Settings',
        'priority' => 200,
    ]);

    $wp_customize->add_setting('site_primary_color', [
        'default'           => get_network_setting('primary_color', '#0066cc'),
        'sanitize_callback' => 'sanitize_hex_color',
        'transport'         => 'postMessage',
    ]);

    $wp_customize->add_control(new WP_Customize_Color_Control($wp_customize, 'site_primary_color', [
        'label'   => 'Primary Color',
        'section' => 'network_settings',
    ]));
});
```

---

## สรุป Part 65

| Feature | รายละเอียด |
|---------|-----------|
| Network Setup | wp-config.php + database tables |
| MU Plugins | Auto-load สำหรับทุก sites |
| User Management | Add users to multiple sites |
| Domain Mapping | Custom domain per site |
| REST API | Network admin endpoints |
| Network Settings | Global + per-site override |
| Cross-site Data | switch_to_blog/restore_current_blog |

ถัดไป → Part 66: Drupal Advanced Development
