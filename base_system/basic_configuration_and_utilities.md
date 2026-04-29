---
---
# Basic configuration and utilities

## Configure apt-line

`apt` command will get only basic software by default. Add `contrib` and `non-free` to `/etc/apt/debian.sources` for more applications.

- As migrated to the new deb822 format at the end of installation, the file looks very different from the old apt-line format.  
  <https://wiki.debian.org/SourcesList#APT_sources_format>

```config
Types: deb
URIs: http://ftp.jp.debian.org/debian/
Suites: trixie
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: http://security.debian.org/debian-security/
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

- `deb-src` is required only if you want to get sources

After updating apt-line, update&upgrade.

```console
sudo apt update
sudo apt upgrade
```

## Snap

Snap is a package management system other than apt. Some applications, such as Certbot, are available through snap. Install and update snapd according to the [official howto for Debian](https://snapcraft.io/docs/tutorials/install-the-daemon/debian/).

```console
sudo apt install snapd
```

Log out and Log in again tp activate the new path, and install the latest snapd with core snap.

```console
sudo snap install snapd
sudo snap install core
```

## Basic utilities

Install basic utilities for server management.

```console
sudo apt install dnsutils man-db net-tools rsync tmux wget curl ca-certificates
```

- dnsutils: DNS-related commands (e.g. dig).
- man-db: Provides “man” command
- net-tools: Network-related commands (e.g. netstat).
- rsync: Synchronize files/directories.
- tmux: Terminal multiplexer.
- wget: Downloader
- curl: Data transfer mainly with HTTP(S)  
  (Should be already installed for CrowdSec)
- ca-certificates: SSL certificates for HTTPS connections

## Programming Languages

Install major programming languages. (They will be required and automatically installed as dependencies.)

### Ruby 3.3

ruby & ruby-dev: ruby-dev will be required when connecting to databases.

```console
sudo apt install ruby ruby-dev
```

#### Multiple Ruby versions with rbenv

System-wide Ruby is usually old and suitable for running applications, but not for development. For development, [rbenv](https://github.com/rbenv/rbenv) will help installing multiple versions (including the latest) into the isolated environment.

As prerequisites, install required build environments according to [rbenv wiki](https://github.com/rbenv/ruby-build/wiki#suggested-build-environment).
(libreadline6-dev is changed to libreadline-dev)

```console
sudo apt install git
sudo apt install autoconf build-essential libffi-dev libgmp-dev libssl-dev libyaml-dev rustc zlib1g-dev
sudo apt install autoconf patch build-essential rustc libssl-dev libyaml-dev libreadline-dev zlib1g-dev libgmp-dev libncurses5-dev libffi-dev libgdbm6 libgdbm-dev libdb-dev uuid-dev
```

Then, use [rbenv installer](https://github.com/rbenv/rbenv-installer) to install rbenv.

Log in as a normal user that you want to install rbenv.

```console
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash
```

The script will install rbenv and configure the shell.

```console
Installing rbenv with git...
(snip)
Setting up your shell with `rbenv init bash' ...
writing ~/.bashrc: now configured for rbenv.

All done! After reloading your terminal window,
rbenv should be good to go.
```

All set. Re-login to enable rbenv, and, for example, install Ruby 3.4.9.

```console
rbenv install 3.4.9
```

It will download the source code, compile, and install it. This may take a while.

See [rbenv GitHub README](https://github.com/rbenv/rbenv?tab=readme-ov-file#installing-ruby-versions) for more details.

### Python 3.13

python3: The package "python" was python2.x and not available anymore.

```console
sudo apt install python3 python3-venv
```

- Python3 should be already installed as a dependency of CrowdSec

For development, venv is useful to create isolated environments.

```console
python3 -m venv directory_name
source directory_name/bin/activate
```

To exit from the venv, just run `deactivate`.

```console
deactivate
```

If you need multiple versions of Python, [pyenv](https://github.com/pyenv/pyenv) will help.

### PHP 8.4

Installing only `php` will install apache2 according to the dependency. To use nginx, you have to explicitly choose fpm version.

```console
sudo apt install php php-fpm php8.4-fpm
```

The timezone has to be set to php.ini. Update both cli: `/etc/php/8.4/cli/php.ini` and fpm: `/etc/php/8.4/fpm/php.ini`.

```php
[Date]
; Defines the default timezone used by the date functions
; http://php.net/date.timezone
date.timezone = "Asia/Tokyo"
```

Restart fpm to reload the config.

```console
sudo systemctl reload php8.4-fpm
```

### Java 21

Headless JRE should be enough for running Java applications.  
Install JDK if you plan to develop with Java.

```console
sudo apt install default-jre-headless
```

### Rust 1.85

```console
sudo apt install rustc
```

- This should be already installed as a dependency of rbenv

### Perl 5.40

```console
sudo apt install perl
```

- This should be already installed

### Libraries for each language

Each language offers external modules. Python pip, Ruby gems, PHP pecl, and so on. There are multiple ways to install them, but if you need only a few major modules, they may be available as Debian packages.  
If the packages work, you don't have to consider the version discrepancies between packaged languages and modules.

For example, PHP cURL is available as php-curl package.

## Docker CE

To use Docker images, install Docker Engine according to the [official howto for Debian](https://docs.docker.com/engine/install/debian/).

- `curl` and `ca-certificates` should be already installed.

```console
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
```

Create `/etc/apt/sources.list.d/docker.sources` to add Docker repository.

```text
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: trixie
Components: stable
Architectures: amd64
Signed-By: /etc/apt/keyrings/docker.asc
```

- Change the `Architectures` line if your architecture is not amd64.

Install Docker Engine.

```console
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Locales (Languages)

Generate locales if you need to display characters other than English. In my case, I need ja_JP.

```console
sudo dpkg-reconfigure locales
```

You can add any locales as you want. The default locale can also be anything, but English is the most safe choice, as explained at the installation.

## Vim

`vim` stands for Vi IMproved. If you decide to use Vi (installed by default), install Vim to enhance simple Vi.

```console
sudo apt install vim
```

Configure `/etc/vim/vimrc` to enable options.

```vim
" Vim5 and later versions support syntax highlighting. Uncommenting the next
" line enables syntax highlighting by default.
syntax on

" If using a dark background within the editing area and syntax highlighting
" turn on this option as well
set background=dark

" Uncomment the following to have Vim jump to the last position when
" reopening a file
"au BufReadPost * if line("'\"") > 1 && line("'\"") <= line("$") | exe "normal! g'\"" | endif

" Uncomment the following to have Vim load indentation rules and plugins
" according to the detected filetype.
filetype plugin indent on

" The following are commented out as they cause vim to behave a lot
" differently from regular Vi. They are highly recommended though.
"set showcmd            " Show (partial) command in status line.
set showmatch           " Show matching brackets.
"set ignorecase         " Do case insensitive matching
"set smartcase          " Do smart case matching
set incsearch           " Incremental search
"set autowrite          " Automatically save before commands like :next and :make
"set hidden             " Hide buffers when they are abandoned
"set mouse=a            " Enable mouse usage (all modes)

" Source a global configuration file if available
if filereadable("/etc/vim/vimrc.local")
  source /etc/vim/vimrc.local
endif

" Additional configuration
set number
set ambiwidth=double
```

## systemd-timesyncd

systemd-timesyncd works like NTP client. Install this if `/etc/systemd/timesyncd.conf` doesn't exist.

```console
sudo apt install systemd-timesyncd
```

It works out of the box by using debian ntp pool servers. If you know better ntp servers (e.g. NTP servers in your network), update `/etc/systemd/timesyncd.conf` to refer them.

```console
[Time]
NTP=ntp.example.com
```

Restart the service.

```console
sudo systemctl restart systemd-timesyncd
```

## IPv6

During the install process, only IPv4 was set. Add IPv6 configurations to `/etc/network/interfaces`.

```console
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug ens3
iface ens3 inet static
        address aaa.bbb.ccc.ddd/23
        gateway aaa.bbb.eee.1
        # dns-* options are implemented by the resolvconf package, if installed
        dns-nameservers a.b.c.d a.b.c.e
        dns-search example.com

iface ens3 inet6 static
        address aaaa:bbbb:cccc:dddd:eee:fff:ggg:hhh/64
        gateway fe80::1
        dns-nameservers aaaa:bbbb:1
```

To update `/etc/resolv.conf`, install `resolvconf`.

```console
sudo apt install resolvconf
```

Restart network and check if it works.

```console
sudo ifdown ens3 && sudo ifup ens3
```

It will return ipv6 error, because IPv6 config is newly added.  
(It's expected.)

```console
RTNETLINK answers: No such process
Error: ipv6: address not found.
Waiting for DAD... Done
```

Check the IP addresses and try pinging via IPv6.

```console
ip address
ping6 google.com
```
