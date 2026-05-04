# Ubuntu Server Deployment Architecture
### Hosting Node.js Applications with Nginx, PM2, and SSL

---

## Overview

This guide covers the complete architecture for hosting multiple Node.js applications from GitHub on any Ubuntu server, exposed via a public IP and DNS record with HTTPS.

```
Internet
   │
   ▼
DNS Record (yourdomain.com → YOUR_SERVER_IP)
   │
   ▼
Ubuntu Server (PUBLIC IP)
   │
   ▼
Cloud Firewall (ports 22, 80, 443)
   │
   ▼
UFW - Linux Firewall (ports 22, 80, 443)
   │
   ▼
Nginx (Reverse Proxy) — port 80 / 443
   ├── app1.yourdomain.com → localhost:3000
   └── app2.yourdomain.com → localhost:3001
           │
           ▼
        PM2 (Process Manager)
           ├── App 1 (Node.js) — port 3000
           └── App 2 (Node.js) — port 3001
```

---

## 1. Initial Server Setup

### Connect and update
```bash
ssh root@YOUR_SERVER_IP

apt update && apt upgrade -y
apt install -y curl git ufw nginx certbot python3-certbot-nginx
```

### Create a non-root user (recommended)
```bash
adduser deploy
usermod -aG sudo deploy
su - deploy
```

---

## 2. Install Node.js and PM2

```bash
# Install Node.js via NVM (recommended)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc

nvm install --lts
nvm use --lts

# Verify
node -v
npm -v

# Install PM2 globally
npm install -g pm2
```

---

## 3. Configure UFW Firewall

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'    # opens ports 80 and 443
ufw enable

# Verify
ufw status verbose
```

Expected output:
```
To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW IN    Anywhere
Nginx Full                 ALLOW IN    Anywhere
```

> **Note:** If using a cloud provider (DigitalOcean, AWS, etc.), also open ports 22, 80, and 443 in the **cloud-level firewall** in the provider dashboard. Both the cloud firewall and UFW must allow the traffic.

---

## 4. Clone and Configure Applications

```bash
# Create app directories
mkdir -p /var/www/app1
mkdir -p /var/www/app2

# Clone from GitHub
git clone https://github.com/youruser/app1.git /var/www/app1
git clone https://github.com/youruser/app2.git /var/www/app2

# Install dependencies
cd /var/www/app1 && npm install
cd /var/www/app2 && npm install
```

### Environment variables
```bash
# Create .env files for each app
nano /var/www/app1/.env
nano /var/www/app2/.env
```

---

## 5. Start Applications with PM2

```bash
# Start apps
pm2 start /var/www/app1/server.js --name "app1" -- --port 3000
pm2 start /var/www/app2/server.js --name "app2" -- --port 3001

# Save process list so it survives reboots
pm2 save

# Generate and enable startup script
pm2 startup
# Copy and run the command it outputs, e.g:
# sudo env PATH=$PATH:/home/deploy/.nvm/versions/node/v20.x.x/bin pm2 startup systemd -u deploy --hp /home/deploy

# Verify both apps are running
pm2 status
pm2 logs
```

---

## 6. Configure DNS

At your DNS provider, add A records pointing to your server IP:

| Type | Name | Value | TTL |
|------|------|-------|-----|
| A | app1.yourdomain.com | YOUR_SERVER_IP | 300 |
| A | app2.yourdomain.com | YOUR_SERVER_IP | 300 |

Verify propagation:
```bash
dig app1.yourdomain.com +short   # should return YOUR_SERVER_IP
dig app2.yourdomain.com +short   # should return YOUR_SERVER_IP
```

---

## 7. Configure Nginx as Reverse Proxy

### App 1
```bash
nano /etc/nginx/sites-available/app1
```

```nginx
server {
    listen 80;
    server_name app1.yourdomain.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
    }

    location / {
        proxy_pass http://localhost:3000;
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

### App 2
```bash
nano /etc/nginx/sites-available/app2
```

```nginx
server {
    listen 80;
    server_name app2.yourdomain.com;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
    }

    location / {
        proxy_pass http://localhost:3001;
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

### Enable both sites
```bash
ln -s /etc/nginx/sites-available/app1 /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/app2 /etc/nginx/sites-enabled/

# Remove default site if present
rm -f /etc/nginx/sites-enabled/default

nginx -t && systemctl reload nginx
```

---

## 8. SSL Certificates with Certbot

### Option A — HTTP challenge (standard, requires port 80 reachable)
```bash
certbot --nginx -d app1.yourdomain.com
certbot --nginx -d app2.yourdomain.com
```

### Option B — DNS challenge (use when port 80 is blocked)
```bash
certbot certonly --manual --preferred-challenges dns -d app1.yourdomain.com
```

Certbot will ask you to add a TXT record:
```
_acme-challenge.app1.yourdomain.com  →  <value certbot gives you>
```

Add it in your DNS provider, wait for propagation, then press Enter.

After obtaining the certificate manually, add SSL to your Nginx config:

```nginx
server {
    listen 80;
    server_name app1.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app1.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/app1.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app1.yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
    }

    location / {
        proxy_pass http://localhost:3000;
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

```bash
nginx -t && systemctl reload nginx
```

### Auto-renewal
```bash
# Test renewal
certbot renew --dry-run

# Certbot installs a systemd timer automatically — verify it
systemctl status certbot.timer
```

---

## 9. Deploying Updates from GitHub

Create a simple deploy script for each app:

```bash
nano /var/www/app1/deploy.sh
```

```bash
#!/bin/bash
set -e

echo "Deploying App 1..."
cd /var/www/app1

git pull origin main
npm install --production
pm2 restart app1

echo "Done."
```

```bash
chmod +x /var/www/app1/deploy.sh

# Deploy anytime with:
bash /var/www/app1/deploy.sh
```

---

## 10. Useful Commands Reference

### PM2
```bash
pm2 status              # list all processes
pm2 logs app1           # tail logs for app1
pm2 restart app1        # restart app
pm2 reload app1         # zero-downtime reload
pm2 stop app1           # stop app
pm2 delete app1         # remove from PM2
pm2 monit               # live dashboard
```

### Nginx
```bash
nginx -t                          # test config
systemctl reload nginx            # reload without downtime
systemctl restart nginx           # full restart
tail -f /var/log/nginx/error.log  # watch error log
tail -f /var/log/nginx/access.log # watch access log
```

### Certbot
```bash
certbot certificates              # list all certs and expiry
certbot renew                     # renew all due certs
certbot renew --dry-run           # test renewal without changes
```

### UFW
```bash
ufw status verbose                # show all rules
ufw allow 8080/tcp                # open a port
ufw delete allow 8080/tcp         # close a port
```

---

## 11. Troubleshooting Checklist

| Symptom | Check |
|---------|-------|
| Connection timeout externally | Cloud provider firewall — ports 80/443 open and attached to droplet |
| Connection refused locally | `ss -tlnp \| grep ':80'` — is Nginx listening? |
| Nginx not listening | `systemctl status nginx` and `nginx -t` — config error? |
| App not responding | `pm2 status` — is the process running? `pm2 logs` for errors |
| Certbot fails (timeout) | Port 80 not reachable externally — use DNS challenge instead |
| SSL cert expired | `certbot renew` — check `systemctl status certbot.timer` |
| Changes not live after git pull | Run `pm2 restart appname` after deploying |

---

## Architecture Summary

```
GitHub Repo
     │
     │  git pull (manual or CI/CD)
     ▼
/var/www/appN  (source code)
     │
     ▼
PM2  (keeps Node.js process alive, auto-restarts on crash/reboot)
     │  localhost:3000 / localhost:3001
     ▼
Nginx  (reverse proxy, SSL termination, HTTP→HTTPS redirect)
     │  ports 80 / 443
     ▼
UFW  (OS-level firewall)
     │
     ▼
Cloud Firewall  (provider-level, e.g. DigitalOcean)
     │
     ▼
Public IP  →  DNS A Record  →  yourdomain.com
```

> **Key principle:** Traffic must pass through both the cloud firewall and UFW before reaching Nginx. If either blocks a port, the app is unreachable — even if everything else is correctly configured.
