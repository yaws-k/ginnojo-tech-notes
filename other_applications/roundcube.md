---
---
# Roundcube

[Roundcube](https://roundcube.net/) is an open-source webmail client. It works as a fully functional MUA (mail user agent) with plugins.

The [official guide wiki](https://github.com/roundcube/roundcubemail/wiki) is available. This article follows the guide and adds some customization based on how this server is configured.

## Install

Download the "Complete" package `roundcubemail-x.x.x-complete.tar.gz` from the [official download page](https://roundcube.net/download/). It contains all required packages (some additional ones are still needed).

Extract compressed files.

```bash
tar xfz roundcubemail-x.x.x-complete.tar.gz
```

- It contains empty `temp` and `logs` directories. You don’t need to create them manually later.

Move the directory to `/var/www/`. For example, use the directory name `mail.example.jp`, the same name as the server's domain.

```bash
sudo mv ./roundcubemail-x.x.x /var/www/mail.example.jp
```

Change owner to `www-data` (nginx should have write access to the directory).

```bash
sudo chown -R www-data:www-data /var/www/mail.example.jp
```

## Database configuration

In my case, the number of users is so small that I use SQLite. SQLite doesn't require any preparation. The database file will be automatically created during the setup process.

## nginx configuration

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

        # Since 1.7, point to public_html for security reasons
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

## Setup Roundcube

### Additional php packages

Access `https://mail.example.jp/installer.php` to start the initial setup.  
This will check if all required packages are available and show missing items.

- The basic extensions [installed with PHP itself](../base_system/basic_configuration_and_utilities#php-84) should cover all requirements.

### Roundcube configuration

- Choose `SQLite` as the database and set the path, for example, `/var/www/mail.example.jp/sqlite.db`
- IMAP should work with default configuration.
- SMTP needs `ssl://` prefix to use Submissions. To avoid certificate errors, set smtp_host to the FQDN: `ssl://mail.example.jp:465`

"Create config" will save the config file. The "Continue" button will check the directory and DB permissions.  
Check if SMTP and IMAP login work.

Delete installer directory.

```bash
sudo rm -r /var/www/roundcube/installer
```

## Access Roundcube

Now `https://mail.example.jp/` should work, and you can log in with your mail addresses and their passwords.

## Upgrade

Follow the instructions in the official Roundcube upgrade guide: [https://github.com/roundcube/roundcubemail/wiki/Upgrade](https://github.com/roundcube/roundcubemail/wiki/Upgrade)

Backup the current Roundcube installation. (It includes the SQLite database.)

```bash
sudo tar czvf /var/www/roundcube-backup.tar.gz /var/www/mail.example.jp
```

Download the latest Complete package from the [official download page](https://roundcube.net/download/) and extract it to a temporary location.

```bash
wget https://github.com/roundcube/roundcubemail/releases/download/1.7.5/roundcubemail-1.x.x-complete.tar.gz
tar xf roundcubemail-1.x.x-complete.tar.gz
```

Use the installer script with the target directory as an argument.

```bash
cd roundcubemail-1.x.x
sudo bin/installto.sh /var/www/mail.example.jp
```

Set the owner back to `www-data`.

```bash
sudo chown -R www-data:www-data /var/www/mail.example.jp
```

Check if the Roundcube installation works by accessing `https://mail.example.jp/` in your web browser.
