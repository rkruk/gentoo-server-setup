
# 02 – SSH Hardening and Firewall

This section covers:

- Disabling password SSH logins
- Enabling key-only access
- Optionally changing the SSH port
- Enabling 2FA (optional, via google-authenticator-libpam)
- A sane, tested iptables firewall with optional ipset for GeoIP blocking
<br><br>

> Lock down external access to your Gentoo VPS. Enforce key-based SSH login, limit brute-force risk, and apply a minimalist but solid firewall.

---

## Step 1: SSH Key Authentication Only

Login as root or `adminuser`, and edit the SSH server config:

```bash
nano /etc/ssh/sshd_config
```

Change or verify the following lines:

```bash
Port 22                      # You can change this to a high port (e.g., 2222 or 6066 whatever you fancy really) if desired
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
AllowUsers adminLarry
```

Replace `adminLarry` with your actual username.

Save and restart SSH:

```bash
/etc/init.d/sshd restart
```

If using systemd:

```bash
systemctl restart sshd.service
```

## Step 2: Set Up SSH Key Authentication

On your local machine (not the server):

```bash
ssh-keygen -t ed25519 -C "your@email.com"
```

Then copy the public key to the server:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub adminLarry@your-server-ip
```
Test that you can SSH in without a password, then disable password logins if you haven’t already.

## (Optional) Step 3: Enable Two-Factor SSH (2FA)

Install PAM Google Authenticator:

```bash
emerge -av sys-auth/google-authenticator
```

Then, as an user (adminLarry in this example - not root!):

```bash
google-authenticator
```

Answer:
`Do you want authentication tokens to be time-based?` → `y`

Save the emergency codes securely

Edit PAM SSH config:

```bash
nano /etc/pam.d/sshd
```

Add this at the top:

```bash
auth required pam_google_authenticator.so
```

Then in /etc/ssh/sshd_config:

```bash
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
```

Restart SSH and test 2FA.

## Step 4: Configure Firewall with iptables

Basic rules (IPv4 only, you can extend to IPv6 if needed):

```bash
iptables -F
iptables -X
iptables -t nat -F

iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT

# Allow established/related
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (adjust port if changed!)
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Log dropped packets (optional)
iptables -A INPUT -j LOG --log-prefix "iptables denied: " --log-level 7
```

Save rules:

```bash
iptables-save > /etc/iptables.rules
```

Then load on boot:

```bash
nano /etc/conf.d/iptables
```

Set:

```bash
IPTABLES_SAVE="/etc/iptables.rules"
```

Enable the service:

```bash
rc-update add iptables default
/etc/init.d/iptables start
```

## (Optional) Step 5: Block Countries Using ipset + xtables-addons

Install required tools:

```bash
emerge --ask net-firewall/ipset
```

Create update script:
Save as `/usr/local/bin/update-ipsets.sh`:

```bash
#!/bin/bash

COUNTRIES="cn ru"
BASE_URL="https://www.ipdeny.com/ipblocks/data/countries"
TMPDIR="/tmp/ipsets"
mkdir -p "$TMPDIR"

for COUNTRY in $COUNTRIES; do
    ZONE_FILE="${TMPDIR}/${COUNTRY}.zone"
    echo "[*] Downloading $COUNTRY block list..."
    curl -s -o "$ZONE_FILE" "${BASE_URL}/${COUNTRY}.zone"

    ipset list "$COUNTRY" >/dev/null 2>&1
    if [ $? -ne 0 ]; then
        ipset create "$COUNTRY" hash:net
    else
        ipset flush "$COUNTRY"
    fi

    while read -r IP; do
        ipset add "$COUNTRY" "$IP" 2>/dev/null
    done < "$ZONE_FILE"

    echo "[+] Loaded $(wc -l < "$ZONE_FILE") IPs into $COUNTRY"
done

for COUNTRY in $COUNTRIES; do
    iptables -C INPUT -m set --match-set "$COUNTRY" src -j DROP 2>/dev/null
    if [ $? -ne 0 ]; then
        iptables -A INPUT -m set --match-set "$COUNTRY" src -j DROP
        echo "[+] Applied iptables rule for $COUNTRY"
    fi
done

echo "[✓] IP sets updated and rules applied."
```

Make it executable:

```bash
chmod +x /usr/local/bin/update-ipsets.sh
```

Run it:

```bash
/usr/local/bin/update-ipsets.sh
```

Auto-run on boot (OpenRC):

Create `/etc/local.d/ipsets.start`:

```bash
#!/bin/sh
/usr/local/bin/update-ipsets.sh
````

Make it executable:

```bash
chmod +x /etc/local.d/ipsets.start
rc-update add local default
```

ipdeny.com provides regularly updated country zone files over HTTPS. While undocumented, it is actively maintained and functional as of 2025. 
You could additionally do some script to check for the updates of the data from the websites. Up to you really.



