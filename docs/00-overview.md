# Gentoo Web Server Setup – Overview

> **A real-world, tested setup for deploying WordPress or Node.js apps on a minimal VPS running Gentoo Linux.**

---

## Target Audience

This guide is for system administrators, developers, or infrastructure engineers who want to deploy and maintain a **lean, hardened, reliable** small server for:
- Hosting WordPress websites
- Running custom Node.js applications
- Handling mail (system + app) via SMTP
- Using a Gentoo-based VPS (for example from Linode)

---

## Target Environment

- **Base OS:** Gentoo Linux (stage3, amd64)
- **Kernel:** VPS-supplied (stock), not self-compiled
- **Init System:** OpenRC _or_ systemd (whichever your VPS image uses)
- **Resources:** Minimum 1 GB RAM, 1 vCPU, 40 GB SSD
- **No Docker, no LXC, no containers** - sorry not in the scope here.

This guide assumes full root access.

---

## What’s Included

| Component           | Purpose                         | Notes |
|---------------------|----------------------------------|-------|
| `nginx`             | Web server                      | With Brotli, HTTP/2, TLS 1.3, static & PHP |
| `php-fpm`           | WordPress runtime               | Minimal modules only |
| `MariaDB`           | MySQL-compatible DB             | For WordPress |
| `Node.js` + `npm`   | Node app runtime                | For APIs or custom apps |
| `msmtp`             | Outbound mail                   | System mail + WP mail + Node SMTP |
| `iptables` + `ipset`| Firewall + country blocking     | Optional GeoIP blocking |
| `fail2ban` or `sshguard` | Brute-force protection    | Lightweight option depends on your use |
| `AppArmor`          | Process-level hardening         | Optional but encouraged |
| `Lynis`             | Security audits                 | Optional |
| `Netdata`           | Resource monitoring             | Optional dashboard |
| `Logrotate`         | Log management                  | Default Gentoo tool |

---

## Design Philosophy

- **Keep it real.** All examples are tested on live VPS deployments.
- **Keep it lean.** Minimal resource use, suitable for 1 GB VPS.
- **Keep the base secure.** Key-based SSH, 2FA, rate-limiting, headers, firewall.

---

## What This Guide Avoids

- ❌ Docker or containers (that would be wasteful usage of resources here)
- ❌ Self-compiled kernel (leave this to VPS provider)
- ❌ Mail server stack (use external SMTP with `msmtp`)
- ❌ Unfiltered PHP extensions or services

---

## How This Guide is Structured

This project is broken into logical, version-controlled steps:

- 01-initial-setup.md → Sync, tools, su -, service user
- 02-user-ssh-firewall.md → SSH hardening, fail2ban/sshguard, firewall
- 03-nginx-php-mysql.md → Full LEMP stack for WordPress
- 04-wordpress-install.md → WordPress download, config, permissions
- 05-nodejs-app.md → Node app setup, reverse proxy, systemd/OpenRC
- 06-logging-monitoring.md → Logs, logrotate, msmtp, netdata
- 07-ids-audits.md → Optional: Lynis, Suricata/Snort
- 08-maintenance.md → Backups, upgrades, monitoring tips

