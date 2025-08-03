<i>Deploy and serve a Node.js web app with NGINX reverse proxy and process supervision (OpenRC or systemd).</i>

# 06 – Node.js App Deployment

> Deploy a Node.js application behind NGINX with system process management.

---

## Step 1: Install Node.js (LTS)

Use the official binary or Gentoo portage.

### Option A: Portage (simpler to update later)

```bash
emerge --ask net-libs/nodejs
```

### Option B: Official Binary (if portage is outdated)

```bash
cd /opt
curl -LO https://nodejs.org/dist/v20.14.0/node-v20.14.0-linux-x64.tar.xz
tar -xf node-*.tar.xz
ln -s node-v20.14.0-linux-x64 node
ln -s /opt/node/bin/node /usr/local/bin/node
ln -s /opt/node/bin/npm /usr/local/bin/npm
```

## Step 2: Create the App

```bash
mkdir -p /var/www/nodeapp
cd /var/www/nodeapp
npm init -y
npm install express
```

Create index.js:

```js
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.send('Node app is working!');
});

app.listen(port, '127.0.0.1', () => {
  console.log(`App listening at http://127.0.0.1:${port}`);
});
```

Make nginx the owner:

```bash
chown -R nginx:nginx /var/www/nodeapp
```

## Step 3: Run App with OpenRC or systemd

Option A: OpenRC service

Create `/etc/init.d/nodeapp`:

```bash
#!/sbin/openrc-run

command="/usr/local/bin/node"
command_args="/var/www/nodeapp/index.js"
command_background=true
pidfile="/run/nodeapp.pid"
name="nodeapp"

depend() {
    need net
}
```

Then:

```bash
chmod +x /etc/init.d/nodeapp
rc-update add nodeapp default
/etc/init.d/nodeapp start
```

Option B: systemd unit

Create `/etc/systemd/system/nodeapp.service`:

```bash
[Unit]
Description=Node.js Express App
After=network.target

[Service]
ExecStart=/usr/local/bin/node /var/www/nodeapp/index.js
WorkingDirectory=/var/www/nodeapp
Restart=always
User=nginx
Group=nginx
Environment=NODE_ENV=production
PIDFile=/run/nodeapp.pid

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reexec
systemctl enable --now nodeapp.service
```

## Step 4: NGINX Reverse Proxy

Create `/etc/nginx/sites-enabled/nodeapp.conf`:

```nginx
server {
    listen 80;
    server_name node.yourdomain.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Test & reload:

```bash
nginx -t
/etc/init.d/nginx reload
```

Or:

```bash
systemctl reload nginx
```

## Step 5: Add TLS

Update the reverse proxy server block to:

```nginx
listen 443 ssl;
server_name node.yourdomain.com;

ssl_certificate     /etc/letsencrypt/live/node.yourdomain.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/node.yourdomain.com/privkey.pem;

location / {
    proxy_pass http://127.0.0.1:3000;
    # ... same as above ...
}
```


