# Part 91: PHP Standards (PSR)
## ขั้นตอนที่ 2591-2620: PSR-1 ถึง PSR-20

PHP Standards Recommendations (PSR) คือ มาตรฐานที่ PHP-FIG กำหนดขึ้น
เพื่อให้ code ทำงานร่วมกันได้ระหว่าง frameworks และ libraries ต่าง ๆ

---

## ขั้นตอนที่ 2591: PSR-1 Basic Coding Standard

```php
<?php
declare(strict_types=1);

// PSR-1: กฎพื้นฐาน

// 1. Files ต้องใช้ <?php หรือ <?= เท่านั้น
// 2. Files ต้องใช้ UTF-8 without BOM
// 3. Files ควรทำอย่างใดอย่างหนึ่ง: declare symbols หรือ side effects
// 4. Namespaces ต้องตาม PSR-4 autoloading
// 5. Class names ต้องเป็น PascalCase
// 6. Class constants ต้องเป็น UPPER_CASE
// 7. Method names ต้องเป็น camelCase

namespace App\Services;

// ถูก
class UserService
{
    public const DEFAULT_ROLE = 'user';
    protected const MAX_LOGIN_ATTEMPTS = 5;

    public function createUser(string $email): User
    {
        // ...
    }

    private function hashPassword(string $password): string
    {
        return password_hash($password, PASSWORD_BCRYPT);
    }
}

// PSR-12: Extended Coding Style
class ExtendedExample extends BaseClass implements InterfaceA, InterfaceB
{
    use TraitA, TraitB;

    private const VERSION = '1.0.0';

    public function __construct(
        private readonly string $name,
        private int $age = 0,
        protected ?string $email = null,
    ) {
        parent::__construct();
    }

    public function process(
        string $input,
        bool $strict = false,
    ): array {
        if ($strict) {
            return $this->strictProcess($input);
        }

        return $this->normalProcess($input);
    }
}
```

---

## ขั้นตอนที่ 2592: PSR-3 Logger Interface

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Logging;

use Psr\Log\AbstractLogger;
use Psr\Log\LogLevel;

class CustomLogger extends AbstractLogger
{
    private array $handlers = [];
    private string $channel;

    public function __construct(string $channel = 'app')
    {
        $this->channel = $channel;
    }

    public function addHandler(callable $handler): void
    {
        $this->handlers[] = $handler;
    }

    /**
     * {@inheritdoc}
     */
    public function log($level, \Stringable|string $message, array $context = []): void
    {
        $formatted = $this->formatMessage($level, $message, $context);

        foreach ($this->handlers as $handler) {
            $handler($formatted);
        }
    }

    private function formatMessage(string $level, string $message, array $context): array
    {
        // Interpolate context values
        $interpolated = $this->interpolate($message, $context);

        return [
            'channel' => $this->channel,
            'level' => $level,
            'message' => $interpolated,
            'context' => $context,
            'timestamp' => date('Y-m-d H:i:s'),
            'pid' => getmypid(),
        ];
    }

    private function interpolate(string $message, array $context): string
    {
        $replace = [];
        foreach ($context as $key => $val) {
            if (!is_array($val) && (!is_object($val) || method_exists($val, '__toString'))) {
                $replace['{' . $key . '}'] = $val;
            }
        }
        return strtr($message, $replace);
    }
}

// ใช้งาน
$logger = new CustomLogger('app');
$logger->addHandler(fn ($log) => error_log(json_encode($log)));
$logger->info('User {username} logged in', ['username' => 'john']);
$logger->error('Payment failed for order #{order_id}', ['order_id' => 123]);
```

---

## ขั้นตอนที่ 2593: PSR-4 Autoloading

```php
<?php
declare(strict_types=1);

// composer.json
// {
//     "autoload": {
//         "psr-4": {
//             "App\\": "app/",
//             "Database\\": "database/",
//             "Tests\\": "tests/"
//         }
//     }
// }

// PSR-4 กำหนด:
// 1. Namespace prefix จะ map ไปที่ base directory
// 2. Namespace ที่เหลือ map เป็น directory path
// 3. Class name map เป็น filename

// App\Http\Controllers\PostController -> app/Http/Controllers/PostController.php
// App\Models\User -> app/Models/User.php
// Database\Factories\UserFactory -> database/Factories/UserFactory.php

class PSR4AutoloaderDemo
{
    /**
     * Implementation ของ PSR-4 autoloader
     */
    public static function register(string $prefix, string $baseDir): void
    {
        spl_autoload_register(function (string $class) use ($prefix, $baseDir) {
            // ตรวจสอบว่า class ตรงกับ prefix หรือไม่
            $len = strlen($prefix);
            if (strncmp($prefix, $class, $len) !== 0) {
                return;
            }

            // แปลง namespace path เป็น file path
            $relativeClass = substr($class, $len);
            $file = $baseDir . str_replace('\\', '/', $relativeClass) . '.php';

            if (file_exists($file)) {
                require $file;
            }
        });
    }
}
```

---

## ขั้นตอนที่ 2594: PSR-7 HTTP Messages

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Http;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\StreamInterface;

// PSR-7: HTTP Message Interface Implementation

class Response implements ResponseInterface
{
    private array $headers = [];
    private StreamInterface $body;
    private int $statusCode;
    private string $reasonPhrase;
    private string $protocolVersion = '1.1';

    public function __construct(
        int $status = 200,
        array $headers = [],
        ?StreamInterface $body = null,
    ) {
        $this->statusCode = $status;
        $this->headers = $headers;
        $this->body = $body ?? new Stream('php://memory', 'r+');
        $this->reasonPhrase = $this->getDefaultReasonPhrase($status);
    }

    public function getStatusCode(): int
    {
        return $this->statusCode;
    }

    public function withStatus($code, $reasonPhrase = ''): static
    {
        $clone = clone $this;
        $clone->statusCode = $code;
        $clone->reasonPhrase = $reasonPhrase ?: $this->getDefaultReasonPhrase($code);
        return $clone;
    }

    public function getReasonPhrase(): string
    {
        return $this->reasonPhrase;
    }

    public function getProtocolVersion(): string
    {
        return $this->protocolVersion;
    }

    public function withProtocolVersion($version): static
    {
        $clone = clone $this;
        $clone->protocolVersion = $version;
        return $clone;
    }

    public function getHeaders(): array
    {
        return $this->headers;
    }

    public function hasHeader($name): bool
    {
        return isset($this->headers[strtolower($name)]);
    }

    public function getHeader($name): array
    {
        $name = strtolower($name);
        return $this->headers[$name] ?? [];
    }

    public function getHeaderLine($name): string
    {
        return implode(', ', $this->getHeader($name));
    }

    public function withHeader($name, $value): static
    {
        $clone = clone $this;
        $clone->headers[strtolower($name)] = (array)$value;
        return $clone;
    }

    public function withAddedHeader($name, $value): static
    {
        $clone = clone $this;
        $name = strtolower($name);
        $clone->headers[$name] = array_merge($clone->headers[$name] ?? [], (array)$value);
        return $clone;
    }

    public function withoutHeader($name): static
    {
        $clone = clone $this;
        unset($clone->headers[strtolower($name)]);
        return $clone;
    }

    public function getBody(): StreamInterface
    {
        return $this->body;
    }

    public function withBody(StreamInterface $body): static
    {
        $clone = clone $this;
        $clone->body = $body;
        return $clone;
    }

    private function getDefaultReasonPhrase(int $code): string
    {
        return match ($code) {
            200 => 'OK', 201 => 'Created', 204 => 'No Content',
            301 => 'Moved Permanently', 302 => 'Found',
            400 => 'Bad Request', 401 => 'Unauthorized',
            403 => 'Forbidden', 404 => 'Not Found',
            422 => 'Unprocessable Entity',
            500 => 'Internal Server Error',
            default => '',
        };
    }
}
```

---

## ขั้นตอนที่ 2595: PSR-11 Container Interface

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Container;

use Psr\Container\ContainerInterface;
use Psr\Container\NotFoundExceptionInterface;

class Container implements ContainerInterface
{
    private array $bindings = [];
    private array $instances = [];
    private array $resolving = [];

    public function bind(string $abstract, callable|string $concrete = null): void
    {
        $this->bindings[$abstract] = $concrete ?? $abstract;
    }

    public function singleton(string $abstract, callable|string $concrete = null): void
    {
        $this->bind($abstract, $concrete);
        $this->instances[$abstract] = null; // Mark as singleton
    }

    /**
     * {@inheritdoc}
     */
    public function get(string $id): mixed
    {
        if (!$this->has($id)) {
            throw new class ("No binding for {$id}") extends \RuntimeException implements NotFoundExceptionInterface {};
        }

        // Return singleton if already instantiated
        if (array_key_exists($id, $this->instances) && $this->instances[$id] !== null) {
            return $this->instances[$id];
        }

        // Detect circular dependencies
        if (isset($this->resolving[$id])) {
            throw new \RuntimeException("Circular dependency detected for {$id}");
        }

        $this->resolving[$id] = true;
        $instance = $this->resolve($id);
        unset($this->resolving[$id]);

        // Cache singleton
        if (array_key_exists($id, $this->instances)) {
            $this->instances[$id] = $instance;
        }

        return $instance;
    }

    /**
     * {@inheritdoc}
     */
    public function has(string $id): bool
    {
        return isset($this->bindings[$id]) || class_exists($id);
    }

    private function resolve(string $id): mixed
    {
        $concrete = $this->bindings[$id] ?? $id;

        if (is_callable($concrete)) {
            return $concrete($this);
        }

        // Auto-wire using Reflection
        $reflector = new \ReflectionClass($concrete);

        if (!$reflector->isInstantiable()) {
            throw new \RuntimeException("Cannot instantiate {$concrete}");
        }

        $constructor = $reflector->getConstructor();
        if ($constructor === null) {
            return new $concrete();
        }

        $dependencies = array_map(function (\ReflectionParameter $param) {
            $type = $param->getType();
            if ($type instanceof \ReflectionNamedType && !$type->isBuiltin()) {
                return $this->get($type->getName());
            }
            if ($param->isDefaultValueAvailable()) {
                return $param->getDefaultValue();
            }
            throw new \RuntimeException("Cannot resolve parameter {$param->getName()}");
        }, $constructor->getParameters());

        return $reflector->newInstanceArgs($dependencies);
    }
}
```

---

## ขั้นตอนที่ 2596: PSR-15 HTTP Middleware

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Middleware;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

class RateLimitMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly int $maxRequests = 100,
        private readonly int $windowSeconds = 60,
        private readonly \Psr\SimpleCache\CacheInterface $cache,
    ) {}

    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $clientIp = $this->getClientIp($request);
        $key = "rate_limit:{$clientIp}";

        $current = (int)($this->cache->get($key, 0));

        if ($current >= $this->maxRequests) {
            return new Response(429, ['Retry-After' => $this->windowSeconds], 'Too Many Requests');
        }

        $this->cache->set($key, $current + 1, $this->windowSeconds);

        $response = $handler->handle($request);

        return $response
            ->withHeader('X-RateLimit-Limit', (string)$this->maxRequests)
            ->withHeader('X-RateLimit-Remaining', (string)($this->maxRequests - $current - 1))
            ->withHeader('X-RateLimit-Reset', (string)(time() + $this->windowSeconds));
    }

    private function getClientIp(ServerRequestInterface $request): string
    {
        $params = $request->getServerParams();
        return $params['HTTP_X_FORWARDED_FOR']
            ?? $params['HTTP_X_REAL_IP']
            ?? $params['REMOTE_ADDR']
            ?? 'unknown';
    }
}

// PSR-15 Request Handler
class MiddlewarePipeline implements RequestHandlerInterface
{
    private array $middleware;
    private int $index = 0;

    public function __construct(
        array $middleware,
        private readonly RequestHandlerInterface $finalHandler,
    ) {
        $this->middleware = $middleware;
    }

    public function handle(ServerRequestInterface $request): ResponseInterface
    {
        if (!isset($this->middleware[$this->index])) {
            return $this->finalHandler->handle($request);
        }

        $middleware = $this->middleware[$this->index];
        $this->index++;

        return $middleware->process($request, $this);
    }
}
```

---

## ขั้นตอนที่ 2597: PSR-18 HTTP Client

```php
<?php
declare(strict_types=1);

namespace App\Infrastructure\Http;

use Psr\Http\Client\ClientInterface;
use Psr\Http\Message\RequestInterface;
use Psr\Http\Message\ResponseInterface;

class HttpClient implements ClientInterface
{
    private \CurlHandle $curl;

    public function __construct(
        private readonly array $defaultOptions = []
    ) {
        $this->curl = curl_init();
    }

    /**
     * {@inheritdoc}
     */
    public function sendRequest(RequestInterface $request): ResponseInterface
    {
        $options = array_merge([
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HEADER => true,
            CURLOPT_TIMEOUT => 30,
            CURLOPT_SSL_VERIFYPEER => true,
        ], $this->defaultOptions);

        $options[CURLOPT_URL] = (string)$request->getUri();
        $options[CURLOPT_CUSTOMREQUEST] = $request->getMethod();

        // Set headers
        $headers = [];
        foreach ($request->getHeaders() as $name => $values) {
            foreach ($values as $value) {
                $headers[] = "{$name}: {$value}";
            }
        }
        $options[CURLOPT_HTTPHEADER] = $headers;

        // Set body for POST/PUT
        $body = (string)$request->getBody();
        if ($body) {
            $options[CURLOPT_POSTFIELDS] = $body;
        }

        curl_setopt_array($this->curl, $options);
        $response = curl_exec($this->curl);

        if ($response === false) {
            throw new \Psr\Http\Client\NetworkExceptionInterface(
                curl_error($this->curl),
                $request
            );
        }

        $headerSize = curl_getinfo($this->curl, CURLINFO_HEADER_SIZE);
        $headerStr = substr($response, 0, $headerSize);
        $bodyStr = substr($response, $headerSize);
        $statusCode = (int)curl_getinfo($this->curl, CURLINFO_HTTP_CODE);

        return $this->parseResponse($statusCode, $headerStr, $bodyStr);
    }

    private function parseResponse(int $status, string $headers, string $body): ResponseInterface
    {
        return (new Response($status))
            ->withBody(new Stream('php://memory', 'rw', $body));
    }

    public function __destruct()
    {
        curl_close($this->curl);
    }
}
```

---

## สรุปบทที่ 91

| PSR | หัวข้อ | ประโยชน์ |
|-----|--------|---------|
| PSR-1 | Basic Coding | Code readability |
| PSR-3 | Logger Interface | Interchangeable loggers |
| PSR-4 | Autoloading | Class loading standard |
| PSR-7 | HTTP Messages | Framework interop |
| PSR-11 | Container | DI Container standard |
| PSR-12 | Coding Style | Consistent code |
| PSR-15 | HTTP Middleware | Middleware pipeline |
| PSR-18 | HTTP Client | HTTP client interop |

ถัดไป → Part 92: Laravel Modules
