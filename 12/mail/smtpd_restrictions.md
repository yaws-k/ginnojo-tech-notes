---
---
# SMTPd restrictions

{% include version_warning.html %}

There are too many connection attempts to find loose servers for malicios purposes. Postfix has built-in restrictions to reject those connections.

## smtpd_*_restrictions

There are several restriction groups; [Postfix SMTP relay and access control](https://www.postfix.org/SMTPD_ACCESS_README.html).

The example below is a bit more strict than the official example.  
In short,

1. Permit: My networks
2. Permit: Authenticated users
3. Reject: Invalid domain names with helo command
4. Reject: Invalid sender address
5. Reject: Invalid destination domains or mail addresses
6. Permit anything else

Update `/etc/postfix/main.cf`

```conf
# Comment out existing smtpd_relay_restrictions
# (And redifine with other restrictions)
#smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination

(snip)

# Restrictions
message_size_limit = 20480000
disable_vrfy_command = yes

unknown_hostname_reject_code = 554
unknown_address_reject_code = 554
unverified_sender_reject_code = 554
unverified_recipient_reject_code = 554

smtpd_helo_required = yes
strict_rfc821_envelopes = yes

mua_client_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated

mua_helo_restrictions =
    permit_mynetworks,
    reject_invalid_helo_hostname,
    reject_non_fqdn_helo_hostname,
    reject_unknown_helo_hostname

mua_sender_restrictions =
    reject_non_fqdn_sender,
    reject_unknown_sender_domain

mua_relay_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination

mua_recipient_restrictions =
    permit_mynetworks,
    reject_non_fqdn_recipient
    reject_unknown_recipient_domain
    reject_unauth_destination

mua_data_restrictions =
    reject_unauth_pipelining

smtpd_client_restrictions = $mua_client_restrictions
smtpd_helo_restrictions = $mua_helo_restrictions
smtpd_sender_restrictions = $mua_sender_restrictions
smtpd_relay_restrictions = $mua_relay_restrictions
smtpd_recipient_restrictions = $mua_recipient_restrictions
smtpd_data_restrictions = $mua_data_restrictions
```

- message_size_limit: 20MB should be enough (default 10MB)
- disable_vrfy_command: Prevent this command to be used for user scanning.
- *_reject_code: 450 (try later) is the default. Spam servers may repeat retry forever.
- smtpd_helo_required: Yes to make the most use of helo_restrictions.
- strict_rfc821_envelopes: For the later installation of content filter

Reload Postfix to aplly new restrictions.

```console
# sudo systemctl reload postfix
```
