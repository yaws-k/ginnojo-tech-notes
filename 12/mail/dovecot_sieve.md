---
---
# Dovecot Sieve

{% include version_warning.html %}

Sieve is a script that manages delivery within the mailbox.

## Install

Sieve is a server-side script, but this should be set on a per-user basis.  
dovecot-managesieved enables users to manage their scripts.

```console
sudo apt install dovecot-sieve dovecot-managesieved
```

- dovecot-sieve: Sieve plugin for Dovecot LMTP
- dovecot-managesieved: It enables per-user Sieve script management

Open the port for managesieved

```console
sudo firewall-cmd --add-service=managesieve --permanent
sudo firewall-cmd --reload
```

## Configure

Edit `/etc/dovecot/conf.d/20-lmtp.conf` mail_plugins line to enable Sieve plugin.

```config
protocol lmtp {
  # Space separated list of plugins to load (default is global mail_plugins).
  mail_plugins = $mail_plugins sieve
}
```

Reload dovecot.

```console
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

### Dovecot sdbox notice

The directory separator differs between Maildir and sdbox (Dovecot original style).

- Maildir: ".(period)"
- sdbox: "/"

For example, `folder01` under `INBOX` location is

Maildir tyle:

```conf
require "fileinto";
if header :contains ["from"] "folder01@example.com" {
  fileinto "INBOX.folder01";
}
```

sdbox style:

```conf
require "fileinto";
if header :contains ["from"] "folder@example.com" {
  fileinto "INBOX/folder01";
}
```
