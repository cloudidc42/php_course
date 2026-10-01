# Part 15: REST API Development
## ขั้นตอนที่ 371-400: สร้าง API มืออาชีพ

---

## ขั้นตอนที่ 371: REST API คืออะไร

```
REST (Representational State Transfer) หลักการ:
- Stateless: แต่ละ request ต้องมีข้อมูลครบ
- Resource-based: URL = Resource (/users, /products)
- HTTP Methods: GET, POST, PUT, PATCH, DELETE
- JSON format (หรือ XML)
- Status Codes: 200, 201, 400, 401, 403, 404, 422, 500

HTTP Methods:
GET    /users          → List users
GET    /users/{id}     → Get user
POST   /users          → Create user
PUT    /users/{id}     → Replace user
PATCH  /users/{id}     → Update user partially
DELETE /users/{id}     → Delete user

HTTP Status Codes:
200 OK               → Success
201 Created          → Resource created
204 No Content       → Success, no response body
400 Bad Request      → Client error (validation)
401 Unauthorized     → Authentication required
403 Forbidden        → No permission
404 Not Found        → Resource not found
405 Method Not Allowed
409 Conflict         → Duplicate resource
422 Unprocessable Entity → Validation error
429 Too Many Requests → Rate limited
500 Internal Server Error
503 Service Unavailable
```

---

## ขั้นตอนที่ 372: Router สำหรับ API

```php
<?php
declare(strict_types=1);

class Router {
    private array $routes = [];
    private array $middleware = [];
    private string $prefix = '';
    private array $groupMiddleware = [];
    
    public function get(string $path, callable|array $handler): self {
        return $this->addRoute('GET', $path, $handler);
    }
    
    public function post(string $path, callable|array $handler): self {
        return $this->addRoute('POST', $path, $handler);
    }
    
    public function put(string $path, callable|array $handler): self {
        return $this->addRoute('PUT', $path, $handler);
    }
    
    public function patch(string $path, callable|array $handler): self {
        return $this->addRoute('PATCH', $path, $handler);
    }
    
    public function delete(string $path, callable|array $handler): self {
        return $this->addRoute('DELETE', $path, $handler);
    }
    
    private function addRoute(string $method, string $path, callable|array $handler): self {
        $fullPath = $this->prefix . $path;
        
        $this->routes[] = [
            'method' => $method,
            'path' => $fullPath,
            'handler' => $handler,
            'middleware' => $this->groupMiddleware,
            'pattern' => $this->pathToRegex($fullPath),
        ];
        
        return $this;
    }
    
    public function group(string $prefix, callable $callback, array $middleware = []): void {
        $prevPrefix = $this->prefix;
        $prevMiddleware = $this->groupMiddleware;
        
        $this->prefix = $prevPrefix . $prefix;
        $this->groupMiddleware = array_merge($prevMiddleware, $middleware);
        
        $callback($this);
        
        $this->prefix = $prevPrefix;
        $this->groupMiddleware = $prevMiddleware;
    }
    
    private function pathToRegex(string $path): string {
        $pattern = preg_replace('/\{(\w+)\}/', '(?P<$1>[^/]+)', $path);
        return '#^' . $pattern . '$#';
    }
    
    public function dispatch(string $method, string $uri): void {
        $uri = strtok($uri, '?'); // Remove query string
        
        foreach ($this->routes as $route) {
            if ($route['method'] !== $method) continue;
            
            if (!preg_match($route['pattern'], $uri, $matches)) continue;
            
            // Extract named params
            $params = array_filter($matches, fn($k) => !is_int($k), ARRAY_FILTER_USE_KEY);
            
            // Run middleware
            $response = $this->runMiddleware($route['middleware'], function() use ($route, $params) {
                return $this->callHandler($route['handler'], $params);
            });
            
            return;
        }
        
        // 404
        Response::json(['error' => 'Route not found'], 404);
    }
    
    private function runMiddleware(array $middleware, callable $next): void {
        if (empty($middleware)) {
            $next();
            return;
        }
        
        $mw = array_shift($middleware);
        $instance = new $mw();
        $instance->handle(fn() => $this->runMiddleware($middleware, $next));
    }
    
    private function callHandler(callable|array $handler, array $params): void {
        if (is_array($handler)) {
            [$class, $method] = $handler;
            $instance = new $class();
            $instance->$method(new Request(), new Response(), $params);
        } else {
            $handler(new Request(), new Response(), $params);
        }
    }
}

// ========================
// Request & Response Classes
// ========================
class Request {
    private array $body;
    
    public function __construct() {
        $this->body = json_decode(file_get_contents('php://input'), true) ?? [];
    }
    
    public function input(string $key, mixed $default = null): mixed {
        return $this->body[$key] ?? $_GET[$key] ?? $_POST[$key] ?? $default;
    }
    
    public function all(): array {
        return array_merge($_GET, $_POST, $this->body);
    }
    
    public function get(string $key, mixed $default = null): mixed {
        return $_GET[$key] ?? $default;
    }
    
    public function post(string $key, mixed $default = null): mixed {
        return $_POST[$key] ?? $this->body[$key] ?? $default;
    }
    
    public function header(string $name): ?string {
        $key = 'HTTP_' . strtoupper(str_replace('-', '_', $name));
        return $_SERVER[$key] ?? null;
    }
    
    public function bearerToken(): ?string {
        $auth = $this->header('Authorization');
        if ($auth && str_starts_with($auth, 'Bearer ')) {
            return substr($auth, 7);
        }
        return null;
    }
    
    public function ip(): string {
        return $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
    }
    
    public function method(): string {
        return $_SERVER['REQUEST_METHOD'];
    }
    
    public function isJson(): bool {
        $contentType = $_SERVER['CONTENT_TYPE'] ?? '';
        return str_contains($contentType, 'application/json');
    }
}

class Response {
    public function json(mixed $data, int $status = 200): never {
        http_response_code($status);
        header('Content-Type: application/json; charset=utf-8');
        echo json_encode($data, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
        exit;
    }
    
    public function success(mixed $data = null, string $message = 'Success', int $status = 200): never {
        $this->json([
            'success' => true,
            'message' => $message,
            'data' => $data,
        ], $status);
    }
    
    public function error(string $message, int $status = 400, mixed $errors = null): never {
        $response = ['success' => false, 'message' => $message];
        if ($errors !== null) $response['errors'] = $errors;
        $this->json($response, $status);
    }
    
    public function paginated(array $items, int $total, int $page, int $perPage): never {
        $this->json([
            'success' => true,
            'data' => $items,
            'meta' => [
                'total' => $total,
                'page' => $page,
                'per_page' => $perPage,
                'last_page' => (int) ceil($total / $perPage),
                'from' => ($page - 1) * $perPage + 1,
                'to' => min($page * $perPage, $total),
            ],
        ]);
    }
}
```

---

## ขั้นตอนที่ 373: JWT Authentication

```php
<?php
declare(strict_types=1);

class JWT {
    private string $secret;
    private string $algorithm;
    private int $expiry;
    
    public function __construct(string $secret, int $expiry = 3600, string $algorithm = 'HS256') {
        $this->secret = $secret;
        $this->algorithm = $algorithm;
        $this->expiry = $expiry;
    }
    
    public function generate(array $payload): string {
        $header = ['alg' => $this->algorithm, 'typ' => 'JWT'];
        
        $payload = array_merge($payload, [
            'iat' => time(),
            'exp' => time() + $this->expiry,
            'jti' => bin2hex(random_bytes(8)),
        ]);
        
        $encodedHeader = $this->base64UrlEncode(json_encode($header));
        $encodedPayload = $this->base64UrlEncode(json_encode($payload));
        
        $signature = $this->sign("{$encodedHeader}.{$encodedPayload}");
        
        return "{$encodedHeader}.{$encodedPayload}.{$signature}";
    }
    
    public function verify(string $token): array {
        $parts = explode('.', $token);
        
        if (count($parts) !== 3) {
            throw new \RuntimeException('Invalid token format');
        }
        
        [$encodedHeader, $encodedPayload, $signature] = $parts;
        
        // Verify signature
        $expectedSignature = $this->sign("{$encodedHeader}.{$encodedPayload}");
        if (!hash_equals($expectedSignature, $signature)) {
            throw new \RuntimeException('Invalid token signature');
        }
        
        // Decode payload
        $payload = json_decode($this->base64UrlDecode($encodedPayload), true);
        
        if (!$payload) {
            throw new \RuntimeException('Invalid token payload');
        }
        
        // Check expiry
        if (isset($payload['exp']) && $payload['exp'] < time()) {
            throw new \RuntimeException('Token expired');
        }
        
        return $payload;
    }
    
    private function sign(string $input): string {
        $hash = hash_hmac('sha256', $input, $this->secret, true);
        return $this->base64UrlEncode($hash);
    }
    
    private function base64UrlEncode(string $data): string {
        return rtrim(strtr(base64_encode($data), '+/', '-_'), '=');
    }
    
    private function base64UrlDecode(string $data): string {
        return base64_decode(strtr($data, '-_', '+/') . str_repeat('=', 3 - (3 + strlen($data)) % 4));
    }
}

// JWT Middleware
class JwtMiddleware {
    private JWT $jwt;
    
    public function __construct() {
        $this->jwt = new JWT($_ENV['JWT_SECRET']);
    }
    
    public function handle(callable $next): void {
        $token = (new Request())->bearerToken();
        
        if (!$token) {
            (new Response())->error('No token provided', 401);
        }
        
        try {
            $payload = $this->jwt->verify($token);
            $_SERVER['JWT_PAYLOAD'] = $payload;
            $next();
        } catch (\RuntimeException $e) {
            (new Response())->error($e->getMessage(), 401);
        }
    }
}

// Token Refresh
class TokenService {
    private JWT $jwt;
    private \PDO $pdo;
    
    public function __construct(\PDO $pdo) {
        $this->pdo = $pdo;
        $this->jwt = new JWT($_ENV['JWT_SECRET'], 900); // 15 min access token
    }
    
    public function generateTokens(array $user): array {
        $accessToken = $this->jwt->generate([
            'user_id' => $user['id'],
            'email' => $user['email'],
            'role' => $user['role'],
        ]);
        
        $refreshToken = bin2hex(random_bytes(64));
        $this->storeRefreshToken($user['id'], $refreshToken);
        
        return [
            'access_token' => $accessToken,
            'refresh_token' => $refreshToken,
            'token_type' => 'Bearer',
            'expires_in' => 900,
        ];
    }
    
    public function refreshAccessToken(string $refreshToken): array {
        $stmt = $this->pdo->prepare(
            "SELECT user_id FROM refresh_tokens 
             WHERE token_hash = ? AND expires_at > NOW() AND revoked = 0"
        );
        $stmt->execute([hash('sha256', $refreshToken)]);
        $result = $stmt->fetch();
        
        if (!$result) {
            throw new \RuntimeException('Invalid or expired refresh token');
        }
        
        // Rotate refresh token (one-time use)
        $this->revokeRefreshToken($refreshToken);
        
        $stmt = $this->pdo->prepare("SELECT * FROM users WHERE id = ?");
        $stmt->execute([$result['user_id']]);
        $user = $stmt->fetch();
        
        return $this->generateTokens($user);
    }
    
    private function storeRefreshToken(int $userId, string $token): void {
        $stmt = $this->pdo->prepare(
            "INSERT INTO refresh_tokens (user_id, token_hash, expires_at) 
             VALUES (?, ?, DATE_ADD(NOW(), INTERVAL 30 DAY))"
        );
        $stmt->execute([$userId, hash('sha256', $token)]);
    }
    
    private function revokeRefreshToken(string $token): void {
        $stmt = $this->pdo->prepare(
            "UPDATE refresh_tokens SET revoked = 1 WHERE token_hash = ?"
        );
        $stmt->execute([hash('sha256', $token)]);
    }
}
```

---

## ขั้นตอนที่ 374: API Controllers

```php
<?php
declare(strict_types=1);

abstract class ApiController {
    protected function validate(Request $request, array $rules): array {
        $validator = Validator::make($request->all(), $rules);
        
        if ($validator->fails()) {
            (new Response())->error('Validation failed', 422, $validator->errors());
        }
        
        return $validator->validated();
    }
    
    protected function currentUser(): ?array {
        return $_SERVER['JWT_PAYLOAD'] ?? null;
    }
    
    protected function authorize(string $ability): void {
        // Basic authorization
        $user = $this->currentUser();
        if (!$user) {
            (new Response())->error('Unauthorized', 401);
        }
    }
}

// Users API Controller
class UserController extends ApiController {
    public function __construct(private UserRepository $users) {}
    
    public function index(Request $req, Response $res, array $params): void {
        $page = (int) $req->get('page', 1);
        $perPage = min((int) $req->get('per_page', 15), 100);
        
        $users = $this->users->findAll($page, $perPage);
        $total = $this->users->count();
        
        $res->paginated(
            array_map(fn($u) => $this->transform($u), $users),
            $total, $page, $perPage
        );
    }
    
    public function show(Request $req, Response $res, array $params): void {
        $user = $this->users->findById((int) $params['id']);
        
        if (!$user) {
            $res->error('User not found', 404);
        }
        
        $res->success($this->transform($user));
    }
    
    public function store(Request $req, Response $res, array $params): void {
        $data = $this->validate($req, [
            'name' => 'required|min:2|max:100',
            'email' => 'required|email',
            'password' => 'required|min:8',
        ]);
        
        // Check email unique
        if ($this->users->findByEmail($data['email'])) {
            $res->error('Email already exists', 409);
        }
        
        $user = new User(
            0,
            $data['name'],
            $data['email'],
        );
        
        $created = $this->users->save($user);
        $res->success($this->transform($created), 'User created', 201);
    }
    
    public function update(Request $req, Response $res, array $params): void {
        $user = $this->users->findById((int) $params['id']);
        if (!$user) $res->error('User not found', 404);
        
        $data = $this->validate($req, [
            'name' => 'min:2|max:100',
            'email' => 'email',
        ]);
        
        $updated = new User(
            $user->id,
            $data['name'] ?? $user->name,
            $data['email'] ?? $user->email,
            $user->role,
            $user->active,
        );
        
        $res->success($this->transform($this->users->save($updated)));
    }
    
    public function destroy(Request $req, Response $res, array $params): void {
        $user = $this->users->findById((int) $params['id']);
        if (!$user) $res->error('User not found', 404);
        
        $this->users->delete($user->id);
        
        http_response_code(204);
        exit;
    }
    
    private function transform(User $user): array {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'role' => $user->role,
            'created_at' => $user->createdAt->format('c'),
        ];
    }
}

// ========================
// Register Routes
// ========================
$router = new Router();

// Public routes
$router->post('/api/auth/login', [AuthController::class, 'login']);
$router->post('/api/auth/register', [AuthController::class, 'register']);
$router->post('/api/auth/refresh', [AuthController::class, 'refresh']);

// Protected routes
$router->group('/api', function(Router $r) {
    $r->get('/users', [UserController::class, 'index']);
    $r->get('/users/{id}', [UserController::class, 'show']);
    $r->post('/users', [UserController::class, 'store']);
    $r->patch('/users/{id}', [UserController::class, 'update']);
    $r->delete('/users/{id}', [UserController::class, 'destroy']);
}, [JwtMiddleware::class]);

// Dispatch
header('Content-Type: application/json');
header('Access-Control-Allow-Origin: *');
header('Access-Control-Allow-Methods: GET, POST, PUT, PATCH, DELETE, OPTIONS');
header('Access-Control-Allow-Headers: Content-Type, Authorization');

if ($_SERVER['REQUEST_METHOD'] === 'OPTIONS') {
    http_response_code(204);
    exit;
}

$router->dispatch($_SERVER['REQUEST_METHOD'], $_SERVER['REQUEST_URI']);
```

---

## ขั้นตอนที่ 375: API Versioning & Documentation

```php
<?php
declare(strict_types=1);

// Version prefix
$router->group('/api/v1', function(Router $r) {
    $r->get('/users', [UserControllerV1::class, 'index']);
});

$router->group('/api/v2', function(Router $r) {
    $r->get('/users', [UserControllerV2::class, 'index']); // New features
});

// API Response Wrapper
class ApiResponse {
    public static function success(mixed $data, string $message = 'OK', int $status = 200): array {
        http_response_code($status);
        return [
            'status' => 'success',
            'code' => $status,
            'message' => $message,
            'data' => $data,
            'meta' => [
                'version' => 'v1',
                'timestamp' => date('c'),
                'request_id' => bin2hex(random_bytes(8)),
            ],
        ];
    }
    
    public static function paginate(array $data, int $total, int $page, int $perPage): array {
        $lastPage = (int) ceil($total / $perPage);
        
        return [
            'status' => 'success',
            'data' => $data,
            'pagination' => [
                'total' => $total,
                'count' => count($data),
                'per_page' => $perPage,
                'current_page' => $page,
                'last_page' => $lastPage,
                'has_more' => $page < $lastPage,
                'links' => [
                    'first' => "?page=1&per_page={$perPage}",
                    'last' => "?page={$lastPage}&per_page={$perPage}",
                    'prev' => $page > 1 ? "?page=" . ($page - 1) . "&per_page={$perPage}" : null,
                    'next' => $page < $lastPage ? "?page=" . ($page + 1) . "&per_page={$perPage}" : null,
                ],
            ],
        ];
    }
    
    public static function error(string $message, int $status = 400, array $errors = []): array {
        http_response_code($status);
        $response = [
            'status' => 'error',
            'code' => $status,
            'message' => $message,
        ];
        
        if (!empty($errors)) {
            $response['errors'] = $errors;
        }
        
        return $response;
    }
}
```

---

## 🎯 สรุป Part 15

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| REST Principles | Stateless, Resource-based, HTTP Methods |
| Router | Pattern matching, Groups, Middleware |
| JWT | Generate, Verify, Refresh tokens |
| Controllers | CRUD, Validation, Transform |
| Response | JSON, Pagination, Error format |
| Security | Rate limiting, CORS, API Keys |

**ถัดไป → Part 16: Composer & Autoloading**
