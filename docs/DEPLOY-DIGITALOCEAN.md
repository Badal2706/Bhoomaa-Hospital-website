# Deploying to a DigitalOcean Droplet

This is a static site — HTML, CSS, images, no server-side code and no database. It needs a web server and nothing else. Nginx is the right choice.

Assumes: Ubuntu 22.04 or 24.04 droplet, domain `bhoomaahospital.com` at GoDaddy.

Replace `bhoomaahospital.com` and `YOUR_DROPLET_IP` throughout.

---

## Step 1 — Point the domain at the droplet

Get your droplet's public IPv4 from the DigitalOcean dashboard.

In **GoDaddy → My Products → Domain → DNS → Manage Zones**, set:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `@` | `YOUR_DROPLET_IP` | 600 |
| A | `www` | `YOUR_DROPLET_IP` | 600 |

Delete any existing parking/forwarding records for `@` and `www` first, or they will conflict.

DNS usually propagates in 10–30 minutes. Check with:

```bash
nslookup bhoomaahospital.com
```

Do not continue to the SSL step until this returns your droplet IP — certificate issuance will fail otherwise.

---

## Step 2 — Connect and prepare the server

```bash
ssh root@YOUR_DROPLET_IP
```

```bash
apt update && apt upgrade -y
apt install nginx -y

ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw --force enable
```

Visit `http://YOUR_DROPLET_IP` — you should see the Nginx welcome page.

---

## Step 3 — Create the site directory

```bash
mkdir -p /var/www/bhoomaahospital.com/html
chown -R $USER:$USER /var/www/bhoomaahospital.com/html
chmod -R 755 /var/www/bhoomaahospital.com
```

---

## Step 4 — Upload the files

From **your Windows machine**, in PowerShell, from inside the `Bhooma Website` folder:

```powershell
scp index.html robots.txt sitemap.xml site.webmanifest root@YOUR_DROPLET_IP:/var/www/bhoomaahospital.com/html/
scp -r assets root@YOUR_DROPLET_IP:/var/www/bhoomaahospital.com/html/
```

Upload **only** those four files plus `assets/`. Do not upload `_DELETE-THESE/`, `_original-photos-do-not-upload/`, or the `.md` files.

Verify on the server:

```bash
ls -la /var/www/bhoomaahospital.com/html
ls /var/www/bhoomaahospital.com/html/assets | wc -l    # expect 20
```

---

## Step 5 — Nginx configuration

```bash
nano /etc/nginx/sites-available/bhoomaahospital.com
```

Paste:

```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/bhoomaahospital.com/html;
    index index.html;

    server_name bhoomaahospital.com www.bhoomaahospital.com;

    location / {
        try_files $uri $uri/ =404;
    }

    # Compression — takes the 148 KB HTML down to roughly 28 KB
    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml image/svg+xml application/manifest+json;

    # Cache images and icons for a year (filenames change when content changes)
    location ~* \.(jpg|jpeg|png|webp|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Never cache the HTML, so edits go live immediately
    location = /index.html {
        expires -1;
        add_header Cache-Control "no-cache, must-revalidate";
    }

    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
}
```

Enable it and remove the default site:

```bash
ln -s /etc/nginx/sites-available/bhoomaahospital.com /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

`nginx -t` must say "syntax is ok" and "test is successful" before you reload.

Visit `http://bhoomaahospital.com` — the site should load.

---

## Step 6 — HTTPS with Let's Encrypt (free)

```bash
apt install certbot python3-certbot-nginx -y
certbot --nginx -d bhoomaahospital.com -d www.bhoomaahospital.com
```

Answer the prompts:
- Email address — use a real one; expiry warnings go there
- Agree to terms: `Y`
- **Redirect HTTP to HTTPS: choose option `2` (Redirect)**

Certbot rewrites your Nginx config automatically and installs a renewal timer. Certificates last 90 days and renew on their own.

Confirm renewal works:

```bash
certbot renew --dry-run
```

---

## Step 7 — Pick one hostname

The site's canonical tag is `https://bhoomaahospital.com/` (no `www`). Make the server agree, or search engines will see two copies of the site.

Certbot will have created a `server` block listening on 443. Edit it:

```bash
nano /etc/nginx/sites-available/bhoomaahospital.com
```

Change the HTTPS block's `server_name` to only `bhoomaahospital.com`, then add this block at the end of the file:

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name www.bhoomaahospital.com;

    ssl_certificate     /etc/letsencrypt/live/bhoomaahospital.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/bhoomaahospital.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    return 301 https://bhoomaahospital.com$request_uri;
}
```

```bash
nginx -t && systemctl reload nginx
```

Test all four forms — every one should land on `https://bhoomaahospital.com`:

```
http://bhoomaahospital.com
http://www.bhoomaahospital.com
https://www.bhoomaahospital.com
https://bhoomaahospital.com
```

---

## Step 8 — Verify before telling Google

```bash
curl -I https://bhoomaahospital.com                      # expect 200
curl -I https://bhoomaahospital.com/robots.txt           # expect 200
curl -I https://bhoomaahospital.com/sitemap.xml          # expect 200
curl -sI https://bhoomaahospital.com | grep -i content-encoding   # expect gzip
```

Then in a browser:
- Images load, gallery and reviews behave normally
- Language switcher works
- Phone link dials, WhatsApp form opens with details pre-filled
- View source: canonical reads `https://bhoomaahospital.com/`, and there is **no** `noindex`

Then work through `LAUNCH-CHECKLIST.md` from step 8 onward (Rich Results Test, PageSpeed, Search Console, sitemap submission, Google Business Profile).

---

## Updating the site later

Re-upload the changed file and reload:

```powershell
scp index.html root@YOUR_DROPLET_IP:/var/www/bhoomaahospital.com/html/
```

HTML is set to never cache, so changes appear immediately. If you change an **image**, either give it a new filename or clear the browser cache — images are cached for a year.

---

## Notes

**Droplet size.** The smallest droplet is far more than this site needs. A static site of this weight will serve thousands of visitors a day on 1 GB RAM without effort.

**Backups.** DigitalOcean's weekly backups cost roughly 20% of the droplet price. Worth it, but your real backup is the `Bhooma Website` folder on your machine — the site can be redeployed from it in ten minutes.

**Alternative worth considering.** DigitalOcean App Platform hosts static sites on a free tier, with HTTPS, a global CDN and Git-based deploys — no server to patch or secure. If this droplet exists only for this site, App Platform is less work and likely faster for visitors. The droplet makes sense if you are already running other things on it.

**Security.** Run `apt update && apt upgrade -y` monthly. There is no database, no login and no server-side code here, so the attack surface is small — but an unpatched server is still a target.
