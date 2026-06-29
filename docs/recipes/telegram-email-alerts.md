---
title: Telegram email alerts
parent: Recipes
nav_order: 2
description: "Send inbound email alerts to Telegram with MailWebhook using a mailbox source, route rule, Telegram sendMessage endpoint, map.telegram_simple pipeline, and delivery checks."
permalink: /docs/recipes/telegram-email-alerts/
---

# Send inbound email alerts to Telegram
{: .no_toc }

Use this recipe when inbound email should create a short Telegram alert in a chat, group, or channel.

MailWebhook connects the mailbox, checks the route rule, transforms the email with `map.telegram_simple`, and posts the final JSON body to Telegram. The Telegram mapper emits `chat_id` and `text`; the bot token belongs in the Telegram endpoint URL.

{: .note }
For product context, see [Email to Webhook](https://www.mailwebhook.com/email-to-webhook). For Telegram API behavior, see Telegram's [`sendMessage` reference](https://core.telegram.org/bots/api#sendmessage).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Create a MailWebhook endpoint that posts to Telegram `sendMessage`, include the bot token in the endpoint URL, and create a route whose final pipeline step is `map.telegram_simple`.

Use a route rule to decide which emails reach Telegram. The Telegram mapper then creates this request body:

```json
{
  "chat_id": "-1001234567890",
  "text": "[Alert] Subject: Payment failed\nFrom: Billing <billing@example.com>\nPayment retry failed for customer 8472."
}
```

MailWebhook sends that JSON body exactly as the endpoint request body. It also adds MailWebhook delivery headers such as `X-MailWebhook-Signature` and `X-Idempotency-Key`.

## When to use this

Use the Telegram recipe for:

- Personal or small-team operational alerts.
- Billing, payment, or invoice notices that need quick attention.
- Monitoring emails that should reach a Telegram group.
- Lightweight lead, form, or support notifications.

Use `map.telegram_simple` when a compact Telegram text message is enough. Use Generic JSON or Custom JSON with your own receiver when the workflow needs attachment metadata, persistence, enrichment, or custom downstream logic.

## Prerequisites

You need:

- A MailWebhook account and project.
- A connected mailbox source, such as Gmail, Microsoft 365, Office 365, Outlook, IMAP, Hosted Mailbox, or Loopback for testing.
- A Telegram bot token from BotFather.
- A Telegram `chat_id` string for the target chat, group, or channel.
- Permission for the bot to post in that target.

Keep the Telegram `chat_id` as a string. Group and channel ids can be negative, such as `-1001234567890`.

## Fast path: use Setup Wizard

Use this path when you want MailWebhook to create the Telegram endpoint and route pair for you.

### 1. Choose a mailbox

Open **Onboarding** and choose the mailbox source that should receive the email.

For a quick test, use the loopback mailbox. For production, use the Gmail, Microsoft 365, Office 365, Outlook, IMAP, or hosted mailbox source that receives the alerts.

### 2. Choose Telegram as the destination

In the endpoint step, choose **Telegram**.

Enter:

- Telegram bot token.
- Telegram chat ID.

Saving the Telegram destination creates:

- A Telegram endpoint that posts to `https://api.telegram.org/bot{bot-token}/sendMessage`.
- A route using `map.telegram_simple`.
- A route rule scoped to the selected mailbox address.

### 3. Send a test email

Send a normal email to the selected mailbox address.

Use a subject and body that make the Telegram result easy to recognize:

```text
Subject: Telegram route test

This email should appear in Telegram through MailWebhook.
```

### 4. Verify Telegram and MailWebhook

Check both places:

- Telegram target: the message appears in the chat, group, or channel.
- MailWebhook **Events** or **Webhook Preview**: the event shows the route and delivery attempt.

The delivery is successful in MailWebhook when Telegram returns a `2xx` HTTP response.

## Manual setup

Use this path when you want to configure the endpoint and route JSON directly.

### 1. Create the Telegram endpoint

Open **Endpoints** and create an endpoint with:

| Field | Value |
| --- | --- |
| Webhook URL | `https://api.telegram.org/bot123456:ABCDEF/sendMessage` |
| Custom headers | Leave empty unless your project has a separate proxy or gateway. |
| Timeout | Keep the default unless your project uses a different standard. |

Replace `123456:ABCDEF` with the real Telegram bot token.

Treat the full endpoint URL as secret because it contains the bot token.

MailWebhook sends `Content-Type: application/json` automatically.

### 2. Create the route rule

Open **Routes** and create a route for the Telegram endpoint.

Start with a narrow rule so Telegram receives only actionable alerts:

```json
{
  "to_emails": ["alerts@example.com"],
  "from_domains": ["billing.example"],
  "subject_contains": ["failed", "overdue", "urgent"]
}
```

Rules are AND-ed across populated top-level fields. In this example, the message must be addressed to `alerts@example.com`, come from `billing.example`, and contain at least one of the subject terms.

### 3. Add the Telegram pipeline

Use `html_to_text` before the mapper when the source emails are mostly HTML.

```json
{
  "steps": [
    {
      "name": "html_to_text",
      "args": {
        "prefer": "html",
        "width": 0
      }
    },
    {
      "name": "map.telegram_simple",
      "args": {
        "chat_id": "-1001234567890",
        "prefix": "[Billing]",
        "max_chars": 700,
        "include_from": true
      }
    }
  ]
}
```

`map.telegram_simple` must be the final pipeline step.

### 4. Full route JSON example

```json
{
  "name": "Billing alerts to Telegram",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["alerts@example.com"],
    "from_domains": ["billing.example"],
    "subject_contains": ["failed", "overdue", "urgent"]
  },
  "pipeline": {
    "steps": [
      {
        "name": "html_to_text",
        "args": {
          "prefer": "html",
          "width": 0
        }
      },
      {
        "name": "map.telegram_simple",
        "args": {
          "chat_id": "-1001234567890",
          "prefix": "[Billing]",
          "max_chars": 700,
          "include_from": true
        }
      }
    ]
  }
}
```

## Endpoint behavior

MailWebhook delivers the route pipeline output as a JSON `POST` request.

For the Telegram simple mapper, the body has this shape:

```json
{
  "chat_id": "-1001234567890",
  "text": "[Billing] Subject: Payment failed\nFrom: Billing <billing@example.com>\nPayment retry failed for customer 8472."
}
```

MailWebhook also sends its normal delivery headers:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Content-Type: application/json`
- Any custom endpoint headers you configured

Telegram receives one JSON body. MailWebhook does not add another wrapper around it.

## Verify the result

A working Telegram route has these signs:

- A MailWebhook event exists for the test email.
- The event matched the Telegram route.
- The request body contains `chat_id` and `text`.
- The delivery attempt returned `2xx`.
- The Telegram target shows the message.

If MailWebhook shows a successful HTTP delivery and Telegram does not show the message, check the Telegram API response body, bot permissions, target id, and whether the bot is allowed to post in that chat, group, or channel.

## Common failure checks

### No MailWebhook event exists

The mailbox did not ingest the message, or the test email went to a different address.

Check the mailbox setup first:

- Gmail labels and OAuth status.
- Microsoft mailbox connection.
- IMAP host, folder, and polling.
- Hosted mailbox address.
- Loopback alias status for onboarding tests.

### An event exists, but the route did not match

Inspect the route rule against the real message.

Common causes:

- `to_emails` does not match the recipient address after forwarding.
- `from_domains` does not match the actual sender domain.
- `subject_contains` is too narrow.
- The route is disabled.

Use the route troubleshooting guide before changing the Telegram mapper.

### The route will not save

Check the pipeline contract:

- Exactly one `map.*` step must exist.
- The `map.*` step must be final.
- `map.telegram_simple` requires a non-empty string `chat_id`.
- `max_chars` must be an integer.
- `include_from` must be a boolean.
- `prefix` must be a string.

### Telegram returns `401`, `403`, or `404`

Check the endpoint URL and Telegram target.

The endpoint URL should include the bot token:

```text
https://api.telegram.org/bot123456:ABCDEF/sendMessage
```

Then confirm:

- The bot token is current.
- The `chat_id` is the target id, not a display name.
- The bot has permission to post in the target.
- The bot has been added to the group or channel when required.

### Delivery is successful, but the Telegram message is too long

Reduce `max_chars` in the Telegram mapper.

The default is `800`. A lower value keeps alerts easier to scan:

```json
{
  "name": "map.telegram_simple",
  "args": {
    "chat_id": "-1001234567890",
    "max_chars": 500
  }
}
```

### Attachments do not appear in Telegram

`map.telegram_simple` sends a compact text message and does not upload attachments to Telegram.

If the workflow needs attachment metadata, route to your own webhook with Generic JSON or Custom JSON, then fetch attachment files through the MailWebhook attachment download API.

## Related docs

- [Connect Gmail as a mailbox source]
- [Microsoft 365 mailbox setup]
- [IMAP mailbox configuration]
- [Create a hosted mailbox]
- [Test routes with a loopback mailbox]
- [Rules]
- [Pipeline]
- [Endpoints]
- [Webhook payload reference]
- [Custom JSON recipes]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]
- [Webhook retries and replay]

[Connect Gmail as a mailbox source]: {% link docs/mailboxes/gmail-setup.md %}
[Microsoft 365 mailbox setup]: {% link docs/mailboxes/microsoft-365-setup.md %}
[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Create a hosted mailbox]: {% link docs/mailboxes/hosted-mailbox-setup.md %}
[Test routes with a loopback mailbox]: {% link docs/mailboxes/loopback-testing.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Custom JSON recipes]: {% link docs/routes/pipeline/custom_json/recipes.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
