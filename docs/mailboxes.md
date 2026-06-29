---
title: Mailboxes
nav_order: 3
has_children: true
description: "Configure MailWebhook mailboxes for Gmail, Microsoft 365, Office 365, Outlook, IMAP, hosted, and loopback email sources before routing messages to webhooks."
---

# Mailboxes
{: .no_toc }

Use mailbox docs when you need to connect an email source before routing messages to a webhook. Choose the provider setup guide that matches the inbox, then use routes and endpoints to control what MailWebhook sends downstream.

## Choose a mailbox source

| Source | Use it when | Setup guide |
| --- | --- | --- |
| Gmail or Google Workspace | The monitored mailbox is a Gmail account or Google Workspace mailbox. | [Connect Gmail as a mailbox source] |
| Microsoft 365, Office 365, or Outlook | The monitored mailbox is connected through Microsoft OAuth, including shared mailbox access when permissions allow it. | [Microsoft 365 mailbox setup] |
| IMAP | The provider supports IMAP access, app passwords, folders, and polling. | [IMAP mailbox configuration] |
| Hosted mailbox | MailWebhook should host the inbound address and receive email directly through mailbox aliases. | [Create a hosted mailbox] |
| Loopback mailbox | You need a temporary mailbox for onboarding, route checks, or endpoint testing before connecting production email. | [Test routes with a loopback mailbox] |

## Common mailbox workflows

- Connect the mailbox source.
- Create or select a [route] that matches messages from that mailbox.
- Send route output to an [endpoint].
- Use message preview to inspect route output for a captured message.
- Use backfill for supported providers when you need to ingest messages that arrived before the mailbox was connected.

## Message preview

Message preview lets you inspect route output for a specific captured email before sending it to an endpoint. Open the mailbox action menu, choose preview, select a route, and select a message.

Preview uses the selected route rules and pipeline. Messages that do not match the selected route are not previewed for that route. Attachments are not included in preview output.

## Message backfill

Backfill re-ingests email that arrived before you connected the mailbox to MailWebhook.

- Supported providers: IMAP, Gmail, and Microsoft 365.
- Behavior: jobs run in the background, pause on errors, and skip messages already delivered by MailWebhook.
- Limits: one backfill job can run at a time, and plan limits control how far back a mailbox can be backfilled.
- Monitoring: check backfill status from the same mailbox action menu.

[Route]: {% link docs/routes.md %}
[Endpoint]: {% link docs/endpoints.md %}
[Connect Gmail as a mailbox source]: {% link docs/mailboxes/gmail-setup.md %}
[Microsoft 365 mailbox setup]: {% link docs/mailboxes/microsoft-365-setup.md %}
[Test routes with a loopback mailbox]: {% link docs/mailboxes/loopback-testing.md %}
[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Create a hosted mailbox]: {% link docs/mailboxes/hosted-mailbox-setup.md %}
