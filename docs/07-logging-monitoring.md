<i>Enable basic logging, mail delivery (for system alerts), and lightweight real-time monitoring — ideal for small VPS setups.</i>

# 07 – Logging, Mail & Monitoring

> Set up log rotation, system email (via `msmtp`), and lightweight monitoring with `netdata`.

---

## Step 1: Configure Logrotate

Gentoo uses `logrotate` by default. Check `/etc/logrotate.conf`:

```bash
cat /etc/logrotate.conf
```

It includes:

```bash
include /etc/logrotate.d
```

Each installed service (e.g., nginx) will drop its own config in `/etc/logrotate.d/`.

Test rotation manually:

```bash
logrotate --debug /etc/logrotate.conf
```

Or force a run:

```bash
logrotate -f /etc/logrotate.conf
```

## Step 2: Setup msmtp for System Mail

Use msmtp to send outgoing mail (e.g., cron or logwatch).

Install:

```bash
emerge -av mail-mta/msmtp
```

Create config `/etc/msmtprc`:

```bash
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        default
host           smtp.mailprovider.com
port           587
from           your@email.com
user           your@email.com
passwordeval   "cat /etc/msmtp-password"
```

Create password file:

```bash
echo "your_password_here" > /etc/msmtp-password
chmod 600 /etc/msmtp-password
chown root:root /etc/msmtp-password
```

Test sending:

```bash
echo "Test" | msmtp your@email.com
```

Optional: send root mail to your address:

```bash
echo "your@email.com" > /root/.forward
```

Also set sendmail alias:

```bash
ln -s /usr/bin/msmtp /usr/sbin/sendmail
```

## Step 3: Netdata (Real-Time Monitoring)

Install:

```bash
emerge -av net-analyzer/netdata
```

Start it:

```bash
rc-update add netdata default
/etc/init.d/netdata start
```

Or with systemd:

```bash
systemctl enable --now netdata
```

Then visit:

```bash
http://your-server-ip:19999
```

To enable TLS or proxy via NGINX, add a reverse proxy like:

```nginx
location /netdata/ {
    proxy_pass http://127.0.0.1:19999/;
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

Optional: restrict IPs with allow/deny or a password.

## Step 4: WordPress-Specific Monitoring

Configure PHP-FPM status page for monitoring:

Edit `/etc/php/fpm-php*/fpm.d/www.conf` and add:

```bash
pm.status_path = /status
ping.path = /ping
```

Add nginx configuration for PHP-FPM monitoring:

```nginx
# Add to your WordPress server block
location ~ ^/(status|ping)$ {
    access_log off;
    allow 127.0.0.1;
    deny all;
    fastcgi_pass unix:/run/php-fpm.sock;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
}
```

Create WordPress monitoring script:

```bash
nano /usr/local/bin/wp-monitor.sh
```

```bash
#!/bin/bash
# WordPress-specific monitoring script

LOG_FILE="/var/log/wp-monitor.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Check WordPress site response
WP_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost/)
if [ "$WP_RESPONSE" != "200" ]; then
    echo "$DATE - WARNING: WordPress not responding (HTTP $WP_RESPONSE)" >> $LOG_FILE
fi

# Check PHP-FPM status
PHP_STATUS=$(curl -s http://localhost/status | grep "idle processes" | wc -l)
if [ "$PHP_STATUS" -eq 0 ]; then
    echo "$DATE - WARNING: PHP-FPM status page not accessible" >> $LOG_FILE
fi

# Check database connectivity
DB_CHECK=$(mysql -u wp_user -p"your_password" -e "SELECT 1" wordpress_db 2>/dev/null | wc -l)
if [ "$DB_CHECK" -eq 0 ]; then
    echo "$DATE - WARNING: WordPress database not accessible" >> $LOG_FILE
fi

# Check critical WordPress files
if [ ! -f "/var/www/wordpress/wp-config.php" ]; then
    echo "$DATE - ERROR: wp-config.php missing!" >> $LOG_FILE
fi

# Monitor upload directory space
UPLOAD_SIZE=$(du -sm /var/www/wordpress/wp-content/uploads 2>/dev/null | cut -f1)
if [ "$UPLOAD_SIZE" -gt 1000 ]; then  # Alert if uploads > 1GB
    echo "$DATE - INFO: WordPress uploads directory is ${UPLOAD_SIZE}MB" >> $LOG_FILE
fi
```

Make executable and schedule:

```bash
chmod +x /usr/local/bin/wp-monitor.sh
echo "*/5 * * * * /usr/local/bin/wp-monitor.sh" >> /var/spool/cron/crontabs/root
```

## (Optional) Alerting

Enable email alerts for `netdata`, `cron`, `fail2ban`, etc., by ensuring:
- msmtp is used as sendmail
- root mail forwards to your address
- You’ve tested sending manually


