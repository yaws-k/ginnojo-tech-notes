---
---
# Roundcube

{% include version_warning.html %}

[Roundcube](https://roundcube.net/) is an open-source webmail client. It works as a fully functional mail user agent.

## Install

There is the [official installation guide](https://github.com/roundcube/roundcubemail/wiki/Installation). The process follows this document.

Download the "Complete" package from the [official site](https://roundcube.net/download/). It contains all required packages (some additional ones are still needed).

```console
wget https://github.com/roundcube/roundcubemail/releases/download/x.x.x/roundcubemail-x.x.x-complete.tar.gz
```

- `x.x.x` depends on the latest stable version

Extract compressed files.

```console
tar xfz roundcubemail-x.x.x-complete.tar.gz
```

- It contains empty `temp` and `logs` directories. You don't have to make them later manually.

Move the directory to `/var/www/`. For future updates, extract the version number from the directory name.

```console
sudo mv ./roundcubemail-x.x.x /var/www/roundcubemail
```

Change owner to `www-data`

```console
sudo chown -R www-data:www-data /var/www/roundcubemail
```

## Database Configuration

As the install guide explains, set up a database and a user on MariaDB.

```console
sudo mariadb
```

Execute SQL.

```sql
CREATE DATABASE roundcubemail CHARACTER SET utf8 COLLATE utf8_general_ci;
GRANT ALL PRIVILEGES ON roundcubemail.* TO username@localhost IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
```

Move to Roundcube home directory and import initial tables.

```console
mariadb -u username -p roundcubemail < SQL/mysql.initial.sql
```

## Nginx Configuration

Add a config file in `/etc/nginx/sites-available/mail.example.jp` for SSL/TLS certificate.

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name mail.example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}
```

Enable this site.

```console
sudo ln -s /etc/nginx/sites-available/mail.example.jp /etc/nginx/sites-enabled/mail.example.jp
sudo systemctl reload nginx
```

Issue an ssl certificate.

```console
sudo certbot certonly --webroot -w /var/www/certbot/ -d mail.example.jp
```

Complete the config file in `/etc/nginx/sites-available/mail.example.jp`

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name mail.example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        ssl_certificate /etc/letsencrypt/live/mail.example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/mail.example.jp/privkey.pem;

        server_name mail.example.jp;

        root /var/www/mail.example.jp/roundcubemail;

        index index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php($|/) {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }

        access_log /var/log/nginx/mail.example.jp-access.log;
        error_log /var/log/nginx/mail.example.jp-error.log;
}
```

Reload nginx.

```console
sudo systemctl reload nginx
```

## Initialize Roundcube

### Additional php packages

Access `https://mail.example.jp/installer/` to start the initial setup.  
This will check if all required packages are available and show missing items.

The easiest solution is to add php packages through apt.

- cURL: `php-curl`
- DOM: `php-xml`
- Imagick: `php-imagick`
- Intl: `php-intl`

### Roundcube configuration

- Input Database access credentials in "Database setup" section.
- IMAP should work with default configuration.
- SMTP needs `tls://` prefix to use STARTTLS. To use TLS and check the domain name, set smtp_host with FQDN; `tls://mail.example.jp:587`

"Create config" will save the config file. The "Continue" button will check the directory and DB permissions.  
Check if SMTP and IMAP login work.

Delete installer directory.

```console
sudo rm -r /var/www/roundcube/installer
```

## Access Roundcube

Now `https://mail.example.jp/` should work, and you can log in with your mail addresses and their passwords.
