---
---
# SSO with Nginx and Vouch Proxy

{% include version_warning.html %}

[Vouch Proxy](https://github.com/vouch/vouch-proxy) works as an authentication gateway for Nginx.  
This provides an easy and reliable way to add the authentication mechanism to any web application.

## System requirement

- Nginx with "auth_request" module  
  Any Nginx flavor of Debian package should have this module.
- Quay.io  
  Vouch Proxy is unavailable as a Debian package, but it provides the docker image through Quay.io.

## The case

- The application is running on `app.example.com`
- Vouch Proxy server is on `vouch.example.com`
  - Vouch Proxy runs as a docker image listening port 9090
- The application accepts HTTPS
- Use Google Account as the IdP

## Configuration

### Nginx as a reverse proxy for Vouch Proxy

Make a site configuration in `/etc/nginx/sites-available/` and enable it. Nginx works as a reverse proxy to pass the connection to Vouch Proxy.

```conf
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name vouch.example.com;

  # Enable SSL
  include snippets/ssl_example.com.conf;

  # Proxy to your Vouch instance
  location / {
    proxy_set_header  Host vouch.example.com;
    proxy_set_header  X-Forwarded-Proto https;
    proxy_pass        http://localhost:9090;
  }
}
```

### Vouch Proxy configuration file

Make `/etc/vouch/config/config.yml`. This is based on the [config examples](https://github.com/vouch/vouch-proxy/blob/master/config/config.yml_example_google).

```yaml
# Vouch Proxy configuration
# bare minimum to get Vouch Proxy running with google

vouch:
  domains:
  - example.com

  # set allowAllUsers: true to use Vouch Proxy to just accept anyone who can authenticate with Google
  allowAllUsers: true

  cookie:
    # allow the jwt/cookie to be set into http://yourdomain.com (defaults to true, requiring https://yourdomain.com) 
    secure: false
    # vouch.cookie.domain must be set when enabling allowAllUsers
    domain: example.com


oauth:
  provider: google
  # get credentials from...
  # https://console.developers.google.com/apis/credentials
  client_id: xxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.apps.googleusercontent.com
  client_secret: xxxxxxxxxxxxxxxxxxxxxxxx
  # Google may require callback_urls (redirect URIs) to be 'https'
  callback_urls: 
    - https://vouch.example.com/auth
  preferredDomain: example.com # be careful with this option, it may conflict with chrome on Android 
  # endpoints are set from https://godoc.org/golang.org/x/oauth2/google
```

### Download latest Vouch Proxy image

Pull the docker image from Quay.io and run it with the config file.

```bash
docker run -d \
-p 9090:9090 \
--name vouch-proxy \
-v /etc/vouch/config:/config \
quay.io/vouch/vouch-proxy
```

- `-d` to use detached mode (run in the background)
- `-p 9090:9090` to connect port 9090 on the server to the docker image port 9090. If you want to use port 1090 on the server, the option will be `-p 1090:9090`
- `--name vouch-proxy` to use a short name for this image
- `-v /etc/vouch/config:/config` to let the image use `/etc/vouch/config` host directory as `/config` directory

### Stop and Start the image

The short name `vouch-proxy` should work now.

```bash
docker stop vouch-proxy
docker start vouch-proxy
```

You need to restart the image to reload config.yml file.

### Remove image

When upgrading, remove the image to get the latest one.

```bash
docker rm vouch-proxy
docker rmi quay.io/vouch/vouch-proxy
```

## Configure the web application

Let Nginx use auth_request to redirect the connection to Vouch Proxy.

### Prepare the snippe for authentication

Make a snippet `/etc/nginx/snippets/vouch.conf`

```conf
# Any request to this server will first be sent to this URL
auth_request /vouch-validate;

# Get the authorized user name (email address)
auth_request_set $auth_user $upstream_http_x_vouch_user;

location = /vouch-validate {
  internal;
  # This address is where Vouch will be listening on
  proxy_pass http://127.0.0.1:9090/validate;
  proxy_pass_request_body off; # no need to send the POST body

  proxy_set_header Content-Length "";
  proxy_set_header Host $http_host; # This is required according to the Vouch-Proxy official example
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;

  # these return values are passed to the @error401 call
  auth_request_set $auth_resp_jwt $upstream_http_x_vouch_jwt;
  auth_request_set $auth_resp_err $upstream_http_x_vouch_err;
  auth_request_set $auth_resp_failcount $upstream_http_x_vouch_failcount;
}

error_page 401 = @error401;

# If the user is not logged in, redirect them to Vouch's login URL
location @error401 {
  return 302 https://vouch.example.com/login?url=$scheme://$http_host$request_uri&vouch-failcount=$auth_resp_failcount&X-Vouch-Token=$auth_resp_jwt&error=$auth_resp_err;
}
```

### Configure the application site

Set up a site for `app.example.com` in `/etc/nginx/sites-available`

```conf
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  server_name app.example.com;

  # Enable SSL
  include snippets/ssl_example.com.conf;

  # The location and index files depend on your server policy
  root /var/www/app.example.com;
  index index.php;

  # Enable authentication via Vouch-Proxy
  include snippets/vouch.conf;

  # In this case, assume the app is based on php.
  location ~ \.php($|/) {
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

### Enabale configurations

Reload Nginx to enable the site configurations.

```bash
sudo systemctl reload nginx
```

## Tips

### Logs for troubleshooting

Nginx access log is available at `/var/log/nginx`

Vouch Proxy log is in the image.

```bash
docker logs vouch-proxy
```

`tail` like command is,

```bash
docker logs --follow vouch-proxy
```

If you need debug log for Vouch Proxy, set vouch.jwt.logLevel to `debug` in config.yml

### allowAllUsers

The default configuration offers to specify the domains.

For example,

- Application: app.example.com
- Vouch Proxy: vouch.example.com
- Users' mail addresses domain: `@mail.example.jp`

`config.yml` should be

```yaml
vouch:
  domains:
  - example.com
  - example.jp
```

If the users' mail addresses have a variety of domains, you have to list all of them to permit access.  
This style works when you need to restrict users based on mail domains.

Instead of listing the domains, `allowAllUsers: true` is an option to accept anybody who is authenticated on the IdP side.  
Even with this style, you still need to specify which domain to use for the cookies. This will be the domain to be protected, i.e., callback and application domain.

```yaml
vouch:
  allowAllUsers: true
  cookie:
    domain: example.com
```

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

### AzureAD config sample

To use AzureAD, ask AAD admins to register the web application and receive client_id and client_secret.

```yaml
# Vouch Proxy configuration
# bare minimum to get Vouch Proxy running with Azure AD
# https://github.com/vouch/vouch-proxy/issues/290

vouch:
  # set allowAllUsers: true to use Vouch Proxy to just accept anyone who can authenticate to Azure AD
  allowAllUsers: true

  cookie:
    # allow the jwt/cookie to be set into http://yourdomain.com (defaults to true, requiring https://yourdomain.com) 
    # secure: false
    # vouch.cookie.domain must be set when enabling allowAllUsers
    domain: example.com

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
