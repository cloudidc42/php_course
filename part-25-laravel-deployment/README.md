# Part 25: Laravel Production Deployment
## ขั้นตอนที่ 631-650: Deploy อย่างมืออาชีพ

---

## ขั้นตอนที่ 631: Server Requirements & Setup

```bash
# Ubuntu 22.04 Setup

# Update system
apt update && apt upgrade -y

# Install PHP 8.3
apt install -y software-properties-common
add-apt-repository ppa:ondrej/php
apt update
apt install -y php8.3-fpm php8.3-cli php8.3-common \
    php8.3-mysql php8.3-redis php8.3-mbstring \
    php8.3-xml php8.3-curl php8.3-zip php8.3-gd \
    php8.3-bcmath php8.3-opcache php8.3-intl

# Install Nginx
apt install -y nginx

# Install MySQL 8.0
apt install -y mysql-server
mysql_secure_installation

# Install Redis
apt install -y redis-server
systemctl enable redis-server

# Install Composer
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer

# Install Node.js (for asset building)
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# Create web user
useradd -m -s /bin/bash webdev
```

---

## ขั้นตอนที่ 632: Nginx Configuration

```nginx
# /etc/nginx/sites-available/myapp.com

server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;
    
    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.com www.myapp.com;
    
    root /var/www/myapp/public;
    index index.php;
    
    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_stapling on;
    ssl_stapling_verify on;
    
    # Security Headers
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload";
    add_header Referrer-Policy strict-origin-when-cross-origin;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' cdn.example.com; style-src 'self' 'unsafe-inline' fonts.googleapis.com;";
    
    # Gzip Compression
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
    gzip_min_length 1000;
    gzip_comp_level 6;
    
    # Max upload size
    client_max_body_size 20M;
    
    charset utf-8;
    
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    # PHP-FPM
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 300;
        
        # Security
        fastcgi_param SERVER_NAME $host;
        fastcgi_hide_header X-Powered-By;
    }
    
    # Static Files Cache
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # Deny access to sensitive files
    location ~ /\. { deny all; }
    location ~ /\.env { deny all; }
    location ~* \.(sh|sql|log|bak)$ { deny all; }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
    location /login {
        limit_req zone=login burst=5 nodelay;
        try_files $uri $uri/ /index.php?$query_string;
    }
    
    access_log /var/log/nginx/myapp.access.log;
    error_log /var/log/nginx/myapp.error.log;
}
```

---

## ขั้นตอนที่ 633: Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -e  # Exit on error

APP_DIR="/var/www/myapp"
SHARED_DIR="/var/www/myapp-shared"
RELEASES_DIR="/var/www/myapp-releases"
TIMESTAMP=$(date +%Y%m%d%H%M%S)
RELEASE_DIR="$RELEASES_DIR/$TIMESTAMP"
KEEP_RELEASES=5

echo "=== Starting deployment ==="

# Create release directory
mkdir -p $RELEASE_DIR

# Clone/pull repository
git clone --depth=1 --branch=main git@github.com:myorg/myapp.git $RELEASE_DIR

# Symlink shared files
ln -sfn $SHARED_DIR/.env $RELEASE_DIR/.env
ln -sfn $SHARED_DIR/storage $RELEASE_DIR/storage

# Install dependencies (no dev, optimized)
cd $RELEASE_DIR
composer install --no-dev --optimize-autoloader --no-interaction

# Build assets
npm ci
npm run build

# Run migrations (with maintenance mode)
php artisan down --refresh=15
php artisan migrate --force
php artisan db:seed --class=ProductionSeeder --force 2>/dev/null || true

# Clear and cache
php artisan cache:clear
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan event:cache
php artisan icons:cache 2>/dev/null || true

# Set permissions
chmod -R 755 $RELEASE_DIR
chmod -R 775 $RELEASE_DIR/storage
chown -R www-data:www-data $RELEASE_DIR

# Switch to new release
ln -sfn $RELEASE_DIR $APP_DIR

# Restart services
php artisan up
php artisan queue:restart
sudo systemctl reload php8.3-fpm
sudo systemctl reload nginx

# Cleanup old releases
cd $RELEASES_DIR
ls -t | tail -n +$((KEEP_RELEASES + 1)) | xargs rm -rf

echo "=== Deployment complete: $TIMESTAMP ==="
```

---

## ขั้นตอนที่ 634: PHP-FPM & OPcache Configuration

```ini
; /etc/php/8.3/fpm/pool.d/www.conf
[www]
user = www-data
group = www-data
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 10
pm.max_requests = 500

; Request timeout
request_terminate_timeout = 60

; /etc/php/8.3/fpm/conf.d/10-opcache.ini
opcache.enable = 1
opcache.enable_cli = 0
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 16
opcache.max_accelerated_files = 20000
opcache.revalidate_freq = 0  ; Never check in production
opcache.validate_timestamps = 0  ; Never check in production
opcache.max_wasted_percentage = 5
opcache.fast_shutdown = 1
opcache.enable_file_override = 0
opcache.jit = tracing
opcache.jit_buffer_size = 64M
```

---

## ขั้นตอนที่ 635: Supervisor (Queue Workers)

```ini
; /etc/supervisor/conf.d/myapp-worker.conf
[program:myapp-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/myapp/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/log/myapp/worker.log
stopwaitsecs=3600

[program:myapp-horizon]
process_name=%(program_name)s
command=php /var/www/myapp/artisan horizon
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/myapp/horizon.log
stopwaitsecs=3600
```

```bash
supervisorctl reread
supervisorctl update
supervisorctl start myapp-worker:*
supervisorctl status
```

---

## ขั้นตอนที่ 636: GitHub Actions CI/CD

```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_DATABASE: testing
          MYSQL_USER: user
          MYSQL_PASSWORD: password
          MYSQL_ROOT_PASSWORD: root
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
          extensions: mbstring, pdo_mysql, redis
          coverage: xdebug
      
      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
      
      - name: Install dependencies
        run: composer install --no-progress --prefer-dist --optimize-autoloader
      
      - name: Copy .env
        run: cp .env.testing .env
      
      - name: Generate key
        run: php artisan key:generate
      
      - name: Run migrations
        run: php artisan migrate
        env:
          DB_CONNECTION: mysql
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_DATABASE: testing
          DB_USERNAME: user
          DB_PASSWORD: password
      
      - name: Run tests
        run: php artisan test --parallel --coverage-clover=coverage.xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml
      
      - name: Run PHPStan
        run: ./vendor/bin/phpstan analyse --no-progress
      
      - name: Run PHP CS Fixer
        run: ./vendor/bin/php-cs-fixer fix --dry-run --diff
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/myapp
            ./deploy.sh
      
      - name: Notify Slack
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_MESSAGE: ':rocket: Deployed to production successfully!'
```

---

## ขั้นตอนที่ 637: Monitoring & Performance

```php
<?php
// Sentry Integration
// composer require sentry/sentry-laravel

// config/sentry.php
return [
    'dsn' => env('SENTRY_LARAVEL_DSN'),
    'traces_sample_rate' => 0.1, // Sample 10% of transactions
    'profiles_sample_rate' => 0.1,
];

// bootstrap/app.php
->withExceptions(function (Exceptions $exceptions) {
    \Sentry\Laravel\Integration::handles($exceptions);
})

// Custom breadcrumbs
\Sentry\addBreadcrumb(new \Sentry\Breadcrumb(
    \Sentry\Breadcrumb::LEVEL_INFO,
    \Sentry\Breadcrumb::TYPE_DEFAULT,
    'auth',
    'User logged in',
    ['user_id' => auth()->id()]
));

// Performance Monitoring
\Sentry\startTransaction(new \Sentry\Tracing\TransactionContext());
```

```bash
# Install New Relic (alternative)
# curl -Ls https://download.newrelic.com/install/newrelic-cli/scripts/install.sh | bash

# Health Check Endpoint
Route::get('/health', function () {
    return response()->json([
        'status' => 'healthy',
        'version' => config('app.version'),
        'timestamp' => now()->toIso8601String(),
        'checks' => [
            'database' => DB::connection()->getPdo() ? 'ok' : 'fail',
            'cache' => Cache::put('health_check', true, 1) ? 'ok' : 'fail',
            'queue' => Queue::size() < 1000 ? 'ok' : 'degraded',
        ],
    ]);
})->name('health');
```

---

## 🎯 สรุป Part 25

| หัวข้อ | สิ่งที่ได้เรียน |
|--------|----------------|
| Server Setup | Ubuntu, Nginx, PHP-FPM, MySQL |
| Nginx Config | SSL, Security headers, Rate limiting |
| Deploy Script | Zero-downtime deployment |
| PHP Config | OPcache tuning, FPM pools |
| Supervisor | Queue workers as services |
| CI/CD | GitHub Actions, test, deploy |
| Monitoring | Sentry, health checks |

**ถัดไป → Part 27: WordPress Theme ขั้นสูง**
