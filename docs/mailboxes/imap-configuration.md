---
title: IMAP configuration
parent: Mailboxes
nav_order: 3
description: "Configure an IMAP mailbox source in MailWebhook with host, port, SSL/TLS, username, app password, folder, polling interval, backfill, and webhook verification."
permalink: /docs/mailboxes/imap-configuration/
---

# IMAP mailbox configuration
{: .no_toc }

Use this guide to connect an IMAP mailbox to MailWebhook and verify that new messages can reach your webhook route.

IMAP is the provider-neutral connector for mailboxes that expose a standard IMAP server. MailWebhook logs in to the configured mailbox, polls the selected folder, stores an IMAP UID cursor, and routes newly observed messages to your webhook.

{: .note }
For product examples and use cases, see [IMAP to webhook](https://www.mailwebhook.com/imap-to-webhook).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You need:

- A MailWebhook account with a verified user email.
- A MailWebhook project with connected-mailbox capacity available.
- An IMAP-enabled mailbox.
- The IMAP host, port, username, and password or app password.
- The folder name you want MailWebhook to monitor.
- A route and endpoint if you want to verify delivery immediately after setup.

Use an app password when your mailbox provider requires one for IMAP access.

## Connection steps

Open **Mailboxes** and click **Add Mailbox**.

1. Set **Provider** to **IMAP**.
2. Enter **Email / Username**.
3. Enter **Password / App Password**.
4. Enter **Host** and **Port**. The common secure IMAP port is `993`.
5. Keep **Use SSL/TLS** on unless your provider requires another setting.
6. Set **Folder**. The default folder is `INBOX`.
7. Set **Poll Interval (seconds)** between `60` and `3600`, subject to your plan's polling floor.
8. Click **Test Connection**.
9. Click **Add Mailbox** after the connection test succeeds.

You can click **Auto-discover** after entering **Email / Username**. Auto-discovery can fill **Host**, **Port**, **Use SSL/TLS**, and **Folder** when the provider publishes IMAP discovery records. Enter the settings manually when auto-discovery does not find a result.

## IMAP-specific fields

| Field | Use |
| --- | --- |
| **Email / Username** | IMAP login username. This is usually the mailbox email address. |
| **Password / App Password** | Password used for IMAP login. Use an app password when the provider blocks normal account passwords for IMAP. |
| **Host** | IMAP server hostname, such as `imap.example.com`. |
| **Port** | IMAP server port. Secure IMAP commonly uses `993`. |
| **Use SSL/TLS** | Enables SSL/TLS for the IMAP connection. Keep it enabled for normal production mailboxes. |
| **Folder** | IMAP folder selected for polling. The default is `INBOX`. |
| **Poll Interval (seconds)** | Requested polling interval. The API accepts `60` to `3600`; your plan can enforce a higher minimum interval. |

The connection test logs in with these settings and selects the configured folder. A successful test confirms that MailWebhook can reach the server, authenticate, and access that folder.

## What happens after save

After the IMAP mailbox is saved, MailWebhook:

- Stores the IMAP secret securely.
- Schedules the enabled active mailbox for polling.
- Applies your requested polling interval and any plan-level minimum interval.
- Runs the first poll as soon as the scheduler loads the mailbox.
- Selects the configured IMAP folder during polling.
- Seeds the mailbox cursor from the current highest UID on first poll.

The first live poll does not import existing messages. It stores the current highest UID and waits for newer messages. Later polls fetch messages with UIDs after the stored cursor.

If you need older messages that arrived before the IMAP mailbox was connected, use mailbox backfill.

## Polling behavior

IMAP polling is UID-based.

- MailWebhook polls enabled active IMAP mailboxes.
- The scheduler reloads active mailboxes periodically.
- A mailbox is not polled again until its interval is due.
- A plan-level polling floor can make polling less frequent than the number entered in the mailbox form.
- MailWebhook tracks the highest UID it has seen for the selected folder.
- Messages larger than the configured raw-message size limit are skipped. The default IMAP raw-message limit is 10 MiB.
- When a skipped message has a newer UID, MailWebhook can still advance the cursor beyond that UID.

If repeated polling errors become consecutive unhandled poller failures, MailWebhook can disable the mailbox after the configured failure limit. The default limit is 5 consecutive failures.

## Setup checks

After the mailbox is saved, check:

- The mailbox is listed in **Mailboxes**.
- The mailbox source is IMAP.
- The mailbox is enabled.
- The selected folder is the folder you expect.
- **Test Connection** succeeds for the saved mailbox.
- Your route rule matches the IMAP mailbox address or the recipients you expect.
- Your route pipeline uses the mapper you want, such as `map.generic_json`.
- Your endpoint returns `2xx` only after it accepts the request.

Send a new email after IMAP setup completes. Messages already present in the folder at setup time are not imported by normal polling.

## Verify payload and delivery

Send a new email to the connected IMAP mailbox. Then open **Events** or **Webhook Preview**.

For a route that uses `map.generic_json`, the payload includes `meta.source` set to `imap`:

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
    "subject": "IMAP test",
    "date": "2026-06-28T12:00:00Z",
    "from": [{ "email": "sender@example.com" }],
    "to": [{ "email": "connected@example.com" }]
  },
  "body": {
    "attachments": [],
    "text": "Test message"
  },
  "meta": {
    "source": "imap",
    "raw_size_bytes": 2048,
    "received_at": "2026-06-28T12:00:00Z"
  }
}
```

The exact payload depends on your route pipeline. See [Webhook payload reference] for mapper choices and [Generic JSON] for the full default payload contract.

## Backfill older IMAP messages

Normal IMAP setup starts from the current folder UID cursor. To ingest older messages, open the mailbox actions menu and start **Backfill**.

IMAP backfill searches the selected folder in UTC day windows, stores messages by IMAP UID, and skips messages that were already ingested. It is separate from live polling and is bounded by your plan's backfill limit.

Backfill uses the same IMAP host, port, SSL/TLS, username, password, and folder settings as the mailbox.

## Common errors

### Please verify your email address

MailWebhook requires a verified user email before connecting or testing an IMAP mailbox. Verify your MailWebhook user email and start the IMAP setup again.

### Auto-discover did not find settings

Enter the IMAP **Host**, **Port**, **Use SSL/TLS**, and **Folder** manually. Auto-discovery depends on provider DNS records and is not required for IMAP setup.

### Connection test failed

Check **Host**, **Port**, **Use SSL/TLS**, **Email / Username**, **Password / App Password**, and **Folder**. The connection test must be able to connect, authenticate, and select the folder.

### Host or port is rejected

MailWebhook validates outbound IMAP targets before connecting. Use the public IMAP host and a valid port between `1` and `65535`.

### Folder was not found

Check the exact IMAP folder name. Folder names are provider-specific. Use `INBOX` for the normal inbox unless your provider exposes a different folder name.

### Mailbox limit reached

Your project has no connected-mailbox capacity left. Remove an unused connected mailbox or change to a plan with more connected mailboxes.

### New mail is not arriving

Check these items:

- The IMAP mailbox is enabled.
- The saved connection test succeeds.
- The message arrived after IMAP setup completed.
- The message is in the configured folder.
- The route rule matches the message.
- The endpoint accepts the delivery with a `2xx` response.

If existing mail is visible in the mailbox but no webhook event appears, confirm whether the message arrived before setup. Normal IMAP polling starts from the current highest UID and does not import older messages.

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
