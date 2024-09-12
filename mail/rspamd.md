# Rspamd

Rspamd is a spam filter that adds 'spam score' to each email.

## Install

Rspamd provides Debian/Ubuntu repository for latest releases. Follow [the official instructions](https://rspamd.com/downloads.html).

```console
sudo apt install -y lsb-release gpg
CODENAME=`lsb_release -c -s`
sudo mkdir -p /etc/apt/keyrings
wget -O- https://rspamd.com/apt-stable/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/rspamd.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main" | sudo tee /etc/apt/sources.list.d/rspamd.list
echo "deb-src [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main"  | sudo tee -a /etc/apt/sources.list.d/rspamd.list
sudo apt update
sudo apt --no-install-recommends install rspamd
```

## Postfix integration

Rspamd can communicate Postfix as a milter. Let Postfix send emails to Rspamd by adding milter lines to `/etc/postfix/main.cf`.

```conf
# milter
milter_default_action = accept
smtpd_milters = inet:localhost:11332
```

Restart postfix.

```console
sudo systemctl restart postfix
```

## Dedicated Redis instance

As described in the Redis article, create a dedicated instance for Rspamd.  
In this case, assume the port is 6378

## Configure Rspamd

There are config wizard `rspamadm configwizard`.

```console
sudo rspamadm configwizard
```

```console
  ____                                     _
 |  _ \  ___  _ __    __ _  _ __ ___    __| |
 | |_) |/ __|| '_ \  / _` || '_ ` _ \  / _` |
 |  _ < \__ \| |_) || (_| || | | | | || (_| |
 |_| \_\|___/| .__/  \__,_||_| |_| |_| \__,_|
             |_|

Welcome to the configuration tool
We use /etc/rspamd/rspamd.conf configuration file, writing results to /etc/rspamd
Modules enabled: once_received, force_actions, forged_recipients, chartable, multimap, whitelist, emails, regexp, arc, hfilter, settings, metadata_exporter, dmarc, mid, elastic, asn, rbl, maillist, milter_headers, trie, dkim_signing, dkim, mime_types, bayes_expiry, spf, fuzzy_check, phishing
Modules disabled (explicitly): rspamd_update, spamtrap, gpt, mx_check, aws_s3, http_headers, dcc, p0f, bimi, external_relay, known_senders
Modules disabled (unconfigured): metric_exporter, spamassassin, ip_score, clustering, reputation, antivirus, external_services, dynamic_conf, fuzzy_collect, maps_stats, clickhouse
Modules disabled (no Redis): greylist, url_redirector, ratelimit, neural, replies, history_redis
Modules disabled (experimental):
Modules disabled (failed):
Do you wish to continue?[Y/n]: Y
Setup WebUI and controller worker:
Controller password is not set, do you want to set one?[Y/n]: Y
Enter passphrase: [password]
Set encrypted password to: $2...ksfy
Redis servers are not set:
The following modules will be enabled if you add Redis servers:
        * greylist
        * url_redirector
        * ratelimit
        * neural
        * replies
        * history_redis
Do you wish to set Redis servers?[Y/n]: Y
Input read only servers separated by `,` [default: localhost]: localhost:6378
Input write only servers separated by `,` [default: localhost:6378]: (Enter)
Do you have any username set for your Redis (ACL SETUSER and Redis 6.0+)[y/N]: N
Do you have any password set for your Redis?[y/N]: N
Do you have any specific database for your Redis?[y/N]: N
Do you want to setup dkim signing feature?[y/N]: N
cannot connect to redis /run/redis/redis-server-rspamd.sock (port 0): No such file or directory
cannot connect to redis (OS error): No such file or directory
cannot create redis connection: bad arguments for redis request
Cannot connect to Redis server
File: /etc/rspamd/local.d/redis.conf, changes list:
read_servers => localhost:6378
write_servers => localhost:6378

File: /etc/rspamd/local.d/worker-controller.inc, changes list:
password => $2...ksfy

Apply changes?[Y/n]: Y
Create file /etc/rspamd/local.d/redis.conf
Create file /etc/rspamd/local.d/worker-controller.inc
2 changes applied, the wizard is finished now
*** Please reload the Rspamd configuration ***
```

Reload Rspamd

```console
sudo systemctl reload rspamd
```

Now Rspamd start using several information to determine if it should accept, soft reject, or reject incoming emails.

## Add mail headers

Add extra mail headers to check if Rspamd is working as expected.  
Create `/etc/rspamd/local.d/milter_headers.conf`

```conf
extended_spam_headers = true;
```

Reload Rspamd.

```console
sudo systemctl reload rspamd
```

## Integrate with ClamAV

Integrate ClamAV(clamdscan) with Rspamd to check virus.  
Create `/etc/rspamd/local.d/antivirus.conf`

```conf
clamav {
  symbol = "CLAM_VIRUS";
  type = "clamav";
  servers = "/var/run/clamav/clamd.ctl";
}
```

Reload Rspamd.

```console
sudo systemctl reload rspamd
```
