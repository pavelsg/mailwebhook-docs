---
title: API keys and attachment downloads
parent: Delivery
nav_order: 4
description: "Use MailWebhook project API keys with X-API-Key to call attachment download URL, Events API, and replay endpoints from your backend."
permalink: /docs/api/api-keys-and-attachments/
---

# API keys and attachment downloads
{: .no_toc }

MailWebhook API requests use a project API key in the `X-API-Key` header.

Use this page when your backend needs to call MailWebhook APIs, especially the attachment download URL API used after receiving a webhook payload.

{: .note }
Building a production receiver for MailWebhook deliveries? See [Email Webhook API](https://www.mailwebhook.com/email-webhook-api) for product context. For payload-level attachment metadata, see [Fetch email attachments from webhook payloads].

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## API key scope

A MailWebhook project API key authenticates backend API requests for one project.

Send it as:

```http
X-API-Key: <project_api_key>
```

Use project API keys for API routes such as:

- `GET /v1/events`
- `GET /v1/events/{event_id}`
- `GET /v1/events/{event_id}/attempts`
- `POST /v1/events/{event_id}/replay`
- `GET /v1/messages/{message_id}/attachments/{attachment_id}/url`

Keep these separate:

| Secret or header | Used for |
| --- | --- |
| `X-API-Key` | Your backend authenticates to the MailWebhook API. |
| `X-MailWebhook-Signature` | Your backend verifies webhook deliveries from MailWebhook. |
| Endpoint custom headers | MailWebhook authenticates to your destination endpoint. |
| `X-Setup-Token` | Operator bootstrap and project-list endpoints. |

Do not use a project API key to verify webhook signatures. Signed delivery uses the route signing secret identified by the signature `kid`.

## Where API keys come from

When a project is created through onboarding, MailWebhook creates a default API key if the project does not already have one.

The API key secret is shown only at creation time. Store it in a password manager or secret manager before leaving the page.

The app Settings view shows the project id and explains whether a one-time API key secret is currently available. If the secret is no longer shown and you did not save it, issue a replacement key through your MailWebhook project administration path and disable the old key when rotation is complete.

## Basic API request

Call MailWebhook API endpoints from your backend:

```bash
curl "https://app.mailwebhook.com/v1/events?limit=10" \
  -H "X-API-Key: <project_api_key>"
```

If the key is missing, MailWebhook returns `401` with error code `auth.missing_api_key`.

If the key is invalid or disabled, MailWebhook returns `401` with error code `auth.invalid_api_key`.

## Attachment download URL endpoint

Webhook payloads include attachment metadata, not file bytes. To fetch file content, your backend first asks MailWebhook for a short-lived download URL.

```bash
curl "https://app.mailwebhook.com/v1/messages/{encoded_message_id}/attachments/{attachment_id}/url" \
  -H "X-API-Key: <project_api_key>"
```

Path values:

| Path value | Source |
| --- | --- |
| `{encoded_message_id}` | URL-encoded `message.message_id` from the webhook payload. |
| `{attachment_id}` | Attachment descriptor `id` from the same payload. |

The endpoint is project-scoped. The API key must belong to the same project as the message and attachment.

## URL response

A successful response has this shape:

```json
{
  "url": "https://app.mailwebhook.com/v1/attachments/...",
  "expires_at": "2026-06-28T12:15:00Z",
  "method": "GET",
  "headers": {}
}
```

Use the returned values exactly:

- Fetch `url`.
- Use `method`.
- Send any returned `headers`.
- Fetch before `expires_at`.

Depending on the storage backend, the returned URL can point to the MailWebhook app proxy or to a storage-issued temporary URL. Treat either URL as a temporary credential.

## URL lifetime and reuse

The attachment URL endpoint accepts these query parameters:

| Query parameter | Meaning |
| --- | --- |
| `multi_use` | `false` by default. Set `true` when the generated URL must be reused briefly. |
| `ttl_seconds` | Optional lifetime from 60 to 86400 seconds. |

Default lifetimes:

- Single-use generated URLs expire after about 15 minutes.
- Multi-use generated URLs expire after about 5 minutes when `ttl_seconds` is omitted.

Example:

```bash
curl "https://app.mailwebhook.com/v1/messages/{encoded_message_id}/attachments/{attachment_id}/url?multi_use=true&ttl_seconds=300" \
  -H "X-API-Key: <project_api_key>"
```

For app proxy downloads, a default single-use URL can only be used once. A second use returns `410` with error code `attachments.reused`.

## Download the file

After receiving the URL response:

```bash
curl -L "<url_from_response>" -o "attachment.bin"
```

The download response can include:

- `Content-Disposition: attachment`
- `Content-Type`
- `Content-Length`
- `X-Content-Type-Options: nosniff` for app proxy downloads

The app proxy download path supports byte range requests.

## Security checklist

- Keep project API keys server-side.
- Store API keys in a password manager, secret manager, or deployment secret.
- Do not put project API keys in browser JavaScript, mobile apps, public repositories, or frontend logs.
- Rotate keys by issuing a replacement key, updating backend secrets, verifying traffic, and disabling the old key.
- Treat attachment download URLs as temporary credentials.
- Request attachment download URLs only when your worker is ready to fetch the file.
- Log attachment ids and message ids instead of full download URLs.
- Keep the webhook signing secret separate from the project API key.

## Common errors

| Error code | Meaning | Fix |
| --- | --- | --- |
| `auth.missing_api_key` | The API request did not include `X-API-Key`. | Add the header from your backend. |
| `auth.invalid_api_key` | The API key is wrong or disabled. | Check the saved key or rotate to a new key. |
| `messages.not_found` | The message id was not found in the API key's project. | URL-encode the payload `message.message_id` and use the correct project key. |
| `attachments.not_found` | The attachment id was not found for that message. | Use the descriptor `id` from the same payload. |
| `attachments.expired` | The generated download URL expired. | Request a fresh URL. |
| `attachments.reused` | A single-use app proxy URL was already used. | Request a fresh URL or use `multi_use=true` for brief reuse. |
| `attachments.range_invalid` | The download request sent an invalid `Range` header. | Fix or remove the `Range` header. |
| `attachments.range_not_satisfiable` | The requested byte range is outside the attachment size. | Request a valid range. |

## Related docs

- [Fetch email attachments from webhook payloads]
- [Webhook payload reference]
- [Endpoints]
- [Verify signed webhook deliveries]
- [Webhook retries and replay]

[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
