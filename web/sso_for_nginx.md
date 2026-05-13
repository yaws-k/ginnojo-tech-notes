---
---
# SSO for nginx

[Vouch Proxy](https://github.com/vouch/vouch-proxy) works as an authentication gateway for Nginx.  
This provides an easy and reliable way to add the authentication mechanism to any web application.

## System requirement

- Nginx with "auth_request" module  
  Any Nginx flavor of Debian package should have this module.
- Quay.io  
  Vouch Proxy is unavailable as a Debian package, but it provides the docker image through Quay.io.

## The case

- The application is running on `app.example.jp`
- Vouch Proxy server is on `vouch.example.jp` (the same server as `app.example.jp`)
  - Vouch Proxy runs as a docker image listening port 9090
- The application accepts HTTPS
- Use Google Account as the IdP
- Allow only `user1@gmail.com`

## Configuration

### Nginx as a reverse proxy for Vouch Proxy

Create `vouch.example.jp` configuration in `/etc/nginx/sites-available/` and enable it. Nginx works as a reverse proxy to pass the connection to Vouch Proxy.

```conf
server {
        listen 443 ssl;
        listen [::]:443 ssl;
        http2 on;

        server_name vouch.example.jp;

        ssl_certificate /etc/letsencrypt/live/vouch.example.jp/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/vouch.example.jp/privkey.pem;

        # Proxy to your Vouch instance
        location / {
                proxy_pass        http://localhost:9090;
                proxy_set_header  Host $host;
        }
}
```

### Vouch Proxy configuration file

Create `/etc/vouch/config/config.yml`.

- This is based on the [config example for Google](https://github.com/vouch/vouch-proxy/blob/master/config/config.yml_example_google).
- For each configuration explanation, refer to the [config example](https://github.com/vouch/vouch-proxy/blob/master/config/config.yml_example).

```yaml
vouch:
  domains:
    - example.jp
  whitelist:
    - user1@gmail.com

  cookie:
    secure: true

oauth:
  provider: google
  # get credentials from...
  # https://bash.developers.google.com/apis/credentials
  client_id: xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com
  client_secret: xxxxxxxxxxxxxxxxxxxxxxxx
  callback_urls:
    - https://vouch.example.jp/auth
  preferredDomain: gmail.com
```

- On the Google Cloud bash, set the "Authorized redirect URIs" to `https://vouch.example.jp/auth` to match the `callback_urls` in the config file.

### Configure Docker composer

Create `/etc/vouch/docker-compose.yml` to configure the docker image.

```yaml
services:
  vouch-proxy:
    image: quay.io/vouch/vouch-proxy:latest
    container_name: vouch-proxy
    restart: always
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - /etc/vouch/config:/config:ro
    environment:
      - VOUCH_CONFIG=/config/config.yml
    networks:
      - vouch_network

networks:
  vouch_network:
    driver: bridge
```

- `ports` connects the server port 9090 (127.0.0.1) to the container port 9090
- `volumes` mounts the host directory `/etc/vouch/config` to the container directory `/config` as read-only (ro)
- `environment` and `networks` are optional, but specifying them to be sure

## Start Vouch Proxy

```bash
sudo docker compose -f /etc/vouch/docker-compose.yml up -d
```

Check the logs to confirm that Vouch Proxy is running.

```bash
sudo docker compose -f /etc/vouch/docker-compose.yml logs -f
```

### Upgrade image

To upgrade the vouch-proxy image, pull the latest image and restart the container.

```bash
sudo docker compose -f /etc/vouch/docker-compose.yml pull
sudo docker compose -f /etc/vouch/docker-compose.yml up -d
sudo docker compose -f /etc/vouch/docker-compose.yml prune -f
```

## Configure the web application

Let Nginx use auth_request to redirect the connection to Vouch Proxy.

### Prepare the snippe for authentication

Make a snippet `/etc/nginx/snippets/vouch.conf`

```nginx
# send all requests to the `/validate` endpoint for authorization
auth_request /validate;

location = /validate {
        internal;
        # forward the /validate request to Vouch Proxy
        proxy_pass http://127.0.0.1:9090/validate;
        proxy_set_header Host $host;

        # Vouch Proxy only acts on the request headers
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";

        auth_request_set $auth_user $upstream_http_x_vouch_user;

        # these return values are used by the @error401 call
        auth_request_set $auth_resp_jwt $upstream_http_x_vouch_jwt;
        auth_request_set $auth_resp_err $upstream_http_x_vouch_err;
        auth_request_set $auth_resp_failcount $upstream_http_x_vouch_failcount;
}

# if validate returns `401 not authorized` then forward the request to the error401block
error_page 401 = @error401;

location @error401 {
        return 302 https://vouch.example.jp/login?url=$scheme://$host$request_uri&vouch-failcount=$auth_resp_failcount&X-Vouch-Token=$auth_resp_jwt&error=$auth_resp_err;
}
```

### Configure the application site

Set up a site for `app.example.com` in `/etc/nginx/sites-available`

```nginx
server {
        (snip)

        # Enable authentication via Vouch-Proxy
        include snippets/vouch.conf;

        # In this case, assume the app is based on php.
        location ~ \.php {
                include snippets/fastcgi-php.conf;
                fastcgi_pass unix:/var/run/php/php-fpm.sock;

                # This line is required to get the user name in PHP
                fastcgi_param REMOTE_USER $auth_user;
        }

        # For the TCP proxy case (with user name: $auth_user)
        #location / {
        #  proxy_set_header Remote-User $auth_user;
        #  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        #  proxy_set_header X-Forwarded-Proto $scheme;
        #  proxy_set_header X-Forwarded-Ssl on;
        #  proxy_set_header X-Forwarded-Port $server_port;
        #  proxy_set_header Host $http_host;
        #  proxy_pass http://127.0.0.1:8080;
        #}
}
```

- User information is available as an HTTP header. Use the following codes to get them from the web app.
  - PHP: `$_SERVER['REMOTE_USER']`
  - Ruby on Rails: `request.env["HTTP_REMOTE_USER"]`

Reload Nginx to enable the site configurations.

```bash
sudo systemctl reload nginx
```

## Tips

### Logs for troubleshooting

Nginx access log is available at `/var/log/nginx`

Vouch Proxy log is in the image.

```bash
sudo docker compose -f /etc/vouch/docker-compose.yml logs -f
```

If you need debug log for Vouch Proxy, set vouch.jwt.logLevel to `debug` in config.yml

### Less frequent authentication

If you feel there are too frequent redirections to the OIDC provider, you can extend the expiration of jwt.  
Extend `vouch.jwt.maxAge` and `vouch.cookie.maxAge`.  
In my case, it's 900 minutes (15 hours) to aim for the "once a day on the business hours".

```yaml
vouch:
  cookie:
    maxAge: 900
  jwt:
    maxAge: 900
```

## Other cases

### allowAllUsers

Allow all users with the specified mail domains. Suitable for the intranet site for the Google Workspace users.

- Application: `app.example.com`
- Vouch Proxy: `vouch.example.com`
- Users' mail addresses domain: `@workmail.example.jp`

`config.yml` should be

```yaml
vouch:
  allowAllUsers: true

  cookie:
    domain: example.jp
    secure: true

oauth:
  provider: google
  # get credentials from...
  # https://bash.developers.google.com/apis/credentials
  client_id: xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com
  client_secret: xxxxxxxxxxxxxxxxxxxxxxxx
  callback_urls:
    - https://vouch.example.jp/auth
  preferredDomain: workmail.example.jp
```

### AzureAD config sample

To use AzureAD, ask AAD admins to register the web application and receive client_id and client_secret.

This is what I did years ago, so the configuration may be outdated.
{: .notice--warning}

```yaml
vouch:
  allowAllUsers: true

  cookie:
    domain: example.com
    secure: true

oauth:
  provider: azure
  client_id: aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
  client_secret: bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb
  auth_url: https://login.microsoftonline.com/cccccccccccccccccccccccccccccc/oauth2/v2.0/authorize
  token_url: https://login.microsoftonline.com/cccccccccccccccccccccccccccccc/oauth2/v2.0/token
  scopes:
    - openid
    - email
    - profile
  callback_url: https://vouch.example.com/auth
  azure_token: id_token # access_token and id_token supported
```
