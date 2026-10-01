# Part 61: PHP-FPM และ Nginx

## ขั้นตอนที่ 1691-1720: PHP-FPM Configuration และ Nginx Optimization

การ tune PHP-FPM และ Nginx อย่างถูกต้องสามารถเพิ่มประสิทธิภาพได้หลายเท่า

---

## ขั้นตอนที่ 1691: PHP-FPM Pool Configuration

```ini
; /etc/php/8.3/fpm/pool.d/www.conf

[www]
; User/Group
user  = www-data
group = www-data

; Socket (เร็วกว่า TCP สำหรับ local Nginx)
listen = /run/php/php8.3-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode  = 0660

; Process Manager
; dynamic: ปรับจำนวน workers อัตโนมัติ
pm = dynamic

; จำนวน workers สูงสุด
; Formula: (RAM - OS overhead) / worker memory
; = (4GB - 512MB) / 50MB per worker = ~70 workers
pm.max_children = 70

; Workers ที่เริ่มต้น
pm.start_servers = 20

; ขั้นต่ำเมื่อ idle
pm.min_spare_servers = 10

; สูงสุดเมื่อ idle
pm.max_spare_servers = 30

; Restart worker หลังรับ requests ครบจำนวน (ป้องกัน memory leak)
pm.max_requests = 500

; Timeout สำหรับ request
request_terminate_timeout = 60s
request_slowlog_timeout    = 5s
slowlog                    = /var/log/php/slow.log

; Environment variables
env[HOSTNAME] = $HOSTNAME
env[PATH]     = /usr/local/bin:/usr/bin:/bin
env[TMP]      = /tmp
env[TMPDIR]   = /tmp
env[TEMP]     = /tmp

; PHP Settings per pool
php_admin_value[error_log]         = /var/log/php/error.log
php_admin_flag[log_errors]         = on
php_admin_value[memory_limit]      = 256M
php_admin_value[max_execution_time] = 30
php_admin_value[upload_max_filesize] = 64M
php_admin_value[post_max_size]     = 64M

; OPcache settings
php_admin_value[opcache.enable]            = 1
php_admin_value[opcache.memory_consumption] = 256
php_admin_value[opcache.max_accelerated_files] = 20000
php_admin_value[opcache.validate_timestamps]   = 0
php_admin_value[opcache.revalidate_freq]        = 0
php_admin_value[opcache.interned_strings_buffer] = 16
php_admin_value[opcache.fast_shutdown]          = 1
php_admin_value[opcache.enable_cli]             = 0
```

---

## ขั้นตอนที่ 1692: Nginx Virtual Host Configuration

```nginx
# /etc/nginx/sites-available/myapp.conf

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api:10m rate=60r/m;
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

# Upstream สำหรับ PHP-FPM
upstream php_fpm {
    server unix:/run/php/php8.3-fpm.sock;
    # หรือ TCP:
    # server 127.0.0.1:9000;
    keepalive 32;
}

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

    root  /var/www/myapp/public;
    index index.php;

    # SSL Configuration
    ssl_certificate     /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-CHACHA20-POLY1305;
    ssl_prefer_server_ciphers off;

    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # Security headers
    add_header X-Content-Type-Options     "nosniff" always;
    add_header X-Frame-Options            "SAMEORIGIN" always;
    add_header X-XSS-Protection           "1; mode=block" always;
    add_header Referrer-Policy            "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy         "camera=(), microphone=(), geolocation=()" always;
    add_header Content-Security-Policy    "default-src 'self'; script-src 'self' 'unsafe-inline' cdn.jsdelivr.net; style-src 'self' 'unsafe-inline' fonts.googleapis.com; font-src 'self' fonts.gstatic.com;" always;

    # Logging
    access_log /var/log/nginx/myapp.access.log combined buffer=512k flush=1m;
    error_log  /var/log/nginx/myapp.error.log warn;

    # Gzip Compression
    gzip            on;
    gzip_vary       on;
    gzip_proxied    any;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml
        application/rss+xml
        application/atom+xml
        image/svg+xml;

    # Connection limits
    limit_conn conn_limit 20;

    # Static files caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Vary "Accept-Encoding";
        access_log off;
    }

    # Laravel routes
    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    # API rate limiting
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_req_status 429;

        try_files $uri $uri/ /index.php?$query_string;
    }

    # Login rate limiting
    location /login {
        limit_req zone=login burst=3 nodelay;
        limit_req_status 429;

        try_files $uri $uri/ /index.php?$query_string;
    }

    # PHP-FPM handling
    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php_fpm;
        fastcgi_index index.php;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param PATH_INFO       $fastcgi_path_info;
        fastcgi_param DOCUMENT_ROOT   $realpath_root;

        # Timeouts
        fastcgi_connect_timeout 60s;
        fastcgi_read_timeout    300s;
        fastcgi_send_timeout    300s;

        # Buffering
        fastcgi_buffers         16 16k;
        fastcgi_buffer_size     32k;
        fastcgi_busy_buffers_size 64k;

        # Hide PHP version
        fastcgi_hide_header X-Powered-By;
    }

    # Block hidden files
    location ~ /\. {
        deny all;
    }

    # Block access to sensitive files
    location ~* \.(env|log|git|htaccess|htpasswd)$ {
        deny all;
    }
}
```

---

## ขั้นตอนที่ 1693: FastCGI Caching

```nginx
# /etc/nginx/nginx.conf - เพิ่มใน http block

fastcgi_cache_path /tmp/nginx-cache
    levels=1:2
    keys_zone=MYAPP:100m
    inactive=60m
    max_size=1g;

fastcgi_cache_key "$scheme$request_method$host$request_uri";

# /etc/nginx/sites-available/myapp.conf - ใน server block

# Cache configuration
set $skip_cache 0;

# Don't cache POST requests
if ($request_method = POST) {
    set $skip_cache 1;
}

# Don't cache URLs with query strings
if ($query_string != "") {
    set $skip_cache 1;
}

# Don't cache authenticated requests
if ($http_cookie ~* "laravel_session|XSRF-TOKEN") {
    set $skip_cache 1;
}

# Don't cache admin area
if ($request_uri ~* "^/admin") {
    set $skip_cache 1;
}

location ~ \.php$ {
    fastcgi_pass php_fpm;

    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;

    # FastCGI Cache
    fastcgi_cache            MYAPP;
    fastcgi_cache_valid       200 301 302 10m;
    fastcgi_cache_valid       404 1m;
    fastcgi_cache_bypass      $skip_cache;
    fastcgi_no_cache          $skip_cache;
    fastcgi_cache_use_stale   error timeout updating http_500 http_503;
    fastcgi_cache_lock        on;
    fastcgi_cache_min_uses    1;

    add_header X-FastCGI-Cache $upstream_cache_status;
}
```

---

## ขั้นตอนที่ 1694: Load Balancing

```nginx
# /etc/nginx/nginx.conf

# Multiple PHP-FPM servers (for horizontal scaling)
upstream php_backends {
    # Least connections algorithm
    least_conn;

    server app1.internal:9000 weight=3;
    server app2.internal:9000 weight=3;
    server app3.internal:9000 weight=2;

    # Health check
    server app4.internal:9000 backup;

    keepalive 32;
}

# Multiple web servers
upstream web_backends {
    ip_hash;  # Session persistence

    server web1.internal:80;
    server web2.internal:80;
    server web3.internal:80;

    keepalive 16;
}

server {
    listen 80;

    location / {
        proxy_pass http://web_backends;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 30s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        proxy_buffering          on;
        proxy_buffer_size        4k;
        proxy_buffers            8 4k;
        proxy_busy_buffers_size  8k;

        # Health check headers
        proxy_next_upstream error timeout invalid_header http_500 http_503;
    }
}
```

---

## ขั้นตอนที่ 1695: HTTP/2 Server Push และ Preloading

```nginx
# HTTP/2 Server Push
server {
    listen 443 ssl http2;

    location = / {
        http2_push /css/app.css;
        http2_push /js/app.js;
        http2_push /images/logo.png;

        try_files $uri $uri/ /index.php?$query_string;
    }
}
```

```php
<?php

declare(strict_types=1);

// app/Http/Middleware/Http2ServerPush.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Http\Response;
use Symfony\Component\HttpFoundation\Response as BaseResponse;

class Http2ServerPush
{
    private array $assets = [
        '</css/app.css>; rel=preload; as=style',
        '</js/app.js>; rel=preload; as=script',
        '</fonts/inter.woff2>; rel=preload; as=font; crossorigin',
    ];

    public function handle(Request $request, Closure $next): BaseResponse
    {
        $response = $next($request);

        if ($response instanceof Response && $request->isMethod('GET')) {
            $response->headers->set(
                'Link',
                implode(', ', $this->assets)
            );
        }

        return $response;
    }
}
```

---

## ขั้นตอนที่ 1696: Nginx Monitoring กับ Status Page

```nginx
# /etc/nginx/sites-available/monitoring.conf
server {
    listen 127.0.0.1:8080;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }

    location /php_status {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        allow 127.0.0.1;
        deny all;
    }

    location /fpm_status {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME /fpm-status;
        fastcgi_param QUERY_STRING    $args;
        include fastcgi_params;
        allow 127.0.0.1;
        deny all;
    }
}
```

```bash
# ตรวจสอบ Nginx status
curl http://127.0.0.1:8080/nginx_status

# Output:
# Active connections: 45
# server accepts handled requests
#  1234567 1234567 9876543
# Reading: 0 Writing: 3 Waiting: 42

# PHP-FPM status
curl "http://127.0.0.1:8080/fpm_status?full&json"
```

---

## ขั้นตอนที่ 1697: Performance Tuning Script

```bash
#!/bin/bash
# /usr/local/bin/tune-php-fpm.sh
# Auto-tune PHP-FPM based on available RAM

set -euo pipefail

# Total RAM in MB
TOTAL_RAM=$(free -m | awk '/^Mem:/ { print $2 }')

# OS overhead (500MB)
OS_OVERHEAD=500

# Available RAM for PHP-FPM
AVAILABLE_RAM=$((TOTAL_RAM - OS_OVERHEAD))

# Average PHP worker memory (MB) - check with: ps aux | grep php-fpm
WORKER_MEMORY=50

# Calculate max workers
MAX_WORKERS=$((AVAILABLE_RAM / WORKER_MEMORY))

# Limit max workers
if [ $MAX_WORKERS -gt 100 ]; then
    MAX_WORKERS=100
fi

# Start servers (25% of max)
START_SERVERS=$((MAX_WORKERS / 4))

# Min spare (20% of max)
MIN_SPARE=$((MAX_WORKERS / 5))

# Max spare (40% of max)
MAX_SPARE=$((MAX_WORKERS * 2 / 5))

echo "Total RAM: ${TOTAL_RAM}MB"
echo "Available for PHP: ${AVAILABLE_RAM}MB"
echo "Max Workers: ${MAX_WORKERS}"
echo "Start Servers: ${START_SERVERS}"
echo "Min Spare: ${MIN_SPARE}"
echo "Max Spare: ${MAX_SPARE}"

# Update PHP-FPM config
sed -i "s/pm.max_children = .*/pm.max_children = ${MAX_WORKERS}/" /etc/php/8.3/fpm/pool.d/www.conf
sed -i "s/pm.start_servers = .*/pm.start_servers = ${START_SERVERS}/" /etc/php/8.3/fpm/pool.d/www.conf
sed -i "s/pm.min_spare_servers = .*/pm.min_spare_servers = ${MIN_SPARE}/" /etc/php/8.3/fpm/pool.d/www.conf
sed -i "s/pm.max_spare_servers = .*/pm.max_spare_servers = ${MAX_SPARE}/" /etc/php/8.3/fpm/pool.d/www.conf

# Restart PHP-FPM
systemctl reload php8.3-fpm
echo "PHP-FPM reloaded with new settings"
```

---

## ขั้นตอนที่ 1698: SSL/TLS Best Practices

```bash
# สร้าง SSL certificate ด้วย Let's Encrypt
certbot --nginx -d myapp.com -d www.myapp.com

# Auto renewal
echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew -q" | sudo tee -a /etc/crontab > /dev/null

# Test SSL configuration
curl https://www.ssllabs.com/ssltest/analyze.html?d=myapp.com
```

```nginx
# Perfect Forward Secrecy + OCSP Stapling
ssl_certificate     /etc/letsencrypt/live/myapp.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

# OCSP Stapling
ssl_stapling           on;
ssl_stapling_verify    on;
ssl_trusted_certificate /etc/letsencrypt/live/myapp.com/chain.pem;
resolver               8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout       5s;

# DH Parameters
ssl_dhparam /etc/ssl/certs/dhparam.pem;

# Generate: openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
```

---

## สรุป Part 61

| Component | การปรับแต่ง | ผลลัพธ์ |
|-----------|------------|---------|
| PHP-FPM Pool | pm.max_children, pm.max_requests | ใช้ RAM อย่างมีประสิทธิภาพ |
| OPcache | opcache.validate_timestamps=0 | ลด disk I/O |
| Nginx FastCGI Cache | fastcgi_cache | ลด PHP execution |
| Gzip Compression | gzip_comp_level 6 | ลด bandwidth 70-90% |
| HTTP/2 | listen 443 ssl http2 | Multiplexing requests |
| SSL/TLS | TLS 1.3, OCSP Stapling | Security + Speed |
| Rate Limiting | limit_req_zone | ป้องกัน abuse |

ถัดไป → Part 62: Laravel Scout - Full-Text Search
