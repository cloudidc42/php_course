# Part 62: Laravel Scout

## ขั้นตอนที่ 1721-1750: Laravel Scout - Full-Text Search

Laravel Scout เป็น driver-based full-text search สำหรับ Eloquent models รองรับ Algolia, Meilisearch และ Database driver

---

## ขั้นตอนที่ 1721: ติดตั้งและ Configure Scout

```bash
# ติดตั้ง Scout
composer require laravel/scout

# ติดตั้ง Meilisearch driver
composer require meilisearch/meilisearch-php http-interop/http-factory-guzzle

# หรือ Algolia
composer require algolia/algoliasearch-client-php

# Publish config
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

```php
<?php

declare(strict_types=1);

// config/scout.php
return [
    'driver' => env('SCOUT_DRIVER', 'meilisearch'),

    'prefix' => env('SCOUT_PREFIX', ''),

    'queue' => [
        'connection' => env('SCOUT_QUEUE_CONNECTION', false),
        'queue'      => env('SCOUT_QUEUE', 'scout'),
    ],

    'after_commit' => false,

    'chunk' => [
        'searchable'   => 500,
        'unsearchable' => 500,
    ],

    'soft_delete' => false,

    'identify' => false,

    'algolia' => [
        'id'     => env('ALGOLIA_APP_ID', ''),
        'secret' => env('ALGOLIA_SECRET', ''),
    ],

    'meilisearch' => [
        'host'   => env('MEILISEARCH_HOST', 'http://localhost:7700'),
        'key'    => env('MEILISEARCH_KEY', null),
        'index-settings' => [
            'products' => [
                'filterableAttributes'  => ['category', 'brand', 'price', 'in_stock'],
                'sortableAttributes'    => ['price', 'rating', 'created_at'],
                'searchableAttributes'  => ['name', 'description', 'tags'],
            ],
        ],
    ],

    'database-driver' => [
        'connection' => env('DB_CONNECTION', 'mysql'),
    ],
];
```

---

## ขั้นตอนที่ 1722: Searchable Model

```php
<?php

declare(strict_types=1);

// app/Models/Article.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;
use Laravel\Scout\Searchable;

class Article extends Model
{
    use Searchable;

    protected $fillable = [
        'title', 'slug', 'content', 'excerpt', 'status',
        'author_id', 'category_id', 'published_at',
    ];

    protected $casts = [
        'published_at' => 'datetime',
        'metadata'     => 'array',
    ];

    // ชื่อ index ใน search engine
    public function searchableAs(): string
    {
        return config('scout.prefix') . 'articles';
    }

    // ข้อมูลที่จะ index
    public function toSearchableArray(): array
    {
        $this->loadMissing(['author', 'category', 'tags']);

        return [
            'id'           => $this->id,
            'title'        => $this->title,
            'slug'         => $this->slug,
            'content'      => strip_tags($this->content),
            'excerpt'      => $this->excerpt,
            'author'       => $this->author?->name,
            'author_id'    => $this->author_id,
            'category'     => $this->category?->name,
            'category_id'  => $this->category_id,
            'tags'         => $this->tags->pluck('name')->toArray(),
            'status'       => $this->status,
            'published_at' => $this->published_at?->timestamp,
            'view_count'   => $this->view_count ?? 0,
        ];
    }

    // เงื่อนไขว่า record ควร index หรือไม่
    public function shouldBeSearchable(): bool
    {
        return $this->status === 'published' && $this->published_at !== null;
    }

    // Relationships
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'author_id');
    }

    public function category(): BelongsTo
    {
        return $this->belongsTo(Category::class);
    }

    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);
    }

    // Scopes สำหรับ Scout
    public function scopeSearch($query, string $term)
    {
        return $query->where('title', 'LIKE', "%{$term}%")
                     ->orWhere('content', 'LIKE', "%{$term}%");
    }
}
```

---

## ขั้นตอนที่ 1723: Search Controller

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/SearchController.php
namespace App\Http\Controllers;

use App\Models\Article;
use App\Models\Product;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;
use Laravel\Scout\Builder;

class SearchController extends Controller
{
    public function search(Request $request): JsonResponse
    {
        $request->validate([
            'q'        => 'required|string|min:2|max:100',
            'category' => 'nullable|string',
            'author'   => 'nullable|integer',
            'from'     => 'nullable|date',
            'to'       => 'nullable|date',
            'sort'     => 'nullable|in:relevance,newest,popular',
            'page'     => 'nullable|integer|min:1',
            'per_page' => 'nullable|integer|min:5|max:50',
        ]);

        $query   = $request->string('q')->toString();
        $perPage = (int) $request->input('per_page', 15);

        $builder = Article::search($query)
            ->query(fn ($query) => $query->with(['author', 'category', 'tags']));

        // Apply filters
        if ($request->filled('category')) {
            $builder->where('category', $request->category);
        }

        if ($request->filled('author')) {
            $builder->where('author_id', (int) $request->author);
        }

        // Date range
        if ($request->filled('from')) {
            $builder->where('published_at', '>=', strtotime($request->from));
        }

        if ($request->filled('to')) {
            $builder->where('published_at', '<=', strtotime($request->to));
        }

        // Sorting
        match ($request->input('sort', 'relevance')) {
            'newest'  => $builder->orderBy('published_at', 'desc'),
            'popular' => $builder->orderBy('view_count', 'desc'),
            default   => null, // relevance = default by search engine
        };

        $results = $builder->paginate($perPage);

        return response()->json([
            'query'   => $query,
            'total'   => $results->total(),
            'results' => $results->items(),
            'meta'    => [
                'current_page' => $results->currentPage(),
                'last_page'    => $results->lastPage(),
                'per_page'     => $results->perPage(),
            ],
        ]);
    }

    public function globalSearch(Request $request): JsonResponse
    {
        $request->validate(['q' => 'required|string|min:2']);

        $query = $request->string('q')->toString();

        // Search across multiple models
        $articles = Article::search($query)->limit(5)->get();
        $products = Product::search($query)->limit(5)->get();

        return response()->json([
            'articles' => $articles,
            'products' => $products,
        ]);
    }
}
```

---

## ขั้นตอนที่ 1724: Algolia Integration

```php
<?php

declare(strict_types=1);

// app/Console/Commands/ConfigureAlgoliaIndex.php
namespace App\Console\Commands;

use Algolia\AlgoliaSearch\SearchClient;
use Illuminate\Console\Command;

class ConfigureAlgoliaIndex extends Command
{
    protected $signature = 'algolia:configure';
    protected $description = 'Configure Algolia index settings';

    public function handle(): int
    {
        $client = SearchClient::create(
            config('scout.algolia.id'),
            config('scout.algolia.secret')
        );

        $index = $client->initIndex('articles');

        // ตั้งค่า searchable attributes
        $index->setSettings([
            'searchableAttributes' => [
                'title',
                'unordered(content)',
                'author',
                'tags',
                'category',
            ],

            // Custom ranking
            'customRanking' => [
                'desc(view_count)',
                'desc(published_at)',
            ],

            // Facets สำหรับ filtering
            'attributesForFaceting' => [
                'searchable(category)',
                'searchable(author)',
                'filterOnly(status)',
                'filterOnly(published_at)',
            ],

            // Highlighting
            'attributesToHighlight'  => ['title', 'content', 'excerpt'],
            'highlightPreTag'        => '<mark>',
            'highlightPostTag'       => '</mark>',
            'snippetEllipsisText'    => '...',
            'attributesToSnippet'    => ['content:50'],

            // Pagination
            'hitsPerPage' => 15,
            'paginationLimitedTo' => 1000,

            // Typo tolerance
            'typoTolerance'          => true,
            'minWordSizefor1Typo'    => 4,
            'minWordSizefor2Typos'   => 8,

            // Language
            'ignorePlurals' => true,
            'removeStopWords' => ['en', 'th'],
        ]);

        $this->info('Algolia index configured!');
        return self::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1725: Custom Scout Driver

```php
<?php

declare(strict_types=1);

// app/Search/TypesenseDriver.php
namespace App\Search;

use Laravel\Scout\Builder;
use Laravel\Scout\Engines\Engine;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Support\LazyCollection;
use Typesense\Client;

class TypesenseEngine extends Engine
{
    public function __construct(private Client $typesense) {}

    public function update($models): void
    {
        if ($models->isEmpty()) return;

        $collectionName = $models->first()->searchableAs();

        $documents = $models->map(fn ($model) => $model->toSearchableArray())->all();

        $this->typesense->collections[$collectionName]->documents->import($documents, ['action' => 'upsert']);
    }

    public function delete($models): void
    {
        if ($models->isEmpty()) return;

        $collectionName = $models->first()->searchableAs();

        $models->each(fn ($model) =>
            $this->typesense->collections[$collectionName]->documents[(string) $model->getScoutKey()]->delete()
        );
    }

    public function search(Builder $builder): mixed
    {
        return $this->performSearch($builder, ['per_page' => $builder->limit ?? 20]);
    }

    public function paginate(Builder $builder, $perPage, $page): mixed
    {
        return $this->performSearch($builder, [
            'per_page' => $perPage,
            'page'     => $page,
        ]);
    }

    protected function performSearch(Builder $builder, array $options = []): array
    {
        $collectionName = $builder->model->searchableAs();

        $parameters = [
            'q'                      => $builder->query ?: '*',
            'query_by'               => 'title,content,tags',
            'per_page'               => $options['per_page'] ?? 15,
            'page'                   => $options['page'] ?? 1,
            'highlight_start_tag'    => '<mark>',
            'highlight_end_tag'      => '</mark>',
        ];

        // Filters
        if (!empty($builder->wheres)) {
            $filterBy = collect($builder->wheres)
                ->map(fn ($value, $key) => "{$key}:={$value}")
                ->implode(' && ');
            $parameters['filter_by'] = $filterBy;
        }

        // Ordering
        if (!empty($builder->orders)) {
            $sortBy = collect($builder->orders)
                ->map(fn ($order) => "{$order['column']}:{$order['direction']}")
                ->implode(',');
            $parameters['sort_by'] = $sortBy;
        }

        return $this->typesense->collections[$collectionName]->documents->search($parameters);
    }

    public function mapIds($results): \Illuminate\Support\Collection
    {
        return collect($results['hits'])->pluck('document.id');
    }

    public function map(Builder $builder, $results, $model): Collection
    {
        if (count($results['hits']) === 0) {
            return $model->newCollection();
        }

        $ids    = $this->mapIds($results)->toArray();
        $models = $model->getScoutModelsByIds($builder, $ids);

        return $models->sortBy(fn ($model) => array_search($model->getScoutKey(), $ids))->values();
    }

    public function lazyMap(Builder $builder, $results, $model): LazyCollection
    {
        return LazyCollection::make($this->map($builder, $results, $model));
    }

    public function getTotalCount($results): int
    {
        return $results['found'] ?? 0;
    }

    public function flush($model): void
    {
        $this->typesense->collections[$model->searchableAs()]->delete();
    }

    public function createIndex($name, array $options = []): void
    {
        // Create Typesense collection
    }

    public function deleteIndex($name): void
    {
        $this->typesense->collections[$name]->delete();
    }
}

// Register custom driver
// app/Providers/AppServiceProvider.php
public function boot(): void
{
    resolve(\Laravel\Scout\EngineManager::class)->extend('typesense', function () {
        $client = new \Typesense\Client([
            'api_key'  => config('services.typesense.api_key'),
            'nodes'    => [['host' => 'localhost', 'port' => '8108', 'protocol' => 'http']],
        ]);
        return new \App\Search\TypesenseEngine($client);
    });
}
```

---

## ขั้นตอนที่ 1726: Real-time Search กับ Debouncing

```javascript
// resources/js/search.js - Alpine.js search component

document.addEventListener('alpine:init', () => {
    Alpine.data('searchComponent', () => ({
        query: '',
        results: [],
        isLoading: false,
        showResults: false,
        debounceTimer: null,
        
        init() {
            this.$watch('query', (value) => {
                this.handleSearch(value);
            });
        },
        
        handleSearch(value) {
            clearTimeout(this.debounceTimer);
            
            if (value.length < 2) {
                this.results = [];
                this.showResults = false;
                return;
            }
            
            this.debounceTimer = setTimeout(() => {
                this.performSearch(value);
            }, 300); // 300ms debounce
        },
        
        async performSearch(value) {
            this.isLoading = true;
            
            try {
                const response = await fetch(`/api/search?q=${encodeURIComponent(value)}`);
                const data = await response.json();
                
                this.results = data.results;
                this.showResults = true;
            } catch (error) {
                console.error('Search failed:', error);
            } finally {
                this.isLoading = false;
            }
        },
        
        clearSearch() {
            this.query = '';
            this.results = [];
            this.showResults = false;
        }
    }));
});
```

```php
<?php

declare(strict_types=1);

// app/Http/Controllers/Api/SearchController.php
namespace App\Http\Controllers\Api;

use App\Models\Article;
use App\Models\Product;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class SearchController extends \App\Http\Controllers\Controller
{
    public function suggestions(Request $request): JsonResponse
    {
        $request->validate(['q' => 'required|string|min:1|max:50']);

        $query = $request->string('q')->toString();

        // ค้นหาจาก multiple models
        $articles = Article::search($query)
            ->take(3)
            ->get(['id', 'title', 'slug', 'category'])
            ->map(fn ($a) => [
                'type'  => 'article',
                'title' => $a->title,
                'url'   => route('articles.show', $a->slug),
            ]);

        $products = Product::search($query)
            ->take(3)
            ->get(['id', 'name', 'slug', 'price'])
            ->map(fn ($p) => [
                'type'  => 'product',
                'title' => $p->name,
                'url'   => route('products.show', $p->slug),
                'price' => number_format($p->price, 2),
            ]);

        return response()->json([
            'suggestions' => $articles->concat($products)->take(6),
        ]);
    }
}
```

---

## สรุป Part 62

| Feature | Driver | รายละเอียด |
|---------|--------|-----------|
| Searchable Trait | ทุก driver | Auto index/update/delete |
| toSearchableArray | ทุก driver | กำหนดข้อมูลที่ index |
| shouldBeSearchable | ทุก driver | เงื่อนไขการ index |
| Filters/Wheres | Algolia/Meilisearch | Attribute filtering |
| Facets | Algolia | Category counts |
| Highlighting | Algolia/Meilisearch | Mark search terms |
| Custom Driver | Custom | รองรับ engine อื่น |
| Real-time Search | JavaScript | Debounce + API |

ถัดไป → Part 63: Real-Time Application Development
