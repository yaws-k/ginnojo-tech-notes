---
---
# SMTP Auth

{% include version_warning.html %}

Postfix accepts relaying (sending out) emails only from the localhost (e.g., cron job). Authorization mechanisms are required for mailbox users to send out their emails.

- Postfix can ask Dovecot to verify users.
- For this purpose, port 587 (submission port) is often used because port 25 (SMTP) is often blocked by internet providers (OP25B).

## SMTP TLS

Let Postfix use the proper server certificate to encrypt the connection. Change the test certificate in `/etc/postfix/main.cf` to Let's Encrypt ones.

```conf
# TLS parameters
smtpd_tls_cert_file=/etc/letsencrypt/live/example.jp/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/example.jp/privkey.pem
smtpd_tls_security_level=may
```

## SMTP Auth configuration

### Dovecot side

Uncomment "# Postfix smtp-auth" section in `/etc/dovecot/conf.d/10-master.conf`.

```conf
service auth {
  (snip)
  # Postfix smtp-auth # Uncomment following lines
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
  }

  # Auth process is run as this user.
  #user = $default_internal_user
}
```

Restart Dovecot.

```console
sudo systemctl restart dovecot
```

## Postfix SASL

Add SASL configuration to `/etc/postfix/main.cf`.

```conf
# SASL
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_tls_auth_only = yes
```

- `smtpd_tls_auth_only = yes` force tls connection for authentication

Reload Postfix

```console
sudo systemctl reload postfix
```

## Submission port

As exaplained in the top, the port 587 (submission port) should be used.  
Enable submission section in `/etc/postfix/master.cf`.

```conf
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_recipient_restrictions=
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

- As the submission port is not for the normal mail transfer from other servers;
  - The connection requires tls encryption
  - No relaying permitted unless authenticated
- $mua_aaa_restritions will be defined later

Reload Postfix

```console
sudo systemctl reload postfix
```

Now, you should be able to connect to the server from your mailer.  
Go to the next step to reject malicious connection attempts.
