---
---
# Postfix and Dovecot LMTP

- Postfix is an MTA (Mail Transfer Agent) that sends and receives emails.
- Dovecot is an IMAP server, and it handles local emails handed over from Postfix.

## Install Postfix

```console
sudo apt install postfix postfix-lmdb
```

- postfix-lmdb is a Postfix module to handle LMDB (Lightning Memory-Mapped Database) files, which is [recommended to migrate from Berkeley DB](https://www.postfix.org/NON_BERKELEYDB_README.html).

The installer will ask two questions.

- General mail configuration type: `Internet Site`
- System mail name: `mail.example.jp`
  (The installer will pick up the server FQDN as default)

Open ports for SMTP (25) and SMTP Submission (587).

```console
sudo firewall-cmd --add-service=smtp --permanent
sudo firewall-cmd --add-service=smtp-submission --permanent
sudo firewall-cmd --reload
```

### LMDB

Update `/etc/postfix/main.cf` to use LMDB instead of Berkeley DB (hash, or btree).

```conf
# Update has: to lmdb:
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

```console
sudo usermod -s /usr/sbin/nologin vmail
```

### Configure Virtual Users

The Postfix document has an example of the [Non-Postfix mailbox store: separate domains, non-UNIX accounts](https://www.postfix.org/VIRTUAL_README.html#in_virtual_other), which means using virtual accounts with non-Postfix delivery (in the example; maildrop, in my case; Dovecot) for the virtual domains.

Modify `/etc/postfix/main.cf` to send all emails to the virtual mailbox.

```conf
# Delete domains for the virtual mailbox
mydestination = localhost

# Add virtual mailbox configurations
virtual_mailbox_domains = mail.example.jp, example.jp
virtual_transport = lmtp:unix:private/dovecot-lmtp
virtual_alias_maps = hash:/etc/postfix/virtual
```

To catch local system mails (e.g. cron job), edit `/etc/postfix/virtual`.

```text
admin@mail.example.jp info@mail.example.jp
mailer-daemon@mail.example.jp info@mail.example.jp
postmaster@mail.example.jp info@mail.example.jp
root@mail.example.jp info@mail.example.jp
```

Make `/etc/postfix/virtual` a db file.

```console
sudo postmap virtual
```

Reload Postfix to apply changes on main.cf

```console
sudo systemctl reload postfix
```

## Dovecot-LMTP

### Install Dovecot-LMTPd

Dovecot-LMTP will take over emails from Postfix and deliver them to the final destination directories in `/home/vmail/`. Dovecot enables Sieve filtering and incoming mail indexing for Dovecot IMAP server.

```console
sudo apt install dovecot-lmtpd
```

### Configure

As explained in the [Dovecot document](https://doc.dovecot.org/2.3/configuration_manual/howto/postfix_dovecot_lmtp/), dovecot-lmtpd is integrated with Postfix via the unix scket. (INET is also available.)  
Configure the lmtp section in `/etc/dovecot/conf.d/10-master.conf` to open the socket where Postfix can access.

- The socket must be in /var/spool/postfix because Postfix is chrooted.

```conf
service lmtp {
 unix_listener /var/spool/postfix/private/dovecot-lmtp {
   mode = 0600
   user = postfix
   group = postfix
  }
}
```

Configure `/etc/dovecot/conf.d/10-auth.conf` to choose how to control the user list.

- Comment out auth-system (because mail accounts are isolated from user accounts)
- Uncomment auth-passwdfile (because there are a small number of users that a simple text file is enough to handle)

```conf
#!include auth-system.conf.ext
#!include auth-sql.conf.ext
#!include auth-ldap.conf.ext
!include auth-passwdfile.conf.ext
#!include auth-checkpassword.conf.ext
#!include auth-static.conf.ext
```

Configure passdb and userdb, and set defaults for the userdb information in `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`.

```conf
passdb {
  driver = passwd-file
  args = scheme=CRYPT username_format=%u /etc/dovecot/users
}

userdb {
  driver = passwd-file
  args = username_format=%u /etc/dovecot/users

  # Default fields that can be overridden by passwd-file
  default_fields = uid=vmail gid=vmail home=/home/vmail/%d/%n mail=sdbox:~/dbox

  # Override fields from passwd-file
  #override_fields = home=/home/virtual/%u
}
```

- passdb and userdb can be the same file
- Password scheme is CRYPT (default)
- Username will be full mail address. e.g. `info@example.jp`
- userdb has to have uid, gid, home directory, and email location.
  - Both uid and gid are "vmail" because this server uses virtual users
  - Virtual users home directory is `/home/vmail/domain/username`
  - All users will use Dovecot single-dbox

For more details, see official documents.

- [User Database](https://doc.dovecot.org/2.3/configuration_manual/authentication/user_databases_userdb)
- [Passed-file](https://doc.dovecot.org/2.3/configuration_manual/authentication/passwd_file)

Reload Dovecot to apply new configurations.

```console
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
info@example.jp:{CRYPT}$2y$0...(snip)...Iiy0.::::::
```

The trailing six colons `::::::` are for uid/gid/home/mail_location. Their default values are specified in `/etc/dovecot/conf.d/auth-passwdfile.conf.ext`.  
If everything is written explicitly, the above line should look like this.

```conf
info@example.jp:{CRYPT}$2y$0...(snip)...Iiy0.:vmail:vmail::/home/vmail/%d/%n::userdb_mail=sdbox:~/dbox
```

If you need to change one of these parameters, override the default values by explicitly writing it on the userdb.

Dovecot checks this every time it gets an email. After updating this userdb, no reload or db compile is required.

## Test

Send a test mail to the valid user on this server. The successful logs should be written in `/var/log/mail.log`.

If you need detailed logs, turn on debug switches in `/etc/dovecot/conf.d/10-logging.conf`.

```console
auth_verbose = yes
auth_debug = yes
auth_debug_passwords = yes
mail_debug = yes
```
