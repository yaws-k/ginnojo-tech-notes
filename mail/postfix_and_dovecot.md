---
---
# Postfix and Dovecot LMTP

- Postfix is an MTA (Mail Transfer Agent) that sends and receives emails.
- Dovecot is an IMAP server, and it handles local emails handed over from Postfix.

WARNING: Dovecot has breaking changes in the configuration between 2.3 and 2.4. Be sure to update the configuration files according to the version you are using.
{: .notice--warning}

## Install Postfix

```bash
sudo apt install postfix postfix-lmdb
```

- postfix-lmdb is a Postfix module to handle LMDB (Lightning Memory-Mapped Database) files, which is [recommended to migrate from Berkeley DB](https://www.postfix.org/NON_BERKELEYDB_README.html).

The installer will ask two questions.

- General mail configuration type: `Internet Site`
- System mail name: `mail.example.jp`
  (The installer will pick up the server FQDN as default)

Open ports for SMTP (25) and Submissions (formerly SMTPS, 465).

```bash
sudo firewall-cmd --add-service=smtp --permanent
sudo firewall-cmd --add-service=smtps --permanent
sudo firewall-cmd --reload
```

### LMDB

Update `/etc/postfix/main.cf` to use LMDB instead of Berkeley DB (hash, or btree).

```conf
# Update hash: to lmdb:
alias_maps = lmdb:/etc/aliases
alias_database = lmdb:/etc/aliases

# Update btree: to lmdb:
smtp_tls_session_cache_database = lmdb:${data_directory}/smtp_scache

# Set default DB to LMDB
default_database_type = lmdb
```

- `default_cache_db_type` cannot be added because it's available from Postfix 3.11 (Debian 13 Postfix is 3.10)

## Virtual Mailbox

To isolate the email accounts and Unix user accounts, set up the virtual mailbox.

### Create Mail Storage User

Make a new user `vmail` and store all mails under its home directory `/home/vmail`.  

```console
$ sudo adduser vmail
New password: <password>
Retype new password: <password>
passwd: password updated successfully
Changing the user information for vmail
Enter the new value, or press ENTER for the default
        Full Name []:
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] y
```

Disable shell login because this account is only for the mail storage.

```bash
sudo usermod -s /usr/sbin/nologin vmail
```

### Configure Virtual Users

The Postfix document has an example of the [Non-Postfix mailbox store: separate domains, non-UNIX accounts](https://www.postfix.org/VIRTUAL_README.html#in_virtual_other).

Modify `/etc/postfix/main.cf` to send all emails to the virtual mailbox.

```conf
# Delete domains for the virtual mailbox
mydestination = localhost

# Add virtual mailbox configurations
virtual_mailbox_domains = mail.example.jp, example.jp
virtual_mailbox_maps = lmdb:/etc/dovecot/users
virtual_transport = lmtp:unix:private/dovecot-lmtp
virtual_alias_maps = lmdb:/etc/postfix/virtual
```

- `virtual_mailbox_maps` will be generated based on the Dovecot user database.

To catch local system mails (e.g. cron job), edit `/etc/postfix/virtual`.

```conf
admin@mail.example.jp info@mail.example.jp
mailer-daemon@mail.example.jp info@mail.example.jp
postmaster@mail.example.jp info@mail.example.jp
root@mail.example.jp info@mail.example.jp
```

Make LMDB file of `/etc/postfix/virtual`.

```bash
sudo postmap virtual
```

Reload Postfix to apply changes on main.cf

```bash
sudo systemctl reload postfix
```

## Dovecot-LMTP

### Install Dovecot-LMTPd

Dovecot-LMTP will take over emails from Postfix and deliver them to the final destination directories in `/home/vmail/`.

```bash
sudo apt install dovecot-lmtpd
```

### Configure Dovecot

As explained in the [Postfix and Dovecot LMTP](https://doc.dovecot.org/2.4.1/howto/lmtp/postfix.html), dovecot-lmtpd is integrated with Postfix via the unix socket.  
Configure the lmtp section in `/etc/dovecot/conf.d/10-master.conf` to open the socket where Postfix can access.

- The socket must be in `/var/spool/postfix` because Postfix is chrooted.

```conf
service lmtp {
 unix_listener /var/spool/postfix/private/dovecot-lmtp {
   mode = 0600
   user = postfix
   group = postfix
  }
}
```

Update Debian defaults in `/etc/dovecot/conf.d/10-mail.conf` to specify the mail defaults.

```conf
# Debian defaults
# Note that upstream considers mbox deprecated and strongly recommends
# against its use in production environments. See further information
# at
# https://doc.dovecot.org/2.4.1/core/config/mailbox/formats/mbox.html

# Update defaults to mdbox (NOT mbox) on virtual mailbox
mail_driver = mdbox
mail_home = /home/vmail/%{user | domain}/%{user | username}
mail_path = %{home}/mdbox
# Comment out (or delete) mail_inbox_path (it's for mbox)
#mail_inbox_path = /var/mail/%{user}

(snip)

# System user and group used to access mails. If you use multiple, userdb
# can override these by returning uid or gid fields. You can use either numbers
# or names. <https://doc.dovecot.org/latest/core/config/system_users.html#uids>
mail_uid = vmail
mail_gid = vmail
```

Comment out `auth_username_format` default in `/etc/dovecot/conf.d/20-lmtp.conf`.

```conf
protocol lmtp {
  #mail_plugins {
  #  sieve = yes
  #}

  # This strips the domain name before delivery, since the default
  # userdb in Debian is /etc/passwd, which doesn't include domain
  # names in the user.  If you're using a different userdb backend
  # that does include domain names, you may wish to remove this.  See
  # https://doc.dovecot.org/2.4.1/howto/lmtp/exim.html and
  # https://doc.dovecot.org/2.4.1/core/summaries/settings.html#auth_username_format
  #auth_username_format = %{user | username | lower}
}
```

Configure `/etc/dovecot/conf.d/10-auth.conf` to choose how to control the user list.

- Comment out auth-system (mail accounts are isolated to the virtual mailbox)
- Uncomment auth-passwdfile (a simple text file is enough to handle users)

```conf
#!include auth-system.conf.ext
#!include auth-sql.conf.ext
#!include auth-ldap.conf.ext
!include auth-passwdfile.conf.ext
#!include auth-checkpassword.conf.ext
#!include auth-static.conf.ext
```

Configure passdb and userdb locations in `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`.

```conf
passdb passwd-file {
  auth_username_format = %{user | lower}
  passwd_file_path = /etc/dovecot/users
}

userdb passwd-file {
  auth_username_format = %{user | lower}
  passwd_file_path = /etc/dovecot/users
}
```

- passdb and userdb can be the same file
- Dovecot will look up the user list with the full email address, with lower case
- Other configurations are left as default or set in the other conf files

For more details, see official documents.

- [Quick Configuration](https://doc.dovecot.org/2.4.1/core/config/quick.html)
- [Password Databases](https://doc.dovecot.org/2.4.1/core/config/auth/passdb.html)
- [Virtual Users with Postfix](https://doc.dovecot.org/2.4.1/howto/virtual/postfix.html)

Reload Dovecot to apply new configurations.

```bash
sudo systemctl reload dovecot
```

## Userdb

The userdb `/etc/dovecot/users` will keep the list of usernames (email addresses) and their encrypted passwords. The command `doveadm` will generate the encrypted password.

```console
# doveadm pw
Enter new password: 
Retype new password: 
{CRYPT}$2y$0...(snip)...Iiy0.
```

Copy and paste the above encrypted password to Userdb as a part of the `info@example.jp` information.

```conf
info@example.jp:{CRYPT}$2y$0...(snip)...W5Qa::::::
```

- The trailing six colons `::::::` are for uid/gid/home/mail_location. All of them are snipped to use the default values.

Dovecot checks this every time it gets an email. After updating this userdb, no reload or db compile is required.

### Userdb for Postfix

Generate postfix lmdb file from this userdb.

Create a new shell script `/etc/dovecot/postfix_lmdb.sh` to generate the lmdb file.

```bash
#!/bin/bash
awk -F: '{print $1 " OK"}' /etc/dovecot/users > /etc/dovecot/users.tmp
postmap lmdb:/etc/dovecot/users.tmp
mv /etc/dovecot/users.tmp.lmdb /etc/dovecot/users.lmdb
rm /etc/dovecot/users.tmp
```

- This lmdb has usernames and dummy `OK` values because it's only used for Postfix to check if the user exists. (It should have the mailbox location.)

Add execute permission to this script.

```bash
sudo chmod +x /etc/dovecot/postfix_lmdb.sh
```

Whenever the userdb is updated, run this script to update the lmdb file for Postfix.

```bash
sudo /etc/dovecot/postfix_lmdb.sh
```

## Test

Follow the logs with `jounalctl` and send a test mail to the valid user on this server.

```bash
sudo jounalctl -f SYSLOG_FACILITY=2
```

If you need detailed logs, turn on debug switches in `/etc/dovecot/conf.d/10-logging.conf`.

```conf
auth_verbose = yes
auth_debug_passwords = yes
```
