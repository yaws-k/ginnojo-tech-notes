---
---
# Rspamd

{% include version_warning.html %}

Rspamd is a spam filter that adds 'spam score' to each email. In addition, it can integrate ClamAV (antivirus) and DKIM signing.

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

## DNS Resolver configuration

Knot-Resolver should be already installed on this server, so let Rspmad use that as a primary DNS resolver. Create `/etc/rspamd/local.d/options.inc` and add DNS lines.

```conf
dns {
  nameserver = "master-slave:127.0.0.1,8.8.8.8";
}
```

- `master-slace` always choose the first DNS server (127.0.0.1) for querying. The second one will be used only when the primary is down. Put a public DNS or your provider's DNS resolver as a backup.

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

Integrate ClamAV(clamdscan) with Rspamd to reject virus.  
Create `/etc/rspamd/local.d/antivirus.conf`

```conf
clamav {
  action = "reject";
  message = '${SCANNER}: virus found: "${VIRUS}"';
  symbol = "CLAM_VIRUS";
  type = "clamav";
  servers = "/var/run/clamav/clamd.ctl";
}
```

- It automatically rejects virus detected emails
- Rspamd log shows the message if any virus are found
- No headers will be added if the mail is clean

Reload Rspamd.

```console
sudo systemctl reload rspamd
```

### Test ClamAV integration

EICAR test virus can be used for this test, but sending a virus mail is very difficult (that's what mail systems should be, though).  

Install mutt (a text-based MUA) and send a virus mail from another server.  
EICAR test virus is available at [the official download site](https://www.eicar.org/download-anti-malware-testfile/).

```console
wget -O eicar.com "https://www.eicar.org/download/eicar-com/(the latest url)"
echo "EICAR test virus" | mutt -a eicar.com -s "Virus scanner test mail `date`" -- info@example.jp
```

## DKIM signing

Rspamd checks DKIM for incoming emails by default. In addition, it can also sign outgoing emails.  
See [DKIM signing module](https://rspamd.com/doc/modules/dkim_signing.html) for details.

This setting is integrated to configwizard. In this case, generate DKIM resources with the following conditions.

- Key for `mail.example.com`
- Different keys for `mail.example.com` and `mail2.example.com`  
  (Not using `example.com` key for multiple subdomains)
- Choose the domain to sign from MIME header "from" address

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
Modules enabled: chartable, whitelist, once_received, force_actions, dkim, hfilter, rbl, phishing, greylist, trie, ratelimit, maillist, asn, history_redis, mid, bayes_expiry, multimap, antivirus, settings, metadata_exporter, milter_headers, emails, neural, mime_types, dkim_signing, arc, fuzzy_check, spf, regexp, replies, dmarc, forged_recipients
Modules disabled (explicitly): rspamd_update, external_relay, mx_check, known_senders, bimi, spamtrap, p0f, gpt, aws_s3, http_headers, dcc  Modules disabled (unconfigured): external_services, spamassassin, ip_score, dynamic_conf, clickhouse, reputation, url_redirector, fuzzy_collect, maps_stats, metric_exporter, clustering, elastic
Modules disabled (no Redis):
Modules disabled (experimental):
Modules disabled (failed):
Do you wish to continue?[Y/n]: y
Setup WebUI and controller worker:
Do you want to setup dkim signing feature?[y/N]: y
How would you like to set up DKIM signing?
1. Use domain from mime from header for sign
2. Use domain from SMTP envelope from for sign
3. Use domain from authenticated user for sign
4. Sign all mail from specific networks

Enter your choice (1, 2, 3, 4) [default: 1]: 1
Do you want to sign mail from authenticated users? [Y/n]: y
Allow data mismatch, e.g. if mime from domain is not equal to authenticated user domain? [Y/n]: y
Do you want to use effective domain (e.g. example.com instead of foo.example.com)? [Y/n]: n
Enter output directory for the keys [default: /var/lib/rspamd/dkim/]:
Enter domain to sign: mail.example.jp
Enter selector [default: dkim]: s20240923
Do you want to create privkey /var/lib/rspamd/dkim/mail.example.jp.s20240923.key[Y/n]: y
You need to chown private key file to rspamd user!!
To make dkim signing working, to place the following record in your DNS zone:
v=DKIM1; k=rsa; p=MIIBIjA(snip)

Do you wish to add another DKIM domain?[y/N]: N
File: /etc/rspamd/local.d/dkim_signing.conf, changes list:
allow_hdrfrom_mismatch_sign_networks => true
allow_username_mismatch => true
sign_authenticated => true
use_esld => true
domain => {[mail.example.jp] = {[selector] = s20240923, [path] = /var/lib/rspamd/dkim/mail.example.jp.s20240923.key}}
use_domain => header
allow_hdrfrom_mismatch => true

Apply changes?[Y/n]: Y
Create file /etc/rspamd/local.d/dkim_signing.conf
1 changes applied, the wizard is finished now
*** Please reload the Rspamd configuration ***
```

As the wizard said, change the owner of the key directory (it is currently owned by root).  
And reload Rspamd.

```console
sudo chown -R _rspamd:_rspamd /var/lib/rspamd/dkim
sudo systemctl reload rspamd
```

Now Rspamd will add DKIM keys to outgoing emails.

FYI: The wizard created `/etc/rspamd/local.d/dkim_signing.conf`

```conf
use_domain = "header";
allow_hdrfrom_mismatch = true;
allow_hdrfrom_mismatch_sign_networks = true;
allow_username_mismatch = true;
domain {
    mail.example.jp {
        selector = "s20240923";
        path = "/var/lib/rspamd/dkim/mail.example.jp.s20240923.key";
    }
}
sign_authenticated = true;
use_esld = false;
```

### DNS record

Add DKIM related records to your DNS records. The key is shown in the wizard above.

```text
s20240923._domainkey.mail  IN  TXT  v=DKIM1; k=rsa; p=MIIBIjA(snip)
```

If you want to test DKIM signatures, add the "t=y" parameter to the DNS record. It means the key is still testing.  
Remember to delete this parameter after you confirm that DKIM is working as expected.

### Add another domain to sign

Generate key pairs for the new domain `mail2.example.jp`

```console
sudo rspamadm dkim_keygen -s 's20240923' -d mail2.example.jp -k /var/lib/rspamd/dkim/mail2.example.jp.s20240923.key
sudo chown -R _rspamd:_rspamd /var/lib/rspamd/dkim
```

Then it will save the private key to Rspam DKIM storage and shows the text for DNS record. Add DNS record and add a config for new domain in `/etc/rspamd/local.d/dkim_signing.conf`

```conf
use_domain = "header";
allow_hdrfrom_mismatch = true;
allow_hdrfrom_mismatch_sign_networks = true;
allow_username_mismatch = true;
domain {
    mail.example.jp {
        selector = "s20240923";
        path = "/var/lib/rspamd/dkim/mail.example.jp.s20240923.key";
    },
    mail2.example.jp {
        selector = "s20240923";
        path = "/var/lib/rspamd/dkim/mail2.example.jp.s20240923.key";
    }
}
sign_authenticated = true;
use_esld = false;
```

Reload Rspamd and it should start signing for new domains.

```console
sudo systemctl reload rspamd
```

## Enable Statistics (Bayesian filter)

Statistics is enabled by default, but it needs to learn before working.  
Create `/etc/rspamd/local.d/classifier-bayes.conf`

```conf
# Configure Bayes classifier to use Redis
servers = "127.0.0.1:6378";
backend = "redis"; # Same as statistic.conf

# Auto-learning
autolearn = true;

# Token expiration
new_schema = true; # Same as statistic.conf
expire = 8640000;
```

- Explicitly configuring some lines the same as defaults to be sure about the requirements

At first the result of statistics may affect too much to the result. Reduce the score to see if it works as expected.  
Create `/etc/rspamd/local.d/groups.conf`

```conf
group  "statistics" {
    symbols = {
        BAYES_SPAM {
            weight = 3.4;
        }
        BAYES_HAM {
            weight = -2;
        }
    }
}
```

After reloading, it should start learning and eventually you'll find BAYES_SPAM and BAYES_HAM headers.

```console
sudo systemctl reload rspamd
```

If you're in a hurry, you can learn spam/ham from local eml files.

```console
sudo rspamc learn_spam spam/*
sudo rspamc learn_ham ham/*
```

You can check the current learning status with rspamc command.

```console
$ rspamc stat

Results for command: stat (0.288 seconds)
(snip)
Statfile: BAYES_SPAM type: redis; length: 0; free blocks: 0; total blocks: 0; free: 0.00%; learned: 376; users: 1; languages: 0
Statfile: BAYES_HAM type: redis; length: 0; free blocks: 0; total blocks: 0; free: 0.00%; learned: 227; users: 1; languages: 0
Total learns: 603
```

## Web UI

Rspamd has a built-in Web UI. Set Nginx as a reverse-proxy to connect localhost:11334 to access from the internet.

According to the [FAQ](https://rspamd.com/doc/faq.html#how-to-use-the-webui-behind-a-proxy-server), add following lines to nginx configuration.

```nginx
location /rspamd/ {
        proxy_pass http://localhost:11334/;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For "";
}
```
