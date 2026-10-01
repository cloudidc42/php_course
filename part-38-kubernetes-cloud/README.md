# Part 38: Kubernetes & Cloud-Native PHP
## ขั้นตอนที่ 1001-1030: Deploy ระดับ Enterprise

---

## ขั้นตอนที่ 1001: Kubernetes Deployment

```yaml
# kubernetes/deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: v1.0.0
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime
  template:
    metadata:
      labels:
        app: myapp
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9000"
    spec:
      containers:
        - name: php-fpm
          image: registry.example.com/myapp:1.0.0
          ports:
            - containerPort: 9000
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          env:
            - name: APP_ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-host
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-password
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: redis-url
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          volumeMounts:
            - name: app-config
              mountPath: /var/www/html/.env
              subPath: .env
              readOnly: true
      
        - name: nginx
          image: nginx:1.25-alpine
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d/
      
      volumes:
        - name: app-config
          secret:
            secretName: myapp-env
        - name: nginx-config
          configMap:
            name: nginx-config
      
      imagePullSecrets:
        - name: registry-credentials
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [myapp]
                topologyKey: kubernetes.io/hostname
```

```yaml
# kubernetes/hpa.yaml - Horizontal Pod Autoscaler

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    # Custom metric: requests per second
    - type: Pods
      pods:
        metric:
          name: php_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

---

## ขั้นตอนที่ 1002: Health Checks & Graceful Shutdown

```php
<?php
declare(strict_types=1);

// Health check endpoint
Route::get('/health', function(): JsonResponse {
    $checks = [];
    $status = 'healthy';
    $code = 200;
    
    // Database check
    try {
        DB::connection()->getPdo();
        $checks['database'] = 'ok';
    } catch (\Exception $e) {
        $checks['database'] = 'fail';
        $status = 'unhealthy';
        $code = 503;
    }
    
    // Redis check
    try {
        Cache::store('redis')->put('health_check', 1, 1);
        $checks['redis'] = 'ok';
    } catch (\Exception $e) {
        $checks['redis'] = 'fail';
        $status = 'degraded';
    }
    
    // Queue check
    try {
        $failed = DB::table('failed_jobs')->count();
        $checks['queue'] = $failed < 100 ? 'ok' : 'degraded';
    } catch (\Exception $e) {
        $checks['queue'] = 'unknown';
    }
    
    // Memory check
    $memory_mb = memory_get_usage(true) / 1024 / 1024;
    $checks['memory'] = $memory_mb < 400 ? 'ok' : 'high';
    
    return response()->json([
        'status'    => $status,
        'version'   => config('app.version', '1.0.0'),
        'timestamp' => now()->toIso8601String(),
        'checks'    => $checks,
        'memory_mb' => round($memory_mb, 2),
    ], $code);
})->name('health');

// Readiness check (ready to accept traffic)
Route::get('/ready', function(): JsonResponse {
    // Check if app can serve requests
    try {
        DB::select('SELECT 1');
        return response()->json(['status' => 'ready']);
    } catch (\Exception $e) {
        return response()->json(['status' => 'not_ready'], 503);
    }
});

// Graceful shutdown handling
class GracefulShutdownHandler {
    private bool $stopping = false;
    
    public function __construct() {
        pcntl_signal(SIGTERM, [$this, 'handleSignal']);
        pcntl_signal(SIGINT,  [$this, 'handleSignal']);
    }
    
    public function handleSignal(int $signal): void {
        $this->stopping = true;
        
        // Stop accepting new jobs
        if (app()->bound('queue')) {
            // Signal queue worker to stop after current job
            file_put_contents(storage_path('framework/down'), '');
        }
        
        Log::info('Graceful shutdown initiated', ['signal' => $signal]);
    }
    
    public function isStopping(): bool {
        return $this->stopping;
    }
}
```

---

## ขั้นตอนที่ 1003: Observability (Metrics, Logs, Traces)

```php
<?php
declare(strict_types=1);

// OpenTelemetry tracing
// composer require open-telemetry/sdk open-telemetry/exporter-otlp

use OpenTelemetry\API\Globals;
use OpenTelemetry\API\Trace\SpanKind;

class TracingMiddleware {
    public function handle(Request $request, Closure $next): Response {
        $tracer = Globals::tracerProvider()->getTracer('myapp');
        
        $span = $tracer->spanBuilder($request->method() . ' ' . $request->path())
            ->setSpanKind(SpanKind::KIND_SERVER)
            ->startSpan();
        
        $span->setAttribute('http.method', $request->method());
        $span->setAttribute('http.url', $request->url());
        $span->setAttribute('http.user_agent', $request->userAgent() ?? '');
        $span->setAttribute('user.id', auth()->id() ?? 0);
        
        $context = $span->activate();
        
        try {
            $response = $next($request);
            
            $span->setAttribute('http.status_code', $response->getStatusCode());
            
            return $response;
        } catch (\Exception $e) {
            $span->recordException($e);
            $span->setStatus(\OpenTelemetry\API\Trace\StatusCode::STATUS_ERROR);
            throw $e;
        } finally {
            $context->detach();
            $span->end();
        }
    }
}

// Prometheus metrics
// composer require promphp/prometheus_client_php

use Prometheus\CollectorRegistry;
use Prometheus\Storage\Redis as PrometheusRedis;

class MetricsController {
    private CollectorRegistry $registry;
    
    public function __construct() {
        $adapter = new PrometheusRedis(['host' => config('database.redis.default.host')]);
        $this->registry = new CollectorRegistry($adapter);
    }
    
    public function metrics(): Response {
        $renderer = new \Prometheus\RenderTextFormat();
        $result = $renderer->render($this->registry->getMetricFamilySamples());
        
        return response($result, 200, ['Content-Type' => \Prometheus\RenderTextFormat::MIME_TYPE]);
    }
    
    public function recordRequest(string $route, string $method, int $status, float $duration): void {
        // Counter
        $requestCounter = $this->registry->getOrRegisterCounter(
            'myapp',
            'http_requests_total',
            'Total HTTP requests',
            ['route', 'method', 'status']
        );
        $requestCounter->incBy(1, [$route, $method, (string) $status]);
        
        // Histogram
        $durationHistogram = $this->registry->getOrRegisterHistogram(
            'myapp',
            'http_request_duration_seconds',
            'HTTP request duration',
            ['route', 'method'],
            [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
        );
        $durationHistogram->observe($duration, [$route, $method]);
    }
}
```

---

## ขั้นตอนที่ 1004: Serverless PHP (AWS Lambda)

```php
<?php
declare(strict_types=1);

// composer require bref/bref
// serverless.yml

/*
service: myapp
provider:
  name: aws
  region: ap-southeast-1
  environment:
    APP_ENV: production
    DB_HOST: ${ssm:/myapp/db-host}
    DB_PASSWORD: ${ssm:/myapp/db-password~true}

functions:
  web:
    handler: public/index.php
    runtime: provided.al2
    timeout: 28
    memorySize: 1024
    layers:
      - ${bref:layer.php-83-fpm}
    events:
      - httpApi: '*'
  
  worker:
    handler: artisan
    runtime: provided.al2
    timeout: 900
    layers:
      - ${bref:layer.php-83}
    events:
      - schedule:
          rate: rate(1 minute)
          input: '"schedule:run"'

plugins:
  - ./vendor/bref/bref
*/

// bootstrap/app.php - adapt for Lambda
// Bref handles the HTTP adapter automatically

// Handle cold starts
if (isset($_SERVER['LAMBDA_TASK_ROOT'])) {
    // Warm up connections
    DB::connection()->getPdo();
    
    // Disable STDOUT buffering
    ini_set('output_buffering', 'off');
}

// Stateless considerations for Lambda:
// 1. No local file storage (use S3)
// 2. No persistent connections (use RDS Proxy)
// 3. No cron in container (use CloudWatch Events)
// 4. Sessions must be external (Redis/DynamoDB)
```

---

## 🎯 สรุป Part 38

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Kubernetes | Deployment, HPA, Rolling updates |
| Health Checks | /health, /ready endpoints |
| Graceful Shutdown | SIGTERM handling |
| Observability | OpenTelemetry, Prometheus metrics |
| Serverless | AWS Lambda with Bref |
| Cloud-Native | 12-factor app principles |

**หลักสูตรครบสมบูรณ์! คุณพร้อมสำหรับงานระดับมืออาชีพแล้ว 🎉**
