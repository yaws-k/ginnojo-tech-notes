---
---
# Knot Resolver

{% include version_warning.html %}

Rspamd (a spam filter) [strongly recommends your own recursive resolver](https://rspamd.com/doc/faq.html#resolver-setup). Knot Resolver is one of the modern DNS resolvers (another option is Unbound).

## Install

[Knot-Resolver official installation instruction](https://www.knot-resolver.cz/documentation/stable/quickstart-install.html) might be outdated. Follow [the official repos for Debian/Ubuntu](https://pkg.labs.nic.cz/doc/?project=knot-resolver) instruction.

```console
sudo apt install apt-transport-https ca-certificates wget
sudo wget -O /usr/share/keyrings/cznic-labs-pkg.gpg https://pkg.labs.nic.cz/gpg
echo "deb [signed-by=/usr/share/keyrings/cznic-labs-pkg.gpg] https://pkg.labs.nic.cz/knot-resolver bookworm main" | sudo tee /etc/apt/sources.list.d/cznic-labs-knot-resolver.list
sudo apt install knot-resolver
```

## Configuration

Knot-Resolver default config `/etc/knot-resolver/kresd.conf` doesn't need any changes. It listens to 53(DNS) and 853(TLS) and accepts access only from localhost.  
(Firewall also rejects access to DNS.)

This Knot-Resolver is for Rspamd. Other services continue using the provider's DNS server on this server.  
There's no need to change `/etc/resolv.conf` unless you have issues with your provider's DNS, or public DNS (8.8.8.8/8.8.4.4 by Google or 1.1.1.1 by Cloudflare)

## Start one instance

Spin up one instance according to [the startup instruction](https://www.knot-resolver.cz/documentation/stable/quickstart-startup.html).  
Remember to enable it, or it won't start automatically after the next system startup.

```console
sudo systemctl start kresd@1
sudo systemctl enable kresd@1
```

Check if Knot-Resolver returns the same answer as the default (your service provider's) DNS.

```console
$ dig a.root-servers.net
(snip)
;; ANSWER SECTION:
a.root-servers.net.     72939   IN      A       198.41.0.4

;; Query time: 0 msec
;; SERVER: (your provider)#53
```

```console
$ dig a.root-servers.net @localhost
(snip)
;; ANSWER SECTION:
a.root-servers.net.     72939   IN      A       198.41.0.4

;; Query time: 0 msec
;; SERVER: ::1#53
```

This is the minimun configuration for Knot-Resolver, but enough to use as a dedicated resolver for Rspamd.  
Please refar official documents for more details and complicated use cases.
