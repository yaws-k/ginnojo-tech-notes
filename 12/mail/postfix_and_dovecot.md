---
---
# Postfix and Dovecot LMTP

{% include version_warning.html %}

Postfix is a major MTA (Mail Transfer Agent) that sends and receives emails. It can save received emails to user directories, but in this case, Postfix hands over all emails to Dovecot via LMTP.  
Dovecot is IMAP server, and it handles local emails efficiently and securely.

## Postfix

### Install Postfix

```console
sudo apt install postfix
```

The installer will ask two questions.

- General mail configuration type: Internet Site
- System mail name: `mail.example.jp`
  (The installer will pick up the server FQDN as default)

Open ports for SMTP (25) and SMTP Submission (587).

```console
sudo firewall-cmd --add-service=smtp --permanent
sudo firewall-cmd --add-service=smtp-submission --permanent
sudo firewall-cmd --reload
```

### Virtual Mailbox

To isolate the email account and Unix user account, set up vmail.

#### Virtual mailbox account

Make a new user "vmail" and store all mails under its home directory `/home/vmail`.  
This account is only for mail storage, so disable shell login for additional security.

```console
$ sudo adduser vmail
Adding user `vmail' ...
Adding new group `vmail' (1001) ...
Adding new user `vmail' (1001) with group `vmail (1001)' ...
Creating home directory `/home/vmail' ...
Copying files from `/etc/skel' ...
New password: 
Retype new password: 
passwd: password updated successfully
Changing the user information for vmail
Enter the new value, or press ENTER for the default
        Full Name []: 
        Room Number []: 
        Work Phone []: 
        Home Phone []: 
        Other []: 
Is the information correct? [Y/n] y
Adding new user `vmail' to supplemental / extra groups `users' ...
Adding user `vmail' to group `users' ...
$ sudo usermod -s /usr/sbin/nologin vmail
```

#### Configure Virtual Users

The Postfix document has an example of the [Non-Postfix mailbox store: separate domains, non-UNIX accounts](https://www.postfix.org/VIRTUAL_README.html#in_virtual_other), which means using virtual accounts with non-Postfix delivery (in the example; maildrop, in my case; Dovecot) for the virtual domains.

Modify `/etc/postfix/main.cf` to send all mails to the virtual mailbox.

```conf
myhostname = mail.example.jp
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = localhost
relayhost = 
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

# Virtual Mailbox
virtual_mailbox_domains = mail.example.jp, example.jp
virtual_transport = lmtp:unix:private/dovecot-lmtp 
virtual_alias_maps = hash:/etc/postfix/virtual
```

- Remember deleting domains for the virtual mailbox from `mydestination` line

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
