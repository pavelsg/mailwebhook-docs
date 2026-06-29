---
title: Webhook delivery failed
parent: Troubleshooting
nav_order: 1
description: "Troubleshoot failed MailWebhook deliveries by checking event status, delivery attempts, HTTP response status, network errors, signatures, idempotency, retries, and replay."
permalink: /docs/troubleshooting/webhook-delivery-failed/
---

# Troubleshoot failed webhook deliveries
{: .no_toc }

Use this guide when a MailWebhook event is stuck in `retry`, has moved to `dead`, or shows failed rows in **Delivery Attempts History**.

The fastest path is to inspect the latest delivery attempt, decide whether MailWebhook received an HTTP response, fix the endpoint or route issue, then replay the event when automatic retries are no longer running.

{: .note }
Building or debugging a production receiver? See [Email Webhook API](https://www.mailwebhook.com/email-webhook-api) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Open **Events**, select the failed event, and inspect **Delivery Attempts History**.

Use the latest attempt to answer one question first:

| Latest attempt shows | What it usually means | Next action |
| --- | --- | --- |
| `http_status` is `0` or response has a network error | MailWebhook did not receive an HTTP response. | Check DNS, TLS, firewall, public routing, and endpoint timeout. |
| `5xx`, `408`, `409`, `425`, or `429` | Temporary receiver failure. | Fix the receiver and wait for retry, or replay after the event is `dead`. |
| Another non-`2xx` status, such as `400`, `401`, `403`, `404`, `410`, or `422` | Permanent receiver rejection. | Fix URL, auth, signature verification, payload handling, or route output, then replay. |
| `2xx` | MailWebhook accepted the delivery as successful. | Check your receiver's internal queue, idempotency, and downstream processing. |

MailWebhook considers a delivery successful only when the endpoint returns `2xx`. Other responses either retry or become terminal according to the retry policy.

## Symptoms

Common symptoms include:

- The event status is `retry`.
- The event status is `dead`.
- The attempt status is `failed`.
- The endpoint did not log a request.
- The endpoint logged the request but returned an error.
- The request appears in **Webhook Preview**, but your application rejected it.
- Replay creates another failed delivery attempt.

## Most common causes

### No HTTP response reached MailWebhook

When MailWebhook cannot get an HTTP response, the attempt is recorded with `http_status` `0` and a response network error.

Common causes:

- DNS lookup failed.
- The endpoint host is unreachable from the public internet.
- TLS certificate validation failed.
- A firewall, tunnel, or allowlist blocked the request.
- The receiver timed out before responding.

Fix the network path first. MailWebhook can only classify the endpoint response after it receives one.

### The receiver returned a retryable status

MailWebhook retries network errors, `5xx`, `408`, `409`, `425`, and `429` while attempts remain.

Use retryable statuses when another attempt can help. Examples:

- `503` while the receiver is temporarily offline.
- `429` when the receiver wants MailWebhook to try again later.
- `409` when a temporary lock or queue state should clear soon.

MailWebhook makes up to six total delivery attempts for retryable failures.

### The receiver returned a permanent failure

MailWebhook treats non-`2xx` statuses outside the retryable set as terminal. Examples include `400`, `401`, `403`, `404`, `410`, and `422`.

Common causes:

- Endpoint URL path is wrong.
- Custom auth header is missing or no longer valid.
- Signature verification uses parsed JSON instead of the raw request body bytes.
- Receiver expects a different payload shape than the route pipeline sends.
- Receiver rejects the request before storing or queueing it.

After you fix a permanent failure, replay the event. Automatic retries do not continue for terminal responses.

### The route output is valid JSON, but the receiver expects different fields

MailWebhook sends exactly the route pipeline output as the request body. It does not wrap the payload.

For the default `map.generic_json` pipeline, inspect fields such as:

- `schema.name`
- `schema.version`
- `event.id`
- `event.route_id`
- `message.message_id`
- `message.subject`
- `body.text`
- `body.attachments`
- `meta.source`

If your receiver expects a different shape, update the receiver or change the route pipeline.

### Signature verification rejects a real MailWebhook request

MailWebhook signs delivery requests with `X-MailWebhook-Signature`.

The receiver must verify the exact raw request body bytes against:

```text
<t>.<raw_request_body_bytes>
```

Do not verify against parsed JSON, pretty-printed JSON, decoded text, or a reserialized copy of the body. Those can change the signed bytes.

### The endpoint accepted the delivery but downstream work failed

If MailWebhook shows `delivered`, the endpoint returned `2xx`. MailWebhook will not automatically retry a delivered event.

Check your receiver for:

- Work queued after the `2xx` response.
- Idempotency handling for `X-Idempotency-Key`.
- Internal job failures after the request was accepted.
- Downstream API failures.

Return `2xx` only after the receiver has durably accepted the request.

## Step-by-step checks

### 1. Open the event

Open **Events** and select the event.

Check:

- **Status**
- **Attempts**
- **Last error**
- **Route**
- **Message-Id**
- **Subject**

Event statuses most relevant to delivery troubleshooting are:

| Status | Meaning |
| --- | --- |
| `delivered` | At least one attempt returned `2xx`. |
| `retry` | A retryable failure happened and another attempt is scheduled. |
| `dead` | Delivery is terminal because the failure was permanent or attempts were exhausted. |
| `quarantined` | A hosted-mailbox spam policy held the event before webhook delivery. |

### 2. Inspect the latest delivery attempt

In **Delivery Attempts History**, open the newest attempt first.

Check:

- Attempt status, usually `success` or `failed`.
- HTTP status.
- Latency.
- Response body preview.
- Response network error.
- Whether request or response previews are marked truncated.

If the newest attempt is older than your fix, replay the event after the fix is deployed.

### 3. Check the request capture

Confirm the request capture matches your receiver.

Expected request properties:

- Method is `POST`.
- `Content-Type` is `application/json`.
- `X-MailWebhook-Signature` is present.
- `X-Idempotency-Key` is present.
- The body is the route pipeline output.

If the request body is not what your receiver expects, inspect the route rule and pipeline before changing retry behavior.

### 4. Match the response to the retry policy

Use the response status to decide what happens next.

| Response | MailWebhook behavior |
| --- | --- |
| `2xx` | Marks the event `delivered`. |
| Network error | Schedules retry while attempts remain. |
| `5xx` | Schedules retry while attempts remain. |
| `408`, `409`, `425`, `429` | Schedules retry while attempts remain. |
| Other non-`2xx` | Marks the event `dead`. |
| Retryable failure after all attempts | Marks the event `dead`. |

Use `400`, `401`, `403`, `404`, `410`, or `422` only when another attempt should stop.

### 5. Fix the receiver or endpoint

Fix the first concrete failure in the latest attempt:

- For network errors, make the endpoint publicly reachable with valid DNS and TLS.
- For `401` or `403`, update endpoint custom headers or receiver auth rules.
- For signature failures, verify against raw request body bytes.
- For `404`, confirm the endpoint URL path.
- For `400` or `422`, compare the captured body with the payload schema your receiver expects.
- For timeouts, return `2xx` after durable enqueue and move slow work to a background job.

Endpoint URLs must use `http` or `https`, must not include userinfo, and must resolve to public addresses.

### 6. Replay when needed

Use replay when the event is `dead` or when you fixed a receiver issue after retries finished.

In the UI:

1. Open **Events**.
2. Select the event.
3. Review the failed attempt.
4. Fix the endpoint, receiver, route, or pipeline issue.
5. Click **Replay Event**.

Replay builds a fresh request from the stored message and current route configuration. It regenerates `X-MailWebhook-Signature` for the new request body and keeps the same logical idempotency key for the same message and route.

## Inspect attempts with the API

Use the API when you need the same data outside the UI.

Fetch the event:

```bash
curl "https://app.mailwebhook.com/v1/events/{event_id}" \
  -H "X-API-Key: <project_api_key>"
```

Fetch delivery attempts:

```bash
curl "https://app.mailwebhook.com/v1/events/{event_id}/attempts" \
  -H "X-API-Key: <project_api_key>"
```

Attempt rows include:

- `status`
- `http_status`
- `latency_ms`
- `request_capture`
- `response_capture.status`
- `response_capture.body`
- `response_capture.network_error`

## Expected result

After the issue is fixed, the next retry or replay should produce:

- Attempt status `success`.
- HTTP status in the `2xx` range.
- Event status `delivered`.
- No new `next_attempt_at`.
- Receiver logs showing the same request accepted.

For receivers that enqueue work, the receiver should store `X-Idempotency-Key` before side effects and return `2xx` only after the event is durably accepted.

## Related docs

- [Endpoints]
- [Webhook retries and replay]
- [Verify signed webhook deliveries]
- [Webhook payload reference]
- [Send a test email and inspect the payload]
- [Receive your first inbound email webhook]
- [API keys and attachment downloads]

[Endpoints]: {% link docs/endpoints.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
