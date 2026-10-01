# Part 33: Drupal Theming
## ขั้นตอนที่ 831-860: สร้าง Theme ด้วย Twig

---

## ขั้นตอนที่ 831: Theme Structure

```
web/themes/custom/mytheme/
├── assets/
│   ├── css/
│   │   ├── style.css
│   │   └── print.css
│   ├── js/
│   │   └── main.js
│   └── images/
├── config/
│   └── install/
│       └── mytheme.settings.yml
├── src/
│   └── Form/
│       └── ThemeSettingsForm.php  # Rarely needed
├── templates/
│   ├── layout/
│   │   ├── html.html.twig
│   │   ├── page.html.twig
│   │   └── region.html.twig
│   ├── content/
│   │   ├── node.html.twig
│   │   ├── node--article.html.twig
│   │   └── node--article--teaser.html.twig
│   ├── block/
│   │   └── block.html.twig
│   ├── form/
│   │   └── input.html.twig
│   └── navigation/
│       └── menu.html.twig
├── mytheme.info.yml
├── mytheme.libraries.yml
├── mytheme.breakpoints.yml
└── mytheme.theme          # Preprocess functions
```

---

## ขั้นตอนที่ 832: Theme Info & Libraries

```yaml
# mytheme.info.yml

name: My Theme
type: theme
description: A professional Drupal 10 theme.
core_version_requirement: ^10
base theme: stable9
regions:
  header:          'Header'
  primary_menu:    'Primary Menu'
  hero:            'Hero'
  breadcrumb:      'Breadcrumb'
  content:         'Content'
  sidebar_first:   'Sidebar First'
  sidebar_second:  'Sidebar Second'
  footer_top:      'Footer Top'
  footer:          'Footer'
  footer_bottom:   'Footer Bottom'
libraries:
  - mytheme/global-styling
  - mytheme/global-scripts
features:
  - logo
  - favicon
  - name
  - slogan
```

```yaml
# mytheme.libraries.yml

global-styling:
  version: VERSION
  css:
    theme:
      assets/css/style.css: {}

global-scripts:
  version: VERSION
  js:
    assets/js/main.js: {}
  dependencies:
    - core/drupal
    - core/jquery

# Page-specific libraries
article-page:
  version: VERSION
  css:
    component:
      assets/css/article.css: {}

# Conditional library (loaded via hook_page_attachments or template)
gallery:
  version: VERSION
  css:
    component:
      assets/css/gallery.css: {}
  js:
    assets/js/gallery.js: {}
  dependencies:
    - core/drupal
```

---

## ขั้นตอนที่ 833: Twig Templates

```twig
{# templates/layout/page.html.twig #}

<!DOCTYPE html>
<html{{ html_attributes }}>
  <head>
    <head-placeholder token="{{ placeholder_token }}">
    <title>{{ head_title | safe_join(' | ') }}</title>
    <css-placeholder token="{{ placeholder_token }}">
    <js-placeholder token="{{ placeholder_token }}">
  </head>
  <body{{ attributes.addClass(body_classes) }}>
    <a href="#main-content" class="sr-only skip-link">
      {{ 'Skip to main content'|t }}
    </a>
    
    <div id="page-wrapper" class="page-wrapper">
      
      {# Header #}
      <header id="header" class="site-header" role="banner">
        <div class="container">
          {% if page.header %}
            {{ page.header }}
          {% endif %}
          
          {% if page.primary_menu %}
            <nav role="navigation" aria-label="{{ 'Main navigation'|t }}">
              {{ page.primary_menu }}
            </nav>
          {% endif %}
        </div>
      </header>
      
      {# Hero - only on front page #}
      {% if is_front and page.hero %}
        <section class="hero-section">
          <div class="container">
            {{ page.hero }}
          </div>
        </section>
      {% endif %}
      
      {# Main Content #}
      <main id="main-content" class="main-content" role="main" tabindex="-1">
        <div class="container">
          
          {% if page.breadcrumb %}
            <div class="breadcrumb-wrapper">
              {{ page.breadcrumb }}
            </div>
          {% endif %}
          
          <div class="layout-content {% if page.sidebar_first or page.sidebar_second %}has-sidebar{% endif %}">
            
            {% if page.sidebar_first %}
              <aside class="sidebar sidebar--first" role="complementary">
                {{ page.sidebar_first }}
              </aside>
            {% endif %}
            
            <section class="content-area">
              {{ page.content }}
            </section>
            
            {% if page.sidebar_second %}
              <aside class="sidebar sidebar--second" role="complementary">
                {{ page.sidebar_second }}
              </aside>
            {% endif %}
            
          </div>
        </div>
      </main>
      
      {# Footer #}
      <footer class="site-footer" role="contentinfo">
        <div class="container">
          {% if page.footer_top %}
            <div class="footer-top">
              {{ page.footer_top }}
            </div>
          {% endif %}
          
          {% if page.footer %}
            <div class="footer-main">
              {{ page.footer }}
            </div>
          {% endif %}
          
          <div class="footer-bottom">
            {% if page.footer_bottom %}
              {{ page.footer_bottom }}
            {% endif %}
            <p class="copyright">
              © {{ "now"|date("Y") }} {{ site_name }}. 
              {{ 'All rights reserved.'|t }}
            </p>
          </div>
        </div>
      </footer>
      
    </div>
    
    <js-bottom-placeholder token="{{ placeholder_token }}">
  </body>
</html>
```

```twig
{# templates/content/node--article.html.twig #}

{%
  set classes = [
    'node',
    'node--type-' ~ node.bundle|clean_class,
    node.isPromoted() ? 'node--promoted',
    node.isSticky() ? 'node--sticky',
    not node.isPublished() ? 'node--unpublished',
    view_mode ? 'node--view-mode-' ~ view_mode|clean_class,
  ]
%}

<article{{ attributes.addClass(classes) }}>
  
  {# Article Header #}
  <header class="node__header">
    {{ title_prefix }}
    {% if not page %}
      <h2{{ title_attributes.addClass('node__title') }}>
        <a href="{{ url }}" rel="bookmark">{{ label }}</a>
      </h2>
    {% endif %}
    {{ title_suffix }}
    
    {% if display_submitted %}
      <div class="node__meta">
        {{ author_picture }}
        <span class="node__submitted">
          {% trans %}
            Posted by {{ author_name }} on {{ date }}
          {% endtrans %}
        </span>
        
        {# Reading time (from preprocess) #}
        {% if reading_time %}
          <span class="reading-time">
            {{ '@min min read'|t({'@min': reading_time}) }}
          </span>
        {% endif %}
      </div>
    {% endif %}
  </header>
  
  {# Featured Image #}
  {% if content.field_image %}
    <div class="node__featured-image">
      {{ content.field_image }}
    </div>
  {% endif %}
  
  {# Content #}
  <div{{ content_attributes.addClass('node__content') }}>
    {{ content|without('field_image', 'field_tags', 'comment') }}
  </div>
  
  {# Tags #}
  {% if content.field_tags %}
    <footer class="node__footer">
      <div class="node__tags">
        <span class="label">{{ 'Tags:'|t }}</span>
        {{ content.field_tags }}
      </div>
    </footer>
  {% endif %}
  
  {# Comments #}
  {% if content.comment %}
    <section class="node__comments">
      {{ content.comment }}
    </section>
  {% endif %}
  
</article>
```

---

## ขั้นตอนที่ 834: Preprocess Functions

```php
<?php
// mytheme.theme

declare(strict_types=1);

use Drupal\node\NodeInterface;
use Drupal\Core\Template\Attribute;

/**
 * Implements template_preprocess_node().
 */
function mytheme_preprocess_node(array &$variables): void {
    /** @var \Drupal\node\NodeInterface $node */
    $node = $variables['node'];
    
    // Add reading time for articles
    if ($node->getType() === 'article' && $node->hasField('body') && !$node->body->isEmpty()) {
        $body_text = $node->body->processed;
        $word_count = str_word_count(strip_tags($body_text));
        $variables['reading_time'] = max(1, (int) round($word_count / 200));
        $variables['word_count'] = number_format($word_count);
    }
    
    // Add author URL
    $variables['author_url'] = $node->getOwner()->toUrl()->toString();
    
    // Add formatted dates
    $variables['created_formatted'] = \Drupal::service('date.formatter')
        ->format($node->getCreatedTime(), 'medium');
}

/**
 * Implements template_preprocess_html().
 */
function mytheme_preprocess_html(array &$variables): void {
    $route_name = \Drupal::routeMatch()->getRouteName();
    
    // Add route as body class
    $variables['attributes']['class'][] = 'route-' . str_replace('.', '-', $route_name ?? '');
    
    // Add language direction
    $language = \Drupal::languageManager()->getCurrentLanguage();
    $variables['attributes']['lang']  = $language->getId();
    $variables['attributes']['dir']   = $language->getDirection();
    
    // Add current user role as class
    $current_user = \Drupal::currentUser();
    if ($current_user->isAuthenticated()) {
        $variables['attributes']['class'][] = 'user-logged-in';
        foreach ($current_user->getRoles() as $role) {
            $variables['attributes']['class'][] = 'role-' . $role;
        }
    }
}

/**
 * Implements template_preprocess_page().
 */
function mytheme_preprocess_page(array &$variables): void {
    $variables['site_name'] = \Drupal::config('system.site')->get('name');
    $variables['site_slogan'] = \Drupal::config('system.site')->get('slogan');
    
    // Breadcrumb
    if (\Drupal::routeMatch()->getRouteName() !== '<front>') {
        $variables['breadcrumb'] = \Drupal::service('breadcrumb')
            ->build(\Drupal::routeMatch())
            ->toRenderable();
    }
}

/**
 * Implements template_preprocess_block().
 */
function mytheme_preprocess_block(array &$variables): void {
    // Add unique class per block ID
    $variables['attributes']['class'][] = 'block--' . str_replace('_', '-', $variables['plugin_id']);
    
    // Remove 'Block' from title if present (cleaner UI)
    if (isset($variables['label'])) {
        $variables['label'] = preg_replace('/\s*Block$/i', '', $variables['label']);
    }
}

/**
 * Implements hook_page_attachments_alter().
 */
function mytheme_page_attachments_alter(array &$attachments): void {
    // Add Open Graph meta tags for article pages
    $route = \Drupal::routeMatch();
    
    if ($route->getRouteName() === 'entity.node.canonical') {
        /** @var \Drupal\node\NodeInterface $node */
        $node = $route->getParameter('node');
        
        if ($node instanceof NodeInterface && $node->getType() === 'article') {
            $attachments['#attached']['html_head'][] = [
                [
                    '#tag'        => 'meta',
                    '#attributes' => [
                        'property' => 'og:title',
                        'content'  => $node->getTitle(),
                    ],
                ],
                'og_title',
            ];
            
            $attachments['#attached']['html_head'][] = [
                [
                    '#tag'        => 'meta',
                    '#attributes' => [
                        'property' => 'og:type',
                        'content'  => 'article',
                    ],
                ],
                'og_type',
            ];
            
            if (!$node->body->isEmpty()) {
                $description = substr(strip_tags($node->body->processed), 0, 200);
                $attachments['#attached']['html_head'][] = [
                    [
                        '#tag'        => 'meta',
                        '#attributes' => [
                            'property' => 'og:description',
                            'content'  => $description,
                        ],
                    ],
                    'og_description',
                ];
            }
        }
    }
}

/**
 * Implements hook_theme_suggestions_node_alter().
 */
function mytheme_theme_suggestions_node_alter(array &$suggestions, array $variables): void {
    /** @var \Drupal\node\NodeInterface $node */
    $node = $variables['elements']['#node'];
    
    // Add suggestion based on view mode
    $view_mode = $variables['elements']['#view_mode'];
    
    // node--article--full.html.twig
    $suggestions[] = 'node__' . $node->bundle() . '__' . $view_mode;
    
    // node--nid--full.html.twig
    $suggestions[] = 'node__' . $node->id() . '__' . $view_mode;
}
```

---

## 🎯 สรุป Part 33

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Theme Structure | info.yml, libraries.yml, templates/ |
| Libraries | CSS/JS per component, dependencies |
| Twig Templates | page, node, block templates |
| Preprocess Functions | Variables, classes, meta tags |
| Theme Suggestions | Dynamic template selection |
| Twig Filters | t(), clean_class, safe_join |

**ถัดไป → Part 34: PHP Performance & Best Practices**
