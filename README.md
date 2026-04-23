# VPS Getting Started: Server Setup + Cloudflare + SSL

A step-by-step guide to spinning up a VPS, connecting your domain through Cloudflare, and getting HTTPS working with a real SSL certificate.

> **Screenshots:** Create an `images/vps/` folder and save screenshots using the filenames referenced below.

---

## Table of Contents

1. [What is a VPS](#what-is-a-vps)
2. [Choose a VPS Provider](#choose-a-vps-provider)
3. [Create Your Server](#create-your-server)
4. [Connect to Your Server](#connect-to-your-server)
5. [Initial Server Setup](#initial-server-setup)
6. [Install a Web Server](#install-a-web-server)
7. [Point Your Domain with Cloudflare](#point-your-domain-with-cloudflare)
8. [Install SSL with Let's Encrypt](#install-ssl-with-lets-encrypt)
9. [Set Cloudflare SSL to Full (Strict)](#set-cloudflare-ssl-to-full-strict)
10. [Test Everything](#test-everything)

---

## What is a VPS

A **Virtual Private Server (VPS)** is a rented Linux server in the cloud. You get full root access — you control what's installed, what runs, and how it's configured. Unlike shared hosting, nothing else on the machine affects your site.

**Common uses:** websites, APIs, game servers, bots, databases, self-hosted apps.

---

## Choose a VPS Provider

These are the most popular options for beginners on a budget:

| Provider | Starting Price | Good For |
|----------|---------------|----------|
| **DigitalOcean** | ~$6/mo (Droplet) | Beginners, clean UI |
| **Vultr** | ~$6/mo | Flexible locations |
| **Linode (Akamai)** | ~$6/mo | Reliability |
| **Hetzner** | ~€4/mo | Cheapest in EU |
| **AWS Lightsail** | ~$3.50/mo | AWS ecosystem |

> For a personal site or small app, any $6/mo plan with **1 vCPU + 1GB RAM** is plenty to start.

---

## Create Your Server

Steps shown for **DigitalOcean** — other providers are nearly identical.

1. Sign up and go to your dashboard
2. Click **Create** → **Droplets**

   ![Create Droplet button](images/vps/01-create-droplet.png)

3. Choose a region close to your users
4. Under **OS**, select **Ubuntu 24.04 LTS** (recommended)

   ![Ubuntu OS selection](images/vps/02-os-selection.png)

5. Choose a plan — **Basic / Regular / 1GB RAM** is fine for starters
6. Under **Authentication**, choose **SSH Key** (more secure than password)
   - If you don't have an SSH key yet, generate one:
     ```bash
     ssh-keygen -t ed25519 -C "your@email.com"
     ```
   - Copy your public key:
     ```bash
     # Windows
     cat ~/.ssh/id_ed25519.pub | clip

     # macOS
     cat ~/.ssh/id_ed25519.pub | pbcopy

     # Linux
     cat ~/.ssh/id_ed25519.pub
     ```
   - Paste it into the SSH key field in the provider dashboard

   ![SSH key input](images/vps/03-ssh-key.png)

7. Give your server a hostname (e.g., `my-website`)
8. Click **Create Droplet** — it takes about 30 seconds

---

## Connect to Your Server

Once created, copy your server's **IP address** from the dashboard.

```bash
ssh root@YOUR_SERVER_IP
```

Example:
```bash
ssh root@203.0.113.10
```

Type `yes` when asked to confirm the fingerprint. You're now inside your server.

---

## Initial Server Setup

Run these commands after your first login:

### 1 — Update the system
```bash
apt update && apt upgrade -y
```

### 2 — Create a non-root user
```bash
adduser yourname
usermod -aG sudo yourname
```

### 3 — Copy SSH key to the new user
```bash
rsync --archive --chown=yourname:yourname ~/.ssh /home/yourname
```

### 4 — Test the new user in a separate terminal
```bash
ssh yourname@YOUR_SERVER_IP
```

### 5 — Set up a basic firewall
```bash
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable
```

Confirm with `y` when prompted.

> From this point, log in as your new user instead of root.

---

## Install a Web Server

### Option A — Nginx (recommended for most)

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Test it by visiting `http://YOUR_SERVER_IP` in a browser — you should see the Nginx welcome page.

### Option B — Apache

```bash
sudo apt install apache2 -y
sudo systemctl enable apache2
sudo systemctl start apache2
```

---

### Create a Basic Site (Nginx)

```bash
sudo mkdir -p /var/www/example.com
sudo chown -R $USER:$USER /var/www/example.com
```

Create a test page:
```bash
nano /var/www/example.com/index.html
```

Paste this in:
```html
<!DOCTYPE html>
<html>
  <head><title>My Site</title></head>
  <body><h1>It works!</h1></body>
</html>
```

Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

Create an Nginx config for your domain:
```bash
sudo nano /etc/nginx/sites-available/example.com
```

Paste this (replace `example.com` with your actual domain):
```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable it and reload:
```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Point Your Domain with Cloudflare

1. Go to your Cloudflare dashboard → your domain → **DNS** → **Records**
2. Add an **A record** pointing to your server's IP:

   ```
   Type:    A
   Name:    @
   Content: YOUR_SERVER_IP
   Proxy:   DNS only (grey cloud) ← important during SSL setup
   TTL:     Auto
   ```

3. Add a **www** record:

   ```
   Type:    CNAME
   Name:    www
   Content: example.com
   Proxy:   DNS only (grey cloud)
   TTL:     Auto
   ```

   ![DNS A record pointing to VPS](images/vps/04-dns-a-record.png)

> Keep both records as **grey cloud (DNS only)** for now — Let's Encrypt needs to reach your server directly to issue the certificate. You'll switch to orange cloud after SSL is installed.

Wait a few minutes for DNS to propagate, then verify with:
```bash
nslookup example.com 8.8.8.8
```

---

## Install SSL with Let's Encrypt

Let's Encrypt gives you a free, trusted SSL certificate. Certbot automates the whole process.

### Install Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### Get Your Certificate

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot will:
- Verify you own the domain (via HTTP challenge)
- Issue the certificate
- Automatically update your Nginx config for HTTPS

When prompted:
- Enter your email address
- Agree to the terms
- Choose whether to share your email with EFF (optional)

If successful you'll see:
```
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/example.com/fullchain.pem
```

### Auto-Renewal

Certbot sets up auto-renewal automatically. Test it with:
```bash
sudo certbot renew --dry-run
```

---

## Set Cloudflare SSL to Full (Strict)

Now that your server has a valid certificate, switch everything to Full (Strict) mode.

### Step 1 — Turn the Cloudflare proxy back on

In your Cloudflare DNS settings, click the grey cloud on both records to turn them **orange (proxied)**.

### Step 2 — Set SSL mode to Full (Strict)

1. Go to **SSL/TLS** → **Overview**
2. Select **Full (Strict)**

   ![SSL Full Strict mode](images/vps/05-ssl-full-strict.png)

### Step 3 — Enable Always Use HTTPS

1. Go to **SSL/TLS** → **Edge Certificates**
2. Toggle **Always Use HTTPS** → On
3. Toggle **Automatic HTTPS Rewrites** → On

---

## Test Everything

Run through this checklist:

- [ ] `http://example.com` redirects to `https://example.com`
- [ ] `https://example.com` loads with a valid padlock in the browser
- [ ] `https://www.example.com` also works
- [ ] No mixed content warnings in browser DevTools (F12 → Console)
- [ ] SSL certificate is from Cloudflare (check in browser padlock → Certificate)

### Check your cert expiry
```bash
sudo certbot certificates
```

### Check Nginx is running
```bash
sudo systemctl status nginx
```

### Check your firewall rules
```bash
sudo ufw status
```

---

## Quick Command Reference

```bash
# Reload Nginx after config changes
sudo systemctl reload nginx

# Test Nginx config for syntax errors
sudo nginx -t

# Renew SSL certificate manually
sudo certbot renew

# Check open ports
sudo ufw status

# View Nginx error logs
sudo tail -f /var/log/nginx/error.log

# View Nginx access logs
sudo tail -f /var/log/nginx/access.log
```

---

## What's Next

- **Deploy an app** — Node.js, Python, PHP behind Nginx as a reverse proxy
- **Set up a database** — MySQL or PostgreSQL
- **Add email** — Use a separate mail service (Mailgun, Resend, Google Workspace) rather than hosting your own
- **Harden your server** — Disable root SSH login, set up fail2ban

---

*Last updated: April 2026*
