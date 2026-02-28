# DigitalOcean Deployment (Simple)

This guide replaces your current site with this Hugo blog, while keeping WordPress backups.

Your Droplet runs **Apache 2.4.41 on Ubuntu**.

## 1) Backup current WordPress first

On your Droplet:

```bash
sudo mkdir -p /root/backups
sudo tar -czf /root/backups/wp-files-$(date +%F).tar.gz /var/www
sudo mysqldump -u root -p --all-databases > /root/backups/wp-db-$(date +%F).sql
```

Also take a Droplet snapshot from the DigitalOcean dashboard.

## 2) Prepare deploy directories

```bash
sudo mkdir -p /srv/blog/releases
sudo useradd -m -s /bin/bash deploy || true
sudo chown -R deploy:deploy /srv/blog
```

Add your GitHub Actions SSH public key to `/home/deploy/.ssh/authorized_keys`:

```bash
sudo mkdir -p /home/deploy/.ssh
sudo touch /home/deploy/.ssh/authorized_keys
# paste the public key into authorized_keys
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

## 3) Apache config (serve current symlink)

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

    # Serve index.html for directory requests
    DirectoryIndex index.html

    # Custom 404
    ErrorDocument 404 /404.html

    # Enable rewrites for clean URLs
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

Enable the site, required modules, and disable the default site:

```bash
# Enable required modules
sudo a2enmod rewrite

# Disable the default site (currently serving WordPress)
sudo a2dissite 000-default.conf

# Enable the new blog site
sudo a2ensite blog.conf

# Test config and reload
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Then enable HTTPS with certbot:

```bash
sudo certbot --apache -d abhijitmohanty.com -d www.abhijitmohanty.com
```

## 4) GitHub repo secrets

In GitHub repo settings (Settings > Secrets and variables > Actions), add:

- `DO_HOST` — `143.110.253.218`
- `DO_USER` — `deploy`
- `DO_PORT` — `22`
- `DO_SSH_KEY` — private key matching the authorized key above (full PEM block)

## 5) Deploy flow

Push to `main`.

GitHub Actions will:

- build with Hugo
- upload to `/srv/blog/releases/<timestamp>`
- switch `/srv/blog/current` symlink atomically
- keep last 7 releases, prune older ones

## 6) Verify after first deploy

- `https://abhijitmohanty.com`
- one post URL (for example `/posts/welcome/`)
- `/index.xml`
- `/sitemap.xml`
- 404 route (try a non-existent URL)

## 7) Rollback (instant)

On Droplet:

```bash
cd /srv/blog/releases && ls -1dt */
sudo ln -sfn /srv/blog/releases/<previous_release> /srv/blog/current
# No reload needed — Apache follows the symlink automatically
```

## 8) Cleanup (after 30 days)

Once you're confident the new blog is stable:

```bash
# Remove WordPress files
sudo rm -rf /var/www/html/wordpress  # adjust path as needed

# Remove MySQL/MariaDB if no longer needed
sudo apt purge mysql-server mysql-client  # or mariadb-server

# Remove PHP if no longer needed
sudo apt purge php* libapache2-mod-php

sudo apt autoremove
```

Keep `/root/backups/` and the Droplet snapshot for at least 30 days.
