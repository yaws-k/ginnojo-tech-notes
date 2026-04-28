---
---
# nginx

## Install

"nginx" package has some variations according to the bundled modules. Choose either one from nginx-light, nginx-core, nginx-full, nginx-extras. In my case, the lightest nginx-light is enough.

```console
sudo apt install nginx-light ssl-cert
```

Open HTTP port for Nginx.

```console
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --reload
```

## Gzip

Gzip compression is turned on by default, but only for the text/html. Enabling compression for all other text contents will increase performance.

- If your server provides HTML files with user input data and sensitive information, enabling gzip compression may expose your server to BREACH attacks. For more details, see the [gzip module explanation](https://nginx.org/en/docs/http/ngx_http_gzip_module.html).  
  (In most cases, this shouldn't happen with the modern web applications.)

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
        listen ...(snip)

        include snippets/gzip.conf;
        (snip)
}
```

## Site configuration

nginx stores website configuration files in `/etc/nginx/sites-available`. Add a symlink to those config files in `/etc/nginx/sites-enabled` to enable a site. (The same as Apache2.)

For more details about each configuration line, please refer to the [official manual for ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html).

### The simplest example

The simplest example:

- Domain name is `example.jp`
- An HTTP site (non-SSL/TLS)
- Providing static files in `/var/www/html`
- Listening both IPv4 and IPv6
- Logs in `/var/log/nginx/example-jp-*`

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
  (It should be set up when installing php-fpm package.)

Prerequisites

- PHP and fpm have to be installed

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

                # Specify php-fpm socket
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }
}
```

### HTTPS server (with HTTP/2)

Open HTTPS port for Nginx.

```console
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --reload
```

Mozilla provides a ["SSL Configuration Generator"](https://ssl-config.mozilla.org/) to generate recommended server configurations for SSL/TLS.  
This site provides the complete configuration, but let's start from the simplest. (You need a proper certificate to complete.)

Use "snakeoil" testing certificate for SSL/TLS

```nginx
server {
        # Change the port from 80 to 443
        listen 443 ssl;
        listen [::]:443 ssl;

        # Enable http2
        http2 on;

        server_name example.jp;

        # Include snakeoil certificate snippet
        include snippets/snakeoil.conf;

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

Using the proper certificate is explained in the "Let's Encrypt part. The "snakeoil" certificate is only for testing and will cause browser warnings.

### Enable HTTP/3 (QUIC)

Open HTTP/3 port for Nginx.

```console
sudo firewall-cmd --add-service=http3 --permanent
sudo firewall-cmd --reload
```

Enable and tell browsers to use HTTP/3 (QUIC) if supported.

```nginx
server {
        listen 443 ssl;
        listen [::]:443 ssl;
        http2 on;

        # Add HTTP/3 (QUIC) listener
        listen 443 quic reuseport;
        listen [::]:443 quic reuseport;

        # Enable http3
        http3 on;

        # Tell browsers to use HTTP/3 (QUIC) if supported
        add_header Alt-Svc 'h3=":443"; ma=86400';

        server_name example.jp;

        # Include snakeoil certificate snippet
        include snippets/snakeoil.conf;

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

Check if the server provides HTTP/3 as expected with [HTTP/3 Check](https://http3check.net/).

### Redirect HTTP to HTTPS

- Redirect all access to `http://example.jp/` to `https://example.jp/`

```nginx
# Redirect all HTTP (port 80) access to HTTPS (port 443)
server {
        listen 80;
        listen [::]:80;
        server_name example.jp;

        # 308 = Permanent Redirect
        return 308 https://$host$request_uri;
}

server {
        listen 443 ssl;
        listen [::]:443 ssl;
        http2 on;

        listen 443 quic reuseport;
        listen [::]:443 quic reuseport;
        http3 on;

        add_header Alt-Svc 'h3=":443"; ma=86400';

        server_name example.jp;

        include snippets/snakeoil.conf;

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
