---
---
# DNS for emails

SPF, DKIM, DMARC, and reverse lookup are important for email sending. These require DNS records to work properly.

## SPF

SPF (Sender Policy Framework) specifies which servers are authorized to send emails for a specified domain.

For example, the domain owner (who can modify DNS records) declares that emails from `@example.jp` should be sent out from the servers listed in `example.jp` MX records.

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

See [DKIM signing](./dkim_signing) for signing and Rspamd integration.

## DMARC

DMARC records advise receiving mail servers what to do when both SPF and DKIM verification fail. The example below recommends handling failed emails as spam.

```conf
_dmarc.example.jp. IN TXT "v=DMARC1; p=quarantine"
```

`_dmarc.example.jp` config will cover all subdomains. If you need to change DMARC config according to subdomains, add a dedicated record for that subdomain.

```conf
_dmarc.example.jp. IN TXT "v=DMARC1; p=quarantine"
_dmarc.another.example.jp. IN TXT "v=DMARC1; p=none"
```

`p=none` will request not to quarantine (do nothing) failed emails.

DMARC can ask/recommend other servers how to handle invalid emails, but the final decision depends on each receiver.

## Reverse lookup

Strict servers may check FQDN -> IP and IP -> FQDN match (e.g., Postfix: reject_unknown_client_hostname).

In short, configure a PTR record for your mail server domain. Unlike the other DNS records, you can’t access the PTR record because it’s under the control of your server or Internet Service Provider (ISP).
Check if you can change PTR record or use the given FQDN as your mail server name.

## Checking your emails

There are some online tools to check if your configuration is correct and emails are recognized as legitimate.

- [mail tester: SPF and DKIM check](https://www.mail-tester.com/spf-dkim-check)
- [MX Toolbox: DKIM and DMARC check](https://mxtoolbox.com/dkim.aspx)
- [DKIM Validator: Email Validator](https://dkimvalidator.com/)
