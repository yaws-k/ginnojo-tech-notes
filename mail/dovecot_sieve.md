---
---
# Dovecot Sieve

Sieve is a script to manages email delivery within the mailbox.  
e.g. An email with `X-Spam: Yes` header will be delivered to `Junk` folder.

WARNING: Dovecot has breaking changes in the configuration between 2.3 and 2.4. Be sure to update the configuration files according to the version you are using.
{: .notice--warning}

## Install

```bash
sudo apt install dovecot-sieve dovecot-managesieved
```

- dovecot-sieve: Sieve plugin for Dovecot LMTP
- dovecot-managesieved: It enables per-user Sieve script management

Open the port for managesieved

```bash
sudo firewall-cmd --add-service=managesieve --permanent
sudo firewall-cmd --reload
```

## Configure

Edit `/etc/dovecot/conf.d/20-lmtp.conf` mail_plugins line to enable Sieve plugin.

```conf
protocol lmtp {
  mail_plugins {
    sieve = yes
  }
}
```

Reload dovecot.

```bash
sudo systemctl reload dovecot
```

## Editing Sieve scripts

As described above, each user can manage their scripts.

- [Sieve Editor](https://github.com/thsmi/sieve/releases): A standalone Sieve Editor
- [Roundcube](https://roundcube.net/): An webmail system with the built-in plugin to manage Sieve scripts

Sieve script examples

- [Pigeonhole Sieve examples](https://doc.dovecot.org/configuration_manual/sieve/examples/)
- [A quick guide on HowtoForge](https://www.howtoforge.com/sieve_mail_filtering)

### Sieve Editor notice

- Make a script and "activate" it to apply the rule

### Dovecot dbox notice

The directory separator differs between Maildir and dbox (Dovecot sdbox/mdbox).

- Maildir: ".(period)"
- dbox: "/"

For example, `folder01` under `INBOX` location is

Maildir style:

```conf
require "fileinto";
if header :contains "from" "folder01@example.com" {
  fileinto "INBOX.folder01";
}
```

dbox style:

```conf
require "fileinto";
if header :contains "from" "folder@example.com" {
  fileinto "INBOX/folder01";
}
```
