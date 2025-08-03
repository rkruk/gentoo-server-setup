<i>Add optional intrusion detection and system auditing using tools like Lynis and Suricata. Lightweight, practical, and suitable for VPS use.</i>

# 08 – Intrusion Detection & Auditing

> This section is optional. Use it to harden and audit your Gentoo VPS with basic IDS and system auditing tools.

---

## Step 1: Install Lynis (System Audit Tool)

Lynis is a lightweight, no-daemon system auditing tool.

Install:

```bash
emerge --ask app-forensics/lynis
```

Run a full audit:

```bash
lynis audit system
```

Review suggestions under:

```bash
Suggestions (system hardening)
Warnings (security risks)
```

Use this to:

- Discover missing kernel or sysctl hardening
- Detect insecure configurations (e.g., SSH, permissions)
- See logs without sending anything externally

You can re-run Lynis after changes to track improvements.

## Step 2: Install Suricata (Network IDS)

! Note: Suricata can consume CPU/RAM. Use only on capable systems!

Install:

```bash
emerge --ask net-analyzer/suricata
```

Create config:

```bash
cp /etc/suricata/suricata.yaml /etc/suricata/suricata.yaml.bak
nano /etc/suricata/suricata.yaml
```

Minimal tuning:
- Set interface name (e.g., eth0)
- Disable or reduce rulesets if on 1 GB RAM

Start it:

```bash
rc-update add suricata default
/etc/init.d/suricata start
```

Or with systemd:

```bash
systemctl enable --now suricata
```

Logs:

```bash
less /var/log/suricata/fast.log
```

Optional: use with eve.json for detailed logs or external integrations.

## Step 3: Alternatives (Low-Resource)

If Suricata is too heavy:
- Use psad (Port Scan Detection)
- Enable more verbose logging via iptables
- Use fail2ban or sshguard (already configured earlier)

## Step 4: Manual Security Checklist

Things worth checking even without tools:
- Root login via SSH is disabled
- Password auth is disabled (keys only)
- User separation (no root work via SSH)
- NGINX/Apache hardened (no info leaks)
- Kernel has sysctl hardening (e.g., kernel.kptr_restrict, randomize_va_space)
- Cron + email notifications working
- Backups (covered next)



