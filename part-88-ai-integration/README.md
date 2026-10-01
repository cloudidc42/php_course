# Part 88: AI Integration กับ PHP
## ขั้นตอนที่ 2501-2530: OpenAI API, ChatGPT, DALL-E และ Embeddings

การเชื่อมต่อ AI APIs กับ PHP application
ครอบคลุม ChatGPT, Image generation, Embeddings และ Vector Search

---

## ขั้นตอนที่ 2501: OpenAI PHP Client Setup

```bash
composer require openai-php/client
```

```php
<?php
declare(strict_types=1);

// config/ai.php
return [
    'openai' => [
        'api_key' => env('OPENAI_API_KEY'),
        'organization' => env('OPENAI_ORGANIZATION'),
        'default_model' => env('OPENAI_MODEL', 'gpt-4o'),
        'embedding_model' => env('OPENAI_EMBEDDING_MODEL', 'text-embedding-3-small'),
        'max_tokens' => (int)env('OPENAI_MAX_TOKENS', 2000),
        'temperature' => (float)env('OPENAI_TEMPERATURE', 0.7),
    ],
];
```

```php
<?php
declare(strict_types=1);

// app/Providers/AIServiceProvider.php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use OpenAI;

class AIServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(OpenAI\Client::class, function () {
            return OpenAI::factory()
                ->withApiKey(config('ai.openai.api_key'))
                ->withOrganization(config('ai.openai.organization'))
                ->withHttpHeader('OpenAI-Beta', 'assistants=v2')
                ->withHttpClient(new \GuzzleHttp\Client(['timeout' => 60]))
                ->make();
        });
    }
}
```

---

## ขั้นตอนที่ 2502: Chat Completion Service

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

use OpenAI\Client;
use OpenAI\Responses\Chat\CreateResponse;

class ChatService
{
    public function __construct(
        private readonly Client $client
    ) {}

    public function chat(string $message, array $context = [], string $systemPrompt = ''): string
    {
        $messages = [];

        if ($systemPrompt) {
            $messages[] = [
                'role' => 'system',
                'content' => $systemPrompt,
            ];
        }

        // เพิ่ม conversation history
        foreach ($context as $item) {
            $messages[] = $item;
        }

        $messages[] = [
            'role' => 'user',
            'content' => $message,
        ];

        $response = $this->client->chat()->create([
            'model' => config('ai.openai.default_model'),
            'messages' => $messages,
            'max_tokens' => config('ai.openai.max_tokens'),
            'temperature' => config('ai.openai.temperature'),
        ]);

        return $response->choices[0]->message->content ?? '';
    }

    public function chatWithFunctions(string $message, array $functions): array
    {
        $response = $this->client->chat()->create([
            'model' => config('ai.openai.default_model'),
            'messages' => [
                ['role' => 'user', 'content' => $message],
            ],
            'tools' => array_map(fn ($fn) => [
                'type' => 'function',
                'function' => $fn,
            ], $functions),
            'tool_choice' => 'auto',
        ]);

        $choice = $response->choices[0];

        if ($choice->finishReason === 'tool_calls') {
            $toolCalls = [];
            foreach ($choice->message->toolCalls as $toolCall) {
                $toolCalls[] = [
                    'name' => $toolCall->function->name,
                    'arguments' => json_decode($toolCall->function->arguments, true),
                ];
            }
            return ['type' => 'tool_calls', 'calls' => $toolCalls];
        }

        return ['type' => 'message', 'content' => $choice->message->content];
    }

    /**
     * Streaming response
     */
    public function streamChat(string $message, callable $onChunk): void
    {
        $stream = $this->client->chat()->createStreamed([
            'model' => config('ai.openai.default_model'),
            'messages' => [
                ['role' => 'user', 'content' => $message],
            ],
        ]);

        foreach ($stream as $response) {
            $content = $response->choices[0]->delta->content ?? '';
            if ($content) {
                $onChunk($content);
            }
        }
    }
}
```

---

## ขั้นตอนที่ 2503: Content Generation สำหรับ Blog

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

use App\Models\Post;
use OpenAI\Client;

class ContentGenerationService
{
    public function __construct(
        private readonly Client $client
    ) {}

    public function generateBlogPost(string $topic, array $keywords = [], string $tone = 'professional'): array
    {
        $prompt = $this->buildBlogPrompt($topic, $keywords, $tone);

        $response = $this->client->chat()->create([
            'model' => 'gpt-4o',
            'messages' => [
                [
                    'role' => 'system',
                    'content' => 'คุณเป็นนักเขียนบทความมืออาชีพที่เชี่ยวชาญด้านเทคโนโลยี เขียนเนื้อหาที่มีคุณภาพสูง ถูกต้อง และอ่านง่าย',
                ],
                [
                    'role' => 'user',
                    'content' => $prompt,
                ],
            ],
            'response_format' => ['type' => 'json_object'],
            'max_tokens' => 3000,
        ]);

        $content = json_decode($response->choices[0]->message->content, true);

        return [
            'title' => $content['title'] ?? '',
            'excerpt' => $content['excerpt'] ?? '',
            'content' => $content['content'] ?? '',
            'meta_description' => $content['meta_description'] ?? '',
            'suggested_tags' => $content['tags'] ?? [],
            'reading_time' => $this->calculateReadingTime($content['content'] ?? ''),
            'tokens_used' => $response->usage->totalTokens,
        ];
    }

    public function improveContent(string $content, string $improvement): string
    {
        $response = $this->client->chat()->create([
            'model' => 'gpt-4o',
            'messages' => [
                [
                    'role' => 'system',
                    'content' => 'คุณเป็น editor ที่ช่วยปรับปรุงเนื้อหา',
                ],
                [
                    'role' => 'user',
                    'content' => "ปรับปรุงเนื้อหาต่อไปนี้โดย: {$improvement}\n\nเนื้อหา:\n{$content}",
                ],
            ],
        ]);

        return $response->choices[0]->message->content ?? $content;
    }

    public function generateSEOMetadata(string $title, string $content): array
    {
        $response = $this->client->chat()->create([
            'model' => 'gpt-4o',
            'messages' => [
                [
                    'role' => 'user',
                    'content' => "สร้าง SEO metadata สำหรับบทความ:\n\nหัวข้อ: {$title}\n\nเนื้อหา: " . substr($content, 0, 1000),
                ],
            ],
            'response_format' => ['type' => 'json_object'],
        ]);

        return json_decode($response->choices[0]->message->content, true) ?? [];
    }

    private function buildBlogPrompt(string $topic, array $keywords, string $tone): string
    {
        $keywordsStr = empty($keywords) ? '' : "\nKeywords ที่ต้องใช้: " . implode(', ', $keywords);
        return "เขียนบทความบล็อกเกี่ยวกับ: {$topic}{$keywordsStr}\nTone: {$tone}\n\nส่งกลับในรูปแบบ JSON: {title, excerpt, content (markdown), meta_description, tags}";
    }

    private function calculateReadingTime(string $content): int
    {
        $wordCount = str_word_count(strip_tags($content));
        return (int)ceil($wordCount / 200); // 200 words per minute
    }
}
```

---

## ขั้นตอนที่ 2504: Image Generation กับ DALL-E

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

use Illuminate\Support\Facades\Storage;
use OpenAI\Client;

class ImageGenerationService
{
    public function __construct(
        private readonly Client $client
    ) {}

    public function generateImage(
        string $prompt,
        string $size = '1024x1024',
        string $quality = 'standard',
        string $style = 'vivid'
    ): string {
        $response = $this->client->images()->create([
            'model' => 'dall-e-3',
            'prompt' => $prompt,
            'n' => 1,
            'size' => $size,
            'quality' => $quality,
            'style' => $style,
            'response_format' => 'url',
        ]);

        return $response->data[0]->url ?? '';
    }

    public function generateAndStore(string $prompt, string $path): string
    {
        $url = $this->generateImage($prompt);

        if (!$url) {
            throw new \RuntimeException('Failed to generate image');
        }

        // ดาวน์โหลดและเก็บใน storage
        $imageContent = file_get_contents($url);
        $filename = $path . '/' . uniqid('ai_') . '.png';

        Storage::put($filename, $imageContent);

        return Storage::url($filename);
    }

    public function generateVariation(string $imagePath): string
    {
        $response = $this->client->images()->createVariation([
            'image' => fopen(Storage::path($imagePath), 'r'),
            'n' => 1,
            'size' => '1024x1024',
            'response_format' => 'url',
        ]);

        return $response->data[0]->url ?? '';
    }

    public function editImage(string $imagePath, string $maskPath, string $prompt): string
    {
        $response = $this->client->images()->edit([
            'image' => fopen(Storage::path($imagePath), 'r'),
            'mask' => fopen(Storage::path($maskPath), 'r'),
            'prompt' => $prompt,
            'n' => 1,
            'size' => '1024x1024',
        ]);

        return $response->data[0]->url ?? '';
    }
}
```

---

## ขั้นตอนที่ 2505: Embeddings และ Vector Search

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

use Illuminate\Support\Facades\DB;
use OpenAI\Client;

class EmbeddingService
{
    public function __construct(
        private readonly Client $client
    ) {}

    public function embed(string|array $text): array
    {
        $input = is_array($text) ? $text : [$text];

        $response = $this->client->embeddings()->create([
            'model' => config('ai.openai.embedding_model'),
            'input' => $input,
        ]);

        if (is_array($text)) {
            return array_map(
                fn ($item) => $item->embedding,
                $response->embeddings
            );
        }

        return $response->embeddings[0]->embedding;
    }

    public function storeEmbedding(int $documentId, string $text, string $type = 'post'): void
    {
        $embedding = $this->embed($text);

        // เก็บ embedding ใน pgvector หรือ MySQL JSON
        DB::table('document_embeddings')->updateOrInsert(
            ['document_id' => $documentId, 'document_type' => $type],
            [
                'embedding' => json_encode($embedding),
                'text_hash' => hash('sha256', $text),
                'updated_at' => now(),
                'created_at' => now(),
            ]
        );
    }

    public function semanticSearch(string $query, int $limit = 10, string $type = 'post'): array
    {
        $queryEmbedding = $this->embed($query);

        // ใช้ pgvector สำหรับ efficient similarity search
        if (config('database.default') === 'pgsql') {
            return DB::select(
                "SELECT document_id, 1 - (embedding <=> ?) AS similarity
                 FROM document_embeddings
                 WHERE document_type = ?
                 ORDER BY embedding <=> ?
                 LIMIT ?",
                [
                    json_encode($queryEmbedding),
                    $type,
                    json_encode($queryEmbedding),
                    $limit,
                ]
            );
        }

        // Fallback: คำนวณ cosine similarity ใน PHP (ช้ากว่าสำหรับ large datasets)
        $documents = DB::table('document_embeddings')
            ->where('document_type', $type)
            ->select('document_id', 'embedding')
            ->get();

        $results = [];
        foreach ($documents as $doc) {
            $docEmbedding = json_decode($doc->embedding, true);
            $similarity = $this->cosineSimilarity($queryEmbedding, $docEmbedding);
            $results[] = [
                'document_id' => $doc->document_id,
                'similarity' => $similarity,
            ];
        }

        usort($results, fn ($a, $b) => $b['similarity'] <=> $a['similarity']);
        return array_slice($results, 0, $limit);
    }

    private function cosineSimilarity(array $a, array $b): float
    {
        $dotProduct = 0.0;
        $normA = 0.0;
        $normB = 0.0;

        for ($i = 0; $i < count($a); $i++) {
            $dotProduct += $a[$i] * $b[$i];
            $normA += $a[$i] * $a[$i];
            $normB += $b[$i] * $b[$i];
        }

        if ($normA === 0.0 || $normB === 0.0) return 0.0;

        return $dotProduct / (sqrt($normA) * sqrt($normB));
    }
}
```

---

## ขั้นตอนที่ 2506: Prompt Engineering

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

class PromptBuilder
{
    private array $messages = [];
    private array $variables = [];

    public function system(string $prompt): static
    {
        $this->messages[] = [
            'role' => 'system',
            'content' => $this->interpolate($prompt),
        ];
        return $this;
    }

    public function user(string $prompt): static
    {
        $this->messages[] = [
            'role' => 'user',
            'content' => $this->interpolate($prompt),
        ];
        return $this;
    }

    public function assistant(string $response): static
    {
        $this->messages[] = [
            'role' => 'assistant',
            'content' => $response,
        ];
        return $this;
    }

    public function with(array $variables): static
    {
        $this->variables = array_merge($this->variables, $variables);
        return $this;
    }

    public function fewShot(array $examples): static
    {
        foreach ($examples as $example) {
            $this->user($example['input']);
            $this->assistant($example['output']);
        }
        return $this;
    }

    public function chainOfThought(string $task): static
    {
        return $this->user("Let's solve this step by step:\n{$task}\n\nStep 1:");
    }

    public function build(): array
    {
        return $this->messages;
    }

    private function interpolate(string $template): string
    {
        foreach ($this->variables as $key => $value) {
            $template = str_replace("{{$key}}", $value, $template);
        }
        return $template;
    }
}

// ใช้งาน
$messages = (new PromptBuilder())
    ->system('คุณเป็น customer support AI สำหรับร้านค้าออนไลน์ {company_name}')
    ->with(['company_name' => 'TechShop Thailand'])
    ->fewShot([
        ['input' => 'ส่งสินค้านานแค่ไหน?', 'output' => 'ปกติจัดส่งภายใน 2-3 วันทำการ'],
        ['input' => 'รับ COD ไหม?', 'output' => 'รับ COD สำหรับ Bangkok และปริมณฑล'],
    ])
    ->user('ฉันสั่งซื้อไปแล้ว 5 วันแต่ยังไม่ได้รับสินค้า')
    ->build();
```

---

## สรุปบทที่ 88

| ฟีเจอร์ | Model | Use Case |
|--------|-------|---------|
| Chat Completion | gpt-4o | Customer service bot |
| Function Calling | gpt-4o | API integration |
| Image Generation | dall-e-3 | Blog thumbnails |
| Embeddings | text-embedding-3-small | Semantic search |
| Streaming | gpt-4o | Real-time chat |
| Prompt Engineering | - | Better responses |

ถัดไป → Part 89: Laravel AI Tools
