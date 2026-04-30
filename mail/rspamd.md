---
---
# Rspamd

Rspamd is a spam filter that adds 'spam score' to each email. In addition, it can integrate ClamAV (antivirus) and DKIM signing.

## Install

Rspamd provides Debian/Ubuntu repository for latest releases. Follow [the official installation Guide for Ubuntu/Debian](https://docs.rspamd.com/getting-started/installation#ubuntudebian).

Add repository:

```console
sudo apt install gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://rspamd.com/apt-stable/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/rspamd.gpg
echo "deb [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ trixie main" | sudo tee /etc/apt/sources.list.d/rspamd.list
```

Install Rspamd:

```console
sudo apt update
sudo apt --no-install-recommends install rspamd
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

```console
sudo rspamadm configtest
```

Restart Rspamd and check the status.

```console
sudo systemctl restart rspamd
sudo systemctl status rspamd
```

Scan test messages.

```console
echo -e "Subject: Test\n\nThis is a test message" | rspamc -h [::1]:11333
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

```console
sudo systemctl reload postfix
```

### Add mail headers

Add extra mail headers to check if Rspamd is working as expected.  
Create `/etc/rspamd/local.d/milter_headers.conf`

```conf
extended_spam_headers = true;
```

Reload Rspamd.

```console
sudo systemctl reload rspamd
```

### Test scanning

Simply send a legitimate email from outside and check if that reaches the inbox. That email should have Rspamd related headers.

## Web UI

Rspamd has a built-in Web UI. Set Nginx as a reverse-proxy to connect localhost:11334 to access from the internet.

According to the [FAQ: How do I run the WebUI behind a proxy](https://docs.rspamd.com/faq/#how-do-i-run-the-webui-behind-a-proxy), add following lines to nginx configuration.

```nginx
location /rspamd/ {
        proxy_pass http://localhost:11334/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For "";
}
```

`https://sever-name/rspamd/` will show the Rspamd Web UI and ask for the password. If you need more secured access, set up any authentication method in Nginx.

## Statistics (Bayesian filter)

Statistics is enabled by default, but it needs to learn before working.  
Without enouch learning, Rspamd skips the Bayesian filter.

```console
bayes_classify: not classified as ham. The ham class needs more training samples. Currently: 0; minimum 200 required
```

According to [Rspamd statistic setting](https://docs.rspamd.com/configuration/statistic/), create `/etc/rspamd/local.d/classifier-bayes.conf` to specify what to learn, and `expire` for [Bayes expiry module](https://docs.rspamd.com/modules/bayes_expiry).

```conf
expire = 8640000;

autolearn {
  spam_threshold = 6.0;
  junk_threshold = 4.0;
  ham_threshold = -0.5;
  check_balance = true;
}
```

Reload Rspamd.

```console
sudo systemctl reload rspamd
```

Rspamd log should show the learning process.

```console
rspamd_stat_check_autolearn: <mail id>: autolearn ham for classifier 'bayes' as message's score is negative: -4.80
```

## Redis memory limit

Set the memory limit for Redis.  
Update `/etc/redis/redis-rspamd.conf` and add following lines.  
(See [Redis article](../database/redis.md)).

```conf
maxmemory 256mb
maxmemory-policy volatile-ttl
```

Restart Redis.

```console
sudo systemctl restart redis-server@rspamd
```
