<i>Keep your Gentoo VPS running clean, secure, and up-to-date with safe system update routines, backups, and runtime checks.</i>

# 09 – Maintenance, Backups & Updates

> Final steps to ensure your Gentoo VPS stays stable, secure, and easy to recover.

---

## Step 1: World Updates

Update system packages regularly:

```bash
emerge --sync
emerge -upvDN @word
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

💾 Step 2: Backups
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


