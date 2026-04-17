---
---
# Let's Encrypt

[Let's Encrypt](https://letsencrypt.org/) privides free TLS certificates. It also provides the official client [Certbot](https://certbot.eff.org/) to create, renew, and revoke certificates.

This article explains the simplest and recommended way, [Certbot snap app](https://eff-certbot.readthedocs.io/en/stable/install.html#snap-recommended) and [webroot](https://eff-certbot.readthedocs.io/en/stable/using.html#webroot) to mange certificates.

- There are options like [DNS plugins](https://eff-certbot.readthedocs.io/en/stable/using.html#dns-plugins) or [Certbot through Python pip](https://eff-certbot.readthedocs.io/en/stable/install.html#alternative-2-pip) for third-party plugins, but they are not recommended.

## Install Certbot

Don't forget to add `--classic` option.

```console
sudo snap install --classic certbot
```

Add certbot symlink to enable `sudo certbot` command.

```console
sudo ln -s /snap/bin/certbot /usr/local/bin/certbot
```

"webroot" doesn't require any plugins. Certbot itself is enough.

## Configure the site for http challenge

Certbot webroot check will make a file into `${webroot-path}/.well-known/acme-challenge/(random url)` and Let's Encrypt server checks if that file is accessible and correct.

For Let's Encrypt validator, make a dedicated directory.

```console
sudo mkdir -p /var/www/certbot/.well-known/acme-challenge
```

Make a snippet for certbot configuration. This will be included in the server block for HTTP access from Let's Encrypt server.

`/etc/nginx/snippets/certbot.conf`

```nginx
# Add root directory
root /var/www/certbot;

# Redirect all by default
location / {
        return 301 https://$host$request_uri;
}

# Add the exception
location /.well-known/acme-challenge/ {
        try_files $uri =404;
}
```

`/etc/nginx/sites-available/exmaple.jp.conf`

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;
        http2 on;

        (snip)
}
```

With this,

- Access to `http://example.jp/.well-known/acme-challenge/(any)` will check if the file exists
- All accesses other than `/.well-known/acme-challenge/*` will be redirected to HTTPS

## Issue a certificate

Request a certificate.

- Certbot will ask for your email address and agreement confirmation to generate the account information.

```console
$ sudo certbot certonly --webroot -w /var/www/certbot/ -d example.jp

Saving debug log to /var/log/letsencrypt/letsencrypt.log
Enter email address (used for urgent renewal and security notices)
 (Enter 'c' to cancel): email@example.jp

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.4-April-3-2024.pdf. You must agree in
order to register with the ACME server. Do you agree?
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
This certificate expires on 2024-11-23.
These files will be updated when the certificate renews.
Certbot has set up a scheduled task to automatically renew this certificate in the background.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

## nginx configuration

For more details, read [the official guide to configure HTTPS servers](https://nginx.org/en/docs/http/configuring_https_servers.html).

### Security enhancement

When using Certbot with the installer option (e.g. --nginx), it automatically takes care of this enhancement. With webroot method, this file has to be made manually.  
Refer: [options-ssl-nginx.conf](https://github.com/certbot/certbot/blob/master/certbot-nginx/certbot_nginx/_internal/tls_configs/options-ssl-nginx.conf)

Add `/etc/letsencrypt/options-ssl-nginx.conf`

```nginx
ssl_session_cache shared:le_nginx_SSL:10m;
ssl_session_timeout 1440m;
ssl_session_tickets off;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;

ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
```

Generate `/etc/letsencrypt/ssl-dhparams.pem`

```console
sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
```

- You may find the command with 2048 just after the 'dhparam', but that will not save the file as you expect. The numbits (number) MUST BE the last option.
See [OpenSSL official document](https://docs.openssl.org/3.0/man1/openssl-dhparam/) for more details.

The file should look like this.

```plaintext
-----BEGIN DH PARAMETERS-----
MIIBxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
(snip)
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxBAg==
-----END DH PARAMETERS-----
```

### Add SSL/TLS configurations

Swap the snakeoil certificate for testing to proper configurations in `/etc/nginx/sites-available/exmaple.jp.conf`

```nginx
server {
        listen 80;
        listen [::]:80;
        server_name example.jp;

        # Include certbot configuration
        include snippets/certbot.conf;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        # Include SSL/TLS configurations
        ssl_certificate /etc/letsencrypt/live/example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.jp/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        server_name example.jp;
        (snip)
}
```

## Revoking certificate

Certbot renews certificates automatically. If you want to stop using the certificate, [revoke it](https://eff-certbot.readthedocs.io/en/latest/using.html#revoking-certificates).

```console
sudo certbot revoke --cert-name example.jp
```
