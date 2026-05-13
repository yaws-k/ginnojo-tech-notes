---
---
# DKIM signing

Rspamd checks DKIM for incoming emails by default. In addition, it can also sign outgoing emails.  
See [DKIM signing module](https://docs.rspamd.com/modules/dkim_signing/) for details.

ed25519 key is the modern and strong algorithm, but unfortunately, it is not supported by major mail services. For the compatibility, set both ed25519 and RSA 2048bit keys.  
According to [the test by Red Shift](https://redsift.com/blog/ed25519-dkim-support-weak-keys), as of April 2026, Gmail, Microsoft 365, Yahoo, and iCloud do not support ed25519 keys.
{: .notice--info }

## Configuration

Create `/etc/rspamd/local.d/dkim_signing.conf` to override defaults for the following conditions.

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
        selectors [
            { selector: "s20260401" },
            { selector: "s20260401rsa" }
        ]
    }
    mail2.example.jp {
        selectors [
            { selector: "s20260401" },
            { selector: "s20260401rsa" }
        ]
    }
}
```

## DKIM keys

Generate DKIM keys.
ed25519 key is the modern and strong algorithm, but unfortunately, it is not supported by major mail services. For the compatibility, set both ed25519 and RSA 2048bit keys.

ed25519 key generation

```bash
rspamadm dkim_keygen -s 's20260401' -t ed25519 -k mail.example.jp.s20260401.key > mail.example.jp.txt
```

RSA  key generation

```bash
rspamadm dkim_keygen -s 's20260401rsa' -b 2048 -k mail.example.jp.s20260401rsa.key > mail.example.jp.rsa.txt
```

`rspamadm dkim_keygen` command generates a private key `mail.example.jp.s20260401.key` and DNS record text `mail.example.jp.txt`.

- Create `/var/lib/rspamd/dkim` directory with `750` permission and `_rspamd:_rspamd` ownership.
- Move private keys to the above directory with `600` permission and `_rspamd:_rspamd` ownership.

## DNS record

Add DKIM key records to your DNS records.

```conf
s20260401._domainkey.mail  86400  IN  TXT  "v=DKIM1; k=ed25519; p=dW...SU="
s20260401rsa._domainkey.mail  86400  IN  TXT  "v=DKIM1; k=rsa; p=MI...AQAB="
```

- ed25519 key is very short and everything can be written in one DNS record.
- If you want to test DKIM signatures, add the "t=y" parameter to the DNS record. It means the key is still testing.  
  Remember to delete this parameter after you confirm that DKIM is working as expected.
- How to input DNS records slightly differs depending on your DNS provider. Please check what you have to do according to your DNS provider's documentation.

RSA key is too long for one DNS record. To avoid this issue, probably the safest way is to delete spaces and newlines like below before pasting to DNS record.

```console
"v=DKIM1; k=rsa; ""p=MIIB...""abcd..."
```

### Check DKIM record

After adding DKIM records, you can check if they are valid with the sites like [DKIM Record Checker](https://dmarcian.com/dkim-inspector/).

## Start signing

Reload Rspamd and it should start signing emails.

```bash
sudo systemctl reload rspamd
```
