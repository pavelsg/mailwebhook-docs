---
title: Slack email webhooks
parent: Recipes
nav_order: 1
description: "Send inbound email to Slack with MailWebhook using a mailbox source, route rule, Slack chat endpoint, map.slack_simple pipeline, and delivery checks."
permalink: /docs/recipes/slack-email-webhooks/
---

# Send inbound email to Slack
{: .no_toc }

Use this recipe when important inbound emails should appear in a Slack channel as short, readable messages.

MailWebhook connects the mailbox, applies a route rule, transforms the email with `map.slack_simple`, and posts the final JSON body to Slack. The Slack mapper emits `channel` and `text`; Slack authentication stays on the endpoint as an HTTP header.

{: .note }
For product context, see [Email to Webhook](https://www.mailwebhook.com/email-to-webhook). For Slack API behavior, see Slack's [`chat.postMessage` reference](https://api.slack.com/methods/chat.postMessage).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Create a MailWebhook endpoint that posts to Slack `chat.postMessage`, add the Slack bot token as a custom `Authorization` header, and create a route whose final pipeline step is `map.slack_simple`.

Use a route rule to decide which emails reach Slack. The Slack mapper then creates this request body:

```json
{
  "channel": "C0123456789",
  "text": "[Ops] Subject: Backup failed\nFrom: Alerts <alerts@monitor.example>\nBackup job nightly-db failed on db-03."
}
```

MailWebhook sends that JSON body exactly as the endpoint request body. It also adds MailWebhook delivery headers such as `X-MailWebhook-Signature` and `X-Idempotency-Key`.

## When to use this

Use the Slack recipe for:

- Operational alerts that arrive by email.
- Vendor, billing, or invoice notifications that a team should see quickly.
- Contact form or lead notifications that should reach a shared channel.
- Monitoring systems that can send email and need a Slack bridge.

Use `map.slack_simple` when a compact Slack message is enough. Use Custom JSON when the Slack receiver needs a richer body, such as custom `blocks` or route-specific fields.

## Prerequisites

You need:

- A MailWebhook account and project.
- A connected mailbox source, such as Gmail, Microsoft 365, Office 365, Outlook, IMAP, Hosted Mailbox, or Loopback for testing.
- A Slack app or bot token that can post messages with `chat.postMessage`.
- A Slack channel ID, such as `C0123456789`.
- The Slack bot invited to the target channel.

For the MailWebhook Slack onboarding helper, use the channel ID. Do not use a `#channel-name` value there.

## Fast path: use Setup Wizard

Use this path when you want MailWebhook to create the Slack endpoint and route pair for you.

### 1. Choose a mailbox

Open **Onboarding** and choose the mailbox source that should receive the email.

For a quick test, use the loopback mailbox. For production, use the Gmail, Microsoft 365, Office 365, Outlook, IMAP, or hosted mailbox source that receives the messages you want in Slack.

### 2. Choose Slack as the destination

In the endpoint step, choose **Slack**.

Enter:

- Slack bot token.
- Slack channel ID.

Saving the Slack destination creates:

- A Slack endpoint that posts to `https://slack.com/api/chat.postMessage`.
- A custom `Authorization: Bearer ...` endpoint header.
- A route using `map.slack_simple`.
- A route rule scoped to the selected mailbox address.

### 3. Send a test email

Send a normal email to the selected mailbox address.

Use a subject and body that make the Slack result easy to recognize:

```text
Subject: Slack route test

This email should appear in the Slack channel through MailWebhook.
```

### 4. Verify Slack and MailWebhook

Check both places:

- Slack channel: the message appears in the selected channel.
- MailWebhook **Events** or **Webhook Preview**: the event shows the route and delivery attempt.

The delivery is successful in MailWebhook when Slack returns a `2xx` HTTP response.

## Manual setup

Use this path when you want to configure the endpoint and route JSON directly.

### 1. Create the Slack endpoint

Open **Endpoints** and create an endpoint with:

| Field | Value |
| --- | --- |
| Webhook URL | `https://slack.com/api/chat.postMessage` |
| Custom header | `Authorization: Bearer xoxb-your-token` |
| Timeout | Keep the default unless your project uses a different standard. |

MailWebhook sends `Content-Type: application/json` automatically.

### 2. Create the route rule

Open **Routes** and create a route for the Slack endpoint.

Start with a narrow rule so Slack receives only useful messages:

```json
{
  "to_emails": ["alerts@example.com"],
  "from_domains": ["monitor.example"],
  "subject_contains": ["failed", "critical", "down"]
}
```

Rules are AND-ed across populated top-level fields. In this example, the message must be addressed to `alerts@example.com`, come from `monitor.example`, and contain at least one of the subject terms.

### 3. Add the Slack pipeline

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
      "name": "map.slack_simple",
      "args": {
        "channel": "C0123456789",
        "prefix": "[Ops]",
        "max_chars": 700,
        "include_from": true
      }
    }
  ]
}
```

`map.slack_simple` must be the final pipeline step.

### 4. Full route JSON example

```json
{
  "name": "Operational alerts to Slack",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["alerts@example.com"],
    "from_domains": ["monitor.example"],
    "subject_contains": ["failed", "critical", "down"]
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
        "name": "map.slack_simple",
        "args": {
          "channel": "C0123456789",
          "prefix": "[Ops]",
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

For the Slack simple mapper, the body has this shape:

```json
{
  "channel": "C0123456789",
  "text": "[Ops] Subject: Backup failed\nFrom: Alerts <alerts@monitor.example>\nBackup job nightly-db failed on db-03."
}
```

MailWebhook also sends its normal delivery headers:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Content-Type: application/json`
- Any custom endpoint headers, including Slack `Authorization`

Slack receives one JSON body. MailWebhook does not add another wrapper around it.

## Verify the result

A working Slack route has these signs:

- A MailWebhook event exists for the test email.
- The event matched the Slack route.
- The request body contains `channel` and `text`.
- The delivery attempt returned `2xx`.
- The Slack channel shows the message.

If the event is delivered in MailWebhook and Slack still does not show a message, recheck the Slack token, bot channel membership, channel ID, and Slack API response details. Slack API errors can appear even when the HTTP response is successful.

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

Use the route troubleshooting guide before changing the Slack mapper.

### The route will not save

Check the pipeline contract:

- Exactly one `map.*` step must exist.
- The `map.*` step must be final.
- `map.slack_simple` requires a non-empty string `channel`.
- `max_chars` must be an integer.
- `include_from` must be a boolean.
- `prefix` must be a string.

### Slack returns `401` or `403`

Check the endpoint custom header and Slack permissions.

The endpoint should include:

```text
Authorization: Bearer xoxb-your-token
```

Also confirm that the bot can post to the selected channel.

### Delivery is successful, but Slack message is too long

Reduce `max_chars` in the Slack mapper.

The default is `800`. A lower value keeps the channel easier to scan:

```json
{
  "name": "map.slack_simple",
  "args": {
    "channel": "C0123456789",
    "max_chars": 500
  }
}
```

### Attachments do not appear in Slack

`map.slack_simple` sends a compact text message and does not upload attachments to Slack.

If the Slack workflow needs attachment metadata, route to your own webhook with Generic JSON or Custom JSON, then fetch attachment files through the MailWebhook attachment download API.

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
