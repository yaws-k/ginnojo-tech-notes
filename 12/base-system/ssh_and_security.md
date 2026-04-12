---
---
# ssh and security

{% include version_warning.html %}

## sudo

Always using root account is not recommended. "sudo" should be used to delegate the privileges to the normal user.

```console
# apt install sudo
# adduser [username] sudo
```

Add specific users to sudo group to enable sudo command.

- If you want to be more restrictive, you can limit the commands available to that user.
- After adding a user to the sudo group, that user has to re-login to enable it.

## Install ssh server

In most cases, the server is located in a secure and isolated location. The most common method of accessing it is via SSH (Secure SHell).

Log in as root, and install ssh.

```console
# apt install ssh
```

The system will install SSH and many more packages that depend on it.

## Set up connection

### Generate key pair

Generate a key pair on the local computer (the computer you mainly use).

```console
$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
```

This will generate the `ed25519` private key and `ed25519.pub` public key pair. Copy the public key's content to the server.

### Set your public key to the server

The SSH should accept user and password authentication for now (SSH default). Log in as a normal user (NOT root) and copy and paste the public key to `~/.ssh/authorized_keys`.

```console
$ mkdir ~/.ssh
$ chmod 700 ~/.ssh
$ nano ~/.ssh/authorized_keys
~~ Copy & Paste your public key (ed25519 is a short key and easy to copy & paste) ~~
$ chmod 600 ~/.ssh/authorized_keys
```

### Check if the key pair works

After storing the public key, log out and try logging in again with the public key authentication.

## Configure ssh server

To edit system configuration, get root privilege.

```console
$ su -
Password: <root password>
#
```

### sshd_config

Configure `/etc/ssh/sshd_config` to prohibit password login.

See sshd_config(5) or [the official document](https://man.openbsd.org/sshd_config) (the official document is the latest version, which is newer than the Debian version.)

The default configuration is restrictive. In short, `PasswordAuthentication yes` should be changed to `no` to reject password authentication.  
Some other configurations should be taken into consideration.

- `#PermitRootLogin prohibit-password`  
  Set "no" or "forced-commands-only" according to the usage.
- `#PasswordAuthentication yes`  
  Set "no" to reject password authentication.
- `KbdInteractiveAuthentication no`  
  Leave this as no. This is explained in the PAM section.
- `UsePAM yes`  
  Leave this as yes. As explained in this configuration, `PasswordAuthentication no` should reject password authentication.

### Restart sshd

After changing sshd_config, restart sshd.

```console
# systemctl restart ssh
```

## firewalld

Debian has been using [nftables](https://wiki.debian.org/nftables) from Buster (Debian 10), and recommends the firewalld on top of it.  
UFW looks easier, but it has issues with docker images. (See details for [docker documents](https://docs.docker.com/network/packet-filtering-firewalls/).)

### Install

```console
# apt install firewalld
```

SSH services are registered by default, so the ssh won't be disconnected after installing this.

### Presets

By default, only SSH (port 22) is open. Presets in `/usr/lib/firewalld/services/` allow you to open more ports for web, mail, and so on.

For example, ssh.xml opens tcp:22.

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>SSH</short>
  <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface, enable this option. You need the openssh-server package installed for this option to be useful.</description>
  <port protocol="tcp" port="22"/>
</service>
```

There is a command to list all presets, but the list is too large and more difficult to read than `ls /usr/lib/firewalld/services/`.

```console
# firewall-cmd --get-services
```

### Opening ports using presets

Pick up the service you want to use and enable it. For example, HTTPS.

```console
# firewall-cmd --add-service=https --zone=public --permanent
# firewall-cmd --reload
```

- The application name is "firewalld" (firewall + d), but the command is "firewall-cmd" without "d" after the firewall.
- `--permanent` is required to set the rules preserved after the firewall reload. Without this parameter, you can test the temporary rules.
- `--zone-public` can be omitted because "public" is the default zone.
- Reload required to enable the new configurations.

### Disabling services

Close the port by disabling the service.

```console
# firewall-cmd --remove-service=https --zone=public --permanent
# firewall-cmd --reload
```

### Complicated patterns

You can manually configure the allowed port and TCP/UDP if you need more complicated patterns or no presets. For more details, please refer to [the official documents](https://firewalld.org/documentation/man-pages/firewall-cmd.html) and other materials.

## CrowdSec

"fail2ban" is a major security tool for rejecting malicious login attempts. CrowdSec is an improved security service. It offers a community (free of charge) version.

### Install Security Engine

To install, curl is required.

```console
# apt install curl
```

Follow the instructions on their [official documents](https://doc.crowdsec.net/docs/getting_started/install_crowdsec/).

```console
# curl -s https://install.crowdsec.net | sh
Detected operating system as debian/12.
(snip)
Installing /etc/apt/sources.list.d/crowdsec_crowdsec.list...

# apt install crowdsec
Reading package lists... Done
(snip)
Get started with CrowdSec:
 * Detailed guides are available in our documentation: https://docs.crowdsec.net
 * Configuration items created by the community can be found at the Hub: https://hub.crowdsec.net
 * Gain insights into your use of CrowdSec with the help of the console https://app.crowdsec.net
You can always run the configuration again interactively by using '/usr/share/crowdsec/wizard.sh -c'
```

The Security Engine starts working by default. Now, it needs remediation components to take actual measures against malicious attempts.

### Install remediation component

The firewall bouncer will work like fail2ban. It adds a blocklist to nftables.

```console
# apt install crowdsec-firewall-bouncer-nftables
```

It will add bunch of ip addresses to nftables. You can check these blocklist with nft command.

```console
# nft list ruleset
```

### Create account to access Console

To use the Web UI, create a CrowdSec account.  
[https://app.crowdsec.net/signup](https://app.crowdsec.net/signup?)

After logging into the console, you can get your key to Enroll the server.

```console
sudo cscli console enroll -e context [enrollment key]
```

Then follow the [official manual](https://doc.crowdsec.net/u/getting_started/post_installation/console) to accept enrollment.  
After restarting the CroudSec service, it will sync with console.

```console
sudo systemctl restart crowdsec
```

Now the console will show statistics of security alerts.  
Turn on notifications as you wish.
