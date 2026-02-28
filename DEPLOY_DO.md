# Deployment Guide

## How it works

```
Push to main
    │
    ▼
GitHub Actions builds Hugo
    │
    ▼
Auto-deploys to GitHub Pages (staging preview)
    https://mohantyabhijit.github.io/
    │
    ▼
You review the preview
    │
    ▼
Approve the "production" deployment in GitHub Actions
    │
    ▼
Rebuilds Hugo and deploys to DigitalOcean Droplet
    https://abhijitmohanty.com
```

- **Staging** = GitHub Pages (free, automatic on every push to `main`)
- **Production** = DigitalOcean Droplet at `143.110.253.218` (requires manual approval)

## Approving a production deployment

1. Go to [Actions](https://github.com/mohantyabhijit/mohantyabhijit.github.io/actions)
2. Click the latest workflow run
3. The `deploy-production` job will show "Waiting for review"
4. Click "Review deployments", check "production", click "Approve and deploy"

## Initial server setup (one-time, on Droplet)

### 1) Backup current WordPress

```bash
sudo mkdir -p /root/backups
sudo tar -czf /root/backups/wp-files-$(date +%F).tar.gz /var/www
sudo mysqldump -u root -p --all-databases > /root/backups/wp-db-$(date +%F).sql
```

Also take a Droplet snapshot from the DigitalOcean dashboard.

### 2) Create deploy user and directories

```bash
sudo mkdir -p /srv/blog/releases
sudo useradd -m -s /bin/bash deploy || true
sudo chown -R deploy:deploy /srv/blog
```

### 3) Set up SSH key for GitHub Actions

Generate a key pair (run this on your local machine):

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/do_deploy_key -N ""
```

Copy the **public** key to the Droplet:

```bash
ssh root@143.110.253.218 'mkdir -p /home/deploy/.ssh && cat >> /home/deploy/.ssh/authorized_keys' < ~/.ssh/do_deploy_key.pub
ssh root@143.110.253.218 'chown -R deploy:deploy /home/deploy/.ssh && chmod 700 /home/deploy/.ssh && chmod 600 /home/deploy/.ssh/authorized_keys'
```

Add the **private** key as a GitHub secret (see section below).

### 4) Apache config

Create `/etc/apache2/sites-available/blog.conf`:

```apache
<VirtualHost *:80>
    ServerName abhijitmohanty.com
    ServerAlias www.abhijitmohanty.com

    DocumentRoot /srv/blog/current

    <Directory /srv/blog/current>
        Options FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

    DirectoryIndex index.html
    ErrorDocument 404 /404.html

    <Directory /srv/blog/current>
        <IfModule mod_rewrite.c>
            RewriteEngine On
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteCond %{REQUEST_FILENAME} !-d
            RewriteRule . /404.html [L,R=404]
        </IfModule>
    </Directory>
</VirtualHost>
```

Enable the site:

```bash
sudo a2enmod rewrite
sudo a2dissite 000-default.conf
sudo a2ensite blog.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### 5) HTTPS with certbot

```bash
sudo certbot --apache -d abhijitmohanty.com -d www.abhijitmohanty.com
```

### 6) GitHub repo secrets

In [repo settings > Secrets > Actions](https://github.com/mohantyabhijit/mohantyabhijit.github.io/settings/secrets/actions), add:

| Secret | Value |
|--------|-------|
| `DO_HOST` | `143.110.253.218` |
| `DO_USER` | `deploy` |
| `DO_PORT` | `22` |
| `DO_SSH_KEY` | Contents of `~/.ssh/do_deploy_key` (the private key) |

## Rollback (instant)

On the Droplet:

```bash
cd /srv/blog/releases && ls -1dt */
sudo ln -sfn /srv/blog/releases/<previous_release> /srv/blog/current
# No reload needed — Apache follows the symlink automatically
```

## Cleanup (after 30 days)

Once the new blog is stable:

```bash
sudo rm -rf /var/www/html/wordpress
sudo apt purge mysql-server mysql-client  # or mariadb-server
sudo apt purge php* libapache2-mod-php
sudo apt autoremove
```

Keep `/root/backups/` and the Droplet snapshot for at least 30 days.
