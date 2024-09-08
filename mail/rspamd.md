# Rspamd

Rspamd is a spam filter that adds 'spam score' to each email.

## Install

Rspamd provides Debian/Ubuntu repository for latest releases. Follow [the official instructions](https://rspamd.com/downloads.html).

```console
sudo apt install -y lsb-release gpg
CODENAME=`lsb_release -c -s`
sudo mkdir -p /etc/apt/keyrings
wget -O- https://rspamd.com/apt-stable/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/rspamd.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main" | sudo tee /etc/apt/sources.list.d/rspamd.list
echo "deb-src [signed-by=/etc/apt/keyrings/rspamd.gpg] http://rspamd.com/apt-stable/ $CODENAME main"  | sudo tee -a /etc/apt/sources.list.d/rspamd.list
sudo apt update
sudo apt --no-install-recommends install rspamd
```

## Postfix integration

Rspamd can communicate Postfix as a milter. Let Postfix send emails to Rspamd by adding milter lines to `/etc/postfix/main.cf`.

```conf
# milter
milter_default_action = accept
smtpd_milters = inet:localhost:11332
```

Restart postfix.

```console
sudo systemctl restart postfix
```

## Configuration

There are config wizard `rspamadm configwizard`.

(Article in progress)
