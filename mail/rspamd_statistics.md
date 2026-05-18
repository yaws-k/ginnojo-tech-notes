---
---
# Statistics (Bayesian filter)

Statistics is enabled by default, but it needs to learn before working.  
Without enough learning, Rspamd skips the Bayesian filter with the following logs.

bayes_classify: not classified as ham. The ham class needs more training samples. Currently: 0; minimum 200 required
{: .notice}

## Bayes expiry module

According to [Rspamd statistic setting](https://docs.rspamd.com/configuration/statistic/), create `/etc/rspamd/local.d/classifier-bayes.conf` to specify what to learn, and `expire` for [Bayes expiry module](https://docs.rspamd.com/modules/bayes_expiry).

```conf
expire = 8640000;

autolearn {
  spam_threshold = 6.0;
  junk_threshold = 4.0;
  ham_threshold = -0.5;
  check_balance = true;
}
```

Reload Rspamd.

```bash
sudo systemctl reload rspamd
```

## Autolearn

Rspamd automatically learns the obvious ham/spam.  
(See [Autolearn configuration](https://docs.rspamd.com/configuration/statistic/) for more details.)

rspamd_stat_check_autolearn: \<mail id\>: autolearn ham for classifier 'bayes' as message's score is negative: -4.80
{: .notice}

If you have enough number of incoming emails, Rspamd start using bayesian filter after it learns more than 200 emails for both ham and spam.

## Manual learn

If you already have ham or spam emails, you can let Rspamd learn from them.

Steps:  
(See [doveadm-search](https://doc.dovecot.org/2.4.1/core/man/doveadm-search.1.html) manual for samples and details.)

1. Extract ham/spam emails from dbox
2. Save emails as `.eml` file
3. Learn extracted ham/spam emails

Create the following shellscript.

```bash
mailbox="INBOX"
user="user@example.jp"

if [ -d /tmp/eml ]; then
  rm -rf /tmp/eml
fi
mkdir /tmp/eml

doveadm search -u "$user" mailbox "$mailbox" ALL
while read guid uid; do
  doveadm fetch -u "$user" text mailbox "$mailbox" uid "$uid" > "/tmp/eml/${mailbox}_${uid}.eml"
done
```

Execute this script as root:

```bash
# bash ./extract.sh
```

Execute Rspamd learn command.

```bash
rspamc learn_ham /tmp/eml
```

Extract spam emails and learn them with the following command (if you have spam mails).

```bash
rspamc learn_spam /tmp/eml
```

Don't forget to cleanup the emails you extracted.

```bash
sudo rm -r /tmp/eml
```
