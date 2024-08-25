# Let's Encrypt

[Let's Encrypt](https://letsencrypt.org/) privides free SSL/TLS certificates. It also provides the official client "Certbot" to create, renew, and revoke certificates.

There are several ways to setup Certbot and plugins.

(1) Certbot only (snap)
If you don't need any plugins, installing Certbot with snap is the easiest way.  
In most cases "[webroot](https://eff-certbot.readthedocs.io/en/stable/using.html#webroot)" should work.

(2) Certbot and plugins that are available as snap apps (snap)
If you need DNS challenges, and using major DNS services (for example Route53) you can use Certbot and plugins both provided as snap apps.
Or, the required plugins are avaialble as snap apps, you can install everything through the snap.
DNS challenges are required when you need a wildcard certificate, or the web server is in the firewall and not accessible from the internet.

(3) Certbot and third-party plugins that don't available as snap app
If any of required plugins are not available as snap apps, you need to install Certbot and plugins through Python pip. How to is explained in the [official explanation](https://certbot.eff.org/).

I used to use a wildcard certificate with Gandi LiveDNS (that requires the most complicated way), but with some tweaks with Nginx I could manage certificates with Certbot only.

## Install Certbot
