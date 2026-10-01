# Part 36: GraphQL API ด้วย PHP
## ขั้นตอนที่ 941-970: Modern API Architecture

---

## ขั้นตอนที่ 941: GraphQL Basics

```bash
# ติดตั้ง graphql-php
composer require webonyx/graphql-php

# หรือสำหรับ Laravel
composer require rebing/graphql-laravel
php artisan vendor:publish --provider="Rebing\GraphQL\GraphQLServiceProvider"
```

---

## ขั้นตอนที่ 942: Schema Definition

```php
<?php
declare(strict_types=1);

use GraphQL\Type\Definition\Type;
use GraphQL\Type\Definition\ObjectType;
use GraphQL\Type\Definition\InputObjectType;
use GraphQL\Type\Schema;
use GraphQL\GraphQL;
use GraphQL\Error\DebugFlag;

// Types
$userType = new ObjectType([
    'name'        => 'User',
    'description' => 'A registered user',
    'fields'      => fn() => [
        'id' => [
            'type'        => Type::nonNull(Type::int()),
            'description' => 'User ID',
        ],
        'name' => [
            'type'        => Type::nonNull(Type::string()),
            'description' => 'Full name',
        ],
        'email' => [
            'type'        => Type::nonNull(Type::string()),
            'description' => 'Email address',
        ],
        'createdAt' => [
            'type'    => Type::string(),
            'resolve' => fn($user) => $user['created_at'],
        ],
        'posts' => [
            'type'    => Type::listOf($postType),
            'resolve' => function($user, $args, AppContext $context): array {
                return $context->postLoader->loadForUser($user['id']);
            },
        ],
        'postCount' => [
            'type'    => Type::int(),
            'resolve' => fn($user, $args, AppContext $context) =>
                $context->postLoader->countForUser($user['id']),
        ],
    ],
]);

$postType = new ObjectType([
    'name'   => 'Post',
    'fields' => fn() => [
        'id'        => ['type' => Type::nonNull(Type::int())],
        'title'     => ['type' => Type::nonNull(Type::string())],
        'body'      => ['type' => Type::string()],
        'status'    => ['type' => Type::string()],
        'author'    => [
            'type'    => $userType,
            'resolve' => fn($post, $args, AppContext $ctx) =>
                $ctx->userLoader->load($post['user_id']),
        ],
        'tags'      => [
            'type'    => Type::listOf(Type::string()),
            'resolve' => fn($post) => explode(',', $post['tags'] ?? ''),
        ],
    ],
]);

// Queries
$queryType = new ObjectType([
    'name'   => 'Query',
    'fields' => [
        'user' => [
            'type' => $userType,
            'args' => [
                'id' => ['type' => Type::nonNull(Type::int())],
            ],
            'resolve' => function($root, array $args, AppContext $context): ?array {
                return $context->db->fetchUser($args['id']);
            },
        ],
        
        'users' => [
            'type' => Type::listOf($userType),
            'args' => [
                'limit'  => ['type' => Type::int(), 'defaultValue' => 10],
                'offset' => ['type' => Type::int(), 'defaultValue' => 0],
                'search' => ['type' => Type::string()],
            ],
            'resolve' => function($root, array $args, AppContext $context): array {
                return $context->db->fetchUsers(
                    limit:  min(100, $args['limit']),
                    offset: $args['offset'],
                    search: $args['search'] ?? null
                );
            },
        ],
        
        'posts' => [
            'type' => Type::listOf($postType),
            'args' => [
                'status' => ['type' => Type::string(), 'defaultValue' => 'published'],
                'limit'  => ['type' => Type::int(), 'defaultValue' => 20],
                'tag'    => ['type' => Type::string()],
            ],
            'resolve' => function($root, array $args, AppContext $context): array {
                $whitelist = ['draft', 'published', 'archived'];
                $status = in_array($args['status'], $whitelist) ? $args['status'] : 'published';
                return $context->db->fetchPosts($status, $args['limit'], $args['tag'] ?? null);
            },
        ],
        
        // Search across types
        'search' => [
            'type' => new \GraphQL\Type\Definition\UnionType([
                'name'        => 'SearchResult',
                'types'       => [$userType, $postType],
                'resolveType' => fn($value) => isset($value['email']) ? $userType : $postType,
            ]),
            'args' => [
                'query' => ['type' => Type::nonNull(Type::string())],
            ],
            'resolve' => fn($root, array $args, AppContext $ctx) =>
                $ctx->db->search($args['query']),
        ],
    ],
]);

// Mutations
$mutationType = new ObjectType([
    'name'   => 'Mutation',
    'fields' => [
        'createUser' => [
            'type' => $userType,
            'args' => [
                'input' => [
                    'type' => Type::nonNull(new InputObjectType([
                        'name'   => 'CreateUserInput',
                        'fields' => [
                            'name'     => ['type' => Type::nonNull(Type::string())],
                            'email'    => ['type' => Type::nonNull(Type::string())],
                            'password' => ['type' => Type::nonNull(Type::string())],
                        ],
                    ])),
                ],
            ],
            'resolve' => function($root, array $args, AppContext $context): array {
                if (!$context->user?->isAdmin()) {
                    throw new \GraphQL\Error\UserError('Unauthorized');
                }
                
                $input = $args['input'];
                
                // Validate
                if (!filter_var($input['email'], FILTER_VALIDATE_EMAIL)) {
                    throw new \GraphQL\Error\UserError('Invalid email');
                }
                
                return $context->db->createUser([
                    'name'     => htmlspecialchars($input['name']),
                    'email'    => $input['email'],
                    'password' => password_hash($input['password'], PASSWORD_ARGON2ID),
                ]);
            },
        ],
        
        'updatePost' => [
            'type' => $postType,
            'args' => [
                'id'    => ['type' => Type::nonNull(Type::int())],
                'title' => ['type' => Type::string()],
                'body'  => ['type' => Type::string()],
            ],
            'resolve' => function($root, array $args, AppContext $ctx): array {
                $post = $ctx->db->fetchPost($args['id']);
                
                if (!$post) throw new \GraphQL\Error\UserError('Post not found');
                if ($post['user_id'] !== $ctx->user?->id) throw new \GraphQL\Error\UserError('Forbidden');
                
                $updates = array_filter([
                    'title' => $args['title'] ?? null,
                    'body'  => $args['body'] ?? null,
                ]);
                
                return $ctx->db->updatePost($args['id'], $updates);
            },
        ],
    ],
]);

// Subscriptions (WebSocket)
$subscriptionType = new ObjectType([
    'name'   => 'Subscription',
    'fields' => [
        'postAdded' => [
            'type'    => $postType,
            'resolve' => fn($post) => $post,
            'subscribe' => fn($root, $args, AppContext $ctx) =>
                $ctx->pubSub->subscribe('post.added'),
        ],
    ],
]);

// Build schema
$schema = new Schema([
    'query'    => $queryType,
    'mutation' => $mutationType,
]);
```

---

## ขั้นตอนที่ 943: GraphQL Server

```php
<?php
declare(strict_types=1);

// api/graphql.php

require_once '../vendor/autoload.php';

use GraphQL\GraphQL;
use GraphQL\Error\DebugFlag;

$context = new AppContext(
    db:         new Database(),
    user:       getCurrentUser(),
    userLoader: new DataLoader\UserLoader(),
    postLoader: new DataLoader\PostLoader(),
);

$input   = json_decode(file_get_contents('php://input'), true);
$query   = $input['query'] ?? '';
$vars    = $input['variables'] ?? null;
$opName  = $input['operationName'] ?? null;

try {
    $result = GraphQL::executeQuery(
        schema:          $schema,
        source:          $query,
        rootValue:       null,
        contextValue:    $context,
        variableValues:  $vars,
        operationName:   $opName,
    );
    
    $debug = getenv('APP_ENV') === 'production'
        ? DebugFlag::NONE
        : DebugFlag::INCLUDE_DEBUG_MESSAGE | DebugFlag::INCLUDE_TRACE;
    
    $output = $result->toArray($debug);
    
} catch (\Exception $e) {
    $output = [
        'errors' => [['message' => 'Internal server error']],
    ];
    error_log($e->getMessage());
}

header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');
echo json_encode($output);
```

---

## ขั้นตอนที่ 944: DataLoader (N+1 Solution)

```php
<?php
declare(strict_types=1);

namespace DataLoader;

use PDO;

class DataLoader {
    private array $batch = [];
    private array $cache = [];
    private bool $batching = true;
    
    public function __construct(
        private callable $batchFn
    ) {}
    
    public function load(int $id): mixed {
        if (isset($this->cache[$id])) {
            return $this->cache[$id];
        }
        
        $this->batch[] = $id;
        
        // Return promise-like - actual resolution at dispatch()
        return $id;
    }
    
    public function dispatch(): void {
        if (empty($this->batch)) return;
        
        $ids = array_unique($this->batch);
        $results = ($this->batchFn)($ids);
        
        foreach ($results as $id => $value) {
            $this->cache[$id] = $value;
        }
        
        $this->batch = [];
    }
    
    public function getLoaded(int $id): mixed {
        return $this->cache[$id] ?? null;
    }
    
    public function clear(int $id): void {
        unset($this->cache[$id]);
    }
    
    public function clearAll(): void {
        $this->cache = [];
    }
}

class UserLoader extends DataLoader {
    public function __construct(private PDO $pdo) {
        parent::__construct(function(array $ids): array {
            $placeholders = implode(',', array_fill(0, count($ids), '?'));
            $stmt = $this->pdo->prepare("SELECT * FROM users WHERE id IN ({$placeholders})");
            $stmt->execute($ids);
            
            $users = [];
            foreach ($stmt->fetchAll() as $user) {
                $users[$user['id']] = $user;
            }
            return $users;
        });
    }
    
    public function loadForIds(array $ids): array {
        if (empty($ids)) return [];
        
        $placeholders = implode(',', array_fill(0, count($ids), '?'));
        $stmt = $this->pdo->prepare("SELECT * FROM users WHERE id IN ({$placeholders})");
        $stmt->execute($ids);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
```

---

## ขั้นตอนที่ 945: Laravel GraphQL (Rebing)

```php
<?php
declare(strict_types=1);

// app/GraphQL/Types/UserType.php

namespace App\GraphQL\Types;

use App\Models\User;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Type as GraphQLType;
use Rebing\GraphQL\Support\Facades\GraphQL;

class UserType extends GraphQLType {
    protected $attributes = [
        'name'        => 'User',
        'description' => 'A user',
        'model'       => User::class,
    ];
    
    public function fields(): array {
        return [
            'id'   => ['type' => Type::nonNull(Type::int())],
            'name' => ['type' => Type::nonNull(Type::string())],
            'email'=> ['type' => Type::nonNull(Type::string())],
            
            'posts' => [
                'type'    => Type::listOf(GraphQL::type('Post')),
                'is_relation' => true,
            ],
            
            'formattedDate' => [
                'type'    => Type::string(),
                'alias'   => 'created_at',
                'resolve' => fn(User $user) => $user->created_at->diffForHumans(),
            ],
        ];
    }
}

// app/GraphQL/Queries/UsersQuery.php

namespace App\GraphQL\Queries;

use App\Models\User;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Query;
use Rebing\GraphQL\Support\Facades\GraphQL;

class UsersQuery extends Query {
    protected $attributes = ['name' => 'users'];
    
    public function type(): Type {
        return Type::listOf(GraphQL::type('User'));
    }
    
    public function args(): array {
        return [
            'limit'  => ['type' => Type::int(), 'defaultValue' => 10],
            'search' => ['type' => Type::string()],
        ];
    }
    
    public function authorize($root, array $args, $ctx, $info): bool {
        return auth()->check();
    }
    
    public function resolve($root, array $args) {
        return User::query()
            ->when($args['search'] ?? null, fn($q, $s) => $q->where('name', 'like', "%{$s}%"))
            ->limit(min(100, $args['limit']))
            ->get();
    }
}

// app/GraphQL/Mutations/CreateUserMutation.php

namespace App\GraphQL\Mutations;

use App\Models\User;
use GraphQL\Type\Definition\Type;
use Rebing\GraphQL\Support\Mutation;
use Rebing\GraphQL\Support\Facades\GraphQL;

class CreateUserMutation extends Mutation {
    protected $attributes = ['name' => 'createUser'];
    
    public function type(): Type {
        return GraphQL::type('User');
    }
    
    public function args(): array {
        return [
            'name'     => ['type' => Type::nonNull(Type::string()), 'rules' => ['required', 'min:2']],
            'email'    => ['type' => Type::nonNull(Type::string()), 'rules' => ['required', 'email', 'unique:users']],
            'password' => ['type' => Type::nonNull(Type::string()), 'rules' => ['required', 'min:8']],
        ];
    }
    
    public function authorize($root, array $args, $ctx, $info): bool {
        return auth()->user()?->isAdmin();
    }
    
    public function resolve($root, array $args): User {
        return User::create([
            'name'     => $args['name'],
            'email'    => $args['email'],
            'password' => bcrypt($args['password']),
        ]);
    }
}
```

---

## 🎯 สรุป Part 36

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Schema | ObjectType, InputObjectType, UnionType |
| Queries | args, resolve, context |
| Mutations | UserError, authorization |
| DataLoader | N+1 prevention, batch loading |
| Laravel GraphQL | Rebing package, Types, Queries, Mutations |
| Authorization | authorize() method |

**ถัดไป → Part 37: PHP Static Analysis & Code Quality**
