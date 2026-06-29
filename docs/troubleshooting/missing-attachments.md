---
title: Missing attachments
parent: Troubleshooting
nav_order: 4
description: "Troubleshoot missing MailWebhook email attachments by checking webhook payload descriptors, Mailbox Preview limitations, route transforms, Custom JSON mapping, MIME policy, and attachment download URL calls."
permalink: /docs/troubleshooting/missing-attachments/
---

# Troubleshoot missing email attachments
{: .no_toc }

Use this guide when an email had a file attached, but your webhook payload, Custom JSON output, or attachment download step does not show the file you expected.

The fastest path is to separate two cases: the attachment descriptor is missing from the webhook JSON, or the descriptor exists but the file download step fails.

{: .note }
Working with structured email payloads? See [Email to JSON](https://www.mailwebhook.com/email-to-json) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

MailWebhook does not embed attachment file bytes or download URLs in the webhook body.

Generic JSON includes attachment descriptors in `body.attachments`. Custom JSON can include attachment metadata only when the route maps `message.attachments` or selected fields from it.

To download a file, use `message.message_id` from the same payload and the attachment descriptor `id`:

```text
GET /v1/messages/{encoded_message_id}/attachments/{attachment_id}/url
X-API-Key: <project_api_key>
```

If `body.attachments` is empty in a delivered Generic JSON payload, check the source email, Mailbox Preview limitations, route transforms, and MIME policy. If descriptors exist but download fails, check the message id, attachment id, API key, URL lifetime, and URL reuse.

## Symptoms

Common symptoms include:

- `body.attachments` is an empty array in a Generic JSON payload.
- A Custom JSON payload has no attachment fields.
- **Mailbox Preview** shows no attachments.
- The receiver expected base64 file content in the webhook body.
- The receiver expected a direct download URL inside each attachment descriptor.
- The attachment download URL API returns `messages.not_found`.
- The attachment download URL API returns `attachments.not_found`.
- A generated download URL returns `attachments.expired` or `attachments.reused`.
- An event is `discarded` with a MIME or attachment policy reason.

## Most common causes

### You are looking at Mailbox Preview

Mailbox Preview is useful for route and payload checks, but it does not include attachments.

Use **Webhook Preview** during onboarding or **Events** and **Delivery Attempts History** for the delivered request body. Those views show the route pipeline output that was sent to the endpoint.

### The payload contains descriptors, not files

Attachment file bytes are stored separately from the webhook JSON.

A Generic JSON attachment descriptor looks like this:

```json
{
  "id": "att-1",
  "filename": "invoice.pdf",
  "content_type": "application/pdf",
  "size": 93259,
  "is_inline": false,
  "sha256": "059a0f5260487bbe663994de1fd641401fec76ac9f6bddfe5b53ae60d4bb2d86"
}
```

The descriptor is enough to request a short-lived download URL later. The webhook body does not include base64 content or a permanent URL.

### The route uses Custom JSON and did not map attachments

Custom JSON sends only the fields defined by the route mapper.

If a downstream system needs to download attachments later, include the message id and attachment metadata in the Custom JSON output:

```json
{
  "output": {
    "message_id": { "var": "message.message_id" },
    "attachments": { "var": "message.attachments" }
  }
}
```

For a smaller payload, map only the fields your receiver needs, such as `message.attachments[0].id`, `filename`, `content_type`, `size`, and `sha256`.

### The route pipeline removed attachments before mapping

Transform steps run before the final `map.*` step.

These pipeline choices can remove attachments from the payload:

- `remove_fields` with `attachments`.
- `strip_attachments_if` with MIME, size, or filename conditions that match the file.
- The UI route option that adds `strip_attachments_if` for `image/*`.

Check the route pipeline before changing receiver code.

### The email did not contain an extracted attachment candidate

MailWebhook extracts MIME parts that are marked as attachments. Inline parts can also be extracted when they have a filename, or when they are inline images with a `Content-ID`.

Some email content is not an attachment:

- A remote image loaded by URL in the HTML body.
- A link to a file in Google Drive, OneDrive, Dropbox, or another service.
- A logo referenced by external URL.
- Text pasted into the email body.

Those items can appear in `body.html` or `body.text`, but they do not become attachment descriptors.

### The attachment was over policy limits

The current worker attachment policy accepts candidates up to 25 MB each and up to 50 MB total per message.

An individual attachment over the per-file limit is skipped during candidate buffering. If every attachment candidate is skipped, the delivered payload can have an empty attachment array.

If total candidate size exceeds the message-level limit, or attachment storage quota is reached, attachment extraction fails before delivery events are created.

### MIME policy discarded the message

MailWebhook verifies attachment content type during ingest. Attachments must use allowed MIME prefixes such as `application/`, `image/`, or `text/`, and the detected content type must match the declared type.

When attachment policy requires a discard, MailWebhook creates a `discarded` event for matched routes and skips delivery. The event `last_error` uses a `mime_discard:<reason>` value, such as `mime_discard:content_type_mismatch` or `mime_discard:disallowed_attachment_type`.

### The download request uses the wrong id

The attachment URL endpoint uses the email Message-ID from the payload path, not the MailWebhook event id.

Use:

- `message.message_id` as `{message_id}` after URL encoding.
- The attachment descriptor `id` as `{attachment_id}`.
- A project API key in `X-API-Key`.

The message and attachment must belong to the same project as the API key.

### The generated download URL expired or was already used

The attachment URL API returns a short-lived URL.

For app proxy downloads, default generated URLs are single use. A second fetch can return `attachments.reused`. If a worker, virus scanner, preview tool, or retry consumes the URL first, request a fresh URL before downloading again.

Use `multi_use=true` only when the same generated URL must be reused briefly.

## Step-by-step checks

### 1. Identify which failure you have

Start with the delivered webhook body.

| What you see | Meaning | Next check |
| --- | --- | --- |
| `body.attachments` contains descriptors | MailWebhook extracted attachments. | Debug the download URL request. |
| `body.attachments` is `[]` | Generic JSON received no attachments after extraction and transforms. | Check source email, pipeline, and policy. |
| Custom JSON has no attachment fields | The mapper did not output them. | Check the Custom JSON route config. |
| No event exists | The route did not match, ingest was blocked, or delivery did not reach event creation. | Check route matching and ingest status. |
| Event status is `discarded` | MIME or attachment policy blocked delivery. | Check `last_error`. |

### 2. Inspect the delivered payload

Use **Webhook Preview** or open **Events**, select the event, then inspect **Delivery Attempts History**.

For Generic JSON, check:

```json
{
  "body": {
    "attachments": []
  }
}
```

If the delivery attempt body has attachment descriptors, the webhook payload is correct and the remaining issue is in your receiver or download flow.

Avoid using **Mailbox Preview** for this check because it does not include attachments.

### 3. Check the source email

Confirm the source message has a real MIME attachment.

Check:

- The file was attached to the email before sending.
- The file was not only linked from cloud storage.
- The attachment is visible in the mailbox provider.
- The message was sent after the mailbox was connected or after backfill was started.
- The route matched the message that had the attachment.

If the route uses `attachments_mime`, a missing or unsupported attachment can also prevent the route from matching. Use the route-rule troubleshooting page for that path.

### 4. Check route transforms

Open the route pipeline JSON.

Look for:

```json
{
  "name": "remove_fields",
  "args": {
    "paths": ["attachments"]
  }
}
```

And:

```json
{
  "name": "strip_attachments_if",
  "args": {
    "mime_in": ["image/*"]
  }
}
```

If you only need a diagnostic test, temporarily use a minimal Generic JSON pipeline:

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

Send a new test email with a small PDF attachment and inspect the delivered body.

### 5. Check Custom JSON mapping

If the route uses `map.custom_json`, confirm the output maps attachment fields.

Minimum download-ready output:

```json
{
  "output": {
    "message_id": { "var": "message.message_id" },
    "attachments": {
      "map": {
        "over": { "var": "message.attachments" },
        "as": "attachment",
        "do": {
          "id": { "var": "attachment.id" },
          "filename": { "var": "attachment.filename" },
          "content_type": { "var": "attachment.content_type" },
          "size": { "var": "attachment.size" },
          "sha256": { "var": "attachment.sha256" }
        }
      }
    }
  }
}
```

The exact output shape can be different, but it must preserve enough data to call the download URL API later.

### 6. Check event status and last error

Open **Events** and inspect the event for the message.

Check:

- `delivered`: the route matched and the endpoint returned `2xx`.
- `retry` or `dead`: inspect delivery attempts first.
- `discarded`: delivery was skipped because MIME parsing or attachment policy blocked the message.
- `last_error`: look for `mime_discard:<reason>`.

For discarded events, send a smaller, allowed attachment type as a control test. A small PDF is a good first check.

### 7. Check the attachment URL API call

If the descriptor exists, build the URL request from the same payload:

```bash
curl "https://app.mailwebhook.com/v1/messages/{encoded_message_id}/attachments/{attachment_id}/url" \
  -H "X-API-Key: <project_api_key>"
```

Check:

- `{encoded_message_id}` is the URL-encoded `message.message_id` value.
- `{attachment_id}` is the descriptor `id`.
- The API key belongs to the same project.
- The call is made from your backend.
- The response `url`, `method`, and `headers` are used exactly.

### 8. Request a fresh URL before each download attempt

Generated download URLs are temporary credentials.

If the download URL fails:

- Request a fresh URL.
- Fetch it before `expires_at`.
- Avoid logging or sharing the returned URL.
- Use `multi_use=true` only when brief reuse is required.
- Do not use the project API key to fetch the returned download URL unless the URL response includes headers that require it.

## Expected result

After the issue is fixed:

- Generic JSON payloads with extracted attachments include descriptors in `body.attachments`.
- Custom JSON payloads include the mapped attachment fields your receiver needs.
- The receiver stores `message.message_id` and each attachment `id`.
- The backend can request a short-lived download URL with `X-API-Key`.
- The backend downloads the file before expiration.
- Replays or new test emails show the same route and payload behavior.

## Related docs

- [Fetch email attachments from webhook payloads]
- [API keys and attachment downloads]
- [Webhook payload reference]
- [Generic JSON]
- [Custom JSON]
- [Transform Steps]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]

[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[Transform Steps]: {% link docs/routes/pipeline/transform_steps.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
