# 01 – Initial System Setup

> Prepare your Gentoo VPS: update packages, install essentials, configure a normal user, and set up basic services. Root access is controlled via `su -`. This is the first step before configuring your services.

---

## Prerequisites

- You are logged in as `root`
- Gentoo is already installed (e.g., via a VPS provider)
- You are using either OpenRC or systemd (we will detect automatically later)
- You are not using containers, docker, or exotic system overlays

---

## Step 1: Sync and Update System

Synchronize Portage tree:

```bash
emerge --sync
eselect news read
```

Then update world:

```bash
emerge -uavDN @world
```

## Step 2: Install Essential System Tools

Install critical packages:

```bash
emerge -av app-admin/logrotate net-analyzer/ipset net-firewall/iptables mail-mta/msmtp sys-process/cronie
```

Optional (but useful):

```bash
emerge -av sys-apps/mlocate app-misc/tmux app-editors/nano
```

## Step 3: Create an Unprivileged User

Instead of using user with sudo privileges, you will escalate privileges via su - using the root password (so many will not agree with me on that - fuck'em). 

```bash
useradd -m -s /bin/bash adminLarry
passwd adminLarry
```

Now log in as admin user `adminLarry` over SSH and escalate only when needed using:

```bash
su -
```

This keeps root access out of regular sessions and reduces attack surface (I think..).

## Step 4: Enable Cron

Cron is used for automation tasks, backups, log rotation, and certificates (obviously..).

Enable and start the cron daemon:

```bash
rc-update add cronie default
rc-service cronie start
```

If your system uses systemd:

```bash
systemctl enable cronie.service
systemctl start cronie.service
```

## Step 5: Configure msmtp (Lightweight Outbound Mail)

This allows system tools and your apps (WordPress, Node.js) to send email (two birds one stone - no need to complicate that).

1. Create the config file:

```bash
nano /etc/msmtprc
```

Example configuration (replace with your actual SMTP provider settings):

```bash
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        /var/log/msmtp.log

account        default
host           smtp.example.com
port           587
from           server@example.com
user           smtp_user
passwordeval   "cat /etc/msmtp-password"
```

2. Create the password file:

```bash
echo "your-smtp-password" > /etc/msmtp-password
chmod 600 /etc/msmtp-password
chown root:root /etc/msmtp-password
```

3. Test outbound email:

```bash
echo "Test message from Gentoo server" | msmtp -a default your@email.com
```

4. Make msmtp your system mailer:
   
Replace sendmail binary (used by cron, logwatch, etc.):

```bash
ln -sf /usr/bin/msmtp /usr/sbin/sendmail
```

## Step 6: Configure Swap (Critical for 1GB RAM VPS)

Create a 2GB swap file for memory management:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

Make it permanent:

```bash
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

Optimize swap usage for VPS:

```bash
echo 'vm.swappiness=10' >> /etc/sysctl.conf
echo 'vm.vfs_cache_pressure=50' >> /etc/sysctl.conf
sysctl -p
```

Verify:

```bash
free -h
cat /proc/swaps
```

---

At this point, your system is:
- Updated to latest packages
- Running as a non-root user by default
- Using su - for privilege escalation
- Equipped with cron and logging
- Capable of sending email via external SMTP
- Optimized with swap for 1GB RAM VPS
- No unnecessary services have been installed. No container tools, web panels, or daemons you don't need to control.

Next step → 02 – SSH and Firewall Hardening


