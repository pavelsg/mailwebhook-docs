---
title: Signed delivery
parent: Delivery
nav_order: 1
description: "Verify MailWebhook signed webhook deliveries with the X-MailWebhook-Signature header, raw request body, timestamp, key id, and HMAC-SHA256 digest."
permalink: /docs/delivery/signatures/
---

# Verify signed webhook deliveries
{: .no_toc }

MailWebhook signs every webhook delivery request with `X-MailWebhook-Signature`.

Use this page when your receiving endpoint needs to prove that a request came from MailWebhook and that the request body was not changed before it reached your application.

{: .note }
Building a receiving service for MailWebhook requests? See [Email Webhook API](https://www.mailwebhook.com/email-webhook-api) for product context. For endpoint setup, static headers, and URL restrictions, see [Endpoints].

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Header format

MailWebhook adds this header to delivery requests:

```text
X-MailWebhook-Signature: t=<unix>, kid=<kid>, v1=<base64_hmac_sha256>
```

| Part | Meaning |
| --- | --- |
| `t` | Unix timestamp in seconds. |
| `kid` | Key identifier for the signing secret used by the route. |
| `v1` | Base64-encoded HMAC-SHA256 digest. |

The signature header is generated for initial delivery attempts, retries, and event replays.

## What is signed

Verify the signature against the exact raw HTTP request body bytes.

The HMAC input is:

```text
<t>.<raw_request_body_bytes>
```

In code, that means:

```text
signed_input = ascii(timestamp) + "." + raw_request_body_bytes
expected = hmac_sha256(signing_secret, signed_input)
```

Do not verify against parsed JSON, pretty-printed JSON, decoded text, a request object, or a reserialized copy of the body. JSON parsing can change whitespace and field ordering, which changes the bytes that were signed.

## Verification steps

1. Read the raw request body bytes before your framework parses or mutates the body.
2. Parse `X-MailWebhook-Signature` into `t`, `kid`, and `v1`.
3. Select the signing secret for the route by `kid`.
4. Build the signed input as `f"{t}.".encode("ascii") + body_bytes`.
5. Compute `HMAC-SHA256` with the selected signing secret.
6. Base64-decode `v1`.
7. Compare the expected digest and provided digest with a constant-time comparison.
8. Reject timestamps outside your receiver's accepted replay window.

Use a replay window that matches your system's tolerance for delayed delivery attempts. A common starting point is 300 seconds.

## Verify the shared fixture in the browser

Use the [HMAC Signature Verifier](https://tools.mailwebhook.com/tools/hmac-signature-verifier) when you want to check the signature contract before writing receiver code. Choose **MailWebhook signature** mode. That mode signs `timestamp + "." + raw body` and compares the result with the base64 `v1` value from `X-MailWebhook-Signature`.

Click **Load MailWebhook example**. The tool loads the same synthetic fixture that this docs site keeps under [`/assets/examples/signatures/`](/assets/examples/signatures/manifest.json).

Expected result with the fixed sample clock:

| Field | Expected value |
| --- | --- |
| Signature result | `Verified` |
| Timestamp status | `Fresh` |
| Delivery decision | `Accepted delivery` |
| `kid` | `mw_test_kid_v1` |
| Signature timestamp | `1700000000` (`2023-11-14T22:13:20Z`) |
| Verification time | `1700000120` (`2023-11-14T22:15:20Z`) |
| Tolerance | `300` seconds |

Then click **Use current time**. The same historical sample should remain cryptographically `Verified`, but its timestamp status should change to `Stale`, and the delivery decision should say it is not a current accepted delivery.

Append one space to the raw body and verify again. The result should change to `Signature mismatch`, because the exact body bytes changed.

Fixture provenance:

- Fixture version: `mailwebhook-signature-v1`
- Source commit: `3e11346ba25e5181cfb4f103b0a8ca558fa580ca` in `mailwebhookhq/examples`
- Body file: [`mailwebhook-body-v1.json`](/assets/examples/signatures/mailwebhook-body-v1.json)
- Fixture manifest: [`mailwebhook-signature-v1.json`](/assets/examples/signatures/mailwebhook-signature-v1.json)
- Provenance manifest: [`manifest.json`](/assets/examples/signatures/manifest.json)
- Body SHA-256: `db71198e31397a66e7cead6e77a841ea1f98c25f5ea58fbbb423de6fe5e11445`

The fixture secret is synthetic and test-only. In production, use the route signing secret selected by `kid`. That secret is separate from MailWebhook API keys, endpoint custom headers, and mailbox provider credentials.

## Python verifier

This example expects `body` to be the raw request body bytes and `secrets_by_kid` to map key ids to secret bytes.

```py
import base64
import binascii
import hashlib
import hmac
import time


def verify_mailwebhook_signature(
    header: str,
    body: bytes,
    secrets_by_kid: dict[str, bytes],
    tolerance_seconds: int = 300,
) -> bool:
    parts: dict[str, str] = {}
    for item in header.split(","):
        if "=" not in item:
            continue
        key, value = item.strip().split("=", 1)
        parts[key] = value

    try:
        timestamp = int(parts["t"])
        kid = parts["kid"]
        signature_b64 = parts["v1"]
    except (KeyError, ValueError):
        return False

    if abs(int(time.time()) - timestamp) > tolerance_seconds:
        return False

    secret = secrets_by_kid.get(kid)
    if secret is None:
        return False

    signed_input = f"{timestamp}.".encode("ascii") + body
    expected = hmac.new(secret, signed_input, hashlib.sha256).digest()

    try:
        provided = base64.b64decode(signature_b64, validate=True)
    except (binascii.Error, ValueError):
        return False

    return hmac.compare_digest(expected, provided)
```

## Node.js verifier

This example expects `bodyBuffer` to be the raw request body buffer and `secretsByKid` to map key ids to signing secrets.

```js
import crypto from "node:crypto";

export function verifyMailWebhookSignature(
  header,
  bodyBuffer,
  secretsByKid,
  toleranceSeconds = 300,
) {
  const parts = Object.fromEntries(
    header
      .split(",")
      .map((part) => part.trim())
      .filter((part) => part.includes("="))
      .map((part) => {
        const index = part.indexOf("=");
        return [part.slice(0, index), part.slice(index + 1)];
      }),
  );

  if (!/^\d+$/.test(parts.t ?? "")) {
    return false;
  }

  const timestamp = Number.parseInt(parts.t, 10);
  const kid = parts.kid;
  const signatureB64 = parts.v1;

  if (!Number.isFinite(timestamp) || !kid || !signatureB64) {
    return false;
  }

  const nowSeconds = Math.floor(Date.now() / 1000);
  if (Math.abs(nowSeconds - timestamp) > toleranceSeconds) {
    return false;
  }

  const secret = secretsByKid[kid];
  if (!secret) {
    return false;
  }

  const signedInput = Buffer.concat([
    Buffer.from(`${timestamp}.`, "ascii"),
    bodyBuffer,
  ]);
  const expected = crypto
    .createHmac("sha256", secret)
    .update(signedInput)
    .digest();

  let provided;
  try {
    provided = Buffer.from(signatureB64, "base64");
  } catch {
    return false;
  }

  return (
    expected.length === provided.length &&
    crypto.timingSafeEqual(expected, provided)
  );
}
```

In Express, use a raw body parser for the webhook route so `req.body` remains the exact bytes MailWebhook signed:

```js
app.post(
  "/mailwebhook",
  express.raw({ type: "application/json" }),
  (req, res) => {
    const ok = verifyMailWebhookSignature(
      req.get("X-MailWebhook-Signature") || "",
      req.body,
      secretsByKid,
    );

    if (!ok) {
      res.sendStatus(401);
      return;
    }

    const payload = JSON.parse(req.body.toString("utf8"));
    processPayload(payload);
    res.sendStatus(204);
  },
);
```

## Next step: build your receiver

After you choose a verifier pattern, use [Build your application receiver](https://blog.mailwebhook.com/blog/inbound-email-webhook-nodejs-python-go/) for complete Node.js, Python, and Go receiver examples. The guide shows how to capture the raw body, verify the signature, process the payload, and return a successful webhook response.

## Common verification mistakes

- Using parsed JSON instead of raw request body bytes.
- Comparing a hex digest to `v1`, which is base64 encoded.
- Including the HTTP method, URL, headers, or response body in the HMAC input.
- Using a MailWebhook API key instead of the route signing secret.
- Ignoring `kid` when more than one signing secret exists.
- Letting a web framework read the body before signature verification.
- Accepting very old timestamps without a receiver-side replay window.

## Inspect a signed request

Use **Webhook Preview** when you are testing a route from onboarding. The **Request** tab shows the outgoing request headers and body for the captured delivery.

Use **Events** and **Delivery Attempts History** when you need to inspect delivery status, response status, response body preview, or replay behavior.

## Related docs

- [Endpoints]
- [Webhook retries and replay]
- [Webhook payload reference]
- [Send a test email and inspect the payload]
- [HMAC Signature Verifier](https://tools.mailwebhook.com/tools/hmac-signature-verifier)
- [Receive your first inbound email webhook]

[Endpoints]: {% link docs/endpoints.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
