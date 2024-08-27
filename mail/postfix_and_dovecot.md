# Postfix and Dovecot LMTP

Postfix is a major MTA (Mail Transfer Agent) that sends and receives emails. It can save received emails to user directories, but in this case, Postfix hands over all emails to Dovecot via LMTP.  
Dovecot is IMAP server, and it handles local emails efficiently and securely.

## Postfix

### Install

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
