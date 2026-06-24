---
---
# SSH and Security

At first, login as `root` to install and configure the basic packages.

## sudo

Using the root account is not recommended. "sudo" should be used to delegate the privileges to the normal user.

```console
# apt install sudo
# adduser [username] sudo
```

Add specific users to the sudo group to enable sudo command.

- If you want to be more restrictive, you can limit the commands available to those users.
- After adding a user to the sudo group, that user must log out and log back in for the changes to take effect.

## Install ssh server

In most cases, the server is located in a secure and isolated location. The most common method of accessing it is via SSH (Secure SHell).

```console
# apt install ssh
```

The system will install SSH and its dependencies.

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
  - Copy & Paste your public key
    (ed25519 is a short key and easy to copy & paste)
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

- NOT `ssh_config` but `sshd_config`. Don't forget the "d" after `ssh`.

See sshd_config(5) or [the official documentation](https://man.openbsd.org/sshd_config) (the official documentation is the latest version, which is newer than the Debian version.)

The default configuration is restrictive. In short, `PasswordAuthentication yes` should be changed to `no` to reject password authentication.  
Some other configurations should be taken into consideration.

- `PermitRootLogin prohibit-password`  
  Set "no" or "forced-commands-only" according to the usage.
- `PasswordAuthentication yes`  
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

SSH services are registered by default, so your SSH connection won't be dropped after installation.

### Presets

Presets in `/usr/lib/firewalld/services/` allow you to open more ports for web, mail, and so on.

For example, `ssh.xml` opens `tcp:22`.

```xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>SSH</short>
  <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface, enable this option. You need the openssh-server package installed for this option to be useful.</description>
  <port protocol="tcp" port="22"/>
</service>
```

There is a command to list all presets, but the list is too large and difficult to read. Probably `ls /usr/lib/firewalld/services/` works better.

```console
# firewall-cmd --get-services
```

### Opening ports using presets

Choose the service you want to use and enable it. For example, HTTPS.

```console
# firewall-cmd --add-service=https --zone=public --permanent
# firewall-cmd --reload
```

- The application name is "firewalld" (firewall + d), but the command is `firewall-cmd` without "d" after the firewall.
- `--permanent` is required to set the rules preserved after the firewall reload. Without this parameter, you can test the temporary rules.
- `--zone=public` can be omitted because "public" is the default zone.
- Reload required to enable the new configurations.

### Disabling services

Close the port by disabling the service.

```console
# firewall-cmd --remove-service=https --zone=public --permanent
# firewall-cmd --reload
```

### Complicated patterns

You can manually configure the allowed port and TCP/UDP if you need more complicated patterns or there is no suitable preset. For more details, please refer to [the official documentation](https://firewalld.org/documentation/man-pages/firewall-cmd.html) and other materials.

Policies or rich rules will be the option for those cases.

## CrowdSec

CrowdSec is a security service. It offers a free community version.

### Install Security Engine

To install, curl is required.

```console
# apt install curl
```

Follow the instructions on their [official documentation](https://doc.crowdsec.net/docs/getting_started/install_crowdsec/).

Update apt-lines.

```console
# curl -s https://install.crowdsec.net | sh
Detected operating system as debian/13.
(snip)
Installing /etc/apt/sources.list.d/crowdsec_crowdsec.list...
```

Install the Security Engine.

```console
# apt install crowdsec
Installing:
  crowdsec
(snip)
You can always run the configuration again interactively by using 'cscli setup'
```

The Security Engine starts working by default. Now, it needs remediation components to take actual measures against malicious attempts.

### Install remediation component

The firewall bouncer will work like fail2ban. It adds a blocklist to nftables.

```console
# apt install crowdsec-firewall-bouncer-nftables
# systemctl reload crowdsec
```

### Create account to access Console

To use the Web UI, create a CrowdSec account.  
[https://app.crowdsec.net/signup](https://app.crowdsec.net/signup?)

After logging into the console, you can get your key to Enroll the server.

```console
# cscli console enroll [enrollment key]
INFO manual set to true
INFO context set to true
INFO Enabled manual : Forward manual decisions to the console
INFO Enabled tainted : Forward alerts from tainted scenarios to the console
INFO Enabled context : Forward context with alerts to the console
INFO Watcher successfully enrolled. Visit https://app.crowdsec.net to accept it.
INFO Please restart crowdsec after accepting the enrollment.
```

Then follow the [official manual](https://doc.crowdsec.net/u/getting_started/post_installation/console) to accept enrollment.  
After restarting the CrowdSec service, it will sync with the console.

```console
# systemctl restart crowdsec
```

Now the console will show statistics of security alerts.  
Turn on notifications as you wish.
