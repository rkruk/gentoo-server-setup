<i>Advanced security and performance enhancements for production Gentoo VPS deployments. File integrity monitoring, AppArmor profiles, automated updates, and performance baselines.</i>

# 10 – Advanced Hardening & Monitoring

> **Optional but recommended:** Enhanced security measures and performance monitoring for production VPS environments.

---

## Step 1: File Integrity Monitoring with AIDE

AIDE (Advanced Intrusion Detection Environment) monitors critical system files for unauthorized changes.

Install AIDE:

```bash
emerge -av app-forensics/aide
```

Create initial configuration:

```bash
nano /etc/aide/aide.conf
```

Basic configuration for VPS monitoring:

```bash
# AIDE configuration for Gentoo VPS
database_in = file:/var/lib/aide/aide.db
database_out = file:/var/lib/aide/aide.db.new
database_new = file:/var/lib/aide/aide.db.new
gzip_dbout = yes

# File selection rules
/boot   NORMAL
/bin    NORMAL
/sbin   NORMAL
/lib    NORMAL
/lib64  NORMAL
/opt    NORMAL
/usr    NORMAL
/root   NORMAL

# Configuration files
/etc    NORMAL
!/etc/mtab
!/etc/.*~
!/etc/backup.*
!/etc/shadow-
!/etc/passwd-
!/etc/group-

# Log directories (content changes frequently)
!/var/log
!/var/cache
!/var/tmp
!/tmp

# Databases and dynamic content
!/var/lib/mysql
!/var/www

# SSH keys and certificates
/home NORMAL
!/home/.*/\..+
!/home/.*/.bash_history

# Define what NORMAL means
NORMAL = p+i+n+u+g+s+m+c+md5+sha256
```

Initialize the database:

```bash
aide --init
mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

Create weekly check script:

```bash
nano /usr/local/bin/aide-check.sh
```

```bash
#!/bin/bash

AIDE_LOG="/var/log/aide.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Starting AIDE integrity check..." >> "$AIDE_LOG"

# Run AIDE check
if aide --check > /tmp/aide-output.txt 2>&1; then
    echo "[$DATE] AIDE check completed - no changes detected" >> "$AIDE_LOG"
else
    echo "[$DATE] AIDE detected changes:" >> "$AIDE_LOG"
    cat /tmp/aide-output.txt >> "$AIDE_LOG"
    
    # Email alert if changes detected
    if command -v msmtp > /dev/null; then
        {
            echo "Subject: AIDE Alert - File Changes Detected on $(hostname)"
            echo ""
            echo "AIDE detected unauthorized file changes on $(hostname) at $DATE"
            echo ""
            cat /tmp/aide-output.txt
        } | msmtp -a default root@localhost
    fi
fi

rm -f /tmp/aide-output.txt
```

Make executable and schedule:

```bash
chmod +x /usr/local/bin/aide-check.sh

# Add to crontab (weekly check)
echo "0 2 * * 0 /usr/local/bin/aide-check.sh" >> /var/spool/cron/crontabs/root
```

## Step 2: AppArmor Profiles

AppArmor provides application-level security by confining programs to limited resources.

Install AppArmor:

```bash
emerge -av sys-apps/apparmor sys-apps/apparmor-utils
```

Enable AppArmor in kernel boot parameters:

```bash
# Add to GRUB_CMDLINE_LINUX in /etc/default/grub
GRUB_CMDLINE_LINUX="apparmor=1 security=apparmor"

# Update GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Create NGINX profile:

```bash
nano /etc/apparmor.d/usr.sbin.nginx
```

```bash
#include <tunables/global>

/usr/sbin/nginx {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/ssl_certs>

  capability dac_override,
  capability net_bind_service,
  capability setgid,
  capability setuid,

  /usr/sbin/nginx mr,
  /etc/nginx/ r,
  /etc/nginx/** r,
  /etc/ssl/certs/ r,
  /etc/ssl/certs/** r,
  
  /var/log/nginx/ rw,
  /var/log/nginx/** rw,
  /var/cache/nginx/ rw,
  /var/cache/nginx/** rw,
  
  /var/www/ r,
  /var/www/** r,
  
  /run/nginx.pid rw,
  /var/tmp/ rw,
  /tmp/ rw,

  # PHP-FPM socket
  /run/php-fpm/ rw,
  /run/php-fpm/** rw,
}
```

Create PHP-FPM profile:

```bash
nano /etc/apparmor.d/usr.sbin.php-fpm
```

```bash
#include <tunables/global>

/usr/sbin/php-fpm {
  #include <abstractions/base>
  #include <abstractions/nameservice>
  #include <abstractions/php>

  capability setgid,
  capability setuid,

  /usr/sbin/php-fpm mr,
  /etc/php/** r,
  
  /var/log/php-fpm/ rw,
  /var/log/php-fpm/** rw,
  
  /var/www/ r,
  /var/www/** rw,
  
  /run/php-fpm/ rw,
  /run/php-fpm/** rw,
  
  /tmp/ rw,
  /tmp/** rw,
  /var/tmp/ rw,
  /var/tmp/** rw,

  # Session files
  /var/lib/php/ rw,
  /var/lib/php/** rw,
}
```

Enable and load profiles:

```bash
rc-update add apparmor default
/etc/init.d/apparmor start

# Load profiles in complain mode first (for testing)
aa-complain /etc/apparmor.d/usr.sbin.nginx
aa-complain /etc/apparmor.d/usr.sbin.php-fpm

# After testing, enforce them
aa-enforce /etc/apparmor.d/usr.sbin.nginx
aa-enforce /etc/apparmor.d/usr.sbin.php-fpm
```

## Step 3: Automated Security Updates

Configure automatic updates for critical security packages.

Create update script:

```bash
nano /usr/local/bin/security-updates.sh
```

```bash
#!/bin/bash

LOG_FILE="/var/log/security-updates.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

echo "[$DATE] Starting security update check..." >> "$LOG_FILE"

# Sync portage tree
emerge --sync >> "$LOG_FILE" 2>&1

# Check for security updates
SECURITY_UPDATES=$(glsa-check -t all 2>/dev/null | wc -l)

if [ "$SECURITY_UPDATES" -gt 0 ]; then
    echo "[$DATE] Found $SECURITY_UPDATES security updates" >> "$LOG_FILE"
    
    # Apply security updates
    glsa-check -f all >> "$LOG_FILE" 2>&1
    
    # Update world with security focus
    emerge -uq --with-bdeps=y @security >> "$LOG_FILE" 2>&1
    
    # Send notification
    if command -v msmtp > /dev/null; then
        {
            echo "Subject: Security Updates Applied on $(hostname)"
            echo ""
            echo "Applied $SECURITY_UPDATES security updates on $(hostname) at $DATE"
            echo ""
            echo "Check /var/log/security-updates.log for details"
        } | msmtp -a default root@localhost
    fi
else
    echo "[$DATE] No security updates found" >> "$LOG_FILE"
fi

# Clean up
emerge --depclean -q >> "$LOG_FILE" 2>&1
```

Make executable and schedule:

```bash
chmod +x /usr/local/bin/security-updates.sh

# Daily security check at 3 AM
echo "0 3 * * * /usr/local/bin/security-updates.sh" >> /var/spool/cron/crontabs/root
```

## Step 4: Performance Monitoring & Baselines

Establish performance baselines and monitoring for your VPS.

Install monitoring tools:

```bash
emerge -av sys-process/htop sys-process/iotop net-analyzer/iftop
```

Create baseline collection script:

```bash
nano /usr/local/bin/collect-baseline.sh
```

```bash
#!/bin/bash

BASELINE_DIR="/var/log/baselines"
DATE=$(date '+%Y%m%d_%H%M%S')
BASELINE_FILE="$BASELINE_DIR/baseline_$DATE.txt"

mkdir -p "$BASELINE_DIR"

echo "=== System Performance Baseline - $DATE ===" > "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== CPU Info ===" >> "$BASELINE_FILE"
cat /proc/cpuinfo | grep -E '(model name|cpu MHz|cache size)' >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Memory Usage ===" >> "$BASELINE_FILE"
free -h >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Disk Usage ===" >> "$BASELINE_FILE"
df -h >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Disk I/O Stats ===" >> "$BASELINE_FILE"
iostat -x 1 3 >> "$BASELINE_FILE" 2>/dev/null || echo "iostat not available" >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Network Stats ===" >> "$BASELINE_FILE"
ss -tuln >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Load Average ===" >> "$BASELINE_FILE"
uptime >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Top Processes ===" >> "$BASELINE_FILE"
ps aux --sort=-%mem | head -20 >> "$BASELINE_FILE"
echo "" >> "$BASELINE_FILE"

echo "=== Service Status ===" >> "$BASELINE_FILE"
rc-status default >> "$BASELINE_FILE" 2>/dev/null || systemctl list-units --type=service --state=running >> "$BASELINE_FILE"

# Keep only last 30 baselines
find "$BASELINE_DIR" -name "baseline_*.txt" -mtime +30 -delete
```

Create monitoring script:

```bash
nano /usr/local/bin/performance-check.sh
```

```bash
#!/bin/bash

LOG_FILE="/var/log/performance-alerts.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Thresholds
MEMORY_THRESHOLD=80    # %
DISK_THRESHOLD=85      # %
LOAD_THRESHOLD=2.0     # load average

# Check memory usage
MEMORY_USAGE=$(free | awk 'NR==2{printf "%.1f", $3*100/$2}')
if (( $(echo "$MEMORY_USAGE > $MEMORY_THRESHOLD" | bc -l) )); then
    echo "[$DATE] WARNING: Memory usage at ${MEMORY_USAGE}%" >> "$LOG_FILE"
fi

# Check disk usage
DISK_USAGE=$(df / | awk 'NR==2{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt "$DISK_THRESHOLD" ]; then
    echo "[$DATE] WARNING: Disk usage at ${DISK_USAGE}%" >> "$LOG_FILE"
fi

# Check load average
LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
if (( $(echo "$LOAD_AVG > $LOAD_THRESHOLD" | bc -l) )); then
    echo "[$DATE] WARNING: Load average at $LOAD_AVG" >> "$LOG_FILE"
fi
```

Make executable and schedule:

```bash
chmod +x /usr/local/bin/collect-baseline.sh
chmod +x /usr/local/bin/performance-check.sh

# Collect baseline daily at 1 AM
echo "0 1 * * * /usr/local/bin/collect-baseline.sh" >> /var/spool/cron/crontabs/root

# Performance check every 15 minutes
echo "*/15 * * * * /usr/local/bin/performance-check.sh" >> /var/spool/cron/crontabs/root
```

## Step 5: Enhanced Logging Configuration

Configure centralized logging for security events.

Create security log configuration:

```bash
nano /etc/rsyslog.d/security.conf
```

```bash
# Security-related logging
auth,authpriv.*                 /var/log/auth.log
kern.*                          /var/log/kern.log
mail.*                          /var/log/mail.log

# AppArmor logs
:msg, contains, "apparmor"      /var/log/apparmor.log

# AIDE logs
:programname, isequal, "aide"   /var/log/aide.log

# Stop processing after logging security events
& stop
```

Restart rsyslog:

```bash
/etc/init.d/rsyslog restart
```

---

These advanced hardening measures significantly improve your VPS security posture while maintaining the lean, practical approach of this guide.

**Resource Impact:** These features add 50-100MB RAM usage and periodic CPU/disk activity. Recommended for VPS with 2GB+ RAM or when enhanced security is required.

**For 1GB VPS:** Consider implementing only AIDE (file integrity) and automated security updates, skip AppArmor and intensive monitoring.
