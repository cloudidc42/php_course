# Part 70: Advanced CI/CD

## ขั้นตอนที่ 1961-1990: Advanced CI/CD Pipelines

CI/CD pipeline ที่ดีทำให้ deploy ได้เร็ว ปลอดภัย และ rollback ได้ง่าย

---

## ขั้นตอนที่ 1961: GitHub Actions - Matrix Build

```yaml
# .github/workflows/test.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      fail-fast: false
      matrix:
        php-version: ['8.2', '8.3']
        db: ['mysql:8.0', 'mysql:5.7', 'pgsql:15']
        include:
          - php-version: '8.3'
            db: 'mysql:8.0'
            coverage: true

    services:
      mysql:
        image: ${{ matrix.db == 'pgsql:15' && 'postgres:15' || matrix.db }}
        env:
          MYSQL_DATABASE: testing
          MYSQL_USER: root
          MYSQL_PASSWORD: secret
          MYSQL_ROOT_PASSWORD: secret
          POSTGRES_DB: testing
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: secret
        ports:
          - 3306:3306
          - 5432:5432
        options: >-
          --health-cmd="${{ matrix.db == 'pgsql:15' && 'pg_isready' || 'mysqladmin ping' }}"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: --health-cmd="redis-cli ping" --health-interval 10s --health-timeout 5s --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup PHP ${{ matrix.php-version }}
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ matrix.php-version }}
          extensions: pdo, pdo_mysql, pdo_pgsql, redis, pcntl, bcmath
          coverage: ${{ matrix.coverage && 'xdebug' || 'none' }}

      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: vendor
          key: ${{ runner.os }}-php-${{ matrix.php-version }}-${{ hashFiles('**/composer.lock') }}
          restore-keys: |
            ${{ runner.os }}-php-${{ matrix.php-version }}-

      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist --optimize-autoloader

      - name: Setup environment
        run: |
          cp .env.ci .env
          php artisan key:generate
          php artisan config:cache

      - name: Run database migrations
        run: php artisan migrate --force

      - name: Run tests
        run: |
          if [ "${{ matrix.coverage }}" = "true" ]; then
            ./vendor/bin/phpunit --coverage-clover coverage.xml
          else
            ./vendor/bin/phpunit
          fi

      - name: Upload coverage
        if: matrix.coverage == true
        uses: codecov/codecov-action@v3
        with:
          files: coverage.xml
```

---

## ขั้นตอนที่ 1962: Automated Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  # Stage 1: Build and test
  build:
    uses: ./.github/workflows/test.yml

  # Stage 2: Build Docker image
  docker:
    needs: build
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}

    steps:
      - uses: actions/checkout@v4

      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=,suffix=,format=short
            type=ref,event=branch
            latest

      - name: Login to registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Stage 3: Deploy to staging
  deploy-staging:
    needs: docker
    runs-on: ubuntu-latest
    environment: staging
    concurrency: staging

    steps:
      - name: Deploy to staging
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: deploy
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            cd /var/www/myapp-staging
            docker pull ghcr.io/${{ github.repository }}:${{ github.sha }}
            docker-compose up -d --no-deps app
            docker-compose exec app php artisan migrate --force
            docker-compose exec app php artisan config:cache
            docker-compose exec app php artisan route:cache

  # Stage 4: Run E2E tests against staging
  e2e:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run E2E tests
        run: |
          npm ci
          npx playwright test --reporter=html
        env:
          BASE_URL: https://staging.myapp.com

      - uses: actions/upload-artifact@v3
        if: always()
        with:
          name: playwright-report
          path: playwright-report/

  # Stage 5: Deploy to production (manual approval required)
  deploy-production:
    needs: e2e
    runs-on: ubuntu-latest
    environment: production  # Requires manual approval in GitHub
    concurrency: production

    steps:
      - name: Deploy to production
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.PROD_HOST }}
          username: deploy
          key: ${{ secrets.DEPLOY_KEY }}
          script: |
            cd /var/www/myapp
            ./scripts/deploy.sh ${{ github.sha }}
```

---

## ขั้นตอนที่ 1963: Deploy Script กับ Zero-Downtime

```bash
#!/bin/bash
# scripts/deploy.sh
set -euo pipefail

IMAGE_TAG="${1:-latest}"
APP_DIR="/var/www/myapp"
RELEASES_DIR="${APP_DIR}/releases"
SHARED_DIR="${APP_DIR}/shared"
CURRENT_LINK="${APP_DIR}/current"

RELEASE_DIR="${RELEASES_DIR}/$(date +%Y%m%d%H%M%S)"

echo "=== Deploying ${IMAGE_TAG} ==="

# สร้าง release directory
mkdir -p "${RELEASE_DIR}"

# Pull Docker image
docker pull "ghcr.io/myorg/myapp:${IMAGE_TAG}"

# Setup shared files/dirs
ln -nfs "${SHARED_DIR}/.env" "${RELEASE_DIR}/.env"
ln -nfs "${SHARED_DIR}/storage" "${RELEASE_DIR}/storage"

# Run migrations (ก่อน switch)
docker run --rm \
  --env-file "${SHARED_DIR}/.env" \
  "ghcr.io/myorg/myapp:${IMAGE_TAG}" \
  php artisan migrate --force

# Cache config/routes
docker run --rm \
  --env-file "${SHARED_DIR}/.env" \
  "ghcr.io/myorg/myapp:${IMAGE_TAG}" \
  bash -c "php artisan config:cache && php artisan route:cache && php artisan view:cache"

# Atomic switch
ln -nsfT "${RELEASE_DIR}" "${CURRENT_LINK}"

# Restart services
docker-compose -f "${APP_DIR}/docker-compose.yml" up -d --no-deps app

# Health check
for i in {1..30}; do
    if curl -sf "https://myapp.com/health" > /dev/null; then
        echo "Health check passed!"
        break
    fi
    if [ $i -eq 30 ]; then
        echo "Health check failed! Rolling back..."
        ./scripts/rollback.sh
        exit 1
    fi
    sleep 2
done

# Cleanup old releases (keep last 5)
ls -dt "${RELEASES_DIR}"/* | tail -n +6 | xargs rm -rf

echo "=== Deploy complete ==="
```

---

## ขั้นตอนที่ 1964: Blue-Green Deployment

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx:/etc/nginx/conf.d
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - blue
      - green

  blue:
    image: ghcr.io/myorg/myapp:${BLUE_TAG:-latest}
    env_file: .env.blue
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  green:
    image: ghcr.io/myorg/myapp:${GREEN_TAG:-latest}
    env_file: .env.green
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

```bash
#!/bin/bash
# scripts/blue-green-deploy.sh

ACTIVE_COLOR=$(cat /var/www/active-color 2>/dev/null || echo "blue")
NEW_COLOR=$([ "$ACTIVE_COLOR" = "blue" ] && echo "green" || echo "blue")

echo "Active: ${ACTIVE_COLOR}, Deploying to: ${NEW_COLOR}"

# Update inactive color
echo "${NEW_TAG}" > "/var/www/${NEW_COLOR}-tag"
docker-compose pull "${NEW_COLOR}"
docker-compose up -d --no-deps "${NEW_COLOR}"

# Wait for health check
echo "Waiting for ${NEW_COLOR} to be healthy..."
for i in {1..60}; do
    if docker-compose exec "${NEW_COLOR}" curl -sf http://localhost/health > /dev/null 2>&1; then
        echo "${NEW_COLOR} is healthy!"
        break
    fi
    if [ $i -eq 60 ]; then
        echo "${NEW_COLOR} failed health check!"
        exit 1
    fi
    sleep 2
done

# Run migrations ก่อน switch traffic
docker-compose exec "${NEW_COLOR}" php artisan migrate --force

# Switch Nginx upstream
sed -i "s/server ${ACTIVE_COLOR}:80/server ${NEW_COLOR}:80/g" /etc/nginx/conf.d/upstream.conf
nginx -s reload

# Verify switch
sleep 5
if curl -sf https://myapp.com/health | grep -q '"status":"ok"'; then
    echo "Switch successful!"
    echo "${NEW_COLOR}" > /var/www/active-color
else
    echo "Switch failed! Rolling back to ${ACTIVE_COLOR}..."
    sed -i "s/server ${NEW_COLOR}:80/server ${ACTIVE_COLOR}:80/g" /etc/nginx/conf.d/upstream.conf
    nginx -s reload
    exit 1
fi
```

---

## ขั้นตอนที่ 1965: Canary Releases

```nginx
# nginx upstream with weights for canary
upstream myapp {
    # 95% traffic -> stable
    server stable.internal:80 weight=95;
    # 5% traffic -> canary
    server canary.internal:80 weight=5;
}

# Cookie-based canary
map $cookie_canary $backend {
    default stable.internal:80;
    "true"  canary.internal:80;
}

server {
    location / {
        proxy_pass http://$backend;
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/CanaryMiddleware.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class CanaryMiddleware
{
    private const CANARY_COOKIE = 'canary_user';
    private const CANARY_PERCENTAGE = 5; // 5% of users

    public function handle(Request $request, Closure $next): Response
    {
        $user     = $request->user();
        $isCanary = $this->isCanaryUser($request, $user);

        // Tag request
        $request->headers->set('X-Canary', $isCanary ? 'true' : 'false');

        $response = $next($request);

        // Set cookie สำหรับ sticky canary
        if ($isCanary) {
            $response->cookie(self::CANARY_COOKIE, '1', 60 * 24 * 7);
        }

        return $response;
    }

    private function isCanaryUser(Request $request, $user): bool
    {
        // ถ้ามี cookie แล้ว ให้ sticky
        if ($request->cookie(self::CANARY_COOKIE)) return true;

        // Feature flag override
        if ($user && \App\Models\CanaryUser::where('user_id', $user->id)->exists()) {
            return true;
        }

        // Random percentage
        return random_int(1, 100) <= self::CANARY_PERCENTAGE;
    }
}
```

---

## ขั้นตอนที่ 1966: Rollback Strategy

```bash
#!/bin/bash
# scripts/rollback.sh
set -euo pipefail

RELEASES_DIR="/var/www/myapp/releases"
CURRENT_LINK="/var/www/myapp/current"

# Get current and previous release
CURRENT=$(readlink -f "${CURRENT_LINK}")
PREVIOUS=$(ls -dt "${RELEASES_DIR}"/* | sed -n '2p')

if [ -z "${PREVIOUS}" ]; then
    echo "No previous release to rollback to!"
    exit 1
fi

echo "Rolling back from: ${CURRENT}"
echo "Rolling back to:   ${PREVIOUS}"

# Atomic switch to previous release
ln -nsfT "${PREVIOUS}" "${CURRENT_LINK}"

# Restart services
docker-compose restart app

# Health check
if curl -sf https://myapp.com/health > /dev/null; then
    echo "Rollback successful!"

    # Optionally rollback migrations (DANGER!)
    # docker-compose exec app php artisan migrate:rollback --step=1
else
    echo "Rollback health check failed!"
    exit 1
fi
```

---

## สรุป Part 70

| Strategy | รายละเอียด | Downtime |
|---------|-----------|---------|
| Rolling Deploy | ค่อยๆ replace instances | Minimal |
| Blue-Green | Switch traffic เร็ว | Zero |
| Canary | ปล่อย % ของ users | Zero |
| Feature Flags | Toggle features | Zero |
| Matrix Build | Test multiple versions | - |
| Health Checks | ตรวจสอบก่อน switch | - |

ถัดไป → Part 71: Security Audit
