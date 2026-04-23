---
---
# SMTP Auth

Postfix accepts relaying (sending out) emails only from the localhost (e.g., cron job). Authorization mechanisms are required for mailbox users to send out their emails.

- Postfix can ask Dovecot to verify users.
- For this purpose, port 465 (submissions, formerly SMTPS) is used
  - Port 25 (SMTP) is often blocked by internet providers (OP25B) that users can't connect to the mail server.
  - Port 587 (submission port) was used according to the RFC 6409 released in 2011, but RFC 8314 released in 2018 recommends using port 465 for submissions.

## SMTP TLS

Let Postfix use the proper server certificate to encrypt the connection. Change the test certificate in `/etc/postfix/main.cf` to Let's Encrypt ones.

```conf
# SMTP server RSA key and certificate in PEM format
smtpd_tls_key_file = /etc/letsencrypt/live/example.jp/fullchain.pem
smtpd_tls_cert_file = /etc/letsencrypt/live/example.jp/privkey.pem
smtpd_tls_security_level=may
```

## SMTP Auth configuration

### Dovecot side

Uncomment "# Postfix smtp-auth" section in `/etc/dovecot/conf.d/10-master.conf`.

```conf
service auth {
  (snip)
  # Postfix smtp-auth
  unix_listener /var/spool/postfix/private/auth {
    mode = 0600
    user = postfix
    group = postfix
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

- [Postfix SASL Howto](https://www.postfix.org/SASL_README.html) explains basic and configuration examples.

Update `/etc/postfix/main.cf`.

- Comment out Cyrus SASL configurations
- Add Dovecot SASL configurations

```conf
# Comment out Cyrus SASL configurations
#cyrus_sasl_config_path = /etc/postfix/sasl

# Add SASL configurations
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

Enable submissions (NOT submission, with an "s"), section in `/etc/postfix/master.cf`.

```conf
submissions inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submissions
  -o smtpd_forbid_unauth_pipelining=no
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o local_header_rewrite_clients=static:all
  -o smtpd_hide_client_session=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=$mua_client_restrictions
  -o smtpd_helo_restrictions=$mua_helo_restrictions
  -o smtpd_sender_restrictions=$mua_sender_restrictions
  -o smtpd_relay_restrictions=$mua_relay_restrictions
  -o smtpd_recipient_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

- As the submissions port is not for the normal mail transfer from other servers;
  - The connection requires tls encryption
  - No relaying permitted unless authenticated
- $mua_..._restrictions will be defined later

Reload Postfix

```console
sudo systemctl reload postfix
```

Now, you should be able to connect to the server from your mailer.  
Go to the next step to reject malicious connection attempts.
