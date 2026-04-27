---
---
# DNS for emails

SPF, DKIM, DMARC, and reverse lookup are important for email sending. These require DNS records to work properly.

## SPF

SPF (Sender Policy Framework) declares the specified domain's email should be sent out from which servers.

For example, the domain owner (who can modify DNS records) declare that emails from `@exmaple.jp` should be sent out from the servers listed in `example.jp` MX records.

```conf
example.jp. IN TXT "v=spf1 mx -all"
example.jp. MX 10 mail1.example.jp
example.jp. MX 20 mail2.example.jp
```

- `mx` means the servers listed in MX records will send emails for `@example.jp`.
- `-all` means the servers not listed in the SPF record will not send emails for `@example.jp`.

If you have domains that should not be used for emails, you can declare it.

```conf
nomail.example.jp. IN TXT "v=spf1 -all"
```

## DKIM

See [DKIM signing](./dkim_signing.md) for signing and Rspamd integration.

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

`p=none` will request not to quarantine (do nothing) failed mails.

DMARC can ask/recommend other servers how to handle invalid emails, but the final decision depends on each receiver.

## Reverse lookup

Strict servers may check FQDN -> IP and IP -> FQDN match (e.g., Postfix: reject_unknown_client_hostname).

In short, configure the PTR record to the mail server doamin. Unlike the other DNS records, you can't access PTR record because it's under the control of your server or internet service provider.  
Check if you can change PTR record or use the given FQDN as your mail server name.
