# nginx

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

Gzip compression is turned on by default, but onlyu for the text/html. Enabling compression for all other text type contents will increase the performance.
NOTE: If you care BREACH attacks, do not turn on this compression with SSL/TLS. See [gzip module explanation](https://nginx.org/en/docs/http/ngx_http_gzip_module.html) for more details.

### Global configuration

If you don't have to consider BREACH attacks, globally turn on gzip. Uncomment all the gzip configurations in `/etc/nginx/nginx.conf`.

```conf
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

Gzip can be enabled within "server" section. Snippets are convinient for this kind of purposes.

Make `/etc/nginx/snippets/gzip.conf` file with configs same as in global config file.

```conf
gzip on;

gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

Include this snippet in the server section.

```conf
server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;

        include snippets/gzip.conf;
        (snip)
}
```

## Site configuration

nginx stores website configuration files in `/etc/nginx/sites-available`. To enable a site, add a symlink to that config file in `/etc/nginx/sites-enabled`. (The same as Apache2.)

For more details about each lines in the configuration, please refer [official manuals](https://nginx.org/en/docs/http/ngx_http_core_module.html).

### The simplest example

The simplest example:

- Domain name is `example.jp`
- An HTTP site (non-SSL/TLS)
- Providing static files in `/var/www/html`
- Listening both IPv4 and IPv6
- Logs in `/var/log/nginx/exmaple-jp-*`

```config
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

Redirect the contents in the specific location to PHP fpm to execute PHP scripts.

```config
server {
        listen 80;
        listen [::]:80;

        server_name example.jp;

        root /var/www/html;

        **# Add index.php if required**
        index index.html;

        location / {
                try_files $uri $uri/ =404;
        }

        access_log /var/log/nginx/example.jp-access.log;
        error_log /var/log/nginx/example.jp-error.log;

        # Pass PHP scripts to FastCGI server
        location ~ \.php$ {
                include snippets/fastcgi-php.conf;

                # With php-fpm (or other unix sockets)
                fastcgi_pass unix:/run/php/php-fpm.sock;
        }
}
```
