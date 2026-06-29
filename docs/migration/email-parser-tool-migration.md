---
title: Email parser tool migration
parent: Migration
nav_order: 3
description: "Migrate from an email parser tool to MailWebhook using mailbox intake, route rules, Generic JSON, Custom JSON mapping, extraction helpers, signed delivery, and rollout checks."
permalink: /docs/migration/email-parser-tool-migration/
---

# Migrate from an email parser tool to MailWebhook
{: .no_toc }

Use this guide when a current email parser tool extracts fields from forwarded messages and sends structured data to an app, webhook, spreadsheet, CRM, or operations workflow.

MailWebhook moves the workflow closer to an email-to-webhook pipeline. You connect the mailbox source, define route rules, choose Generic JSON or Custom JSON, and send a signed JSON webhook to your receiver.

{: .note }
For structured payload positioning, see [Email to JSON](https://www.mailwebhook.com/email-to-json). For the broader email-to-webhook workflow, see [Email to Webhook](https://www.mailwebhook.com/email-to-webhook).

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Do not migrate an email parser tool by copying every old template rule into one receiver.

Start with the workflow contract:

1. Choose the mailbox source that receives the emails.
2. Recreate message selection with route rules.
3. Start with `map.generic_json` so you can inspect normalized messages.
4. Move stable field mapping into `map.custom_json`.
5. Use extraction helpers only where the email format is stable enough.
6. Keep business validation and fallback handling in your receiver.
7. Verify `X-MailWebhook-Signature`.
8. Store `X-Idempotency-Key` before creating downstream records.

MailWebhook is a good fit when the end state should be a signed JSON webhook event with clear routing, mapping, delivery history, retries, replay, and attachment metadata.

## What changes during migration

Most parser tools center the workflow around extraction templates. MailWebhook centers it around mailbox intake, route matching, and webhook payload output.

| Old parser-tool area | MailWebhook migration target |
| --- | --- |
| Parser inbox or forwarding address | Gmail, Microsoft 365, Office 365, Outlook, IMAP, or hosted mailbox source |
| Template selection | Route rule JSON |
| Extracted field list | Custom JSON output |
| Parser-specific field names | MailWebhook mapper variables and output keys |
| Parser webhooks or exports | MailWebhook endpoint delivery |
| File fields or attachment exports | Attachment metadata plus attachment download API |
| Parser retry or task logs | Delivery Attempts History and replay |
| Duplicate task handling | `X-Idempotency-Key` plus receiver-side storage |

Some old templates should stay in application code for a while. Move stable routing and payload shaping into MailWebhook first, then remove old parser logic after the receiver behaves the same way on real emails.

## Prerequisites

You need:

- A MailWebhook project.
- A supported mailbox source for the incoming messages.
- A public HTTP or HTTPS receiver endpoint.
- A list of the old parser templates and extracted fields.
- Sample source emails for each important template.
- A receiver that accepts JSON `POST` requests.
- A plan for duplicate handling and replay.

MailWebhook endpoint URLs must resolve to public addresses. Private, loopback, link-local, reserved, and multicast targets are blocked.

## 1. Inventory the parser workflow

List what the current parser tool does before creating MailWebhook routes.

Capture:

- How emails reach the parser tool today.
- Template names or workflow names.
- Sender, recipient, subject, and attachment filters.
- Extracted fields and their downstream names.
- Whether a field comes from body text, HTML, a table, an attachment name, or a link.
- Required fields versus optional fields.
- Current webhook, export, or automation destination.
- Duplicate handling.
- Failure behavior when a field is missing.

Group templates by workflow, not by visual similarity. For example, invoice intake, lead intake, support intake, and alert routing usually deserve separate routes or endpoints.

## 2. Choose the mailbox source

Pick the source that should receive the email before routing starts.

| Source | Use when |
| --- | --- |
| Gmail or Google Workspace | The workflow already lives in a Gmail or Google Workspace inbox. |
| Microsoft 365, Office 365, or Outlook | The workflow belongs to an Outlook or Microsoft mailbox. |
| IMAP | The provider exposes a standard IMAP folder. |
| Hosted mailbox | A dedicated MailWebhook-hosted intake address is simpler than connecting an existing mailbox. |
| Loopback | You need a quick test route before using production mail. |

If the old parser tool used a generated parser address, update forwarding rules, application settings, groups, or aliases so matching emails can reach the MailWebhook source.

## 3. Create the receiver endpoint

Create a MailWebhook endpoint for your receiving service.

Example:

| Field | Value |
| --- | --- |
| Webhook URL | `https://app.example.com/mailwebhook/parser-migration` |
| Custom header | `Authorization: Bearer intake-token` |
| Timeout | Keep the default unless your receiver needs a different project standard. |

The receiver should:

- Read the raw request body before parsing JSON if it verifies `X-MailWebhook-Signature`.
- Store `X-Idempotency-Key` before creating downstream records.
- Return `2xx` only after it stores or enqueues the work.
- Return a retryable status only when another delivery attempt can help.
- Log missing fields during migration instead of silently creating bad records.

## 4. Recreate template selection as route rules

Use route rules to select which emails enter a workflow.

For a dedicated parser inbox:

```json
{
  "to_emails": ["parser-intake@example.com"]
}
```

For a vendor-specific invoice workflow:

```json
{
  "to_emails": ["ap@example.com"],
  "from_domains": ["vendor.example"],
  "subject_contains": ["invoice", "receipt"],
  "attachments_mime": ["application/pdf"],
  "none": [
    {
      "subject_contains": ["test", "void"]
    }
  ]
}
```

Top-level populated fields are AND-ed. In the second example, the message must match the recipient, sender domain, subject terms, and attachment MIME type, while avoiding subjects that mark a message as test or void.

Keep route rules narrow enough to avoid sending unrelated messages to a mapper that expects a specific format.

## 5. Start with Generic JSON

Use `map.generic_json` during the first pass.

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

Generic JSON helps you compare real emails against the old parser output.

Check:

- `schema.name` is `mailwebhook.generic`.
- `message.subject` matches the source email.
- `message.from` and `message.to` contain the expected addresses.
- The `body` object contains the text or HTML content your old parser used.
- Attachment descriptors appear under the `attachments` key inside the `body` object.
- `meta.source` matches the mailbox source, such as `gmail`, `ms365`, `imap`, or `hosted`.

## 6. Map extracted fields with Custom JSON

Use `map.custom_json` when the receiver needs a field-specific body.

This example migrates a common key/value parser template:

```json
{
  "steps": [
    {
      "name": "map.custom_json",
      "args": {
        "version": "v1",
        "vars": [
          {
            "name": "kv",
            "expr": {
              "call.extract.key_value_pairs": {
                "mode": "auto",
                "html": { "var": "message.html" },
                "text": { "var": "message.text" }
              }
            }
          },
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
          "type": "parser_migration.invoice",
          "event_id": { "var": "ctx.event_id" },
          "source_type": { "var": "ctx.source_type" },
          "message_id": { "var": "message.message_id" },
          "vendor_email": { "var": "message.from[0].email" },
          "invoice": {
            "number": { "var": "vars.kv.values.invoice_number" },
            "date": { "var": "vars.kv.values.invoice_date" },
            "total": { "var": "vars.kv.values.total" }
          },
          "subject": { "var": "message.subject" },
          "body": { "var": "vars.plain_text" },
          "raw_fields": { "var": "vars.kv.items" },
          "attachments": { "var": "message.attachments" }
        }
      }
    }
  ]
}
```

Use provider-real sample emails before relying on an extraction helper. Parser tools often hide format drift. MailWebhook will emit `null` for missing paths, so your receiver should validate required fields before creating downstream records.

## Full route JSON example

```json
{
  "name": "Parser migration invoice intake",
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
        "name": "map.custom_json",
        "args": {
          "version": "v1",
          "vars": [
            {
              "name": "kv",
              "expr": {
                "call.extract.key_value_pairs": {
                  "mode": "auto",
                  "html": { "var": "message.html" },
                  "text": { "var": "message.text" }
                }
              }
            }
          ],
          "output": {
            "type": "parser_migration.invoice",
            "event_id": { "var": "ctx.event_id" },
            "message_id": { "var": "message.message_id" },
            "invoice_number": { "var": "vars.kv.values.invoice_number" },
            "total": { "var": "vars.kv.values.total" },
            "raw_fields": { "var": "vars.kv.items" },
            "attachments": { "var": "message.attachments" }
          }
        }
      }
    ]
  }
}
```

`map.custom_json` must be the final pipeline step.

## Attachment migration

Parser tools sometimes pass attachment files directly to another app. MailWebhook sends attachment metadata in the webhook body and stores file bytes separately.

The migration pattern is:

1. Include `message.message_id` and attachment metadata in your Custom JSON output, or use Generic JSON.
2. Store the attachment `id`, filename, content type, size, and checksum when available.
3. Request a short-lived download URL from the attachment download API with a MailWebhook API key.
4. Fetch the file server-side.
5. Upload the file to the downstream system or attach it to the created record.

Keep project API keys out of browser code and client-side logs.

## Endpoint behavior

MailWebhook delivers the route pipeline output as a JSON `POST` request.

Delivery headers include:

- `X-MailWebhook-Signature`
- `X-Idempotency-Key`
- `Content-Type: application/json`
- Any custom endpoint headers you configured

The request body is exactly the route pipeline output. MailWebhook does not add another wrapper around the mapper result.

Verify the signature against the raw request body bytes. Use the idempotency key as the duplicate guard for retries and replay.

## Cutover plan

Use a staged migration so parser-tool output and MailWebhook output can be compared.

1. Keep the old parser workflow active.
2. Connect the MailWebhook mailbox source.
3. Create the endpoint and a narrow route rule.
4. Start with Generic JSON and send provider-real test emails.
5. Compare the MailWebhook payload with the old parser output.
6. Add a Custom JSON mapper for stable fields.
7. Update the receiver to validate required fields and store idempotency keys.
8. Move forwarding, mailbox rules, aliases, or application settings to MailWebhook.
9. Watch **Events** and **Delivery Attempts History**.
10. Disable the old parser workflow after downstream records match.

If both paths run at once, make sure the downstream system can dedupe records or point one path at a test destination.

## Validation checklist

Before cutover, confirm:

- The selected mailbox source receives a new test email.
- The route rule matches only intended messages.
- Generic JSON payloads have `schema.name` set to `mailwebhook.generic`.
- The Custom JSON mapper saves without validation errors.
- Extracted fields match provider-real sample emails.
- Required missing fields are rejected or queued for review by the receiver.
- The receiver verifies `X-MailWebhook-Signature` against raw body bytes.
- The receiver stores `X-Idempotency-Key`.
- Replay of the same event does not create a duplicate record.
- Attachments are fetched server-side through the attachment download API when needed.

## Common migration issues

### Extracted fields are `null`

Missing Custom JSON paths evaluate to `null`.

Common causes:

- The source email label changed.
- The old parser template used a visual selector that does not map to key/value text.
- The email contains HTML tables and should use table extraction or receiver-side parsing.
- The test email is not the same shape as production mail.

Keep `raw_fields` during migration so you can see which keys the helper found.

### The route matches too many messages

Tighten the route rule before changing the mapper.

Use recipient, sender domain, subject terms, attachment MIME type, and `none` rules to keep unrelated emails out of workflow-specific mappers.

### Attachments are listed, but the downstream app has no file

MailWebhook webhook JSON includes attachment metadata. It does not inline file bytes.

Fetch file content through the attachment download API after the receiver has accepted and stored the work item.

### Duplicate records appear

Store `X-Idempotency-Key` before creating downstream records.

MailWebhook may retry retryable failures and can replay an event. Repeated deliveries with the same idempotency key should be treated as the same processing attempt.

### Signature verification fails

Verify `X-MailWebhook-Signature` against the exact raw request body bytes.

Do not verify against parsed JSON, pretty-printed JSON, decoded text, or a reserialized copy of the body.

### The old parser handled messy documents better

Keep that parsing in the receiver or migrate those workflows later.

MailWebhook Custom JSON works best when the email structure is stable enough for route rules, normalized fields, extraction helpers, and explicit receiver validation.

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
- [JsonLogic-Style DSL]
- [Custom JSON recipes]
- [Webhook payload reference]
- [Endpoints]
- [Verify signed webhook deliveries]
- [Webhook retries and replay]
- [Fetch email attachments from webhook payloads]
- [API keys and attachment downloads]
- [Send a test email and inspect the payload]
- [Troubleshoot Custom JSON mapper errors]
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
[JsonLogic-Style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
[Custom JSON recipes]: {% link docs/routes/pipeline/custom_json/recipes.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Troubleshoot Custom JSON mapper errors]: {% link docs/troubleshooting/custom-json-mapper-errors.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
