---
title: Create a hosted mailbox
parent: Mailboxes
nav_order: 4
description: "Create a MailWebhook-hosted mailbox with inbound aliases, hosted mailbox capacity, message size policy, spam thresholds, route checks, and webhook verification."
permalink: /docs/mailboxes/hosted-mailbox-setup/
---

# Create a hosted mailbox
{: .no_toc }

Use this guide to create a MailWebhook-hosted inbound address and verify that messages sent to it can reach your webhook route.

A hosted mailbox is an inbound email address hosted by MailWebhook. It receives mail directly through MailWebhook's inbound mail system, stores the raw message, evaluates your routes, and delivers the route payload to your endpoint.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You need:

- A MailWebhook account with a verified user email.
- A MailWebhook project with hosted-mailbox capacity available.
- At least one alias localpart, such as `invoices`, `alerts`, or `orders`.
- A route and endpoint if you want to verify delivery immediately after setup.

Hosted mailbox capacity is separate from connected mailbox capacity. A disabled hosted mailbox still consumes hosted-mailbox capacity until it is deleted.

## Creation steps

Open **Mailboxes** and click **Add Mailbox**.

1. Choose **Hosted Mailbox**.
2. Select **Domain**.
3. Add one or more **Aliases**. Enter one alias per row.
4. Leave **Mailbox Enabled** on.
5. Click **Add Mailbox**.
6. Confirm the hosted mailbox appears in **Mailboxes** with the expected full email address.

The create flow requires a domain and at least one alias. If no policy values are set, MailWebhook applies the hosted policy defaults listed below.

## Hosted-specific fields

| Field | Use |
| --- | --- |
| **Domain** | MailWebhook-hosted domain for the mailbox address. |
| **Aliases** | Localparts or full email addresses that route to the hosted mailbox. At least one alias is required. |
| **Mailbox Enabled** | Allows the hosted mailbox and its enabled aliases to receive mail. |
| **Max message size (bytes)** | Maximum accepted raw message size. The default policy value is `26214400` bytes, or 25 MiB. |
| **Spam quarantine score** | Score at or above which a hosted message is quarantined instead of delivered. The default is `6.0`. |
| **Spam reject score** | Score at or above which a hosted message is rejected before storage. The default is `12.0`. |

Aliases are normalized before save. Localparts are lowercased. If you enter only a localpart, MailWebhook combines it with the selected hosted domain.

## Aliases

Each hosted mailbox can have multiple aliases. Incoming mail sent to any enabled alias is processed through the same hosted mailbox.

Use aliases to make routing and filtering simple:

- `invoices@...` for billing intake.
- `alerts@...` for monitoring emails.
- `orders@...` for order notifications.

An alias must be unique. If an alias already belongs to an active hosted mailbox, MailWebhook rejects the create or update request.

## What happens after save

After the hosted mailbox is saved, MailWebhook:

- Creates a hosted mailbox record for the project.
- Creates a hosted policy with message size and spam threshold settings.
- Creates one inbound address row for each alias.
- Marks the mailbox and aliases enabled when **Mailbox Enabled** is on.
- Makes enabled aliases available for route rules and mailbox selection.

Hosted mailboxes receive mail directly. They are not scheduled by the IMAP/Gmail/Microsoft polling service.

## Inbound delivery behavior

When an email arrives at a hosted alias, MailWebhook:

- Resolves the recipient address to an enabled, non-expired alias.
- Rejects unknown, disabled, or expired aliases.
- Enforces the hosted mailbox max message size before storing the message.
- Reads SpamAssassin or amavisd spam headers when present.
- Rejects mail with a spam score at or above the reject threshold.
- Marks mail with a spam score at or above the quarantine threshold as quarantined.
- Stores accepted raw MIME and enqueues route processing.
- Creates events for matching routes.
- Skips webhook delivery for quarantined events until you review or replay them.

Backfill applies to connected provider mailboxes. Hosted mailboxes begin receiving messages after the alias exists and is enabled.

## Setup checks

After the hosted mailbox is saved, check:

- The mailbox is listed in **Mailboxes**.
- The mailbox source is hosted.
- The mailbox is enabled.
- The alias you plan to use is enabled.
- Your route rule matches the hosted alias or recipients you expect.
- Your route pipeline uses the mapper you want, such as `map.generic_json`.
- Your endpoint returns `2xx` only after it accepts the request.

Use a normal inbound email to verify hosted mailbox delivery. Route-level test events are useful for route and endpoint checks, but they do not prove external mail reached the hosted alias.

## Verify payload and delivery

Send a new email to the hosted alias. Then open **Events** or **Webhook Preview**.

For a route that uses `map.generic_json`, the payload includes `meta.source` set to `hosted`. Hosted messages can also include `envelope` and `meta.spam`:

```json
{
  "schema": {
    "name": "mailwebhook.generic",
    "version": "1"
  },
  "event": {
    "id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
    "project_id": "dca29061-c4a7-4687-a8dd-24d2f26548c7",
    "route_id": "2f3713bf-88cc-46c6-aaa3-ea9d6e9d20f3",
    "created_at": "2026-06-28T12:00:02Z"
  },
  "message": {
    "message_id": "<message-id@example.com>",
    "message_id_type": "original",
    "subject": "Hosted mailbox test",
    "date": "2026-06-28T12:00:00Z",
    "from": [{ "email": "sender@example.com" }],
    "to": [{ "email": "orders@project.in.mailwebhook.dev" }]
  },
  "envelope": {
    "mail_from": "sender@example.com",
    "rcpt_to": ["orders@project.in.mailwebhook.dev"]
  },
  "body": {
    "attachments": [],
    "text": "Test message"
  },
  "meta": {
    "source": "hosted",
    "raw_size_bytes": 2048,
    "received_at": "2026-06-28T12:00:00Z",
    "spam": {
      "score": 1.2,
      "quarantined": false,
      "status": "No, score=1.2"
    }
  }
}
```

The exact payload depends on your route pipeline. See [Webhook payload reference] for mapper choices and [Generic JSON] for the full default payload contract.

## Common errors

### Please verify your email address

MailWebhook requires a verified user email before creating a hosted mailbox. Verify your MailWebhook user email and create the mailbox again.

### Hosted mailbox limit reached

Your project has no hosted-mailbox capacity left. Delete an unused hosted mailbox or change to a plan with more hosted mailbox capacity.

### At least one alias is required

Add an alias localpart before saving the hosted mailbox. Example localparts include `inbox`, `orders`, and `alerts`.

### Alias already exists

Choose a different alias. A hosted alias must be unique across active hosted mailbox records.

### Message exceeds max size

The raw email is larger than the hosted mailbox's max message size. Increase the policy limit where available or reduce the size of the incoming email.

### Message was rejected as spam

The message spam score was at or above the hosted mailbox reject threshold. Adjust the reject threshold only when you trust the source and understand the spam risk.

### Message is quarantined

The message spam score was at or above the quarantine threshold and below the reject threshold. MailWebhook creates a quarantined event and skips webhook delivery until you review or replay it.

### New mail is not arriving

Check these items:

- The hosted mailbox is enabled.
- The alias is enabled.
- The sender used the exact hosted alias address.
- The message is below the max message size.
- The message was not rejected by spam policy.
- The route rule matches the message.
- The endpoint accepts delivery with a `2xx` response.

## Related docs

- [Mailboxes]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Routes]
- [Rules]
- [Endpoints]
- [Webhook payload reference]
- [Generic JSON]
- [Webhook retries and replay]
- [Attachments]
- [Email to webhook]

[Mailboxes]: {% link docs/mailboxes.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Attachments]: {% link docs/payloads/attachments.md %}
[Email to webhook]: https://www.mailwebhook.com/email-to-webhook
