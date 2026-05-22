---
---
# Rspamd GPT plugin

Rspamd can send emails to OpenAI's GPT API and get the response.

- Obvious spam and ham will not be sent to avoid wasting the API quota
- You need GPT API key to use this plugin (or own ollama server)

## Configuration

Create `/etc/rspamd/local.d/gpt.conf` and add the following lines.

```conf
enabled = true;
api_key = "<apikey>";
model = "gpt-5.4-mini";
autolearn = true
reason_header = "X-GPT-Reason";

# Add this for testing
allow_ham = true;
```

- Delete `autolearn = true` if you feel GPT result is not so reliable.

Add the following line to `/etc/rspamd/local.d/options.inc` to extend the timeout to wait for the GPT API response.

```conf
# 20 seconds should be enough though...
task_timeout = 30s;
```

Reload Rspamd.

```bash
sudo systemctl reload rspamd
```

## Test GPT plugin

Send a legitimate test mail and check the headers. It should have `X-GPT-Reason` header with the response from GPT API and `GPT_HAM` symbol under `X-Spamd-Result` header.

Remember deleting the `allow_ham = true;` line after testing to avoid wasting the API quota.
{: .notice--warning}
