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
bind_socket = "localhost:11134";
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

- cf. [Antivirus module](https://docs.rspamd.com/modules/antivirus)

```conf
clamav {
  action = "reject";
  message = '${SCANNER}: virus found: "${VIRUS}"';

  scan_mime_parts = true;

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

## Test scanning

Simply send a legitimate email from outside and check if that reaches the inbox. That email should have Rspamd related headers.

For ClamAV integration testing, [download EICAR test virus](https://www.eicar.org/download-anti-malware-testfile/) and send it from localhost using mutt.

```console
sudo apt install mutt
wget "https://secure.eicar.org/eicar.com"
echo "EICAR test virus" | mutt -a eicar.com -s "Virus scanner test mail `date`" -- info@example.jp
```

You should find the virus detection log;

```text
milter-reject: END-OF-MESSAGE from localhost[127.0.0.1]: 5.7.1 clamav: virus found: "Eicar-Signature"; from=<xxx@example.jp> to=<info@example.jp>
```

## DKIM signing

Rspamd checks DKIM for incoming emails by default. In addition, it can also sign outgoing emails.  
See [DKIM signing module](https://rspamd.com/doc/modules/dkim_signing.html) for details.

This setting is integrated to configwizard. In this case, generate DKIM resources with the following conditions.

- Key for `mail.example.com`
- Different keys for `mail.example.com` and `mail2.example.com`  
  (Not using `example.com` key for multiple subdomains)
- Choose the domain to sign from MIME header "from" address

```conf
sign_authenticated = true;
use_esld = false;
use_domain = "header";
allow_hdrfrom_mismatch = true;
domain {
    mail.example.com {
        path = "/var/lib/rspamd/dkim/mail.example.com.s20241222.key";
        selector = "s20241222";
    }
    mail2.example.com {
        path = "/var/lib/rspamd/dkim/mail2.example.com.s20241222.key";
        selector = "s20241222";
    }
}
allow_username_mismatch = true;
allow_hdrfrom_mismatch_sign_networks = true;
```


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
