# Part 57: Search with Elasticsearch

## ขั้นตอนที่ 1571-1600: Elasticsearch และ Full-Text Search

Elasticsearch เป็น distributed search engine ที่ทรงพลัง รองรับ full-text search, aggregations และ real-time analytics

---

## ขั้นตอนที่ 1571: ติดตั้ง Elasticsearch PHP Client

```bash
# ติดตั้ง official Elasticsearch PHP client
composer require elastic/elasticsearch

# หรือใช้ Laravel Scout กับ Elasticsearch driver
composer require laravel/scout
composer require babenkoivan/elastic-scout-driver

# ติดตั้ง Elasticsearch ด้วย Docker
docker run -d --name elasticsearch \
  -p 9200:9200 -p 9300:9300 \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  elasticsearch:8.10.0
```

```php
<?php

declare(strict_types=1);

// app/Services/ElasticsearchService.php
namespace App\Services;

use Elastic\Elasticsearch\Client;
use Elastic\Elasticsearch\ClientBuilder;
use Illuminate\Support\Facades\Log;

class ElasticsearchService
{
    private Client $client;

    public function __construct()
    {
        $this->client = ClientBuilder::create()
            ->setHosts([config('elasticsearch.host', 'http://localhost:9200')])
            ->build();
    }

    public function getClient(): Client
    {
        return $this->client;
    }

    public function indexExists(string $index): bool
    {
        return $this->client->indices()->exists(['index' => $index])->asBool();
    }

    public function createIndex(string $index, array $settings = [], array $mappings = []): void
    {
        if ($this->indexExists($index)) {
            return;
        }

        $params = ['index' => $index, 'body' => []];

        if (!empty($settings)) {
            $params['body']['settings'] = $settings;
        }

        if (!empty($mappings)) {
            $params['body']['mappings'] = $mappings;
        }

        $this->client->indices()->create($params);
        Log::info("Created Elasticsearch index: {$index}");
    }

    public function deleteIndex(string $index): void
    {
        if ($this->indexExists($index)) {
            $this->client->indices()->delete(['index' => $index]);
        }
    }
}
```

---

## ขั้นตอนที่ 1572: การสร้าง Index และ Mappings

```php
<?php

declare(strict_types=1);

// app/Console/Commands/CreateProductIndex.php
namespace App\Console\Commands;

use App\Services\ElasticsearchService;
use Illuminate\Console\Command;

class CreateProductIndex extends Command
{
    protected $signature = 'elasticsearch:create-product-index {--force}';
    protected $description = 'Create the products index in Elasticsearch';

    public function __construct(private ElasticsearchService $es)
    {
        parent::__construct();
    }

    public function handle(): int
    {
        if ($this->option('force')) {
            $this->es->deleteIndex('products');
        }

        $settings = [
            'number_of_shards'   => 1,
            'number_of_replicas' => 0,
            'analysis'           => [
                'analyzer' => [
                    'thai_analyzer' => [
                        'type'      => 'custom',
                        'tokenizer' => 'thai',
                        'filter'    => ['lowercase', 'stop'],
                    ],
                    'product_search' => [
                        'type'      => 'custom',
                        'tokenizer' => 'standard',
                        'filter'    => [
                            'lowercase',
                            'stop',
                            'snowball',
                            'synonym',
                        ],
                    ],
                ],
                'filter' => [
                    'synonym' => [
                        'type'     => 'synonym',
                        'synonyms' => [
                            'phone, smartphone, mobile',
                            'laptop, notebook, computer',
                        ],
                    ],
                ],
            ],
        ];

        $mappings = [
            'properties' => [
                'id' => ['type' => 'integer'],

                'name' => [
                    'type'   => 'text',
                    'fields' => [
                        'keyword' => ['type' => 'keyword'],
                        'suggest' => [
                            'type'     => 'completion',
                            'analyzer' => 'standard',
                        ],
                    ],
                    'analyzer' => 'product_search',
                ],

                'description' => [
                    'type'     => 'text',
                    'analyzer' => 'product_search',
                ],

                'price' => ['type' => 'float'],

                'category' => [
                    'type'       => 'keyword',
                    'fields'     => [
                        'text' => ['type' => 'text'],
                    ],
                ],

                'tags' => ['type' => 'keyword'],

                'brand' => ['type' => 'keyword'],

                'rating' => ['type' => 'float'],

                'stock' => ['type' => 'integer'],

                'in_stock' => ['type' => 'boolean'],

                'images' => [
                    'type'    => 'nested',
                    'properties' => [
                        'url'     => ['type' => 'keyword'],
                        'alt'     => ['type' => 'text'],
                        'is_main' => ['type' => 'boolean'],
                    ],
                ],

                'attributes' => [
                    'type'    => 'nested',
                    'properties' => [
                        'name'  => ['type' => 'keyword'],
                        'value' => ['type' => 'keyword'],
                    ],
                ],

                'created_at' => ['type' => 'date'],
                'updated_at' => ['type' => 'date'],
            ],
        ];

        $this->es->createIndex('products', $settings, $mappings);
        $this->info('Products index created successfully!');

        return self::SUCCESS;
    }
}
```

---

## ขั้นตอนที่ 1573: Indexing Documents

```php
<?php

declare(strict_types=1);

// app/Services/ProductIndexingService.php
namespace App\Services;

use App\Models\Product;
use Elastic\Elasticsearch\Client;
use Illuminate\Support\Collection;

class ProductIndexingService
{
    private const INDEX = 'products';

    public function __construct(private Client $client) {}

    public function indexProduct(Product $product): void
    {
        $this->client->index([
            'index' => self::INDEX,
            'id'    => $product->id,
            'body'  => $this->buildDocument($product),
        ]);
    }

    public function bulkIndex(Collection $products): array
    {
        $params = ['body' => []];

        foreach ($products as $product) {
            $params['body'][] = [
                'index' => [
                    '_index' => self::INDEX,
                    '_id'    => $product->id,
                ],
            ];
            $params['body'][] = $this->buildDocument($product);
        }

        $response = $this->client->bulk($params);

        $errors = [];
        if ($response['errors']) {
            foreach ($response['items'] as $item) {
                if (isset($item['index']['error'])) {
                    $errors[] = $item['index']['error'];
                }
            }
        }

        return [
            'took'   => $response['took'],
            'errors' => $errors,
            'count'  => count($products),
        ];
    }

    public function updateProduct(Product $product): void
    {
        $this->client->update([
            'index' => self::INDEX,
            'id'    => $product->id,
            'body'  => [
                'doc' => $this->buildDocument($product),
            ],
        ]);
    }

    public function deleteProduct(int $productId): void
    {
        $this->client->delete([
            'index' => self::INDEX,
            'id'    => $productId,
        ]);
    }

    private function buildDocument(Product $product): array
    {
        return [
            'id'          => $product->id,
            'name'        => $product->name,
            'description' => $product->description,
            'price'       => (float) $product->price,
            'category'    => $product->category?->name,
            'brand'       => $product->brand?->name,
            'tags'        => $product->tags->pluck('name')->toArray(),
            'rating'      => (float) $product->average_rating,
            'stock'       => $product->stock_quantity,
            'in_stock'    => $product->stock_quantity > 0,
            'images'      => $product->images->map(fn ($img) => [
                'url'     => $img->url,
                'alt'     => $img->alt_text,
                'is_main' => $img->is_main,
            ])->toArray(),
            'attributes'  => $product->attributes->map(fn ($attr) => [
                'name'  => $attr->name,
                'value' => $attr->value,
            ])->toArray(),
            'created_at'  => $product->created_at->toISOString(),
            'updated_at'  => $product->updated_at->toISOString(),
        ];
    }
}
```

---

## ขั้นตอนที่ 1574: Full-Text Search Queries

```php
<?php

declare(strict_types=1);

// app/Services/ProductSearchService.php
namespace App\Services;

use Elastic\Elasticsearch\Client;
use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Support\Collection;

class ProductSearchService
{
    private const INDEX = 'products';
    private const DEFAULT_SIZE = 20;

    public function __construct(private Client $client) {}

    public function search(array $params): array
    {
        $query   = $params['query'] ?? '';
        $filters = $params['filters'] ?? [];
        $sort    = $params['sort'] ?? 'relevance';
        $page    = (int) ($params['page'] ?? 1);
        $size    = (int) ($params['size'] ?? self::DEFAULT_SIZE);

        $body = $this->buildSearchQuery($query, $filters, $sort);
        $body['from'] = ($page - 1) * $size;
        $body['size'] = $size;

        // Add aggregations สำหรับ facets
        $body['aggs'] = $this->buildAggregations();

        $response = $this->client->search([
            'index' => self::INDEX,
            'body'  => $body,
        ]);

        return $this->formatResponse($response, $page, $size);
    }

    private function buildSearchQuery(string $query, array $filters, string $sort): array
    {
        $must   = [];
        $filter = [];

        // Full-text search
        if (!empty($query)) {
            $must[] = [
                'multi_match' => [
                    'query'  => $query,
                    'fields' => [
                        'name^3',           // boost name field
                        'name.keyword^2',
                        'description',
                        'tags',
                        'brand',
                        'category',
                    ],
                    'type'          => 'best_fields',
                    'fuzziness'     => 'AUTO',
                    'prefix_length' => 2,
                ],
            ];
        }

        // Filters
        if (!empty($filters['category'])) {
            $filter[] = ['terms' => ['category' => (array) $filters['category']]];
        }

        if (!empty($filters['brand'])) {
            $filter[] = ['terms' => ['brand' => (array) $filters['brand']]];
        }

        if (!empty($filters['price_min']) || !empty($filters['price_max'])) {
            $range = [];
            if (!empty($filters['price_min'])) {
                $range['gte'] = (float) $filters['price_min'];
            }
            if (!empty($filters['price_max'])) {
                $range['lte'] = (float) $filters['price_max'];
            }
            $filter[] = ['range' => ['price' => $range]];
        }

        if (isset($filters['in_stock']) && $filters['in_stock']) {
            $filter[] = ['term' => ['in_stock' => true]];
        }

        if (!empty($filters['rating_min'])) {
            $filter[] = ['range' => ['rating' => ['gte' => (float) $filters['rating_min']]]];
        }

        // Build final query
        $boolQuery = [];
        if (!empty($must)) {
            $boolQuery['must'] = $must;
        } else {
            $boolQuery['must'] = [['match_all' => (object)[]]];
        }

        if (!empty($filter)) {
            $boolQuery['filter'] = $filter;
        }

        // Sort
        $sortClause = match ($sort) {
            'price_asc'   => [['price' => 'asc']],
            'price_desc'  => [['price' => 'desc']],
            'newest'      => [['created_at' => 'desc']],
            'rating'      => [['rating' => 'desc']],
            default       => ['_score'],
        };

        return [
            'query'   => ['bool' => $boolQuery],
            'sort'    => $sortClause,
            'highlight' => [
                'fields' => [
                    'name'        => ['number_of_fragments' => 0],
                    'description' => ['fragment_size' => 150, 'number_of_fragments' => 3],
                ],
                'pre_tags'  => ['<mark>'],
                'post_tags' => ['</mark>'],
            ],
        ];
    }

    private function buildAggregations(): array
    {
        return [
            'categories' => [
                'terms' => [
                    'field' => 'category',
                    'size'  => 20,
                ],
            ],
            'brands' => [
                'terms' => [
                    'field' => 'brand',
                    'size'  => 30,
                ],
            ],
            'price_ranges' => [
                'range' => [
                    'field'  => 'price',
                    'ranges' => [
                        ['to' => 500],
                        ['from' => 500, 'to' => 1000],
                        ['from' => 1000, 'to' => 5000],
                        ['from' => 5000, 'to' => 10000],
                        ['from' => 10000],
                    ],
                ],
            ],
            'avg_price' => [
                'avg' => ['field' => 'price'],
            ],
            'max_price' => [
                'max' => ['field' => 'price'],
            ],
            'min_price' => [
                'min' => ['field' => 'price'],
            ],
            'ratings' => [
                'histogram' => [
                    'field'    => 'rating',
                    'interval' => 1,
                    'min_doc_count' => 1,
                ],
            ],
        ];
    }

    private function formatResponse(mixed $response, int $page, int $size): array
    {
        $hits  = $response['hits'];
        $total = $hits['total']['value'];

        $items = collect($hits['hits'])->map(function ($hit) {
            $source               = $hit['_source'];
            $source['_score']     = $hit['_score'];
            $source['highlights'] = $hit['highlight'] ?? [];
            return $source;
        });

        $facets = [];
        if (isset($response['aggregations'])) {
            $aggs = $response['aggregations'];

            $facets['categories']   = collect($aggs['categories']['buckets'])
                ->map(fn ($b) => ['name' => $b['key'], 'count' => $b['doc_count']])
                ->toArray();

            $facets['brands']       = collect($aggs['brands']['buckets'])
                ->map(fn ($b) => ['name' => $b['key'], 'count' => $b['doc_count']])
                ->toArray();

            $facets['price_ranges'] = collect($aggs['price_ranges']['buckets'])
                ->map(fn ($b) => [
                    'from'  => $b['from'] ?? null,
                    'to'    => $b['to'] ?? null,
                    'count' => $b['doc_count'],
                ])
                ->toArray();

            $facets['price_stats'] = [
                'avg' => $aggs['avg_price']['value'],
                'min' => $aggs['min_price']['value'],
                'max' => $aggs['max_price']['value'],
            ];
        }

        return [
            'total'      => $total,
            'page'       => $page,
            'per_page'   => $size,
            'last_page'  => (int) ceil($total / $size),
            'items'      => $items->toArray(),
            'facets'     => $facets,
            'took_ms'    => $response['took'],
        ];
    }

    public function suggest(string $query): array
    {
        $response = $this->client->search([
            'index' => self::INDEX,
            'body'  => [
                'suggest' => [
                    'product_suggest' => [
                        'prefix'     => $query,
                        'completion' => [
                            'field' => 'name.suggest',
                            'size'  => 5,
                        ],
                    ],
                ],
                '_source' => false,
                'size'    => 0,
            ],
        ]);

        return collect($response['suggest']['product_suggest'][0]['options'])
            ->map(fn ($opt) => $opt['text'])
            ->toArray();
    }
}
```

---

## ขั้นตอนที่ 1575: Laravel Scout กับ Elasticsearch

```php
<?php

declare(strict_types=1);

// app/Models/Product.php (Scout integration)
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Laravel\Scout\Searchable;

class Product extends Model
{
    use Searchable;

    protected $fillable = ['name', 'description', 'price', 'category_id', 'stock_quantity'];

    public function searchableAs(): string
    {
        return 'products';
    }

    public function toSearchableArray(): array
    {
        return [
            'id'          => $this->id,
            'name'        => $this->name,
            'description' => $this->description,
            'price'       => (float) $this->price,
            'category'    => $this->category?->name,
            'brand'       => $this->brand?->name,
            'tags'        => $this->tags->pluck('name')->toArray(),
            'stock'       => $this->stock_quantity,
            'in_stock'    => $this->stock_quantity > 0,
        ];
    }

    public function shouldBeSearchable(): bool
    {
        return $this->status === 'active';
    }
}
```

```bash
# Import existing records
php artisan scout:import "App\Models\Product"

# Flush index
php artisan scout:flush "App\Models\Product"
```

```php
<?php

declare(strict_types=1);

// การค้นหาด้วย Scout
namespace App\Http\Controllers;

use App\Models\Product;
use Illuminate\Http\Request;

class ProductController extends Controller
{
    public function search(Request $request)
    {
        $products = Product::search($request->q)
            ->where('in_stock', true)
            ->paginate(20);

        return response()->json($products);
    }
}
```

---

## ขั้นตอนที่ 1576: Meilisearch Integration

```bash
# ติดตั้ง Meilisearch
docker run -d --name meilisearch \
  -p 7700:7700 \
  -v $(pwd)/meili_data:/meili_data \
  getmeili/meilisearch:latest

# ติดตั้ง PHP client
composer require meilisearch/meilisearch-php
```

```php
<?php

declare(strict_types=1);

// config/scout.php
return [
    'driver' => env('SCOUT_DRIVER', 'meilisearch'),

    'meilisearch' => [
        'host'   => env('MEILISEARCH_HOST', 'http://localhost:7700'),
        'key'    => env('MEILISEARCH_KEY'),
        'index-settings' => [
            'products' => [
                'filterableAttributes' => [
                    'category',
                    'brand',
                    'price',
                    'in_stock',
                    'rating',
                ],
                'sortableAttributes' => [
                    'price',
                    'rating',
                    'created_at',
                ],
                'searchableAttributes' => [
                    'name',
                    'description',
                    'category',
                    'brand',
                    'tags',
                ],
                'rankingRules' => [
                    'words',
                    'typo',
                    'proximity',
                    'attribute',
                    'sort',
                    'exactness',
                    'rating:desc',
                ],
            ],
        ],
    ],
];
```

```php
<?php

declare(strict_types=1);

// app/Services/MeilisearchService.php
namespace App\Services;

use MeiliSearch\Client;

class MeilisearchService
{
    private Client $client;

    public function __construct()
    {
        $this->client = new Client(
            config('scout.meilisearch.host'),
            config('scout.meilisearch.key')
        );
    }

    public function search(string $indexName, string $query, array $options = []): array
    {
        $index = $this->client->index($indexName);

        $defaults = [
            'limit'               => 20,
            'offset'              => 0,
            'attributesToHighlight' => ['name', 'description'],
            'highlightPreTag'     => '<mark>',
            'highlightPostTag'    => '</mark>',
            'facets'              => ['category', 'brand'],
        ];

        $searchOptions = array_merge($defaults, $options);

        return $index->search($query, $searchOptions)->getRaw();
    }

    public function getSearchStats(string $indexName): array
    {
        return $this->client->index($indexName)->stats();
    }

    public function updateSettings(string $indexName, array $settings): void
    {
        $this->client->index($indexName)->updateSettings($settings);
    }
}
```

---

## ขั้นตอนที่ 1577: Aggregations และ Analytics

```php
<?php

declare(strict_types=1);

// app/Services/SearchAnalyticsService.php
namespace App\Services;

use Elastic\Elasticsearch\Client;

class SearchAnalyticsService
{
    public function __construct(private Client $client) {}

    public function getCategoryStats(): array
    {
        $response = $this->client->search([
            'index' => 'products',
            'body'  => [
                'size' => 0,
                'aggs' => [
                    'categories' => [
                        'terms' => [
                            'field' => 'category',
                            'size'  => 50,
                        ],
                        'aggs' => [
                            'avg_price'    => ['avg' => ['field' => 'price']],
                            'total_stock'  => ['sum' => ['field' => 'stock']],
                            'avg_rating'   => ['avg' => ['field' => 'rating']],
                            'price_ranges' => [
                                'range' => [
                                    'field'  => 'price',
                                    'ranges' => [
                                        ['to' => 1000],
                                        ['from' => 1000, 'to' => 5000],
                                        ['from' => 5000],
                                    ],
                                ],
                            ],
                        ],
                    ],
                ],
            ],
        ]);

        return collect($response['aggregations']['categories']['buckets'])
            ->map(fn ($bucket) => [
                'category'    => $bucket['key'],
                'count'       => $bucket['doc_count'],
                'avg_price'   => round($bucket['avg_price']['value'] ?? 0, 2),
                'total_stock' => $bucket['total_stock']['value'] ?? 0,
                'avg_rating'  => round($bucket['avg_rating']['value'] ?? 0, 2),
            ])
            ->toArray();
    }

    public function getSalesTrend(int $days = 30): array
    {
        $response = $this->client->search([
            'index' => 'orders',
            'body'  => [
                'size'  => 0,
                'query' => [
                    'range' => [
                        'created_at' => [
                            'gte' => now()->subDays($days)->toISOString(),
                        ],
                    ],
                ],
                'aggs' => [
                    'sales_over_time' => [
                        'date_histogram' => [
                            'field'             => 'created_at',
                            'calendar_interval' => 'day',
                            'format'            => 'yyyy-MM-dd',
                        ],
                        'aggs' => [
                            'revenue'       => ['sum' => ['field' => 'total']],
                            'order_count'   => ['value_count' => ['field' => 'id']],
                            'avg_order_val' => ['avg' => ['field' => 'total']],
                        ],
                    ],
                ],
            ],
        ]);

        return collect($response['aggregations']['sales_over_time']['buckets'])
            ->map(fn ($bucket) => [
                'date'         => $bucket['key_as_string'],
                'orders'       => $bucket['doc_count'],
                'revenue'      => $bucket['revenue']['value'],
                'avg_order'    => round($bucket['avg_order_val']['value'] ?? 0, 2),
            ])
            ->toArray();
    }
}
```

---

## ขั้นตอนที่ 1578: Search Observers และ Real-time Indexing

```php
<?php

declare(strict_types=1);

// app/Observers/ProductObserver.php
namespace App\Observers;

use App\Jobs\IndexProductJob;
use App\Jobs\DeleteProductFromIndexJob;
use App\Models\Product;

class ProductObserver
{
    public function created(Product $product): void
    {
        if ($product->shouldBeSearchable()) {
            IndexProductJob::dispatch($product)->onQueue('search');
        }
    }

    public function updated(Product $product): void
    {
        if ($product->shouldBeSearchable()) {
            IndexProductJob::dispatch($product)->onQueue('search');
        } else {
            DeleteProductFromIndexJob::dispatch($product->id)->onQueue('search');
        }
    }

    public function deleted(Product $product): void
    {
        DeleteProductFromIndexJob::dispatch($product->id)->onQueue('search');
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Jobs/IndexProductJob.php
namespace App\Jobs;

use App\Models\Product;
use App\Services\ProductIndexingService;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class IndexProductJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public int $tries     = 3;
    public int $backoff   = 60;

    public function __construct(private Product $product) {}

    public function handle(ProductIndexingService $indexingService): void
    {
        $indexingService->indexProduct($this->product);
    }

    public function failed(\Throwable $e): void
    {
        \Log::error("Failed to index product {$this->product->id}: " . $e->getMessage());
    }
}
```

---

## สรุป Part 57

| หัวข้อ | รายละเอียด |
|--------|-----------|
| Elasticsearch Client | การเชื่อมต่อและ setup client |
| Index Mappings | กำหนด field types, analyzers |
| Document Indexing | index เดียว, bulk indexing |
| Full-text Search | multi_match, fuzziness, boosting |
| Aggregations | facets, stats, date histograms |
| Laravel Scout | integration กับ model |
| Meilisearch | alternative search engine |
| Real-time Indexing | Observer + Queue jobs |

ถัดไป → Part 58: Advanced Caching Strategies
