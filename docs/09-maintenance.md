<i>Keep your Gentoo VPS running clean, secure, and up-to-date with safe system update routines, backups, and runtime checks.</i>

# 09 – Maintenance, Backups & Updates

> Final steps to ensure your Gentoo VPS stays stable, secure, and easy to recover.

---

## Step 1: World Updates

Update system packages regularly:

```bash
emerge --sync
emerge -upvDN @world
```

Read carefully what is to be updated and act on this information accordingly.

If it's safe to update:

```bash
emerge -uavDU @world
```

Then rebuild anything broken or outdated:

```bash
emerge @preserved-rebuild
emerge --depclean
revdep-rebuild
```

Rebuild reverse dependencies (if needed):

```bash
perl-cleaner --all
python-updater
```

If you're not using a custom kernel (as probably not on the VPS):

```bash
emerge -av gentoo-kernel-bin
```

## Step 2: Backups
Use rsync, borg, or restic for lightweight VPS backups.

Rsync example:

```bash
rsync -aAXv --exclude={"/proc","/sys","/dev","/tmp","/run","/mnt","/media","/lost+found"} / root@backupserver:/backups/your-vps/
```

Borg example:

```bash
borg init --encryption=repokey /mnt/backup/repo
borg create /mnt/backup/repo::vps-$(date +%F) /etc /var/www /home
```

Store:
- /etc/
- /var/www/
- /root/
- /home/
- Custom app data (e.g., /opt/nodeapp/)

! Avoid copying /var/log or /tmp.

## WordPress Automated Backup

Create a WordPress backup script:

```bash
nano /usr/local/bin/wordpress-backup.sh
```

```bash
#!/bin/bash
# ENTERPRISE WordPress Backup System (Better than UpdraftPlus)
# Features: Incremental backups, integrity checks, multiple destinations, hot backups

BACKUP_DIR="/var/backups/wordpress"
REMOTE_BACKUP_DIR="/mnt/remote-backup/wordpress"  # NFS/CIFS mount or rsync destination
WP_DIR="/var/www/wordpress"
DB_NAME="wordpress"
DB_USER="wpuser"
DB_PASS="your_db_password"
DATE=$(date +%Y%m%d_%H%M%S)
DAILY_RETENTION=7
WEEKLY_RETENTION=4
MONTHLY_RETENTION=12
ALERT_EMAIL="admin@yourdomain.com"
LOG_FILE="/var/log/wordpress-backup.log"

# Create backup directories
mkdir -p "$BACKUP_DIR"/{daily,weekly,monthly,incremental}
mkdir -p "$REMOTE_BACKUP_DIR" 2>/dev/null || true

# Logging function
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

# Error handling
backup_error() {
    log_message "ERROR: $1"
    send_alert "BACKUP FAILED" "$1"
    exit 1
}

# Send email alerts
send_alert() {
    local subject="$1"
    local message="$2"
    if command -v msmtp &> /dev/null; then
        {
            echo "Subject: WordPress Backup - $subject on $(hostname)"
            echo "To: $ALERT_EMAIL"
            echo ""
            echo "WordPress Backup Status: $subject"
            echo "Server: $(hostname)"
            echo "Time: $(date)"
            echo "Message: $message"
            echo ""
            echo "Backup location: $BACKUP_DIR"
            echo "Log file: $LOG_FILE"
        } | msmtp "$ALERT_EMAIL" 2>/dev/null || log_message "Failed to send email alert"
    fi
}

# Database integrity check before backup
check_database() {
    log_message "Checking database integrity..."
    mysqlcheck -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" --check --auto-repair > /tmp/db_check.log 2>&1
    if [ $? -ne 0 ]; then
        backup_error "Database integrity check failed. See /tmp/db_check.log"
    fi
    log_message "Database integrity check passed"
}

# Hot database backup (minimal locking)
backup_database() {
    local backup_type="$1"
    local backup_path="$BACKUP_DIR/$backup_type"
    
    log_message "Starting hot database backup ($backup_type)..."
    
    # Use mysqldump with minimal locking for hot backup
    mysqldump -u "$DB_USER" -p"$DB_PASS" \
        --single-transaction \
        --routines \
        --triggers \
        --events \
        --hex-blob \
        --opt \
        "$DB_NAME" | gzip > "$backup_path/wp_db_$DATE.sql.gz"
    
    if [ $? -ne 0 ]; then
        backup_error "Database backup failed"
    fi
    
    # Verify backup integrity
    gunzip -t "$backup_path/wp_db_$DATE.sql.gz" || backup_error "Database backup file is corrupted"
    
    log_message "Database backup completed and verified ($backup_type)"
}

# Incremental file backup using rsync
backup_files_incremental() {
    local backup_path="$BACKUP_DIR/incremental"
    local link_dest=""
    
    # Find the most recent incremental backup to link against
    latest_backup=$(find "$backup_path" -maxdepth 1 -type d -name "wp_files_*" | sort | tail -1)
    if [ -n "$latest_backup" ]; then
        link_dest="--link-dest=$latest_backup"
    fi
    
    log_message "Starting incremental file backup..."
    
    rsync -av $link_dest \
        --exclude="wp-content/cache/" \
        --exclude="wp-content/debug.log" \
        --exclude="wp-content/backups/" \
        --exclude=".htaccess.bak" \
        "$WP_DIR/" "$backup_path/wp_files_$DATE/"
    
    if [ $? -ne 0 ]; then
        backup_error "Incremental file backup failed"
    fi
    
    log_message "Incremental file backup completed"
}

# Full file backup with compression
backup_files_full() {
    local backup_type="$1"
    local backup_path="$BACKUP_DIR/$backup_type"
    
    log_message "Starting full file backup ($backup_type)..."
    
    # Core WordPress files (excluding uploads for speed)
    tar czf "$backup_path/wp_core_$DATE.tar.gz" \
        --exclude="$WP_DIR/wp-content/cache" \
        --exclude="$WP_DIR/wp-content/uploads" \
        --exclude="$WP_DIR/wp-content/debug.log" \
        --exclude="$WP_DIR/wp-content/backups" \
        "$WP_DIR"
    
    if [ $? -ne 0 ]; then
        backup_error "Core files backup failed"
    fi
    
    # Uploads directory (separate for efficiency)
    if [ -d "$WP_DIR/wp-content/uploads" ]; then
        tar czf "$backup_path/wp_uploads_$DATE.tar.gz" "$WP_DIR/wp-content/uploads"
        if [ $? -ne 0 ]; then
            backup_error "Uploads backup failed"
        fi
    fi
    
    log_message "Full file backup completed ($backup_type)"
}

# Backup verification
verify_backup() {
    local backup_type="$1"
    local backup_path="$BACKUP_DIR/$backup_type"
    
    log_message "Verifying backup integrity ($backup_type)..."
    
    # Verify database backup
    if [ -f "$backup_path/wp_db_$DATE.sql.gz" ]; then
        gunzip -t "$backup_path/wp_db_$DATE.sql.gz" || backup_error "Database backup verification failed"
    fi
    
    # Verify core files backup
    if [ -f "$backup_path/wp_core_$DATE.tar.gz" ]; then
        tar tzf "$backup_path/wp_core_$DATE.tar.gz" > /dev/null || backup_error "Core files backup verification failed"
    fi
    
    # Verify uploads backup
    if [ -f "$backup_path/wp_uploads_$DATE.tar.gz" ]; then
        tar tzf "$backup_path/wp_uploads_$DATE.tar.gz" > /dev/null || backup_error "Uploads backup verification failed"
    fi
    
    log_message "Backup verification completed ($backup_type)"
}

# Remote sync (if configured)
sync_to_remote() {
    if [ -d "$REMOTE_BACKUP_DIR" ] || [[ "$REMOTE_BACKUP_DIR" == rsync://* ]]; then
        log_message "Syncing to remote location..."
        rsync -av --delete "$BACKUP_DIR/" "$REMOTE_BACKUP_DIR/"
        if [ $? -eq 0 ]; then
            log_message "Remote sync completed"
        else
            log_message "WARNING: Remote sync failed"
        fi
    fi
}

# Cleanup old backups with intelligent retention
cleanup_backups() {
    log_message "Cleaning up old backups..."
    
    # Clean daily backups
    find "$BACKUP_DIR/daily" -name "wp_*" -mtime +$DAILY_RETENTION -delete
    
    # Clean weekly backups
    find "$BACKUP_DIR/weekly" -name "wp_*" -mtime +$((WEEKLY_RETENTION * 7)) -delete
    
    # Clean monthly backups
    find "$BACKUP_DIR/monthly" -name "wp_*" -mtime +$((MONTHLY_RETENTION * 30)) -delete
    
    # Clean incremental backups (keep last 14 days)
    find "$BACKUP_DIR/incremental" -maxdepth 1 -name "wp_files_*" -mtime +14 -exec rm -rf {} \;
    
    log_message "Backup cleanup completed"
}

# Main backup logic
main() {
    log_message "=== WordPress Enterprise Backup Started ==="
    
    # Determine backup type based on day
    day_of_week=$(date +%u)  # 1=Monday, 7=Sunday
    day_of_month=$(date +%d)
    
    if [ "$day_of_month" = "01" ]; then
        backup_type="monthly"
        log_message "Performing monthly backup"
    elif [ "$day_of_week" = "7" ]; then
        backup_type="weekly"
        log_message "Performing weekly backup"
    else
        backup_type="daily"
        log_message "Performing daily backup"
    fi
    
    # Pre-backup checks
    check_database
    
    # Perform backups
    backup_database "$backup_type"
    
    if [ "$backup_type" = "daily" ]; then
        backup_files_incremental
    else
        backup_files_full "$backup_type"
    fi
    
    # Verify backup integrity
    verify_backup "$backup_type"
    
    # Sync to remote location
    sync_to_remote
    
    # Cleanup old backups
    cleanup_backups
    
    # Calculate backup size
    backup_size=$(du -sh "$BACKUP_DIR" | cut -f1)
    log_message "Total backup size: $backup_size"
    
    log_message "=== WordPress Enterprise Backup Completed Successfully ==="
    send_alert "SUCCESS" "Backup completed successfully. Total size: $backup_size"
}

# Run main function
main
fi
```

Make it executable and schedule:

```bash
chmod +x /usr/local/bin/wordpress-backup.sh

# Add to crontab (daily backup at 2 AM)
echo "0 2 * * * /usr/local/bin/wordpress-backup.sh" >> /var/spool/cron/crontabs/root
```

Test the backup script:

```bash
/usr/local/bin/wordpress-backup.sh
ls -la /var/backups/wordpress/
```

## Step 3: Rebooting Safely

Before rebooting:

Ensure no emerge or package tasks are running

Verify `nginx`, `php-fpm`, `mysql`, and other services are enabled in boot:

OpenRC: `rc-status default`

systemd: `systemctl list-unit-files | grep enabled`

Double-check `fstab`, `network`, and `/boot` integrity

Then:

```bash
reboot
```

## Step 4: Automated VPS Optimization Scripts

Create performance monitoring script `/usr/local/bin/vps-monitor.sh`:

```bash
#!/bin/bash
# VPS Performance Monitor for 1GB RAM systems

LOG="/var/log/vps-monitor.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Memory usage check
MEM_USED=$(free | grep Mem | awk '{printf("%.1f", ($3/$2) * 100.0)}')
if (( $(echo "$MEM_USED > 85" | bc -l) )); then
    echo "$DATE - WARNING: Memory usage at ${MEM_USED}%" >> $LOG
    
    # Restart PHP-FPM if memory usage too high
    if (( $(echo "$MEM_USED > 90" | bc -l) )); then
        echo "$DATE - CRITICAL: Restarting PHP-FPM due to high memory usage" >> $LOG
        /etc/init.d/php-fpm restart
    fi
fi

# Disk usage check
DISK_USED=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ $DISK_USED -gt 85 ]; then
    echo "$DATE - WARNING: Disk usage at ${DISK_USED}%" >> $LOG
    
    # Clean old logs
    find /var/log -name "*.log" -mtime +7 -delete
    find /var/cache/nginx -mtime +1 -delete 2>/dev/null
fi

# Check service status
for service in nginx php-fpm mariadb; do
    if ! rc-service $service status > /dev/null 2>&1; then
        echo "$DATE - ERROR: $service is down, attempting restart" >> $LOG
        rc-service $service start
    fi
done
```

Make it executable and add to cron:

```bash
chmod +x /usr/local/bin/vps-monitor.sh
echo "*/10 * * * * /usr/local/bin/vps-monitor.sh" >> /etc/crontab
```

Create log cleanup script `/usr/local/bin/cleanup-vps.sh`:

```bash
#!/bin/bash
# Weekly VPS cleanup for low-end hardware

# Clear old logs
find /var/log -name "*.log" -mtime +14 -delete
journalctl --vacuum-time=7d

# Clear PHP sessions
find /tmp -name "sess_*" -mtime +1 -delete

# Clear nginx cache if too large
CACHE_SIZE=$(du -sm /var/cache/nginx 2>/dev/null | cut -f1)
if [ "$CACHE_SIZE" -gt 50 ]; then
    rm -rf /var/cache/nginx/*
    /etc/init.d/nginx reload
fi

# Optimize MySQL tables weekly
mysql -u root -p"$MYSQL_ROOT_PASSWORD" -e "OPTIMIZE TABLE information_schema.tables;" 2>/dev/null

echo "$(date): VPS cleanup completed" >> /var/log/vps-cleanup.log
```

Schedule weekly cleanup:

```bash
chmod +x /usr/local/bin/cleanup-vps.sh
echo "0 2 * * 0 /usr/local/bin/cleanup-vps.sh" >> /etc/crontab
```

## Step 5: Monitoring & Security Check Routine

If you've implemented [Advanced Hardening](10-advanced-hardening.md), integrate these checks into your maintenance routine:

### Weekly Security Checks:

```bash
# Check AIDE integrity
/usr/local/bin/aide-check.sh

# Review AppArmor denials
dmesg | grep -i apparmor | tail -20

# Check security update log
tail -50 /var/log/security-updates.log

# Review failed login attempts
grep "Failed password" /var/log/auth.log | tail -20
```

### Monthly Performance Review:

```bash
# Review performance baselines
ls -la /var/log/baselines/ | tail -10

# Check performance alerts
tail -50 /var/log/performance-alerts.log

# Review system resource trends
free -h && df -h && uptime
```

### Security Maintenance Commands:

```bash
# Update AIDE database after legitimate system changes
aide --update
mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Check AppArmor profile status
aa-status

# Review fail2ban status
fail2ban-client status
fail2ban-client status sshd
```

---

**Recommended Maintenance Schedule:**

- **Daily:** Automated security updates (via cron)
- **Weekly:** Manual system updates, AIDE checks, cleanup script
- **Monthly:** Performance baseline review, log analysis
- **Quarterly:** Full Lynis audit, AppArmor profile review, backup restore tests


