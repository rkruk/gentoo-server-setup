<p align="center">
  <img src="https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png">
</p>
<br>
<p align="center"><b>GENTOO WEB SERVER SETUP GUIDE</b></p>

<p align="center"><b>A comprehensive guide for setting up a secure Gentoo server with Nginx for WordPress hosting.</b></p>

## Table of Contents
- [Initial Server Setup](#initial-server-setup)
  - [System Update](#system-update)
  - [Install Essential Tools](#install-essential-tools)
  - [Create Service User](#create-service-user)
- [Basic Security](#basic-security)
  - [SSH Configuration](#ssh-configuration)
  - [Secure SSH Access](#secure-ssh-access)
  - [Firewall Configuration](#firewall-configuration)
  - [Monitor Logs](#monitor-logs)
  - [Enable Two-Factor Authentication (2FA)](#enable-two-factor-authentication-2fa)
  - [Limit User Privileges](#limit-user-privileges)
  - [Use Fail2Ban](#use-fail2ban)
- [Nginx Installation](#nginx-installation)
- [Node.js Setup](#nodejs-setup)
- [PHP Installation](#php-installation)
- [MySQL/MariaDB Setup](#database-setup)
- [SSL Certificates](#ssl-certificates)
- [WordPress Configuration](#wordpress-configuration)
- [Server Hardening](#server-hardening)
- [Monitoring Setup](#monitoring-setup)
- [Virus Protection](#virus-protection)
- [Automate IPSet Updates](#automate-ipset-updates)
- [Implement SELinux or AppArmor](#implement-selinux-or-apparmor)
- [Final Recommendations](#final-recommendations)

## Project Description

This project provides a comprehensive guide for setting up a secure Gentoo server with Nginx for WordPress hosting. It covers initial server setup, basic security measures, installation of necessary software, and configuration details to ensure optimal performance and security.
<br><br>
Follow the steps in each section of this guide to set up your server.<br>

## Usage Instructions
This guide is intended for setting up a secure Gentoo server for hosting WordPress websites.<br> 
Follow the detailed instructions provided in each section to complete the setup.

## Contributing Guidelines
Contributions are welcome! Please submit a pull request or open an issue to discuss any changes or improvements.

## License Information
This project is licensed under the MIT License. See the [LICENSE](https://github.com/rkruk/gentoo-server-setup/blob/6b6dfbca24b6335f35da514924ac1524eec5456a/LICENSE) file for more details.
<br><br>

## Installation Instructions
1. Clone the repository:
```bash
git clone https://github.com/rkruk/gentoo-server-setup.git
cd gentoo-server-setup
```

## Initial Server Setup

### System Update

```bash
# Update Gentoo system
emerge --sync
emerge -avuDN @world
```

## Install essential tools

```bash
emerge --ask app-admin/sudo
emerge --ask app-admin/logrotate
emerge --ask app-admin/fail2ban
emerge --ask net-firewall/iptables
emerge --ask net-analyzer/ipset
```

## Create Service User

```bash
useradd -m -G users,wheel -s /bin/bash adminuser
passwd adminuser
```

## Basic Security<br>

## SSH Configuration
Edit `/etc/ssh/sshd_config`:

## Example SSH configuration changes

```bash
Port 2222
PermitRootLogin no
PasswordAuthentication yes
```

## Secure SSH Access
Generate SSH Keys
On your local machine, generate an SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Press Enter to accept the default file location and enter a passphrase for added security.

Copy Public Key to Server

```bash
ssh-copy-id -p 2222 user@example.com
```

Replace `2222` with your SSH port if different, and user@example.com with your server's user and address.

Configure SSH Daemon
Edit the SSH daemon configuration file:

```bash
nano /etc/ssh/sshd_config
```

Modify the following settings:

```bash
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile  .ssh/authorized_keys
```

Explanation:

`Port 2222`: Change the SSH port to a custom port for security.<br>
`PermitRootLogin no`: Disable root login over SSH.<br>
`PasswordAuthentication no`: Disable password-based authentication, enforcing key-based authentication.<br>
`PubkeyAuthentication yes`: Ensure key-based authentication is enabled.

Restart SSH Service

```bash
rc-service sshd restart
```

## Firewall Configuration
Continue to manage IPTables and IPSet rules to block unwanted traffic.

Update IPTables to Block Additional Countries
Create or update the `update-country-blocks.sh` script to block specific countries.

```bash
# /usr/local/sbin/update-country-blocks.sh
#!/bin/bash

# Countries to block
COUNTRIES=("CN" "RU" "BY" "KP" "IR" "IN" "PK" "DZ" "AO" "EG" "NG" "ZA" "KE" "UG" "GH" "TZ" "MA" "TN" "CM" "IQ" "SY")

TMPDIR="/tmp/country-blocks"

# Create temporary directory
mkdir -p $TMPDIR

# Download latest country IP lists
for country in "${COUNTRIES[@]}"; do
    wget -q "https://www.ipdeny.com/ipblocks/data/countries/${country}.zone" -O "$TMPDIR/${country}.zone"
done

# Create ipset for each country
for country in "${COUNTRIES[@]}"; do
    # Destroy existing set if exists
    ipset destroy ${country,,} 2>/dev/null
    # Create new set
    ipset create ${country,,} hash:net
    # Add IPs to set
    while IFS= read -r ip; do
        ipset add ${country,,} $ip
    done < "$TMPDIR/${country}.zone"
done

# Clean up
rm -rf $TMPDIR

echo "Country IP sets updated."
```

Make the script executable:

```bash
chmod +x /usr/local/sbin/update-country-blocks.sh
```

Enhanced IPTables Configuration
Update `/etc/iptables/firewall.rules` to include the additional countries:

```bash
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:FAIL2BAN - [0:0]

# Allow loopback
-A INPUT -i lo -j ACCEPT

# Allow established connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH on custom port with rate limiting
-A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --set
-A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
-A INPUT -p tcp --dport 2222 -j ACCEPT

# Allow HTTP and HTTPS
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

# Block traffic from specified countries
-A INPUT -m set --match-set cn src -j DROP
-A INPUT -m set --match-set ru src -j DROP
-A INPUT -m set --match-set by src -j DROP
-A INPUT -m set --match-set kp src -j DROP
-A INPUT -m set --match-set ir src -j DROP
-A INPUT -m set --match-set in src -j DROP
-A INPUT -m set --match-set pk src -j DROP
-A INPUT -m set --match-set dz src -j DROP
-A INPUT -m set --match-set ao src -j DROP
-A INPUT -m set --match-set eg src -j DROP
-A INPUT -m set --match-set ng src -j DROP
-A INPUT -m set --match-set za src -j DROP
-A INPUT -m set --match-set ke src -j DROP
-A INPUT -m set --match-set ug src -j DROP
-A INPUT -m set --match-set gh src -j DROP
-A INPUT -m set --match-set tz src -j DROP
-A INPUT -m set --match-set ma src -j DROP
-A INPUT -m set --match-set tn src -j DROP
-A INPUT -m set --match-set cm src -j DROP
-A INPUT -m set --match-set iq src -j DROP
-A INPUT -m set --match-set sy src -j DROP

# ICMP rate limiting
-A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT

# Rate limiting for HTTP(S)
-A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
-A INPUT -p tcp --dport 443 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT

# Custom chain for fail2ban
-A INPUT -j FAIL2BAN

# Log dropped packets
-A INPUT -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "IPTables Dropped: " --log-level 7

COMMIT
```

### Explanation:

Added `ipset` rules to block traffic from:

- **India (IN)**
- **Pakistan (PK)**
- **Russia (RU)**
- **Belarus (BY)**
- **China (CN)**

and selected African countries:

- **Algeria (DZ)**
- **Angola (AO)**
- **Egypt (EG)**
- **Nigeria (NG)**
- **South Africa (ZA)**
- **Kenya (KE)**
- **Uganda (UG)**
- **Ghana (GH)**
- **Tanzania (TZ)**
- **Morocco (MA)**
- **Tunisia (TN)**
- **Cameroon (CM)**
- **Iraq (IQ)**
- **Syria (SY)**

Schedule IPSet Updates<br>
Automate the `update-country-blocks.sh` script to run periodically.

## Schedule the script to run daily at 1 AM

```bash
echo "0 1 * * * root /usr/local/sbin/update-country-blocks.sh" >> /etc/crontab
```

## Monitor Logs
Install Logwatch<br>
Logwatch summarizes logs and sends reports via email.

```bash
# Install Logwatch
emerge --ask app-admin/logwatch
```

Configure Logwatch<br>
Edit Logwatch configuration file:

```bash
nano /etc/logwatch/conf/logwatch.conf
```

Basic Configuration:

```bash
MailTo = admin@example.com
Detail = High
Service = All
Range = yesterday
```

Schedule Logwatch Reports<br>
Create a cron job to send daily log summaries.

```bash
# Schedule Logwatch to run daily at 6 AM
echo "0 6 * * * root /usr/sbin/logwatch --output mail --mailto admin@example.com --detail high" >> /etc/crontab
```

## Enable Two-Factor Authentication (2FA)<br>
Install Google Authenticator PAM Module

```bash
# Install Google Authenticator
emerge --ask app-auth/libpam-google-authenticator
```

Configure PAM for SSH<br>
Edit the SSH PAM configuration:

```bash
nano /etc/pam.d/sshd
```

Add the following line at the top:

```bash
auth required pam_google_authenticator.so
```

Configure SSH Daemon for 2FA<br>
Edit the SSH daemon configuration file:

```bash
nano /etc/ssh/sshd_config
```

Add or modify the following settings:

```bash
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

Restart SSH Service

```bash
rc-service sshd restart
```

Configure 2FA for Each User<br>
On the server, for each user, run:

```bash
google-authenticator
```

Follow the prompts to set up 2FA, scanning the QR code with an authenticator app.

## Limit User Privileges
Ensure users have the minimum required permissions.

Use sudo for Administrative Tasks<br>
Edit the sudoers file:

```bash
visudo
```

Add the following line to grant sudo privileges to adminuser:

```bash
adminuser ALL=(ALL) ALL
```

Explanation:

Users should perform administrative tasks using sudo rather than logging in as root, minimizing the risk of accidental system-wide changes.


## Use Fail2Ban
Protect against brute-force attacks.

Configure Fail2Ban
Edit Fail2Ban configuration for SSH:

```bash
nano /etc/fail2ban/jail.local
```

Add the following configuration:

```bash
[sshd]
enabled = true
port    = 2222
filter  = sshd
logpath = /var/log/auth.log
maxretry = 5
bantime  = 3600
```

Start and Enable Fail2Ban

```bash
# Start Fail2Ban service
rc-service fail2ban start

# Enable Fail2Ban to start on boot
rc-update add fail2ban default
```

## Nginx Installation

## Add USE flags

```bash
echo "www-servers/nginx http2 ssl" >> /etc/portage/package.use/nginx
```

## Install Nginx

```bash
emerge --ask www-servers/nginx
```

Directory Structure

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

    # MIME Types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    # Gzip Compression
    gzip on;
    gzip_disable "msie6";
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml image/webp image/avif;

    # Brotli Compression
    brotli on;
    brotli_comp_level 6;
    brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml image/webp image/avif;

    # Include Site Configurations
    include /etc/nginx/sites-enabled/*.conf;
}
```

Enhanced Configuration for Security and Performance<br>
Enforce TLS 1.2 and Higher<br>
Ensure only TLS 1.2 and TLS 1.3 protocols are used.

```bash
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
ssl_dhparam /etc/nginx/ssl/dhparam.pem;
```

## Generate Diffie-Hellman parameters

```bash
mkdir -p /etc/nginx/ssl
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
```

Redirect All HTTP Traffic to HTTPS

```bash
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Redirect all HTTP requests to HTTPS
    return 301 https://$host$request_uri;
}
```

Optimize Caching for WordPress

```bash
http {
    # FastCGI Cache Path
    fastcgi_cache_path /var/cache/nginx/fastcgi-cache levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;

    # Define Cache Key
    fastcgi_cache_key "$scheme$request_method$host$request_uri";

    # ... existing settings ...

    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com www.example.com;
        root /var/www/example.com;
        index index.php index.html;

        # SSL Configuration
        ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        ssl_dhparam /etc/nginx/ssl/dhparam.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers off;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';

        # Security Headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header Referrer-Policy "no-referrer-when-downgrade" always;
        add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
        add_header Permissions-Policy "camera=(), geolocation=(), microphone=()" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

        # Logging
        access_log /var/log/nginx/sites/example.com/access.log main_ext;
        access_log /var/log/nginx/sites/example.com/security.log security;
        error_log /var/log/nginx/sites/example.com/error.log warn;
        access_log /var/log/nginx/sites/example.com/slow.log main_ext if=$slow_request;

        # Slow request tracking
        map $request_time $slow_request {
            default 0;
            "~^[3-9]|[0-9]{2,}" 1;
        }

        # Block disallowed countries
        if ($allowed_country = no) {
            return 444;
        }

        # PHP Handling with FastCGI Cache
        location ~ \.php$ {
            try_files $uri =404;
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
            fastcgi_index index.php;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;

            # FastCGI Cache Configuration
            fastcgi_cache WORDPRESS;
            fastcgi_cache_valid 200 60m;
            fastcgi_cache_use_stale error timeout invalid_header updating http_500;
            fastcgi_cache_bypass $skip_cache;
            fastcgi_no_cache $skip_cache;
        }

        # Protect wp-config.php
        location ~ ^/wp-config.php$ {
            deny all;
        }

        # Protect .htaccess and other hidden files
        location ~ /\.(ht|git|env|config|composer\.json|composer\.lock|gitlab-ci\.yml)$ {
            deny all;
            return 404;
        }

        # Block PHP execution in uploads
        location /wp-content/uploads/ {
            location ~ \.php$ {
                deny all;
            }
        }

        # Handle WordPress Login with Rate Limiting
        location = /wp-login.php {
            limit_req zone=wp_login burst=2 nodelay;
            try_files $uri =404;
            include fastcgi_params;
            fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }

        # Protect wp-admin with Rate Limiting
        location /wp-admin/ {
            limit_req zone=wp_admin burst=5 nodelay;
            try_files $uri $uri/ /index.php?$args;
        }

        # Static Files with Caching and Referrer Validation
        location ~* \.(gif|jpg|jpeg|png|webp|avif|ico|css|js|woff|woff2|ttf|otf|svg)$ {
            expires 1y;
            add_header Cache-Control "public, no-transform";

            valid_referers none blocked example.com www.example.com ~\.google\. ~\.bing\. ~\.facebook\. ~\.fbcdn\.;
            if ($invalid_referer) {
                return 403;
            }

            access_log off;
        }

        # Image Hotlinking Protection
        location ~* \.(gif|jpg|jpeg|png|webp|avif|ico)$ {
            valid_referers none blocked example.com www.example.com ~\.google\. ~\.bing\. ~\.facebook\. ~\.fbcdn\.;
            if ($invalid_referer) {
                return 403;
            }
            expires 1y;
            add_header Cache-Control "public, no-transform";
            access_log off;
        }

        # Gzip Compression
        gzip on;
        gzip_disable "msie6";
        gzip_vary on;
        gzip_proxied any;
        gzip_comp_level 6;
        gzip_buffers 16 8k;
        gzip_http_version 1.1;
        gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml image/webp image/avif;

        # Brotli Compression
        brotli on;
        brotli_comp_level 6;
        brotli_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript image/svg+xml image/webp image/avif;
    }
}
```

Add Additional Security Enhancements

```bash
server {
    # Existing configurations...

    # HTTP Strict Transport Security (HSTS)
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Content Security Policy (CSP)
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' https://trusted.cdn.com; object-src 'none'; frame-ancestors 'none';" always;
    # the https://trusted.cdn.com is an CDN example :D - use whatever you need/want/know.
    # Disable Unnecessary HTTP Methods
    location / {
        if ($request_method !~ ^(GET|HEAD|POST)$ ) {
            return 405;
        }
        # Existing configurations...
    }

    # Limit Request Size
    client_max_body_size 16M;

    # Disable Server Signature
    server_tokens off;

    # Use limit_conn to Prevent Connection Flooding
    limit_conn_zone $binary_remote_addr zone=addr:10m;

    location / {
        limit_conn addr 10;
        # Existing configurations...
    }
}
```

Enable Nginx Site Configurations with Descriptions<br>
Create symbolic links to enable site configurations and provide short descriptions.

## Enable sites by creating symbolic links

```bash
ln -s /etc/nginx/sites-available/wordpress.conf /etc/nginx/sites-enabled/wordpress.conf  # Enables WordPress site
ln -s /etc/nginx/sites-available/static-html.conf /etc/nginx/sites-enabled/static-html.conf  # Enables Static HTML site
ln -s /etc/nginx/sites-available/nodejs-app.conf /etc/nginx/sites-enabled/nodejs-app.conf  # Enables Node.js application
```

Short Descriptions:

WordPress: Link to enable the WordPress website configuration.<br>
Static HTML: Link to enable the Static HTML website configuration.<br>
Node.js App: Link to enable the Node.js application configuration.


Test and Manage Nginx Service<br>
! Ensure configurations are correct and manage the Nginx service using OpenRC !<br>

## Test Nginx configuration for syntax errors

```bash
nginx -t
```

```bash
# Start Nginx service
rc-service nginx start
```

```bash
# Stop Nginx service
rc-service nginx stop
```

```bash
# Restart Nginx service
rc-service nginx restart
```

```bash
# Reload Nginx configuration without downtime
rc-service nginx reload
```

```bash
# Enable Nginx to start on boot
rc-update add nginx default
```

Descriptions:

`nginx -t`: Tests the Nginx configuration for syntax errors and validity.<br>
`rc-service nginx start`: Starts the Nginx service.<br>
`rc-service nginx stop`: Stops the Nginx service.<br>
`rc-service nginx restart`: Restarts the Nginx service.<br>
`rc-service nginx reload`: Reloads the Nginx configuration without stopping the service.<br>
`rc-update add nginx default`: Enables Nginx to start automatically on system boot.<br>

## Node.js Setup

### Install Node.js and npm

```bash
# Sync the Portage tree
emerge --sync

# Update the system to ensure all packages are up to date
emerge -avuDN @world

# Install Node.js and npm
emerge --ask dev-lang/nodejs
```

Configure Node.js Environment

1.Set Up a Dedicated User for Node.js Applications:
```bash
useradd -m -G users,wheel -s /bin/bash nodeuser
passwd nodeuser
```

2.Switch to the Node.js User:

```bash
su - nodeuser
```

3.Initialize a New Node.js Project:
```bash
mkdir ~/myapp
cd ~/myapp
npm init -y
```

Install Essential Node.js Packages
```bash
npm install express helmet morgan rate-limiter-flexible
```

- **express: Web framework for Node.js.**
- **helmet: Secures Express apps by setting various HTTP headers.**
- **morgan: HTTP request logger middleware.**
- **rate-limiter-flexible: Flexible rate limiter for Express.**

Create a Secure Express Application<br>
1.Create `app.js`:
```bash
const express = require('express');
const helmet = require('helmet');
const morgan = require('morgan');
const { RateLimiterMemory } = require('rate-limiter-flexible');

const app = express();

// Set security-related HTTP headers
app.use(helmet());

// Setup request logging
app.use(morgan('combined'));

// Rate limiting middleware
const rateLimiter = new RateLimiterMemory({
  points: 100, // Number of points
  duration: 60, // Per second
});

app.use((req, res, next) => {
  rateLimiter.consume(req.ip)
    .then(() => {
      next();
    })
    .catch(() => {
      res.status(429).send('Too Many Requests');
    });
});

app.get('/', (req, res) => {
  res.send('Hello, secure Node.js app!');
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
```

2.Secure Application Configuration:
  - **Environment Variables: Store sensitive information like API keys and database credentials in environment variables or a `.env` file using packages like `dotenv.`**

```bash
npm install dotenv
```

```bash
require('dotenv').config();
const dbPassword = process.env.DB_PASSWORD;
```

Set Up Reverse Proxy with Nginx<br>
Configure Nginx as a Reverse Proxy:

```bash
nano /etc/nginx/sites-available/nodejs-app.conf
```

Example Configuration:

```bash
server {
    listen 80;
    server_name node.example.com;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_cache_bypass $http_upgrade;
    }

    # Optional: Enable SSL
    listen 443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/node.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/node.example.com/privkey.pem;

    include /etc/nginx/snippets/ssl-params.conf;
}
```

2.Enable the Node.js Site Configuration:

```bash
ln -s /etc/nginx/sites-available/nodejs-app.conf /etc/nginx/sites-enabled/nodejs-app.conf
```

3.Test and Reload Nginx:

```bash
nginx -t
rc-service nginx reload
```

Implement Proper Logging<br>
1.Configure Log Rotation for Application Logs:

```bash
nano /etc/logrotate.d/nodejs-app
```

Example Configuration:

```bash
/var/www/myapp/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 nodeuser www-data
    sharedscripts
    postrotate
        systemctl reload nginx > /dev/null 2>/dev/null || true
    endscript
}
```

2.Use a Process Manager (e.g., PM2) for Enhanced Logging and Management:

```bash
npm install pm2 -g
```

```bash
pm2 start app.js
pm2 startup
pm2 save
```

Note: PM2 can be configured to start on system boot and manage logs effectively.<br><br>

## Security Best Practices
Keep Dependencies Updated:<br>

Regularly update Node.js and all npm packages to patch vulnerabilities.

```bash
npm update
```

Use HTTPS:

- **Always serve your application over HTTPS to encrypt data in transit.**
- **Validate and Sanitize Inputs:**
- **Ensure all user inputs are validated and sanitized to prevent injection attacks.**
- **Implement CORS Policies:**
- **Configure Cross-Origin Resource Sharing (CORS) appropriately.**

```bash
npm install cors
```

```bash
const cors = require('cors');
app.use(cors({
  origin: 'https://yourdomain.com',
  optionsSuccessStatus: 200
}));
```

Limit Request Sizes:<br><br>

Prevent denial-of-service (DoS) attacks by limiting the size of incoming requests.

```bash
app.use(express.json({ limit: '10kb' }));
app.use(express.urlencoded({ limit: '10kb', extended: true }));
```

Start and Enable Node.js Application<br>
1.Using PM2:

```bash
pm2 start app.js
pm2 startup
pm2 save
```

2.Using OpenRC:<br>
Create a service script for your Node.js application.

```bash
nano /etc/init.d/nodejs-app
```

Example Service Script:

```bash
#!/sbin/openrc-run
command="/usr/bin/node"
command_args="/var/www/myapp/app.js"
name="nodejs-app"
description="Node.js Application"

depend() {
    need net
    after nginx
}
```

Make the script executable and add it to the default runlevel.

```bash
chmod +x /etc/init.d/nodejs-app
rc-update add nodejs-app default
rc-service nodejs-app start
```
<br><br>


## PHP Installation

### Install PHP and Extensions

```bash
# Add USE flags for PHP
echo "dev-lang/php fpm mysql mysqli pdo_mysql" >> /etc/portage/package.use/php
```

## Install PHP with necessary extensions
```bash
emerge --ask dev-lang/php dev-lang/php-extensions
```

Configure PHP-FPM
1.Copy Default Configuration:
```bash
cp /etc/php/fpm/php-fpm.conf.default /etc/php/fpm/php-fpm.conf
cp /etc/php/fpm/www.conf.default /etc/php/fpm/www.conf
```

2.Edit php-fpm.conf:
```bash
nano /etc/php/fpm/php-fpm.conf
```

Example Configuration:
```bash
[global]
pid = /var/run/php-fpm.pid
error_log = /var/log/php-fpm/error.log
include=/etc/php/fpm/pool.d/*.conf
```

3.Edit www.conf:
```bash
nano /etc/php/fpm/www.conf
```

Example Configuration:
```bash
[www]
user = nginx
group = nginx
listen = /run/php-fpm/php-fpm.sock
listen.owner = nginx
listen.group = nginx
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
```

Start and Enable PHP-FPM
```bash
# Start PHP-FPM service
rc-service php-fpm start

# Enable PHP-FPM to start on boot
rc-update add php-fpm default
```

Verify PHP Installation<br>
Create a `phpinfo.php` file to verify PHP is working correctly.

```bash
echo "<?php phpinfo(); ?>" > /var/www/example.com/phpinfo.php
```

Navigate to `http://example.com/phpinfo.php` in your browser. You should see the PHP information page. Remove this file after verification for security reasons.

```bash
rm /var/www/example.com/phpinfo.php
```

## Database setup

Install MariaDB
```bash
# Install MariaDB
emerge --ask dev-db/mariadb

# Initialize MariaDB
mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql

# Start MariaDB service
rc-service mysql start

# Enable MariaDB to start on boot
rc-update add mysql default
```

Secure MariaDB Installation<br>
Run the secure installation script to improve the security of your MariaDB setup.

```bash
mysql_secure_installation
```

Follow the prompts:<br>

- Set a strong root password.
- Remove anonymous users.
- Disallow root login remotely.
- Remove test database.
- Reload privilege tables.

Create a Database and User for WordPress<br>
1.Log into MariaDB as Root:

```bash
mysql -u root -p
```

2.Create Database:

```bash
CREATE DATABASE wordpress_db;
```

3.Create User and Grant Permissions:

```bash
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'strong_password';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

Test Database Connection
Ensure that the newly created user can connect to the database.

```bash
mysql -u wp_user -p wordpress_db
```

## SSL Certificates
Install Certbot

```bash
# Install Certbot for obtaining SSL certificates
emerge --ask dev-util/certbot
```

Obtain SSL Certificate
Use Certbot to obtain and install SSL certificates for your domain.

```bash
certbot certonly --webroot -w /var/www/example.com -d example.com -d www.example.com
```

Follow the prompts to complete the certificate issuance.

Configure Nginx to Use SSL
Edit your Nginx server block to include SSL settings.

```bash
nano /etc/nginx/sites-available/wordpress.conf
```

Example Configuration:
```bash
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.php index.html;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
    add_header Permissions-Policy "camera=(), geolocation=(), microphone=()" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

    # Logging
    access_log /var/log/nginx/sites/example.com/access.log main_ext;
    error_log /var/log/nginx/sites/example.com/error.log warn;

    # PHP Handling with FastCGI Cache
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;

        # FastCGI Cache Configuration
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 60m;
        fastcgi_cache_use_stale error timeout invalid_header updating http_500;
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;
    }

    # Additional Configurations...
}
```

Redirect HTTP to HTTPS
Ensure all HTTP traffic is redirected to HTTPS by updating your HTTP server block.

```bash
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # Redirect all HTTP requests to HTTPS
    return 301 https://$host$request_uri;
}
```

Test and Reload Nginx

```bash
# Test Nginx configuration for syntax errors
nginx -t

# Reload Nginx to apply changes
rc-service nginx reload
```

Automate Certificate Renewal
Certbot sets up a cron job for automatic renewal. Verify the cron job exists.

```bash
cat /etc/crontab | grep certbot
```

If not present, add the following to renew certificates automatically:

```bash
echo "0 3 * * * root certbot renew --quiet && rc-service nginx reload" >> /etc/crontab
```

## WordPress Configuration

### Download and Install WordPress

```bash
# Change to the web root directory
cd /var/www/example.com

# Download the latest WordPress package
wget https://wordpress.org/latest.tar.gz

# Extract the WordPress package
tar -xzvf latest.tar.gz

# Move WordPress files to the root of the web directory
mv wordpress/* .

# Remove unnecessary files
rm -rf wordpress latest.tar.gz

# Set the correct ownership
chown -R nginx:nginx /var/www/example.com
```

Create WordPress Configuration File
1.Copy the Sample Configuration:

```bash
cp wp-config-sample.php wp-config.php
```

2.Edit wp-config.php:

```bash
nano wp-config.php
```

Update the following settings:

```bash
<?php
// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress_db' );

/** MySQL database username */
define( 'DB_USER', 'wp_user' );

/** MySQL database password */
define( 'DB_PASSWORD', 'strong_password' );

/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```

3.Set Unique Keys and Salts:

Visit `https://api.wordpress.org/secret-key/1.1/salt/` to generate unique keys, then replace the placeholders in `wp-config.php` with the generated values.

Configure Nginx for WordPress Permalinks
1.Edit Nginx Server Block:

```bash
nano /etc/nginx/sites-available/wordpress.conf
```

2.Add the Following Location Block Inside the Server Block:

```bash
location / {
    try_files $uri $uri/ /index.php?$args;
}

location ~ \.php$ {
    try_files $uri =404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    include fastcgi_params;
}
```

Secure WordPress
1.Set Proper File Permissions:

```bash
find /var/www/example.com/ -type d -exec chmod 755 {} \;
find /var/www/example.com/ -type f -exec chmod 644 {} \;
```

2.Protect `wp-config.php`:

```bash
location ~ ^/wp-config.php$ {
    deny all;
}
```

Complete WordPress Installation
1.Access WordPress in a Browser:

Navigate to `https://example.com` and follow the on-screen instructions to complete the installation.

2.Remove Installation Directory (if applicable).

Not necessary in WordPress, but ensure security.

## Server Hardening

IPTables Configuration
Update IPTables to Block Additional Countries
Create or update the `update-country-blocks.sh` script to block specific countries.

```bash
# /usr/local/sbin/update-country-blocks.sh
#!/bin/bash

# Countries to block
COUNTRIES=("CN" "RU" "BY" "KP" "IR" "IN" "PK" "DZ" "AO" "EG" "NG" "ZA" "KE" "UG" "GH" "TZ" "MA" "TN" "CM" "IQ" "SY")

TMPDIR="/tmp/country-blocks"

# Create temporary directory
mkdir -p $TMPDIR

# Download latest country IP lists
for country in "${COUNTRIES[@]}"; do
    wget -q "https://www.ipdeny.com/ipblocks/data/countries/${country}.zone" -O "$TMPDIR/${country}.zone"
done

# Create ipset for each country
for country in "${COUNTRIES[@]}"; do
    # Destroy existing set if exists
    ipset destroy ${country,,} 2>/dev/null
    # Create new set
    ipset create ${country,,} hash:net
    # Add IPs to set
    while IFS= read -r ip; do
        ipset add ${country,,} $ip
    done < "$TMPDIR/${country}.zone"
done

# Clean up
rm -rf $TMPDIR

echo "Country IP sets updated."
```

Make the script executable:

```bash
chmod +x /usr/local/sbin/update-country-blocks.sh
```

Enhanced IPTables Configuration
Update `/etc/iptables/firewall.rules` to include the additional countries:

```bash
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
:FAIL2BAN - [0:0]

# Whitelist admin IP addresses
-A INPUT -p tcp -s <ADMIN_IP1> --dport 2222 -j ACCEPT
-A INPUT -p tcp -s <ADMIN_IP2> --dport 2222 -j ACCEPT

# Allow loopback
-A INPUT -i lo -j ACCEPT

# Allow established connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Drop invalid packets
-A INPUT -m state --state INVALID -j DROP

# Allow SSH on custom port with rate limiting
-A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --set
-A INPUT -p tcp --dport 2222 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
-A INPUT -p tcp --dport 2222 -j ACCEPT

# Allow HTTP and HTTPS
-A INPUT -p tcp --dport 80 -j ACCEPT
-A INPUT -p tcp --dport 443 -j ACCEPT

# Block traffic from specified countries
-A INPUT -m set --match-set cn src -j DROP
-A INPUT -m set --match-set ru src -j DROP
-A INPUT -m set --match-set by src -j DROP
-A INPUT -m set --match-set kp src -j DROP
-A INPUT -m set --match-set ir src -j DROP
-A INPUT -m set --match-set in src -j DROP
-A INPUT -m set --match-set pk src -j DROP
-A INPUT -m set --match-set dz src -j DROP
-A INPUT -m set --match-set ao src -j DROP
-A INPUT -m set --match-set eg src -j DROP
-A INPUT -m set --match-set ng src -j DROP
-A INPUT -m set --match-set za src -j DROP
-A INPUT -m set --match-set ke src -j DROP
-A INPUT -m set --match-set ug src -j DROP
-A INPUT -m set --match-set gh src -j DROP
-A INPUT -m set --match-set tz src -j DROP
-A INPUT -m set --match-set ma src -j DROP
-A INPUT -m set --match-set tn src -j DROP
-A INPUT -m set --match-set cm src -j DROP
-A INPUT -m set --match-set iq src -j DROP
-A INPUT -m set --match-set sy src -j DROP
-A INPUT -m set --match-set vn src -j DROP
-A INPUT -m set --match-set id src -j DROP
-A INPUT -m set --match-set tr src -j DROP
-A INPUT -m set --match-set bd src -j DROP

# ICMP rate limiting
-A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT

# Rate limiting for SSH
-A INPUT -p tcp --dport 2222 -m limit --limit 10/minute --limit-burst 20 -j ACCEPT

# Rate limiting for HTTP
-A INPUT -p tcp --dport 80 -m limit --limit 50/minute --limit-burst 200 -j ACCEPT

# Rate limiting for HTTPS
-A INPUT -p tcp --dport 443 -m limit --limit 50/minute --limit-burst 200 -j ACCEPT

# Protect against SYN flood
-A INPUT -p tcp ! --syn -m state --state NEW -j DROP
-A INPUT -p tcp --syn -m limit --limit 1/s --limit-burst 4 -j ACCEPT

# Drop invalid packets
-A INPUT -m state --state INVALID -j DROP

# Custom chain for fail2ban
-A INPUT -j FAIL2BAN

# Log dropped traffic
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPTables Dropped: " --log-level 4

COMMIT
```

Explanation:

Added ipset rules to block traffic from:

- **Russia (RU):** Known for advanced cyber-attacks and state-sponsored malicious activities.
- **China (CN):** High volume of unsolicited traffic, potential source of vulnerability scans.
- **Belarus (BY):** Increasing instances of cyber threats with emerging malicious actors.
- **India (IN):** Significant number of login attempts and potential for brute-force attacks.
- **Pakistan (PK):** Frequent sources of malicious traffic and security threats.
- **Algeria (DZ):** Notable for suspicious traffic patterns.
- **Angola (AO):** Reports of cyber threats and malicious activities.
- **Egypt (EG):** Associated with higher rates of attacks.
- **Nigeria (NG):** Common source of spam and malicious traffic.
- **South Africa (ZA):** Increasing cyber threats targeting web servers.
- **Kenya (KE):** Frequent sources of unsolicited traffic.
- **Uganda (UG):** Observed as a source of malicious activities.
- **Ghana (GH):** Higher incidence of cyber threats.
- **Tanzania (TZ):** Notable for suspicious traffic.
- **Morocco (MA):** Increased malicious traffic attempts.
- **Tunisia (TN):** Reports of cyber threats.
- **Cameroon (CM):** Sources of malicious traffic.
- **Iraq (IQ):** Associated with higher cyber-attack rates.
- **Syria (SY):** Known for state-sponsored cyber activities.
- **Vietnam (VN):** Increasing number of cyber-attacks targeting web servers.
- **Indonesia (ID):** Notable for unsolicited traffic and potential threats.
- **Turkey (TR):** Reports of cyber threats and attacks.
- **Bangladesh (BD):** Sources of malicious traffic activities.

These countries have been identified based on security assessments and observed malicious traffic patterns. Blocking them helps in reducing the attack surface and protecting the server from potential threats.<br><br>
Can be extended to additional countries.<br><br>



#### 1. Whitelist Admin IP Addresses

To prevent locking yourself out, whitelist trusted admin IP addresses by allowing SSH connections from these IPs before applying the general SSH rules.

```bash
# Whitelist admin IP addresses
-A INPUT -p tcp -s <ADMIN_IP1> --dport 2222 -j ACCEPT
-A INPUT -p tcp -s <ADMIN_IP2> --dport 2222 -j ACCEPT
```

#### 2.Replace <ADMIN_IP1> and <ADMIN_IP2> with your actual admin IP addresses.

```bash
# Drop invalid packets
-A INPUT -m state --state INVALID -j DROP
```

#### 3. Enable Reverse Path Filtering
Improve protection against IP spoofing by enabling reverse path filtering.

```bash
# Enable Reverse Path Filtering
echo 1 > /proc/sys/net/ipv4/conf/all/rp_filter
echo 1 > /proc/sys/net/ipv4/conf/default/rp_filter
```

To make these changes persistent, add the following lines to `/etc/sysctl.conf`:

```bash
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
```

### 4. Limit ICMP Types
Allow only specific ICMP types to control network diagnostics traffic.

```bash
# Allow specific ICMP types
-A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
-A INPUT -p icmp --icmp-type destination-unreachable -j ACCEPT
-A INPUT -p icmp --icmp-type time-exceeded -j ACCEPT
-A INPUT -p icmp --icmp-type parameter-problem -j ACCEPT
-A INPUT -p icmp -j DROP
```

### 5. Enhance Logging for Dropped Traffic
Improve logging to capture more details about dropped packets for better monitoring and analysis.

```bash
# Log dropped traffic
-A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPTables Dropped: " --log-level 4
```

<br>
Backup Current iptables Configuration: Before making any changes, backup your existing `firewall.rules` to prevent accidental lockouts.

```bash
cp /etc/iptables/firewall.rules /etc/iptables/firewall.rules.backup
```

Apply and Test Changes Carefully: After updating the rules, apply them and verify connectivity, especially SSH access from whitelisted IPs.

```bash
iptables-restore < /etc/iptables/firewall.rules
```

Use SSH from Whitelisted IPs: Always manage your server from the whitelisted admin IP addresses to maintain uninterrupted access.<br>
Regularly Review and Update Rules: Periodically assess and update your firewall rules to adapt to evolving security threats and administrative needs.<br>
Implement Additional Security Layers: Consider using intrusion detection systems (IDS) like Fail2Ban, regular security audits with Lynis, and keeping all software up to date.<br>
<br>

## Intrusion Detection Systems

### Introduction<br>
Implementing an Intrusion Detection System (IDS) like Snort or Suricata adds an additional layer of security by monitoring network traffic for suspicious activities and potential threats.
<br><br><br>
<h1><b></b>Some of these recommendations require powerful hardware with lots of RAM<br> (Elasticsearch, Kibana, etc.. eat up RAM like cookies)</b></h1>
<br><br>
If you don't have hardware with 32Gb RAM,... just tinker around the monitoring, drop the elasticsearch, kibana, snort, suricata - and you can fit this configuration into the small VPS with the 2-4Gb RAM (for a few websites with moderate traffic).<br>
<br><br>

## Option 1: Setting Up Suricata<br>
Install Suricata

```bash
emerge --ask net-analyzer/suricata
```

Configure Suricata<br>
1.Copy Default Configuration:

```bash
cp /etc/suricata/suricata.yaml.example /etc/suricata/suricata.yaml
```

2.Edit suricata.yaml:

```bash
nano /etc/suricata/suricata.yaml
```

- **Interface Configuration:** 
Specify the network interface to monitor.

```bash
af-packet:
  - interface: eth0
    threads: auto
    defrag: yes
```

- **Logging Configuration:** 
Define logging preferences.

```bash
outputs:
  - eve-log:
      enabled: yes
      filetype: regular
      filename: /var/log/suricata/eve.json
      types:
        - alert:
            payload: yes
            payload-printable: yes
            packet: yes
            hostnames: yes
            http: yes
```

Enable and Start Suricata

```bash
rc-service suricata start
rc-update add suricata default
```

Update Ruleset<br>
1.Install suricata-update:

```bash
emerge --ask net-analyzer/suricata-update
```

2.Update and Manage Rules:

```bash
suricata-update update-sources
suricata-update
rc-service suricata restart
```

<br><br>
## Option 2: Setting Up Snort<br>

Install Snort

```bash
emerge --ask net-analyzer/snort
```

Configure Snort<br>
1. Copy Default Configuration:

```bash
cp /etc/snort/snort.conf.example /etc/snort/snort.conf
```

2.Edit snort.conf:

```bash
nano /etc/snort/snort.conf
```

Variable Definitions:<br>
Set network variables.

```bash
var HOME_NET 192.168.1.0/24
var EXTERNAL_NET !$HOME_NET
```

Include Rules:<br>
Include necessary rule files.

```bash
include $RULE_PATH/local.rules
include $RULE_PATH/community.rules
```

Output Configuration:<br>
Define output formats.

```bash
output alert_fast: stdout
output alert_syslog: LOG_AUTH LOG_ALERT
```

Enable and Start Snort

```bash
rc-service snort start
rc-update add snort default
```

Update Ruleset
1.Subscribe to Snort Rules:<br>
  Register and obtain a ruleset from Snort.org.<br><br>
2.Download and Install Rules:

```bash
snort-update download-community-rules
snort-update install-community-rules
rc-service snort restart
```

## Integrate IDS with Logging and Alerts <br>
1.Install Elasticsearch, Logstash, and Kibana (ELK Stack) for Log Management:

```bash
emerge --ask dev-db/elasticsearch dev-app-analogue/logstash dev-app-analogue/kibana
```

2.Configure Logstash to Parse IDS Logs:

```bash
nano /etc/logstash/conf.d/suricata.conf
```

Example Configuration for Suricata:

```bash
input {
  file {
    path => "/var/log/suricata/eve.json"
    codec => json
  }
}

filter {
  if [event_type] == "alert" {
    mutate { rename => { "src_ip" => "source.ip" } }
    mutate { rename => { "dest_ip" => "destination.ip" } }
    mutate { rename => { "alert.signature" => "event.signature" } }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "suricata-%{+YYYY.MM.dd}"
  }
}
```

3.Configure Kibana Dashboards for IDS Alerts:<br>

Access Kibana at `http://your_server_ip:5601` and set up dashboards to visualize and monitor IDS alerts.<br><br>

Best Practices for IDS<br>
Regularly Update Rulesets: Keep your IDS rules updated to recognize the latest threats.

```bash
suricata-update
rc-service suricata restart
```

- **Monitor Logs Continuously: Use centralized logging solutions like the ELK Stack to analyze and visualize IDS alerts.**
- **Implement Automated Responses: Configure scripts or tools to respond to certain IDS alerts automatically, such as blocking offending IPs.**
- **Conduct Regular Audits: Periodically review IDS logs and alerts to assess the security posture and adjust rules as necessary.**
<br>

Summary<br>
Implementing a robust Intrusion Detection System like Suricata or Snort significantly enhances your server's security by providing real-time monitoring and alerting capabilities. Coupled with proper configuration, regular updates, and integration with logging tools, an IDS serves as a critical component in defending against unauthorized access and malicious activities.<br><br>

## Automated Security Audits and Email Reporting<br>


## Install Lynis

Lynis is used for Security Audits.

```bash
emerge --ask sys-apps/lynis
```

Configure Lynis to Perform Periodic Security Audits<br>
Create a cron job to run Lynis weekly and email the results.<br>

## Create a script for running Lynis and emailing the report
```bash
cat > /usr/local/sbin/lynis_audit.sh <<'EOF'
#!/bin/bash

# Define email variables
EMAIL="admin@example.com"
SUBJECT="Weekly Lynis Security Audit Report - $(date +%F)"
REPORT="/var/log/lynis_audit_report.txt"

# Run Lynis audit and save the report
lynis audit system > "$REPORT"

# Send the report via email
mail -s "$SUBJECT" "$EMAIL" < "$REPORT"
EOF
```

## Make the script executable
```bash
chmod +x /usr/local/sbin/lynis_audit.sh
```

## Schedule the cron job to run every Sunday at 2 AM
```bash
echo "0 2 * * 0 /usr/local/sbin/lynis_audit.sh" >> /etc/crontab
```

## Monitoring Setup

Real-Time Monitoring and Alerts: Utilize Netdata

## Install Netdata
```bash
emerge --ask net-analyzer/netdata
```

## Start Netdata service
```bash
rc-service netdata start
```

## Enable Netdata to start on boot
```bash
rc-update add netdata default
```

Configure Netdata Email Alerts<br>
1.Configure Mail Transfer Agent (MTA):<br>

Ensure an MTA like postfix is installed and configured (as shown in Email Configuration).

## Install Postfix
```bash
emerge --ask mail-mta/postfix
```

## Configure Postfix
```bash
nano /etc/postfix/postfix.conf
```

Basic Configuration Example:

```bash
smtpd_banner = $myhostname ESMTP $mail_name (Gentoo)
myhostname = mail.example.com
mydomain = example.com
myorigin = /etc/mailname
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
relayhost =
mynetworks = 127.0.0.0/8
home_mailbox = Maildir/
smtpd_tls_security_level = may
smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
smtpd_use_tls = yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

2.Configure Netdata Alerts:<br>

Edit Netdata's health configuration to define alert conditions.
```bash
nano /etc/netdata/health.d/netdata.conf
```

Add Alert Definitions:

```bash
alarm: high_cpu
on: system.cpu
lookup: average -1m > 80
units: %
every: 10s
warn: 80
crit: 90
info: "High CPU usage detected"
to: admin@example.com

alarm: traffic_spike
on: net.in
lookup: average -5m > 1000
units: bps
every: 1m
warn: 1000
crit: 1500
info: "Traffic spike detected"
to: admin@example.com

alarm: website_down
on: http.up
lookup: current != 1
units: status
every: 1m
warn: 0
crit: 0
info: "Website is down"
to: admin@example.com
```

3.Restart Netdata to Apply Changes:
```bash
rc-service netdata restart
```

Email Configuration: Set Up a Reliable MTA


## Install Postfix
```bash
emerge --ask mail-mta/postfix
```

## Configure Postfix
```bash
nano /etc/postfix/postfix.conf
```

Basic Configuration Example:

```bash
smtpd_banner = $myhostname ESMTP $mail_name (Gentoo)
myhostname = mail.example.com
mydomain = example.com
myorigin = /etc/mailname
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain
relayhost =
mynetworks = 127.0.0.0/8
home_mailbox = Maildir/
smtpd_tls_security_level = may
smtpd_tls_cert_file = /etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file = /etc/ssl/private/ssl-cert-snakeoil.key
smtpd_use_tls = yes
smtpd_tls_session_cache_database = btree:${data_directory}/smtpd_scache
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

Start and Enable Postfix

## Start Postfix service
```bash
rc-service postfix start
```

## Enable Postfix to start on boot
```bash
rc-update add postfix default
```

## Virus Protection

## Install ClamAV
```bash
emerge --ask app-antivirus/clamav
```

## Update ClamAV database
```bash
freshclam
```

## Start ClamAV daemon
```bash
rc-service clamav-daemon start
```

## Enable ClamAV to start on boot
```bash
rc-update add clamav-daemon default
```

Configure Regular Virus Scans<br>
Create a cron job to perform daily virus scans and email the results.<br>

## Create a script for running ClamAV scans
```bash
cat > /usr/local/sbin/clamav_scan.sh <<'EOF'
#!/bin/bash

# Define email variables
EMAIL="admin@example.com"
SUBJECT="Daily ClamAV Virus Scan Report - $(date +%F)"
REPORT="/var/log/clamav_scan_report.txt"

# Run ClamAV scan and save the report
clamscan -r /var/www > "$REPORT"

# Send the report via email
mail -s "$SUBJECT" "$EMAIL" < "$REPORT"
EOF
```

## Make the script executable
```bash
chmod +x /usr/local/sbin/clamav_scan.sh
```

## Schedule the cron job to run daily at 3 AM
```bash
echo "0 3 * * * root /usr/local/sbin/clamav_scan.sh" >> /etc/crontab
```

## Automate IPSet Updates<br>
Ensure that the `update-country-blocks.sh` script runs periodically to keep IP sets updated.<br>

## Already scheduled in [Firewall Configuration](#firewall-configuration) <br>
## Ensure the cron job exists

```bash
grep 'update-country-blocks.sh' /etc/crontab || echo "0 1 * * * root /usr/local/sbin/update-country-blocks.sh" >> /etc/crontab
```

## Implement SELinux or AppArmor<br>
For enhanced security policies (optional).

## Install AppArmor
```bash
emerge --ask sys-apps/apparmor
```

## Start AppArmor service
```bash
rc-service apparmor start
```

## Enable AppArmor to start on boot
```bash
rc-update add apparmor default
```

Configure AppArmor<br>
Generate profiles for critical applications or use predefined profiles.<br>

## Example: Generate profile for Nginx
```bash
aa-genprof nginx
```

## Follow prompts to create and enforce the profile

## Final Recommendations

- **Regular Updates:** Continuously update your server and all installed software to ensure you have the latest security patches and features.

- **Backup Strategy:** Implement a regular backup routine for your website files and databases. Store backups securely and test restoration processes periodically.

- **Monitor Server Performance:** Use monitoring tools like Netdata to keep an eye on server performance, resource usage, and potential issues.

- **Enhance Security Practices:**
  - **Strong Passwords:** Enforce the use of strong, unique passwords for all user accounts.
  - **Two-Factor Authentication (2FA):** Encourage or enforce the use of 2FA for all administrative accounts.
  - **Least Privilege Principle:** Grant users only the minimum permissions necessary to perform their tasks.

- **Log Management:** Regularly review and analyze server logs to detect and respond to suspicious activities promptly.

- **Automate Routine Tasks:** Use cron jobs to automate maintenance tasks such as updates, backups, and security scans.

- **Optimize Nginx and PHP-FPM Configuration:** Regularly review and optimize your Nginx and PHP-FPM settings for performance and security.

- **Stay Informed:** Keep abreast of the latest security threats and best practices related to server administration and WordPress management.

- **Documentation:** Maintain comprehensive documentation of your server setup, configurations, and procedures to aid in maintenance and troubleshooting.

- **Disaster Recovery Plan:** Develop and regularly update a disaster recovery plan to ensure business continuity in case of server failure or data loss.

- **Regular Security Audits:** Use tools like Lynis to perform periodic security audits and address any identified vulnerabilities.
<br><br>
THE END?

