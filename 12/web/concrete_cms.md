---
---
# Concrete CMS

{% include version_warning.html %}

[Concrete CMS](https://www.concretecms.org/) is a Content Management System. Unlike Wordpress, it focuses on independent pages than blog posts.

## Install

There is the [official install guide](https://documentation.concretecms.org/developers/introduction/installing-concrete-cms).

Download [the latest version](https://www.concretecms.org/download).  
Though the recommended method is Composer, it is too much to use only for Concrete installation. This article shows a simple way to download and unzip the file.

```console
wget https://www.concretecms.org/download_file/2dxxxx/xxxx -O concrete.zip
```

- Specify the output filename, or the saved file will be 4-digit without a .zip extension.

Unzip the downloaded file and move all files to `/var/www`.

```console
unzip concrete.zip
sudo mv concrete-cms-9.x.x /var/www/concrete
```

Change the owner.

```console
sudo chown -R www-data:www-data /var/www/concrete
```

## Prepare Database

```console
sudo mariadb
```

```sql
CREATE DATABASE concrete_cms;
GRANT ALL PRIVILEGES ON concrete_cms.* TO 'concrete'@'localhost' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

## Nginx configuration

Add a config file in `/etc/nginx/sites-available/concrete.example.jp` for SSL/TLS certificate.

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name concrete.example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}
```

Enable this site.

```console
sudo ln -s /etc/nginx/sites-available/concrete.example.jp /etc/nginx/sites-enabled/concrete.example.jp
sudo systemctl reload nginx
```

Issue an ssl certificate.

```console
sudo certbot certonly --webroot -w /var/www/certbot/ -d concrete.example.jp
```

Complete the config file in `/etc/nginx/sites-available/concrete.example.jp`

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name concrete.example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        ssl_certificate /etc/letsencrypt/live/concrete.example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/concrete.example.jp/privkey.pem;

        server_name concrete.example.jp;

        root /var/www/concrete;

        index index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php($|/) {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }

        access_log /var/log/nginx/concrete.example.jp-access.log;
        error_log /var/log/nginx/concrete.example.jp-error.log;
}
```

## Initial screens

Access `https://concrete.example.com/` and installation will start. The installer will check if the required php modules are available.

They should be available through apt.

- DOM Extension: `php-xml`
- Image Manipulation: `php-gd`
- XML support: `php-xml` (same as DOM Extension)
- Internationalization Support: `php-mbstring`
- Zip support: `php-zip`

The installation screens are self-explanatory, and there is no need to explain them here. Proceed the steps, and you will be logged in as an "admin" user.
