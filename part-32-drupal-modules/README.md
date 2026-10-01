# Part 32: Drupal Custom Module Development
## ขั้นตอนที่ 801-830: สร้าง Module ระดับมืออาชีพ

---

## ขั้นตอนที่ 801: Module Structure

```
web/modules/custom/mymodule/
├── config/
│   ├── install/          # Default config when module is installed
│   │   └── mymodule.settings.yml
│   └── schema/           # Config schema for validation
│       └── mymodule.schema.yml
├── src/
│   ├── Controller/
│   ├── Form/
│   ├── Plugin/
│   │   ├── Block/
│   │   └── Field/
│   ├── Service/
│   ├── Entity/
│   └── EventSubscriber/
├── templates/
│   └── mymodule-article-card.html.twig
├── tests/
│   ├── src/
│   │   ├── Unit/
│   │   └── Functional/
├── mymodule.info.yml       # Module metadata
├── mymodule.module         # Hook implementations
├── mymodule.routing.yml    # Routes
├── mymodule.services.yml   # Service definitions
├── mymodule.permissions.yml # Custom permissions
├── mymodule.links.menu.yml # Menu links
├── mymodule.links.task.yml # Task links (tabs)
└── mymodule.install        # Install/update hooks
```

---

## ขั้นตอนที่ 802: Module Info & Config

```yaml
# mymodule.info.yml

name: 'My Module'
type: module
description: 'A professional Drupal 10 module.'
package: Custom
core_version_requirement: ^10
dependencies:
  - drupal:node
  - drupal:user
  - drupal:views
configure: mymodule.admin.settings
```

```yaml
# mymodule.permissions.yml

administer mymodule:
  title: 'Administer MyModule settings'
  description: 'Configure MyModule settings.'
  restrict access: true

access mymodule content:
  title: 'Access MyModule content'

create mymodule content:
  title: 'Create MyModule content'
```

```yaml
# config/install/mymodule.settings.yml

items_per_page: 10
show_author: true
date_format: 'medium'
featured_categories: []
cache_lifetime: 3600

# config/schema/mymodule.schema.yml

mymodule.settings:
  type: config_object
  label: 'MyModule settings'
  mapping:
    items_per_page:
      type: integer
      label: 'Items per page'
    show_author:
      type: boolean
      label: 'Show author'
    date_format:
      type: string
      label: 'Date format'
    featured_categories:
      type: sequence
      label: 'Featured categories'
      sequence:
        type: string
    cache_lifetime:
      type: integer
      label: 'Cache lifetime in seconds'
```

---

## ขั้นตอนที่ 803: Config Form

```php
<?php
// src/Form/SettingsForm.php

declare(strict_types=1);

namespace Drupal\mymodule\Form;

use Drupal\Core\Form\ConfigFormBase;
use Drupal\Core\Form\FormStateInterface;

class SettingsForm extends ConfigFormBase {
    public function getFormId(): string {
        return 'mymodule_settings_form';
    }
    
    protected function getEditableConfigNames(): array {
        return ['mymodule.settings'];
    }
    
    public function buildForm(array $form, FormStateInterface $form_state): array {
        $config = $this->config('mymodule.settings');
        
        $form['display'] = [
            '#type'  => 'details',
            '#title' => $this->t('Display Settings'),
            '#open'  => TRUE,
        ];
        
        $form['display']['items_per_page'] = [
            '#type'          => 'number',
            '#title'         => $this->t('Items per page'),
            '#default_value' => $config->get('items_per_page'),
            '#min'           => 1,
            '#max'           => 100,
            '#required'      => TRUE,
        ];
        
        $form['display']['show_author'] = [
            '#type'          => 'checkbox',
            '#title'         => $this->t('Show author information'),
            '#default_value' => $config->get('show_author'),
        ];
        
        $form['display']['date_format'] = [
            '#type'          => 'select',
            '#title'         => $this->t('Date Format'),
            '#options'       => [
                'short'  => $this->t('Short'),
                'medium' => $this->t('Medium'),
                'long'   => $this->t('Long'),
                'custom' => $this->t('Custom'),
            ],
            '#default_value' => $config->get('date_format'),
        ];
        
        $form['performance'] = [
            '#type'  => 'details',
            '#title' => $this->t('Performance'),
        ];
        
        $form['performance']['cache_lifetime'] = [
            '#type'          => 'number',
            '#title'         => $this->t('Cache lifetime (seconds)'),
            '#description'   => $this->t('0 to disable caching.'),
            '#default_value' => $config->get('cache_lifetime'),
            '#min'           => 0,
        ];
        
        return parent::buildForm($form, $form_state);
    }
    
    public function validateForm(array &$form, FormStateInterface $form_state): void {
        $items = $form_state->getValue('items_per_page');
        if ($items < 1 || $items > 100) {
            $form_state->setErrorByName('items_per_page', $this->t('Items per page must be between 1 and 100.'));
        }
    }
    
    public function submitForm(array &$form, FormStateInterface $form_state): void {
        $this->config('mymodule.settings')
            ->set('items_per_page', (int) $form_state->getValue('items_per_page'))
            ->set('show_author', (bool) $form_state->getValue('show_author'))
            ->set('date_format', $form_state->getValue('date_format'))
            ->set('cache_lifetime', (int) $form_state->getValue('cache_lifetime'))
            ->save();
        
        parent::submitForm($form, $form_state);
        
        // Invalidate cache when settings change
        \Drupal::service('cache_tags.invalidator')->invalidateTags(['mymodule_config']);
    }
}
```

---

## ขั้นตอนที่ 804: Custom Block Plugin

```php
<?php
// src/Plugin/Block/FeaturedArticlesBlock.php

declare(strict_types=1);

namespace Drupal\mymodule\Plugin\Block;

use Drupal\Core\Block\BlockBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Plugin\ContainerFactoryPluginInterface;
use Drupal\mymodule\Service\ArticleService;
use Symfony\Component\DependencyInjection\ContainerInterface;

/**
 * @Block(
 *   id = "mymodule_featured_articles",
 *   admin_label = @Translation("Featured Articles"),
 *   category = @Translation("MyModule"),
 * )
 */
class FeaturedArticlesBlock extends BlockBase implements ContainerFactoryPluginInterface {
    public function __construct(
        array $configuration,
        string $plugin_id,
        mixed $plugin_definition,
        private ArticleService $articleService
    ) {
        parent::__construct($configuration, $plugin_id, $plugin_definition);
    }
    
    public static function create(
        ContainerInterface $container,
        array $configuration,
        string $plugin_id,
        mixed $plugin_definition
    ): static {
        return new static(
            $configuration,
            $plugin_id,
            $plugin_definition,
            $container->get('mymodule.article_service')
        );
    }
    
    public function defaultConfiguration(): array {
        return [
            'limit' => 3,
            'category' => '',
        ];
    }
    
    public function blockForm(array $form, FormStateInterface $form_state): array {
        $form = parent::blockForm($form, $form_state);
        $config = $this->configuration;
        
        $form['limit'] = [
            '#type'          => 'number',
            '#title'         => $this->t('Number of articles'),
            '#default_value' => $config['limit'],
            '#min'           => 1,
            '#max'           => 20,
        ];
        
        return $form;
    }
    
    public function blockSubmit(array $form, FormStateInterface $form_state): void {
        $this->configuration['limit'] = (int) $form_state->getValue('limit');
    }
    
    public function build(): array {
        $limit    = (int) $this->configuration['limit'];
        $articles = $this->articleService->getFeaturedArticles($limit);
        
        if (empty($articles)) {
            return [
                '#markup' => $this->t('No featured articles found.'),
                '#cache'  => ['tags' => ['node_list']],
            ];
        }
        
        $items = [];
        foreach ($articles as $node) {
            $items[] = [
                '#theme'    => 'mymodule_article_card',
                '#node'     => $node,
                '#teaser'   => TRUE,
                '#cache'    => ['tags' => $node->getCacheTags()],
            ];
        }
        
        return [
            '#theme'    => 'item_list',
            '#items'    => $items,
            '#title'    => $this->t('Featured Articles'),
            '#list_type'=> 'ul',
            '#attributes'=> ['class' => ['featured-articles']],
            '#cache'    => [
                'tags'    => ['node_list'],
                'contexts'=> ['url.query_args'],
                'max-age' => 3600,
            ],
        ];
    }
    
    public function getCacheMaxAge(): int {
        return \Drupal::config('mymodule.settings')->get('cache_lifetime') ?? 3600;
    }
}
```

---

## ขั้นตอนที่ 805: Event Subscriber

```php
<?php
// src/EventSubscriber/MyModuleSubscriber.php

declare(strict_types=1);

namespace Drupal\mymodule\EventSubscriber;

use Drupal\Core\Routing\RouteMatchInterface;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class MyModuleSubscriber implements EventSubscriberInterface {
    public function __construct(
        private RouteMatchInterface $routeMatch
    ) {}
    
    public static function getSubscribedEvents(): array {
        return [
            KernelEvents::REQUEST  => ['onRequest', 20],
            KernelEvents::RESPONSE => ['onResponse', 0],
        ];
    }
    
    public function onRequest(RequestEvent $event): void {
        if (!$event->isMainRequest()) return;
        
        $request = $event->getRequest();
        $route   = $this->routeMatch->getRouteName();
        
        // Log article views
        if ($route === 'mymodule.article.view') {
            $node = $this->routeMatch->getParameter('node');
            if ($node) {
                \Drupal::database()
                    ->merge('mymodule_node_stats')
                    ->key('nid', $node->id())
                    ->fields(['view_count' => 1, 'last_viewed' => \Drupal::time()->getCurrentTime()])
                    ->expression('view_count', 'view_count + :inc', [':inc' => 1])
                    ->execute();
            }
        }
    }
    
    public function onResponse(ResponseEvent $event): void {
        if (!$event->isMainRequest()) return;
        
        $response = $event->getResponse();
        
        // Add custom header
        $response->headers->set('X-Powered-By-Module', 'MyModule');
    }
}

// Register in mymodule.services.yml:
/*
services:
  mymodule.subscriber:
    class: Drupal\mymodule\EventSubscriber\MyModuleSubscriber
    arguments:
      - '@current_route_match'
    tags:
      - { name: event_subscriber }
*/
```

---

## ขั้นตอนที่ 806: Custom Entity (Advanced)

```php
<?php
// src/Entity/Review.php

declare(strict_types=1);

namespace Drupal\mymodule\Entity;

use Drupal\Core\Entity\ContentEntityBase;
use Drupal\Core\Entity\EntityTypeInterface;
use Drupal\Core\Field\BaseFieldDefinition;
use Drupal\Core\Entity\EntityChangedTrait;
use Drupal\user\EntityOwnerTrait;

/**
 * @ContentEntityType(
 *   id = "mymodule_review",
 *   label = @Translation("Review"),
 *   label_collection = @Translation("Reviews"),
 *   handlers = {
 *     "view_builder" = "Drupal\Core\Entity\EntityViewBuilder",
 *     "list_builder" = "Drupal\mymodule\ReviewListBuilder",
 *     "form" = {
 *       "add"    = "Drupal\mymodule\Form\ReviewForm",
 *       "edit"   = "Drupal\mymodule\Form\ReviewForm",
 *       "delete" = "Drupal\Core\Entity\ContentEntityDeleteForm",
 *     },
 *     "views_data" = "Drupal\views\EntityViewsData",
 *   },
 *   base_table = "mymodule_review",
 *   admin_permission = "administer mymodule",
 *   entity_keys = {
 *     "id"    = "id",
 *     "uuid"  = "uuid",
 *     "label" = "subject",
 *     "owner" = "uid",
 *   },
 *   links = {
 *     "canonical"   = "/reviews/{mymodule_review}",
 *     "add-form"    = "/reviews/add",
 *     "edit-form"   = "/reviews/{mymodule_review}/edit",
 *     "delete-form" = "/reviews/{mymodule_review}/delete",
 *     "collection"  = "/admin/content/reviews",
 *   },
 * )
 */
class Review extends ContentEntityBase {
    use EntityChangedTrait;
    use EntityOwnerTrait;
    
    public static function baseFieldDefinitions(EntityTypeInterface $entity_type): array {
        $fields = parent::baseFieldDefinitions($entity_type);
        $fields += static::ownerBaseFieldDefinitions($entity_type);
        
        $fields['subject'] = BaseFieldDefinition::create('string')
            ->setLabel(t('Subject'))
            ->setRequired(TRUE)
            ->setSetting('max_length', 255)
            ->setDisplayOptions('form', [
                'type'   => 'string_textfield',
                'weight' => -5,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);
        
        $fields['body'] = BaseFieldDefinition::create('text_long')
            ->setLabel(t('Review'))
            ->setRequired(TRUE)
            ->setDisplayOptions('view', [
                'label'  => 'hidden',
                'type'   => 'text_default',
                'weight' => 0,
            ])
            ->setDisplayOptions('form', [
                'type'   => 'text_textarea',
                'weight' => 0,
            ])
            ->setDisplayConfigurable('form', TRUE)
            ->setDisplayConfigurable('view', TRUE);
        
        $fields['rating'] = BaseFieldDefinition::create('integer')
            ->setLabel(t('Rating'))
            ->setRequired(TRUE)
            ->setSetting('min', 1)
            ->setSetting('max', 5)
            ->setDisplayOptions('form', [
                'type'   => 'number',
                'weight' => 5,
            ]);
        
        $fields['status'] = BaseFieldDefinition::create('boolean')
            ->setLabel(t('Published'))
            ->setDefaultValue(FALSE)
            ->setDisplayOptions('form', [
                'type'   => 'boolean_checkbox',
                'weight' => 10,
            ]);
        
        $fields['created'] = BaseFieldDefinition::create('created')
            ->setLabel(t('Created'));
        
        $fields['changed'] = BaseFieldDefinition::create('changed')
            ->setLabel(t('Changed'));
        
        return $fields;
    }
    
    public function getRating(): int {
        return (int) $this->get('rating')->value;
    }
    
    public function getSubject(): string {
        return $this->get('subject')->value ?? '';
    }
    
    public function isPublished(): bool {
        return (bool) $this->get('status')->value;
    }
}
```

---

## 🎯 สรุป Part 32

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Module Files | info.yml, routing.yml, services.yml |
| Config System | Schema, ConfigFormBase |
| Block Plugins | @Block annotation, ContainerFactoryPluginInterface |
| Event Subscriber | KernelEvents, getSubscribedEvents |
| Custom Entity | @ContentEntityType, BaseFieldDefinition |
| Dependency Injection | ContainerInterface, create() pattern |

**ถัดไป → Part 33: Drupal Theming**
