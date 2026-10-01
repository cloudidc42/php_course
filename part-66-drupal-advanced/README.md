# Part 66: Drupal Advanced

## ขั้นตอนที่ 1841-1870: Drupal Advanced Development

Drupal เป็น CMS ระดับ enterprise ที่มีความยืดหยุ่นสูง รองรับ commerce, migrations และ complex workflows

---

## ขั้นตอนที่ 1841: Drupal Commerce

```php
<?php

declare(strict_types=1);

// modules/custom/my_commerce/src/Plugin/Commerce/CheckoutPane/CustomPane.php
namespace Drupal\my_commerce\Plugin\Commerce\CheckoutPane;

use Drupal\commerce_checkout\Plugin\Commerce\CheckoutPane\CheckoutPaneBase;
use Drupal\commerce_checkout\Plugin\Commerce\CheckoutPane\CheckoutPaneInterface;
use Drupal\Core\Form\FormStateInterface;

/**
 * @CommerceCheckoutPane(
 *   id = "my_custom_pane",
 *   label = @Translation("Custom Checkout Pane"),
 *   default_step = "order_information",
 *   wrapper_element = "fieldset",
 * )
 */
class CustomPane extends CheckoutPaneBase implements CheckoutPaneInterface
{
    public function buildPaneSummary(): array
    {
        return [
            '#plain_text' => $this->t('Custom data collected'),
        ];
    }

    public function buildPaneForm(array $pane_form, FormStateInterface $form_state, array &$complete_form): array
    {
        $pane_form['special_instructions'] = [
            '#type'          => 'textarea',
            '#title'         => $this->t('Special Instructions'),
            '#default_value' => $this->order->getData('special_instructions', ''),
            '#rows'          => 3,
        ];

        $pane_form['gift_wrap'] = [
            '#type'          => 'checkbox',
            '#title'         => $this->t('Gift Wrap (+฿50)'),
            '#default_value' => $this->order->getData('gift_wrap', false),
        ];

        return $pane_form;
    }

    public function validatePaneForm(array &$pane_form, FormStateInterface $form_state, array &$complete_form): void
    {
        $values = $form_state->getValue($pane_form['#parents']);

        if (strlen($values['special_instructions']) > 500) {
            $form_state->setError(
                $pane_form['special_instructions'],
                $this->t('Instructions cannot exceed 500 characters.')
            );
        }
    }

    public function submitPaneForm(array &$pane_form, FormStateInterface $form_state, array &$complete_form): void
    {
        $values = $form_state->getValue($pane_form['#parents']);

        $this->order->setData('special_instructions', $values['special_instructions']);
        $this->order->setData('gift_wrap', (bool) $values['gift_wrap']);

        if ($values['gift_wrap']) {
            // เพิ่ม adjustment สำหรับ gift wrap
            $this->order->addAdjustment(new \Drupal\commerce_order\Adjustment([
                'type'               => 'custom',
                'label'              => $this->t('Gift Wrap'),
                'amount'             => new \Drupal\commerce_price\Price('50', 'THB'),
                'source_id'          => 'gift_wrap',
                'included'           => FALSE,
                'locked'             => TRUE,
            ]));
        }
    }
}
```

---

## ขั้นตอนที่ 1842: Drupal Paragraphs

```php
<?php

declare(strict_types=1);

// modules/custom/my_paragraphs/src/Plugin/paragraphs/Behavior/ResponsiveImageBehavior.php
namespace Drupal\my_paragraphs\Plugin\paragraphs\Behavior;

use Drupal\paragraphs\Annotation\ParagraphsBehavior;
use Drupal\paragraphs\ParagraphsBehaviorBase;
use Drupal\paragraphs\ParagraphInterface;
use Drupal\Core\Form\FormStateInterface;

/**
 * @ParagraphsBehavior(
 *   id = "responsive_image",
 *   label = @Translation("Responsive Image"),
 *   description = @Translation("Add responsive image settings"),
 *   weight = 0,
 * )
 */
class ResponsiveImageBehavior extends ParagraphsBehaviorBase
{
    public function buildBehaviorForm(ParagraphInterface $paragraph, array &$form, FormStateInterface $form_state): array
    {
        $form['full_width'] = [
            '#type'          => 'checkbox',
            '#title'         => $this->t('Full width image'),
            '#default_value' => $paragraph->getBehaviorSetting($this->pluginId, 'full_width', false),
        ];

        $form['image_style'] = [
            '#type'          => 'select',
            '#title'         => $this->t('Image Style'),
            '#options'       => [
                'standard'   => $this->t('Standard'),
                'hero'       => $this->t('Hero'),
                'thumbnail'  => $this->t('Thumbnail'),
            ],
            '#default_value' => $paragraph->getBehaviorSetting($this->pluginId, 'image_style', 'standard'),
        ];

        return $form;
    }

    public function preprocess(array &$variables, ParagraphInterface $paragraph): void
    {
        $isFullWidth = $paragraph->getBehaviorSetting($this->pluginId, 'full_width', false);
        $imageStyle  = $paragraph->getBehaviorSetting($this->pluginId, 'image_style', 'standard');

        $variables['paragraph']['#attributes']['class'][] = "image-style--{$imageStyle}";

        if ($isFullWidth) {
            $variables['paragraph']['#attributes']['class'][] = 'full-width';
        }
    }
}
```

---

## ขั้นตอนที่ 1843: Views Programmatic

```php
<?php

declare(strict_types=1);

// modules/custom/my_module/src/Controller/ArticlesController.php
namespace Drupal\my_module\Controller;

use Drupal\Core\Controller\ControllerBase;
use Drupal\views\Views;
use Symfony\Component\HttpFoundation\Request;

class ArticlesController extends ControllerBase
{
    public function listArticles(Request $request): array
    {
        // Load and execute a View programmatically
        $view = Views::getView('articles');

        if (!$view) {
            return ['#markup' => $this->t('View not found')];
        }

        $view->setDisplay('page_1');
        $view->setArguments([$request->query->get('category', '')]);

        // Apply filters
        $filters = $view->display_handler->getOption('filters');
        if ($request->query->has('status')) {
            $filters['status']['value'] = [$request->query->get('status')];
            $view->display_handler->setOption('filters', $filters);
        }

        $view->execute();

        return [
            '#type'  => 'view',
            '#name'  => 'articles',
            '#display_id' => 'page_1',
            '#arguments'  => [$request->query->get('category', '')],
        ];
    }

    public function getArticlesData(): array
    {
        // Execute view and get raw results
        $view = Views::getView('articles');
        $view->setDisplay('rest_export_1');
        $view->execute();

        $results = [];
        foreach ($view->result as $row) {
            $entity = $row->_entity;
            $results[] = [
                'id'      => $entity->id(),
                'title'   => $entity->getTitle(),
                'created' => $entity->getCreatedTime(),
                'author'  => $entity->getOwner()->getDisplayName(),
            ];
        }

        return $results;
    }
}
```

---

## ขั้นตอนที่ 1844: Drupal Migrations

```php
<?php

declare(strict_types=1);

// modules/custom/my_migration/src/Plugin/migrate/source/LegacyArticleSource.php
namespace Drupal\my_migration\Plugin\migrate\source;

use Drupal\migrate\Plugin\migrate\source\SqlBase;
use Drupal\migrate\Row;

/**
 * @MigrateSource(
 *   id = "legacy_article",
 *   source_module = "my_migration",
 * )
 */
class LegacyArticleSource extends SqlBase
{
    public function query(): \Drupal\Core\Database\Query\SelectInterface
    {
        return $this->select('old_articles', 'a')
            ->fields('a', ['id', 'title', 'body', 'author_id', 'created_date', 'status'])
            ->condition('a.deleted', 0)
            ->orderBy('a.id');
    }

    public function fields(): array
    {
        return [
            'id'           => $this->t('Article ID'),
            'title'        => $this->t('Title'),
            'body'         => $this->t('Body'),
            'author_id'    => $this->t('Author ID'),
            'created_date' => $this->t('Created Date'),
            'status'       => $this->t('Status'),
        ];
    }

    public function getIds(): array
    {
        return ['id' => ['type' => 'integer', 'alias' => 'a']];
    }

    public function prepareRow(Row $row): bool
    {
        // Transform data
        $row->setSourceProperty('computed_status', $row->getSourceProperty('status') === 'Y' ? 1 : 0);

        // Fetch related tags
        $tags = $this->select('old_article_tags', 't')
            ->fields('t', ['tag'])
            ->condition('t.article_id', $row->getSourceProperty('id'))
            ->execute()
            ->fetchCol();

        $row->setSourceProperty('tags', $tags);

        return parent::prepareRow($row);
    }
}
```

```yaml
# modules/custom/my_migration/migrations/migrate_articles.yml
id: legacy_articles
label: 'Migrate Legacy Articles'
migration_group: legacy_content
migration_dependencies:
  required:
    - legacy_users

source:
  plugin: legacy_article
  key: legacy
  database:
    driver: mysql
    host: legacy-db.internal
    database: legacy_site
    username: readonly_user
    password: secret

process:
  type:
    plugin: default_value
    default_value: article

  title: title

  body/value: body
  body/format:
    plugin: default_value
    default_value: full_html

  uid:
    plugin: migration_lookup
    migration: legacy_users
    source: author_id

  status: computed_status

  created:
    plugin: format_date
    source: created_date
    from_format: 'Y-m-d H:i:s'
    to_format: 'U'

  field_tags:
    plugin: entity_lookup
    source: tags
    entity_type: taxonomy_term
    bundle: tags
    value_key: name

destination:
  plugin: 'entity:node'
```

---

## ขั้นตอนที่ 1845: Layout Builder Programmatic

```php
<?php

declare(strict_types=1);

// modules/custom/my_layout/src/Form/AddBlockToLayoutForm.php
namespace Drupal\my_layout\Form;

use Drupal\Core\Form\FormBase;
use Drupal\Core\Form\FormStateInterface;
use Drupal\layout_builder\LayoutEntityHelperTrait;
use Drupal\layout_builder\LayoutTempstoreRepositoryInterface;
use Drupal\layout_builder\SectionStorageInterface;

class AddBlockToLayoutForm extends FormBase
{
    use LayoutEntityHelperTrait;

    public function getFormId(): string
    {
        return 'my_layout_add_block_form';
    }

    public function buildForm(array $form, FormStateInterface $form_state, ?SectionStorageInterface $section_storage = null): array
    {
        $form['block_type'] = [
            '#type'    => 'select',
            '#title'   => $this->t('Block Type'),
            '#options' => [
                'hero_banner'  => $this->t('Hero Banner'),
                'cta_block'    => $this->t('Call to Action'),
                'testimonials' => $this->t('Testimonials'),
            ],
        ];

        $form['region'] = [
            '#type'    => 'select',
            '#title'   => $this->t('Region'),
            '#options' => [
                'content'  => $this->t('Content'),
                'sidebar'  => $this->t('Sidebar'),
            ],
        ];

        $form['submit'] = [
            '#type'  => 'submit',
            '#value' => $this->t('Add Block'),
        ];

        $form['#section_storage'] = $section_storage;
        return $form;
    }

    public function submitForm(array &$form, FormStateInterface $form_state): void
    {
        /** @var SectionStorageInterface $sectionStorage */
        $sectionStorage = $form['#section_storage'];
        $blockType      = $form_state->getValue('block_type');
        $region         = $form_state->getValue('region');

        // Add block to layout section
        $sectionStorage->getSection(0)->appendComponent(
            new \Drupal\layout_builder\SectionComponent(
                \Drupal\Component\Utility\Html::getUniqueId($blockType),
                $region,
                [
                    'id'     => "custom:{$blockType}",
                    'label'  => $this->t(ucfirst(str_replace('_', ' ', $blockType))),
                    'label_display' => '0',
                ]
            )
        );

        // Save tempstore
        \Drupal::service('layout_builder.tempstore_repository')->set($sectionStorage);

        $this->messenger()->addStatus($this->t('Block added to layout.'));
    }
}
```

---

## ขั้นตอนที่ 1846: Drupal Performance Tuning

```php
<?php

declare(strict_types=1);

// modules/custom/performance/src/EventSubscriber/CacheSubscriber.php
namespace Drupal\performance\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class CacheSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::RESPONSE => ['onResponse', 100],
        ];
    }

    public function onResponse(ResponseEvent $event): void
    {
        $request  = $event->getRequest();
        $response = $event->getResponse();

        // เพิ่ม cache headers สำหรับ static content
        if ($this->isStaticContent($request)) {
            $response->setMaxAge(31536000);   // 1 year
            $response->setSharedMaxAge(31536000);
            $response->headers->set('Vary', 'Accept-Encoding');
        }

        // API responses
        if ($request->getPathInfo() === '/api/') {
            $response->setMaxAge(300);
            $response->setSharedMaxAge(300);
            $response->headers->set('CDN-Cache-Control', 'max-age=300');
        }
    }

    private function isStaticContent($request): bool
    {
        $path = $request->getPathInfo();
        return preg_match('/\.(css|js|png|jpg|gif|svg|woff|woff2)$/', $path);
    }
}
```

```bash
# Drupal performance optimization commands
drush cr                     # Clear all caches
drush php:eval "drupal_flush_all_caches();"

# Enable Drupal caching
drush config:set system.performance cache.page.max_age 3600
drush config:set system.performance css.preprocess true
drush config:set system.performance js.preprocess true

# Database optimization
drush sql:query "ANALYZE TABLE node_field_data"
drush sql:query "OPTIMIZE TABLE cache_default"

# Check slow queries
drush sql:query "SHOW FULL PROCESSLIST"
```

---

## สรุป Part 66

| Module | รายละเอียด |
|--------|-----------|
| Commerce | Checkout panes, adjustments |
| Paragraphs | Content components กับ behaviors |
| Views | Programmatic execution |
| Migrations | ETL จาก legacy systems |
| Layout Builder | Drag-and-drop layouts |
| Performance | Cache headers, preprocess assets |

ถัดไป → Part 67: PHP Concurrency
