# FastAPI Deployment Guide on VPS — Complete Edition

> Covers every step from a blank VPS to a live, secured, production-ready FastAPI app.
> Replace `YOUR_VPS_IP`, `api.yourdomain.com`, `deploy`, and `myfastapi` with your own values throughout.

---

## Architecture Overview

```
Internet (HTTPS: 443)
        │
   ┌────▼────────────────────┐
   │   Nginx (Reverse Proxy) │  ← public-facing, handles SSL
   │   api.yourdomain.com    │
   └────┬────────────────────┘
        │ localhost:8000
   ┌────▼────────────────────┐
   │   Uvicorn (ASGI Server) │  ← managed by systemd
   └────┬────────────────────┘
        │
   ┌────▼────────────────────┐
   │   FastAPI Application   │
   │  /var/www/myfastapi/app │
   └─────────────────────────┘
```

---

## What You Need Before Starting

| Item | Example |
|------|---------|
| VPS IP address | `123.123.123.123` |
| Domain or subdomain | `api.yourdomain.com` |
| Your FastAPI repo URL | `https://github.com/you/repo.git` |
| SSH client on your machine | Terminal / PuTTY |

---

## Phase 1 — Initial Server Access

### Step 1 — Login as root

```bash
ssh root@YOUR_VPS_IP
```

---

### Step 2 — Check Python version

Confirm Python 3.8 or newer is available. FastAPI requires at least 3.8.

```bash
python3 --version
```

If the version is below 3.8, install a newer one:

```bash
sudo apt install -y python3.11 python3.11-venv python3.11-pip
```

---

### Step 3 — Check available disk and RAM

Low disk or RAM causes silent failures during pip installs or app startup.

```bash
df -h        # disk usage
free -h      # RAM and swap
```

---

### Step 4 — Add swap space (if RAM is 1 GB or less)

Many cheap VPS plans have 512 MB–1 GB RAM. Without swap, pip installs and Python processes can be killed by the OOM killer.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Make it permanent across reboots:

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify:

```bash
free -h
# Swap row should now show 2.0G
```

---

### Step 5 — Set the server timezone

Logs are much easier to read when timestamps match your local time or UTC consistently.

```bash
timedatectl list-timezones | grep Asia    # or grep Europe, America, etc.
sudo timedatectl set-timezone Asia/Dhaka  # change to your timezone
timedatectl status
```

---

### Step 6 — Update the server

```bash
apt update && apt upgrade -y
apt autoremove -y
```

---

### Step 7 — Enable automatic security updates

This installs security patches automatically so you don't have to log in just for that.

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Choose "Yes" when prompted
```

---

## Phase 2 — Users and SSH Security

### Step 8 — Create a deploy user

```bash
adduser deploy
```

Choose a strong password when prompted.

---

### Step 9 — Grant sudo privileges

```bash
usermod -aG sudo deploy
```

---

### Step 10 — Add deploy user to www-data group

Nginx runs as `www-data`. Adding deploy to this group prevents permission errors when Nginx reads app files.

```bash
sudo usermod -aG www-data deploy
```

---

### Step 11 — Verify the new user

Switch to deploy and confirm sudo works:

```bash
su - deploy
sudo whoami
# Expected output: root
exit
```

---

### Step 12 — Set up SSH key login

**On your local machine** (not the server):

```bash
# Generate a key if you don't have one
ssh-keygen -t ed25519 -C "your@email.com"

# Copy public key to the server
ssh-copy-id deploy@YOUR_VPS_IP
```

Test key login:

```bash
ssh deploy@YOUR_VPS_IP
```

---

### Step 13 — Disable root SSH and password login

```bash
sudo nano /etc/ssh/sshd_config
```

Find or add these exact lines:

```
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
```

**Test the config before reloading** — a syntax error here locks you out:

```bash
sudo sshd -t
# No output = no errors
```

Reload:

```bash
sudo systemctl reload ssh
```

**Open a new terminal and confirm you can still log in as deploy before closing the current session.**

---

### Step 14 — Lock root password

```bash
sudo passwd -l root
```

---

### Step 15 — Install fail2ban

fail2ban monitors SSH logs and bans IPs that repeatedly fail login attempts.

```bash
sudo apt install -y fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check it is running:

```bash
sudo systemctl status fail2ban
```

---

## Phase 3 — Firewall

### Step 16 — Configure ufw

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

Expected output:

```
To                Action      From
--                ------      ----
OpenSSH           ALLOW       Anywhere
80/tcp            ALLOW       Anywhere
443/tcp           ALLOW       Anywhere
```

**Do NOT open port 8000.** Uvicorn listens on `127.0.0.1:8000` — local only. Only Nginx should talk to it.

---

### Step 17 — Check your VPS provider's firewall

Most cloud providers (DigitalOcean, AWS, Hetzner, Linode, Vultr) have a **separate firewall panel** in their web dashboard, independent of ufw. If this is not configured, ufw rules alone may not be enough.

Log in to your provider dashboard and confirm:
- Port 22 (SSH) is open
- Port 80 (HTTP) is open
- Port 443 (HTTPS) is open
- Port 8000 is **not** open publicly

---

## Phase 4 — DNS Configuration

### Step 18 — Point your domain to the VPS

Log in to your domain registrar (Namecheap, GoDaddy, Cloudflare, etc.) and add an **A record**:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | api | YOUR_VPS_IP | 300 |

If your full domain is `api.yourdomain.com`, the Name field is `api`.  
If deploying on the root domain, the Name field is `@`.

---

### Step 19 — Verify DNS propagation

Wait 5–15 minutes for DNS to propagate, then verify:

```bash
# On your local machine or the server
nslookup api.yourdomain.com
# or
dig api.yourdomain.com +short
```

The IP returned must match `YOUR_VPS_IP` before you proceed to SSL. Certbot will fail if DNS is not resolved correctly.

---

## Phase 5 — Install Required Tools

### Step 20 — Install all dependencies

```bash
sudo apt install -y git curl nginx python3 python3-pip python3-venv build-essential
```

| Tool | Purpose |
|------|---------|
| `git` | Pull code from GitHub |
| `nginx` | Reverse proxy |
| `python3-venv` | Isolated Python environment |
| `build-essential` | C compiler needed by some Python packages |

---

## Phase 6 — Application Setup

### Step 21 — Create app directory structure

```bash
sudo mkdir -p /var/www/myfastapi
sudo chown -R deploy:deploy /var/www/myfastapi
mkdir -p /var/www/myfastapi/app
mkdir -p /var/www/myfastapi/shared
mkdir -p /var/www/myfastapi/logs
```

Final structure:

```
/var/www/myfastapi/
├── app/        ← your cloned code
├── shared/     ← .env and other shared config
├── logs/       ← app log files
└── venv/       ← Python virtual environment (created next)
```

---

### Step 22 — Clone your project

```bash
cd /var/www/myfastapi
git clone https://github.com/YOUR_USER/YOUR_REPO.git app
cd app
```

---

### Step 23 — Ensure .env is in .gitignore

If it is not already, make sure your `.env` file is excluded from git. Run this inside the app folder:

```bash
grep -q "\.env" .gitignore && echo "Already ignored" || echo ".env" >> .gitignore
```

---

### Step 24 — Create the virtual environment

```bash
cd /var/www/myfastapi
python3 -m venv venv
source /var/www/myfastapi/venv/bin/activate
```

Your prompt will change to show `(venv)` — this confirms it is active.

---

### Step 25 — Install Python packages

```bash
python -m pip install --upgrade pip setuptools wheel
cd /var/www/myfastapi/app
pip install -r requirements.txt
pip install fastapi "uvicorn[standard]" gunicorn
```

`uvicorn[standard]` includes `uvloop` and `httptools` for better performance.  
`gunicorn` is used in the systemd service as a process manager.

---

### Step 26 — Create the .env file

```bash
nano /var/www/myfastapi/shared/.env
```

Paste your environment variables:

```env
APP_ENV=production
DEBUG=false
PORT=8000
SECRET_KEY=replace_with_a_long_random_string
DATABASE_URL=postgresql://user:pass@localhost/dbname
```

Save with `Ctrl+X`, `Y`, `Enter`.

Lock file permissions so only the deploy user can read it:

```bash
chmod 600 /var/www/myfastapi/shared/.env
```

---

### Step 27 — Identify your FastAPI app entry point

You need to know the `module:variable` path for Uvicorn/Gunicorn.

| File location | Entry point string |
|--------------|-------------------|
| `/var/www/myfastapi/app/main.py` | `main:app` |
| `/var/www/myfastapi/app/app/main.py` | `app.main:app` |
| `/var/www/myfastapi/app/src/main.py` | `src.main:app` |

Check your file:

```bash
grep -r "FastAPI()" /var/www/myfastapi/app --include="*.py" -l
```

---

### Step 28 — Test manually before automating

```bash
cd /var/www/myfastapi/app
source /var/www/myfastapi/venv/bin/activate
uvicorn main:app --host 127.0.0.1 --port 8000
```

In a second terminal, test it:

```bash
curl http://127.0.0.1:8000
curl http://127.0.0.1:8000/docs
```

If it responds correctly, stop it:

```bash
Ctrl + C
```

Deactivate the virtual environment:

```bash
deactivate
```

---

### Step 29 — Verify port 8000 is listening on localhost only

```bash
ss -tlnp | grep 8000
# Should show 127.0.0.1:8000 — NOT 0.0.0.0:8000
```

If it shows `0.0.0.0:8000`, Uvicorn is publicly exposed. Fix this by always using `--host 127.0.0.1`.

---

## Phase 7 — systemd Service

### Step 30 — Create the systemd service file

```bash
sudo nano /etc/systemd/system/myfastapi.service
```

Paste the following:

```ini
[Unit]
Description=FastAPI app via Gunicorn + Uvicorn workers
After=network.target

[Service]
User=deploy
Group=www-data
WorkingDirectory=/var/www/myfastapi/app
EnvironmentFile=/var/www/myfastapi/shared/.env
ExecStart=/var/www/myfastapi/venv/bin/gunicorn \
    main:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 127.0.0.1:8000 \
    --access-logfile /var/www/myfastapi/logs/access.log \
    --error-logfile /var/www/myfastapi/logs/error.log
ExecReload=/bin/kill -s HUP $MAINPID
Restart=always
RestartSec=5
KillMode=mixed
TimeoutStopSec=5

[Install]
WantedBy=multi-user.target
```

**Note on workers:** `--workers 4` is a good starting point. The common formula is `(2 × CPU cores) + 1`. Check your CPU count with `nproc`.

**Why Gunicorn instead of plain Uvicorn?**  
Gunicorn manages multiple worker processes, handles graceful restarts (`ExecReload`), and is more battle-tested for production traffic than running Uvicorn directly.

---

### Step 31 — Reload, enable, and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable myfastapi
sudo systemctl start myfastapi
```

---

### Step 32 — Verify the service is running

```bash
sudo systemctl status myfastapi
```

Check that it is enabled for auto-start on boot:

```bash
sudo systemctl is-enabled myfastapi
# Expected: enabled
```

Check it is currently active:

```bash
sudo systemctl is-active myfastapi
# Expected: active
```

---

### Step 33 — Check service logs

```bash
# Last 50 lines
journalctl -u myfastapi -n 50 --no-pager

# Live stream
journalctl -u myfastapi -f
```

Also check the app log files directly:

```bash
tail -f /var/www/myfastapi/logs/error.log
tail -f /var/www/myfastapi/logs/access.log
```

---

### Step 34 — Confirm app is reachable locally

```bash
curl http://127.0.0.1:8000
```

This should return a valid response from your FastAPI app before you set up Nginx.

---

## Phase 8 — Nginx Configuration

### Step 35 — Remove the default Nginx site

The default Nginx site listens on port 80 and **will conflict** with your config if not removed.

```bash
sudo rm /etc/nginx/sites-enabled/default
```

Verify it is gone:

```bash
ls /etc/nginx/sites-enabled/
# Should not list "default"
```

---

### Step 36 — Create your Nginx server block

```bash
sudo nano /etc/nginx/sites-available/myfastapi
```

Paste the following. **Replace `api.yourdomain.com` with your real domain.**

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name api.yourdomain.com;

    client_max_body_size 20M;
    client_body_timeout 30s;
    client_header_timeout 30s;

    # Logging
    access_log /var/log/nginx/myfastapi_access.log;
    error_log  /var/log/nginx/myfastapi_error.log;

    location / {
        proxy_pass         http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection "upgrade";
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_redirect     off;
        proxy_read_timeout 60s;
        proxy_connect_timeout 10s;
    }
}
```

`proxy_http_version 1.1` and the `Upgrade`/`Connection` headers are required for WebSocket support. Safe to include even if you don't use WebSockets.

---

### Step 37 — Enable the site with a symlink

```bash
sudo ln -s /etc/nginx/sites-available/myfastapi /etc/nginx/sites-enabled/myfastapi
```

Verify the symlink was created correctly:

```bash
ls -la /etc/nginx/sites-enabled/
# Should show: myfastapi -> /etc/nginx/sites-available/myfastapi
```

---

### Step 38 — Test the Nginx config

Always test before reloading — a bad config will crash Nginx:

```bash
sudo nginx -t
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

---

### Step 39 — Reload Nginx

```bash
sudo systemctl reload nginx
```

---

### Step 40 — Test HTTP (before SSL)

Confirm Nginx is forwarding traffic to your app over plain HTTP first:

```bash
curl -I http://api.yourdomain.com
```

Expected: `HTTP/1.1 200 OK` (or whichever status your root route returns).  
If you get a 502, the app is not running. Check `sudo systemctl status myfastapi`.  
If you get a 404 from Nginx, your server block config has an error.

---

## Phase 9 — SSL Certificate

### Step 41 — Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

---

### Step 42 — Verify DNS resolves before running Certbot

Certbot performs an HTTP challenge — your domain must point to this server or it will fail.

```bash
curl -I http://api.yourdomain.com
# Must return a response from YOUR server, not a default page
```

---

### Step 43 — Obtain the SSL certificate

```bash
sudo certbot --nginx -d api.yourdomain.com
```

When prompted:
- Enter your email address (for renewal notifications)
- Agree to the terms
- Choose whether to share your email with EFF (optional)
- **Select option 2: Redirect HTTP to HTTPS** (recommended)

Certbot will:
1. Verify domain ownership
2. Download the certificate
3. Automatically update your Nginx config to add HTTPS

---

### Step 44 — Verify the updated Nginx config

Certbot modifies your Nginx config automatically. Review what it added:

```bash
sudo cat /etc/nginx/sites-available/myfastapi
```

You should now see a `server` block listening on port 443 with `ssl_certificate` lines, and the original port 80 block redirecting to HTTPS.

---

### Step 45 — Add security headers to Nginx

Open your Nginx config and add these headers inside the port 443 `server` block:

```bash
sudo nano /etc/nginx/sites-available/myfastapi
```

Add inside the `server { listen 443 ... }` block, before the `location /` block:

```nginx
# Security headers
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

Test and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

### Step 46 — Test SSL auto-renewal

Certbot installs a systemd timer for automatic renewal. Verify it exists and is active:

```bash
sudo systemctl status certbot.timer
```

Run a dry run to confirm renewal will work when the time comes:

```bash
sudo certbot renew --dry-run
```

Expected output ends with: `Congratulations, all simulated renewals succeeded.`

---

## Phase 10 — Log Rotation

### Step 47 — Set up log rotation for app logs

Without log rotation, your log files grow forever and will fill the disk.

```bash
sudo nano /etc/logrotate.d/myfastapi
```

Paste:

```
/var/www/myfastapi/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 deploy www-data
    sharedscripts
    postrotate
        systemctl kill -s USR1 myfastapi
    endscript
}
```

Test the logrotate config:

```bash
sudo logrotate --debug /etc/logrotate.d/myfastapi
```

---

## Phase 11 — Final Verification

### Step 48 — Full end-to-end test

```bash
# Test HTTPS response
curl https://api.yourdomain.com

# Test headers (check for HSTS, SSL, security headers)
curl -I https://api.yourdomain.com

# Test HTTP redirects to HTTPS
curl -I http://api.yourdomain.com
# Should return: 301 Moved Permanently → https://...
```

---

### Step 49 — Check all service statuses

```bash
sudo systemctl status myfastapi
sudo systemctl status nginx
sudo systemctl status fail2ban
sudo systemctl status certbot.timer
```

All should show `active (running)` or `active (waiting)` for the timer.

---

### Step 50 — Check Nginx logs for errors

```bash
sudo tail -n 50 /var/log/nginx/myfastapi_error.log
sudo tail -n 20 /var/log/nginx/myfastapi_access.log
```

---

## Phase 12 — Deployment Workflow (Updating Your App)

### How to deploy a new version

Each time you push changes to GitHub, run this on the server:

```bash
ssh deploy@YOUR_VPS_IP
cd /var/www/myfastapi/app
git pull origin main
source /var/www/myfastapi/venv/bin/activate
pip install -r requirements.txt
deactivate
sudo systemctl restart myfastapi
sudo systemctl status myfastapi
```

Nginx does **not** need to restart — only the app service.

---

### How to roll back to a previous version

If a deploy breaks something:

```bash
cd /var/www/myfastapi/app
git log --oneline -10       # find the last good commit hash
git checkout <commit-hash>
sudo systemctl restart myfastapi
```

---

## Quick Reference — Common Commands

### App management

| Action | Command |
|--------|---------|
| Start app | `sudo systemctl start myfastapi` |
| Stop app | `sudo systemctl stop myfastapi` |
| Restart app | `sudo systemctl restart myfastapi` |
| Reload without downtime | `sudo systemctl reload myfastapi` |
| Check status | `sudo systemctl status myfastapi` |
| Live logs | `journalctl -u myfastapi -f` |
| Last 100 log lines | `journalctl -u myfastapi -n 100 --no-pager` |

### Nginx management

| Action | Command |
|--------|---------|
| Test config | `sudo nginx -t` |
| Reload config | `sudo systemctl reload nginx` |
| Restart Nginx | `sudo systemctl restart nginx` |
| View error log | `sudo tail -f /var/log/nginx/myfastapi_error.log` |
| List enabled sites | `ls -la /etc/nginx/sites-enabled/` |

### SSL management

| Action | Command |
|--------|---------|
| Check cert expiry | `sudo certbot certificates` |
| Test renewal | `sudo certbot renew --dry-run` |
| Force renew | `sudo certbot renew --force-renewal` |
| Check renewal timer | `sudo systemctl status certbot.timer` |

### Firewall

| Action | Command |
|--------|---------|
| View rules | `sudo ufw status verbose` |
| Add a rule | `sudo ufw allow <port>/tcp` |
| Remove a rule | `sudo ufw delete allow <port>/tcp` |

---

## Common Problems and Fixes

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `502 Bad Gateway` from Nginx | App is not running | `sudo systemctl restart myfastapi` |
| `curl: (7) Failed to connect` on port 8000 | App binding to wrong host | Ensure `--bind 127.0.0.1:8000` in service file |
| Certbot fails domain validation | DNS not propagated yet | Wait and re-run `dig api.yourdomain.com` |
| Nginx config test fails | Syntax error after editing | Check for missing semicolons or braces |
| App starts but returns 500 | Missing env variable | Check `/var/www/myfastapi/shared/.env` |
| SSH locked out | sshd_config error | Use VPS provider's web console to fix |
| Port 8000 publicly accessible | Firewall misconfigured | `sudo ufw delete allow 8000` and check provider dashboard |
| `Permission denied` on log files | Wrong ownership | `sudo chown -R deploy:www-data /var/www/myfastapi/logs` |
| pip install killed mid-way | Out of RAM | Add swap (see Step 4) |
| Old Nginx default page showing | Default site still enabled | `sudo rm /etc/nginx/sites-enabled/default` then reload |

---

## What Was Missing from Common Guides

This guide adds the following steps that are routinely skipped but cause real production problems:

1. **Swap space** — pip kills itself on low-RAM VPS without it
2. **Timezone setup** — makes logs readable
3. **VPS provider firewall** — separate from ufw, often overlooked
4. **DNS verification before SSL** — Certbot fails silently if DNS isn't ready
5. **Remove default Nginx site** — causes port 80 conflict
6. **Symlink verification** — confirming `sites-enabled` actually points to `sites-available`
7. **`www-data` group for deploy user** — prevents Nginx permission errors
8. **Gunicorn as process manager** — more robust than raw Uvicorn in production
9. **Multiple Uvicorn workers** — single worker cannot use multiple CPU cores
10. **WebSocket headers in Nginx** — required if your app uses WebSockets or SSE
11. **Security headers** — HSTS, X-Frame-Options, etc.
12. **fail2ban** — blocks SSH brute-force attempts
13. **Certbot timer verification** — confirming auto-renewal is actually scheduled
14. **Log rotation** — prevents disk from filling up
15. **Port 8000 exposure check** — ensuring Uvicorn is NOT publicly accessible
16. **`is-enabled` and `is-active` checks** — confirm systemd setup is correct
17. **Deactivate venv** after manual testing
18. **Rollback procedure** — git checkout to last known good commit
19. **Unattended upgrades** — automatic security patches
20. **`.gitignore` check for .env** — prevent secret leaks

---

*Replace all placeholder values before running any command. Test each phase before moving to the next.*
