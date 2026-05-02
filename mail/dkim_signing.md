---
---
# DKIM signing

Rspamd checks DKIM for incoming emails by default. In addition, it can also sign outgoing emails.  
See [DKIM signing module](https://rspamd.com/doc/modules/dkim_signing.html) for details.

## Configuration

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

## DKIM keys

Generate DKIM keys.  
ed255519 is recommended as a modern way, but it may not be supported be all mail servers. If you consider compatibility, RSA is a safer choice.

ed25519 key generation

```bash
rspamadm dkim_keygen -s 's20260401' -t ed25519 -k mail.example.jp.s20260401.key > dns-mail.example.jp.txt
```

RSA  key generation

```bash
rspamadm dkim_keygen -s 's20260401' -b 2048 -k mail.example.jp.s20260401.key > dns-mail.example.jp.txt
```

`rspamadmin dkim_keygen` command generates a private key `mail.example.jp.s20260401.key` and DNS record text `dns0mail.example.jp.txt`.  
Move the private key to Rspam DKIM key path and change the owner to `_rspamd` user.

```bash
sudo mv mail.example.jp.s20260401.key /var/lib/rspamd/dkim/
sudo chmod 600 /var/lib/rspamd/dkim/mail.example.jp.s20260401.key
sudo chown _rspamd:_rspamd /var/lib/rspamd/dkim/mail.example.jp.s20260401.key
```

## DNS record

Add DKIM key records to your DNS records.

```conf
s20260401._domainkey.mail  IN  TXT  v=DKIM1; k=ed25519; p=dW...SU="
```

- ed25519 key is very short and everything can be written in one DNS record.
- If you want to test DKIM signatures, add the "t=y" parameter to the DNS record. It means the key is still testing.  
  Remember to delete this parameter after you confirm that DKIM is working as expected.

Reload Rspamd and it should start signing emails.

```bash
sudo systemctl reload rspamd
```
