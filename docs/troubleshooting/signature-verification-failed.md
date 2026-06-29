---
title: Signature verification failed
parent: Troubleshooting
nav_order: 2
description: "Troubleshoot MailWebhook signature verification failures by checking X-MailWebhook-Signature, raw request body bytes, timestamp tolerance, kid lookup, base64 HMAC, and receiver parsing."
permalink: /docs/troubleshooting/signature-verification-failed/
---

# Troubleshoot signature verification failures
{: .no_toc }

Use this guide when your endpoint receives a MailWebhook delivery request but rejects `X-MailWebhook-Signature`.

Most signature failures come from verifying the wrong bytes, using the wrong secret, parsing the header incorrectly, or rejecting a valid timestamp too aggressively.

{: .note }
Building or debugging a production receiver? See [Email Webhook API](https://www.mailwebhook.com/email-webhook-api) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Verify the signature against the exact raw HTTP request body bytes that MailWebhook sent.

MailWebhook signs this input:

```text
<t>.<raw_request_body_bytes>
```

The signature header has this format:

```text
X-MailWebhook-Signature: t=<unix>, kid=<kid>, v1=<base64_hmac_sha256>
```

Use `kid` to select the route signing secret, compute HMAC-SHA256 over the signed input, base64-decode `v1`, and compare the raw digest bytes with a constant-time comparison.

## Symptoms

Common symptoms include:

- Your receiver returns `401`, `403`, or another non-`2xx` status for real MailWebhook requests.
- **Delivery Attempts History** shows request headers with `X-MailWebhook-Signature`, but your verifier returns false.
- The same code works with a local test string but fails for captured MailWebhook requests.
- Replay keeps failing with the same signature error.
- Signature checks fail only after middleware, JSON parsing, or request logging was added.
- Signature checks fail only for larger payloads or payloads with attachments.

## Most common causes

### The receiver verifies parsed JSON instead of raw bytes

The HMAC is computed over the raw request body bytes.

Signature verification fails when the receiver uses:

- A parsed JSON object.
- Pretty-printed JSON.
- A string decoded and re-encoded by the framework.
- A body object that middleware already changed.
- A payload copied from logs or a formatted UI view.

Read the raw body before any JSON parser, body parser, validator, or request logging middleware mutates it.

### The receiver uses the wrong secret

Signed delivery uses the route signing secret identified by the header `kid`.

Do not use:

- A MailWebhook project API key.
- An endpoint custom header value.
- An old signing secret after rotation.
- A secret for another project or route.

When multiple signing secrets exist, use `kid` as the lookup key.

### The header parser drops or changes fields

The signature header is a comma-separated list of key-value parts:

```text
t=<unix>, kid=<kid>, v1=<base64_hmac_sha256>
```

The parser must preserve:

- `t`
- `kid`
- `v1`

Trim spaces around each part. Do not trim or split the base64 value beyond the first `=` after `v1`. Base64 values can include padding.

### The digest encoding is wrong

`v1` is base64-encoded raw HMAC-SHA256 bytes.

Common mistakes:

- Comparing a hex digest string to `v1`.
- Base64-encoding a hex digest instead of the raw digest bytes.
- Comparing strings with different encodings.
- Using SHA-1, SHA-512, or plain SHA-256 instead of HMAC-SHA256.

Compute raw HMAC-SHA256 bytes, base64-decode `v1`, then compare bytes.

### The timestamp window is too strict

The `t` value is a Unix timestamp in seconds.

Your receiver should reject timestamps outside its accepted replay window, but the window must allow normal clock drift and delivery delay. A common starting point is `300` seconds.

If every request fails the timestamp check, compare server clocks and confirm your code treats `t` as seconds, not milliseconds.

### The replay request has a different signature than the original attempt

Replay builds a fresh HTTP request from the stored message and current route configuration. MailWebhook signs the new raw body bytes with a new timestamp.

The `X-MailWebhook-Signature` value can change between initial delivery, retry, and replay. Use `X-Idempotency-Key` for deduplication, not the signature header.

## Step-by-step checks

### 1. Inspect the captured request

Open **Events**, select the event, and open **Delivery Attempts History**.

Check the request capture:

- Method is `POST`.
- `Content-Type` is `application/json`.
- `X-MailWebhook-Signature` is present.
- The body preview is the route pipeline output.
- The response status is the status your receiver returned.

If the attempt has no request capture, use **Webhook Preview** during onboarding or replay the event after enabling receiver logging.

### 2. Parse the signature header

Confirm your parser extracts exactly:

| Part | Expected value |
| --- | --- |
| `t` | Unix timestamp in seconds. |
| `kid` | Route signing key id. |
| `v1` | Base64-encoded HMAC-SHA256 bytes. |

Reject the request if any part is missing.

### 3. Select the secret by `kid`

Look up the route signing secret for the exact `kid` from the header.

Check:

- The `kid` from the header exists in your receiver's secret map.
- The stored secret value is the signing secret, not a project API key.
- The receiver has the current secret after any rotation.
- The route that produced the event uses the same signing key id.

### 4. Build the signed input

Build the bytes in this exact order:

```text
ascii(t) + "." + raw_request_body_bytes
```

For example:

```text
signed_input = f"{t}.".encode("ascii") + body_bytes
```

Do not include the HTTP method, URL, headers, query string, response body, or a trailing newline.

### 5. Compare raw digest bytes

Compute HMAC-SHA256 with the selected secret.

Then:

1. Base64-decode `v1`.
2. Compare the computed digest bytes with the decoded `v1` bytes.
3. Use a constant-time comparison.

If the byte lengths differ, the verification should fail.

### 6. Check framework body handling

For Node.js and Express, use a raw body parser for the webhook route:

```js
app.post("/mailwebhook", express.raw({ type: "application/json" }), handler);
```

For Python frameworks, read the raw request body bytes before JSON parsing. The exact API depends on the framework, but the verifier must receive the original byte stream.

Avoid middleware that reads, parses, decompresses, formats, or logs the request body before verification unless it preserves the exact bytes for the verifier.

### 7. Return the right status

When verification fails, return a terminal client error such as `401` or `403`.

MailWebhook treats most non-`2xx` statuses outside the retryable set as terminal. If your receiver returns `5xx` for signature failures, MailWebhook can retry a request that will keep failing until the attempt budget is exhausted.

## Minimal verifier checklist

Use this checklist against your verifier:

- Reads raw body bytes before parsing.
- Parses `t`, `kid`, and `v1`.
- Treats `t` as Unix seconds.
- Looks up the signing secret by `kid`.
- Builds `f"{t}.".encode("ascii") + body_bytes`.
- Computes HMAC-SHA256 with the signing secret.
- Base64-decodes `v1`.
- Compares raw digest bytes in constant time.
- Rejects timestamps outside a reasonable replay window.
- Keeps project API keys separate from signing secrets.

## Expected result

After the verifier is fixed:

- Valid MailWebhook requests pass signature verification.
- Invalid signatures return `401` or `403`.
- The receiver stores or queues the payload only after verification passes.
- The receiver returns `2xx` after durable acceptance.
- MailWebhook marks the next retry or replay attempt `delivered`.

## Related docs

- [Verify signed webhook deliveries]
- [Webhook delivery failed]
- [Endpoints]
- [Webhook retries and replay]
- [Webhook payload reference]
- [API keys and attachment downloads]
- [Send a test email and inspect the payload]

[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook delivery failed]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
