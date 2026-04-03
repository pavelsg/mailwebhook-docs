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

## Fully Supported Mailbox Providers

### Gmail
{: .d-inline-block }

New March 2026
{: .label .label-green }

Configure Gmail mailbox:
* `Connect with Google`: start OAuth2 authorization flow for Gmail or Google Workspace mailbox.
* `Label` (optional): limit monitoring to messages with a specific Gmail label. If not set, MailWebhook watches the default mailbox scope configured by the integration.

### Microsoft 365
{: .d-inline-block }

New March 2026
{: .label .label-green }

Configure Microsoft 365 mailbox:
* `Connect with Microsoft 365`: start OAuth2 authorization flow for Microsoft 365 mailbox.
* `Folder`: specify the mailbox folder to monitor.
* `Shared mailbox email` (optional): specify shared mailbox address when the target mailbox is not the authenticated user's primary inbox.

Microsoft 365 mailboxes include Outlook 365 work mailboxes used inside Microsoft 365 organizations.

### Loopback
{: .d-inline-block }

New March 2026
{: .label .label-green }

{: .note }
`loopback` is a temporary mailbox type for onboarding and testing.

Loopback mailbox behavior:
* MailWebhook creates the mailbox address automatically during onboarding.
* Temporary address is ready to receive test emails immediately.
* Loopback mailboxes are not intended for long-term production use.
* Loopback mailboxes stay out of normal mailbox lists.
* Loopback mailboxes do not count against normal mailbox limits.

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

## Other OAuth Providers

If you need to connect a mailbox provided by an OAuth2-based email service that MailWebhook does not yet support natively, you can use one of the following workarounds:

1. **Individual user**. As an individual user you have following options:
    1. Create "App Password" for your account in the provider's security settings to allow IMAP access using username and app password. Then use those credentials to connect mailbox using IMAP provider in MailWebhook.
    2. Create email forwarding rule in your email account settings to forward incoming emails to another mailbox which you can connect using IMAP provider in MailWebhook.
    3. Add email forwarding from your email account to [hosted mailbox](#hosted-mailbox) in MailWebhook. You will need to set up an endpoint to receive the verification email from the provider and confirm forwarding.
2. **Organization/Company**. In addition to options for individual users above, as an organization/company you have following additional option:
    1. Create email group/distribution list in your organization email system (e.g., Exchange, GSuite) and add hosted mailbox email address in MailWebhook as a member of that group/list. Incoming emails sent to that group/list will be delivered to hosted mailbox in MailWebhook.

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

Backfill lets you re‑ingest email that arrived **before** you connected the mailbox to MailWebhook.

- Where: open the mailbox "..." (Actions) menu for supported providers (IMAP, Gmail).
- How: set the slider to the number of days you want to go back in time, hit "Start", done.
- Behavior: jobs run in the background, pause automatically on errors, and are idempotent (already delivered emails are skipped).
- Availability: included on all plans; one backfill job can run at a time.
- Scope: your plan level caps how many days can be backfilled.
- Monitoring: check the status from the same mailbox menu.

[Route]: {% link docs/routes.md %}
[Endpoint]: {% link docs/endpoints.md %}
[Rules]: {% link docs/routes/rules.md %}
[Routes]: {% link docs/routes.md %}
[Endpoints]: {% link docs/endpoints.md %}
