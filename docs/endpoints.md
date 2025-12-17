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

## Endpoint limitations

- Allowed schemes: `http` or `https`.
- Userinfo (`user:pass@`) is rejected.
- Hostname must resolve; if it resolves to a literal IP, we check it directly.
- Every resolved address must be public: private, loopback, link-local, reserved, or multicast ranges are blocked.

