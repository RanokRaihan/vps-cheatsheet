# Portfolio Backend — VPS Deploy Cheatsheet

---

## 1. Git Clone

```bash
cd /var/www
git clone <github repository>
cd <dir>
npm install --production=false
npm run build
```

Create `.env`:

```bash
nano .env
```

Fill in: `PORT=5001`, `MONGODB_URI`, `JWT_SECRET`, `CORS_ORIGIN=https://ranokraihan.com`, etc.

**Generating secrets on the terminal:**

```bash
# Option 1 — openssl (recommended, always available) with hex
openssl rand -hex 12
# Option 2 — openssl (recommended, always available)
openssl rand -base64 128

# Option 3 — Node.js
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
```

Copy the output and paste it directly as the value for `JWT_SECRET` (or any secret) in your `.env`. Run it once per secret you need.

---

## 2. Check Already Used Ports

Before picking a port, confirm what's already running:

```bash
pm2 list                            # see all PM2 processes and their names

sudo lsof -i -P -n | grep LISTEN   # all ports currently listening on the VPS
sudo ss -tlnp                       # alternative, faster
```

> Pick a port not already in use. Doable is on 8001 — so use something like 8002, 8003, etc.

---

## 3. PM2

Start the app:

```bash
pm2 start dist/server.js --name <name> // name: provide any name you want for the app
pm2 save
```

Useful commands:

```bash
pm2 list                    # confirm it's running
pm2 logs <name>      # live logs
pm2 restart <name>   # after a redeploy
```

---

## 3. Nginx

Create the config file:

```bash
sudo nano /etc/nginx/sites-available/<name> // give the same name as pm2 (recommended)
```

Paste this:

```nginx
server {
    listen 80;
    server_name <domain name ex:api.example.com>;

    location / {
        proxy_pass http://localhost:5001; // replace the post where the app runnuing
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

Enable and reload:

```bash
sudo ln -s /etc/nginx/sites-available/<name> /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 4. Certbot (SSL)

```bash
sudo certbot --nginx -d <domain ex:api.example.com>
```

> **Before running this:** make sure the `api` A record in Cloudflare is set to **DNS only** (grey cloud), not Proxied. Flip it back to Proxied after if you want.

Verify auto-renewal works:

```bash
sudo certbot renew --dry-run
```

---

## Smoke Test

```bash
curl <health check url>
```

---

## Future Redeploy (quick reference)

```bash
cd <directory of the app>
git pull origin main
npm install --production=false
npm run build
pm2 restart <name>
pm2 logs <name>
```
