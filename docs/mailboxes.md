---
title: Mailboxes
nav_order: 1
---

# Mailboxes
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Fully Supported Mailbox providers

### IMAP

Configure IMAP mailbox:
* `Email/Username`: specify IMAP login username.
* `Password / App Password`: specify IMAP password for authentication.
* `Host`: specify IMAP server hostname.
* `Port`: specify IMAP server port.
* `Use SSL/TLS`: enable/disable TLS for IMAP connection.
* `Folder`: specify mailbox folder to monitor (e.g., `INBOX`).

You can use "Auto-discover" feature to automatically fill in IMAP server settings for providers properly configured SRV records in DNS.

### Hosted Mailbox
{: .d-inline-block }

New v.0.10.2
{: .label .label-green }

Paid plans
{: .label .label-yellow }

You can create hosted mailbox directly in MailWebhook system.

Hosted mailbox is a virtual email address hosted by MailWebhook that only task is to receive incoming emails, match them against configured [routes], and forward resulting JSON to configured [endpoints]. You can not connect to hosted mailbox using conventional email clients.

Each mailbox can have an unlimited number of email aliases. Incoming emails sent to any of the aliases will be delivered to the same mailbox. You can use aliases to create more user-friendly email addresses for each individual purpose (i.e. `jira@<domain>`, `cloudflare@<domain>` etc.) to match routes easily.

At least one alias is required to create hosted mailbox.

You can create as much hosted mailboxes as your plan allows. 

Each hosted mailbox allows to define following settings:
* **Aliases**: specify additional email addresses (aliases) for the mailbox. Incoming emails sent to any of the aliases will be delivered to the same mailbox.
* **SPAM scores**:
    * `Reject threshold`: specify SPAM score threshold above which incoming emails will be rejected. Those emails will not be evaluated against any [routes].
    * `Quarantine threshold`: specify SPAM score threshold above which incoming emails will be marked as Quarantine. Those emails will still be evaluated against [routes], but not be delivered to the [endpoint]. You can review quarantined emails in the event log and process them manually using "Replay" feature.
* **Maximum message size (bytes)**: specify maximum allowed size of incoming emails. Emails exceeding this size will be rejected without being evaluated against any [routes].

## OAuth2 Mailboxes

### Workarounds for OAuth2 providers

If you need to connect to mailbox provided by OAuth2 provider (e.g., GMail, Office365), but MailWebhook does not support this provider natively, you can try following workarounds:

1. **Individual user**. As an individual user you have following options:
    1. Create "App Password" for your account in the provider's security settings to allow IMAP access using username and app password. Then use those credentials to connect mailbox using IMAP provider in MailWebhook.
    2. Create email forwarding rule in your email account settings to forward incoming emails to another mailbox which you can connect using IMAP provider in MailWebhook.
    3. Add email forwading from your email account to [hosted mailbox](#hosted-mailbox) in MailWebhook. You will need to setup endpoint to receive verification email from the provider and confirm forwarding.
2. **Organization/Company**. In addition to options for individual users above, as an organization/company you have following additional option:
    1. Create email group/distribution list in your organization email system (e.g., Exchange, GSuite) and add hosted mailbox email address in MailWebhook as a member of that group/list. Incoming emails sent to that group/list will be delivered to hosted mailbox in MailWebhook.

### Gmail
{: .d-inline-block }

New v.0.10.2
{: .label .label-green }

{: .warning }
Experimental support only. Currently Google have not yet approved the app for production use, therefore you will get warning messages from Google when authenticating. **Connected mailboxes may require re-authentication every 14 days.**

Access to an actual email messages requires `restricted` scope approval from Google. This comes with additional review process from Google, which may take several weeks. This process will involve 3rd-party security assessment which will incur additional costs. Whether MailWebhook project will proceed with this process depends on support from users. We do not currently have sufficient number of paying users to cover those costs, but you can sponsor this process  [here](https://buy.stripe.com/bJe14mcMk764148eRucMM00){:target="_blank"}.

### Office365

Coming soon.

## Message preview

For each mailbox you can see JSON output preview for specific message. 
Click "Preview" in action menu of a mailbox to open preview modal.
Select [route] to use for transformation and message from mailbox to preview.

Preview will be available only for messages matching [rules] for the selected [route].

You can click "Reply" button to run specific message through the [route] and 
see resulting JSON delivered to the [endpoint] linked to selected [route].

{: .note }
Limitation: no attachments are included in preview.

## Message backfill

Email backfill feature allows you to re-ingest email messages received before mailbox was added to MailWebhook.
This feature is available through the "..." (Actions) menu for supported mailbox providers (IMAP and GMail).

Backfill is available on all plans.

Backfill process runs in the background and automatically pauses on error(s).

You can check status of currently running backfill job through the same mailbox menu.

Only 1 running backfill task is allowed.

You plan level defines maximum number of days which could be backfilled.

Backfilling process is idempotent and does not repeat eamils which already been delivered to registered webhooks.

[Route]: {% link docs/routes.md %}
[Endpoint]: {% link docs/endpoints.md %}
[Rules]: {% link docs/routes/rules.md %}
[Routes]: {% link docs/routes.md %}
[Endpoints]: {% link docs/endpoints.md %}
