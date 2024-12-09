<p align="center">
  <img src="https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png">
</p>
<br>
<p align="center"><b>GENTOO SERVER SETUP GUIDE</b></p>

<p align="center"><b>A comprehensive guide for setting up a secure Gentoo server with Nginx for WordPress hosting.</b></p>

## Table of Contents
- [Initial Server Setup](#initial-server-setup)
- [Basic Security](#basic-security)
- [Nginx Installation](#nginx-installation)
- [PHP Installation](#php-installation)
- [MySQL/MariaDB Setup](#database-setup)
- [SSL Certificates](#ssl-certificates)
- [WordPress Configuration](#wordpress-configuration)
- [Server Hardening](#server-hardening)
- [Monitoring Setup](#monitoring-setup)

## Initial Server Setup

### System Update

```bash
# Update Gentoo system
emerge --sync
emerge -avuDN @world
```

```bash
# Install essential tools
emerge --ask app-admin/sudo
emerge --ask app-admin/logrotate
emerge --ask app-admin/fail2ban
emerge --ask net-firewall/iptables
```

### Create Service User
```bash
useradd -m -G users,wheel -s /bin/bash adminuser
passwd adminuser
```

### Basic Security
<br>
SSH Configuration

Edit `/etc/ssh/sshd_config`:

```bash
Port 2222                    # Change default SSH port
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Firewall Setup
<br>
```bash
# IPTables basic rules
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 2222 -j ACCEPT
iptables -P INPUT DROP
```

Nginx Installation
<br>
Install Nginx

```bash
# Add USE flags
echo "www-servers/nginx http2 ssl" >> /etc/portage/package.use/nginx

# Install Nginx
emerge --ask www-servers/nginx
```

Directory Structure
<br>
```bash
mkdir -p /etc/nginx/{sites-available,sites-enabled}
mkdir -p /var/www/example.com
chown -R nginx:nginx /var/www/example.com
```

Base Configuration

Create `/etc/nginx/nginx.conf`:

```bash
user nginx nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    multi_accept on;
    worker_connections 65535;
}

http {
    charset utf-8;
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;
    log_not_found off;
    types_hash_max_size 2048;
    types_hash_bucket_size 64;
    client_max_body_size 16M;

    # MIME
    include mime.types;
    default_type application/octet-stream;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    # SSL
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    
    # Modern configuration
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 1.1.1.1 1.0.0.1 valid=300s;
    resolver_timeout 5s;

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=wp_login:10m rate=1r/s;
    limit_req_zone $binary_remote_addr zone=wp_admin:10m rate=10r/s;

    # Load configs
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```
<br>
Site Configuration

Create `/etc/nginx/sites-available/example.conf`:

```bash
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    # HTTP/3 support
    listen 443 quic reuseport;
    listen [::]:443 quic reuseport;
    add_header Alt-Svc 'h3=":443"; ma=86400';
    
    server_name example.com;
    root /var/www/example.com;
    index index.php;

    # SSL
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    add_header Permissions-Policy "camera=(), geolocation=(), microphone=()" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Logging
    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;

    # WordPress security
    location = /wp-login.php {
        limit_req zone=wp_login burst=2 nodelay;
        try_files $uri =404;
        include fastcgi_params;
        fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
    }

    location /wp-admin/ {
        limit_req zone=wp_admin burst=5 nodelay;
        try_files $uri $uri/ /index.php?$args;
    }

    # Static files
    location ~* \.(gif|jpg|jpeg|png|webp|avif|ico|css|js)$ {
        expires 1y;
        add_header Cache-Control "public, no-transform";
        
        valid_referers none blocked example.com www.example.com 
                      ~\.google\. ~\.bing\. ~\.facebook\. ~\.fbcdn\.;
        if ($invalid_referer) {
            return 403;
        }
        
        access_log off;
    }

    # PHP handling
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_read_timeout 300;
    }

    # Block access to sensitive files
    location ~ /\.(ht|git|env|config) {
        deny all;
        return 404;
    }

    # Block PHP execution in uploads
    location /wp-content/uploads/ {
        location ~ \.php$ {
            deny all;
        }
    }
}

# HTTP redirect
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    
    location / {
        return 301 https://example.com$request_uri;
    }
}
```
<br>
Enable Configuration

```bash
ln -s /etc/nginx/sites-available/example.conf /etc/nginx/sites-enabled/example.conf

# Test configuration
nginx -t

# Start Nginx
rc-service nginx start
rc-update add nginx default
```
<br>
### PHP Installation
<br>
Install PHP-FPM
```bash
echo "dev-lang/php fpm mysqli pdo ssl" >> /etc/portage/package.use/php
emerge --ask dev-lang/php
```
<br>
Configure PHP-FPM

Edit `/etc/php/fpm-php8.2/php-fpm.conf`:

```bash
[global]
pid = /run/php-fpm/php-fpm.pid
error_log = /var/log/php-fpm.log

[www]
user = nginx
group = nginx
listen = /run/php-fpm/php-fpm.sock
listen.owner = nginx
listen.group = nginx
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```
<br>
Start PHP-FPM

```bash
rc-service php-fpm start
rc-update add php-fpm default
```
<br>
### SSL Certificates
<br>
Install acme.sh

```bash
emerge --ask app-crypt/acme.sh
```
<br>
Generate Certificates

```bash
acme.sh --issue -d example.com -w /var/www/example.com
```
<br>
### Server Monitoring
<br>
Install Monitoring Tools

```bash
emerge --ask app-admin/telegraf
emerge --ask net-analyzer/netdata
```
<br>
Enable Services

```bash
rc-update add telegraf default
rc-update add netdata default
```
<br>
### Regular Maintenance
<br>
Create Maintenance Script
<br>

Create `/root/server-maintenance.sh`:

```bash
#!/bin/bash

# Update system
emerge --sync
emerge -avuDN @world

# Rotate logs
logrotate -f /etc/logrotate.conf

# Backup MySQL databases
mysqldump --all-databases > /backup/mysql/all-databases-$(date +%Y%m%d).sql

# Clean old backups (keep last 30 days)
find /backup/mysql/ -type f -mtime +30 -delete

# Check services
rc-service nginx status
rc-service php-fpm status
rc-service mysql status

# Test Nginx configuration
nginx -t
```
<br>
Make script executable:

```bash
chmod +x /root/server-maintenance.sh
```
<br>
Add to crontab:

```bash
0 3 * * * /root/server-maintenance.sh
```
<br>
### Troubleshooting
<br>
Common Issues:
<br>
1. Nginx won't start

```bash
# Check error log
tail -f /var/log/nginx/error.log

# Check permissions
ls -la /var/www/example.com
ls -la /run/nginx.pid
```
<br>
2. PHP-FPM issues

```bash
# Check PHP-FPM log
tail -f /var/log/php-fpm.log

# Check socket
ls -la /run/php-fpm/php-fpm.sock
```
<br>
3. SSL Certificate issues

```bash
# Test SSL configuration
openssl s_client -connect example.com:443 -tls1_3

# Check certificate expiry
openssl x509 -enddate -noout -in /etc/letsencrypt/live/example.com/cert.pem
```
<br>
### Performance Optimization
<br>
Enable HTTP/3:

```bash
# In nginx.conf
load_module modules/ngx_http_v3_module.so;

# In server block
listen 443 quic reuseport;
add_header Alt-Svc 'h3=":443"; ma=86400';
```
<br>
Enable Brotli Compression:

```bash
emerge --ask www-servers/nginx # [ USE="brotli" - add it to make.conf USE flags]
```
<br>
Add to nginx.conf:

```bash
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/javascript application/json image/svg+xml;
```
<br>
### WordPress Optimization
<br>
Add to wp-config.php:

```bash
<?php
define('WP_CACHE', true);
define('DISABLE_WP_CRON', true);
```
<br>
### Security Hardening
<br>
Fail2ban Configuration:

Create `/etc/fail2ban/jail.local`:

```bash
[wordpress]
enabled = true
filter = wordpress
logpath = /var/log/nginx/example.com.access.log
maxretry = 3
bantime = 3600
findtime = 600
```
<br>
ModSecurity Setup

```bash
emerge --ask www-servers/nginx # [USE="modsecurity"]
```
<br>
Add to nginx.conf:

```bash
modsecurity on;
modsecurity_rules_file /etc/nginx/modsecurity/main.conf;
```
<br>
### Backup Strategy
<br>
Create Backup Script

Create `/root/backup.sh`:

```bash
#!/bin/bash

BACKUP_DIR="/backup"
DATE=$(date +%Y-%m-%d)
SITE="example.com"

# Backup WordPress files
tar -czf "$BACKUP_DIR/files-$SITE-$DATE.tar.gz" /var/www/$SITE

# Backup database
mysqldump --single-transaction wordpress > "$BACKUP_DIR/db-$SITE-$DATE.sql"

# Backup Nginx configuration
cp -r /etc/nginx "$BACKUP_DIR/nginx-$DATE"

# Remove backups older than 30 days
find $BACKUP_DIR -type f -mtime +30 -delete
```
<br>
Add to crontab:

```bash
0 2 * * * /root/backup.sh
```
<br>
<br>
This guide includes modern best practices for 2024, including:

- **HTTP/3 Support**  
- **Modern SSL/TLS Configuration**  
- **Enhanced Security Headers**  
- **Rate Limiting**  
- **Performance**

<br><br>
