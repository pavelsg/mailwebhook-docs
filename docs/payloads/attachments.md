---
title: Attachments
parent: Payloads
nav_order: 2
description: "Fetch MailWebhook email attachments from webhook payload metadata using attachment ids, message ids, X-API-Key, and short-lived download URLs."
permalink: /docs/payloads/attachments/
---

# Fetch email attachments from webhook payloads
{: .no_toc }

MailWebhook sends attachment metadata in webhook payloads and stores file bytes separately.

Use this page when your receiver needs to find attachment descriptors in a payload, request a short-lived download URL, and fetch the file content.

{: .note }
For product context around structured attachment metadata, see [Email to JSON](https://www.mailwebhook.com/email-to-json). For the broader email routing workflow, see [Email to Webhook](https://www.mailwebhook.com/email-to-webhook). For API key handling, see [API keys and attachment downloads].

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Where attachments appear

Generic JSON includes attachment descriptors in `body.attachments`.

```json
{
  "message": {
    "message_id": "<3F8E62B7-2BD6-4EC8-AD74-7B775CD0DE81@gmail.com>"
  },
  "body": {
    "attachments": [
      {
        "id": "att-1",
        "filename": "invoice.pdf",
        "content_type": "application/pdf",
        "size": 93259,
        "is_inline": false,
        "sha256": "059a0f5260487bbe663994de1fd641401fec76ac9f6bddfe5b53ae60d4bb2d86"
      }
    ]
  }
}
```

Custom JSON routes can map attachment metadata from `message.attachments`. If a downstream system needs to download files later, include the email `message_id` and each attachment `id` in your custom output.

## Attachment descriptor fields

| Field | Meaning |
| --- | --- |
| `id` | Stable attachment identifier for the message. Use this as `{attachment_id}` in the download URL request. |
| `filename` | Sanitized filename presented by the parser and download API. |
| `content_type` | Attachment MIME type, such as `application/pdf`. |
| `size` | Attachment size in bytes. |
| `is_inline` | Whether the MIME part was marked inline. |
| `content_id` | Optional MIME `Content-ID`, often used by inline images. |
| `sha256` | Optional lowercase SHA-256 digest of the attachment bytes. |

Generic JSON does not embed file bytes or download URLs in the webhook body.

## Download flow

Use the attachment descriptor with the message id from the same payload.

1. Read `message.message_id` from the webhook payload.
2. URL-encode the message id for use in the path.
3. Read the attachment descriptor `id`.
4. Request a download URL from MailWebhook with `X-API-Key`.
5. Fetch the returned `url` with the returned `method` and `headers` before `expires_at`.

The path uses the email Message-ID value, not the MailWebhook event id.

## Request a download URL

Call the attachment URL endpoint from your backend:

```bash
curl "https://app.mailwebhook.com/v1/messages/{encoded_message_id}/attachments/{attachment_id}/url" \
  -H "X-API-Key: <project_api_key>"
```

For example, this message id:

```text
<3F8E62B7-2BD6-4EC8-AD74-7B775CD0DE81@gmail.com>
```

becomes this path segment:

```text
%3C3F8E62B7-2BD6-4EC8-AD74-7B775CD0DE81%40gmail.com%3E
```

So the request path is:

```text
/v1/messages/%3C3F8E62B7-2BD6-4EC8-AD74-7B775CD0DE81%40gmail.com%3E/attachments/att-1/url
```

The endpoint is project-scoped. If the message or attachment does not belong to the API key's project, MailWebhook returns a not-found response.

## Download URL response

The response includes the URL to fetch:

```json
{
  "url": "https://app.mailwebhook.com/v1/attachments/...",
  "expires_at": "2026-06-28T12:15:00Z",
  "method": "GET",
  "headers": {}
}
```

Use the response exactly:

- Send the method in `method`.
- Send the URL in `url`.
- Include any headers from `headers`.
- Fetch before `expires_at`.

Download URLs are short-lived. By default, generated app proxy URLs are single use and expire after about 15 minutes. If your integration needs to reuse a generated URL for a short window, request the URL with `multi_use=true`.

You can request a specific TTL with `ttl_seconds`. Accepted values are from 60 to 86400 seconds.

```bash
curl "https://app.mailwebhook.com/v1/messages/{encoded_message_id}/attachments/{attachment_id}/url?multi_use=true&ttl_seconds=300" \
  -H "X-API-Key: <project_api_key>"
```

## Fetch the file

After you receive the URL response, fetch the file:

```bash
curl -L "<url_from_response>" -o "invoice.pdf"
```

The download response sets:

- `Content-Disposition: attachment`
- `Content-Type`, when available from the parsed attachment
- `Content-Length`
- `X-Content-Type-Options: nosniff` for app proxy downloads

Range requests are supported by the app proxy download path.

## Receiver guidance

Keep attachment download work server-side.

- Verify `X-MailWebhook-Signature` before trusting the payload.
- Store `X-Idempotency-Key` before starting attachment processing.
- Store the attachment `id`, `filename`, `content_type`, `size`, and `sha256` with your work item.
- Request download URLs only when your worker is ready to fetch the file.
- Keep project API keys out of browser code and client-side logs.
- Treat returned download URLs as temporary credentials.

If your receiver processes attachments asynchronously, return `2xx` only after the work item and attachment metadata are durably accepted.

## Custom JSON example

When using `map.custom_json`, include the fields your downstream system needs to request attachment URLs later.

```json
{
  "output": {
    "message_id": { "var": "message.message_id" },
    "subject": { "var": "message.subject" },
    "attachments": { "var": "message.attachments" }
  }
}
```

For a smaller payload, map only the fields needed by the next system:

```json
{
  "output": {
    "message_id": { "var": "message.message_id" },
    "first_attachment": {
      "id": { "var": "message.attachments[0].id" },
      "filename": { "var": "message.attachments[0].filename" },
      "content_type": { "var": "message.attachments[0].content_type" },
      "size": { "var": "message.attachments[0].size" },
      "sha256": { "var": "message.attachments[0].sha256" }
    }
  }
}
```

Use the public attachment download API with `message_id` and attachment `id`. Treat storage fields such as `blob_key` as internal implementation details.

## Troubleshooting

- `body.attachments` is empty: the message had no extracted attachments, or the route pipeline removed them before mapping.
- The download URL request returns `messages.not_found`: check that you URL-encoded the email `message_id` from the payload and used the correct project API key.
- The download URL request returns `attachments.not_found`: check that the attachment `id` came from the same message payload.
- The download fails after the URL response: request a fresh URL and fetch it before `expires_at`.
- A second download attempt fails: request with `multi_use=true` when your workflow needs a generated URL to be reused briefly.

## Related docs

- [Webhook payload reference]
- [API keys and attachment downloads]
- [Generic JSON]
- [Custom JSON]
- [Endpoints]
- [Verify signed webhook deliveries]
- [Webhook retries and replay]

[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
