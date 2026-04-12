---
---
# Dovecot IMAPd

{% include version_warning.html %}

Dovecot IMAPd manages the mails on the server and responds to MUA.  
Dovecot IMAPd can share the same userdb and other configurations with Dovecot-LMTP.

## Install

Install dovecot-imapd packages and open IMAPS (993) port.

- Open IMAP (143) port if you need

```console
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

Then restart Dovecot.

```console
sudo systemctl restart dovecot
```
