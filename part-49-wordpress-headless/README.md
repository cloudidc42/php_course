# Part 49: WordPress Headless

## ขั้นตอนที่ 1331-1360: WordPress เป็น Headless CMS

Headless WordPress ใช้ WordPress เป็นระบบจัดการเนื้อหา แต่แสดงผลด้วย Frontend framework อื่น เช่น Next.js

---

## ขั้นตอนที่ 1331: WP REST API พื้นฐาน

```php
<?php

declare(strict_types=1);

// functions.php - Enable CORS สำหรับ Headless

add_action('rest_api_init', function () {
    remove_filter('rest_pre_serve_request', 'rest_send_cors_headers');

    add_filter('rest_pre_serve_request', function ($value) {
        $origin = get_http_origin();

        $allowed_origins = [
            'https://myapp.vercel.app',
            'http://localhost:3000',
        ];

        if (in_array($origin, $allowed_origins)) {
            header('Access-Control-Allow-Origin: ' . $origin);
            header('Access-Control-Allow-Methods: GET, POST, OPTIONS');
            header('Access-Control-Allow-Headers: Authorization, Content-Type, X-WP-Nonce');
            header('Access-Control-Allow-Credentials: true');
        }

        return $value;
    });
});

// ตอบ OPTIONS request
add_action('init', function () {
    if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
        status_header(200);
        exit();
    }
});
```

```php
<?php

declare(strict_types=1);

// ลงทะเบียน Custom REST API Fields
add_action('rest_api_init', function () {
    // เพิ่ม featured_image_url ใน post response
    register_rest_field('post', 'featured_image_url', [
        'get_callback' => function (array $post): ?string {
            $imageId = get_post_thumbnail_id($post['id']);

            if (!$imageId) {
                return null;
            }

            $image = wp_get_attachment_image_src($imageId, 'full');
            return $image ? $image[0] : null;
        },
        'schema' => [
            'type'        => 'string',
            'description' => 'Full URL of featured image',
            'format'      => 'uri',
        ],
    ]);

    // เพิ่ม reading_time
    register_rest_field('post', 'reading_time', [
        'get_callback' => function (array $post): int {
            $wordCount = str_word_count(strip_tags($post['content']['rendered']));
            return (int)ceil($wordCount / 200); // 200 words per minute
        },
        'schema' => [
            'type'        => 'integer',
            'description' => 'Estimated reading time in minutes',
        ],
    ]);

    // เพิ่ม author details
    register_rest_field('post', 'author_details', [
        'get_callback' => function (array $post): array {
            $authorId = $post['author'];
            return [
                'id'     => $authorId,
                'name'   => get_the_author_meta('display_name', $authorId),
                'avatar' => get_avatar_url($authorId, ['size' => 96]),
                'bio'    => get_the_author_meta('description', $authorId),
            ];
        },
    ]);
});
```

---

## ขั้นตอนที่ 1332: Custom REST API Endpoints

```php
<?php

declare(strict_types=1);

class HeadlessApiController
{
    public function __construct()
    {
        add_action('rest_api_init', [$this, 'register_routes']);
    }

    public function register_routes(): void
    {
        $namespace = 'headless/v1';

        // Search posts
        register_rest_route($namespace, '/search', [
            'methods'             => 'GET',
            'callback'            => [$this, 'search'],
            'permission_callback' => '__return_true',
            'args'                => [
                'q' => [
                    'required'          => true,
                    'type'              => 'string',
                    'sanitize_callback' => 'sanitize_text_field',
                    'validate_callback' => fn ($v) => strlen($v) >= 2,
                ],
                'post_type' => [
                    'default'           => 'post',
                    'sanitize_callback' => 'sanitize_key',
                ],
            ],
        ]);

        // Get posts by category slug
        register_rest_route($namespace, '/categories/(?P<slug>[a-z0-9-]+)/posts', [
            'methods'             => 'GET',
            'callback'            => [$this, 'get_category_posts'],
            'permission_callback' => '__return_true',
            'args'                => [
                'slug'    => ['type' => 'string'],
                'page'    => ['default' => 1, 'type' => 'integer'],
                'per_page'=> ['default' => 10, 'type' => 'integer'],
            ],
        ]);

        // Navigation menus
        register_rest_route($namespace, '/menus/(?P<location>[a-z_-]+)', [
            'methods'             => 'GET',
            'callback'            => [$this, 'get_menu'],
            'permission_callback' => '__return_true',
        ]);

        // Site settings
        register_rest_route($namespace, '/settings', [
            'methods'             => 'GET',
            'callback'            => [$this, 'get_settings'],
            'permission_callback' => '__return_true',
        ]);
    }

    public function search(\WP_REST_Request $request): \WP_REST_Response
    {
        $keyword   = $request->get_param('q');
        $post_type = $request->get_param('post_type');

        $posts = get_posts([
            's'           => $keyword,
            'post_type'   => $post_type,
            'numberposts' => 10,
            'post_status' => 'publish',
        ]);

        $results = array_map(function (\WP_Post $post) {
            return [
                'id'      => $post->ID,
                'title'   => get_the_title($post),
                'excerpt' => get_the_excerpt($post),
                'slug'    => $post->post_name,
                'url'     => get_permalink($post),
                'date'    => get_the_date('c', $post),
                'image'   => get_the_post_thumbnail_url($post, 'medium'),
            ];
        }, $posts);

        return new \WP_REST_Response($results);
    }

    public function get_category_posts(\WP_REST_Request $request): \WP_REST_Response
    {
        $slug     = $request->get_param('slug');
        $page     = $request->get_param('page');
        $perPage  = $request->get_param('per_page');

        $category = get_term_by('slug', $slug, 'category');

        if (!$category) {
            return new \WP_REST_Response(['error' => 'Category not found'], 404);
        }

        $query = new \WP_Query([
            'cat'            => $category->term_id,
            'posts_per_page' => $perPage,
            'paged'          => $page,
            'post_status'    => 'publish',
        ]);

        $posts = array_map(function (\WP_Post $post) {
            return $this->format_post($post);
        }, $query->posts);

        return new \WP_REST_Response([
            'posts'       => $posts,
            'total'       => $query->found_posts,
            'total_pages' => $query->max_num_pages,
            'category'    => [
                'id'          => $category->term_id,
                'name'        => $category->name,
                'description' => $category->description,
            ],
        ]);
    }

    public function get_menu(\WP_REST_Request $request): \WP_REST_Response
    {
        $location = $request->get_param('location');
        $locations = get_nav_menu_locations();

        if (!isset($locations[$location])) {
            return new \WP_REST_Response(['error' => 'Menu not found'], 404);
        }

        $menu  = wp_get_nav_menu_object($locations[$location]);
        $items = wp_get_nav_menu_items($menu->term_id);

        return new \WP_REST_Response([
            'name'  => $menu->name,
            'items' => $this->build_menu_tree($items),
        ]);
    }

    public function get_settings(\WP_REST_Request $request): \WP_REST_Response
    {
        return new \WP_REST_Response([
            'name'           => get_bloginfo('name'),
            'description'    => get_bloginfo('description'),
            'url'            => get_bloginfo('url'),
            'language'       => get_bloginfo('language'),
            'logo'           => wp_get_attachment_image_url(get_theme_mod('custom_logo'), 'full'),
            'social_links'   => [
                'facebook'   => get_option('social_facebook'),
                'twitter'    => get_option('social_twitter'),
                'instagram'  => get_option('social_instagram'),
            ],
        ]);
    }

    private function format_post(\WP_Post $post): array
    {
        return [
            'id'            => $post->ID,
            'slug'          => $post->post_name,
            'title'         => get_the_title($post),
            'content'       => apply_filters('the_content', $post->post_content),
            'excerpt'       => get_the_excerpt($post),
            'date'          => get_the_date('c', $post),
            'modified'      => get_the_modified_date('c', $post),
            'image'         => get_the_post_thumbnail_url($post, 'full'),
            'categories'    => wp_get_post_categories($post->ID, ['fields' => 'names']),
            'tags'          => wp_get_post_tags($post->ID, ['fields' => 'names']),
            'author'        => get_the_author_meta('display_name', $post->post_author),
        ];
    }

    private function build_menu_tree(array $items, int $parentId = 0): array
    {
        $tree = [];

        foreach ($items as $item) {
            if ((int)$item->menu_item_parent === $parentId) {
                $children = $this->build_menu_tree($items, $item->ID);
                $tree[] = [
                    'id'       => $item->ID,
                    'title'    => $item->title,
                    'url'      => $item->url,
                    'target'   => $item->target,
                    'children' => $children,
                ];
            }
        }

        return $tree;
    }
}

new HeadlessApiController();
```

---

## ขั้นตอนที่ 1333: Authentication สำหรับ Headless

```php
<?php

declare(strict_types=1);

// JWT Authentication สำหรับ WordPress Headless
// ใช้ plugin: wp-api-jwt-auth

add_filter('jwt_auth_token_before_dispatch', function (array $data, \WP_User $user): array {
    // เพิ่มข้อมูลใน JWT token
    $data['user_display_name'] = $user->display_name;
    $data['user_email']        = $user->user_email;
    $data['user_roles']        = $user->roles;
    $data['avatar']            = get_avatar_url($user->ID);

    return $data;
}, 10, 2);

// Custom login endpoint
add_action('rest_api_init', function () {
    register_rest_route('auth/v1', '/login', [
        'methods'             => 'POST',
        'callback'            => 'headless_login',
        'permission_callback' => '__return_true',
        'args'                => [
            'username' => ['required' => true],
            'password' => ['required' => true],
        ],
    ]);

    register_rest_route('auth/v1', '/me', [
        'methods'             => 'GET',
        'callback'            => 'headless_get_current_user',
        'permission_callback' => 'is_user_logged_in',
    ]);
});

function headless_login(\WP_REST_Request $request): \WP_REST_Response
{
    $username = sanitize_user($request->get_param('username'));
    $password = $request->get_param('password');

    $user = wp_authenticate($username, $password);

    if (is_wp_error($user)) {
        return new \WP_REST_Response([
            'error'   => 'invalid_credentials',
            'message' => 'Invalid username or password',
        ], 401);
    }

    // Generate JWT
    $token = generate_jwt_token($user);

    return new \WP_REST_Response([
        'token' => $token,
        'user'  => [
            'id'          => $user->ID,
            'name'        => $user->display_name,
            'email'       => $user->user_email,
            'roles'       => $user->roles,
            'avatar'      => get_avatar_url($user->ID),
        ],
    ]);
}
```

---

## ขั้นตอนที่ 1334: ACF (Advanced Custom Fields) API

```php
<?php

declare(strict_types=1);

// ส่ง ACF data ใน REST API response
add_filter('rest_prepare_post', function ($response, $post, $request) {
    if (function_exists('get_fields')) {
        $fields = get_fields($post->ID);

        if ($fields) {
            $response->data['acf'] = $fields;
        }
    }

    return $response;
}, 10, 3);

// Custom post type กับ ACF
add_action('init', function () {
    register_post_type('property', [
        'public'       => true,
        'show_in_rest' => true,  // สำคัญ: ต้องเปิด REST API
        'label'        => 'Properties',
        'supports'     => ['title', 'editor', 'thumbnail', 'custom-fields'],
        'rewrite'      => ['slug' => 'properties'],
    ]);
});

// Expose ACF fields ใน REST
add_action('rest_api_init', function () {
    register_rest_field('property', 'property_details', [
        'get_callback' => function ($post_arr) {
            $id = $post_arr['id'];

            return [
                'price'        => get_field('price', $id),
                'bedrooms'     => get_field('bedrooms', $id),
                'bathrooms'    => get_field('bathrooms', $id),
                'area_sqm'     => get_field('area_sqm', $id),
                'address'      => get_field('address', $id),
                'latitude'     => get_field('latitude', $id),
                'longitude'    => get_field('longitude', $id),
                'features'     => get_field('features', $id),     // checkbox field
                'gallery'      => array_map(function ($img) {
                    return [
                        'id'  => $img['ID'],
                        'url' => $img['url'],
                        'alt' => $img['alt'],
                    ];
                }, get_field('gallery', $id) ?: []),
            ];
        },
    ]);
});
```

---

## ขั้นตอนที่ 1335: Custom Endpoints สำหรับ SPA

```php
<?php

declare(strict_types=1);

// Endpoint สำหรับ page by slug (ใช้กับ Next.js getStaticPaths)
add_action('rest_api_init', function () {
    register_rest_route('headless/v1', '/page-by-slug/(?P<slug>[a-z0-9-]+)', [
        'methods'             => 'GET',
        'callback'            => function (\WP_REST_Request $request) {
            $slug = $request->get_param('slug');

            $page = get_page_by_path($slug, OBJECT, 'page');

            if (!$page) {
                return new \WP_REST_Response(['error' => 'Page not found'], 404);
            }

            return new \WP_REST_Response([
                'id'       => $page->ID,
                'slug'     => $page->post_name,
                'title'    => get_the_title($page),
                'content'  => apply_filters('the_content', $page->post_content),
                'meta'     => [
                    'title'       => get_post_meta($page->ID, '_yoast_wpseo_title', true),
                    'description' => get_post_meta($page->ID, '_yoast_wpseo_metadesc', true),
                    'og_image'    => get_the_post_thumbnail_url($page, 'full'),
                ],
                'acf'      => function_exists('get_fields') ? get_fields($page->ID) : [],
            ]);
        },
        'permission_callback' => '__return_true',
    ]);

    // All page slugs (สำหรับ getStaticPaths)
    register_rest_route('headless/v1', '/page-slugs', [
        'methods'             => 'GET',
        'callback'            => function () {
            $pages = get_pages(['post_status' => 'publish']);

            return new \WP_REST_Response(
                array_map(fn ($p) => ['slug' => $p->post_name], $pages)
            );
        },
        'permission_callback' => '__return_true',
    ]);

    // All post slugs with pagination info
    register_rest_route('headless/v1', '/post-slugs', [
        'methods'             => 'GET',
        'callback'            => function () {
            $posts = get_posts([
                'post_type'   => 'post',
                'numberposts' => -1,
                'post_status' => 'publish',
                'fields'      => 'ids',
            ]);

            return new \WP_REST_Response(
                array_map(fn ($id) => [
                    'slug' => get_post_field('post_name', $id),
                ], $posts)
            );
        },
        'permission_callback' => '__return_true',
    ]);
});
```

---

## สรุปบทที่ 49

| หัวข้อ | เทคนิค | ประโยชน์ |
|--------|--------|---------|
| WP REST API | register_rest_field | Extend existing endpoints |
| Custom Endpoints | register_rest_route | New API routes |
| CORS | rest_pre_serve_request | Allow frontend domains |
| JWT Auth | wp-api-jwt-auth plugin | Stateless authentication |
| ACF Integration | get_fields + REST | Rich content data |
| Headless Setup | Custom endpoints | Next.js/Nuxt integration |

**ต่อไป**: Part 50 - Monitoring & Logging

---
