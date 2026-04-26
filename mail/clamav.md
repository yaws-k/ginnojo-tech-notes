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

### Concurrent Database Reload

If OOM killer aborts the database refresh process due to the memory usage, disable "Concurrent Database Reload".  
Clamav temporarily uses double the memory during the refresh process by default. This config will stop that process, but there will be a little downtime for scanning to replace the virus database.

Add the following line to `/etc/clamav/clamd.conf`.

```config
ConcurrentDatabaseReload no
```

## Integrate with Rspamd

Integrate ClamAV(clamdscan) with Rspamd to reject virus.  
Create `/etc/rspamd/local.d/antivirus.conf`

- cf. [Antivirus module](https://docs.rspamd.com/modules/antivirus)

```conf
clamav {
  action = "reject";
  message = '${SCANNER}: virus found: "${VIRUS}"';

  scan_mime_parts = true;

  symbol = "CLAM_VIRUS";
  type = "clamav";

  servers = "/var/run/clamav/clamd.ctl";
}
```

- It automatically rejects virus detected emails
- Rspamd log shows the message if any virus are found
- No headers will be added if the mail is clean

Reload Rspamd.

```console
sudo systemctl reload rspamd
```

### Test scanning

[Download EICAR test virus](https://www.eicar.org/download-anti-malware-testfile/) and send it from localhost using mutt.

```console
sudo apt install mutt
wget "https://secure.eicar.org/eicar.com"
echo "EICAR test virus" | mutt -a eicar.com -s "Virus scanner test mail `date`" -- info@example.jp
```

You should find the virus detection log;

```text
milter-reject: END-OF-MESSAGE from localhost[127.0.0.1]: 5.7.1 clamav: virus found: "Eicar-Signature"; from=<xxx@example.jp> to=<info@example.jp>
```
