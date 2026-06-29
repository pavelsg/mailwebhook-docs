---
title: SendGrid Inbound Parse migration
parent: Migration
nav_order: 2
description: "Migrate a SendGrid Inbound Parse workflow to MailWebhook with supported mailbox intake, route rules, JSON payload mapping, signed delivery, idempotency, and attachment handling."
permalink: /docs/migration/sendgrid-inbound-parse/
---

# Migrate from SendGrid Inbound Parse to MailWebhook
{: .no_toc }

Use this guide when an existing SendGrid Inbound Parse workflow posts inbound email to your application and you want to move that workflow to MailWebhook.

MailWebhook uses a different receiver contract. Instead of accepting SendGrid parse payloads at the same endpoint, you choose a MailWebhook mailbox source, define route rules, select a JSON mapper, and update your receiver to accept signed MailWebhook JSON.

{: .note }
For the commercial comparison page, see [SendGrid Inbound Parse alternative](https://www.mailwebhook.com/sendgrid-inbound-parse-alternative). For SendGrid product behavior, see Twilio SendGrid's [Inbound Email Parse Webhook](https://www.twilio.com/docs/sendgrid/for-developers/parsing-email/inbound-email) docs.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Do not point your existing SendGrid Inbound Parse receiver at MailWebhook unchanged.

Treat the migration as a small workflow redesign:

1. Choose the MailWebhook intake source.
2. Register the receiver endpoint.
3. Recreate routing logic with route rules.
4. Start with `map.generic_json` for validation.
5. Use `map.custom_json` when your receiver needs a smaller or transitional body.
6. Replace SendGrid request verification with `X-MailWebhook-Signature`.
7. Use `X-Idempotency-Key` to dedupe retries and replay.
8. Update attachment handling so the receiver reads metadata first and downloads file bytes only when needed.

This page is about SendGrid Inbound Parse for incoming email. It is not about SendGrid Event Webhook events such as delivered, opened, clicked, bounced, or spam-report notifications.

## What changes during migration

SendGrid Inbound Parse and MailWebhook solve a similar inbound-email job through different contracts.

| Existing SendGrid Inbound Parse area | MailWebhook migration target |
| --- | --- |
| MX-based receiving domain or subdomain | Gmail, Microsoft 365, Office 365, Outlook, IMAP, or MailWebhook-hosted mailbox source |
| Destination URL in SendGrid parse settings | MailWebhook endpoint record |
| SendGrid parse fields, multipart form parts, or raw MIME | Route pipeline output JSON |
| Receiver-side parser glue | Generic JSON or Custom JSON mapping |
| SendGrid parse security policy | `X-MailWebhook-Signature` plus endpoint static headers |
| Receiver duplicate handling | `X-Idempotency-Key` plus receiver-side storage |
| Multipart file parts or raw MIME attachment handling | Attachment metadata in JSON plus attachment download API |
| SendGrid retry behavior | MailWebhook delivery attempts, event history, and replay |

MailWebhook is not a SendGrid-style custom MX receiver in this guide. Choose one of the supported MailWebhook source types and route mail to that source.

## Prerequisites

You need:

- A MailWebhook project.
- A supported mailbox source for the messages that SendGrid Inbound Parse receives today.
- A public HTTP or HTTPS receiver endpoint.
- A receiver that can parse JSON request bodies.
- A route signing secret if the receiver verifies signed delivery.
- Sample SendGrid parse requests or logs from the current workflow.
- Test emails that represent the production messages you need to migrate.

MailWebhook endpoint URLs must resolve to public addresses. Private, loopback, link-local, reserved, and multicast targets are blocked.

## 1. Inventory the SendGrid workflow

Start by capturing what the current SendGrid setup does.

Record:

- Receiving domain, hostname, or subdomain.
- Current destination URL.
- Whether SendGrid sends parsed fields or raw MIME.
- Fields the receiver reads, such as sender, recipient, subject, headers, text, HTML, envelope, spam fields, and charsets.
- Attachment behavior, including whether the receiver expects multipart file parts.
- Any current request verification logic.
- Retry and failure handling in the receiver.
- Downstream records created by the workflow.

Also capture negative behavior:

- Messages the receiver ignores.
- Senders or subjects that are excluded.
- Attachments that are skipped.
- Payload fields that the application no longer uses.

This inventory becomes the MailWebhook source choice, route rule, mapper choice, and validation checklist.

## 2. Choose the MailWebhook intake source

Pick the source that matches how mail should enter MailWebhook.

| Source | Use when |
| --- | --- |
| Gmail or Google Workspace | The workflow belongs to an existing Gmail or Google Workspace inbox. |
| Microsoft 365, Office 365, or Outlook | The workflow belongs to an Outlook mailbox or Microsoft 365 mail flow. |
| IMAP | The mailbox provider exposes a standard IMAP folder. |
| Hosted mailbox | A dedicated MailWebhook-hosted address is cleaner than connecting an existing mailbox. |
| Loopback | You need a quick setup test before moving real mail. |

If your current SendGrid setup uses MX records for a parse subdomain, decide how mail will reach the selected MailWebhook source. That might mean forwarding, a provider rule, a group, a distribution list, an application setting, or a new hosted address.

Do not assume the old SendGrid MX configuration moves unchanged.

## 3. Create the receiver endpoint

Create a MailWebhook endpoint for the application that will receive the migrated workflow.

Example endpoint settings:

| Field | Value |
| --- | --- |
| Webhook URL | `https://app.example.com/mailwebhook/sendgrid-migration` |
| Custom header | `Authorization: Bearer intake-token` |
| Timeout | Keep the default unless your receiver needs a different project standard. |

Update the receiver before production cutover:

- Accept `POST` requests with `Content-Type: application/json`.
- Read the raw body bytes before parsing JSON if verifying `X-MailWebhook-Signature`.
- Store `X-Idempotency-Key` before creating downstream records.
- Return `2xx` only after durable acceptance.
- Return a retryable status only when another delivery attempt can help.

## 4. Recreate routing as route rules

Move SendGrid receiver-side filtering into MailWebhook route rules where possible.

For a dedicated intake address:

```json
{
  "to_emails": ["inbound@example.com"]
}
```

For a workflow that only processes certain senders and attachments:

```json
{
  "to_emails": ["ap@example.com"],
  "from_domains": ["vendor.example"],
  "subject_contains": ["invoice", "receipt"],
  "attachments_mime": ["application/pdf"],
  "none": [
    {
      "from_emails": ["noreply@vendor.example"]
    }
  ]
}
```

Top-level populated fields are AND-ed. In the second example, the message must match the recipient, sender domain, subject terms, and attachment MIME type, while avoiding the blocked sender.

Keep receiver-side business validation in your application. Route rules should decide which messages enter a workflow, not replace domain-specific checks.

## 5. Start with Generic JSON

Use `map.generic_json` for the first migration pass.

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

Generic JSON gives the receiver a stable normalized email shape:

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
    "subject": "Invoice 1042",
    "date": "2026-06-28T12:00:00Z",
    "from": [{ "email": "billing@vendor.example" }],
    "to": [{ "email": "ap@example.com" }]
  },
  "body": {
    "attachments": [
      {
        "id": "att_01",
        "filename": "invoice-1042.pdf",
        "content_type": "application/pdf",
        "size": 58213,
        "is_inline": false,
        "sha256": "..."
      }
    ],
    "text": "Invoice 1042 is attached."
  },
  "meta": {
    "source": "hosted",
    "raw_size_bytes": 4096,
    "received_at": "2026-06-28T12:00:00Z"
  }
}
```

The exact `meta.source` value depends on the mailbox source, such as `gmail`, `ms365`, `imap`, or `hosted`.

## 6. Use Custom JSON for a transitional body

After validation, use `map.custom_json` when your receiver should receive only specific fields.

This example emits a compact migration payload. It uses MailWebhook field names and keeps enough metadata for traceability:

```json
{
  "steps": [
    {
      "name": "map.custom_json",
      "args": {
        "version": "v1",
        "vars": [
          {
            "name": "plain_text",
            "expr": {
              "call.transform.html_to_text": {
                "html": { "var": "message.html" },
                "text": { "var": "message.text" }
              }
            }
          }
        ],
        "output": {
          "type": "sendgrid_parse_migration.email_received",
          "event_id": { "var": "ctx.event_id" },
          "source_type": { "var": "ctx.source_type" },
          "message_id": { "var": "message.message_id" },
          "from": { "var": "message.from[0].email" },
          "to": { "var": "message.to" },
          "subject": { "var": "message.subject" },
          "text": { "var": "vars.plain_text" },
          "headers": { "var": "message.headers" },
          "attachments": { "var": "message.attachments" }
        }
      }
    }
  ]
}
```

This is still a MailWebhook JSON body. It is not a SendGrid multipart parse payload.

## Full route JSON example

```json
{
  "name": "SendGrid Inbound Parse migration",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["ap@example.com"],
    "from_domains": ["vendor.example"],
    "subject_contains": ["invoice", "receipt"],
    "attachments_mime": ["application/pdf"]
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

Use the Generic JSON version during initial comparison. Switch the pipeline to Custom JSON after your receiver contract is stable.

## Attachment migration

SendGrid Inbound Parse workflows often expect attachment bytes inside the incoming parse request.

MailWebhook separates the metadata from the file download:

1. The webhook JSON includes attachment metadata.
2. Your receiver stores the message id and attachment ids.
3. Your backend requests a short-lived download URL with a MailWebhook API key.
4. Your backend downloads the file and sends it to the downstream system.

This keeps normal webhook requests smaller and lets your receiver fetch file bytes only when the workflow needs them.

## Endpoint behavior

MailWebhook delivers the route pipeline output as a JSON `POST` request.

Delivery headers include:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Content-Type: application/json`
- Any custom endpoint headers you configured

The request body is exactly the route pipeline output. MailWebhook does not add another envelope around the payload.

Verify `X-MailWebhook-Signature` against the raw request body bytes. Use `X-Idempotency-Key` as the receiver's duplicate guard across retries and replay.

## Cutover plan

Use a staged cutover so you can compare output before moving production traffic.

1. Keep the SendGrid workflow active.
2. Create the MailWebhook mailbox source, endpoint, route, and Generic JSON pipeline.
3. Send provider-real test emails through the selected MailWebhook source.
4. Compare MailWebhook events with existing SendGrid receiver logs.
5. Update route rules until the same message set is selected.
6. Update the receiver to accept the MailWebhook JSON contract.
7. Add idempotency storage before production traffic.
8. Move forwarding, mailbox rules, application settings, or address usage to MailWebhook.
9. Watch **Events** and **Delivery Attempts History**.
10. Keep the old SendGrid path disabled but recoverable until the migration is stable.

During parallel testing, avoid sending the same production message to both receivers unless the downstream system can safely dedupe it.

## Validation checklist

Before cutover, confirm:

- The MailWebhook source receives a new test email.
- The route rule matches only the intended messages.
- The request body is valid JSON.
- Generic JSON payloads have `schema.name` set to `mailwebhook.generic`.
- The receiver verifies `X-MailWebhook-Signature` against raw body bytes.
- The receiver stores `X-Idempotency-Key`.
- A replay of the same event does not create a duplicate downstream record.
- Attachment metadata contains the ids and filenames your workflow needs.
- Attachment downloads happen server-side with a MailWebhook API key.
- Failure responses appear in **Delivery Attempts History**.

## Common migration issues

### The existing SendGrid endpoint does not work unchanged

MailWebhook sends JSON route output. Existing SendGrid receiver code that expects multipart fields or raw MIME needs an adapter update.

Start with Generic JSON and log the delivered body before removing the old parser code.

### The route misses messages

Compare the route rule against the real message fields.

Common causes:

- The selected mailbox source receives a different recipient address than the old SendGrid parse domain.
- `from_domains` does not match forwarded sender behavior.
- `subject_contains` is too narrow.
- `attachments_mime` excludes a valid attachment type.
- A `none` rule blocks a sender that should be allowed.

### Attachments are present, but file bytes are missing

MailWebhook webhook JSON includes attachment metadata. It does not inline attachment bytes.

Use the attachment download API from your backend after signature verification and durable acceptance.

### Signature verification fails

MailWebhook uses `X-MailWebhook-Signature`, not SendGrid's request verification format.

Verify the exact raw request body bytes. Parsed or reserialized JSON changes the bytes and invalidates the signature.

### Duplicate records appear

Store `X-Idempotency-Key` before creating downstream records.

MailWebhook may retry retryable failures and can replay an event. Repeated deliveries with the same idempotency key should be treated as the same processing attempt.

### Traffic is mixed with SendGrid Event Webhook expectations

SendGrid Event Webhook is a separate SendGrid product surface for delivery and engagement events. This migration guide covers inbound email parsing.

If your current SendGrid integration is about delivered, opened, clicked, bounced, or spam-report events, this page is the wrong migration path.

## Related docs

- [Mailboxes]
- [Connect Gmail as a mailbox source]
- [Microsoft 365 mailbox setup]
- [IMAP mailbox configuration]
- [Create a hosted mailbox]
- [Rules]
- [Pipeline]
- [Generic JSON]
- [Custom JSON]
- [Webhook payload reference]
- [Endpoints]
- [Verify signed webhook deliveries]
- [Webhook retries and replay]
- [Fetch email attachments from webhook payloads]
- [API keys and attachment downloads]
- [Send a test email and inspect the payload]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]

[Mailboxes]: {% link docs/mailboxes.md %}
[Connect Gmail as a mailbox source]: {% link docs/mailboxes/gmail-setup.md %}
[Microsoft 365 mailbox setup]: {% link docs/mailboxes/microsoft-365-setup.md %}
[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Create a hosted mailbox]: {% link docs/mailboxes/hosted-mailbox-setup.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
