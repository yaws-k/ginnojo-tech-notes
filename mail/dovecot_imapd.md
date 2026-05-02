---
---
# Dovecot IMAPd

- Dovecot IMAPd manages the mails on the server and responds to MUA.
- Dovecot IMAPd shares the userdb with Dovecot-LMTP.

WARNING: Dovecot has breaking changes in the configuration between 2.3 and 2.4. Be sure to update the configuration files according to the version you are using.
{: .notice--warning}

## Install

Install dovecot-imapd packages and open IMAPS (993) port.

- Open IMAP (143) port if you need STARTTLS

```bash
sudo apt install dovecot-imapd
sudo firewall-cmd --add-service=imaps --permanent
sudo firewall-cmd --reload
```

## SSL/TLS certificate

Set proper certificate in `/etc/dovecot/conf.d/10-ssl.conf`.

```conf
ssl_cert = </etc/letsencrypt/live/example.jp/fullchain.pem
ssl_key = </etc/letsencrypt/live/example.jp/privkey.pem
```

Then reload Dovecot.

```bash
sudo systemctl reload dovecot
```

Now you should be able to connect to the mailbox from your local mailer (MUA), but probably it's better to set up SMTP before testing. (The mailer may require the valid SMTP server to send emails.)

### Certificate rotation

As explained in [Let's Encrypt certificate rotation](../web/lets_encrypt.md), Let's Encrypt can reload applications after the certificate is renewed.  
Add the following line to `/etc/letsencrypt/renewal-hooks/deploy/reload_services.sh`.

```bash
systemctl reload dovecot
```

## Default folders

Special folders other than INBOX (Sent, Drafts, Junk, Trash) should be created by default.  
Modify `/etc/dovecot/conf.d/15-mailboxes.conf` to create the folders.

- Add `auto = subscribe` to automatically create and subscribe the folder.

```conf
namespace inbox {
  # These mailboxes are widely used and could perhaps be created automatically:
  mailbox Drafts {
    special_use = \Drafts
    auto = subscribe
  }
  mailbox Junk {
    special_use = \Junk
    auto = subscribe
  }
  mailbox Trash {
    special_use = \Trash
    auto = subscribe
  }

  # For \Sent mailboxes there are two widely used names. We'll mark both of
  # them as \Sent. User typically deletes one of them if duplicates are created.
  mailbox Sent {
    special_use = \Sent
    auto = subscribe
  }
  mailbox "Sent Messages" {
    special_use = \Sent
  }
  (snip)
}
```

Define filder separator as `/` in `/etc/dovecot/conf.d/10-mail.conf`.  
(If you plan to use multi-byte character for folder name, set `mailbox_list_utf8 = yes`.)

```conf
namespace inbox {
  # Add this to use UTF-8 for folder names on the server side
  mailbox_list_utf8 = yes

  (snip)

  # Hierarchy separator to use. You should use the same separator for all
  # namespaces or some clients get confused. '/' is usually a good one.
  # The default however depends on the underlying mail storage format.
  separator = /

  (snip)
}
```
