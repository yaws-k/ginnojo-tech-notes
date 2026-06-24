---
---
# Knot Resolver

Rspamd (a spam filter) [requires your own recursive resolver](https://docs.rspamd.com/faq/#how-do-i-configure-dns-resolution). Knot Resolver is one of the modern DNS resolvers (another option is Unbound).

## Install

According to the [Knot Resolver official installation instructions](https://www.knot-resolver.cz/documentation/latest/gettingstarted-install.html), follow [the guide for official Debian/Ubuntu repositories](https://pkg.labs.nic.cz/doc/?project=knot-resolver) to install the package.  
(The following instruction stores the GPG key in `/etc/apt/keyrings` as recommended by Debian.)

```bash
sudo apt install apt-transport-https ca-certificates wget
sudo wget -O /etc/apt/keyrings/cznic-labs-pkg.gpg https://pkg.labs.nic.cz/gpg
```

Create `/etc/apt/sources.list.d/cznic-labs-knot-resolver.sources`.

```conf
Types: deb
URIs: https://pkg.labs.nic.cz/knot-resolver/
Suites: trixie
Components: main
Signed-By: /etc/apt/keyrings/cznic-labs-pkg.gpg
```

Install the package.

```bash
sudo apt update
sudo apt install knot-resolver6
```

## Configuration

Knot Resolver default config `/etc/knot-resolver/config.yaml` doesn't need any changes. It listens on port 53 (DNS) and accepts access only from localhost.  
(Firewall also rejects access to DNS.)

## Check if it works as expected

Check if Knot Resolver returns the same answer as the default (your service provider's) DNS.

```console
$ dig a.root-servers.net
(snip)
;; ANSWER SECTION:
a.root-servers.net.     72939   IN      A       198.41.0.4

;; Query time: 0 msec
;; SERVER: (your provider's server)
```

```console
$ dig a.root-servers.net @localhost
(snip)
;; ANSWER SECTION:
a.root-servers.net.     72939   IN      A       198.41.0.4

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
```

This is the minimum configuration for Knot Resolver, but enough to use as a dedicated resolver for Rspamd.  
Please refer to the official documentation for more details.
