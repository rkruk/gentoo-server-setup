<i>Download, configure, and secure a fresh WordPress installation for your Gentoo-based web server.</i>

# 05 – WordPress Installation

> Install and configure a secure WordPress instance using the LEMP stack.  
> Covers file placement, permissions, configuration, and recommended CLI usage.

---

## Step 1: Download WordPress

Switch to your web root:

```bash
cd /var/www
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
rm latest.tar.gz
```

Now your structure should look like:

```bash
/var/www/wordpress/
```

## Step 2: Set Permissions

Make nginx the owner of all files:

```bash
chown -R nginx:nginx /var/www/wordpress
```

For additional hardening (optional):

```bash
find /var/www/wordpress/ -type d -exec chmod 750 {} \;
find /var/www/wordpress/ -type f -exec chmod 640 {} \;

```

## Step 3: Create Database

Login to MariaDB:

```bash
mysql -u root -p
```
Inside MySQL shell:

```sql
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'secure_password_here';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Step 4: Create wp-config.php

Copy default config:

```bash
cp /var/www/wordpress/wp-config-sample.php /var/www/wordpress/wp-config.php
```

Edit it:

```bash
nano /var/www/wordpress/wp-config.php
```

Update the following:

```php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wpuser' );
define( 'DB_PASSWORD', 'secure_password_here' );
define( 'DB_HOST', 'localhost' );
define( 'DB_CHARSET', 'utf8mb4' );
```

Replace the salts with fresh ones from:

`https://api.wordpress.org/secret-key/1.1/salt/`

Paste them into this block in `wp-config.php`:

```php
define('AUTH_KEY',         '...');
define('SECURE_AUTH_KEY',  '...');
```

...
## (Optional) Step 5: Use WP-CLI

Install WP-CLI:

```bash
curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x wp-cli.phar
mv wp-cli.phar /usr/local/bin/wp
```

Verify:

```bash
wp --info
```

Then you can install WordPress like this:

```bash
cd /var/www/wordpress

wp core install \
  --url="https://yourdomain.com" \
  --title="Your Site Title" \
  --admin_user="admin" \
  --admin_password="secure_password" \
  --admin_email="you@example.com"
```

Make sure the nginx user has ownership before running this:

```bash
chown -R nginx:nginx /var/www/wordpress
```

## Step 6: Finish in Browser

Visit:

```bash
https://yourdomain.com/
```

Follow the WordPress installation prompts, or — if using WP-CLI — you’re already done.

## Step 7: Security & Maintenance

Remove readme.html and license.txt from root location

Disable file editing via `wp-config.php`:

```php
define('DISALLOW_FILE_EDIT', true);
```

Install security & caching plugins

!!! Use strong passwords + 2FA plugin


