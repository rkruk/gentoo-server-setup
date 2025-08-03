<i>Installing a lean web stack: NGINX, PHP-FPM, and MariaDB.
Designed for low-resource Gentoo VPS with WordPress or Node.js in mind.</i>

# 03 – NGINX, PHP, and MariaDB Setup

> Set up a lightweight and production-ready web stack: NGINX for HTTP(S), PHP-FPM for WordPress, and MariaDB for the database.

---

## Step 1: NGINX

Install:

```bash
emerge --ask www-servers/nginx
```

Configure OpenRC service:

```bash
rc-update add nginx default
/etc/init.d/nginx start
```

Or for systemd:

```bash
systemctl enable nginx
systemctl start nginx
```

Minimal nginx.conf for a small VPS:

Edit `/etc/nginx/nginx.conf`:

```bash
user nginx nginx;
worker_processes 1;
worker_rlimit_nofile 4096;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # Optimize for low-end VPS
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 30;
    keepalive_requests 100;
    client_max_body_size 64M;
    client_body_timeout 12;
    client_header_timeout 12;
    send_timeout 10;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_comp_level 6;
    gzip_min_length 1000;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/js
        text/xml
        text/javascript
        application/javascript
        application/json
        application/xml
        application/rss+xml
        application/atom+xml
        image/svg+xml;

    # FastCGI cache for WordPress (enterprise-grade caching)
    fastcgi_cache_path /var/cache/nginx levels=1:2 keys_zone=WORDPRESS:10m max_size=100m inactive=60m use_temp_path=off;
    fastcgi_cache_key "$scheme$request_method$host$request_uri";
    
    # Advanced rate limiting zones (enterprise security)
    limit_req_zone $binary_remote_addr zone=wp_login:10m rate=3r/m;
    limit_req_zone $binary_remote_addr zone=wp_admin:10m rate=10r/m;
    limit_req_zone $binary_remote_addr zone=wp_api:10m rate=30r/m;
    limit_req_zone $binary_remote_addr zone=wp_upload:10m rate=5r/m;
    limit_req_zone $binary_remote_addr zone=wp_comments:10m rate=2r/m;
    
    # GeoIP-based blocking (optional: install nginx-module-geoip)
    # geo $blocked_country {
    #     default 0;
    #     CN 1;  # Block China
    #     RU 1;  # Block Russia
    #     # Add more countries as needed
    # }
    
    # Bot detection map
    map $http_user_agent $is_bot {
        default 0;
        ~*bot 1;
        ~*crawler 1;
        ~*spider 1;
        ~*scraper 1;
        ~*(nmap|nikto|sqlmap|whatweb) 1;
    }
    
    # WordPress security patterns
    map $request_uri $is_wp_attack {
        default 0;
        ~/wp-admin/install.php 1;
        ~/wp-admin/upgrade.php 1;
        ~/wp-config.php 1;
        ~/(wp-config|wp-mail)\.php 1;
        ~/readme\.html 1;
        ~/license\.txt 1;
    }
    
    # Cache bypass rules for dynamic content
    map $request_uri $skip_cache {
        default 0;
        ~*/wp-admin/ 1;
        ~*/wp-[a-zA-Z0-9-]+\.php 1;
        ~*preview=true 1;
    }

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

Create directory structure:

```bash
mkdir -p /etc/nginx/sites-{available,enabled}
mkdir -p /var/cache/nginx
chown -R nginx:nginx /var/cache/nginx
```

Create `/etc/nginx/sites-available/wordpress.conf`:

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/wordpress;
    index index.php index.html;
    
    # Block attacks and bots immediately
    if ($is_wp_attack) { return 403; }
    if ($is_bot) { return 429; }
    # if ($blocked_country) { return 403; }  # Uncomment if using GeoIP

    # Advanced security headers (enterprise-grade)
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' cdn.jsdelivr.net cdnjs.cloudflare.com; style-src 'self' 'unsafe-inline' fonts.googleapis.com; img-src 'self' data: https: *.gravatar.com; font-src 'self' fonts.gstatic.com; connect-src 'self';" always;
    add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Hide server information
    add_header Server "WebServer" always;
    server_tokens off;

    # WordPress permalinks
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    
    # Block access to sensitive files (better than plugin protection)
    location ~* ^/(wp-config\.php|wp-config-sample\.php|readme\.html|license\.txt)$ {
        deny all;
        return 403;
    }
    
    # Block access to hidden files and directories
    location ~ /\. {
        deny all;
        return 403;
    }
    
    # Prevent execution of PHP in uploads directory
    location ~* ^/wp-content/uploads/.*\.php$ {
        deny all;
        return 403;
    }
    
    # Advanced upload security with file type validation
    location ~* ^/wp-content/uploads/.*\.(exe|bat|cmd|com|pif|scr|vbs|js|jar|zip|rar|7z)$ {
        deny all;
        return 403;
    }

    # WordPress login protection (stricter than Wordfence)
    location = /wp-login.php {
        limit_req zone=wp_login burst=2 nodelay;
        limit_req_status 429;
        
        # Additional bot protection for login
        if ($http_user_agent ~* "(bot|crawler|spider|scraper)") {
            return 403;
        }
        
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 60;
    }
    
    # WordPress admin protection
    location ~* ^/wp-admin/.*\.php$ {
        limit_req zone=wp_admin burst=5 nodelay;
        
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 300;
        
        # Disable caching for admin
        fastcgi_cache_bypass 1;
        fastcgi_no_cache 1;
    }
    
    # WordPress comments protection
    location = /wp-comments-post.php {
        limit_req zone=wp_comments burst=1 nodelay;
        
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # File upload protection
    location ~* ^/wp-admin/async-upload\.php$ {
        limit_req zone=wp_upload burst=3 nodelay;
        
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 600;  # Allow longer upload times
    }
    
    # General PHP handling with intelligent caching
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        
        # Smart timeout based on request
        fastcgi_read_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_connect_timeout 60;
        
        # Intelligent caching (skip for admin/logged-in users)
        set $skip_cache 0;
        if ($request_method = POST) { set $skip_cache 1; }
        if ($query_string != "") { set $skip_cache 1; }
        if ($request_uri ~* "/(wp-admin/|wp-login\.php|wp-register\.php)") { set $skip_cache 1; }
        if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_logged_in") { set $skip_cache 1; }
        
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 301 302 60m;
        fastcgi_cache_valid 404 1m;
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;
        add_header X-FastCGI-Cache $upstream_cache_status always;
    }

    # WordPress API rate limiting
    location ~ ^/wp-json/ {
        limit_req zone=wp_api burst=10 nodelay;
        try_files $uri $uri/ /index.php?$args;
    }

    # Static files optimization (better than W3TC)
    location ~* \.(jpg|jpeg|png|gif|ico|webp|avif|css|js|woff|woff2|ttf|eot|svg|pdf|doc|docx)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Vary "Accept-Encoding";
        access_log off;
        
        # Enable gzip for compressible files
        gzip_static on;
    }
    
    # WordPress feeds caching
    location ~* ^/feed/?.*$ {
        expires 1h;
        add_header Cache-Control "public";
    }
    
    # Advanced protection for uploads directory
    location ^~ /wp-content/uploads/ {
        # Block execution of any scripts
        location ~* \.(php|php3|php4|php5|phtml|pl|py|jsp|asp|sh|cgi|exe|bat|cmd)$ {
            deny all;
            return 403;
        }
        
        # Allow only safe file types with proper caching
        location ~* \.(jpg|jpeg|png|gif|ico|webp|avif|svg|pdf|doc|docx|txt|zip|mp3|mp4|avi|mov|woff|woff2|ttf|eot)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
            add_header Vary "Accept-Encoding";
            access_log off;
        }
        
        # Fallback - deny everything else in uploads
        try_files $uri =404;
    }
    
    # Security: Block access to sensitive WordPress files
    location ~* ^/(xmlrpc\.php|wp-trackback\.php|wp-links-opml\.php)$ {
        deny all;
        return 403;
    }
    
    # Block access to backup and configuration files
    location ~* \.(bak|backup|old|tmp|temp|log|sql|gz|tar|conf|config|ini|yml|yaml|env)$ {
        deny all;
        return 403;
    }
    
    # Block WordPress installation and upgrade files
    location ~* ^/(install\.php|upgrade\.php|wp-admin/install\.php|wp-admin/upgrade\.php)$ {
        deny all;
        return 403;
    }
    
    # Deny access to critical directories
    location ~* ^/(\.ht|\.git|\.svn|\.bzr|\.hg) {
        deny all;
        return 403;
    }
}
```

Enable the site:

```bash
ln -s /etc/nginx/sites-available/wordpress.conf /etc/nginx/sites-enabled/
```

## Step 2: PHP-FPM

Install PHP with minimal extensions:

```bash
echo 'dev-lang/php bcmath cli ctype fileinfo filter hash intl json nls opcache pdo phar posix readline session simplexml sockets ssl tokenizer xml xmlreader xmlwriter zip fpm' >> /etc/portage/package.use/php
emerge -av dev-lang/php
```

Enable the FPM service:

```bash
rc-update add php-fpm default
/etc/init.d/php-fpm start
```

Or for systemd:

```bash
systemctl enable php-fpm
systemctl start php-fpm
```

Make sure /etc/php/fpm-php*/php-fpm.conf uses a Unix socket:

```bash
listen = /run/php-fpm.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
```

**Critical:** Tune for 1GB RAM VPS in `/etc/php/fpm-php*/fpm.d/www.conf`:

```bash
[www]
user = nginx
group = nginx
listen = /run/php-fpm.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660

; Process management for 1GB RAM VPS
pm = dynamic
pm.max_children = 15
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
pm.max_requests = 200

; Memory limits
php_admin_value[memory_limit] = 128M
php_admin_value[max_execution_time] = 30
php_admin_value[max_input_vars] = 3000

; File upload settings (must match nginx client_max_body_size)
php_admin_value[upload_max_filesize] = 64M
php_admin_value[post_max_size] = 64M
php_admin_value[max_file_uploads] = 20

; Session configuration for WordPress
php_admin_value[session.save_path] = /tmp
php_admin_value[session.cookie_secure] = 1
php_admin_value[session.cookie_httponly] = 1
php_admin_value[session.use_strict_mode] = 1
php_admin_value[session.gc_maxlifetime] = 1440

; OPcache settings for performance
php_admin_value[opcache.enable] = 1
php_admin_value[opcache.memory_consumption] = 64
php_admin_value[opcache.max_accelerated_files] = 4000
php_admin_value[opcache.validate_timestamps] = 1
php_admin_value[opcache.revalidate_freq] = 60
php_admin_value[opcache.save_comments] = 1

; Logging
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
```

## Step 3: MariaDB

Install:

```bash
emerge -av dev-db/mariadb
```

Run initial setup:

```bash
emerge --config dev-db/mariadb
```

Enable the service:

```bash
rc-update add mariadb default
/etc/init.d/mariadb start
```

Or with systemd:

```bash
systemctl enable mariadb
systemctl start mariadb
```

Secure installation:

```bash
mysql_secure_installation
```

**Critical:** Optimize MariaDB for 1GB RAM VPS.

Create `/etc/mysql/mariadb.conf.d/99-vps-optimization.cnf`:

```bash
[mysqld]
# Memory optimization for 1GB RAM VPS
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
innodb_flush_log_at_trx_commit = 2
innodb_flush_method = O_DIRECT

# Query cache for WordPress
query_cache_type = 1
query_cache_size = 32M
query_cache_limit = 2M

# Connection optimization
max_connections = 50
thread_cache_size = 8
table_open_cache = 400

# Logging (disable for performance)
slow_query_log = 0
general_log = 0

# MyISAM optimization
key_buffer_size = 32M
myisam_sort_buffer_size = 8M

# Tmp table optimization
tmp_table_size = 32M
max_heap_table_size = 32M

# Disable performance schema to save RAM
performance_schema = OFF
```

Restart MariaDB:

```bash
/etc/init.d/mariadb restart
```

Verify memory usage:

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
```

## Step 4: Test Everything

Create PHP test file:

```bash
mkdir -p /var/www/wordpress
echo "<?php phpinfo(); ?>" > /var/www/wordpress/index.php
chown -R nginx:nginx /var/www/wordpress
```

Access via browser:
`http://your-server-ip/`

You should see the PHP info page.

Test nginx configuration and reload:

```bash
nginx -t
/etc/init.d/nginx reload
```

Or with systemd:

```bash
systemctl reload nginx
```

## Step 5: Verify File Upload Configuration

**Critical:** Ensure nginx and PHP upload limits are properly aligned.

Test the configuration:

```bash
# Check nginx client_max_body_size
grep "client_max_body_size" /etc/nginx/nginx.conf

# Check PHP upload settings
php -i | grep -E "(upload_max_filesize|post_max_size|max_file_uploads)"
```

Expected output:
- nginx: `client_max_body_size 64M;`
- PHP: `upload_max_filesize => 64M`, `post_max_size => 64M`, `max_file_uploads => 20`

**If mismatched:** The most common WordPress issue is upload failures due to mismatched limits between nginx and PHP.

## Step 6: Clean Up and Harden

After verifying that PHP works with the test file:

- Delete `/var/www/wordpress/index.php` (the `phpinfo()` file)
- If you plan to run WordPress or any public site:
  - Configure TLS (see next section)
  - Review and harden `/etc/php/*/php.ini` and `/etc/nginx/nginx.conf`

