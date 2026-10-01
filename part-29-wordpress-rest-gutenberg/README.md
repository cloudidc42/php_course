# Part 29: WordPress REST API & Gutenberg Blocks
## ขั้นตอนที่ 711-740: Modern WordPress Development

---

## ขั้นตอนที่ 711: WordPress REST API

```php
<?php
declare(strict_types=1);

namespace MyPlugin\Api;

class RestController {
    private const NAMESPACE = 'my-plugin/v1';
    
    public function register_routes(): void {
        // Reviews endpoint
        register_rest_route(self::NAMESPACE, '/reviews', [
            [
                'methods'             => \WP_REST_Server::READABLE,
                'callback'            => [$this, 'get_reviews'],
                'permission_callback' => '__return_true',
                'args'                => $this->get_collection_params(),
            ],
            [
                'methods'             => \WP_REST_Server::CREATABLE,
                'callback'            => [$this, 'create_review'],
                'permission_callback' => [$this, 'create_permissions'],
                'args'                => $this->get_create_params(),
            ],
        ]);
        
        register_rest_route(self::NAMESPACE, '/reviews/(?P<id>[\d]+)', [
            [
                'methods'             => \WP_REST_Server::READABLE,
                'callback'            => [$this, 'get_review'],
                'permission_callback' => '__return_true',
                'args'                => ['id' => ['validate_callback' => 'is_numeric']],
            ],
            [
                'methods'             => \WP_REST_Server::EDITABLE,
                'callback'            => [$this, 'update_review'],
                'permission_callback' => [$this, 'update_permissions'],
            ],
            [
                'methods'             => \WP_REST_Server::DELETABLE,
                'callback'            => [$this, 'delete_review'],
                'permission_callback' => [$this, 'delete_permissions'],
            ],
        ]);
        
        // Stats endpoint
        register_rest_route(self::NAMESPACE, '/stats/(?P<post_id>[\d]+)', [
            'methods'             => \WP_REST_Server::READABLE,
            'callback'            => [$this, 'get_stats'],
            'permission_callback' => '__return_true',
        ]);
    }
    
    public function get_reviews(\WP_REST_Request $request): \WP_REST_Response|\WP_Error {
        global $wpdb;
        $table = $wpdb->prefix . 'my_plugin_reviews';
        
        $page     = absint($request->get_param('page') ?? 1);
        $per_page = min(50, absint($request->get_param('per_page') ?? 10));
        $post_id  = absint($request->get_param('post_id') ?? 0);
        $offset   = ($page - 1) * $per_page;
        
        $where = "status = 'approved'";
        $values = [];
        
        if ($post_id > 0) {
            $where .= " AND post_id = %d";
            $values[] = $post_id;
        }
        
        $values[] = $per_page;
        $values[] = $offset;
        
        // phpcs:ignore WordPress.DB.PreparedSQLPlaceholders.UnfinishedPrepare
        $query = $wpdb->prepare(
            "SELECT * FROM {$table} WHERE {$where} ORDER BY created_at DESC LIMIT %d OFFSET %d",
            ...$values
        );
        
        $reviews = $wpdb->get_results($query);
        $total   = $wpdb->get_var("SELECT COUNT(*) FROM {$table} WHERE {$where}");
        
        $response = rest_ensure_response(
            array_map([$this, 'prepare_review_for_response'], $reviews)
        );
        
        $response->header('X-WP-Total', $total);
        $response->header('X-WP-TotalPages', ceil($total / $per_page));
        
        return $response;
    }
    
    public function create_review(\WP_REST_Request $request): \WP_REST_Response|\WP_Error {
        global $wpdb;
        
        $post_id = absint($request->get_param('post_id'));
        $post = get_post($post_id);
        
        if (!$post || $post->post_status !== 'publish') {
            return new \WP_Error('invalid_post', __('Invalid post.', 'my-plugin'), ['status' => 400]);
        }
        
        // Rate limit: 1 review per user per post
        $user_id = get_current_user_id();
        $existing = $wpdb->get_var($wpdb->prepare(
            "SELECT COUNT(*) FROM {$wpdb->prefix}my_plugin_reviews WHERE post_id = %d AND user_id = %d",
            $post_id, $user_id
        ));
        
        if ($existing > 0) {
            return new \WP_Error('already_reviewed', __('You have already reviewed this.', 'my-plugin'), ['status' => 409]);
        }
        
        $result = $wpdb->insert(
            $wpdb->prefix . 'my_plugin_reviews',
            [
                'post_id'    => $post_id,
                'user_id'    => $user_id,
                'rating'     => min(5, max(1, absint($request->get_param('rating')))),
                'comment'    => sanitize_textarea_field($request->get_param('comment')),
                'status'     => current_user_can('manage_options') ? 'approved' : 'pending',
                'ip_address' => sanitize_text_field($_SERVER['REMOTE_ADDR'] ?? ''),
            ],
            ['%d', '%d', '%d', '%s', '%s', '%s']
        );
        
        if (!$result) {
            return new \WP_Error('db_error', __('Could not save review.', 'my-plugin'), ['status' => 500]);
        }
        
        $review = $wpdb->get_row($wpdb->prepare(
            "SELECT * FROM {$wpdb->prefix}my_plugin_reviews WHERE id = %d",
            $wpdb->insert_id
        ));
        
        return rest_ensure_response(
            $this->prepare_review_for_response($review)
        );
    }
    
    public function create_permissions(): bool|\WP_Error {
        if (!is_user_logged_in()) {
            return new \WP_Error('rest_forbidden', __('You must be logged in.', 'my-plugin'), ['status' => 401]);
        }
        return true;
    }
    
    public function update_permissions(\WP_REST_Request $request): bool {
        global $wpdb;
        $review = $wpdb->get_row($wpdb->prepare(
            "SELECT * FROM {$wpdb->prefix}my_plugin_reviews WHERE id = %d",
            $request->get_param('id')
        ));
        
        return $review && (
            get_current_user_id() === (int) $review->user_id ||
            current_user_can('manage_options')
        );
    }
    
    public function delete_permissions(\WP_REST_Request $request): bool {
        return current_user_can('manage_options');
    }
    
    private function prepare_review_for_response(object $review): array {
        $user = get_userdata($review->user_id);
        
        return [
            'id'         => (int) $review->id,
            'post_id'    => (int) $review->post_id,
            'rating'     => (int) $review->rating,
            'comment'    => $review->comment,
            'status'     => $review->status,
            'author'     => [
                'id'     => (int) $review->user_id,
                'name'   => $user ? $user->display_name : __('Guest', 'my-plugin'),
                'avatar' => get_avatar_url($review->user_id, ['size' => 48]),
            ],
            'created_at' => $review->created_at,
            '_links'     => [
                'self' => [['href' => rest_url('my-plugin/v1/reviews/' . $review->id)]],
                'post' => [['href' => rest_url('wp/v2/posts/' . $review->post_id)]],
            ],
        ];
    }
    
    private function get_collection_params(): array {
        return [
            'page'     => ['type' => 'integer', 'default' => 1, 'minimum' => 1],
            'per_page' => ['type' => 'integer', 'default' => 10, 'minimum' => 1, 'maximum' => 50],
            'post_id'  => ['type' => 'integer', 'default' => 0],
        ];
    }
    
    private function get_create_params(): array {
        return [
            'post_id' => ['required' => true, 'type' => 'integer'],
            'rating'  => ['required' => true, 'type' => 'integer', 'minimum' => 1, 'maximum' => 5],
            'comment' => ['required' => true, 'type' => 'string', 'minLength' => 10, 'maxLength' => 1000],
        ];
    }
}
```

---

## ขั้นตอนที่ 712: Gutenberg Block (PHP Register)

```php
<?php
// includes/class-blocks.php

declare(strict_types=1);

namespace MyPlugin;

class Blocks {
    public function register_all(): void {
        // Method 1: PHP-rendered block
        register_block_type('my-plugin/reviews', [
            'api_version'     => 3,
            'title'           => __('Reviews', 'my-plugin'),
            'description'     => __('Show reviews for a post.', 'my-plugin'),
            'category'        => 'widgets',
            'icon'            => 'star-filled',
            'supports'        => ['align' => ['wide', 'full'], 'html' => false],
            'attributes'      => [
                'postId'   => ['type' => 'integer', 'default' => 0],
                'limit'    => ['type' => 'integer', 'default' => 5],
                'showForm' => ['type' => 'boolean', 'default' => true],
            ],
            'render_callback' => [$this, 'render_reviews_block'],
            'editor_script'   => 'my-plugin-editor',
            'editor_style'    => 'my-plugin-editor-style',
            'style'           => 'my-plugin-style',
        ]);
        
        // Method 2: block.json (recommended for complex blocks)
        register_block_type(MY_PLUGIN_DIR . 'blocks/rating-stars/');
        register_block_type(MY_PLUGIN_DIR . 'blocks/review-form/');
    }
    
    public function render_reviews_block(array $attributes): string {
        $post_id   = $attributes['postId'] ?: get_the_ID();
        $limit     = min(50, absint($attributes['limit']));
        $show_form = (bool) $attributes['showForm'];
        
        global $wpdb;
        $table = $wpdb->prefix . 'my_plugin_reviews';
        
        $reviews = $wpdb->get_results($wpdb->prepare(
            "SELECT * FROM {$table} WHERE post_id = %d AND status = 'approved' ORDER BY created_at DESC LIMIT %d",
            $post_id, $limit
        ));
        
        ob_start();
        ?>
        <div <?= get_block_wrapper_attributes(['class' => 'my-plugin-reviews-block']) ?>>
            <?php if (empty($reviews)): ?>
                <p class="no-reviews"><?= esc_html__('No reviews yet.', 'my-plugin') ?></p>
            <?php else: ?>
                <ul class="reviews-list">
                    <?php foreach ($reviews as $review): ?>
                        <li class="review-item">
                            <div class="review-rating">
                                <?= esc_html(str_repeat('★', $review->rating) . str_repeat('☆', 5 - $review->rating)) ?>
                            </div>
                            <p class="review-comment"><?= esc_html($review->comment) ?></p>
                        </li>
                    <?php endforeach; ?>
                </ul>
            <?php endif; ?>
            
            <?php if ($show_form && is_user_logged_in()): ?>
                <div class="review-form-wrapper" data-post-id="<?= esc_attr($post_id) ?>"></div>
            <?php endif; ?>
        </div>
        <?php
        return ob_get_clean();
    }
}
```

---

## ขั้นตอนที่ 713: block.json (JavaScript Block)

```json
// blocks/rating-stars/block.json
{
    "$schema": "https://schemas.wp.org/trunk/block.json",
    "apiVersion": 3,
    "name": "my-plugin/rating-stars",
    "version": "1.0.0",
    "title": "Rating Stars",
    "category": "widgets",
    "icon": "star-filled",
    "description": "Display star rating",
    "supports": {
        "html": false,
        "align": ["center", "left", "right"],
        "color": { "background": true, "text": true }
    },
    "attributes": {
        "rating": { "type": "number", "default": 0 },
        "maxRating": { "type": "number", "default": 5 },
        "size": { "type": "string", "default": "medium", "enum": ["small", "medium", "large"] },
        "color": { "type": "string", "default": "#FFD700" }
    },
    "textdomain": "my-plugin",
    "editorScript": "file:./index.js",
    "editorStyle": "file:./editor.css",
    "style": "file:./style.css",
    "render": "file:./render.php"
}
```

```php
<?php
// blocks/rating-stars/render.php

declare(strict_types=1);

$rating     = (float) ($attributes['rating'] ?? 0);
$max_rating = absint($attributes['maxRating'] ?? 5);
$size       = sanitize_key($attributes['size'] ?? 'medium');
$color      = sanitize_hex_color($attributes['color'] ?? '#FFD700');

$allowed_sizes = ['small', 'medium', 'large'];
if (!in_array($size, $allowed_sizes, true)) {
    $size = 'medium';
}
?>
<div <?= get_block_wrapper_attributes([
    'class' => "rating-stars rating-stars--{$size}",
    'style' => "--star-color: {$color}",
    'aria-label' => sprintf(__('%s out of %s stars', 'my-plugin'), $rating, $max_rating),
]) ?>>
    <?php for ($i = 1; $i <= $max_rating; $i++): ?>
        <span class="star <?= $i <= $rating ? 'star--filled' : ($i - $rating < 1 ? 'star--half' : 'star--empty') ?>">★</span>
    <?php endfor; ?>
    <span class="rating-value"><?= esc_html($rating . '/' . $max_rating) ?></span>
</div>
```

---

## ขั้นตอนที่ 714: Block JavaScript (React/JSX)

```javascript
// blocks/rating-stars/index.js

import { registerBlockType } from '@wordpress/blocks';
import { useBlockProps, InspectorControls, PanelColorSettings } from '@wordpress/block-editor';
import { PanelBody, RangeControl, SelectControl } from '@wordpress/components';
import { __ } from '@wordpress/i18n';
import metadata from './block.json';

registerBlockType(metadata.name, {
    edit: ({ attributes, setAttributes }) => {
        const { rating, maxRating, size, color } = attributes;
        const blockProps = useBlockProps({
            className: `rating-stars rating-stars--${size}`,
            style: { '--star-color': color },
        });
        
        return (
            <>
                <InspectorControls>
                    <PanelBody title={__('Rating Settings', 'my-plugin')}>
                        <RangeControl
                            label={__('Rating', 'my-plugin')}
                            value={rating}
                            onChange={(value) => setAttributes({ rating: value })}
                            min={0}
                            max={maxRating}
                            step={0.5}
                        />
                        <RangeControl
                            label={__('Max Rating', 'my-plugin')}
                            value={maxRating}
                            onChange={(value) => setAttributes({ maxRating: value })}
                            min={3}
                            max={10}
                        />
                        <SelectControl
                            label={__('Size', 'my-plugin')}
                            value={size}
                            options={[
                                { label: __('Small', 'my-plugin'), value: 'small' },
                                { label: __('Medium', 'my-plugin'), value: 'medium' },
                                { label: __('Large', 'my-plugin'), value: 'large' },
                            ]}
                            onChange={(value) => setAttributes({ size: value })}
                        />
                    </PanelBody>
                    <PanelColorSettings
                        title={__('Color Settings', 'my-plugin')}
                        colorSettings={[{
                            value: color,
                            onChange: (value) => setAttributes({ color: value }),
                            label: __('Star Color', 'my-plugin'),
                        }]}
                    />
                </InspectorControls>
                
                <div {...blockProps}>
                    {[...Array(maxRating)].map((_, i) => (
                        <span
                            key={i}
                            className={`star ${i < rating ? 'star--filled' : 'star--empty'}`}
                            onClick={() => setAttributes({ rating: i + 1 })}
                        >
                            ★
                        </span>
                    ))}
                    <span className="rating-value">{rating}/{maxRating}</span>
                </div>
            </>
        );
    },
    // No save function needed - uses render.php
    save: () => null,
});
```

---

## 🎯 สรุป Part 29

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| REST API | register_rest_route, callbacks, permissions |
| WP_REST_Request | params, nonce, error handling |
| REST Response | rest_ensure_response, headers |
| Gutenberg Block | register_block_type, render_callback |
| block.json | Declarative block config |
| React Blocks | edit function, InspectorControls |

**ถัดไป → Part 30: WooCommerce Development**
