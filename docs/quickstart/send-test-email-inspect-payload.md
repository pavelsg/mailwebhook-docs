---
title: Test email and payload inspection
parent: Quickstarts
nav_order: 2
description: "Send a MailWebhook test email and inspect the captured request, response, Generic JSON payload, route output, and delivery attempt status."
permalink: /docs/quickstart/send-test-email-inspect-payload/
---

# Send a test email and inspect the payload
{: .no_toc }

Use this quickstart when you already have a route and want to see what MailWebhook sends to your endpoint.

You will send a test message, inspect the webhook request body, compare it with the Generic JSON contract, and confirm the delivery status.

{: .note }
For product context, see [Email to Webhook](https://www.mailwebhook.com/email-to-webhook). For structured payload positioning, see [Email to JSON](https://www.mailwebhook.com/email-to-json).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You need:

- A saved mailbox, endpoint, and enabled route.
- A route pipeline that uses `map.generic_json`, unless you intentionally want to inspect another mapper.
- A public endpoint that returns a `2xx` response for successful tests.

If you have not created a route yet, start with [Receive your first inbound email webhook].

## Choose a test method

Use the method that matches what you need to verify.

| Method | Use when | What it verifies |
| --- | --- | --- |
| **Setup Wizard** > **Send Test Email** | You want the fastest full request preview. | Route, endpoint, request body, response, snippets. |
| Send a normal email | You want provider-real parsing. | Mailbox ingestion, route match, payload, endpoint delivery. |
| **Routes** > **Send Test** | You want a route and endpoint smoke test. | Pipeline and delivery using a synthetic message. |
| **Mailbox Preview** | You want to inspect route output for a stored message. | Whether a selected message matches a route and what JSON it produces. |

## Method 1: inspect with Webhook Preview

Use this method from onboarding when you want to see the actual request body in the product UI.

### 1. Open Setup Wizard

Open **Onboarding**. If the modal is closed, click **Setup Wizard**.

### 2. Save the endpoint step

In **2. Endpoint**, choose **Configure** and enter your endpoint URL, or choose **Test loopback** for a UI-only first pass.

Click **Save**.

Saving this step creates or reuses an endpoint and route for the selected mailbox. The default HTTP route uses `map.generic_json`.

### 3. Send the test email

In **3. Send Email**, click **Send Test Email** when it is available, or send a normal email to the displayed mailbox address.

The preview may show **Waiting for your test email** while MailWebhook waits for the message and first delivery attempt. Email delivery can take 30-60 seconds depending on the sender and mailbox provider.

### 4. Inspect the Request tab

Open **Webhook Preview** and select **Request**.

Check:

- The method is `POST`.
- The request has `Content-Type: application/json`.
- The request includes `X-MailWebhook-Signature`.
- The request includes `X-Idempotency-Key`.
- The body is the route pipeline output.

For `map.generic_json`, the body should include:

- `schema`
- `event`
- `message`
- `body`
- `meta`

### 5. Inspect the Response tab

Select **Response**.

Check:

- Your endpoint returned a `2xx` status for success.
- The status label is **Success** after a delivered event.
- Failed delivery attempts show response status and body preview when a response was captured.

### 6. Inspect generated examples

Use the generated tabs when you want to reproduce the captured request:

- `curl`
- Python
- Node.js

These examples are generated from the captured request. If no request has been captured yet, the preview shows a no-capture message.

## Method 2: inspect with Mailbox Preview

Use this method when a real message is already stored and you want to test it against a route before replaying or changing the route.

### 1. Open Mailbox Preview

Open **Mailboxes** and use **Preview** for the mailbox that received the message.

### 2. Select the message and route

Choose a message, then choose the route you want to test.

Mailbox Preview checks the selected message against the selected route rule.

Expected status messages include:

- **Selected message does not match this route.**
- **Route matched. Preview payload is shown below.**

### 3. Inspect the JSON output

When the route matches, inspect the payload shown in **Mailbox Preview**.

Use this for route and mapper debugging. Attachment files are not included in mailbox preview output.

## Method 3: send a route-level test event

Use this method when you want to test an existing route and endpoint without waiting for a real email.

### 1. Open Routes

Open **Routes**.

Find the route and open its action menu.

### 2. Click Send Test

Click **Send Test**.

MailWebhook queues a synthetic event and redirects to **Events** with the success message **Test event queued**.

The synthetic route test uses a generated message with these values:

- Subject: `Hello from mailwebhook`
- From: `Tester <tester@example.com>`
- To: `You <you@example.com>`
- Header: `x-test: ui`
- Text body: `This is a synthetic test message.`
- Attachments: none

This tests the route pipeline and endpoint delivery. It does not validate provider-specific parsing, Gmail labels, Microsoft folders, IMAP UID behavior, or attachment extraction.

### 3. Check Events

Open **Events** and select the test event.

Use **Event Details** to confirm:

- Status
- Route
- Attempts
- Message-Id
- Subject
- From
- To

Use **Delivery Attempts History** to confirm:

- Attempt time
- Delivery status
- Latency
- Response status

## Payload inspection checklist

For a Generic JSON route, inspect these fields first:

| Field | What to check |
| --- | --- |
| `schema.name` | Should be `mailwebhook.generic`. |
| `schema.version` | Should be `1`. |
| `event.id` | Unique MailWebhook event id for this delivery. |
| `event.route_id` | Route that produced this payload. |
| `message.message_id` | Original email Message-Id when available, synthetic id for test events. |
| `message.subject` | Normalized subject from the email. |
| `message.from` and `message.to` | Normalized people arrays with email addresses. |
| `body.text` and `body.html` | Parsed body fields when present. |
| `body.attachments` | Attachment descriptors only. Download URLs are not embedded. |
| `meta.source` | Mailbox source such as `imap`, `gmail`, `hosted`, or `ms365` for real mailbox events. |

See [Generic JSON] for the full schema and determinism rules.

## Attachment checks

Generic JSON includes attachment descriptors in `body.attachments`.

It does not embed files or download URLs in the webhook body.

For each attachment descriptor, check:

- `id`
- `filename`
- `content_type`
- `size`
- `is_inline`
- `sha256`, when present

Use the attachment download API later to mint a short-lived download URL for an attachment.

## Common issues

### Webhook Preview says no request capture is available

The event has not reached a captured delivery attempt yet.

Check:

- The endpoint step was saved.
- The route is enabled.
- The message was sent after the preview started.
- The endpoint is public and reachable.
- The event is no longer pending.

### The response is not successful

Open **Response** in **Webhook Preview** or **Delivery Attempts History** in **Events**.

Check the endpoint response status and body. Fix the endpoint, then use **Replay Event** from **Events** when you want to deliver the same event again.

### The payload does not match the Generic JSON reference

Check the route pipeline.

The Generic JSON payload requires this final mapper:

```json
{
  "steps": [
    {
      "name": "map.generic_json",
      "args": {}
    }
  ]
}
```

If the route uses `map.custom_json`, `map.slack_simple`, or `map.telegram_simple`, the body follows that mapper instead.

### Route-level Send Test does not match my real email

**Send Test** on a route uses a synthetic message. It is useful for endpoint and mapper checks.

Send a normal email to the mailbox when you need to inspect provider-real message fields, labels, folders, headers, or attachments.

## Related docs

- [Receive your first inbound email webhook]
- [Generic JSON]
- [Generic JSON example]
- [Routes]
- [Rules]
- [Pipeline]
- [Endpoints]

[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Generic JSON example]: {% link docs/routes/pipeline/generic_json/example.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Endpoints]: {% link docs/endpoints.md %}

