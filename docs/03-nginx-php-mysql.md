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
worker_processes auto;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    gzip on;
    gzip_disable "msie6";

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

Create `/etc/nginx/sites-enabled/wordpress.conf`:

```bash
server {
    listen 80;
    server_name example.com;
    root /var/www/wordpress;

    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass unix:/run/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
}
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

## Step 5: Clean Up and Harden

After verifying that PHP works with the test file:

- Delete `/var/www/wordpress/index.php` (the `phpinfo()` file)
- If you plan to run WordPress or any public site:
  - Configure TLS (see next section)
  - Review and harden `/etc/php/*/php.ini` and `/etc/nginx/nginx.conf`

