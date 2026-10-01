# Part 31: Drupal Basics
## ขั้นตอนที่ 771-800: เรียน Drupal ตั้งแต่ต้น

---

## ขั้นตอนที่ 771: Drupal Overview & Installation

```bash
# Drupal ต่างจาก WordPress อย่างไร?
# - Enterprise-grade CMS
# - Node-based content (everything is a node)
# - Entity system (content types, users, taxonomy terms, etc.)
# - Views for flexible content displays
# - Fields UI - drag-and-drop fields
# - Multilingual out of the box
# - RESTful services built-in

# System Requirements
# - PHP 8.1+
# - MySQL 5.7.8+ / PostgreSQL 10+
# - 256MB+ RAM (512MB recommended)

# Install via Composer
composer create-project drupal/recommended-project:^10 mysite
cd mysite

# Install Drush (Drupal CLI)
composer require drush/drush

# Install Drupal
./vendor/bin/drush site:install standard \
    --db-url=mysql://root:password@localhost/drupal \
    --site-name="My Drupal Site" \
    --account-name=admin \
    --account-pass=admin123 \
    --yes

# Start development server
php -S 0.0.0.0:8080 -t web/

# Common Drush commands
./vendor/bin/drush cache:rebuild     # Clear cache (cr)
./vendor/bin/drush config:export     # Export config (cex)
./vendor/bin/drush config:import     # Import config (cim)
./vendor/bin/drush updatedb          # Run database updates (updb)
./vendor/bin/drush entity:updates    # Run entity updates (entup)
./vendor/bin/drush user:login        # Get one-time login link (uli)
./vendor/bin/drush pm:enable views   # Enable module (en)
./vendor/bin/drush pm:disable views  # Disable module (pmd)
```

---

## ขั้นตอนที่ 772: Drupal Directory Structure

```
mysite/
├── composer.json
├── web/
│   ├── core/                    # Drupal core (never edit)
│   ├── modules/
│   │   ├── contrib/             # Contributed modules
│   │   └── custom/              # Your custom modules
│   ├── themes/
│   │   ├── contrib/             # Contributed themes
│   │   └── custom/              # Your custom themes
│   ├── profiles/                # Installation profiles
│   ├── sites/
│   │   ├── default/
│   │   │   ├── settings.php     # Database config
│   │   │   ├── settings.local.php
│   │   │   └── files/           # Uploaded files
│   │   └── all/
│   │       └── modules/         # Site-wide modules (older approach)
│   └── index.php
├── config/
│   └── sync/                    # Config YAML exports
└── vendor/                      # Composer dependencies
```

---

## ขั้นตอนที่ 773: Content Types & Fields

```php
<?php
// Programmatic content type creation (usually done via UI, but can be in code)

use Drupal\node\Entity\NodeType;
use Drupal\field\Entity\FieldStorageConfig;
use Drupal\field\Entity\FieldConfig;

// Create content type
$node_type = NodeType::create([
    'type'        => 'article',
    'name'        => 'Article',
    'description' => 'Use articles for time-sensitive content like news.',
    'help'        => '',
    'base'        => 'node_content',
    'display_submitted' => TRUE,
    'new_revision' => TRUE,
]);
$node_type->save();

// Add body field (standard)
node_add_body_field($node_type);

// Add custom field: category
FieldStorageConfig::create([
    'field_name'  => 'field_category',
    'entity_type' => 'node',
    'type'        => 'list_string',
    'settings'    => [
        'allowed_values' => [
            'tech'     => 'Technology',
            'business' => 'Business',
            'lifestyle'=> 'Lifestyle',
        ],
    ],
    'cardinality' => 1,
])->save();

FieldConfig::create([
    'field_name'   => 'field_category',
    'entity_type'  => 'node',
    'bundle'       => 'article',
    'label'        => 'Category',
    'required'     => TRUE,
])->save();

// Add image field
FieldStorageConfig::create([
    'field_name'  => 'field_image',
    'entity_type' => 'node',
    'type'        => 'image',
    'cardinality' => 1,
])->save();

FieldConfig::create([
    'field_name'  => 'field_image',
    'entity_type' => 'node',
    'bundle'      => 'article',
    'label'       => 'Featured Image',
    'settings'    => [
        'file_extensions' => 'png gif jpg jpeg webp',
        'max_filesize'    => '10 MB',
        'alt_field'       => TRUE,
    ],
])->save();
```

---

## ขั้นตอนที่ 774: Drupal Hooks System

```php
<?php
// mymodule/mymodule.module

declare(strict_types=1);

use Drupal\Core\Form\FormStateInterface;
use Drupal\node\NodeInterface;
use Drupal\Core\Routing\RouteMatchInterface;

/**
 * Implements hook_help().
 */
function mymodule_help(string $route_name, RouteMatchInterface $route_match): string {
    return match($route_name) {
        'help.page.mymodule' => '<p>' . t('My module provides custom functionality.') . '</p>',
        default => '',
    };
}

/**
 * Implements hook_node_presave().
 * Runs before a node is saved.
 */
function mymodule_node_presave(NodeInterface $node): void {
    // Auto-set summary from body
    if ($node->getType() === 'article' && $node->body->value && !$node->body->summary) {
        $node->body->summary = substr(strip_tags($node->body->value), 0, 200);
    }
}

/**
 * Implements hook_node_insert().
 * Runs after a new node is created.
 */
function mymodule_node_insert(NodeInterface $node): void {
    if ($node->getType() === 'article') {
        \Drupal::logger('mymodule')->info(
            'Article created: @title by @user',
            [
                '@title' => $node->getTitle(),
                '@user'  => \Drupal::currentUser()->getAccountName(),
            ]
        );
    }
}

/**
 * Implements hook_node_delete().
 */
function mymodule_node_delete(NodeInterface $node): void {
    // Clean up related data
    \Drupal::database()->delete('mymodule_node_stats')
        ->condition('nid', $node->id())
        ->execute();
}

/**
 * Implements hook_form_alter().
 */
function mymodule_form_alter(array &$form, FormStateInterface $form_state, string $form_id): void {
    if ($form_id === 'node_article_form' || $form_id === 'node_article_edit_form') {
        // Add custom validation
        $form['#validate'][] = 'mymodule_article_form_validate';
        
        // Add custom submit
        $form['actions']['submit']['#submit'][] = 'mymodule_article_form_submit';
        
        // Rearrange fields
        $form['field_category']['#weight'] = -5;
    }
}

function mymodule_article_form_validate(array $form, FormStateInterface $form_state): void {
    $title = $form_state->getValue('title');
    if (strlen($title[0]['value']) < 5) {
        $form_state->setErrorByName('title', t('Title must be at least 5 characters.'));
    }
}

function mymodule_article_form_submit(array $form, FormStateInterface $form_state): void {
    \Drupal::messenger()->addStatus(t('Article saved successfully!'));
}

/**
 * Implements hook_theme().
 */
function mymodule_theme(): array {
    return [
        'mymodule_article_card' => [
            'variables' => [
                'node'    => NULL,
                'teaser'  => FALSE,
                'classes' => [],
            ],
            'template' => 'mymodule-article-card',
            'path'     => \Drupal::service('extension.list.module')->getPath('mymodule') . '/templates',
        ],
    ];
}

/**
 * Implements hook_preprocess_node().
 */
function mymodule_preprocess_node(array &$variables): void {
    /** @var \Drupal\node\NodeInterface $node */
    $node = $variables['node'];
    
    if ($node->getType() === 'article') {
        $variables['reading_time'] = mymodule_calculate_reading_time($node->body->value);
        $variables['word_count']   = str_word_count(strip_tags($node->body->value));
    }
}

function mymodule_calculate_reading_time(string $text): int {
    $words = str_word_count(strip_tags($text));
    return (int) ceil($words / 200); // 200 words per minute
}
```

---

## ขั้นตอนที่ 775: Drupal Service Container

```php
<?php
// mymodule/src/Service/ArticleService.php

declare(strict_types=1);

namespace Drupal\mymodule\Service;

use Drupal\Core\Entity\EntityTypeManagerInterface;
use Drupal\Core\Session\AccountInterface;
use Drupal\Core\Logger\LoggerChannelFactoryInterface;
use Psr\Log\LoggerInterface;

class ArticleService {
    private LoggerInterface $logger;
    
    public function __construct(
        private EntityTypeManagerInterface $entityTypeManager,
        private AccountInterface $currentUser,
        LoggerChannelFactoryInterface $loggerFactory
    ) {
        $this->logger = $loggerFactory->get('mymodule');
    }
    
    public function getFeaturedArticles(int $limit = 5): array {
        $query = $this->entityTypeManager
            ->getStorage('node')
            ->getQuery()
            ->condition('type', 'article')
            ->condition('status', 1)
            ->condition('field_featured', 1)
            ->sort('created', 'DESC')
            ->range(0, $limit)
            ->accessCheck(TRUE);
        
        $nids = $query->execute();
        
        if (empty($nids)) return [];
        
        return $this->entityTypeManager
            ->getStorage('node')
            ->loadMultiple($nids);
    }
    
    public function createArticle(array $values): ?\Drupal\node\NodeInterface {
        try {
            /** @var \Drupal\node\NodeInterface $node */
            $node = $this->entityTypeManager->getStorage('node')->create([
                'type'           => 'article',
                'title'          => $values['title'],
                'body'           => [
                    'value'  => $values['body'],
                    'format' => 'full_html',
                ],
                'field_category' => $values['category'] ?? '',
                'uid'            => $this->currentUser->id(),
                'status'         => 1,
            ]);
            
            $node->save();
            
            $this->logger->info('Article created: @title', ['@title' => $values['title']]);
            
            return $node;
        } catch (\Exception $e) {
            $this->logger->error('Failed to create article: @message', ['@message' => $e->getMessage()]);
            return null;
        }
    }
}

// Register service in mymodule.services.yml
/*
services:
  mymodule.article_service:
    class: Drupal\mymodule\Service\ArticleService
    arguments:
      - '@entity_type.manager'
      - '@current_user'
      - '@logger.factory'
*/

// Usage in other services or controllers:
// $service = \Drupal::service('mymodule.article_service');
// OR via dependency injection in controller
```

---

## ขั้นตอนที่ 776: Drupal Routing & Controllers

```yaml
# mymodule/mymodule.routing.yml

mymodule.articles:
  path: '/articles'
  defaults:
    _controller: '\Drupal\mymodule\Controller\ArticleController::index'
    _title: 'Articles'
  requirements:
    _permission: 'access content'

mymodule.article.view:
  path: '/articles/{node}'
  defaults:
    _controller: '\Drupal\mymodule\Controller\ArticleController::view'
    _title_callback: '\Drupal\mymodule\Controller\ArticleController::title'
  requirements:
    _permission: 'access content'
    node: \d+

mymodule.admin.settings:
  path: '/admin/config/content/mymodule'
  defaults:
    _form: '\Drupal\mymodule\Form\SettingsForm'
    _title: 'MyModule Settings'
  requirements:
    _permission: 'administer mymodule'
  options:
    _admin_route: TRUE
```

```php
<?php
// mymodule/src/Controller/ArticleController.php

declare(strict_types=1);

namespace Drupal\mymodule\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\mymodule\Service\ArticleService;
use Drupal\node\NodeInterface;
use Symfony\Component\DependencyInjection\ContainerInterface;

class ArticleController extends ControllerBase {
    public function __construct(
        private ArticleService $articleService
    ) {}
    
    public static function create(ContainerInterface $container): static {
        return new static(
            $container->get('mymodule.article_service')
        );
    }
    
    public function index(): array {
        $articles = $this->articleService->getFeaturedArticles(10);
        
        return [
            '#theme' => 'mymodule_article_list',
            '#articles' => $articles,
            '#cache' => [
                'tags'    => ['node_list'],
                'max-age' => 3600,
            ],
        ];
    }
    
    public function view(NodeInterface $node): array {
        if ($node->getType() !== 'article') {
            throw new \Symfony\Component\HttpKernel\Exception\NotFoundHttpException();
        }
        
        $build = $this->entityTypeManager()
            ->getViewBuilder('node')
            ->view($node, 'full');
        
        return [
            '#theme'    => 'mymodule_article_view',
            '#node'     => $node,
            '#content'  => $build,
            '#cache'    => ['tags' => $node->getCacheTags()],
        ];
    }
    
    public function title(NodeInterface $node): string {
        return $node->getTitle();
    }
}
```

---

## 🎯 สรุป Part 31

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Installation | Composer, Drush, settings.php |
| Directory Structure | core, modules, themes, config |
| Content Types | NodeType, FieldStorage, FieldConfig |
| Hooks | hook_node_presave, hook_form_alter, hook_theme |
| Services | EntityTypeManager, dependency injection |
| Routing | YAML routes, ControllerBase |

**ถัดไป → Part 32: Drupal Custom Modules & Plugins**
