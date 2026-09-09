---
title: Gmail setup
parent: Mailboxes
nav_order: 1
description: "Connect Gmail as a MailWebhook mailbox source with Google OAuth, optional Gmail label ID filtering, new-message sync, route checks, and webhook verification."
permalink: /docs/mailboxes/gmail-setup/
---

# Connect Gmail as a mailbox source
{: .no_toc }

Use this guide to connect a Gmail mailbox to MailWebhook and verify that new Gmail messages can reach your webhook route.

MailWebhook uses Google OAuth for Gmail. You do not enter an IMAP host, password, or app password for the native Gmail connector.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You need:

- A MailWebhook account with a verified user email.
- A MailWebhook project with connected-mailbox capacity available.
- A Gmail or Google Workspace mailbox you can authorize through Google OAuth.
- A route and endpoint if you want to verify delivery immediately after setup.

MailWebhook requests read-only Gmail access. The default Gmail OAuth scope is:

```text
https://www.googleapis.com/auth/gmail.readonly
```

## Connection steps

Open **Mailboxes** and click **Add Mailbox**.

1. Set **Provider** to **Gmail**.
2. Leave **Gmail label ID (optional)** blank unless you want to monitor only one Gmail label.
3. Click **Add Mailbox** to start the Google OAuth flow.
4. Complete the Google OAuth consent flow.
5. Return to MailWebhook and confirm the mailbox appears in **Mailboxes**.

In onboarding or other modal flows, choosing **Gmail** can open the Google OAuth flow in a popup. If the popup is blocked, allow popups for MailWebhook and start the connection again.

Use `INBOX` when you want MailWebhook to watch only new messages that carry Gmail's inbox label. For a custom Gmail label, use the underlying Gmail label ID instead of the display name.

<figure>
  <img src="/assets/images/guides/gmail/provider-label.webp" alt="Add Mailbox modal with Gmail selected and INBOX entered in the optional Gmail label ID field." width="800" height="658" loading="lazy">
  <figcaption>Select Gmail and enter a supported label ID such as INBOX when you want MailWebhook to watch only that Gmail label.</figcaption>
</figure>

## Gmail label ID field

The Gmail-specific field is **Gmail label ID (optional)**.

Use it when the MailWebhook route should only receive messages from one Gmail label.

Common values:

| Value | Meaning |
| --- | --- |
| `INBOX` | Monitor messages that have Gmail's inbox label. |
| A Gmail label id | Monitor messages that have that exact Gmail label id. |
| Blank | Use the default mailbox scope configured by the Gmail integration. |

The value is stored as the mailbox `label_filter`. MailWebhook includes it when it establishes the Gmail watch and when it reads Gmail history for new messages.

Use the label id for custom Gmail labels. Display names can be ambiguous.

## What happens after consent

After Google returns control to MailWebhook, MailWebhook:

- Reads the Gmail profile email address.
- Creates or reconnects the Gmail mailbox for the current project.
- Stores the OAuth refresh token securely.
- Stores a Gmail history cursor.
- Establishes a Gmail watch for new message notifications.

New Gmail connections start from the current Gmail history cursor. They do not import older mail during normal setup. Use mailbox backfill when you need messages that arrived before the Gmail mailbox was connected.

<figure>
  <img src="/assets/images/guides/gmail/connected-mailbox.webp" alt="Mailboxes table excerpt showing a Gmail mailbox row with type gmail and Enabled status." width="1254" height="111" loading="lazy">
  <figcaption>After OAuth, the Gmail mailbox appears in Mailboxes with type gmail and Enabled status.</figcaption>
</figure>

## Setup checks

After the OAuth flow completes, check:

- The mailbox is listed in **Mailboxes**.
- The mailbox source is Gmail.
- The mailbox is enabled.
- Your route rule matches the Gmail address or the recipients you expect.
- Your route pipeline uses the mapper you want, such as `map.generic_json`.
- Your endpoint returns `2xx` only after it accepts the request.

If you configured a Gmail label ID, send a new message that receives that label as it arrives. Live Gmail sync uses the stored label filter. Use Gmail backfill for older labeled messages.

## Verify payload and delivery

Send a new email to the connected Gmail mailbox. Then open **Events** or **Webhook Preview**.

Use a message that arrives after the Gmail mailbox is connected. That is the setup check for live Gmail sync. Backfill is a separate action for older Gmail messages and should not be used as the first setup check.

<figure>
  <img src="/assets/images/guides/gmail/received-event.webp" alt="Event Details modal for a delivered Gmail mailbox event with message summary, delivery attempt, latency, HTTP 200 status, and receiver response." width="900" height="623" loading="lazy">
  <figcaption>A delivered Gmail mailbox event confirms the Gmail route reached the test receiver and returned a successful response.</figcaption>
</figure>

For a route that uses `map.generic_json`, the payload includes `meta.source` set to `gmail`:

```json
{
  "schema": {
    "name": "mailwebhook.generic",
    "version": "1"
  },
  "event": {
    "id": "11111111-1111-4111-8111-111111111111",
    "project_id": "22222222-2222-4222-8222-222222222222",
    "route_id": "33333333-3333-4333-8333-333333333333",
    "created_at": "2026-06-28T12:00:02Z"
  },
  "message": {
    "message_id": "<message-id@gmail.com>",
    "message_id_type": "original",
    "subject": "Gmail test",
    "date": "2026-06-28T12:00:00Z",
    "from": [{ "email": "sender@example.com" }],
    "to": [{ "email": "connected@example.com" }]
  },
  "body": {
    "attachments": [],
    "text": "Test message"
  },
  "meta": {
    "source": "gmail",
    "raw_size_bytes": 2048,
    "received_at": "2026-06-28T12:00:00Z"
  }
}
```

The exact payload depends on your route pipeline. See [Webhook payload reference] for mapper choices and [Generic JSON] for the full default payload contract.

## Send Gmail messages to Slack

After Gmail events are arriving in MailWebhook, use [Send Gmail messages to Slack] to route label-scoped Gmail messages into a Slack channel. Start with a narrow route rule, test with a new labeled message, and confirm the MailWebhook event before checking the Slack result.

## Backfill older Gmail messages

Normal Gmail setup starts from the current mailbox cursor. To ingest older messages, open the mailbox actions menu and start **Backfill**.

Backfill runs in UTC day windows and is bounded by your plan's backfill limit. It is separate from live Gmail sync.

## Common errors

### Please verify your email address

MailWebhook requires a verified user email before connecting a Gmail mailbox. Verify your MailWebhook user email and start the Gmail connection again.

### Popup was blocked

Allow popups for MailWebhook and retry the Gmail connection. This applies to modal flows that open Google OAuth in a popup window.

### Gmail connection failed

Start the connection again and complete the Google consent flow. If the error persists, check whether your Google account or organization blocks third-party OAuth access.

### No refresh token returned

The OAuth flow must return a refresh token so MailWebhook can keep the mailbox connected. Retry the flow and approve consent when Google asks.

### Mailbox limit reached

Your project has no connected-mailbox capacity left. Remove an unused connected mailbox or change to a plan with more connected mailboxes.

### New mail is not arriving

Check these items:

- The Gmail mailbox is enabled.
- The message arrived after Gmail setup completed.
- The message has the Gmail label you configured, if you set **Gmail label ID (optional)**.
- The route rule matches the message.
- The endpoint accepts the delivery with a `2xx` response.

If you need to inspect provider-real message fields, send a normal email to the Gmail mailbox. Route-level test events are useful for route and endpoint checks, but they do not prove Gmail label filtering.

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
- [Troubleshoot Gmail label filtering]
- [Gmail to webhook]

[Mailboxes]: {% link docs/mailboxes.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Troubleshoot Gmail label filtering]: {% link docs/troubleshooting/gmail-label-issues.md %}
[Send Gmail messages to Slack]: {% link docs/recipes/slack-email-webhooks.md %}#gmail-label-to-slack
[Gmail to webhook]: https://www.mailwebhook.com/gmail-to-webhook
