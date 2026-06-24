---
---
# ClamAV

ClamAV is an open-source antivirus software engine.

## Install

The package is `clamav`.

```bash
sudo apt install clamav clamav-daemon
```

After installation, `clamav-daemon` automatically starts and may fail. The service must wait until clamav-freshclam downloads the initial database.

ClamAV can connect to Postfix via milter, but I will use this through Rspamd.

## Configuration

Turn off phishing URL detection because Rspamd will take care of this kind of content filtering.  
Update `/etc/clamav/clamd.conf`.

```conf
PhishingScanURLs false
```

After the virus database is ready and config files are updated, start clamav-daemon.

```bash
sudo systemctl start clamav-daemon
```

### Concurrent Database Reload

If OOM killer aborts the database refresh process due to the memory usage, disable "Concurrent Database Reload".  
ClamAV temporarily uses double the memory during the refresh process by default. This configuration disables this behavior, but there will be a little downtime for scanning to replace the virus database.

Add the following line to `/etc/clamav/clamd.conf`.

```conf
ConcurrentDatabaseReload no
```

## Integrate with Rspamd

Integrate ClamAV (clamdscan) with Rspamd to reject viruses.  
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

- It automatically rejects virus-detected emails
- Rspamd log shows the message if any viruses are found
- No headers will be added if the mail is clean

Reload Rspamd.

```bash
sudo systemctl reload rspamd
```

### Scan test virus

[Download EICAR test virus](https://www.eicar.org/download-anti-malware-testfile/) and send it from localhost using mutt.

```bash
sudo apt install mutt
wget "https://secure.eicar.org/eicar.com"
echo "EICAR test virus" | mutt -a eicar.com -s "Virus scanner test mail `date`" -- info@example.jp
```

You should find the virus detection log;

milter-reject: END-OF-MESSAGE from localhost[127.0.0.1]: 5.7.1 clamav: virus found: "Eicar-Signature"; from=<xxx@example.jp> to=<info@example.jp>
{: .notice}
