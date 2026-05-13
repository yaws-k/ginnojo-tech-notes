---
---
# Matomo

[Matomo](https://matomo.org/) is a free and open-source web analytics application with high accuracy and a strong focus on data privacy.  
If you want to control every data, or need to consider GDPR compliance, it can be a good choice.

There is the [official installation guide](https://matomo.org/faq/on-premise/installing-matomo/). This article follows the guide and adds some customization based on how this server is configured.

## Download

Download the latest package `matomo-latest.zip` from the [official download page](https://matomo.org/download/) and extract it.

```bash
wget https://builds.matomo.org/matomo-latest.zip
unzip matomo-latest.zip
```

## Database preparation

Prepare a database for Matomo.

Login to MariaDB as root (=MariaDB administrator).

```bash
sudo mariadb
```

On MariaDB console, create a database `matomo` and a user `matomo` with password `password`.  
(Of course you must set a strong password in production environment.)

```sql
CREATE DATABASE matomo;
CREATE USER 'matomo'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON matomo.* TO 'matomo'@'localhost';
GRANT FILE ON *.* TO 'matomo'@'localhost';
FLUSH PRIVILEGES;
```

## nginx configuration

Prepare the SSL/TLS certificate for this site and create `/etc/nginx/sites-available/matomo.example.jp` for Matomo and enable it.  
nginx configuration template is on the [official GitHub repository](https://github.com/matomo-org/matomo-nginx/blob/master/sites-available/matomo.conf).

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name matomo.example.jp;

        include snippets/certbot.conf;
}

server {
        include snippets/https-common.conf;

        ssl_certificate /etc/letsencrypt/live/matomo.example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/matomo.example.jp/privkey.pem;

        server_name matomo.example.jp;

        add_header Referrer-Policy origin always;
        add_header X-Content-Type-Options "nosniff" always;

        root /var/www/matomo.example.jp/;

        index index.php;

        # only allow accessing the following php files
        location ~ ^/(index|matomo|piwik|js/index|plugins/HeatmapSessionRecording/configs)\.php$ {
                include snippets/fastcgi-php.conf;
                fastcgi_param HTTP_PROXY "";
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }

        # deny access to all other .php files
        location ~* ^.+\.php$ {
                deny all;
                return 403;
        }

        # serve all other files normally
        location / {
                try_files $uri $uri/ =404;
        }

        # disable all access to the following directories
        location ~ ^/(config|tmp|core|lang) {
                deny all;
                return 404;
        }

        location ~ /\.ht {
                deny  all;
                return 403;
        }

        location ~ js/container_.*_preview\.js$ {
                expires off;
                add_header Cache-Control 'private, no-cache, no-store';
        }

        location ~ \.(gif|ico|jpg|png|svg|js|css|htm|html|mp3|mp4|wav|ogg|avi|ttf|eot|woff|woff2)$ {
                allow all;
                expires 1h;
                add_header Pragma public;
                add_header Cache-Control "public";
        }

        location ~ ^/(libs|vendor|plugins|misc|node_modules) {
                deny all;
                return 403;
        }

        # properly display textfiles in root directory
        location ~/(.*\.md|LEGALNOTICE|LICENSE) {
                default_type text/plain;
        }

        access_log /var/log/nginx/matomo.example.jp-access.log;
        error_log /var/log/nginx/matomo.example.jp-error.log;
}
```

## Setup Matomo

Access `https://matomo.example.jp/` to start the initial setup.

When accessing the site, turn of uBlock Origin (Lite) or other ad blockers. Matomo related resources are blocked as tracking scripts, and the page may not work as expected.
{: .notice--warning}

### System Check

Matomo will check the system requirements. The basic extensions installed with PHP itself should cover all requirements.

### Database setup

Enter the database information according to the preparation.

- Don't forget to set the Database Engine to `MariaDB`.

### Creating the Tables

Matomo will automatically create tables.

### Super User

Set the administrator of this Matomo instance.

### Set up a Website

Set the website to track.

### JavaScript Tracking Code

As described in the page, copy & paste the script just before the end of HTML headers, `</head>` tag.

## Update config.ini.php

If you get this error when logging in to Matomo, add the FQDN to `trusted_hosts[]` in `config/config.ini.php`.

**Error**: The form security failed because of an invalid "Referer" header. If you are using a proxy server, you must [configure Matomo to accept the proxy header that forwards the host header](https://matomo.org/faq/how-to-install/faq_98). Also, check that your "Referer" header is sent correctly. If you previously connected using HTTPS, please ensure you are connecting over a secure (SSL/TLS) connection and try again.
{: .notice--danger}

It seems that the proxy configuration is not required. Just add the Matomo instance's FQDN to `trusted_hosts[]`.

```ini
[General]
trusted_hosts[] = "matomo.example.jp"
```
