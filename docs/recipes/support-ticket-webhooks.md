---
title: Support ticket webhooks
parent: Recipes
nav_order: 3
description: "Turn support emails into ticket webhooks with MailWebhook using a mailbox source, route rules, Custom JSON mapping, signed delivery, idempotency, and attachment metadata."
permalink: /docs/recipes/support-ticket-webhooks/
---

# Turn support emails into ticket webhooks
{: .no_toc }

Use this recipe when emails sent to a support inbox should create tickets in your own help desk, queue, CRM, or internal support system.

MailWebhook receives the email, matches a route rule, maps the message into a ticket-shaped JSON body, and posts that body to your ticket endpoint. The endpoint can verify the MailWebhook signature, use the idempotency key to avoid duplicate tickets, and fetch attachments only when needed.

{: .note }
For product context, see [Email to Webhook](https://www.mailwebhook.com/email-to-webhook). If you are building the receiving service, see [Email Webhook API](https://www.mailwebhook.com/email-webhook-api).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Create a MailWebhook endpoint for your ticket API, then create a route for the support mailbox. Use `map.custom_json` to emit a ticket-create body with requester, subject, body text, message ids, and attachment metadata.

Example ticket payload:

```json
{
  "type": "support_ticket.create",
  "external_id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
  "source": "email",
  "requester": {
    "email": "customer@example.com",
    "name": "Customer Name"
  },
  "subject": "Cannot access account",
  "body": "I cannot sign in after resetting my password.",
  "attachments": []
}
```

MailWebhook sends that JSON body exactly as the endpoint request body. Delivery headers include `X-MailWebhook-Signature` and `X-Idempotency-Key`.

## When to use this

Use this recipe for:

- A shared support inbox such as `support@example.com`.
- Contact-form emails that should become tickets.
- Customer replies that should enter an internal queue.
- Vendor or partner support requests that need triage.
- Low-volume help desk intake before you build a deeper provider integration.

Start with `map.custom_json` when the ticket endpoint expects a specific create-ticket body. Start with `map.generic_json` when your receiver wants the full normalized email object and will transform it later.

## Prerequisites

You need:

- A MailWebhook account and project.
- A connected support mailbox source, such as Gmail, Microsoft 365, Office 365, Outlook, IMAP, Hosted Mailbox, or Loopback for testing.
- A public HTTP or HTTPS ticket endpoint that accepts JSON `POST` requests.
- A ticket receiver that can store or enqueue the ticket before returning `2xx`.
- A MailWebhook signing secret if your receiver verifies signed delivery.

MailWebhook endpoint URLs must resolve to public addresses. Private, loopback, link-local, reserved, and multicast targets are blocked.

## Setup overview

The flow is:

1. Connect the mailbox that receives support email.
2. Create the ticket endpoint in MailWebhook.
3. Create a route rule that matches support emails.
4. Use a Custom JSON pipeline to shape the ticket body.
5. Send a test email and verify the event, request body, response, and ticket record.

## 1. Connect the support mailbox

Connect the mailbox that receives customer messages.

Common choices:

- Gmail mailbox for `support@example.com`.
- Microsoft 365, Office 365, or Outlook mailbox for a shared support address.
- IMAP mailbox for an existing help desk inbox.
- Hosted Mailbox when MailWebhook should provide the intake address.
- Loopback mailbox for a first local test.

Copy the exact mailbox address you want the route rule to match.

## 2. Create the ticket endpoint

Open **Endpoints** and create an endpoint for your ticket receiver.

Example:

| Field | Value |
| --- | --- |
| Webhook URL | `https://tickets.example.com/mailwebhook/support` |
| Custom header | `Authorization: Bearer ticket-api-token` |
| Timeout | Keep the default unless your receiver needs a different project standard. |

Your receiver should:

- Read the raw request body before parsing JSON if it verifies `X-MailWebhook-Signature`.
- Use `X-Idempotency-Key` or `external_id` to avoid duplicate tickets during retries and replay.
- Store or enqueue the ticket before returning `2xx`.
- Return a retryable status such as `503` when another delivery attempt can help.

## 3. Create the route rule

Open **Routes** and create a route for the ticket endpoint.

For a dedicated support inbox, start with the recipient rule:

```json
{
  "to_emails": ["support@example.com"]
}
```

For a shared mailbox, add subject or sender constraints:

```json
{
  "to_emails": ["team@example.com"],
  "subject_contains": ["support", "help", "issue"],
  "none": [
    {
      "from_domains": ["internal.example"]
    }
  ]
}
```

Top-level rule fields are AND-ed. In the second example, the message must go to `team@example.com`, include one of the subject terms, and avoid the internal sender domain.

## 4. Add the ticket payload mapper

Use `map.custom_json` as the final pipeline step.

```json
{
  "steps": [
    {
      "name": "map.custom_json",
      "args": {
        "version": "v1",
        "vars": [
          {
            "name": "body_text",
            "expr": {
              "call.transform.html_to_text": {
                "html": { "var": "message.html" },
                "text": { "var": "message.text" }
              }
            }
          }
        ],
        "output": {
          "type": "support_ticket.create",
          "external_id": { "var": "ctx.event_id" },
          "source": "email",
          "mailwebhook": {
            "event_id": { "var": "ctx.event_id" },
            "route_id": { "var": "ctx.route_id" },
            "source_type": { "var": "ctx.source_type" }
          },
          "requester": {
            "email": { "var": "message.from[0].email" },
            "name": { "var": "message.from[0].name" }
          },
          "recipients": {
            "to": { "var": "message.to" },
            "reply_to": { "var": "message.reply_to" }
          },
          "subject": { "var": "message.subject" },
          "body": { "var": "vars.body_text" },
          "message": {
            "message_id": { "var": "message.message_id" },
            "date": { "var": "message.date" }
          },
          "attachments": { "var": "message.attachments" }
        }
      }
    }
  ]
}
```

This mapper keeps the ticket body small while preserving the MailWebhook event id, route id, source type, sender, recipients, email body, message id, message date, and attachment metadata.

## Full route JSON example

```json
{
  "name": "Support inbox to ticket API",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["support@example.com"],
    "none": [
      {
        "from_domains": ["internal.example"]
      }
    ]
  },
  "pipeline": {
    "steps": [
      {
        "name": "map.custom_json",
        "args": {
          "version": "v1",
          "vars": [
            {
              "name": "body_text",
              "expr": {
                "call.transform.html_to_text": {
                  "html": { "var": "message.html" },
                  "text": { "var": "message.text" }
                }
              }
            }
          ],
          "output": {
            "type": "support_ticket.create",
            "external_id": { "var": "ctx.event_id" },
            "source": "email",
            "mailwebhook": {
              "event_id": { "var": "ctx.event_id" },
              "route_id": { "var": "ctx.route_id" },
              "source_type": { "var": "ctx.source_type" }
            },
            "requester": {
              "email": { "var": "message.from[0].email" },
              "name": { "var": "message.from[0].name" }
            },
            "recipients": {
              "to": { "var": "message.to" },
              "reply_to": { "var": "message.reply_to" }
            },
            "subject": { "var": "message.subject" },
            "body": { "var": "vars.body_text" },
            "message": {
              "message_id": { "var": "message.message_id" },
              "date": { "var": "message.date" }
            },
            "attachments": { "var": "message.attachments" }
          }
        }
      }
    ]
  }
}
```

## Endpoint behavior

MailWebhook delivers the route pipeline output as a JSON `POST` request.

For the Custom JSON mapper above, the body has this shape:

```json
{
  "type": "support_ticket.create",
  "external_id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
  "source": "email",
  "mailwebhook": {
    "event_id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
    "route_id": "2f3713bf-88cc-46c6-aaa3-ea9d6e9d20f3",
    "source_type": "gmail"
  },
  "requester": {
    "email": "customer@example.com",
    "name": "Customer Name"
  },
  "recipients": {
    "to": [{ "email": "support@example.com" }],
    "reply_to": []
  },
  "subject": "Cannot access account",
  "body": "I cannot sign in after resetting my password.",
  "message": {
    "message_id": "<customer-message@example.com>",
    "date": "2026-06-28T12:00:00Z"
  },
  "attachments": [
    {
      "id": "att_01",
      "filename": "screenshot.png",
      "content_type": "image/png",
      "size": 48213,
      "sha256": "..."
    }
  ]
}
```

MailWebhook also sends:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Content-Type: application/json`
- Any custom endpoint headers you configured

Attachments are metadata only in this request body. Fetch file bytes through the MailWebhook attachment download API after your receiver decides it needs them.

## Verify the result

A working support-ticket route has these signs:

- A MailWebhook event exists for the test email.
- The event matched the support-ticket route.
- The request body contains `type`, `external_id`, `requester`, `subject`, and `body`.
- The delivery attempt returned `2xx`.
- The receiving service created or enqueued exactly one ticket.
- Replay of the same event does not create a duplicate ticket.

Use **Webhook Preview** or **Events** to compare the delivered request body with the ticket record created by your receiver.

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

- `to_emails` does not match the final recipient after forwarding.
- `subject_contains` excludes a valid support email.
- `none` blocks a sender that should be allowed.
- The route is disabled.

### The route will not save

Check the pipeline contract:

- Exactly one `map.*` step must exist.
- The `map.*` step must be final.
- `map.custom_json` requires `version: "v1"`.
- `map.custom_json` requires an `output` object or value.
- Custom JSON paths must use roots such as `message`, `ctx`, `meta`, and `vars`.

### The ticket body has `null` fields

Missing Custom JSON paths evaluate to `null`.

Common causes:

- The message has no `reply_to`.
- The sender did not include a display name.
- The email has text but no HTML, or HTML but no plain text.
- The route test message lacks provider-real headers.

Use a provider-real test email when the mapper depends on headers, attachments, or forwarding behavior.

### The receiver creates duplicate tickets

Use `X-Idempotency-Key` as the primary duplicate guard. You can also store `external_id`.

MailWebhook may retry retryable failures and can replay an event after you request replay. Your receiver should treat the same idempotency key as the same ticket creation attempt.

### Attachments are missing from the ticket

Custom JSON can include `message.attachments` metadata. It does not inline file bytes.

To attach files to the ticket:

1. Read the attachment metadata from the webhook body.
2. Call the attachment download URL API with your MailWebhook API key.
3. Upload the downloaded file to your ticket system.
4. Store the attachment id and filename on the ticket for traceability.

### Signature verification fails

Verify `X-MailWebhook-Signature` against the exact raw request body bytes.

Do not verify against parsed JSON, pretty-printed JSON, decoded text, or a reserialized copy of the body.

## Related docs

- [Connect Gmail as a mailbox source]
- [Microsoft 365 mailbox setup]
- [IMAP mailbox configuration]
- [Create a hosted mailbox]
- [Test routes with a loopback mailbox]
- [Rules]
- [Pipeline]
- [Custom JSON]
- [JsonLogic-Style DSL]
- [Endpoints]
- [Webhook payload reference]
- [Verify signed webhook deliveries]
- [Fetch email attachments from webhook payloads]
- [API keys and attachment downloads]
- [Troubleshoot route rules that do not match]
- [Troubleshoot Custom JSON mapper errors]
- [Troubleshoot failed webhook deliveries]
- [Webhook retries and replay]

[Connect Gmail as a mailbox source]: {% link docs/mailboxes/gmail-setup.md %}
[Microsoft 365 mailbox setup]: {% link docs/mailboxes/microsoft-365-setup.md %}
[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Create a hosted mailbox]: {% link docs/mailboxes/hosted-mailbox-setup.md %}
[Test routes with a loopback mailbox]: {% link docs/mailboxes/loopback-testing.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[JsonLogic-Style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot Custom JSON mapper errors]: {% link docs/troubleshooting/custom-json-mapper-errors.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
