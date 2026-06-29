---
title: Webhook payload reference
parent: Payloads
nav_order: 1
description: "Use the MailWebhook webhook payload reference hub to choose Generic JSON, Custom JSON, chat payloads, attachment descriptors, and delivery request docs."
permalink: /docs/payloads/webhook-payload-reference/
---

# Webhook payload reference
{: .no_toc }

Use this hub to find the right payload contract for a MailWebhook route.

MailWebhook sends the route pipeline output as the HTTP request body. No extra wrapper is added around the mapper output. Delivery headers such as `X-MailWebhook-Signature` and `X-Idempotency-Key` are part of the HTTP request, not fields inside the JSON payload.

{: .note }
For product context around structured email data, see [Email to JSON](https://www.mailwebhook.com/email-to-json). If you are building a receiving endpoint, see [Email Webhook API](https://www.mailwebhook.com/email-webhook-api).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Payload selection

Choose the payload based on what the downstream system expects.

| Payload type | Pipeline mapper | Use when | Reference |
| --- | --- | --- | --- |
| Generic JSON | `map.generic_json` | You want MailWebhook's stable normalized email shape. | [Generic JSON] |
| Custom JSON | `map.custom_json` | You need route-specific fields or a downstream-specific body shape. | [Custom JSON] |
| Slack simple | `map.slack_simple` | You want a short Slack-compatible message body. | [Pipeline] |
| Telegram simple | `map.telegram_simple` | You want a short Telegram-compatible message body. | [Pipeline] |

Most webhook integrations should start with `map.generic_json`. Move to `map.custom_json` when the receiver needs a smaller or different shape.

## HTTP request body contract

Every route pipeline follows the same delivery rule:

- The final `map.*` step creates the payload.
- MailWebhook posts that payload as JSON to the route endpoint.
- The request uses `POST` with `Content-Type: application/json`.
- The body is exactly the pipeline output bytes.
- No additional envelope is added outside the payload.

For the full HTTP request contract, including signatures, idempotency, endpoint headers, and URL restrictions, see [Endpoints].

## Generic JSON

`map.generic_json` emits the default MailWebhook payload.

Use it when you want a stable normalized email object with:

- `schema`
- `event`
- `message`
- `body`
- `meta`
- optional `envelope`

The schema identifies the payload as:

```json
{
  "schema": {
    "name": "mailwebhook.generic",
    "version": "1"
  }
}
```

Generic JSON is deterministic:

- People arrays are sorted by email.
- Email addresses are lowercased and trimmed.
- Headers are lowercased and normalized.
- Times are UTC whole-second timestamps.
- Empty optional fields are omitted.
- Attachment descriptors are sorted and included in `body.attachments`.

Read next:

- [Generic JSON]
- [Generic JSON example]
- [Fetch email attachments from webhook payloads]
- [Send a test email and inspect the payload]

## Custom JSON

`map.custom_json` emits the JSON shape you define.

Use it when the receiver expects fields such as:

- `ticket_id`
- `customer_email`
- `invoice_number`
- `alert_source`
- `plain_text`
- `attachments`

Custom JSON maps from a runtime evaluation root that includes parsed message fields, headers, attachments, context, static mapper metadata, and computed variables.

The evaluated `output` value is sent as the webhook body. It can be an object, array, string, number, boolean, or `null`. If the top-level output evaluates to `null`, MailWebhook emits an empty object.

Read next:

- [Custom JSON]
- [JsonLogic-Style DSL]
- [Custom JSON recipes]

## Transform steps before mapping

Transform steps can run before the final mapper.

Use them when you need to normalize or remove data before payload mapping:

- `html_to_text`: convert HTML into deterministic plain text.
- `remove_fields`: remove selected parsed message fields.
- `replace_values`: set subject, body, or header values from literals or templates.
- `strip_attachments_if`: remove attachments that match MIME, size, or filename conditions.

The final step still must be exactly one `map.*` mapper.

Read next:

- [Transform Steps]
- [Pipeline]

## Attachment descriptors

Attachments are not embedded in the webhook body.

Generic JSON includes attachment descriptors in `body.attachments`. Custom JSON can include `message.attachments` metadata when you map it into your output.

A Generic JSON attachment descriptor can include:

- `id`
- `filename`
- `content_type`
- `size`
- `is_inline`
- `content_id`
- `sha256`

To fetch file content, call the attachment download URL API with `X-API-Key`. The API returns a short-lived URL with `url`, `expires_at`, `method`, and `headers`.

Current reference:

- [Fetch email attachments from webhook payloads]
- [API keys and attachment downloads]

## Delivery metadata and headers

Keep payload fields separate from delivery headers.

Payload fields are in the JSON body. Delivery metadata is in HTTP headers and event records.

Important delivery headers:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Idempotency-Key`, when the customer endpoint headers do not already include either idempotency header variant.

Use these headers in your receiver for verification and idempotent processing. Use **Events** and **Delivery Attempts History** to inspect delivery status, response status, and replay behavior.

Current reference:

- [Endpoints]
- [Webhook retries and replay]
- [Send a test email and inspect the payload]

## Quick validation checklist

When inspecting a webhook request, check these first:

- The request method is `POST`.
- `Content-Type` is `application/json`.
- The route pipeline ends with the mapper you expect.
- Generic JSON payloads have `schema.name` set to `mailwebhook.generic`.
- Generic JSON payloads have `schema.version` set to `1`.
- Attachment files are represented as metadata, not embedded file bytes.
- The endpoint returns `2xx` for successful delivery.
- Failed attempts appear in **Delivery Attempts History**.

## Related docs

- [Send a test email and inspect the payload]
- [Receive your first inbound email webhook]
- [Pipeline]
- [Generic JSON]
- [Generic JSON example]
- [Custom JSON]
- [JsonLogic-Style DSL]
- [Custom JSON recipes]
- [Transform Steps]
- [Endpoints]
- [API keys and attachment downloads]
- [Webhook retries and replay]
- [Fetch email attachments from webhook payloads]

[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Generic JSON example]: {% link docs/routes/pipeline/generic_json/example.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[JsonLogic-Style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
[Custom JSON recipes]: {% link docs/routes/pipeline/custom_json/recipes.md %}
[Transform Steps]: {% link docs/routes/pipeline/transform_steps.md %}
[Endpoints]: {% link docs/endpoints.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
