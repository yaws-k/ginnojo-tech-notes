---
---
# Roundcube

[Roundcube](https://roundcube.net/) is an open-source webmail client. It works as a fully functional MUA (mail user agent) with plugins.

There is the [official guide wiki](https://github.com/roundcube/roundcubemail/wiki). This article follows this guide and adds some customization based on how this server is set according to this site instructions.

## Install

Download the "Complete" package `roundcubemail-x.x.x-complete.tar.gz` from the [official download page](https://roundcube.net/download/). It contains all required packages (some additional ones are still needed).

Extract compressed files.

```bash
tar xfz roundcubemail-x.x.x-complete.tar.gz
```

- It contains empty `temp` and `logs` directories. You don't have to make them later manually.

Move the directory to `/var/www/`. For future updates, delete the version number from the directory name.

```bash
sudo mv ./roundcubemail-x.x.x /var/www/roundcube
```

Change owner to `www-data` (nginx should have write access to the directory).

```bash
sudo chown -R www-data:www-data /var/www/roundcube
```

## Database Configuration

In my case, the number of users is so small that I use Sqlite. Sqlite doesn't require any preparation. The database file will be automatically created during the setup process.

## Nginx Configuration

Prepare the SSL/TLS certificate for this site and create `/etc/nginx/sites-available/mail.example.jp` for Roundcube and enable it.

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name mail.example.jp;

        include snippets/certbot.conf;
}

server {
        include snippets/https-common.conf;

        ssl_certificate /etc/letsencrypt/live/mail.example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/mail.example.jp/privkey.pem;

        server_name mail.example.jp;

        # Since 1.7, root point to public_html for security reason
        root /var/www/mail.example.jp/public_html;

        index index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        location ~ \.php {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }

        access_log /var/log/nginx/mail.example.jp-access.log;
        error_log /var/log/nginx/mail.example.jp-error.log;
}
```

## Initialize Roundcube

### Additional php packages

Access `https://mail.example.jp/installer.php` to start the initial setup.  
This will check if all required packages are available and show missing items.

- The basic extensions [installed with PHP iteself](../base_system/basic_configuration_and_utilities#php-84) should cover all requirements.

### Roundcube configuration

- Choose `SQLite` as the database and set the path, for example, `/var/www/roundcube/sqlite.db`
- IMAP should work with default configuration.
- SMTP needs `ssl://` prefix to use Submissions. To avoid the certificate error, set smtp_host with FQDN; `ssl://mail.example.jp:465`

"Create config" will save the config file. The "Continue" button will check the directory and DB permissions.  
Check if SMTP and IMAP login work.

Delete installer directory.

```bash
sudo rm -r /var/www/roundcube/installer
```

## Access Roundcube

Now `https://mail.example.jp/` should work, and you can log in with your mail addresses and their passwords.
