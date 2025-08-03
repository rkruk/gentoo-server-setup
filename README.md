<p align="center">
  <img src="https://github.com/rkruk/gentoo-server-setup/blob/master/larry.png">
</p>
<br>
<p align="center"><b>GENTOO WEB SERVER<br>COMPLETE SETUP GUIDE</b></p>

<p align="center"><b>A real-world, tested guide for deploying secure WordPress or Node.js apps on a minimal VPS running Gentoo Linux</b></p>

---

## Target Audience

This guide is for system administrators, developers, or infrastructure engineers who want to deploy and maintain a **lean, hardened, reliable** small server for:
-  Hosting WordPress websites
-  Running custom Node.js applications  
-  Handling mail (system + app) via SMTP
-  Using a Gentoo-based VPS (Linode, Vultr, etc.)

---

##  Target Environment

- **Base OS:** Gentoo Linux (stage3, amd64)
- **Kernel:** VPS-supplied (stock), not self-compiled
- **Init System:** OpenRC _or_ systemd (auto-detected)
- **Resources:** Minimum 1 GB RAM, 1 vCPU, 40 GB SSD
- **Philosophy:** No Docker, no containers - pure metal efficiency

---

##  Step-by-Step Documentation

This guide is organized into logical, version-controlled steps. **Follow them in order:**

| Step | Document | Description |
|------|----------|-------------|
| **00** | [Overview](docs/00-overview.md) | Project overview, philosophy, and what's included |
| **01** | [Initial Setup](docs/01-initial-setup.md) | Sync packages, install tools, create users, enable cron + msmtp |
| **02** | [SSH & Firewall](docs/02-user-ssh-firewall.md) | SSH hardening, key-only auth, 2FA, iptables + ipset, country blocking |
| **03** | [NGINX + PHP + MySQL](docs/03-nginx-php-mysql.md) | Complete LEMP stack installation and basic configuration |
| **04** | [HTTPS Setup](docs/04-HTTPS.md) | Let's Encrypt SSL certificates (HTTP + DNS challenges) |
| **05** | [WordPress Install](docs/05-wordpress-install.md) | WordPress download, database setup, security, WP-CLI |
| **06** | [Node.js Apps](docs/06-nodejs-app.md) | Node.js deployment, reverse proxy, process management |
| **07** | [Logging & Monitoring](docs/07-logging-monitoring.md) | Log rotation, msmtp integration, Netdata setup |
| **08** | [IDS & Audits](docs/08-ids-audits.md) | **Optional:** Lynis audits, Suricata/Snort IDS |
| **09** | [Maintenance](docs/09-maintenance.md) | Backups, system updates, monitoring best practices |

---

## What's Included

| Component | Purpose | Notes |
|-----------|---------|-------|
| **nginx** | Web server | HTTP/2, TLS 1.3, Brotli, static + PHP-FPM |
| **php-fpm** | WordPress runtime | Minimal extensions, security-focused |
| **MariaDB** | Database | MySQL-compatible, lightweight config |
| **Node.js** | App runtime | For APIs or custom applications |
| **msmtp** | Outbound mail | System alerts + WordPress + Node SMTP |
| **iptables + ipset** | Firewall | GeoIP blocking, rate limiting |
| **fail2ban** | Brute-force protection | SSH and web service protection |
| **Let's Encrypt** | SSL/TLS certificates | Automated renewal via certbot/acme.sh |
| **Netdata** | Real-time monitoring | Resource usage, performance metrics |
| **Lynis** | Security auditing | **Optional** system hardening checks |

---

## Quick Start

1. **Clone this repository:**
   ```bash
   git clone https://github.com/rkruk/gentoo-server-setup.git
   cd gentoo-server-setup
   ```

2. **Start with the overview:**
   ```bash
   cat docs/00-overview.md
   ```

3. **Follow the steps in order:**
   - Each document builds on the previous one
   - All commands are tested on live VPS deployments
   - Skip optional sections (marked clearly) if desired

---

## Design Philosophy

- **Keep it real.** All examples tested on production VPS deployments
- **Keep it lean.** Optimized for 1-2 GB RAM VPS environments  
- **Keep it secure.** Defense in depth: SSH keys, 2FA, firewalls, monitoring
- **Keep it maintainable.** Clear separation of concerns, documented decisions

---

## What This Guide Avoids

-  Docker or containers (resource overhead on small VPS)
-  Self-compiled kernels (leave to VPS provider)
-  Full mail server setup (use external SMTP instead)
-  Kitchen-sink PHP/service installations

---

##  Tips & Best Practices

###  **System Administration**
- Use `su -` for root escalation (no sudo by default)
- Keep packages minimal - only install what you need
- Test configurations before applying (`nginx -t`, `iptables-restore --test`)

###  **Security Hardening**
- **SSH:** Keys-only, custom port, 2FA optional
- **Firewall:** Default deny, explicit allows, country blocking
- **Updates:** Regular `emerge --sync && emerge -uavDN @world`
- **Monitoring:** Netdata + email alerts for anomalies

###  **Performance Optimization**
- **NGINX:** HTTP/2, Brotli compression, FastCGI caching
- **PHP-FPM:** Tuned for small VPS (5-50 workers)
- **MariaDB:** Minimal configuration, query cache
- **Node.js:** PM2 process manager, reverse proxy

###  **Monitoring & Alerts**
- **Logs:** Centralized via rsyslog, rotated daily
- **Mail:** System alerts via msmtp to external SMTP
- **Metrics:** Netdata web interface on port 19999
- **Audits:** Weekly Lynis reports (optional)

###  **Maintenance Workflow**
1. **Weekly:** Check logs, review Netdata metrics
2. **Monthly:** Update packages, restart services if needed  
3. **Quarterly:** Run Lynis audit, review firewall rules
4. **As needed:** Backup WordPress/Node.js app data


---

##  Troubleshooting

### Common Issues:

**SSH Connection Problems:**
- Check firewall rules allow your IP
- Verify SSH service is running: `rc-service sshd status`
- Check SSH logs: `tail -f /var/log/auth.log`

**SSL Certificate Issues:**
- Verify DNS points to your server
- Check port 80 accessibility for HTTP challenge
- Use DNS challenge if port 80 blocked

**Performance Issues:**
- Monitor with Netdata: `http://your-server:19999`
- Check PHP-FPM status: `rc-service php-fpm status`
- Review NGINX error logs: `tail -f /var/log/nginx/error.log`

**Package Conflicts:**
- Update Portage tree: `emerge --sync`
- Check USE flags: `emerge --info | grep USE`
- Resolve blocks: `emerge --depclean`

---

##  Contributing

Contributions welcome! This guide is battle-tested but always improving:

-  **Bug reports:** Found an error? Open an issue
-  **Improvements:** Better commands or configurations? Pull request
-  **Documentation:** Clearer explanations? Much appreciated
-  **Testing:** Verified on different VPS providers? Share your experience

### How to Contribute:
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Test your changes on a real VPS
4. Commit with clear messages (`git commit -m 'Add amazing feature'`)
5. Push to the branch (`git push origin feature/amazing-feature`)
6. Open a Pull Request

---

##  License

This project is licensed under the **MIT License**. See [LICENSE](LICENSE) for details.

---

##  Support & Community

- **Issues:** [GitHub Issues](https://github.com/rkruk/gentoo-server-setup/issues)
- **Discussions:** Use GitHub Discussions for questions
- **Security:** For security issues, please use private reporting

---

##  Acknowledgments

- **Gentoo Community:** For the amazing distribution
- **Contributors:** Everyone who tested and improved this guide
- **VPS Providers:** Linode, Vultr, DigitalOcean for test environments
- **Security Researchers:** For ongoing security best practices

---

*Built with ❤️ for the Gentoo community. Keep it lean, keep it secure.*
