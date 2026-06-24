---
---
# Mail Overview

An email system requires multiple components to work together. Here is the overview of incoming and outgoing mail flows on the server.  
It's too complicated to set up everything at once, so the articles explain step by step, starting from the minimum.

- Some details may be simplified for clarity

## Incoming mail flow

- Scan emails with Rspamd to reject/quarantine spam and viruses.

{% include figure popup=true image_path="/assets/images/incoming_mail.drawio.svg" alt="Incoming Mail Overview" %}

## Outgoing mail flow

- Accept emails from authenticated users only before sending them out
- Add DKIM signatures to outgoing emails

{% include figure popup=true image_path="/assets/images/outgoing_mail.drawio.svg" alt="Outgoing Mail Overview" %}
