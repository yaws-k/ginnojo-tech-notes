---
---
# Basic configuration and utilities

{% include version_warning.html %}

## Configure apt-line

`apt` command will get only basic software by default. Add `contrib` and `non-free` to `/etc/apt/sources.list` for more applications.

```config
deb http://ftp.jp.debian.org/debian/ bookworm main contrib non-free non-free-firmware

deb http://security.debian.org/debian-security bookworm-security main contrib non-free non-free-firmware

# bookworm-updates, to get updates before a point release is made;
# see https://www.debian.org/doc/manuals/debian-reference/ch02.en.html#_updates_and_backports
deb http://ftp.jp.debian.org/debian/ bookworm-updates main contrib non-free non-free-firmware
```

- `deb-src` is required only when you want to get sources

After updating apt-line, update&upgrade.

```console
sudo apt update
sudo apt upgrade
```

## Basic utilities

Install basic utilities. What you need will change according to the server usage.

```console
sudo apt install bind9-dnsutils man-db net-tools rsync tmux wget curl
```

- bind9-dnsutils: DNS-related commands (e.g. dig).
- man-db: Provides “man” command
- net-tools: Network-related commands (e.g. netstat).
- rsync: Synchronize files/directories.
- tmux: Terminal multiplexer.
- wget: Downloader
- curl: Data transfer mainly with HTTP(S)

## Programming Languages

Install major programming languages. (They will be required and automatically installed as dependencies.)

### Ruby 3.1

ruby & ruby-dev: ruby-dev will be required when connecting to databases.

```console
sudo apt install ruby ruby-dev
```

### Multiple Ruby versions with rbenv

[rbenv](https://github.com/rbenv/rbenv) manages multiple versions, including the latest. Install required build environments according to [rbenv wiki](https://github.com/rbenv/ruby-build/wiki#suggested-build-environment).
(libreadline6-dev is changed to libreadline-dev)

```console
sudo apt install git
sudo apt install autoconf patch build-essential rustc libssl-dev libyaml-dev libreadline-dev zlib1g-dev libgmp-dev libncurses5-dev libffi-dev libgdbm6 libgdbm-dev libdb-dev uuid-dev
```

Then, use [https://github.com/rbenv/rbenv-installer](https://github.com/rbenv/rbenv-installer) to install rbenv.

Log in as a normal user that you want to install rbenv.

```console
curl -fsSL https://github.com/rbenv/rbenv-installer/raw/HEAD/bin/rbenv-installer | bash
```

```console
Installing rbenv with git...
(snip)
Setting up your shell with `rbenv init bash' ...
writing ~/.bashrc: now configured for rbenv.

All done! After reloading your terminal window, rbenv should be good to go.
```

All set. Re-login to enable rbenv.  
For the ruby installation process, see [rbenv GitHub README](https://github.com/rbenv/rbenv?tab=readme-ov-file#installing-ruby-versions).

### Python 3.11

python3: The package "python" was python2.x and not available anymore.

```console
sudo apt install python3
```

### PHP 8.2

php & php-fpm & php8.2-fpm: Installing only “php” will install apache2 according to the dependency. To use nginx, you have to explicitly choose fpm version.

```console
sudo apt install php php-fpm php8.2-fpm
```

The timezone has to be set to php.ini. Update both cli: `/etc/php/8.2/cli/php.ini` and fpm: `/etc/php/8.2/fpm/php.ini`.

```php
[Date]
; Defines the default timezone used by the date functions
; http://php.net/date.timezone
date.timezone = "Asia/Tokyo"
```

Restart fpm to reload the config.

```console
sudo systemctl reload php8.2-fpm
```

### Java 17

openjdk-17-jre: This is JRE. Install JDK if you plan to develop with Java.

```console
sudo apt install openjdk-17-jre
```

### Rust 1.63

```console
sudo apt install rustc
```

### Perl 5.36

```console
sudo apt install perl
```

### Libraries for each language

Each language offers external modules. Python pip, Ruby gems, PHP pecl, and so on. There are multiple ways to install them, but if you need only a few major modules, they may be available as Debian packages.  
If the packages work, you don't have to consider the version discrepancies between packaged languages and modules.

For example, PHP cURL is available as php-curl package.

## Locales (Languages)

Generate locales if you need to display characters other than English. In my case, I need ja_JP.

```console
sudo dpkg-reconfigure locales
```

You can add any locales as you want. The default locale can also be anything, but English is the most safe choice, as explained at the installation.

## Vim

"Vim" is Vi IMproved. If you struggle with Vi (installed by default), install Vim to enhance simple Vi.

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

" Additional configuration for me
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
        address aaa.bbb.ccc.ddd
        netmask 255.255.254.0
        gateway aaa.bbb.eee.1
        # dns-* options are implemented by the resolvconf package, if installed
        #dns-nameservers 8.8.8.8 8.8.4.4

iface ens3 inet6 static
        address aaaa:bbbb:cccc:dddd:eee:fff:ggg:hhh
        netmask 64
        gateway fe80::1
```

The nameserver information should be on `/etc/resolv.conf`. This file should have what you set during the installation. Add IPv6 DNS server address if you like.

```console
search example.com
nameserver 8.8.8.8
nameserver 8.8.4.4
nameserver aaaa:bbbb:1
```

After changing the configuration, restart networking and check it works.

```console
sudo systemctl restart networking
ip address
ping google.com
```
