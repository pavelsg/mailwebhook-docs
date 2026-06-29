---
title: Endpoints
parent: Delivery
nav_order: 3
description: "Configure MailWebhook HTTP endpoints for signed JSON POST delivery, static headers, idempotency keys, and exact route pipeline output."
---

# Endpoints
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Endpoints represent HTTP delivery targets that routes point to. They are project-scoped records with a URL, optional static headers (for API keys, auth tokens, etc.), and a timeout hint.

{: .note }
Building a receiving endpoint for MailWebhook deliveries? See [Email Webhook API](https://www.mailwebhook.com/email-webhook-api) for product context and implementation positioning.

## HTTP request contract

- Request method/body:
  - `POST` with `Content-Type: application/json`.
  - Body is exactly the pipeline output bytes (e.g., generic mapper JSON, Slack-simple, custom JSON). No additional wrapping occurs.
- Headers are deterministic:
  - `User-Agent: mailwebhook/...` from the HTTP client defaults.
  - `X-Idempotency-Key`: SHA-256 of `{message_id}|{route_id}`.
  - `X-MailWebhook-Signature`: `t=<unix>, kid=<kid>, v1=<base64(hmac_sha256(t+"."+body, secret))>`.
  - `Idempotency-Key` is also included if the customer headers lack either variant.
- Custom headers:
  - Custom headers defined on the endpoint are included. This is useful for API keys, auth tokens, etc.

### Basic authentication example

To create an endpoint with basic authentication, you can include the `Authorization` header with a base64-encoded username and password. The general format is: `Authorization: Basic <base64(user:pass)>`. To construct base64 do the following in your terminal:

```bash
$ echo -n "user:pass" | base64
dXNlcjpwYXNz
```

So the final header will look like this:

```plaintext
Authorization: Basic dXNlcjpwYXNz
```

When setting up your endpoint, include this header in the Custom headers section.

## Fetching attachments

Webhook payloads include attachment metadata. File bytes stay in MailWebhook storage and are fetched through the attachment download API.

For the full descriptor shape, URL request, `X-API-Key` usage, expiration behavior, and receiver guidance, see [Fetch email attachments from webhook payloads] and [API keys and attachment downloads].

## Endpoint limitations

- Allowed schemes: `http` or `https`.
- Userinfo (`user:pass@`) is rejected.
- Hostname must resolve; if it resolves to a literal IP, we check it directly.
- Every resolved address must be public: private, loopback, link-local, reserved, or multicast ranges are blocked.

[Fetch email attachments from webhook payloads]: {% link docs/payloads/attachments.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
