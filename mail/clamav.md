---
---
# ClamAV

ClamAV is anti-virus software.

## Install

The package is `clamav`.

```console
sudo apt install clamav clamav-daemon
```

After installation, `clamav-daemon` automatically starts and may fail. It has to wait till clamav-freshclam downloads the initial database.

Clamav can connect to Postfix via milter, but I will use this through Rspamd.

## Configuration

Turn off phishin URL detection because Rspamd will take care of this kind of content filtering.  
Update `/etc/clamav/clamd.conf`.

```config
PhishingScanURLs false
```

After the virus database is ready and config files are updated, start clamav-daemon.

```console
sudo systemctl start clamav-daemon
```

If OOM killer aborts the database refresh process due to the memory usage, disable "Concurrent Database Reload".  
Clamav temporarily uses double the memory during the refresh process by default. This config will stop that process, but there will be a little downtime for scanning to replace the virus database.

Add the following line to `/etc/clamav/clamd.conf`.

```config
ConcurrentDatabaseReload no
```
