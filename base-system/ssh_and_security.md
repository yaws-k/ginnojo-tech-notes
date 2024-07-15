# ssh and security

## Install ssh server

In most cases the server is located in the secure and isolated places. Accessing via ssh is the most common way.

Log in as root, and install ssh.

``` console
# apt install ssh
```

The system will install ssh and many more packages that depends on.

## Set up connection

### Generate key pair

On the local computer (the computer you mainly use), generate a key pair.

``` console
$ ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
```

This will generate `ed25519` private key and `ed25519.pub` public key pair. Copy the content of public key to the server.

### Set your public key to the server

The ssh should accept user & password authentication. Log in as anormal user (NOT root), and copy&paste the public key to `~/.ssh/authorized_keys`.

``` console
$ mkdir ~/.ssh
$ chmod 700 ~/.ssh
$ nano ~/.ssh/authorized_keys
~~ Copy & Paste your public key (ed25519 is a short key and easy to copy & paste) ~~
$ chmod 600 ~/.ssh/authorized_keys
```

### Check if the key pair works

After storing the public key, log out and try loggin in again with the public key authentication.

## Configure ssh server

To edit system configuration, get root priviledge.

``` console
$ su -
Password: <root password>
#
```

### sshd_config

Configure `/etc/ssh/sshd_config` to prohibit password login.

See sshd_config(5) or [the official document](https://man.openbsd.org/sshd_config) (the official document is the latest version, which is newer than Debian version.)

The default configuration is restrictive. In short, `PasswordAuthentication yes` should be changed to `no` to reject password authentication.  
There are some other configurations that should be taken into considerations.

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

``` console
# systemctl restart ssh
```

## firewalld

Debian has been using [nftables](https://wiki.debian.org/nftables) from Buster (Debian 10), and recommends the firewalld on top of it.  
UFW looks easier, but it has issues with docker images. (See details for [docker documents](https://docs.docker.com/network/packet-filtering-firewalls/).)

### Install

``` console
# apt install firewalld
```

ssh services are registered by default, so the ssh won't be disconnected after installing this.

### Presets

Only ssh (port 22) is open by default. To open more ports for web, mail, and so on, there are presets in `/usr/lib/firewalld/services/`.

For example, ssh.xml opens tcp:22.

``` xml
<?xml version="1.0" encoding="utf-8"?>
<service>
  <short>SSH</short>
  <description>Secure Shell (SSH) is a protocol for logging into and executing commands on remote machines. It provides secure encrypted communications. If you plan on accessing your machine remotely via SSH over a firewalled interface, enable this option. You need the openssh-server package installed for this option to be useful.</description>
  <port protocol="tcp" port="22"/>
</service>
```

There is a command to list all presets, but the list is too large and more difficult to read than `ls /usr/lib/firewalld/services/`.

``` console
# firewall-cmd --get-services
```

### Opening ports using presets

Pick up the service you want to use and enable it. For example, HTTPS.

``` console
# firewall-cmd --add-service=https --zone=public --permanent
# firewall-cmd --reload
```

- The application name is "firewalld" (firewall + d), but command is "firewall-cmd" without "d" after the firewall.
- `--permanent` is required to set the rules preserved after the firewall reload. Without this parameter, you can test the temporary rules.
- `--zone-public` can be omitted because "public" is the default zone.
- Reload required to enable the new configurations.

### Disabling services

Close the port by disabling the service.

``` console
# firewall-cmd --remove-service=https --zone=public --permanent
# firewall-cmd --reload
```

### Complicated patterns

If you need more complicated patterns or no presets available, you can manually configure allowed port and tcp/udp. For more details please refar [the official documents](https://firewalld.org/documentation/man-pages/firewall-cmd.html) and other materials.

## CrowdSec

"fail2ban" is one of the major security tool to reject malicious login attempts. CrowdSec is a kind of improved security service. They offer a community free version.

### Install Security Engine

To install, curl is required.

``` console
# apt install curl
```

Follow the instructions on their [official documents](https://doc.crowdsec.net/docs/getting_started/install_crowdsec/).

``` console
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

The Security Engine start working by default. Now it needs to add remediation component, to take actual measures to malicious attempts.

### Install remediation component

Firewall bouncer will work like fail2ban. It adds a blocklist to nftables.

``` console
# apt install crowdsec-firewall-bouncer-nftables
```
