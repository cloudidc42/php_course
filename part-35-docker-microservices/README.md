# Part 35: Docker & PHP Microservices
## ขั้นตอนที่ 901-940: Architecture ระดับ Enterprise

---

## ขั้นตอนที่ 901: Docker for PHP Development

```dockerfile
# Dockerfile.dev - Development image

FROM php:8.3-fpm-alpine

WORKDIR /var/www/html

# Install dependencies
RUN apk add --no-cache \
    git \
    curl \
    libpng-dev \
    libxml2-dev \
    zip \
    unzip \
    linux-headers

# Install PHP extensions
RUN docker-php-ext-install \
    pdo_mysql \
    mbstring \
    exif \
    pcntl \
    bcmath \
    gd \
    opcache

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Install Xdebug for development
RUN pecl install xdebug && docker-php-ext-enable xdebug

# Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# PHP config
COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini
COPY docker/php/xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini

# Non-root user
RUN addgroup -g 1000 appuser && adduser -D -u 1000 -G appuser appuser
USER appuser

EXPOSE 9000
```

```dockerfile
# Dockerfile.prod - Production image (multi-stage)

# Stage 1: Composer install
FROM composer:2 AS composer
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-interaction

# Stage 2: Node build
FROM node:20-alpine AS node-build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY resources/ ./resources/
COPY vite.config.js ./
RUN npm run build

# Stage 3: Production PHP
FROM php:8.3-fpm-alpine AS production

WORKDIR /var/www/html

RUN apk add --no-cache libpng-dev libxml2-dev

RUN docker-php-ext-install pdo_mysql mbstring gd opcache pcntl bcmath

RUN pecl install redis && docker-php-ext-enable redis

# Copy PHP configuration
COPY docker/php/php-prod.ini /usr/local/etc/php/conf.d/custom.ini
COPY docker/php/opcache.ini /usr/local/etc/php/conf.d/opcache.ini

# Copy application
COPY --chown=www-data:www-data . .
COPY --from=composer /app/vendor ./vendor
COPY --from=node-build /app/public/build ./public/build

RUN chmod -R 755 storage bootstrap/cache

USER www-data

EXPOSE 9000
```

---

## ขั้นตอนที่ 902: Docker Compose

```yaml
# docker-compose.yml

version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: myapp
    restart: unless-stopped
    working_dir: /var/www/html
    volumes:
      - .:/var/www/html:cached
      - vendor:/var/www/html/vendor
    networks:
      - myapp-network
    depends_on:
      - mysql
      - redis
    environment:
      - PHP_IDE_CONFIG=serverName=myapp
      - XDEBUG_CONFIG=client_host=host.docker.internal
    
  nginx:
    image: nginx:1.25-alpine
    container_name: myapp-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - .:/var/www/html:cached
      - ./docker/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./docker/nginx/sites:/etc/nginx/conf.d:ro
    networks:
      - myapp-network
    depends_on:
      - app
    
  mysql:
    image: mysql:8.0
    container_name: myapp-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-secret}
      MYSQL_DATABASE: ${DB_DATABASE:-myapp}
      MYSQL_USER: ${DB_USERNAME:-myapp}
      MYSQL_PASSWORD: ${DB_PASSWORD:-secret}
    volumes:
      - mysql-data:/var/lib/mysql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf:ro
    networks:
      - myapp-network
    
  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: redis-server --requirepass ${REDIS_PASSWORD:-secret} --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - myapp-network
    
  mailhog:
    image: mailhog/mailhog:latest
    container_name: myapp-mailhog
    ports:
      - "1025:1025"  # SMTP
      - "8025:8025"  # Web UI
    networks:
      - myapp-network
    
  queue-worker:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: myapp-queue
    restart: unless-stopped
    working_dir: /var/www/html
    command: php artisan queue:work redis --sleep=3 --tries=3
    volumes:
      - .:/var/www/html:cached
    networks:
      - myapp-network
    depends_on:
      - mysql
      - redis
    
  scheduler:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: myapp-scheduler
    restart: unless-stopped
    working_dir: /var/www/html
    command: /bin/sh -c "while true; do php artisan schedule:run; sleep 60; done"
    volumes:
      - .:/var/www/html:cached
    networks:
      - myapp-network

networks:
  myapp-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
  vendor:
```

---

## ขั้นตอนที่ 903: Microservices Architecture

```php
<?php
declare(strict_types=1);

// API Gateway Pattern - routes to microservices

namespace ApiGateway;

use GuzzleHttp\Client;
use GuzzleHttp\Promise\Utils;
use GuzzleHttp\Exception\RequestException;

class ServiceGateway {
    private array $services = [
        'users'    => 'http://user-service:3001',
        'orders'   => 'http://order-service:3002',
        'products' => 'http://product-service:3003',
        'payments' => 'http://payment-service:3004',
    ];
    
    public function __construct(private Client $http) {}
    
    public function getOrderDashboard(int $userId): array {
        // Concurrent requests to multiple services
        $promises = [
            'user'    => $this->http->getAsync("{$this->services['users']}/api/users/{$userId}"),
            'orders'  => $this->http->getAsync("{$this->services['orders']}/api/orders?user_id={$userId}&limit=5"),
            'balance' => $this->http->getAsync("{$this->services['payments']}/api/balance/{$userId}"),
        ];
        
        // Wait for all - with error handling
        $results = Utils::settle($promises)->wait();
        
        $dashboard = [];
        
        foreach ($results as $key => $result) {
            if ($result['state'] === 'fulfilled') {
                $dashboard[$key] = json_decode($result['value']->getBody(), true);
            } else {
                $dashboard[$key] = null;
                // Log circuit breaker or retry
            }
        }
        
        return $dashboard;
    }
    
    public function placeOrder(array $data): array {
        // Saga pattern - distributed transaction
        $saga = new OrderSaga($this);
        return $saga->execute($data);
    }
}

// Saga Pattern for distributed transactions
class OrderSaga {
    private array $compensations = [];
    
    public function __construct(private ServiceGateway $gateway) {}
    
    public function execute(array $data): array {
        try {
            // Step 1: Reserve inventory
            $reservation = $this->reserveInventory($data['items']);
            $this->compensations[] = fn() => $this->releaseInventory($reservation['id']);
            
            // Step 2: Process payment
            $payment = $this->processPayment($data['payment']);
            $this->compensations[] = fn() => $this->refundPayment($payment['id']);
            
            // Step 3: Create order
            $order = $this->createOrder($data, $reservation['id'], $payment['id']);
            
            // Step 4: Confirm reservation
            $this->confirmInventory($reservation['id'], $order['id']);
            
            return $order;
            
        } catch (\Exception $e) {
            // Compensate in reverse order
            foreach (array_reverse($this->compensations) as $compensate) {
                try {
                    $compensate();
                } catch (\Exception $compensationError) {
                    // Log compensation failure - needs manual intervention
                }
            }
            throw $e;
        }
    }
    
    private function reserveInventory(array $items): array {
        // Call inventory service
        return ['id' => 'res_123', 'items' => $items];
    }
    
    private function releaseInventory(string $reservationId): void {
        // Reverse step 1
    }
    
    private function processPayment(array $paymentData): array {
        return ['id' => 'pay_456', 'status' => 'captured'];
    }
    
    private function refundPayment(string $paymentId): void {
        // Reverse step 2
    }
    
    private function createOrder(array $data, string $reservationId, string $paymentId): array {
        return ['id' => 'ord_789', 'status' => 'confirmed'];
    }
    
    private function confirmInventory(string $reservationId, string $orderId): void {}
}
```

---

## ขั้นตอนที่ 904: Message Queue (RabbitMQ/SQS)

```php
<?php
declare(strict_types=1);

// RabbitMQ with php-amqplib
// composer require php-amqplib/php-amqplib

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class MessageBroker {
    private \PhpAmqpLib\Channel\AMQPChannel $channel;
    
    public function __construct(
        private AMQPStreamConnection $connection
    ) {
        $this->channel = $this->connection->channel();
    }
    
    public function publish(string $exchange, string $routingKey, array $data): void {
        // Declare exchange
        $this->channel->exchange_declare($exchange, 'topic', false, true, false);
        
        $message = new AMQPMessage(
            json_encode($data),
            [
                'content_type'  => 'application/json',
                'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
                'correlation_id'=> uniqid('', true),
                'timestamp'     => time(),
            ]
        );
        
        $this->channel->basic_publish($message, $exchange, $routingKey);
    }
    
    public function subscribe(string $exchange, string $routingKey, callable $handler): void {
        $queue = "{$exchange}.{$routingKey}";
        
        $this->channel->exchange_declare($exchange, 'topic', false, true, false);
        $this->channel->queue_declare($queue, false, true, false, false);
        $this->channel->queue_bind($queue, $exchange, $routingKey);
        
        $this->channel->basic_qos(null, 10, null); // Prefetch count
        
        $this->channel->basic_consume(
            $queue,
            '',
            false, // no-local
            false, // no-ack
            false, // exclusive
            false, // nowait
            function(AMQPMessage $message) use ($handler): void {
                try {
                    $data = json_decode($message->body, true);
                    $handler($data, $message);
                    $message->ack(); // Acknowledge success
                    
                } catch (\Exception $e) {
                    // Reject and requeue if retriable
                    if ($this->isRetriable($e)) {
                        $message->nack(true); // Requeue
                    } else {
                        $message->nack(false); // Dead letter
                    }
                }
            }
        );
        
        // Start consuming
        while ($this->channel->is_consuming()) {
            $this->channel->wait(null, false, 0); // Non-blocking
        }
    }
    
    private function isRetriable(\Exception $e): bool {
        return $e instanceof \RuntimeException &&
               str_contains($e->getMessage(), 'temporary');
    }
    
    public function __destruct() {
        $this->channel->close();
        $this->connection->close();
    }
}

// Usage: Order placed event
$broker = new MessageBroker(new AMQPStreamConnection('rabbitmq', 5672, 'user', 'pass'));

// Publisher (Order Service)
$broker->publish('orders', 'order.placed', [
    'order_id' => 123,
    'user_id'  => 456,
    'total'    => 1500.00,
    'items'    => [['product_id' => 1, 'qty' => 2]],
]);

// Consumer (Email Service)
$broker->subscribe('orders', 'order.placed', function(array $data): void {
    sendOrderConfirmationEmail($data['order_id'], $data['user_id']);
});

// Consumer (Inventory Service)
$broker->subscribe('orders', 'order.placed', function(array $data): void {
    updateInventory($data['items']);
});
```

---

## 🎯 สรุป Part 35

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Docker | Multi-stage build, dev vs prod images |
| Docker Compose | Full stack: PHP, Nginx, MySQL, Redis |
| Microservices | API Gateway, service discovery |
| Saga Pattern | Distributed transactions |
| Message Queue | RabbitMQ, pub/sub, ack/nack |
| Async Requests | Guzzle promises, concurrent calls |

**ถัดไป → Part 36: GraphQL API ด้วย PHP**
