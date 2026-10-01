# Part 80: PHP Data Structures & Algorithms
## ขั้นตอนที่ 2261-2290: โครงสร้างข้อมูลและ Algorithms ใน PHP

SPL Data Structures, การ implement โครงสร้างข้อมูลเอง,
Graph Algorithms และการวิเคราะห์ความซับซ้อน

---

## ขั้นตอนที่ 2261: SPL Data Structures

```php
<?php
declare(strict_types=1);

// SplStack - Last In First Out (LIFO)
$stack = new SplStack();
$stack->push('PHP');
$stack->push('Laravel');
$stack->push('Vue');

echo $stack->top();  // Vue
echo $stack->pop();  // Vue
echo $stack->count(); // 2

// SplQueue - First In First Out (FIFO)
$queue = new SplQueue();
$queue->enqueue('Task 1');
$queue->enqueue('Task 2');
$queue->enqueue('Task 3');

echo $queue->dequeue(); // Task 1
echo $queue->count();   // 2

// SplMinHeap - Min Heap (เรียงจากน้อยไปมาก)
$minHeap = new SplMinHeap();
$minHeap->insert(5);
$minHeap->insert(1);
$minHeap->insert(3);

echo $minHeap->extract(); // 1
echo $minHeap->extract(); // 3

// SplMaxHeap - Max Heap (เรียงจากมากไปน้อย)
$maxHeap = new SplMaxHeap();
$maxHeap->insert(5);
$maxHeap->insert(1);
$maxHeap->insert(3);

echo $maxHeap->extract(); // 5

// SplDoublyLinkedList
$list = new SplDoublyLinkedList();
$list->push('A');
$list->push('B');
$list->push('C');
$list->unshift('Z'); // เพิ่มหัว

foreach ($list as $item) {
    echo $item . ' '; // Z A B C
}

// SplFixedArray - Fixed size, เร็วกว่า array ปกติ
$fixed = new SplFixedArray(5);
$fixed[0] = 'a';
$fixed[1] = 'b';
$fixed[2] = 'c';
echo $fixed->getSize(); // 5

// SplPriorityQueue
$pq = new SplPriorityQueue();
$pq->insert('Low priority task', 1);
$pq->insert('High priority task', 10);
$pq->insert('Medium priority task', 5);

echo $pq->extract(); // High priority task
```

---

## ขั้นตอนที่ 2262: Linked List Implementation

```php
<?php
declare(strict_types=1);

class Node
{
    public ?Node $next = null;

    public function __construct(
        public mixed $data
    ) {}
}

class LinkedList
{
    private ?Node $head = null;
    private int $size = 0;

    public function prepend(mixed $data): void
    {
        $node = new Node($data);
        $node->next = $this->head;
        $this->head = $node;
        $this->size++;
    }

    public function append(mixed $data): void
    {
        $node = new Node($data);

        if ($this->head === null) {
            $this->head = $node;
            $this->size++;
            return;
        }

        $current = $this->head;
        while ($current->next !== null) {
            $current = $current->next;
        }
        $current->next = $node;
        $this->size++;
    }

    public function insert(mixed $data, int $position): void
    {
        if ($position === 0) {
            $this->prepend($data);
            return;
        }

        $node = new Node($data);
        $current = $this->head;
        $index = 0;

        while ($current !== null && $index < $position - 1) {
            $current = $current->next;
            $index++;
        }

        if ($current === null) {
            throw new \OutOfRangeException("Position {$position} out of range");
        }

        $node->next = $current->next;
        $current->next = $node;
        $this->size++;
    }

    public function remove(mixed $data): bool
    {
        if ($this->head === null) return false;

        if ($this->head->data === $data) {
            $this->head = $this->head->next;
            $this->size--;
            return true;
        }

        $current = $this->head;
        while ($current->next !== null) {
            if ($current->next->data === $data) {
                $current->next = $current->next->next;
                $this->size--;
                return true;
            }
            $current = $current->next;
        }

        return false;
    }

    public function find(mixed $data): bool
    {
        $current = $this->head;
        while ($current !== null) {
            if ($current->data === $data) return true;
            $current = $current->next;
        }
        return false;
    }

    public function reverse(): void
    {
        $prev = null;
        $current = $this->head;

        while ($current !== null) {
            $next = $current->next;
            $current->next = $prev;
            $prev = $current;
            $current = $next;
        }

        $this->head = $prev;
    }

    public function toArray(): array
    {
        $result = [];
        $current = $this->head;
        while ($current !== null) {
            $result[] = $current->data;
            $current = $current->next;
        }
        return $result;
    }

    public function size(): int
    {
        return $this->size;
    }
}

// ทดสอบ
$list = new LinkedList();
$list->append(1);
$list->append(2);
$list->append(3);
$list->prepend(0);

print_r($list->toArray()); // [0, 1, 2, 3]

$list->reverse();
print_r($list->toArray()); // [3, 2, 1, 0]

$list->remove(2);
print_r($list->toArray()); // [3, 1, 0]
```

---

## ขั้นตอนที่ 2263: Binary Search Tree

```php
<?php
declare(strict_types=1);

class BSTNode
{
    public ?BSTNode $left = null;
    public ?BSTNode $right = null;

    public function __construct(public int $value) {}
}

class BinarySearchTree
{
    private ?BSTNode $root = null;

    public function insert(int $value): void
    {
        $this->root = $this->insertNode($this->root, $value);
    }

    private function insertNode(?BSTNode $node, int $value): BSTNode
    {
        if ($node === null) {
            return new BSTNode($value);
        }

        if ($value < $node->value) {
            $node->left = $this->insertNode($node->left, $value);
        } elseif ($value > $node->value) {
            $node->right = $this->insertNode($node->right, $value);
        }

        return $node;
    }

    public function search(int $value): bool
    {
        return $this->searchNode($this->root, $value);
    }

    private function searchNode(?BSTNode $node, int $value): bool
    {
        if ($node === null) return false;
        if ($node->value === $value) return true;

        if ($value < $node->value) {
            return $this->searchNode($node->left, $value);
        }
        return $this->searchNode($node->right, $value);
    }

    public function delete(int $value): void
    {
        $this->root = $this->deleteNode($this->root, $value);
    }

    private function deleteNode(?BSTNode $node, int $value): ?BSTNode
    {
        if ($node === null) return null;

        if ($value < $node->value) {
            $node->left = $this->deleteNode($node->left, $value);
        } elseif ($value > $node->value) {
            $node->right = $this->deleteNode($node->right, $value);
        } else {
            // Node found
            if ($node->left === null) return $node->right;
            if ($node->right === null) return $node->left;

            // Node has two children - find inorder successor
            $successor = $this->findMin($node->right);
            $node->value = $successor->value;
            $node->right = $this->deleteNode($node->right, $successor->value);
        }

        return $node;
    }

    private function findMin(BSTNode $node): BSTNode
    {
        while ($node->left !== null) {
            $node = $node->left;
        }
        return $node;
    }

    // In-order traversal (sorted order)
    public function inOrder(): array
    {
        $result = [];
        $this->inOrderTraversal($this->root, $result);
        return $result;
    }

    private function inOrderTraversal(?BSTNode $node, array &$result): void
    {
        if ($node === null) return;
        $this->inOrderTraversal($node->left, $result);
        $result[] = $node->value;
        $this->inOrderTraversal($node->right, $result);
    }

    public function height(): int
    {
        return $this->calculateHeight($this->root);
    }

    private function calculateHeight(?BSTNode $node): int
    {
        if ($node === null) return 0;
        return 1 + max(
            $this->calculateHeight($node->left),
            $this->calculateHeight($node->right)
        );
    }
}

$bst = new BinarySearchTree();
foreach ([5, 3, 7, 1, 4, 6, 8] as $v) {
    $bst->insert($v);
}
print_r($bst->inOrder()); // [1, 3, 4, 5, 6, 7, 8]
echo $bst->height();      // 3
echo $bst->search(4) ? "found" : "not found"; // found
```

---

## ขั้นตอนที่ 2264: Hash Map Implementation

```php
<?php
declare(strict_types=1);

class HashMap
{
    private array $buckets;
    private int $capacity;
    private int $size = 0;
    private const LOAD_FACTOR = 0.75;

    public function __construct(int $capacity = 16)
    {
        $this->capacity = $capacity;
        $this->buckets = array_fill(0, $capacity, []);
    }

    private function hash(string $key): int
    {
        $hash = 0;
        for ($i = 0; $i < strlen($key); $i++) {
            $hash = ($hash * 31 + ord($key[$i])) % $this->capacity;
        }
        return $hash;
    }

    public function set(string $key, mixed $value): void
    {
        $index = $this->hash($key);

        foreach ($this->buckets[$index] as &$entry) {
            if ($entry[0] === $key) {
                $entry[1] = $value;
                return;
            }
        }

        $this->buckets[$index][] = [$key, $value];
        $this->size++;

        // Resize ถ้า load factor เกิน
        if ($this->size / $this->capacity > self::LOAD_FACTOR) {
            $this->resize();
        }
    }

    public function get(string $key): mixed
    {
        $index = $this->hash($key);

        foreach ($this->buckets[$index] as $entry) {
            if ($entry[0] === $key) {
                return $entry[1];
            }
        }

        return null;
    }

    public function has(string $key): bool
    {
        return $this->get($key) !== null;
    }

    public function delete(string $key): bool
    {
        $index = $this->hash($key);

        foreach ($this->buckets[$index] as $i => $entry) {
            if ($entry[0] === $key) {
                unset($this->buckets[$index][$i]);
                $this->buckets[$index] = array_values($this->buckets[$index]);
                $this->size--;
                return true;
            }
        }

        return false;
    }

    private function resize(): void
    {
        $newCapacity = $this->capacity * 2;
        $newBuckets = array_fill(0, $newCapacity, []);
        $oldCapacity = $this->capacity;

        $this->capacity = $newCapacity;

        foreach ($this->buckets as $bucket) {
            foreach ($bucket as $entry) {
                $newIndex = $this->hash($entry[0]);
                $newBuckets[$newIndex][] = $entry;
            }
        }

        $this->buckets = $newBuckets;
    }

    public function size(): int
    {
        return $this->size;
    }
}

$map = new HashMap();
$map->set('name', 'John');
$map->set('age', 30);
$map->set('city', 'Bangkok');

echo $map->get('name'); // John
echo $map->size();       // 3
$map->delete('age');
echo $map->size();       // 2
```

---

## ขั้นตอนที่ 2265: Graph Algorithms

```php
<?php
declare(strict_types=1);

class Graph
{
    private array $adjacencyList = [];
    private bool $directed;

    public function __construct(bool $directed = false)
    {
        $this->directed = $directed;
    }

    public function addVertex(string $vertex): void
    {
        if (! isset($this->adjacencyList[$vertex])) {
            $this->adjacencyList[$vertex] = [];
        }
    }

    public function addEdge(string $from, string $to, int $weight = 1): void
    {
        $this->addVertex($from);
        $this->addVertex($to);

        $this->adjacencyList[$from][$to] = $weight;

        if (! $this->directed) {
            $this->adjacencyList[$to][$from] = $weight;
        }
    }

    // Breadth-First Search
    public function bfs(string $start): array
    {
        $visited = [$start => true];
        $queue = [$start];
        $result = [];

        while (! empty($queue)) {
            $vertex = array_shift($queue);
            $result[] = $vertex;

            foreach ($this->adjacencyList[$vertex] as $neighbor => $weight) {
                if (! isset($visited[$neighbor])) {
                    $visited[$neighbor] = true;
                    $queue[] = $neighbor;
                }
            }
        }

        return $result;
    }

    // Depth-First Search
    public function dfs(string $start): array
    {
        $visited = [];
        $result = [];
        $this->dfsHelper($start, $visited, $result);
        return $result;
    }

    private function dfsHelper(string $vertex, array &$visited, array &$result): void
    {
        $visited[$vertex] = true;
        $result[] = $vertex;

        foreach ($this->adjacencyList[$vertex] as $neighbor => $weight) {
            if (! isset($visited[$neighbor])) {
                $this->dfsHelper($neighbor, $visited, $result);
            }
        }
    }

    // Dijkstra's Shortest Path
    public function dijkstra(string $start): array
    {
        $distances = [];
        $visited = [];
        $previous = [];

        foreach (array_keys($this->adjacencyList) as $vertex) {
            $distances[$vertex] = PHP_INT_MAX;
            $previous[$vertex] = null;
        }
        $distances[$start] = 0;

        $heap = new SplMinHeap();
        $heap->insert([0, $start]);

        while (! $heap->isEmpty()) {
            [$dist, $current] = $heap->extract();

            if (isset($visited[$current])) continue;
            $visited[$current] = true;

            foreach ($this->adjacencyList[$current] as $neighbor => $weight) {
                $newDist = $distances[$current] + $weight;

                if ($newDist < $distances[$neighbor]) {
                    $distances[$neighbor] = $newDist;
                    $previous[$neighbor] = $current;
                    $heap->insert([$newDist, $neighbor]);
                }
            }
        }

        return ['distances' => $distances, 'previous' => $previous];
    }

    public function shortestPath(string $start, string $end): array
    {
        $result = $this->dijkstra($start);
        $path = [];
        $current = $end;

        while ($current !== null) {
            array_unshift($path, $current);
            $current = $result['previous'][$current];
        }

        return [
            'path' => $path,
            'distance' => $result['distances'][$end],
        ];
    }

    // Detect cycle
    public function hasCycle(): bool
    {
        $visited = [];
        $recStack = [];

        foreach (array_keys($this->adjacencyList) as $vertex) {
            if (! isset($visited[$vertex])) {
                if ($this->detectCycle($vertex, $visited, $recStack)) {
                    return true;
                }
            }
        }

        return false;
    }

    private function detectCycle(string $vertex, array &$visited, array &$recStack): bool
    {
        $visited[$vertex] = true;
        $recStack[$vertex] = true;

        foreach ($this->adjacencyList[$vertex] as $neighbor => $weight) {
            if (! isset($visited[$neighbor])) {
                if ($this->detectCycle($neighbor, $visited, $recStack)) {
                    return true;
                }
            } elseif (isset($recStack[$neighbor])) {
                return true;
            }
        }

        unset($recStack[$vertex]);
        return false;
    }
}

// ทดสอบ Graph
$g = new Graph(false);
$g->addEdge('A', 'B', 4);
$g->addEdge('A', 'C', 2);
$g->addEdge('B', 'D', 3);
$g->addEdge('C', 'D', 1);
$g->addEdge('D', 'E', 5);

echo "BFS: " . implode(' -> ', $g->bfs('A')) . "\n";

$path = $g->shortestPath('A', 'E');
echo "Shortest path A->E: " . implode(' -> ', $path['path']) . "\n";
echo "Distance: " . $path['distance'] . "\n";
```

---

## ขั้นตอนที่ 2266: Sorting Algorithms

```php
<?php
declare(strict_types=1);

class SortingAlgorithms
{
    // Quick Sort O(n log n) average
    public static function quickSort(array $arr): array
    {
        if (count($arr) <= 1) return $arr;

        $pivot = $arr[0];
        $left = $right = [];

        for ($i = 1; $i < count($arr); $i++) {
            if ($arr[$i] <= $pivot) {
                $left[] = $arr[$i];
            } else {
                $right[] = $arr[$i];
            }
        }

        return array_merge(
            self::quickSort($left),
            [$pivot],
            self::quickSort($right)
        );
    }

    // Merge Sort O(n log n)
    public static function mergeSort(array $arr): array
    {
        $n = count($arr);
        if ($n <= 1) return $arr;

        $mid = intdiv($n, 2);
        $left = self::mergeSort(array_slice($arr, 0, $mid));
        $right = self::mergeSort(array_slice($arr, $mid));

        return self::merge($left, $right);
    }

    private static function merge(array $left, array $right): array
    {
        $result = [];
        $i = $j = 0;

        while ($i < count($left) && $j < count($right)) {
            if ($left[$i] <= $right[$j]) {
                $result[] = $left[$i++];
            } else {
                $result[] = $right[$j++];
            }
        }

        return array_merge($result, array_slice($left, $i), array_slice($right, $j));
    }

    // Heap Sort O(n log n)
    public static function heapSort(array $arr): array
    {
        $n = count($arr);

        // Build max heap
        for ($i = intdiv($n, 2) - 1; $i >= 0; $i--) {
            self::heapify($arr, $n, $i);
        }

        // Extract elements
        for ($i = $n - 1; $i > 0; $i--) {
            [$arr[0], $arr[$i]] = [$arr[$i], $arr[0]];
            self::heapify($arr, $i, 0);
        }

        return $arr;
    }

    private static function heapify(array &$arr, int $n, int $i): void
    {
        $largest = $i;
        $left = 2 * $i + 1;
        $right = 2 * $i + 2;

        if ($left < $n && $arr[$left] > $arr[$largest]) {
            $largest = $left;
        }

        if ($right < $n && $arr[$right] > $arr[$largest]) {
            $largest = $right;
        }

        if ($largest !== $i) {
            [$arr[$i], $arr[$largest]] = [$arr[$largest], $arr[$i]];
            self::heapify($arr, $n, $largest);
        }
    }
}

$arr = [64, 34, 25, 12, 22, 11, 90];
print_r(SortingAlgorithms::quickSort($arr));
print_r(SortingAlgorithms::mergeSort($arr));
```

---

## สรุปบทที่ 80

| โครงสร้างข้อมูล | Complexity | Use Case |
|----------------|-----------|---------|
| SplStack | O(1) push/pop | DFS, undo/redo |
| SplQueue | O(1) enqueue/dequeue | BFS, job queue |
| SplHeap | O(log n) insert | Priority queue |
| BST | O(log n) average | Sorted data |
| HashMap | O(1) average | Key-value store |
| Graph (Dijkstra) | O(V²) | Shortest path |

ถัดไป → Part 81: Spatie Laravel Packages
