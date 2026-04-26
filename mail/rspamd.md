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

### Configuration

Create `/etc/rspamd/local.d/dkim_signing.conf` to enable DKIM signing with the following conditions.

- Different keys for `mail.example.jp` and `mail2.example.jp`  
  (Not using `example.jp` key for multiple subdomains)
- Choose the domain to sign from MIME header "from" address

```conf
# If true, envelope/header domain mismatch is ignored
allow_hdrfrom_mismatch = true;

# If true, domain mismatch is ignored for sign_networks
allow_hdrfrom_mismatch_sign_networks = true;

# If true, username does not need to contain matching domain
allow_username_mismatch = true;

# Whether to normalise domains to eSLD (e.g. example.jp instead of foo.example.jp).
use_esld = false;

# Default path to key, can include '$domain' and '$selector' variables
path = "/var/lib/rspamd/dkim/$domain.$selector.key";

domain {
    mail.example.jp {
        selector = "s20260401";
    }
    mail2.example.jp {
        selector = "s20260401";
    }
}
```

### DKIM keys

Generate DKIM keys.  
ed255519 is recommended as a modern way, but it may not be supported be all mail servers. If you consider compatibility, RSA is a safer choice.

ed25519 key generation

```console
rspamadm dkim_keygen -s 's20260401' -t ed25519 -k mail.example.jp.s20260401.key > dns-mail.example.jp.txt
```

RSA  key generation

```console
rspamadm dkim_keygen -s 's20260401' -b 2048 -k mail.example.jp.s20260401.key > dns-mail.example.jp.txt
```

`rspamadmin dkim_keygen` command generates a private key `mail.example.jp.s20260401.key` and DNS record text `dns0mail.example.jp.txt`.  
Move the private key to Rspam DKIM key path and change the owner to `_rspamd` user.

```console
sudo mv mail.example.jp.s20260401.key /var/lib/rspamd/dkim/
sudo chmod 600 /var/lib/rspamd/dkim/mail.example.jp.s20260401.key
sudo chown _rspamd:_rspamd /var/lib/rspamd/dkim/mail.example.jp.s20260401.key
```

### DNS record

Add DKIM key records to your DNS records.

```text
s20260401._domainkey.mail  IN  TXT  v=DKIM1; k=ed25519; p=dW...SU="
```

- ed25519 key is very short and everything can be written in one DNS record.
- If you want to test DKIM signatures, add the "t=y" parameter to the DNS record. It means the key is still testing.  
  Remember to delete this parameter after you confirm that DKIM is working as expected.

Reload Rspamd and it should start signing emails.

```console
sudo systemctl reload rspamd
```

## Statistics (Bayesian filter)

Statistics is enabled by default, but it needs to learn before working.  
Without enouch learning, Rspamd skips the Bayesian filter.

```text
bayes_classify: not classified as ham. The ham class needs more training samples. Currently: 0; minimum 200 required
```

According to [Rspamd statistic setting](https://docs.rspamd.com/configuration/statistic/), create `/etc/rspamd/local.d/classifier-bayes.conf` to specify what to learn.

```conf
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

```text
rspamd_stat_check_autolearn: <mail id>: autolearn ham for classifier 'bayes' as message's score is negative: -4.80
```

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
