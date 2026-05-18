---
---
# Rspamd

Rspamd is a spam filter that adds "spam score" to each email. In addition, it can integrate ClamAV (antivirus) and DKIM signing.

## Install

Rspamd provides Debian/Ubuntu repository for latest releases. Follow [the official installation Guide for Ubuntu/Debian](https://docs.rspamd.com/getting-started/installation#ubuntudebian).

Prerequisites (should be already done for other software).

```bash
sudo apt install gpg
sudo mkdir -p /etc/apt/keyrings
```

Add key.  
(The following instruction stores the GPG key in `/etc/apt/keyrings` as recommended by Debian.)

```bash
curl -fsSL https://rspamd.com/apt-stable/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/rspamd.gpg
```

Create `/etc/apt/sources.list.d/rspamd.sources`.

```conf
Types: deb
URIs: http://rspamd.com/apt-stable/
Suites: trixie
Components: main
Signed-By: /etc/apt/keyrings/rspamd.gpg
```

Install Rspamd:

```bash
sudo apt update
sudo apt install rspamd
```

## First Setup

Start configuration by following [the official first setup guide](https://docs.rspamd.com/getting-started/first-setup). (Some steps can be skipped because defaults are set out of the box.)

Create `/etc/rspamd/local.d/redis.conf` to connect to Redis for Rspamd prepared in the [Redis article](../database/redis.md).

```conf
# Port 6380 is an instance dedicated for Rspamd
servers = "127.0.0.1:6380";
timeout = 1s;
```

Get the encrypted password for Rspamd web UI.

```console
$ rspamadm pw
Enter passphrase: [password]
$2$j...yfeb
```

Create `/etc/rspamd/local.d/worker-controller.inc` and add the password line.

```conf
password = "$2$j...yfeb";
```

Create `/etc/rspamd/local.d/options.inc` to use Knot-Resolver as a primary DNS resolver.

```conf
dns {
  nameserver = ["127.0.0.1"];
}
```

### Check if Rspamd is working

Check if configutest returns `syntax OK`.

```bash
sudo rspamadm configtest
```

Restart Rspamd and check the status.

```bash
sudo systemctl restart rspamd
sudo systemctl status rspamd
```

Scan test messages.

```console
$ echo -e "Subject: Test\n\nThis is a test message" | rspamc -h [::1]:11333
Results for file: stdin (0.076 seconds)
[Metric: default]
Action: add header
Spam: true
Score: 10.40 / 15.00
Symbol: ARC_NA (0.00)
Symbol: DMARC_NA (0.00)[No From header]
Symbol: HFILTER_HOSTNAME_UNKNOWN (2.50)
Symbol: MIME_GOOD (-0.10)[text/plain]
Symbol: MIME_TRACE (0.00)[0:+]
Symbol: MISSING_DATE (1.00)
Symbol: MISSING_FROM (2.00)
Symbol: MISSING_MID (2.50)
Symbol: MISSING_TO (2.00)
Symbol: MISSING_XM_UA (0.00)
Symbol: ONCE_RECEIVED (0.00)
Symbol: RCVD_COUNT_ZERO (0.00)[0]
Symbol: R_DKIM_NA (0.00)
Symbol: R_MISSING_CHARSET (0.50)
Symbol: SINGLE_SHORT_PART (0.00)
Message-ID: undef
```

## Postfix integration

Rspamd can communicate Postfix as a milter. Let Postfix send emails to Rspamd by adding milter lines to `/etc/postfix/main.cf`.

```conf
# Rspamd milter
smtpd_milters = inet:localhost:11332
non_smtpd_milters = inet:localhost:11332
milter_default_action = accept
```

Reload postfix.

```bash
sudo systemctl reload postfix
```

### Add mail headers

Add extra mail headers to check if Rspamd is working as expected.  
Create `/etc/rspamd/local.d/milter_headers.conf`

```conf
extended_spam_headers = true;
```

Reload Rspamd.

```bash
sudo systemctl reload rspamd
```

### Test scanning

Simply send a legitimate email from outside and check if that reaches the inbox. That email should have Rspamd related headers.

## Web UI

Rspamd has a built-in Web UI. Set Nginx as a reverse-proxy to connect localhost:11334 to access from the internet.

According to the [FAQ: How do I run the WebUI behind a proxy](https://docs.rspamd.com/faq/#how-do-i-run-the-webui-behind-a-proxy), add following lines to nginx configuration.

```nginx
server_name rspamd.example.jp;

location / {
        proxy_pass http://localhost:11334/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For "";
}
```

`https://rspamd.example.jp/` will show the Rspamd Web UI and ask for the password.

- If multiple users need to access the Web UI but you don't want to shere the password, consider using [SSO for nginx](../web/sso_for_nginx).

## Redis memory limit

Set the memory limit for Redis.  
Update `/etc/redis/redis-rspamd.conf` and add following lines.  
(See [Redis article](../database/redis.md)).

```conf
maxmemory 256mb
maxmemory-policy volatile-ttl
```

Restart Redis.

```bash
sudo systemctl restart redis-server@rspamd
```
