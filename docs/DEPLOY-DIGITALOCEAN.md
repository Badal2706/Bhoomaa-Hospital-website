# Deploying to a DigitalOcean Droplet (via GitHub)

Static site — HTML, CSS, images. No build step, no runtime, no database. Nginx serves it directly.

**This droplet already runs other services.** Every step below is additive. Nothing here removes or edits an existing site's configuration. Read the warnings; a careless `rm` in `/etc/nginx/sites-enabled/` will take your SaaS products offline.

- Droplet IP: `167.71.236.138`
- Repo: `https://github.com/Badal2706/Bhooma-Hospital-website.git`
- Domain: `bhoomaahospital.com` (replace throughout if different)

---

## Step 1 — Push to GitHub (from Windows)

The repository is already initialised, committed and pointed at your remote. From the `Bhooma Website` folder in PowerShell:

```powershell
git push -u origin main
```

Git will prompt for credentials. Use your GitHub **username** and a **personal access token** as the password (GitHub stopped accepting account passwords in 2021).

To create a token: GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token → select only this repository → Repository permissions → **Contents: Read and write**. Nothing else.

If Git doesn't prompt, run `git config --global credential.helper manager` first.

Verify at `https://github.com/Badal2706/Bhooma-Hospital-website` — you should see `public/`, `docs/`, `README.md`. You should **not** see `source-photos/`.

---

## Step 2 — Survey the droplet before touching anything

```bash
ssh root@167.71.236.138
```

```bash
# What is already configured and enabled?
ls -la /etc/nginx/sites-enabled/

# Which domains are already claimed?
grep -r "server_name" /etc/nginx/sites-enabled/ /etc/nginx/conf.d/ 2>/dev/null

# What is listening on 80/443?
ss -tlnp | grep -E ':80|:443'

# Is it Nginx, or Apache/Caddy/a Docker reverse proxy?
systemctl status nginx --no-pager | head -5
docker ps 2>/dev/null | head
```

**Stop and reassess if:**
- Nothing is listening on 80/443 via Nginx — you may be using Apache, Caddy or a Docker reverse proxy (Traefik, nginx-proxy). In that case the config below does not apply; the site must be added to whatever is already terminating traffic.
- A server block already claims `bhoomaahospital.com`.

Back up the config before proceeding:

```bash
cp -r /etc/nginx /root/nginx-backup-$(date +%F)
```

---

## Step 3 — DNS

In **GoDaddy → My Products → Domain → DNS → Manage Zones**:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | `@` | `167.71.236.138` | 600 |
| A | `www` | `167.71.236.138` | 600 |

Delete existing parking/forwarding records for `@` and `www` first — they will conflict.

Keep TTL at **600 seconds**. A low TTL makes the future droplet migration near-instant. Raise it to 3600 only after everything is stable.

Wait for propagation, then confirm:

```bash
dig +short bhoomaahospital.com
```

Do not attempt SSL until this returns `167.71.236.138`. Certificate issuance will fail otherwise.

---

## Step 4 — Clone the repository

```bash
mkdir -p /var/www
cd /var/www
git clone https://github.com/Badal2706/Bhooma-Hospital-website.git bhoomaahospital
```

If the repo is **private**, use a deploy key instead of pasting a token onto the server:

```bash
ssh-keygen -t ed25519 -C "droplet-bhoomaa" -f ~/.ssh/bhoomaa_deploy -N ""
cat ~/.ssh/bhoomaa_deploy.pub
```

Add that public key at GitHub → repo → Settings → Deploy keys → Add deploy key (leave "Allow write access" **unchecked**). Then:

```bash
cat >> ~/.ssh/config <<'EOF'
Host github-bhoomaa
  HostName github.com
  User git
  IdentityFile ~/.ssh/bhoomaa_deploy
EOF
cd /var/www
git clone git@github-bhoomaa:Badal2706/Bhooma-Hospital-website.git bhoomaahospital
```

Set ownership for Nginx:

```bash
chown -R www-data:www-data /var/www/bhoomaahospital
chmod -R 755 /var/www/bhoomaahospital
```

---

## Step 5 — Add an Nginx server block

This creates a **new** file. It does not touch existing sites.

```bash
nano /etc/nginx/sites-available/bhoomaahospital.com
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name bhoomaahospital.com www.bhoomaahospital.com;

    # Note: the repo root is NOT the web root — public/ is.
    root /var/www/bhoomaahospital/public;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    gzip on;
    gzip_vary on;
    gzip_min_length 256;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml image/svg+xml application/manifest+json;

    # Images cached for a year — change the filename when you change an image
    location ~* \.(jpg|jpeg|png|webp|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # HTML never cached, so edits appear immediately
    location = /index.html {
        expires -1;
        add_header Cache-Control "no-cache, must-revalidate";
    }

    # Block access to anything git-related, just in case
    location ~ /\.git { deny all; return 404; }

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    access_log /var/log/nginx/bhoomaahospital.access.log;
    error_log  /var/log/nginx/bhoomaahospital.error.log;
}
```

Enable it — **do not delete anything else in `sites-enabled/`**:

```bash
ln -s /etc/nginx/sites-available/bhoomaahospital.com /etc/nginx/sites-enabled/
nginx -t
```

`nginx -t` must report "syntax is ok" and "test is successful". If it errors, fix it before reloading — reloading a broken config will not drop your existing sites, but do not risk it.

```bash
systemctl reload nginx
```

`reload` is graceful and will not interrupt your other services. Never use `restart` on a shared droplet unless you have to.

Check `http://bhoomaahospital.com`, then confirm your other sites still respond.

---

## Step 6 — HTTPS

```bash
apt install certbot python3-certbot-nginx -y
certbot --nginx -d bhoomaahospital.com -d www.bhoomaahospital.com
```

Certbot only modifies the server block matching the domains you pass with `-d`. Your other sites are untouched.

Choose **`2` (Redirect)** when asked about HTTP → HTTPS.

```bash
certbot renew --dry-run
```

If your other sites already use Certbot, this simply adds another certificate to the existing renewal timer.

---

## Step 7 — One canonical hostname

The site's canonical tag is `https://bhoomaahospital.com/` (no `www`). The server must agree, or Google sees two copies of the site.

Edit `/etc/nginx/sites-available/bhoomaahospital.com`. In the HTTPS block Certbot created, change `server_name` to only `bhoomaahospital.com`. Then append:

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

All four forms must land on `https://bhoomaahospital.com`:

```bash
for u in http://bhoomaahospital.com http://www.bhoomaahospital.com https://www.bhoomaahospital.com; do
  echo "$u -> $(curl -sIL -o /dev/null -w '%{url_effective}' $u)"
done
```

---

## Step 8 — Verify

```bash
curl -I https://bhoomaahospital.com                    # 200
curl -I https://bhoomaahospital.com/robots.txt         # 200
curl -I https://bhoomaahospital.com/sitemap.xml        # 200
curl -sI https://bhoomaahospital.com | grep -i content-encoding   # gzip
curl -I https://bhoomaahospital.com/../README.md       # must NOT be 200
```

In a browser: images load, gallery and review strip behave, language switcher works, phone dials, WhatsApp form pre-fills. View source — canonical is correct and there is no `noindex`.

Then continue with `LAUNCH-CHECKLIST.md` from step 8.

---

## Updating the site

Locally: edit `public/index.html`, then

```powershell
git add -A
git commit -m "describe the change"
git push
```

On the droplet:

```bash
cd /var/www/bhoomaahospital && git pull
```

That is the whole deploy. No Nginx reload needed — Nginx reads files from disk on each request.

### Optional: one-command deploy

```bash
cat > /usr/local/bin/deploy-bhoomaa <<'EOF'
#!/bin/bash
set -e
cd /var/www/bhoomaahospital
git pull
chown -R www-data:www-data /var/www/bhoomaahospital
echo "Deployed: $(git log -1 --pretty=%s)"
EOF
chmod +x /usr/local/bin/deploy-bhoomaa
```

Then just `deploy-bhoomaa`.

### Optional: automatic deploy on push

GitHub Actions can SSH in and pull on every push to `main`. Worth setting up once the site is stable and being edited often; unnecessary while changes are occasional.

---

---

## As deployed (2026-09-02)

The site is live at <https://bhoomaahospital.com>. The exact Nginx server block
in production is committed at [`nginx/bhoomaahospital.com.conf`](nginx/bhoomaahospital.com.conf)
— copy it verbatim onto a new droplet rather than rebuilding it from step 5.

Notes specific to droplet `167.71.236.138`:

- The droplet is IPv4-only, so there are **no `listen [::]:…` directives**. Adding
  them will fail to bind.
- Nothing listened on port 80 before this site existed, so the config includes a
  `listen 80 default_server` catch-all returning `444`. Without it the hospital
  site would answer plain-HTTP requests to the bare IP.
- Certbot's `--redirect` sends `http://www` through `https://www` before reaching
  the canonical host. The committed config hardcodes the bare domain instead, so
  every entry point is one hop.
- `git config --global --add safe.directory /var/www/bhoomaahospital` is required,
  because the working tree is owned by `www-data` but `git pull` runs as root.
- Existing services left untouched: `cylinderpro.guruindustries.co.in` (static
  root `/var/www/cylinderpro` + Node/PM2 backend proxied from `localhost:3001`).

Config backups on the droplet: `/root/nginx-backup-2026-09-02-1738` (whole
`/etc/nginx` before any change) and `/root/bhoomaahospital.com.certbot-original`
(Certbot's unmodified output, before the canonical redirect was applied).

# Moving to a different droplet later

**Short answer: no SEO impact, provided the domain and URLs stay the same.**

Google indexes and ranks the **domain**, not the server IP. It has no concept of "which droplet". Your rankings, backlinks, Search Console history and Google Business Profile all attach to `bhoomaahospital.com`. Changing the A record is invisible to Google as long as the site keeps responding.

You do **not** need Search Console's Change of Address tool — that is only for moving to a different *domain*.

The genuine risks are operational, not algorithmic:

| Risk | Prevention |
|------|-----------|
| Downtime during the switch | Build the new server fully and test it **before** touching DNS |
| SSL certificate missing on the new server | Run Certbot on the new droplet **after** DNS points there; expect a few minutes of browser warnings otherwise |
| Slow DNS propagation | Keep TTL at 600s. Lower to 300s a day before migrating |
| Redirects lost | Re-apply the `www` → bare-domain redirect on the new server |
| Old server still answering | Keep it running 48h, then shut down |

### Migration procedure

1. **A day before:** lower both A records' TTL to 300 in GoDaddy.
2. **On the new droplet:** repeat steps 2, 4, 5 of this guide. Do not run Certbot yet.
3. **Test before switching.** On your Windows machine, edit `C:\Windows\System32\drivers\etc\hosts` (as Administrator) and add:
   ```
   NEW_DROPLET_IP    bhoomaahospital.com
   ```
   Browse the site — you are now hitting the new server while the rest of the world still hits the old one. Confirm everything works, then remove the line.
4. **Switch DNS:** update both A records to the new IP.
5. **Wait** for `dig +short bhoomaahospital.com` to return the new IP (usually minutes at TTL 300).
6. **Issue SSL on the new droplet:** `certbot --nginx -d bhoomaahospital.com -d www.bhoomaahospital.com`
7. **Re-apply** the `www` redirect (step 7).
8. **Verify** with the step 8 checks.
9. **Leave the old droplet running for 48 hours** — some DNS resolvers ignore TTL. Then decommission.
10. **Afterwards:** in Search Console, run a URL Inspection → Test Live URL to confirm Googlebot reaches the new server. Raise TTL back to 3600.

Because the site is in Git, step 2 is a `git clone` — the new server is identical to the old one by construction. That is the main practical reason to deploy this way rather than uploading files by hand.

**One caveat:** if you keep the site on the shared droplet long-term, remember that a problem with your SaaS products — a crash, a runaway process, an Nginx misconfiguration — takes the hospital site down with it. For a business where people search "emergency hospital Palanpur", that matters more than it would for most sites. Moving it to its own droplet, or to DigitalOcean App Platform's free static tier, removes that coupling.
