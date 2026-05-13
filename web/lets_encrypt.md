---
---
# Let's Encrypt

[Let's Encrypt](https://letsencrypt.org/) privides free TLS certificates. It also provides the official client [Certbot](https://certbot.eff.org/) to create, renew, and revoke certificates.

This article explains the simplest and recommended way, [Certbot snap app](https://eff-certbot.readthedocs.io/en/stable/install.html#snap-recommended) and [webroot](https://eff-certbot.readthedocs.io/en/stable/using.html#webroot) to mange certificates.

- There are options like [DNS plugins](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins) or [Certbot through Python pip](https://eff-certbot.readthedocs.io/en/stable/install.html#alternative-2-pip) for third-party plugins, but they are not recommended.

## Install Certbot

Don't forget to add `--classic` option.

```bash
sudo snap install --classic certbot
```

Add certbot symlink to enable `sudo certbot` command.

```bash
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
```

"webroot" doesn't require any plugins. Certbot itself is enough.

## HTTP challenge

### Directory for webroot

Certbot webroot check will make a file into `${webroot-path}/.well-known/acme-challenge/(random url)` and Let's Encrypt server checks if that file is accessible and correct.

For Let's Encrypt validator, make a dedicated directory.

```bash
sudo mkdir -p /var/www/certbot/.well-known/acme-challenge
```

### Default site configuration

Accept HTTP access by Let's Encrypt validator with the default server (catch-all server).

- Catch-all server is convenient for the first certificate issue before setting up the per-site configuration. (Renewal can be done with the per-site configuration.)
- You may want certificates not for websites, for example, for mail servers.

```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/certbot;

        # Disconnect all by default
        location / {
                return 444;
        }

        # Accept Let's Encrypt validation, or disconnect
        location /.well-known/acme-challenge/ {
                try_files $uri =444;
        }
}

server {
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;
        http2 on;

        listen 443 quic reuseport default_server;
        listen [::]:443 quic reuseport default_server;
        http3 on;
        add_header Alt-Svc 'h3=":443"; ma=86400';

        include snippets/snakeoil.conf;

        ssl_protocols TLSv1.2 TLSv1.3;

        server_name _;

        # Disconnect
        return 444;
}
```

### Per-site snippet

Make a snippet for certbot configuration. This will be included in the server block for HTTP access from Let's Encrypt server.

`/etc/nginx/snippets/certbot.conf`

```nginx
# Add root directory
root /var/www/certbot;

# Redirect all by default
location / {
        return 308 https://$host$request_uri;
}

# Exception for Let's Encrypt validation
location /.well-known/acme-challenge/ {
        try_files $uri =404;
}
```

`/etc/nginx/sites-available/exmaple.jp.conf` will look like the following, after the certificate is issued.

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}

server {
        include snippets/https-common.conf;

        (snip)

        # Include SSL/TLS configurations
        ssl_certificate /etc/letsencrypt/live/example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.jp/privkey.pem;
}
```

With this,

- Access to `http://example.jp/.well-known/acme-challenge/(any)` will check if the file exists
- All accesses other than `/.well-known/acme-challenge/*` will be redirected to HTTPS

## Issue a certificate

Request a certificate.

- Certbot will ask for your email address and agreement confirmation to generate the account information.

```bash
$ sudo certbot certonly --webroot -w /var/www/certbot/ -d example.jp

Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): email@example.jp

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at:
https://letsencrypt.org/documents/LE-SA-v1.6-August-18-2025.pdf
You must agree in order to register with the ACME server. Do you agree?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you be willing, once your first certificate is successfully issued, to
share your email address with the Electronic Frontier Foundation, a founding
partner of the Let's Encrypt project and the non-profit organization that
develops Certbot? We'd like to send you email about our work encrypting the web,
EFF news, campaigns, and ways to support digital freedom.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: N
Account registered.
Requesting a certificate for example.jp

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/example.jp/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/example.jp/privkey.pem
This certificate expires on YYYY-MM-DD.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

## nginx configuration

Configure nginx based on Mozilla's [SSL Configuration Generator](https://ssl-config.mozilla.org/).

- [The official guide to configure HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html) explains the basic configurations.

### nginx default

According to the Mozilla's [SSL Configuration Generator (nginx 1.26.3, OpenSSL 3.5.5)](https://ssl-config.mozilla.org/#server=nginx&version=1.26.3&config=intermediate&openssl=3.5.5&hsts).

Update `/etc/nginx/nginx.conf` to set the default SSL/TLS configurations.

```nginx
        ##
        # SSL Settings
        ##

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ecdh_curve X25519MLKEM768:X25519:prime256v1:secp384r1;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
        ssl_prefer_server_ciphers off;

        ssl_session_timeout 1d;
        ssl_session_cache shared:MozSSL:10m;
```

- `ssl-dhparams.pem` file used to be generated with `openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048` command for DHE-\* ciphers. However, DHE-* cyphers are not listed that dhparam file is not required.

### Per-server configurations

Swap the snakeoil certificate to Let's Encrypt certificate in `/etc/nginx/sites-available/example.jp.conf`

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name example.jp;

        include snippets/certbot.conf;
}

server {
        include snippets/https-common.conf;

        server_name example.jp;

        # Include SSL/TLS configurations
        ssl_certificate /etc/letsencrypt/live/example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.jp/privkey.pem;

        root /var/www/html;

        index index.html index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        access_log /var/log/nginx/example.jp-access.log;
        error_log /var/log/nginx/example.jp-error.log;

        location ~ \.php {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }
}
```

Check the server security configuration with [Qualis SSL Labs](https://www.ssllabs.com/ssltest/index.html).

- If you want more strict access than HSTS, check [HSTS Preload List](https://hstspreload.org/).

## Renewal hook

Certbot will automatically renew the certificate before it expires, and it can reload other applications after the renewal.

Add script files `/etc/letsencrypt/renewal-hooks/deploy/reload-services.sh`.

```bash
#!/bin/bash
systemctl reload nginx
```

- Add `postfix` and `dovecot` after installing them.

Add execute permission to the script.

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-services.sh
```

## Revoking certificate

Certbot renews certificates automatically. If you want to stop using the certificate, [revoke it](https://eff-certbot.readthedocs.io/en/latest/using.html#revoking-certificates).  
It can also delete the certificate files.

```console
$ sudo certbot revoke --cert-name example.jp
Saving debug log to /var/log/letsencrypt/letsencrypt.log

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Would you like to delete the certificate(s) you just revoked, along with all
earlier and later versions of the certificate?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es (recommended)/(N)o: Y

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
The following certificate(s) are selected for deletion:

  * example.jp

WARNING: Before continuing, ensure that the listed certificates are not being
used by any installed server software (e.g. Apache, nginx, mail servers).
Deleting a certificate that is still being used will cause the server software
to stop working. See https://certbot.org/deleting-certs for information on
deleting certificates safely.

Are you sure you want to delete the above certificate(s)?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
Deleted all files relating to certificate example.jp.
Congratulations! You have successfully revoked the certificate that was located
at /etc/letsencrypt/live/example.jp/cert.pem.
```
