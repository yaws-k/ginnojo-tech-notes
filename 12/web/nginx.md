---
---
# nginx

{% include version_warning.html %}

## Install

"nginx" package has some variations according to the bundled modules. Choose either one from nginx-light, nginx-core, nginx-full, nginx-extras. In my case, the lightest nginx-light is enough.

```console
sudo apt install nginx-light ssl-cert
```

Open HTTP(S) ports

```console
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --reload
```

## Gzip

Gzip compression is turned on by default, but only for the text/html. Enabling compression for all other text contents will increase performance.
NOTE: Do not turn on this compression with SSL/TLS if you care about BREACH attacks. For more details, see the [gzip module explanation](https://nginx.org/en/docs/http/ngx_http_gzip_module.html).

### Global configuration

If you don't have to consider BREACH attacks, turn on gzip globally. Uncomment all the gzip configurations in `/etc/nginx/nginx.conf`.

```nginx
##
# Gzip Settings
##

gzip on;

gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

Reload nginx to enable.

```console
sudo systemctl reload nginx
```

### Per-site configuration

You can turn on Gzip within the "server" section. Snippets will help control per-server configurations.

Make `/etc/nginx/snippets/gzip.conf` file with the same configurations as in the global config file.

```nginx
gzip on;

gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

Include this snippet in the server section.

```nginx
server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        include snippets/gzip.conf;
        (snip)
}
```

## Site configuration

nginx stores website configuration files in `/etc/nginx/sites-available`. Add a symlink to that config file in `/etc/nginx/sites-enabled` to enable a site. (The same as Apache2.)

For more details about each configuration line, please refer to [official manuals](https://nginx.org/en/docs/http/ngx_http_core_module.html).

### The simplest example

The simplest example:

- Domain name is `example.jp`
- An HTTP site (non-SSL/TLS)
- Providing static files in `/var/www/html`
- Listening both IPv4 and IPv6
- Logs in `/var/log/nginx/exmaple-jp-*`

```nginx
server {
        listen 80;
        listen [::]:80;

        server_name example.jp;

        root /var/www/html;

        index index.html;

        location / {
                try_files $uri $uri/ =404;
        }

        access_log /var/log/nginx/example.jp-access.log;
        error_log /var/log/nginx/example.jp-error.log;
}
```

To enable this site, make a symlink at `/etc/nginx/sites-enabled/example.jp` and reload nginx.

```console
sudo ln -s /etc/nginx/sites-available/example.jp /etc/nginx/sites-enabled/example.jp
sudo systemctl reload nginx
```

### Enable PHP

- Add `index.php` as an index file
- PHP fpm is listening unix socket at `/run/php/php-fpm.sock`

Prerequisites

- PHP and fpm have to be installed
- The fpm automatically makes an unix socket to listen to

```nginx
server {
        listen 80;
        listen [::]:80;

        server_name example.jp;

        root /var/www/html;

        # Add index.php if required
        index index.html index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        access_log /var/log/nginx/example.jp-access.log;
        error_log /var/log/nginx/example.jp-error.log;

        # Pass PHP scripts to FastCGI server
        location ~ \.php($|/) {
                # Include PHP snippet
                include snippets/fastcgi-php.conf;

                # With php-fpm (or other unix sockets)
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }
}
```

### Enable HTTPS (with http2)

- Use "snakeoil" testing certificate for SSL/TLS

```nginx
server {
        # Add "http2" and change the port from 80 to 443
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        # Include snakeoil certificate snippet
        include snippets/snakeoil.conf;

        server_name example.jp;

        root /var/www/html;

        index index.html index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        access_log /var/log/nginx/example.jp-access.log;
        error_log /var/log/nginx/example.jp-error.log;

        location ~ \.php($|/) {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }
}
```

The next step is getting a proper certificate. This is explained in the "Let's Encrypt part.

### Redirect HTTP to HTTPS

- Redirect all access to `http://example.jp/`(non-SSL/TLS) to `https://example.jp/`(SSL/TLS)

```nginx
# Redirect all HTTP (port 80) access to HTTPS (port 443)
server {
        listen 80;
        listen [::]:80;
        server_name example.jp;
        return 301 https://$host$request_uri;
}

server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        include snippets/snakeoil.conf;

        server_name example.jp;

        root /var/www/html;

        index index.html index.php;

        location / {
                try_files $uri $uri/ =404;
        }

        access_log /var/log/nginx/example.jp-access.log;
        error_log /var/log/nginx/example.jp-error.log;

        location ~ \.php($|/) {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }
}
```
