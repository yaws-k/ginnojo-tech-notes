---
---
# DNS records

{% include version_warning.html %}

In addition to DKIM _domainkey DNS record, there are several required records for the mail sending.

## SPF

The SPF record declares which servers emails are sent out. This will help receivers to check if emails are sent from valid servers.

For example, if emails from `@example.jp` should be sent out only from the servers listed in `example.jp` MX records, SPF and MX records should look like the following:

```conf
example.jp. IN TXT "v=spf1 mx -all"
example.jp. MX 10 mail1.example.jp
example.jp. MX 20 mail2.example.jp
```

If you have domains that will not send emails, declare that to protect those domains.

```conf
nomail.example.jp. IN TXT "v=spf1 -all"
```

## DMARC

DMARC records will advise other mail servers what to do when both SPF and DKIM verification fail. The example below recommends handling failed emails as spam.

```conf
_dmarc.example.jp. IN TXT "v=DMARC1; p=quarantine"
```

`_dmarc.example.jp` config will cover all subdomains. If you need to change DMARC config according to subdomains, add a dedicated record for that subdomain.

```conf
_dmarc.example.jp. IN TXT "v=DMARC1; p=quarantine"
_dmarc.another.example.jp. IN TXT "v=DMARC1; p=none"
```

`p=none` will request not to quarantine failed mails.

DMARC can ask/recommend other servers how to handle invalid emails, but the final decision depends on each server.

## Reverse lookup

Strict servers may check if FQDN -> IP and IP -> FQDN match to accept emails (e.g., Postfix: reject_unknown_client_hostname). This should match if possible.

In short, configure the PTR record to the mail server doamin. Unlike the other DNS records, you can't access PTR record because it's under the control of your server or internet service provider.  
Check if you can change PTR record or use the given FQDN as your mail server name.
