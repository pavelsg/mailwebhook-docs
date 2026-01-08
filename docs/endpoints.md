---
title: Endpoints
nav_order: 3
---

# Endpoints
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Endpoints represent HTTP delivery targets that routes point to. They are project-scoped records with a URL, optional static headers (for API keys, auth tokens, etc.), and a timeout hint.

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

To create an endpoint with basic authentication, you can include the `Authorization` header with a base64-encoded username and password. The geral format is: `Authorization: Basic <base64(user:pass)>`. To construct base64 do the following in your terminal:

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



When the pipeline output includes attachments your backend can fetch them via pre-signed URLs. 

Attachmnts are listed under `body.attachments` array in the generic JSON mapper output:

```json
    "attachments": [
      {
        "id": "9f5a1ded-538d-4f5f-a7a9-d3eacf9e58a0",
        "filename": "5430524288.pdf",
        "content_type": "application/pdf",
        "size": 93259,
        "is_inline": false,
        "sha256": "059a0f5260487bbe663994de1fd641401fec76ac9f6bddfe5b53ae60d4bb2d86"
      }
    ]
```

To request pre-signed URL for an attachment make a `GET` request to: 
```
https://app.mailwebhook.com/v1/messages/{message_id}/attachments/{attachment_id}/url
```

{: .note }
You should URL-encode `message_id`. For example `<3F8E62B7-2BD6-4EC8-AD74-7B775CD0DE81@gmail.com>` becomes `%3C3F8E62B7-2BD6-4EC8-AD74-7B775CD0DE81%40gmail.com%3E`.

Do not forget to include `X-Api-Key` header with your project API key.
The response will include a JSON object with `url`, `expires_at`, `method`, and `headers` fields:

```json
{
  "url": "...",
  "expires_at": "2025-12-15T14:26:19Z",
  "method": "GET",
  "headers": {}
}
```

Send GET request to the provided `url` with optional `headers` before `expires_at` to fetch the attachment.

## Endpoint limitations

- Allowed schemes: `http` or `https`.
- Userinfo (`user:pass@`) is rejected.
- Hostname must resolve; if it resolves to a literal IP, we check it directly.
- Every resolved address must be public: private, loopback, link-local, reserved, or multicast ranges are blocked.

