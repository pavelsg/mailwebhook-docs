---
title: Quickstarts
nav_order: 2
has_children: true
description: "Start with MailWebhook quickstarts for connecting an email source, routing messages, sending a test email, and verifying webhook delivery."
---

# Quickstarts
{: .no_toc }

Use quickstarts when you want the shortest working path from an email source to a delivered webhook. Start with the first webhook flow, then use payload inspection to confirm the request body, headers, route output, and delivery status.

Core quickstarts:

- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]

Mailbox setup guides:

- [Connect Gmail as a mailbox source]
- [Microsoft 365 mailbox setup]
- [IMAP mailbox configuration]
- [Create a hosted mailbox]
- [Test routes with a loopback mailbox]

Delivery references:

- [Endpoints]
- [Verify signed webhook deliveries]
- [Webhook retries and replay]

[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Connect Gmail as a mailbox source]: {% link docs/mailboxes/gmail-setup.md %}
[Microsoft 365 mailbox setup]: {% link docs/mailboxes/microsoft-365-setup.md %}
[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Create a hosted mailbox]: {% link docs/mailboxes/hosted-mailbox-setup.md %}
[Test routes with a loopback mailbox]: {% link docs/mailboxes/loopback-testing.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
