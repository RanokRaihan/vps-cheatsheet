# Node.js Backend — VPS Deploy Cheatsheet

A quick reference for deploying a Node.js backend app to a VPS, fronted by Nginx with a free SSL certificate.

Replace anything in `<angle brackets>` with your actual values.

---

## 0. Prerequisites

Make sure these are already installed on the VPS:

```bash
node -v     # Node.js (LTS recommended)
npm -v
pm2 -v      # process manager — install with: npm install -g pm2
nginx -v
certbot --version
```

You'll also need:

- A domain (or subdomain) pointed at the VPS's IP address.
- SSH access to the VPS.

---

## 1. Clone and Build

```bash
cd /var/www
git clone <repository-url>
cd <app-directory>
npm install --production=false
npm run build
```

Create the environment file:

```bash
nano .env
```

Fill in whatever your app needs, e.g. `PORT=5001`, `MONGODB_URI=...`, `JWT_SECRET=...`, `CORS_ORIGIN=https://<your-domain>`.

**Generating secrets:**

```bash
# Option 1 — openssl, hex output
openssl rand -hex 128

# Option 2 — openssl, base64 output
openssl rand -base64 128

# Option 3 — Node.js
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

Run once per secret and paste the output as the value (e.g. for `JWT_SECRET`) in `.env`.

---

## 2. Pick a Free Port

Check what's already running before choosing a port for the new app:

```bash
pm2 list                          # PM2 processes and their names
sudo lsof -i -P -n | grep LISTEN  # every port currently listening
sudo ss -tlnp                     # faster alternative to lsof
```

Pick a port that isn't already in use, and make sure it matches the `PORT` value in your `.env`.

---

## 3. Run the App with PM2

Start the app:

```bash
pm2 start dist/server.js --name <app-name>
pm2 save
```

`<app-name>` can be anything — it's just how you'll refer to the process in PM2 commands below.

Make PM2 restart your apps automatically after a server reboot (run once per server):

```bash
pm2 startup
pm2 save
```

Useful commands:

```bash
pm2 list                  # confirm it's running
pm2 logs <app-name>       # live logs
pm2 restart <app-name>    # after a redeploy
pm2 delete <app-name>     # remove the process entirely
```

---

## 4. Configure Nginx as a Reverse Proxy

Create the config file (using the same name as the PM2 process is a good convention):

```bash
sudo nano /etc/nginx/sites-available/<app-name>
```

Paste this, replacing `<domain>` and the port:

```nginx
server {
    listen 80;
    server_name <domain>;

    location / {
        proxy_pass http://localhost:<port>;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable the site and reload Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/<app-name> /etc/nginx/sites-enabled/
sudo nginx -t              # test the config before reloading
sudo systemctl reload nginx
```

---

## 5. Firewall (if using ufw)

Make sure HTTP/HTTPS traffic is allowed:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw status
```

---

## 6. SSL with Certbot

```bash
sudo certbot --nginx -d <domain>
```

> **Before running this:** if your DNS is behind a proxy (e.g. Cloudflare's orange cloud), switch the record to **DNS only** (grey cloud) first, so Certbot can verify the domain. You can switch it back to proxied afterwards.

Verify auto-renewal works:

```bash
sudo certbot renew --dry-run
```

---

## 7. Smoke Test

```bash
curl https://<domain>/<health-check-path>
```

---

## Future Redeploys (quick reference)

```bash
cd /var/www/<app-directory>
git pull origin main
npm install --production=false
npm run build
pm2 restart <app-name>
pm2 logs <app-name>
```
