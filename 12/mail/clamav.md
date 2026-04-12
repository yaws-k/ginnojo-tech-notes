---
---
# ClamAV

{% include version_warning.html %}

ClamAV is anti-virus software.

## Install

The package is "clamav".

```console
sudo apt install clamav clamav-daemon
```

After installation, clamav-daemon automatically starts and may fail. It has to wait till clamav-freshclam downloads the initial database.

Clamav can connect to Postfix via milter, but I will use this through Rspamd.

## Configuration

ClamAV scan sometimes does false positives for Phishing URL detection. In my case, some emails from Amex and Hilton were caught by this filter.  
To turn it off, tweak `/etc/clamav/clamd.conf`.

```config
PhishingScanURLs false
```

After the virus database is ready and config files are updated, start clamav-daemon.

```console
systemctl start clamav-daemon
```

If OOM killer abrts the database refresh process due to the memory usage, add `ConcurrentDatabaseReload no` line to disable "Concurrent Database Reload".  
Clamav temporarily uses double the memory during the refresh process by default. This config will stop that process, but there will be a little downtime for scanning to replace the virus database.

```config
ConcurrentDatabaseReload no
```
