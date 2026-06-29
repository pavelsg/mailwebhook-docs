---
title: Replace IMAP polling
parent: Migration
nav_order: 1
description: "Replace a custom IMAP polling script with MailWebhook using IMAP mailbox setup, route rules, Generic JSON payloads, signed delivery, idempotency, and rollout checks."
permalink: /docs/migration/replace-imap-polling/
---

# Replace an IMAP polling script with MailWebhook
{: .no_toc }

Use this guide when an existing script logs in to an IMAP inbox, stores UID state, parses new messages, and calls an internal service.

MailWebhook can take over the mailbox polling, UID cursor, parsing, route matching, delivery retries, replay, signatures, and attachment metadata. Your application keeps the business logic behind a webhook endpoint.

{: .note }
For product context, see [IMAP to webhook](https://www.mailwebhook.com/imap-to-webhook) and [Email to Webhook](https://www.mailwebhook.com/email-to-webhook).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Replace the IMAP polling loop with a MailWebhook IMAP mailbox, route, and endpoint.

Keep your existing message-processing logic behind an HTTP endpoint. Start with `map.generic_json` so the receiver gets the normalized email shape, then move to `map.custom_json` only if the receiver needs a smaller or route-specific body.

The migration path is:

1. Connect the same mailbox as an IMAP source.
2. Create an endpoint for your existing processor.
3. Create a route rule that matches the messages the script used to process.
4. Use a Generic JSON pipeline.
5. Run both systems in a controlled validation window.
6. Pause the old poller after MailWebhook events and receiver records match.

## When to use this

Use this migration guide when your current integration:

- Polls an IMAP folder on a schedule.
- Stores a last-seen UID, timestamp, message id, or local checkpoint.
- Parses subject, sender, body, headers, or attachments.
- Calls an internal API after parsing each message.
- Retries failed processing with local logic or a job queue.
- Needs a cleaner delivery and replay trail.

If your current system receives forwarded emails through a hosted mailbox or provider-specific API, start with the mailbox setup guide that matches that source.

## What changes

The main change is ownership of the polling loop.

| Old IMAP script responsibility | MailWebhook replacement |
| --- | --- |
| Connect to IMAP host and folder | IMAP mailbox source |
| Store UID cursor | MailWebhook IMAP cursor |
| Decide which messages count | Route rule JSON |
| Parse the email | MailWebhook normalized message model |
| Build downstream payload | Route pipeline mapper |
| Call the application | Endpoint JSON `POST` |
| Retry temporary failures | Delivery retry and replay |
| Prevent duplicate processing | `X-Idempotency-Key` and receiver storage |
| Fetch files | Attachment metadata plus attachment download API |

Your receiver should still own domain-specific work such as account lookup, CRM writes, ticket creation, database updates, and human review queues.

## Prerequisites

You need:

- The IMAP host, port, SSL/TLS setting, username, password or app password, and folder name.
- A MailWebhook project with connected-mailbox capacity available.
- A public HTTP or HTTPS endpoint that accepts JSON `POST` requests.
- A receiver that can verify signatures, store idempotency keys, and return `2xx` after accepting work.
- A test email that represents the messages the old script normally processes.

MailWebhook endpoint URLs must resolve to public addresses. Private, loopback, link-local, reserved, and multicast targets are blocked.

## 1. Inventory the old poller

Before creating the MailWebhook route, write down what the script uses from each email.

Capture:

- IMAP host, folder, and polling interval.
- Sender, recipient, subject, and header filters.
- Whether the script processes only unread mail or all new UIDs.
- Body preference, such as plain text or HTML.
- Attachment rules, such as required MIME types or file names.
- The downstream API body the script sends today.
- The duplicate guard, such as message id, UID, or a local job id.
- The retry behavior for `429`, `500`, `502`, `503`, and timeouts.

This inventory becomes the route rule, pipeline choice, and receiver contract.

## 2. Connect the IMAP mailbox

Open **Mailboxes** and add an IMAP mailbox.

Use the same mailbox and folder the old poller reads from:

| Field | Example |
| --- | --- |
| Email / Username | `intake@example.com` |
| Host | `imap.example.com` |
| Port | `993` |
| Use SSL/TLS | Enabled |
| Folder | `INBOX` |
| Poll Interval (seconds) | `300` |

Click **Test Connection** before saving.

After save, MailWebhook selects the configured folder and stores a UID cursor. The first live poll seeds from the current highest UID and waits for newer messages. Send a new test email after setup when you want to verify live routing.

Use mailbox backfill only when you intentionally need older messages that arrived before MailWebhook was connected.

## 3. Create the receiver endpoint

Create or expose a webhook endpoint in your application.

Example endpoint settings:

| Field | Value |
| --- | --- |
| Webhook URL | `https://app.example.com/mailwebhook/imap-intake` |
| Custom header | `Authorization: Bearer intake-token` |
| Timeout | Keep the default unless your receiver needs a different project standard. |

The receiver should:

- Read the raw request body before parsing JSON if it verifies `X-MailWebhook-Signature`.
- Store `X-Idempotency-Key` before creating downstream records.
- Return `2xx` only after it stores or enqueues the work.
- Return a retryable status when another delivery attempt can help.
- Treat replay of the same event as the same processing attempt.

## 4. Create the route rule

Translate the old script filters into route rule JSON.

For a dedicated intake inbox, start with the recipient:

```json
{
  "to_emails": ["intake@example.com"]
}
```

For a shared inbox, narrow the rule:

```json
{
  "to_emails": ["ops@example.com"],
  "from_domains": ["vendor.example"],
  "subject_contains": ["invoice", "receipt", "statement"],
  "none": [
    {
      "from_emails": ["noreply@vendor.example"]
    }
  ]
}
```

Top-level populated fields are AND-ed. In the second example, the message must go to `ops@example.com`, come from `vendor.example`, include one of the subject terms, and avoid the blocked sender.

## 5. Start with Generic JSON

Use Generic JSON during migration because it keeps the full normalized message shape available to the receiver.

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

MailWebhook posts the mapper output as the exact JSON request body. It does not wrap the payload in another object.

The payload includes the message, body, metadata, and attachment descriptors:

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
    "message_id": "<vendor-message@example.com>",
    "message_id_type": "original",
    "subject": "Invoice 1042",
    "date": "2026-06-28T12:00:00Z",
    "from": [{ "email": "billing@vendor.example" }],
    "to": [{ "email": "ops@example.com" }]
  },
  "body": {
    "attachments": [
      {
        "id": "att_01",
        "filename": "invoice-1042.pdf",
        "content_type": "application/pdf",
        "size": 58213,
        "sha256": "..."
      }
    ],
    "text": "Invoice 1042 is attached."
  },
  "meta": {
    "source": "imap",
    "raw_size_bytes": 4096,
    "received_at": "2026-06-28T12:00:00Z"
  }
}
```

Attachment file bytes are not embedded in the webhook body. Fetch them through the MailWebhook attachment download API when the receiver needs the file.

## Full route JSON example

```json
{
  "name": "IMAP intake replacement",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["ops@example.com"],
    "from_domains": ["vendor.example"],
    "subject_contains": ["invoice", "receipt", "statement"]
  },
  "pipeline": {
    "steps": [
      {
        "name": "map.generic_json",
        "args": {}
      }
    ]
  }
}
```

## Endpoint behavior

MailWebhook delivers the route pipeline output as a JSON `POST` request.

Delivery headers include:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Content-Type: application/json`
- Any custom endpoint headers you configured

Use `X-Idempotency-Key` as the primary duplicate guard in the receiver. MailWebhook may retry retryable failures and can replay an event after you request replay.

If your old script used IMAP UID as the only duplicate key, move the durable duplicate guard to the webhook receiver. UIDs are mailbox-folder state. The webhook receiver should deduplicate by MailWebhook delivery idempotency key, event id, or a business key from the payload.

## Rollout plan

Use a staged cutover so you can compare behavior before pausing the old poller.

1. Connect the IMAP mailbox in MailWebhook.
2. Keep the old poller active until you understand the first MailWebhook events.
3. Send a provider-real test email after setup.
4. Compare the old script output with the MailWebhook request body.
5. Update the route rule until it matches the same message set.
6. Add receiver-side idempotency before sending production traffic.
7. Pause the old poller.
8. Watch MailWebhook **Events** and **Delivery Attempts History** during the first production window.
9. Keep the old poller disabled but recoverable until the migration is stable.

If both systems can process the same live mailbox during validation, make the old path read-only or point one path at a test receiver. Avoid creating duplicate records in the downstream system.

## Validation checklist

Before cutover, confirm:

- The IMAP mailbox connection test succeeds.
- The MailWebhook mailbox is enabled.
- The selected folder is the folder the old poller used.
- A new email creates a MailWebhook event.
- The route rule matches the event you expect.
- The delivered JSON has `schema.name` set to `mailwebhook.generic`.
- The payload has `meta.source` set to `imap`.
- The receiver verifies `X-MailWebhook-Signature` against the raw body bytes.
- The receiver stores `X-Idempotency-Key`.
- A replay of the same event does not create a duplicate record.
- Attachment files, if needed, are fetched through the attachment download API.

## Common migration issues

### Existing messages do not appear

Normal IMAP polling starts from the current highest UID when the mailbox is first polled. Send a new test email after setup.

Use backfill when you intentionally need older messages.

### The route does not match the same messages as the script

Compare the old filters with the normalized route rule inputs.

Common differences:

- The old script matched a forwarded address, while the route checks the final recipient.
- The old script used case-sensitive subject checks.
- The old script ignored internal senders that the route currently allows.
- The route has a `none` branch that blocks more messages than intended.

Use the route troubleshooting guide before changing the receiver.

### The receiver creates duplicates

Store `X-Idempotency-Key` before creating the downstream record. Treat repeated deliveries with the same idempotency key as the same attempt.

Do this even if the old script used IMAP UID, because webhook retries and replay happen outside the old polling loop.

### Signature verification fails

Verify `X-MailWebhook-Signature` against the exact raw request body bytes.

Do not verify against parsed JSON, pretty-printed JSON, decoded text, or a reserialized copy of the body.

### Attachments are missing

Generic JSON includes attachment metadata under the `attachments` key inside the `body` object. It does not inline file bytes.

Fetch the file through the attachment download API with your MailWebhook API key, then upload it to your downstream system.

### Polling is slower than the old script

The IMAP mailbox accepts a polling interval between `60` and `3600` seconds, subject to the plan's polling floor. If your saved interval is lower than the plan allows, the plan floor can make polling less frequent.

## Related docs

- [IMAP mailbox configuration]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Rules]
- [Pipeline]
- [Generic JSON]
- [Custom JSON]
- [Endpoints]
- [Webhook payload reference]
- [Verify signed webhook deliveries]
- [Webhook retries and replay]
- [Fetch email attachments from webhook payloads]
- [API keys and attachment downloads]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]
- [IMAP folder and UID issues]

[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[IMAP folder and UID issues]: {% link docs/troubleshooting/imap-folder-and-uid-issues.md %}
