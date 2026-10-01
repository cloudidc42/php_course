# Part 93: Blockchain กับ PHP
## ขั้นตอนที่ 2651-2680: Blockchain, Hash Functions, Smart Contracts

การสร้าง Blockchain ด้วย PHP, Ethereum integration,
Web3 และ NFT metadata APIs

---

## ขั้นตอนที่ 2651: Blockchain พื้นฐาน

```php
<?php
declare(strict_types=1);

namespace App\Blockchain;

class Block
{
    public readonly string $hash;

    public function __construct(
        public readonly int $index,
        public readonly string $previousHash,
        public readonly string $data,
        public readonly int $timestamp,
        public readonly int $nonce = 0,
    ) {
        $this->hash = $this->calculateHash();
    }

    public function calculateHash(): string
    {
        return hash('sha256',
            $this->index .
            $this->previousHash .
            $this->data .
            $this->timestamp .
            $this->nonce
        );
    }

    public function mine(int $difficulty): static
    {
        $target = str_repeat('0', $difficulty);
        $nonce = 0;

        do {
            $block = new static(
                $this->index,
                $this->previousHash,
                $this->data,
                $this->timestamp,
                $nonce
            );
            $nonce++;
        } while (substr($block->hash, 0, $difficulty) !== $target);

        return $block;
    }

    public function isValid(): bool
    {
        return $this->hash === $this->calculateHash();
    }
}

class Blockchain
{
    private array $chain = [];
    private int $difficulty;

    public function __construct(int $difficulty = 4)
    {
        $this->difficulty = $difficulty;
        $this->chain[] = $this->createGenesisBlock();
    }

    private function createGenesisBlock(): Block
    {
        $genesis = new Block(0, '0000000000000000', 'Genesis Block', time());
        return $genesis->mine($this->difficulty);
    }

    public function addBlock(string $data): Block
    {
        $previousBlock = $this->getLatestBlock();
        $newBlock = new Block(
            count($this->chain),
            $previousBlock->hash,
            $data,
            time(),
        );

        $minedBlock = $newBlock->mine($this->difficulty);
        $this->chain[] = $minedBlock;

        return $minedBlock;
    }

    public function getLatestBlock(): Block
    {
        return end($this->chain);
    }

    public function isValid(): bool
    {
        for ($i = 1; $i < count($this->chain); $i++) {
            $current = $this->chain[$i];
            $previous = $this->chain[$i - 1];

            if (!$current->isValid()) {
                return false;
            }

            if ($current->previousHash !== $previous->hash) {
                return false;
            }
        }
        return true;
    }

    public function getChain(): array
    {
        return $this->chain;
    }

    public function getBlock(int $index): ?Block
    {
        return $this->chain[$index] ?? null;
    }
}

// ทดสอบ
$blockchain = new Blockchain(difficulty: 3);
echo "Mining block 1...\n";
$blockchain->addBlock(json_encode([
    'from' => 'Alice',
    'to' => 'Bob',
    'amount' => 100,
]));

echo "Mining block 2...\n";
$blockchain->addBlock(json_encode([
    'from' => 'Bob',
    'to' => 'Charlie',
    'amount' => 50,
]));

echo "Blockchain valid: " . ($blockchain->isValid() ? 'YES' : 'NO') . "\n";
echo "Chain length: " . count($blockchain->getChain()) . "\n";
```

---

## ขั้นตอนที่ 2652: Merkle Tree

```php
<?php
declare(strict_types=1);

namespace App\Blockchain;

class MerkleTree
{
    private array $leaves;
    private string $root;

    public function __construct(array $transactions)
    {
        $this->leaves = array_map(
            fn ($tx) => hash('sha256', json_encode($tx)),
            $transactions
        );
        $this->root = $this->buildTree($this->leaves);
    }

    private function buildTree(array $nodes): string
    {
        if (count($nodes) === 0) {
            return hash('sha256', '');
        }

        if (count($nodes) === 1) {
            return $nodes[0];
        }

        // เพิ่ม node ถ้าจำนวนเป็นคี่
        if (count($nodes) % 2 !== 0) {
            $nodes[] = end($nodes);
        }

        $parentNodes = [];
        for ($i = 0; $i < count($nodes); $i += 2) {
            $parentNodes[] = hash('sha256', $nodes[$i] . $nodes[$i + 1]);
        }

        return $this->buildTree($parentNodes);
    }

    public function getRoot(): string
    {
        return $this->root;
    }

    public function generateProof(int $index): array
    {
        $proof = [];
        $nodes = $this->leaves;

        if (count($nodes) % 2 !== 0) {
            $nodes[] = end($nodes);
        }

        while (count($nodes) > 1) {
            $pairIndex = $index % 2 === 0 ? $index + 1 : $index - 1;

            if ($pairIndex < count($nodes)) {
                $proof[] = [
                    'position' => $index % 2 === 0 ? 'right' : 'left',
                    'hash' => $nodes[$pairIndex],
                ];
            }

            $index = intdiv($index, 2);
            $newNodes = [];

            for ($i = 0; $i < count($nodes); $i += 2) {
                $newNodes[] = hash('sha256', $nodes[$i] . ($nodes[$i + 1] ?? $nodes[$i]));
            }
            $nodes = $newNodes;
        }

        return $proof;
    }

    public static function verify(string $leaf, array $proof, string $root): bool
    {
        $hash = $leaf;

        foreach ($proof as $step) {
            if ($step['position'] === 'right') {
                $hash = hash('sha256', $hash . $step['hash']);
            } else {
                $hash = hash('sha256', $step['hash'] . $hash);
            }
        }

        return $hash === $root;
    }
}
```

---

## ขั้นตอนที่ 2653: Web3 PHP Integration

```bash
composer require web3p/web3.php
```

```php
<?php
declare(strict_types=1);

namespace App\Services\Blockchain;

use Web3\Web3;
use Web3\Contract;
use Web3\Providers\HttpProvider;
use Web3\RequestManagers\HttpRequestManager;

class EthereumService
{
    private Web3 $web3;
    private string $rpcUrl;

    public function __construct()
    {
        $this->rpcUrl = config('blockchain.ethereum.rpc_url');
        $this->web3 = new Web3(new HttpProvider(
            new HttpRequestManager($this->rpcUrl, 30)
        ));
    }

    public function getBalance(string $address): string
    {
        $balance = null;
        $this->web3->eth->getBalance($address, 'latest', function ($err, $result) use (&$balance) {
            if ($err) {
                throw new \RuntimeException($err->getMessage());
            }
            $balance = $result;
        });

        // แปลง Wei เป็น Ether
        return $this->weiToEther((string)$balance);
    }

    public function getTransaction(string $txHash): ?array
    {
        $transaction = null;
        $this->web3->eth->getTransactionByHash($txHash, function ($err, $tx) use (&$transaction) {
            if ($err) return;
            $transaction = $tx ? (array)$tx : null;
        });
        return $transaction;
    }

    public function getBlockNumber(): int
    {
        $blockNumber = 0;
        $this->web3->eth->blockNumber(function ($err, $number) use (&$blockNumber) {
            if ($err) return;
            $blockNumber = (int)$number->toString();
        });
        return $blockNumber;
    }

    public function callContract(
        string $contractAddress,
        string $abi,
        string $method,
        array $params = []
    ): mixed {
        $contract = new Contract($this->web3->provider, $abi);
        $result = null;

        $contract->at($contractAddress)->call($method, ...$params, function ($err, $res) use (&$result) {
            if ($err) {
                throw new \RuntimeException($err->getMessage());
            }
            $result = $res;
        });

        return $result;
    }

    private function weiToEther(string $wei): string
    {
        $divisor = bcpow('10', '18');
        return bcdiv($wei, $divisor, 18);
    }

    private function etherToWei(string $ether): string
    {
        $multiplier = bcpow('10', '18');
        return bcmul($ether, $multiplier, 0);
    }
}
```

---

## ขั้นตอนที่ 2654: NFT Metadata API

```php
<?php
declare(strict_types=1);

namespace App\Http\Controllers\Api;

use App\Models\NFTCollection;
use App\Models\NFTToken;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class NFTMetadataController extends \App\Http\Controllers\Controller
{
    /**
     * ERC-721 Metadata Standard
     */
    public function tokenMetadata(string $contractAddress, int $tokenId): JsonResponse
    {
        $token = NFTToken::where('contract_address', strtolower($contractAddress))
            ->where('token_id', $tokenId)
            ->with('collection')
            ->first();

        if (!$token) {
            return response()->json(['error' => 'Token not found'], 404);
        }

        // OpenSea Metadata Standard
        return response()->json([
            'name' => $token->name,
            'description' => $token->description,
            'image' => $token->image_url,
            'external_url' => route('nft.detail', [$contractAddress, $tokenId]),
            'background_color' => $token->background_color,
            'animation_url' => $token->animation_url,
            'attributes' => $token->attributes->map(fn ($attr) => [
                'trait_type' => $attr->trait_type,
                'value' => $attr->value,
                'display_type' => $attr->display_type,
                'max_value' => $attr->max_value,
            ])->toArray(),
            'collection' => [
                'name' => $token->collection->name,
                'family' => $token->collection->family,
            ],
        ]);
    }

    public function contractMetadata(string $contractAddress): JsonResponse
    {
        $collection = NFTCollection::where('contract_address', strtolower($contractAddress))
            ->firstOrFail();

        return response()->json([
            'name' => $collection->name,
            'description' => $collection->description,
            'image' => $collection->image_url,
            'external_link' => $collection->external_url,
            'seller_fee_basis_points' => $collection->royalty_basis_points,
            'fee_recipient' => $collection->royalty_recipient,
        ]);
    }

    public function generateMetadata(Request $request): JsonResponse
    {
        $request->validate([
            'name' => ['required', 'string'],
            'description' => ['required', 'string'],
            'image' => ['required', 'file', 'image'],
            'attributes' => ['array'],
            'attributes.*.trait_type' => ['required_with:attributes', 'string'],
            'attributes.*.value' => ['required_with:attributes'],
        ]);

        // อัพโหลดรูปไปยัง IPFS หรือ Arweave
        $imageHash = $this->uploadToIPFS($request->file('image'));

        $metadata = [
            'name' => $request->name,
            'description' => $request->description,
            'image' => "ipfs://{$imageHash}",
            'attributes' => $request->attributes ?? [],
        ];

        // อัพโหลด metadata ไปยัง IPFS
        $metadataHash = $this->uploadJsonToIPFS($metadata);

        return response()->json([
            'metadata_uri' => "ipfs://{$metadataHash}",
            'metadata' => $metadata,
        ], 201);
    }

    private function uploadToIPFS(mixed $file): string
    {
        // ใช้ Pinata หรือ nft.storage
        $response = \Illuminate\Support\Facades\Http::withToken(config('blockchain.pinata.jwt'))
            ->attach('file', $file->getContent(), $file->getClientOriginalName())
            ->post('https://api.pinata.cloud/pinning/pinFileToIPFS');

        return $response->json('IpfsHash');
    }

    private function uploadJsonToIPFS(array $data): string
    {
        $response = \Illuminate\Support\Facades\Http::withToken(config('blockchain.pinata.jwt'))
            ->post('https://api.pinata.cloud/pinning/pinJSONToIPFS', [
                'pinataContent' => $data,
                'pinataMetadata' => ['name' => $data['name']],
            ]);

        return $response->json('IpfsHash');
    }
}
```

---

## ขั้นตอนที่ 2655: Cryptocurrency Payment Verification

```php
<?php
declare(strict_types=1);

namespace App\Services\Blockchain;

use App\Models\CryptoPayment;
use Illuminate\Support\Facades\Http;

class CryptoPaymentVerifier
{
    public function verifyBitcoinPayment(string $address, float $expectedAmount, string $txId): bool
    {
        $response = Http::get("https://api.blockcypher.com/v1/btc/main/txs/{$txId}");

        if (!$response->successful()) {
            return false;
        }

        $tx = $response->json();

        // ตรวจสอบว่า address ได้รับเงินหรือไม่
        $received = 0;
        foreach ($tx['outputs'] as $output) {
            if (in_array($address, $output['addresses'] ?? [])) {
                $received += $output['value'];
            }
        }

        // Bitcoin ใช้ satoshi (1 BTC = 100,000,000 satoshi)
        $expectedSatoshi = (int)($expectedAmount * 100_000_000);

        // ตรวจสอบ confirmations (ต้องการอย่างน้อย 2)
        $confirmations = $tx['confirmations'] ?? 0;

        return $received >= $expectedSatoshi && $confirmations >= 2;
    }

    public function verifyEthereumPayment(
        string $address,
        float $expectedAmountEth,
        string $txHash
    ): bool {
        $ethereumService = app(EthereumService::class);
        $tx = $ethereumService->getTransaction($txHash);

        if (!$tx) return false;

        // ตรวจสอบ to address
        if (strtolower($tx['to'] ?? '') !== strtolower($address)) {
            return false;
        }

        // ตรวจสอบ amount (Wei)
        $expectedWei = bcmul((string)$expectedAmountEth, bcpow('10', '18'));
        $actualWei = hexdec($tx['value'] ?? '0x0');

        return $actualWei >= (int)$expectedWei;
    }
}
```

---

## สรุปบทที่ 93

| หัวข้อ | เทคนิค | Use Case |
|--------|--------|---------|
| Blockchain | Hash + Chain | Immutable records |
| Merkle Tree | Binary hash tree | Transaction verification |
| Web3 PHP | RPC calls | Ethereum interaction |
| NFT Metadata | ERC-721 standard | Digital collectibles |
| IPFS | Distributed storage | Decentralized files |
| Crypto Payment | Transaction verify | Accept crypto |

ถัดไป → Part 94: IoT กับ PHP
