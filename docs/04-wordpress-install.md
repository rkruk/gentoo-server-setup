
<i>Add HTTPS support to your Gentoo VPS using Let's Encrypt, certbot, and NGINX. Supports both HTTP challenge and DNS challenge (for headless/VPS setups).</i>

# 04 – HTTPS with Let's Encrypt

> Configure free HTTPS using Let's Encrypt certificates via `certbot`.  
> Supports HTTP-01 challenge (automatic) and DNS-01 challenge (manual or API-based).

---

## Step 1: Install Certbot

Install certbot with NGINX plugin:

```bash
emerge --ask app-crypt/certbot www-servers/nginx app-crypt/acme-sh
```

You can use `acme.sh` as a lighter alternative to certbot, especially for DNS-only VPS setups. See below for both options.

Method A: HTTP Challenge (automatic)

Requires port 80 to be open and pointed to your server.

Make sure your NGINX site is correctly configured:

```bash
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/wordpress;
}
```

Run certbot:

```bash
certbot certonly --nginx -d example.com -d www.example.com
```

Certbot installs certs under:

```bash
/etc/letsencrypt/live/example.com/
```

Update NGINX to use HTTPS:

```bash
server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    root /var/www/wordpress;
    index index.php index.html;

    # ... PHP location etc ...
}
```

Reload nginx:

```bash
/etc/init.d/nginx reload
```
Or:

```bash
systemctl reload nginx
```

<i>Here is the rest of the section with DNS challenge setup and automation:</i>  

## Method B: DNS Challenge (headless server or no port 80 access)

Use this if:
- Port 80 is blocked
- You don't want to expose HTTP during issuance
- You're using Cloudflare or any DNS provider with an API

---

### Option 1: Manual DNS challenge (basic)

Run:

```bash
certbot certonly --manual --preferred-challenges dns -d example.com -d www.example.com
```

Certbot will prompt you to:

Create a TXT record under `_acme-challenge.example.com`

Wait for DNS to propagate

After validation, certificates are saved to:

```bash
/etc/letsencrypt/live/example.com/
```

❗ You’ll need to repeat this manually every 90 days unless automated.

### Option 2: Automated with acme.sh (lightweight, scriptable)

Install:

```bash
acme.sh --install
```

Set default CA (optional):

```bash
acme.sh --set-default-ca --server letsencrypt
```

Use with DNS API (example: Cloudflare):

```bash
export CF_Token="your-cloudflare-token"
export CF_Account_ID="your-account-id"

acme.sh --issue --dns dns_cf -d example.com -d www.example.com
```

Install cert to system paths:

```bash
acme.sh --install-cert -d example.com \
  --key-file /etc/ssl/private/example.key \
  --fullchain-file /etc/ssl/certs/example.crt \
  --reloadcmd "rc-service nginx reload"
```

Add to crontab:

```bash
crontab -e
```

Add:

```bash
0 3 * * * "/root/.acme.sh"/acme.sh --cron --home "/root/.acme.sh" > /dev/null
```

Auto Renewal (for certbot)
Create a cron job for certbot (if not using `acme.sh`):

```bash
crontab -e
```

Add:

```bash
0 2 * * * certbot renew --quiet --deploy-hook "/etc/init.d/nginx reload"
```

Test it anytime with:

```bash
certbot renew --dry-run
```

Optional: Harden TLS

In your NGINX server block:

```bash
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers on;
ssl_ciphers EECDH+AESGCM:EDH+AESGCM;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_stapling on;
ssl_stapling_verify on;

add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
```


