<i>Ultra-lightweight optimizations and service replacements for maximum performance on resource-constrained VPS environments. Every megabyte and CPU cycle counts.</i>

# 11 – Ultra-Lightweight Optimization

> **For performance enthusiasts:** Squeeze every bit of performance from your teeny-tiny VPS by replacing standard services with lightweight alternatives and applying micro-optimizations.

---

## Step 1: Lightweight Service Replacements

Replace resource-heavy services with their minimal counterparts.

### Replace rsyslog with syslog-ng

syslog-ng is more efficient and uses less memory:

```bash
emerge -av app-admin/syslog-ng
rc-update del rsyslog default
rc-update add syslog-ng default

# Minimal syslog-ng configuration
nano /etc/syslog-ng/syslog-ng.conf
```

Minimal syslog-ng config:

```bash
@version: 3.38

options {
    threaded(yes);
    chain_hostnames(no);
    stats_freq(0);
    mark_freq(0);
    keep_hostname(yes);
    use_dns(no);
    use_fqdn(no);
    create_dirs(yes);
    dir_perm(0755);
    perm(0644);
};

source s_local {
    unix-dgram("/dev/log");
    internal();
};

destination d_messages { file("/var/log/messages"); };
destination d_auth { file("/var/log/auth.log"); };
destination d_kern { file("/var/log/kern.log"); };

filter f_auth { facility(auth, authpriv); };
filter f_kern { facility(kern); };
filter f_messages { not facility(auth, authpriv, kern); };

log { source(s_local); filter(f_auth); destination(d_auth); };
log { source(s_local); filter(f_kern); destination(d_kern); };
log { source(s_local); filter(f_messages); destination(d_messages); };
```

### Replace cronie with dcron

dcron is significantly lighter:

```bash
emerge -av sys-process/dcron
rc-update del cronie default  
rc-update add dcron default
```

### Lightweight NTP with chrony

chrony uses ~2MB vs ~8MB for ntp:

```bash
emerge -av net-misc/chrony
nano /etc/chrony/chrony.conf
```

Minimal chrony config:

```bash
pool pool.ntp.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
```

Enable:

```bash
rc-update add chronyd default
```

### Alternative: Replace nginx with lighttpd

If you want even lighter web server (saves ~5-10MB):

```bash
emerge -av www-servers/lighttpd
```

Lighttpd config optimized for 1GB RAM:

```bash
nano /etc/lighttpd/lighttpd.conf
```

```bash
server.modules = (
    "mod_rewrite",
    "mod_fastcgi",
    "mod_compress",
    "mod_accesslog"
)

server.document-root = "/var/www/wordpress"
server.port = 80
server.username = "lighttpd"
server.groupname = "lighttpd"

# Minimal worker processes
server.max-worker = 2
server.max-fds = 1024
server.max-connections = 512

# Memory optimizations
server.max-request-size = 33554432     # 32MB to match PHP settings
server.upload-dirs = ( "/tmp" )

# FastCGI for PHP
fastcgi.server = ( ".php" =>
    (( "socket" => "/run/php-fpm/php-fpm.sock",
       "broken-scriptfilename" => "enable"
    ))
)

# Compression
compress.cache-dir = "/var/cache/lighttpd/compress/"
compress.filetype = ("application/javascript", "text/css", "text/html", "text/plain")

# Logging
accesslog.filename = "/var/log/lighttpd/access.log"
server.errorlog = "/var/log/lighttpd/error.log"
```

## Step 2: Aggressive Kernel Optimizations

Push kernel parameters to the limit for small systems:

```bash
nano /etc/sysctl.conf
```

Add ultra-aggressive settings:

```bash
# Memory management for small systems
vm.swappiness=5                    # Avoid swap at all costs
vm.dirty_ratio=10                  # Small dirty cache
vm.dirty_background_ratio=3        # Early writeback
vm.dirty_expire_centisecs=1000     # Expire dirty pages quickly
vm.dirty_writeback_centisecs=500   # Frequent writeback
vm.overcommit_memory=1             # Allow memory overcommit
vm.overcommit_ratio=80             # 80% overcommit limit
vm.oom_kill_allocating_task=1      # Kill the OOM-causing task
vm.min_free_kbytes=8192            # Minimum free memory (8MB)
vm.zone_reclaim_mode=1             # Aggressive memory reclaim

# Network optimizations for single-core VPS
net.core.rmem_default=65536
net.core.rmem_max=1048576
net.core.wmem_default=65536  
net.core.wmem_max=1048576
net.core.netdev_max_backlog=1000
net.core.somaxconn=512
net.ipv4.tcp_rmem=4096 87380 1048576
net.ipv4.tcp_wmem=4096 87380 1048576
net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_congestion_control=bbr
net.ipv4.tcp_fastopen=3
net.ipv4.tcp_low_latency=1
net.ipv4.tcp_no_metrics_save=1

# File system optimizations
fs.file-max=32768                  # Reduced file limit for 1GB
fs.inotify.max_user_watches=8192   # Reduce inotify overhead

# Process scheduling
kernel.sched_min_granularity_ns=2000000     # 2ms (responsive)
kernel.sched_wakeup_granularity_ns=3000000  # 3ms
kernel.sched_migration_cost_ns=250000       # 0.25ms
```

Apply immediately:

```bash
sysctl -p
```

## Step 3: Ultra-Aggressive Mount Optimizations (that is probably not possible on most of the vps providers)

Optimize filesystem mounts for maximum performance:

```bash
nano /etc/fstab
```

Replace your root mount with ultra-optimized options:

```bash
# Ultra-optimized ext4 mount for SSD VPS
/dev/vda1 / ext4 defaults,noatime,nodiratime,nobarrier,commit=60,data=writeback,journal_async_commit 0 1

# Aggressive tmpfs for frequently written directories
tmpfs /tmp tmpfs defaults,noatime,mode=1777,size=128M 0 0
tmpfs /var/tmp tmpfs defaults,noatime,mode=1777,size=64M 0 0
tmpfs /var/log tmpfs defaults,noatime,mode=0755,size=32M 0 0
tmpfs /run tmpfs defaults,noatime,mode=0755,size=16M 0 0

# Swap with priority
/swapfile none swap sw,pri=1 0 0
```

**Warning:** These aggressive options prioritize performance over safety. Only use on VPS where you have really good backups.

## Step 4: Shell and Binary Optimizations (that is probably not possible on most of the vps providers)

Replace bash with dash for faster script execution:

```bash
emerge -av app-shells/dash

# Make dash the default shell for scripts
rm /bin/sh
ln -s /bin/dash /bin/sh

# Verify
ls -la /bin/sh
```

Use musl-based binaries where possible:

```bash
# Add musl overlay for smaller binaries
emerge -av app-eselect/eselect-repository
eselect repository enable musl
emerge --sync musl
```

## Step 5: Service Startup Optimization

**For OpenRC systems** - optimize service startup order and disable unnecessary services:

```bash
# Remove unnecessary boot services
rc-update del hwclock boot          # VPS handles time
rc-update del consolefont boot      # No console needed
rc-update del keymaps boot          # No keyboard on VPS  
rc-update del modules boot          # If no custom modules needed
rc-update del netmount default      # If no network mounts

# Parallelize service startup
nano /etc/rc.conf
```

Optimize rc.conf:

```bash
# Parallel startup
rc_parallel="YES"

# Faster startup
rc_hotplug="YES"

# Minimal logging during boot
rc_logger="NO"

# Fast dependency resolution  
rc_depend_strict="NO"
```

**For systemd systems** - optimize boot performance:

```bash
# Disable slow boot services
systemctl disable NetworkManager-wait-online.service
systemctl disable systemd-networkd-wait-online.service
systemctl disable plymouth-start.service
systemctl disable plymouth-read-write.service
systemctl disable plymouth-quit.service
systemctl disable plymouth-quit-wait.service

# Analyze boot performance
systemd-analyze blame | head -20
systemd-analyze critical-chain

# Set aggressive service timeouts
mkdir -p /etc/systemd/system.conf.d
nano /etc/systemd/system.conf.d/timeouts.conf
```

```bash
[Manager]
# Aggressive boot timeouts for VPS
DefaultTimeoutStartSec=15s
DefaultTimeoutStopSec=10s
DefaultTimeoutAbortSec=10s
DefaultDeviceTimeoutSec=10s
```

Create custom target for ultra-fast boot:

```bash
# Create minimal VPS target
nano /etc/systemd/system/vps-minimal.target
```

```bash
[Unit]
Description=Minimal VPS Target
Requires=multi-user.target
Conflicts=rescue.target
After=multi-user.target
AllowIsolate=yes

[Install]
Alias=default.target
```

```bash
# Set as default target
systemctl set-default vps-minimal.target
systemctl daemon-reload
```

## Step 6: Database Micro-Optimizations

Ultra-lightweight MariaDB configuration for 1GB RAM:

```bash
nano /etc/mysql/my.cnf
```

```bash
[mysqld]
# Ultra-minimal memory settings
key_buffer_size=32M
max_allowed_packet=16M
table_open_cache=64
sort_buffer_size=1M
read_buffer_size=1M
read_rnd_buffer_size=2M
myisam_sort_buffer_size=16M
thread_cache_size=4
query_cache_size=16M
query_cache_limit=2M

# InnoDB ultra-minimal
innodb_buffer_pool_size=128M        # Minimal for 1GB system
innodb_log_file_size=8M
innodb_log_buffer_size=4M
innodb_flush_log_at_trx_commit=2    # Faster, less safe
innodb_lock_wait_timeout=30
innodb_file_per_table=1

# Performance Schema - disable completely
performance_schema=OFF

# Disable binary logging if not needed
skip-log-bin

# Network optimizations
max_connections=20                  # Very low for 1GB RAM
connect_timeout=10
wait_timeout=300
interactive_timeout=300

# Skip DNS lookups
skip-name-resolve

# Reduce memory for temporary tables
tmp_table_size=16M
max_heap_table_size=16M
```

## Step 7: PHP-FPM Ultra-Optimization

Minimal PHP-FPM pool configuration:

```bash
nano /etc/php/fpm-php8.2/fpm.d/www.conf
```

```bash
[www]
user = nginx
group = nginx
listen = /run/php-fpm/php-fpm.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660

# Ultra-minimal process management for 1GB RAM
pm = dynamic
pm.max_children = 5                 # Very low for 1GB
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 2
pm.max_requests = 100               # Restart workers frequently

# Memory limits
php_admin_value[memory_limit] = 64M  # Low memory per process

# Reduce timeouts
request_terminate_timeout = 30s
```

Optimize php.ini for minimal memory usage:

```bash
nano /etc/php/fpm-php8.2/php.ini
```

Key optimizations:

```bash
memory_limit = 64M                  # Very low
max_execution_time = 30
max_input_time = 30
post_max_size = 32M                 # Reasonable for 1GB VPS
upload_max_filesize = 32M           # Reasonable for 1GB VPS
max_file_uploads = 10
max_input_vars = 1500               # Reduced but sufficient

# Disable unused extensions
; extension=calendar
; extension=exif
; extension=ftp
; extension=gettext
; extension=sockets
; extension=sysvmsg
; extension=sysvsem
; extension=sysvshm

# OPcache optimization for 1GB
opcache.enable=1
opcache.memory_consumption=32       # Very small
opcache.max_accelerated_files=2000  # Reduced
opcache.validate_timestamps=0       # Production setting
```

## Step 8: Network Stack Tuning

Fine-tune network for single-core, low-memory VPS:

```bash
# Network interface optimizations
echo 'net.core.netdev_budget=150' >> /etc/sysctl.conf
echo 'net.core.netdev_max_backlog=1000' >> /etc/sysctl.conf

# TCP optimizations for low-latency
echo 'net.ipv4.tcp_timestamps=0' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_sack=1' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_fack=1' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_window_scaling=1' >> /etc/sysctl.conf

# Reduce TCP timeouts for faster connection recycling
echo 'net.ipv4.tcp_fin_timeout=15' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_keepalive_time=300' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_keepalive_probes=3' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_keepalive_intvl=15' >> /etc/sysctl.conf

sysctl -p
```

## Step 9: Memory Profiling and Monitoring

Create a lightweight memory monitor:

```bash
nano /usr/local/bin/memory-squeeze.sh
```

```bash
#!/bin/bash

LOG="/var/log/memory-squeeze.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Get memory stats
MEM_TOTAL=$(free -m | awk 'NR==2{print $2}')
MEM_USED=$(free -m | awk 'NR==2{print $3}')
MEM_PERCENT=$(( MEM_USED * 100 / MEM_TOTAL ))

# If memory usage > 90%, take action
if [ $MEM_PERCENT -gt 90 ]; then
    echo "[$DATE] CRITICAL: Memory at ${MEM_PERCENT}%, taking action" >> $LOG
    
    # Drop caches
    echo 3 > /proc/sys/vm/drop_caches
    
    # Restart PHP-FPM to free memory
    /etc/init.d/php-fpm restart
    
    # Log top memory consumers
    echo "Top memory consumers:" >> $LOG
    ps aux --sort=-%mem | head -10 >> $LOG
    
elif [ $MEM_PERCENT -gt 80 ]; then
    echo "[$DATE] WARNING: Memory at ${MEM_PERCENT}%" >> $LOG
    
    # Just drop caches
    echo 1 > /proc/sys/vm/drop_caches
fi
```

Make executable and run every 5 minutes:

```bash
chmod +x /usr/local/bin/memory-squeeze.sh
echo "*/5 * * * * /usr/local/bin/memory-squeeze.sh" >> /var/spool/cron/crontabs/root
```

## Step 10: Startup Time Optimization

Create boot time analysis:

```bash
nano /usr/local/bin/boot-analyze.sh
```

```bash
#!/bin/bash

echo "=== Boot Time Analysis ==="
echo "Systemd boot time:"
systemd-analyze 2>/dev/null || echo "OpenRC system"

echo ""
echo "Service startup times:"
rc-status -s 2>/dev/null | sort -k2 -n || echo "Not available on OpenRC"

echo ""
echo "Current memory usage:"
free -h

echo ""
echo "Process count:"
ps aux | wc -l

echo ""
echo "Open file descriptors:"
lsof | wc -l
```

```bash
chmod +x /usr/local/bin/boot-analyze.sh
```

---

## Resource Savings Summary

These optimizations can save significant resources on a 1GB VPS:

- **RAM savings:** 50-150MB (5-15% of total RAM)
- **CPU reduction:** 10-20% lower baseline CPU usage  
- **Boot time:** 20-30% faster startup
- **I/O reduction:** 30-50% less disk activity

**Performance Impact:**
- **Recommended for:** 1GB RAM VPS with performance requirements
- **Trade-offs:** Reduced safety margins, more aggressive resource management
- **Maintenance:** Requires closer monitoring due to optimized settings

**Testing Required:** Always test these optimizations thoroughly as they push the system closer to resource limits.

---

## Step 11: Ultra-Micro Optimizations

Squeeze every last drop of performance with these advanced techniques:

### Disable Unnecessary Kernel Features

```bash
# Disable kernel features that consume memory/CPU
echo 'kernel.printk = 3 4 1 3' >> /etc/sysctl.conf              # Reduce kernel logging
echo 'kernel.dmesg_restrict = 1' >> /etc/sysctl.conf             # Restrict dmesg access
echo 'vm.panic_on_oom = 0' >> /etc/sysctl.conf                  # Don't panic on OOM
echo 'kernel.panic = 10' >> /etc/sysctl.conf                    # Reboot after panic
echo 'kernel.sysrq = 0' >> /etc/sysctl.conf                     # Disable SysRq key
```

### Ultra-Minimal systemd/OpenRC Configuration

**For OpenRC systems**, disable even more services:

```bash
# Ultra-minimal service list for 1GB VPS
rc-update del urandom boot          # Not needed on VPS
rc-update del swap boot             # Handle manually if needed
rc-update del fsck boot             # VPS provider handles this
rc-update del root boot             # VPS provider handles this
rc-update del localmount boot       # VPS provider handles this

# Only keep absolutely essential services
rc-status boot | grep started       # Review what's actually needed
```

**For systemd systems**, disable unnecessary services and optimize:

```bash
# Disable unnecessary systemd services for VPS
systemctl disable systemd-resolved          # Use simple resolv.conf
systemctl disable systemd-timesyncd        # Use chrony instead
systemctl disable systemd-networkd         # VPS handles networking
systemctl disable accounts-daemon          # Not needed on server
systemctl disable avahi-daemon             # No local network discovery
systemctl disable cups                     # No printing on VPS
systemctl disable bluetooth               # No bluetooth on VPS
systemctl disable ModemManager            # No modem on VPS
systemctl disable NetworkManager-wait-online
systemctl disable wpa_supplicant          # No WiFi on VPS

# Mask services that might get re-enabled
systemctl mask systemd-resolved
systemctl mask systemd-timesyncd
systemctl mask accounts-daemon
systemctl mask avahi-daemon

# Check what's left running
systemctl list-unit-files --state=enabled --type=service
```

**Systemd-specific optimizations:**

```bash
# Configure systemd for minimal resource usage
nano /etc/systemd/system.conf
```

```bash
# Ultra-minimal systemd configuration
[Manager]
# Reduce default timeouts
DefaultTimeoutStartSec=30s
DefaultTimeoutStopSec=15s
DefaultRestartSec=5s

# Reduce logging
LogLevel=warning
LogTarget=journal
MaxLevelStore=warning
MaxLevelSyslog=warning

# Resource limits
DefaultMemoryAccounting=yes
DefaultMemoryHigh=800M
DefaultMemoryMax=900M
DefaultTasksMax=512

# Reduce journal size
SystemMaxUse=50M
SystemKeepFree=100M
SystemMaxFileSize=10M
RuntimeMaxUse=50M
RuntimeKeepFree=100M
RuntimeMaxFileSize=10M

# Faster shutdown
DefaultTimeoutStopSec=10s
ShutdownWatchdogSec=30s
```

Configure journald for minimal resource usage:

```bash
nano /etc/systemd/journald.conf
```

```bash
[Journal]
# Minimize journal storage
Storage=persistent
Compress=yes
SystemMaxUse=50M
SystemKeepFree=100M
SystemMaxFileSize=10M
RuntimeMaxUse=50M
RuntimeKeepFree=100M
RuntimeMaxFileSize=10M
MaxRetentionSec=604800    # 7 days
MaxFileSec=86400          # 1 day per file

# Reduce logging levels
MaxLevelStore=warning
MaxLevelSyslog=warning
ForwardToSyslog=no
ForwardToKMsg=no
ForwardToConsole=no
ForwardToWall=no
```

Optimize systemd unit files for critical services:

```bash
# Optimize nginx systemd service
mkdir -p /etc/systemd/system/nginx.service.d
nano /etc/systemd/system/nginx.service.d/override.conf
```

```bash
[Service]
# Resource limits for nginx
MemoryHigh=100M
MemoryMax=150M
TasksMax=50
Nice=-5
OOMScoreAdjust=-500
PrivateDevices=yes
ProtectSystem=strict
ProtectHome=yes
NoNewPrivileges=yes
```

```bash
# Optimize PHP-FPM systemd service
mkdir -p /etc/systemd/system/php-fpm.service.d
nano /etc/systemd/system/php-fpm.service.d/override.conf
```

```bash
[Service]
# Resource limits for PHP-FPM
MemoryHigh=200M
MemoryMax=300M
TasksMax=20
Nice=-3
OOMScoreAdjust=-300
PrivateDevices=yes
ProtectSystem=strict
ProtectHome=yes
```

```bash
# Optimize MariaDB systemd service
mkdir -p /etc/systemd/system/mariadb.service.d
nano /etc/systemd/system/mariadb.service.d/override.conf
```

```bash
[Service]
# Resource limits for MariaDB
MemoryHigh=300M
MemoryMax=400M
TasksMax=100
Nice=0
OOMScoreAdjust=-200
```

Apply systemd optimizations:

```bash
# Reload systemd configuration
systemctl daemon-reload

# Restart systemd-journald with new settings
systemctl restart systemd-journald

# Clean old journal files
journalctl --vacuum-time=1d
journalctl --vacuum-size=50M
```

### Memory Allocation Tweaks

```bash
# Ultra-aggressive memory settings
echo 'vm.vfs_cache_pressure=200' >> /etc/sysctl.conf            # Aggressive cache reclaim
echo 'vm.laptop_mode=1' >> /etc/sysctl.conf                    # Laptop mode (reduces disk writes)
echo 'vm.block_dump=0' >> /etc/sysctl.conf                     # Disable block debugging
echo 'vm.oom_dump_tasks=0' >> /etc/sysctl.conf                 # Don't dump tasks on OOM
```

### Process and File Descriptor Limits

```bash
# Reduce limits for 1GB system
echo '* soft nofile 1024' >> /etc/security/limits.conf
echo '* hard nofile 2048' >> /etc/security/limits.conf
echo '* soft nproc 512' >> /etc/security/limits.conf
echo '* hard nproc 1024' >> /etc/security/limits.conf
```

### Kernel Module Blacklisting

```bash
# Blacklist unnecessary modules to save memory
nano /etc/modprobe.d/blacklist-unnecessary.conf
```

```bash
# Sound drivers (not needed on VPS)
blacklist snd
blacklist snd_hda_intel
blacklist snd_ac97_codec

# Graphics drivers (not needed on VPS)
blacklist nouveau
blacklist radeon
blacklist i915

# Wireless drivers (not needed on VPS)
blacklist iwlwifi
blacklist ath9k
blacklist rtl8192ce

# Bluetooth (not needed on VPS)
blacklist btusb
blacklist bluetooth

# Webcam drivers (not needed on VPS)
blacklist uvcvideo
```

### Ultra-Lean PHP Configuration

Even more aggressive PHP settings:

```bash
nano /etc/php/fpm-php8.2/php.ini
```

Additional ultra-minimal settings:

```bash
# Disable more features
expose_php = Off
log_errors = Off                    # Disable error logging to save I/O
display_errors = Off
display_startup_errors = Off
track_errors = Off
html_errors = Off

# Minimal session settings
session.gc_probability = 0          # Disable session garbage collection
session.cache_limiter = ""          # No cache headers
session.use_cookies = 0             # If sessions not needed

# Disable more extensions
; extension=bcmath
; extension=bz2
; extension=enchant
; extension=gmp
; extension=iconv
; extension=intl
; extension=ldap
; extension=odbc
; extension=pgsql
; extension=pspell
; extension=readline
; extension=recode
; extension=snmp
; extension=soap
; extension=tidy
; extension=wddx
; extension=xmlrpc
; extension=xsl
```

### Ultra-Minimal WordPress Optimizations

```bash
# WordPress-specific optimizations for 1GB VPS
nano /var/www/wordpress/wp-config.php
```

Add these ultra-performance settings:

```php
// Ultra-minimal WordPress configuration
define('WP_MEMORY_LIMIT', '32M');           // Very low memory limit
define('DISABLE_WP_CRON', true);            // Disable WP cron (use system cron)
define('AUTOMATIC_UPDATER_DISABLED', true); // Disable auto-updates
define('WP_POST_REVISIONS', 2);             // Minimal revisions
define('AUTOSAVE_INTERVAL', 300);           // 5-minute autosave
define('EMPTY_TRASH_DAYS', 7);              // Empty trash weekly
define('WP_DEBUG', false);                  // No debugging
define('WP_DEBUG_LOG', false);              // No debug logging

// Disable pingbacks/trackbacks
define('DISALLOW_FILE_EDIT', true);         // No file editing in admin

// Database optimization
define('WP_ALLOW_REPAIR', false);           // Disable DB repair
```

## Step 11.5: WordPress optimisations
Since WordPress is often the biggest memory hog, let's optimize it aggressively:

### Core WordPress File Optimizations

Remove unnecessary WordPress core files and features:

```bash
cd /var/www/wordpress

# Remove unnecessary language files (keep only English)
find wp-includes/languages/ -name "*.po" -delete
find wp-includes/languages/ -name "*.mo" ! -name "en_*" -delete
find wp-content/languages/ -name "*.po" -delete 2>/dev/null
find wp-content/languages/ -name "*.mo" ! -name "en_*" -delete 2>/dev/null

# Remove default themes (keep only one minimal theme)
rm -rf wp-content/themes/twentytwentyone
rm -rf wp-content/themes/twentytwentytwo
rm -rf wp-content/themes/twentytwentythree
# Keep only twentytwentyfour or install a minimal theme

# Remove default plugins
rm -rf wp-content/plugins/akismet
rm -rf wp-content/plugins/hello.php

# Remove unnecessary core files
rm -f wp-config-sample.php
rm -f readme.html
rm -f license.txt
```

### Ultra-Minimal WordPress Theme

Create or use an ultra-minimal theme:

```bash
mkdir -p /var/www/wordpress/wp-content/themes/minimal-vps
cd /var/www/wordpress/wp-content/themes/minimal-vps
```

Create minimal theme files:

```bash
# Ultra-minimal index.php
cat > index.php << 'EOF'
<?php
get_header();
if (have_posts()) {
    while (have_posts()) {
        the_post();
        echo '<article><h1>' . get_the_title() . '</h1>';
        the_content();
        echo '</article>';
    }
}
get_footer();
?>
EOF
```

```bash
# Minimal functions.php
cat > functions.php << 'EOF'
<?php
// Remove WordPress bloat
remove_action('wp_head', 'wp_generator');
remove_action('wp_head', 'wp_shortlink_wp_head');
remove_action('wp_head', 'rsd_link');
remove_action('wp_head', 'wlwmanifest_link');
remove_action('wp_head', 'wp_resource_hints', 2);
remove_action('wp_head', 'feed_links_extra', 3);
remove_action('wp_head', 'feed_links', 2);
remove_action('wp_head', 'rest_output_link_wp_head');
remove_action('wp_head', 'wp_oembed_add_discovery_links');
remove_action('wp_head', 'rel_canonical');
remove_action('wp_head', 'print_emoji_detection_script', 7);
remove_action('wp_print_styles', 'print_emoji_styles');

// Disable jQuery migration
function remove_jquery_migrate($scripts) {
    if (!is_admin() && isset($scripts->registered['jquery'])) {
        $script = $scripts->registered['jquery'];
        if ($script->deps) {
            $script->deps = array_diff($script->deps, array('jquery-migrate'));
        }
    }
}
add_action('wp_default_scripts', 'remove_jquery_migrate');

// Disable WordPress embeds
function disable_embeds_init() {
    remove_action('rest_api_init', 'wp_oembed_register_route');
    add_filter('embed_oembed_discover', '__return_false');
    remove_filter('oembed_dataparse', 'wp_filter_oembed_result', 10);
    remove_action('wp_head', 'wp_oembed_add_discovery_links');
    remove_action('wp_head', 'wp_oembed_add_host_js');
}
add_action('init', 'disable_embeds_init', 9999);

// Disable XML-RPC
add_filter('xmlrpc_enabled', '__return_false');

// Remove query strings from static resources
function remove_script_version($src) {
    return remove_query_arg('ver', $src);
}
add_filter('script_loader_src', 'remove_script_version', 15, 1);
add_filter('style_loader_src', 'remove_script_version', 15, 1);

// Disable self pingbacks
function no_self_ping(&$links) {
    $home = get_option('home');
    foreach ($links as $l => $link)
        if (0 === strpos($link, $home))
            unset($links[$l]);
}
add_action('pre_ping', 'no_self_ping');

// Limit post revisions
if (!defined('WP_POST_REVISIONS')) define('WP_POST_REVISIONS', 2);

// Disable theme and plugin editor
if (!defined('DISALLOW_FILE_EDIT')) define('DISALLOW_FILE_EDIT', true);
?>
EOF
```

```bash
# Minimal style.css
cat > style.css << 'EOF'
/*
Theme Name: Minimal VPS
Description: Ultra-minimal theme for 1GB VPS
Version: 1.0
*/
body{font-family:Arial,sans-serif;line-height:1.6;margin:20px;max-width:800px}
h1,h2,h3{color:#333}
a{color:#0066cc}
article{margin-bottom:20px;padding-bottom:20px;border-bottom:1px solid #eee}
EOF
```

```bash
# Basic header.php
cat > header.php << 'EOF'
<!DOCTYPE html>
<html <?php language_attributes(); ?>>
<head>
<meta charset="<?php bloginfo('charset'); ?>">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title><?php wp_title('|', true, 'right'); bloginfo('name'); ?></title>
<?php wp_head(); ?>
</head>
<body <?php body_class(); ?>>
<header><h1><a href="<?php echo home_url(); ?>"><?php bloginfo('name'); ?></a></h1></header>
<main>
EOF
```

```bash
# Basic footer.php
cat > footer.php << 'EOF'
</main>
<footer><p>&copy; <?php echo date('Y'); ?> <?php bloginfo('name'); ?></p></footer>
<?php wp_footer(); ?>
</body>
</html>
EOF
```

### WordPress Database Optimizations

Clean up WordPress database:

```bash
# Create database cleanup script
nano /usr/local/bin/wp-db-cleanup.sh
```

```bash
#!/bin/bash
# WordPress database cleanup for 1GB VPS

DB_NAME="wordpress"
DB_USER="wordpress"
DB_PASS="your_password"

mysql -u $DB_USER -p$DB_PASS $DB_NAME << 'EOF'
-- Remove spam comments
DELETE FROM wp_comments WHERE comment_approved = 'spam';

-- Remove trashed comments  
DELETE FROM wp_comments WHERE comment_approved = 'trash';

-- Remove orphaned comment meta
DELETE FROM wp_commentmeta WHERE comment_id NOT IN (SELECT comment_ID FROM wp_comments);

-- Remove post revisions (keep only 2)
DELETE p1 FROM wp_posts p1 
INNER JOIN wp_posts p2 
WHERE p1.post_parent = p2.post_parent 
AND p1.post_type = 'revision' 
AND p2.post_type = 'revision' 
AND p1.ID > p2.ID 
AND p1.post_parent IN (
    SELECT post_parent FROM (
        SELECT post_parent, COUNT(*) as revision_count 
        FROM wp_posts 
        WHERE post_type = 'revision' 
        GROUP BY post_parent 
        HAVING revision_count > 2
    ) as t
);

-- Remove auto-drafts older than 7 days
DELETE FROM wp_posts WHERE post_status = 'auto-draft' AND post_date < DATE_SUB(NOW(), INTERVAL 7 DAY);

-- Remove orphaned post meta
DELETE FROM wp_postmeta WHERE post_id NOT IN (SELECT ID FROM wp_posts);

-- Remove transients
DELETE FROM wp_options WHERE option_name LIKE '_transient_%';
DELETE FROM wp_options WHERE option_name LIKE '_site_transient_%';

-- Optimize tables
OPTIMIZE TABLE wp_posts, wp_postmeta, wp_comments, wp_commentmeta, wp_options;
EOF

echo "WordPress database cleaned and optimized"
```

```bash
chmod +x /usr/local/bin/wp-db-cleanup.sh

# Run weekly
echo "0 2 * * 0 /usr/local/bin/wp-db-cleanup.sh" >> /var/spool/cron/crontabs/root
```

### WordPress Plugin Restrictions

Create a must-use plugin to enforce performance:

```bash
mkdir -p /var/www/wordpress/wp-content/mu-plugins
nano /var/www/wordpress/wp-content/mu-plugins/vps-performance.php
```

```php
<?php
/*
Plugin Name: VPS Performance Enforcer
Description: Enforces performance settings for 1GB VPS
Version: 1.0
*/

// Disable plugin and theme installation from admin
define('DISALLOW_FILE_MODS', true);

// Limit memory for WordPress
ini_set('memory_limit', '32M');

// Disable file editing
define('DISALLOW_FILE_EDIT', true);

// Force object cache expiration
add_action('init', function() {
    if (!defined('WP_CACHE_KEY_SALT')) {
        define('WP_CACHE_KEY_SALT', md5(get_option('siteurl')));
    }
});

// Limit database queries
add_action('wp_footer', function() {
    if (defined('WP_DEBUG') && WP_DEBUG && current_user_can('manage_options')) {
        global $wpdb;
        echo '<!-- DB Queries: ' . $wpdb->num_queries . ' -->';
        if ($wpdb->num_queries > 50) {
            error_log('WordPress: High query count: ' . $wpdb->num_queries);
        }
    }
});

// Disable WordPress heartbeat (reduces AJAX calls)
add_action('init', function() {
    wp_deregister_script('heartbeat');
});

// Reduce heartbeat frequency when enabled
add_filter('heartbeat_settings', function($settings) {
    $settings['interval'] = 60; // 60 seconds instead of 15
    return $settings;
});

// Disable WordPress update checks for better performance
remove_action('init', 'wp_version_check');
remove_action('admin_init', '_maybe_update_core');
remove_action('admin_init', '_maybe_update_plugins');
remove_action('admin_init', '_maybe_update_themes');

// Limit login attempts (basic protection)
add_action('wp_login_failed', function($username) {
    $key = 'login_attempts_' . md5($_SERVER['REMOTE_ADDR']);
    $attempts = get_transient($key) ?: 0;
    set_transient($key, $attempts + 1, 300); // 5 minutes
    
    if ($attempts > 3) {
        wp_die('Too many login attempts. Please try again in 5 minutes.');
    }
});

// Successful login resets attempts
add_action('wp_login', function() {
    $key = 'login_attempts_' . md5($_SERVER['REMOTE_ADDR']);
    delete_transient($key);
});
?>
```

### WordPress Caching Without Plugins

Implement basic caching with nginx configuration:

```bash
# Configure nginx for WordPress caching
nano /etc/nginx/sites-available/wordpress
```

```nginx
server {
    listen 80;
    server_name your-domain.com;
    root /var/www/wordpress;
    index index.php index.html;

    # File upload limits (match PHP settings)
    client_max_body_size 32M;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;

    # Gzip compression for text files
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Cache static assets
    location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1M;
        add_header Cache-Control "public, immutable";
        access_log off;
        try_files $uri =404;
    }

    # Cache HTML for logged-out users
    location / {
        try_files $uri $uri/ /index.php?$args;
        
        # Simple page cache for anonymous users
        set $skip_cache 0;
        
        # Skip cache for logged in users or recent commenters
        if ($http_cookie ~* "comment_author|wordpress_[a-f0-9]+|wp-postpass|wordpress_logged_in") {
            set $skip_cache 1;
        }
        
        # Skip cache for WP admin, login, register
        if ($request_uri ~* "/wp-admin/|/wp-login.php|/wp-register.php") {
            set $skip_cache 1;
        }
        
        # Skip cache for POST requests
        if ($request_method = POST) {
            set $skip_cache 1;
        }
        
        # Skip cache for query strings
        if ($query_string != "") {
            set $skip_cache 1;
        }
    }

    # PHP processing
    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_intercept_errors on;
        fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        
        # Cache control for PHP
        fastcgi_cache_bypass $skip_cache;
        fastcgi_no_cache $skip_cache;
        fastcgi_cache WORDPRESS;
        fastcgi_cache_valid 200 60m;
        fastcgi_cache_valid 404 1m;
    }

    # Deny access to sensitive files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
    
    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }
    
    location ~* wp-config.php {
        deny all;
    }
}
```

Add FastCGI cache configuration to main nginx.conf:

```bash
nano /etc/nginx/nginx.conf
```

Add this inside the `http` block:

```nginx
# FastCGI cache for WordPress
fastcgi_cache_path /var/cache/nginx levels=1:2 keys_zone=WORDPRESS:100m inactive=60m;
fastcgi_cache_key "$scheme$request_method$host$request_uri";
fastcgi_cache_use_stale error timeout invalid_header http_500;
fastcgi_ignore_headers Cache-Control Expires Set-Cookie;
```

Create cache directory:

```bash
mkdir -p /var/cache/nginx
chown nginx:nginx /var/cache/nginx
chmod 755 /var/cache/nginx
```

### WordPress System Cron Replacement

Replace WordPress cron with system cron:

```bash
# Add to system crontab for better performance
echo "*/15 * * * * www-data /usr/bin/php /var/www/wordpress/wp-cron.php >/dev/null 2>&1" >> /var/spool/cron/crontabs/root
```

### WordPress Media Optimization

Optimize uploaded media automatically:

```bash
# Install image optimization tools
emerge -av media-gfx/jpegoptim media-gfx/optipng

# Create media optimization script
nano /usr/local/bin/wp-optimize-images.sh
```

```bash
#!/bin/bash
# Optimize WordPress media uploads

UPLOAD_DIR="/var/www/wordpress/wp-content/uploads"

# Optimize JPEG files
find "$UPLOAD_DIR" -name "*.jpg" -o -name "*.jpeg" | while read -r file; do
    jpegoptim --max=85 --strip-all "$file" 2>/dev/null
done

# Optimize PNG files  
find "$UPLOAD_DIR" -name "*.png" | while read -r file; do
    optipng -o2 "$file" 2>/dev/null
done

echo "WordPress media optimized"
```

```bash
chmod +x /usr/local/bin/wp-optimize-images.sh

# Run daily
echo "0 3 * * * /usr/local/bin/wp-optimize-images.sh" >> /var/spool/cron/crontabs/root
```

### CPU Governor Optimization

```bash
# Set CPU governor for maximum performance
echo 'performance' > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Make permanent
echo 'echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor' >> /etc/local.d/cpu-performance.start
chmod +x /etc/local.d/cpu-performance.start
```

### I/O Scheduler Optimization

```bash
# Optimize I/O scheduler for VPS SSD
echo 'noop' > /sys/block/vda/queue/scheduler

# Make permanent
echo 'echo noop > /sys/block/vda/queue/scheduler' >> /etc/local.d/io-scheduler.start
chmod +x /etc/local.d/io-scheduler.start
```

### Ultra-Minimal Logrotate

```bash
nano /etc/logrotate.d/ultra-minimal
```

```bash
/var/log/*.log {
    daily
    rotate 2              # Keep only 2 days
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
    maxsize 1M            # Rotate at 1MB
}
```

### Process Priority Optimization

```bash
# Set process priorities for critical services
nano /etc/local.d/process-priority.start
```

```bash
#!/bin/bash
# Ultra-performance process priorities

# High priority for network and critical services
renice -5 $(pgrep nginx) 2>/dev/null
renice -3 $(pgrep php-fpm) 2>/dev/null
renice -10 $(pgrep sshd) 2>/dev/null

# Lower priority for maintenance tasks
renice 10 $(pgrep mysql) 2>/dev/null
renice 15 $(pgrep cron) 2>/dev/null
```

```bash
chmod +x /etc/local.d/process-priority.start
```

---

## Important Warnings & Risks

These ultra-optimizations come with serious trade-offs:

- **Stability:** Pushing resource limits may lead to instability under heavy load
- **Data Loss:** Aggressive caching and memory management can risk data loss if not properly managed
- **Compatibility:** Some optimizations may not be compatible with all applications or services
- **Recovery:** Ultra-aggressive settings make system recovery more difficult
- **Debugging:** Minimal logging makes troubleshooting nearly impossible
- **Maintenance:** Some optimizations may break during system updates

**Critical Recommendation:** Test every single optimization on a staging environment first. These settings push a 1GB VPS to its absolute limits.

---

## Step 12: Extreme Performance Hacks

For the truly desperate who need every last byte:

### Disable Unused Filesystem Features

```bash
# Mount with even more aggressive options (DANGEROUS)
# Only if you have EXCELLENT backups
nano /etc/fstab
```

```bash
# EXTREME mount options (use at your own risk)
/dev/vda1 / ext4 defaults,noatime,nodiratime,nobarrier,commit=120,data=writeback,journal_async_commit,nodev,nosuid 0 0
```

### Ultra-Minimal Swap Configuration

```bash
# If you must have swap, make it ultra-minimal
dd if=/dev/zero of=/swapfile bs=1M count=512  # Only 512MB swap
chmod 600 /swapfile
mkswap /swapfile
echo '/swapfile none swap sw,pri=1,discard 0 0' >> /etc/fstab

# Ultra-aggressive swap settings
echo 'vm.swappiness=1' >> /etc/sysctl.conf        # Almost never swap
echo 'vm.vfs_cache_pressure=500' >> /etc/sysctl.conf  # Extreme cache pressure
```

### Memory Compaction

```bash
# Enable memory compaction for better memory utilization
echo 'vm.compact_memory=1' >> /etc/sysctl.conf
echo 'vm.compaction_proactiveness=0' >> /etc/sysctl.conf  # No proactive compaction

# Force memory compaction script
nano /usr/local/bin/force-compact.sh
```

```bash
#!/bin/bash
# Force memory compaction when memory is tight
MEM_AVAIL=$(awk '/MemAvailable/ {print $2}' /proc/meminfo)
if [ $MEM_AVAIL -lt 204800 ]; then  # Less than 200MB available
    echo 1 > /proc/sys/vm/compact_memory
    echo 3 > /proc/sys/vm/drop_caches
fi
```

```bash
chmod +x /usr/local/bin/force-compact.sh
echo "*/2 * * * * /usr/local/bin/force-compact.sh" >> /var/spool/cron/crontabs/root
```

### Ultra-Minimal DNS Configuration

```bash
# Use minimal DNS configuration
nano /etc/resolv.conf
```

```bash
# Only one DNS server to reduce memory
nameserver 1.1.1.1
options timeout:1 attempts:1 rotate
```

### Process Memory Limits

**For systemd systems:**

```bash
# Set strict memory limits for processes via systemd
nano /etc/systemd/system.conf
```

```bash
# Ultra-low default limits
[Manager]
DefaultMemoryAccounting=yes
DefaultMemoryHigh=800M
DefaultMemoryMax=900M
DefaultTasksMax=512
DefaultLimitNOFILE=1024
DefaultLimitNPROC=512
```

Create per-service memory limits:

```bash
# Already configured above in service override files
# nginx: 100M high, 150M max
# php-fpm: 200M high, 300M max  
# mariadb: 300M high, 400M max

# Apply limits
systemctl daemon-reload
systemctl restart nginx php-fpm mariadb
```

**For OpenRC systems:**

```bash
# Add to service scripts
nano /etc/conf.d/nginx
```

```bash
# Memory limit for nginx
command_args_background="--pid-file ${pidfile}"
start_stop_daemon_args="--nicelevel -5 --rlimit-as 157286400"  # 150MB limit
```

```bash
nano /etc/conf.d/php-fpm
```

```bash
# Memory limit for PHP-FPM
start_stop_daemon_args="--nicelevel -3 --rlimit-as 314572800"  # 300MB limit
```

**For both systems**, set global limits:

```bash
# Reduce limits for 1GB system
echo '* soft nofile 1024' >> /etc/security/limits.conf
echo '* hard nofile 2048' >> /etc/security/limits.conf
echo '* soft nproc 512' >> /etc/security/limits.conf
echo '* hard nproc 1024' >> /etc/security/limits.conf
echo '* soft memlock 64' >> /etc/security/limits.conf
echo '* hard memlock 128' >> /etc/security/limits.conf
```

---

## Verification & Testing Checklist

After applying these extreme optimizations:

```bash
# Memory usage verification
free -h && echo "Target: <800MB used"

# Service verification (OpenRC)
rc-status default && echo "All critical services running"

# Service verification (systemd)  
systemctl list-units --type=service --state=running
systemctl is-active nginx php-fpm mariadb

# Performance test
curl -w "@/dev/stdin" -o /dev/null -s "http://localhost/" <<< "
    time_namelookup:  %{time_namelookup}\n
    time_connect:     %{time_connect}\n
    time_total:       %{time_total}\n
"

# Load test (be careful!)
ab -n 100 -c 5 http://localhost/ 2>/dev/null | grep "Requests per second"

# Memory pressure test
stress --vm 1 --vm-bytes 128M --timeout 30s && echo "Survived memory stress"

# Systemd-specific checks
if command -v systemctl >/dev/null; then
    echo "=== Systemd Analysis ==="
    systemd-analyze blame | head -10
    systemd-analyze critical-chain
    systemctl status nginx php-fpm mariadb --no-pager
    journalctl --disk-usage
fi

# OpenRC-specific checks  
if command -v rc-status >/dev/null; then
    echo "=== OpenRC Analysis ==="
    rc-status -a
    /usr/local/bin/boot-analyze.sh
fi
```

### Emergency Recovery Commands

Keep these handy in case optimizations go too far:

```bash
# Emergency memory recovery
echo 3 > /proc/sys/vm/drop_caches
killall -9 php-fpm && /etc/init.d/php-fpm start

# Emergency service restart
/etc/init.d/nginx restart
/etc/init.d/mysql restart

# Restore from backup
cp /etc/sysctl.conf.backup /etc/sysctl.conf && sysctl -p
cp /etc/fstab.backup /etc/fstab
```

---

## Ultimate Resource Savings

With ALL optimizations applied, expect:

- **RAM savings:** 100-200MB (10-20% of total RAM)
- **CPU reduction:** 15-30% lower baseline usage
- **Boot time:** 30-50% faster startup  
- **I/O reduction:** 40-60% less disk activity
- **Network latency:** 10-20% improvement

**WordPress-specific savings:**
- **WordPress memory usage:** Reduced from ~80-120MB to ~30-50MB per page load
- **Database size:** 30-50% smaller after cleanup optimizations
- **Page load time:** 40-60% faster with minimal theme and optimizations
- **Plugin overhead:** Eliminated by using must-use performance enforcer
- **Media storage:** 20-40% smaller files with image optimization

**Total memory footprint for basic LEMP + WordPress:**
- **Before optimizations:** ~600-800MB
- **After system optimizations:** ~400-600MB  
- **After WordPress optimizations:** ~350-500MB
- **Extreme optimizations:** ~300-400MB

**WordPress performance improvements:**
- **Database queries:** Reduced from 50+ to <30 per page
- **HTTP requests:** Minimized by removing unnecessary assets
- **Cache hit ratio:** Improved with .htaccess caching rules
- **Admin performance:** Faster with disabled auto-updates and heartbeat

---

## Integration with Main Guide

This ultra-lightweight optimization works with:

- **Step 09:** [Maintenance](09-maintenance.md) - Critical monitoring required
- **Step 10:** [Advanced Hardening](10-advanced-hardening.md) - Choose carefully  
- **Step 07:** [Logging & Monitoring](07-logging-monitoring.md) - Essential for these settings

**Implementation Strategy:**
1. Complete main guide (Steps 1-9) and test thoroughly
2. Apply basic optimizations (Steps 1-6) and monitor for 48 hours
3. Add aggressive optimizations (Steps 7-10) if stable
4. Only apply extreme optimizations (Steps 11-12) if absolutely necessary

---

**Ultra-Optimization Complete** - You've squeezed every possible byte and CPU cycle from your 1GB VPS. Your server is now running at the absolute limits of efficiency. Monitor religiously and have excellent backups!