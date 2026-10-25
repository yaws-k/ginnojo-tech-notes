---
---
# WAF

CrowdSec (installed in `/base_system/ssh_and_security`) has the WAF (Web Application Firewall) Security Engine.  
It works as an proxy to analyze incoming web traffic and block malicious requests. See [the official explanations](https://docs.crowdsec.net/docs/appsec/intro/).

## nginx setup

- Based on the official documentation [Protect NGINX with IP Blocking, CAPTCHA & WAF](https://docs.crowdsec.net/u/bouncers/nginx/#installation)

Setup nginx to work with lua.

```bash
sudo apt install lua5.1 libnginx-mod-http-lua luarocks gettext-base lua-cjson
sudo apt install crowdsec-nginx-bouncer
```

## Setup collections

- Based on the official documentation [CrowdSec WAF QuickStart for Nginx / OpenResty](https://docs.crowdsec.net/docs/appsec/quickstart/nginxopenresty/)

Install the AppSec rule collections

```bash
sudo cscli collections install \
  crowdsecurity/appsec-virtual-patching \
  crowdsecurity/appsec-generic-rules
```

This will download the rule for known CVEs and common attack patterns. After downloading, it shows `Run 'sudo systemctl reload crowdsec' for the new configuration to be effective.` but wait it till you finish the following steps.

Add `/etc/crowdsec/acquis.d/appsec.yaml` to enable the AppSec acquisition.

```yaml
appsec_configs:
  - crowdsecurity/appsec-default
labels:
  type: appsec
listen_addr: 127.0.0.1:7422
source: appsec
```

Now restart CrowdSec to apply the new configuration.

```bash
sudo systemctl restart crowdsec
```

Set `APPSEC_URL` and change the mode in `/etc/bouncers/crowdsec-nginx-bouncer.conf`.

```ini
MODE=stream
(snip)
APPSEC_URL=http://127.0.0.1:7422
```

Restart nginx.

```bash
sudo systemctl restart nginx
```

## Testing

Access the actual domain that your server provides.

- The example commands on the official documentation tells to access `http://localhost/.env`, but according to the nginx configuration, this will return nothing (HTTP 444).

```bash
curl -I https://your-domain.com/.env
```

It should return `HTTP/2 403` and some additional lines.  
CrowdSec metrics should show there was a blocked request.

```bash
sudo cscli metrics show appsec
```

It should show the number of blocked access in the `Appsec Metrics` table and rules in the `Appsec '127.0.0.1:7422/' Rules Metrics` table.

If you access `.env` file from your browser, it will also return 403 with the message `CrowdSec Access Forbidden`.
