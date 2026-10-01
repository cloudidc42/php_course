# Part 95: Machine Learning กับ PHP
## ขั้นตอนที่ 2711-2740: PHP-ML, Classification, NLP และ Recommendation

Machine Learning ด้วย PHP-ML library
ครอบคลุม Classification, Regression, NLP และ Recommendation Engine

---

## ขั้นตอนที่ 2711: ติดตั้ง PHP-ML

```bash
composer require php-ai/php-ml
```

---

## ขั้นตอนที่ 2712: Classification

```php
<?php
declare(strict_types=1);

namespace App\Services\ML;

use Phpml\Classification\KNearestNeighbors;
use Phpml\Classification\SVC;
use Phpml\Classification\NaiveBayes;
use Phpml\SupportVectorMachine\Kernel;
use Phpml\CrossValidation\StratifiedRandomSplit;
use Phpml\Metric\Accuracy;
use Phpml\Dataset\ArrayDataset;

class SpamClassifier
{
    private NaiveBayes $classifier;
    private TextFeatureExtractor $featureExtractor;
    private bool $trained = false;

    public function __construct()
    {
        $this->classifier = new NaiveBayes();
        $this->featureExtractor = new TextFeatureExtractor();
    }

    public function train(array $texts, array $labels): array
    {
        // แปลงข้อความเป็น features
        $samples = array_map(
            fn ($text) => $this->featureExtractor->extract($text),
            $texts
        );

        // แบ่ง train/test
        $dataset = new ArrayDataset($samples, $labels);
        $split = new StratifiedRandomSplit($dataset, 0.2);

        // Train
        $this->classifier->train($split->getTrainSamples(), $split->getTrainLabels());

        // Evaluate
        $predictions = $this->classifier->predict($split->getTestSamples());
        $accuracy = Accuracy::score($split->getTestLabels(), $predictions);

        $this->trained = true;

        return [
            'accuracy' => round($accuracy * 100, 2),
            'train_size' => count($split->getTrainSamples()),
            'test_size' => count($split->getTestSamples()),
        ];
    }

    public function predict(string $text): array
    {
        if (!$this->trained) {
            $this->loadModel();
        }

        $features = $this->featureExtractor->extract($text);
        $prediction = $this->classifier->predict([$features]);

        return [
            'label' => $prediction[0],
            'is_spam' => $prediction[0] === 'spam',
        ];
    }

    public function saveModel(string $path): void
    {
        $serialized = serialize([
            'classifier' => $this->classifier,
            'feature_extractor' => $this->featureExtractor,
        ]);
        file_put_contents($path, $serialized);
    }

    public function loadModel(string $path = null): void
    {
        $path ??= storage_path('ml/spam_classifier.model');
        $data = unserialize(file_get_contents($path));
        $this->classifier = $data['classifier'];
        $this->featureExtractor = $data['feature_extractor'];
        $this->trained = true;
    }
}

class TextFeatureExtractor
{
    private array $vocabulary = [];

    public function extract(string $text): array
    {
        $tokens = $this->tokenize($text);
        return $this->bagOfWords($tokens);
    }

    private function tokenize(string $text): array
    {
        $text = strtolower($text);
        $text = preg_replace('/[^a-z0-9\s]/', '', $text);
        $words = explode(' ', $text);
        return array_filter($words, fn ($w) => strlen($w) > 2);
    }

    private function bagOfWords(array $tokens): array
    {
        if (empty($this->vocabulary)) {
            // ถ้าไม่มี vocabulary ให้สร้างจาก tokens
            $this->vocabulary = array_unique($tokens);
        }

        $features = array_fill(0, count($this->vocabulary), 0);

        foreach ($tokens as $token) {
            $index = array_search($token, $this->vocabulary);
            if ($index !== false) {
                $features[$index]++;
            }
        }

        return $features;
    }
}
```

---

## ขั้นตอนที่ 2713: Regression Model

```php
<?php
declare(strict_types=1);

namespace App\Services\ML;

use Phpml\Regression\LeastSquares;
use Phpml\Regression\SVR;
use Phpml\Preprocessing\Normalizer;
use Phpml\Math\Statistic\Mean;

class PricePredictionModel
{
    private LeastSquares $regressor;
    private Normalizer $normalizer;

    public function __construct()
    {
        $this->regressor = new LeastSquares();
        $this->normalizer = new Normalizer();
    }

    public function train(array $features, array $prices): array
    {
        // Normalize features
        $this->normalizer->fit($features);
        $normalizedFeatures = $this->normalizer->transform($features);

        // Train model
        $this->regressor->train($normalizedFeatures, $prices);

        // Calculate R² score
        $predictions = $this->regressor->predict($normalizedFeatures);
        $r2 = $this->calculateR2($prices, $predictions);
        $rmse = $this->calculateRMSE($prices, $predictions);

        return [
            'r2_score' => round($r2, 4),
            'rmse' => round($rmse, 2),
            'samples' => count($features),
        ];
    }

    public function predict(array $features): float
    {
        $normalized = $this->normalizer->transform([$features]);
        $prediction = $this->regressor->predict($normalized);
        return $prediction[0];
    }

    private function calculateR2(array $actual, array $predicted): float
    {
        $mean = array_sum($actual) / count($actual);
        $ssTot = array_sum(array_map(fn ($y) => pow($y - $mean, 2), $actual));
        $ssRes = array_sum(array_map(
            fn ($y, $yHat) => pow($y - $yHat, 2),
            $actual, $predicted
        ));

        return 1 - ($ssRes / $ssTot);
    }

    private function calculateRMSE(array $actual, array $predicted): float
    {
        $errors = array_map(fn ($y, $yHat) => pow($y - $yHat, 2), $actual, $predicted);
        return sqrt(array_sum($errors) / count($errors));
    }
}
```

---

## ขั้นตอนที่ 2714: NLP - Text Analysis

```php
<?php
declare(strict_types=1);

namespace App\Services\ML;

use Phpml\FeatureExtraction\TfIdfTransformer;
use Phpml\FeatureExtraction\TokenCountVectorizer;
use Phpml\Tokenization\WhitespaceTokenizer;

class TextAnalyzer
{
    private TokenCountVectorizer $vectorizer;
    private TfIdfTransformer $tfidf;

    public function __construct()
    {
        $this->vectorizer = new TokenCountVectorizer(new WhitespaceTokenizer());
        $this->tfidf = new TfIdfTransformer();
    }

    public function extractKeywords(string $text, int $topN = 10): array
    {
        $sentences = $this->tokenizeSentences($text);
        $samples = [$this->preprocessText($text)];

        $this->vectorizer->fit($samples);
        $counts = $this->vectorizer->transform($samples);

        $this->tfidf->fit($counts);
        $tfidfScores = $this->tfidf->transform($counts);

        $vocabulary = $this->vectorizer->getVocabulary();
        $scores = array_combine($vocabulary, $tfidfScores[0]);

        arsort($scores);
        return array_slice($scores, 0, $topN, true);
    }

    public function sentiment(string $text): array
    {
        $positiveWords = ['ดี', 'เยี่ยม', 'ชอบ', 'สวย', 'คุ้มค่า', 'แนะนำ', 'ประทับใจ', 'good', 'great', 'excellent', 'love'];
        $negativeWords = ['แย่', 'ไม่ดี', 'เสียใจ', 'ผิดหวัง', 'แพง', 'bad', 'terrible', 'awful', 'hate', 'worst'];

        $words = explode(' ', strtolower($text));
        $positiveCount = count(array_intersect($words, $positiveWords));
        $negativeCount = count(array_intersect($words, $negativeWords));

        $score = ($positiveCount - $negativeCount) / max(count($words), 1);

        return [
            'label' => $score > 0.02 ? 'positive' : ($score < -0.02 ? 'negative' : 'neutral'),
            'score' => round($score, 4),
            'positive_words' => $positiveCount,
            'negative_words' => $negativeCount,
        ];
    }

    public function summarize(string $text, int $sentences = 3): string
    {
        $sentenceList = $this->tokenizeSentences($text);

        if (count($sentenceList) <= $sentences) {
            return $text;
        }

        // Score sentences based on word frequency
        $words = str_word_count(strtolower($text), 1);
        $wordFreq = array_count_values($words);

        $sentenceScores = [];
        foreach ($sentenceList as $i => $sentence) {
            $sentenceWords = str_word_count(strtolower($sentence), 1);
            $score = array_sum(array_map(fn ($w) => $wordFreq[$w] ?? 0, $sentenceWords));
            $sentenceScores[$i] = $score / max(count($sentenceWords), 1);
        }

        arsort($sentenceScores);
        $topIndices = array_slice(array_keys($sentenceScores), 0, $sentences);
        sort($topIndices);

        return implode(' ', array_map(fn ($i) => $sentenceList[$i], $topIndices));
    }

    private function tokenizeSentences(string $text): array
    {
        return preg_split('/(?<=[.!?])\s+/', $text, -1, PREG_SPLIT_NO_EMPTY);
    }

    private function preprocessText(string $text): string
    {
        $text = strtolower($text);
        $text = preg_replace('/[^a-z0-9\s]/', ' ', $text);
        return preg_replace('/\s+/', ' ', trim($text));
    }
}
```

---

## ขั้นตอนที่ 2715: Recommendation Engine

```php
<?php
declare(strict_types=1);

namespace App\Services\ML;

use App\Models\User;
use App\Models\Product;
use App\Models\UserProductInteraction;
use Illuminate\Support\Facades\Cache;

class RecommendationEngine
{
    /**
     * Collaborative Filtering - User-based
     */
    public function getUserBasedRecommendations(int $userId, int $limit = 10): array
    {
        $cacheKey = "recommendations:user:{$userId}";
        return Cache::remember($cacheKey, 3600, function () use ($userId, $limit) {
            // ดึง ratings ของ user ทุกคน
            $allRatings = UserProductInteraction::select('user_id', 'product_id', 'rating')
                ->where('rating', '>', 0)
                ->get()
                ->groupBy('user_id')
                ->map(fn ($items) => $items->pluck('rating', 'product_id')->toArray())
                ->toArray();

            if (!isset($allRatings[$userId])) return [];

            $userRatings = $allRatings[$userId];

            // หา similar users
            $similarities = [];
            foreach ($allRatings as $otherUserId => $otherRatings) {
                if ($otherUserId === $userId) continue;
                $similarities[$otherUserId] = $this->cosineSimilarity($userRatings, $otherRatings);
            }

            arsort($similarities);
            $topUsers = array_slice($similarities, 0, 20, true);

            // หา products ที่ similar users ชอบแต่ user ยังไม่ได้ดู
            $recommendations = [];
            $userProducts = array_keys($userRatings);

            foreach ($topUsers as $similarUserId => $similarity) {
                $similarRatings = $allRatings[$similarUserId];
                foreach ($similarRatings as $productId => $rating) {
                    if (!in_array($productId, $userProducts) && $rating >= 4) {
                        $recommendations[$productId] = ($recommendations[$productId] ?? 0) + ($similarity * $rating);
                    }
                }
            }

            arsort($recommendations);
            return array_slice(array_keys($recommendations), 0, $limit);
        });
    }

    /**
     * Content-based Filtering
     */
    public function getContentBasedRecommendations(int $productId, int $limit = 10): array
    {
        $product = Product::find($productId);
        if (!$product) return [];

        $allProducts = Product::where('id', '!=', $productId)
            ->where('is_active', true)
            ->select(['id', 'name', 'description', 'category_id', 'tags'])
            ->get();

        $similarities = [];
        foreach ($allProducts as $otherProduct) {
            $similarity = $this->calculateProductSimilarity($product, $otherProduct);
            $similarities[$otherProduct->id] = $similarity;
        }

        arsort($similarities);
        return array_slice(array_keys($similarities), 0, $limit);
    }

    private function cosineSimilarity(array $a, array $b): float
    {
        $allKeys = array_unique(array_merge(array_keys($a), array_keys($b)));

        $dotProduct = 0.0;
        $normA = 0.0;
        $normB = 0.0;

        foreach ($allKeys as $key) {
            $valA = $a[$key] ?? 0;
            $valB = $b[$key] ?? 0;
            $dotProduct += $valA * $valB;
            $normA += $valA * $valA;
            $normB += $valB * $valB;
        }

        if ($normA === 0.0 || $normB === 0.0) return 0.0;

        return $dotProduct / (sqrt($normA) * sqrt($normB));
    }

    private function calculateProductSimilarity(Product $a, Product $b): float
    {
        $score = 0.0;

        // Category match
        if ($a->category_id === $b->category_id) $score += 0.4;

        // Tags similarity
        $tagsA = json_decode($a->tags ?? '[]', true);
        $tagsB = json_decode($b->tags ?? '[]', true);

        if (!empty($tagsA) && !empty($tagsB)) {
            $intersection = count(array_intersect($tagsA, $tagsB));
            $union = count(array_unique(array_merge($tagsA, $tagsB)));
            $jaccardSim = $union > 0 ? $intersection / $union : 0;
            $score += $jaccardSim * 0.6;
        }

        return $score;
    }
}
```

---

## สรุปบทที่ 95

| หัวข้อ | Algorithm | Use Case |
|--------|----------|---------|
| Classification | Naive Bayes, SVC, KNN | Spam filter, Sentiment |
| Regression | Least Squares, SVR | Price prediction |
| NLP | TF-IDF, Bag of Words | Keyword extraction |
| Collaborative Filtering | Cosine Similarity | User-based recommendations |
| Content-based | Jaccard, Feature matching | Product recommendations |
| Anomaly Detection | Z-Score | Fraud detection |

ถัดไป → Part 96: Mobile App Backend
