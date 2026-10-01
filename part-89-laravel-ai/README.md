# Part 89: Laravel AI Tools
## ขั้นตอนที่ 2531-2560: RAG, AI Chatbot และ Semantic Search

Laravel AI integration ขั้นสูง ครอบคลุม RAG Pattern,
AI Chatbot, Semantic Search และ AI-powered Features

---

## ขั้นตอนที่ 2531: RAG (Retrieval Augmented Generation)

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

use App\Models\KnowledgeBase;
use OpenAI\Client;

class RAGService
{
    private const TOP_K = 5;
    private const SIMILARITY_THRESHOLD = 0.75;

    public function __construct(
        private readonly Client $openai,
        private readonly EmbeddingService $embeddings,
    ) {}

    public function query(string $question, array $context = []): array
    {
        // 1. สร้าง embedding สำหรับคำถาม
        $questionEmbedding = $this->embeddings->embed($question);

        // 2. ค้นหาเอกสารที่เกี่ยวข้อง
        $relevantDocs = $this->retrieveRelevantDocuments($questionEmbedding);

        if (empty($relevantDocs)) {
            return [
                'answer' => 'ขออภัย ฉันไม่มีข้อมูลเพียงพอในการตอบคำถามนี้',
                'sources' => [],
                'confidence' => 0.0,
            ];
        }

        // 3. สร้าง context จากเอกสาร
        $contextText = $this->buildContextFromDocs($relevantDocs);

        // 4. Generate คำตอบด้วย LLM
        $response = $this->generateAnswer($question, $contextText, $context);

        return [
            'answer' => $response,
            'sources' => array_map(fn ($doc) => [
                'id' => $doc->id,
                'title' => $doc->title,
                'similarity' => $doc->similarity,
                'url' => $doc->url ?? null,
            ], $relevantDocs),
            'confidence' => $relevantDocs[0]->similarity ?? 0.0,
        ];
    }

    private function retrieveRelevantDocuments(array $embedding): array
    {
        // ใช้ pgvector หรือ calculate manually
        $docs = KnowledgeBase::selectRaw(
            '*, (embedding <=> ?) as distance, 1 - (embedding <=> ?) as similarity',
            [json_encode($embedding), json_encode($embedding)]
        )
        ->where('is_active', true)
        ->orderBy('distance')
        ->limit(self::TOP_K)
        ->get()
        ->filter(fn ($doc) => $doc->similarity >= self::SIMILARITY_THRESHOLD)
        ->values()
        ->toArray();

        return $docs;
    }

    private function buildContextFromDocs(array $docs): string
    {
        $contexts = [];
        foreach ($docs as $doc) {
            $contexts[] = "เอกสาร: {$doc['title']}\n{$doc['content']}";
        }
        return implode("\n\n---\n\n", $contexts);
    }

    private function generateAnswer(string $question, string $context, array $history): string
    {
        $messages = [
            [
                'role' => 'system',
                'content' => "คุณเป็น AI assistant ที่ตอบคำถามโดยอ้างอิงจากเอกสารที่ให้มาเท่านั้น
                หากข้อมูลในเอกสารไม่เพียงพอ ให้บอกว่าไม่ทราบ
                ตอบเป็นภาษาเดียวกับคำถาม",
            ],
            [
                'role' => 'user',
                'content' => "เอกสารอ้างอิง:\n{$context}\n\nคำถาม: {$question}",
            ],
        ];

        // เพิ่ม conversation history
        foreach (array_slice($history, -6) as $item) {
            array_splice($messages, -1, 0, [$item]);
        }

        $response = $this->openai->chat()->create([
            'model' => 'gpt-4o',
            'messages' => $messages,
            'temperature' => 0.3, // ต้องการความแม่นยำ
            'max_tokens' => 1000,
        ]);

        return $response->choices[0]->message->content ?? '';
    }

    public function ingest(string $title, string $content, array $metadata = []): KnowledgeBase
    {
        // แบ่งเนื้อหาเป็น chunks
        $chunks = $this->splitIntoChunks($content);

        $lastDoc = null;
        foreach ($chunks as $i => $chunk) {
            $embedding = $this->embeddings->embed($chunk);

            $lastDoc = KnowledgeBase::updateOrCreate(
                ['title' => $title, 'chunk_index' => $i],
                [
                    'content' => $chunk,
                    'embedding' => $embedding,
                    'metadata' => array_merge($metadata, ['chunk_index' => $i, 'total_chunks' => count($chunks)]),
                    'is_active' => true,
                ]
            );
        }

        return $lastDoc;
    }

    private function splitIntoChunks(string $text, int $chunkSize = 500, int $overlap = 50): array
    {
        $words = explode(' ', $text);
        $chunks = [];
        $i = 0;

        while ($i < count($words)) {
            $chunk = implode(' ', array_slice($words, $i, $chunkSize));
            $chunks[] = $chunk;
            $i += ($chunkSize - $overlap);
        }

        return array_filter($chunks);
    }
}
```

---

## ขั้นตอนที่ 2532: AI Chatbot กับ Laravel

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Models\ChatSession;
use App\Services\AI\RAGService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Http\Response;

class ChatbotController extends \App\Http\Controllers\Controller
{
    public function __construct(
        private readonly RAGService $ragService
    ) {}

    public function createSession(Request $request): JsonResponse
    {
        $session = ChatSession::create([
            'user_id' => $request->user()?->id,
            'session_id' => \Str::uuid(),
            'metadata' => [
                'user_agent' => $request->userAgent(),
                'ip' => $request->ip(),
            ],
        ]);

        return response()->json([
            'session_id' => $session->session_id,
            'greeting' => 'สวัสดีครับ! ฉันเป็น AI assistant พร้อมช่วยเหลือคุณ',
        ]);
    }

    public function chat(Request $request): JsonResponse
    {
        $request->validate([
            'session_id' => ['required', 'string'],
            'message' => ['required', 'string', 'max:2000'],
        ]);

        $session = ChatSession::where('session_id', $request->session_id)->firstOrFail();

        // บันทึก user message
        $session->messages()->create([
            'role' => 'user',
            'content' => $request->message,
        ]);

        // ดึง conversation history
        $history = $session->messages()
            ->latest()
            ->limit(10)
            ->get()
            ->reverse()
            ->map(fn ($msg) => ['role' => $msg->role, 'content' => $msg->content])
            ->toArray();

        // ถาม RAG
        $result = $this->ragService->query($request->message, $history);

        // บันทึก assistant response
        $session->messages()->create([
            'role' => 'assistant',
            'content' => $result['answer'],
            'metadata' => ['sources' => $result['sources']],
        ]);

        return response()->json([
            'answer' => $result['answer'],
            'sources' => $result['sources'],
            'confidence' => round($result['confidence'], 2),
        ]);
    }

    public function stream(Request $request): \Symfony\Component\HttpFoundation\StreamedResponse
    {
        $request->validate([
            'session_id' => ['required', 'string'],
            'message' => ['required', 'string'],
        ]);

        return response()->stream(function () use ($request) {
            header('Content-Type: text/event-stream');
            header('Cache-Control: no-cache');

            // ส่ง SSE events
            $session = ChatSession::where('session_id', $request->session_id)->firstOrFail();

            $fullResponse = '';

            $this->ragService->streamQuery(
                $request->message,
                function (string $chunk) use (&$fullResponse) {
                    $fullResponse .= $chunk;
                    echo "data: " . json_encode(['content' => $chunk]) . "\n\n";
                    ob_flush();
                    flush();
                }
            );

            // บันทึก response หลังจาก stream จบ
            $session->messages()->create([
                'role' => 'assistant',
                'content' => $fullResponse,
            ]);

            echo "data: " . json_encode(['done' => true]) . "\n\n";
        });
    }
}
```

---

## ขั้นตอนที่ 2533: AI-powered Code Review

```php
<?php
declare(strict_types=1);

namespace App\Services\AI;

use OpenAI\Client;

class CodeReviewService
{
    public function __construct(
        private readonly Client $openai
    ) {}

    public function reviewCode(string $code, string $language = 'php'): array
    {
        $response = $this->openai->chat()->create([
            'model' => 'gpt-4o',
            'messages' => [
                [
                    'role' => 'system',
                    'content' => "คุณเป็น senior software engineer ที่เชี่ยวชาญ {$language}
                    ทำ code review และให้คำแนะนำในการปรับปรุง
                    ตรวจหา: security vulnerabilities, performance issues, bugs, code smells
                    ส่งผลลัพธ์เป็น JSON",
                ],
                [
                    'role' => 'user',
                    'content' => "Review โค้ดนี้:\n```{$language}\n{$code}\n```",
                ],
            ],
            'response_format' => ['type' => 'json_object'],
            'temperature' => 0.2,
        ]);

        return json_decode($response->choices[0]->message->content, true) ?? [];
    }

    public function generateTests(string $code, string $framework = 'PHPUnit'): string
    {
        $response = $this->openai->chat()->create([
            'model' => 'gpt-4o',
            'messages' => [
                [
                    'role' => 'system',
                    'content' => "คุณเป็น PHP developer ที่เชี่ยวชาญการเขียน unit tests ด้วย {$framework}
                    เขียน tests ที่ครอบคลุม edge cases ทั้งหมด",
                ],
                [
                    'role' => 'user',
                    'content' => "เขียน unit tests สำหรับโค้ดนี้:\n```php\n{$code}\n```",
                ],
            ],
        ]);

        return $response->choices[0]->message->content ?? '';
    }

    public function explainCode(string $code): string
    {
        $response = $this->openai->chat()->create([
            'model' => 'gpt-4o',
            'messages' => [
                [
                    'role' => 'user',
                    'content' => "อธิบายโค้ดนี้เป็นภาษาไทยแบบเข้าใจง่าย:\n```php\n{$code}\n```",
                ],
            ],
        ]);

        return $response->choices[0]->message->content ?? '';
    }
}
```

---

## ขั้นตอนที่ 2534: Knowledge Base Management

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Admin;

use App\Services\AI\RAGService;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class KnowledgeBaseController extends \App\Http\Controllers\Controller
{
    public function __construct(
        private readonly RAGService $ragService
    ) {}

    public function ingest(Request $request): JsonResponse
    {
        $request->validate([
            'title' => ['required', 'string', 'max:500'],
            'content' => ['required', 'string'],
            'source_url' => ['nullable', 'url'],
            'category' => ['nullable', 'string'],
        ]);

        $doc = $this->ragService->ingest(
            title: $request->title,
            content: $request->content,
            metadata: [
                'source_url' => $request->source_url,
                'category' => $request->category,
                'ingested_at' => now()->toISOString(),
            ]
        );

        return response()->json([
            'message' => 'Document ingested successfully',
            'document_id' => $doc->id,
        ], 201);
    }

    public function ingestFromUrl(Request $request): JsonResponse
    {
        $request->validate([
            'url' => ['required', 'url'],
            'title' => ['nullable', 'string'],
        ]);

        $content = $this->scrapeWebpage($request->url);
        $title = $request->title ?? parse_url($request->url, PHP_URL_HOST);

        $doc = $this->ragService->ingest(
            title: $title,
            content: $content,
            metadata: ['source_url' => $request->url]
        );

        return response()->json([
            'message' => 'URL content ingested successfully',
            'document_id' => $doc->id,
            'characters_indexed' => strlen($content),
        ]);
    }

    private function scrapeWebpage(string $url): string
    {
        $response = \Illuminate\Support\Facades\Http::get($url);
        $html = $response->body();

        // แปลง HTML เป็น plain text
        $text = strip_tags($html);
        $text = preg_replace('/\s+/', ' ', $text);
        return trim($text);
    }
}
```

---

## สรุปบทที่ 89

| ฟีเจอร์ | เทคนิค | Model |
|--------|--------|-------|
| RAG | Retrieval + Generation | gpt-4o + embeddings |
| Chatbot | Conversation history | gpt-4o |
| Streaming | SSE + Response stream | gpt-4o |
| Code Review | System prompt | gpt-4o |
| Semantic Search | Cosine similarity | text-embedding-3-small |
| Content Generation | JSON response format | gpt-4o |

ถัดไป → Part 90: Performance Testing
