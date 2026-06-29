---
title: Retries and replay
parent: Delivery
nav_order: 2
description: "Understand MailWebhook webhook retries, replay behavior, delivery attempt history, retryable responses, terminal failures, and idempotent receivers."
permalink: /docs/delivery/retries-and-replay/
---

# Webhook retries and replay
{: .no_toc }

MailWebhook records each webhook delivery attempt and retries transient failures.

Use this page when you need to understand when MailWebhook retries a delivery, when an event becomes terminal, how to replay an event, and how your receiver should handle duplicate attempts.

{: .note }
Building a production receiver for MailWebhook deliveries? See [Email Webhook API](https://www.mailwebhook.com/email-webhook-api) for product context. For the HTTP request contract, see [Endpoints].

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Delivery outcome summary

MailWebhook sends a route's pipeline output to the route endpoint with `POST` and `Content-Type: application/json`.

Each attempt is classified from the endpoint response:

| Result | Event status | What MailWebhook does |
| --- | --- | --- |
| Endpoint returns `2xx` | `delivered` | Records a successful attempt and clears retry state. |
| Network error | `retry` when attempts remain | Schedules another attempt. |
| Endpoint returns `5xx` | `retry` when attempts remain | Schedules another attempt. |
| Endpoint returns `408`, `409`, `425`, or `429` | `retry` when attempts remain | Treats the response as transient and schedules another attempt. |
| Endpoint returns another non-`2xx` status | `dead` | Records a terminal failure. |
| Retryable failures exhaust the attempt budget | `dead` | Stops automatic retries. |

Delivery attempts appear in **Events** under **Delivery Attempts History**. Each attempt can include the request, response status, response body preview, latency, and network error details when available.

## Retry schedule

MailWebhook makes up to six total delivery attempts for retryable failures.

| Attempt | When it happens |
| --- | --- |
| 1 | Initial delivery attempt. |
| 2 | Immediately after the first retryable failure. |
| 3 | About 30 seconds after the previous retryable failure. |
| 4 | About 2 minutes after the previous retryable failure. |
| 5 | About 10 minutes after the previous retryable failure. |
| 6 | About 30 minutes after the previous retryable failure. |

If the sixth attempt also fails with a retryable result, the event becomes `dead`.

Retry timing is best treated as approximate. Queue load, worker scheduling, and endpoint timing can shift the exact delivery time.

## Receiver response guidance

Choose the response status based on whether your receiver accepted the event.

- Return `2xx` after your receiver has durably accepted the request.
- Return `400`, `401`, `403`, `404`, `410`, or `422` when the request should stop retrying.
- Return `408`, `409`, `425`, `429`, or `5xx` when the failure is temporary and another attempt should happen.

For asynchronous receivers, a common pattern is:

1. Verify `X-MailWebhook-Signature`.
2. Store `X-Idempotency-Key` with the work item.
3. Save the payload or enqueue durable work.
4. Return `204`.

This keeps MailWebhook retry behavior aligned with your receiver's real processing state.

## Idempotency across retries and replay

Retries can send the same logical email event more than once. Design your receiver around `X-Idempotency-Key`.

The MailWebhook-generated `X-Idempotency-Key` is deterministic for the original email message id and route id. For the same message and route, retries and replay use the same key.

Use that key to make downstream work idempotent:

- Store the key before performing side effects.
- Treat repeated keys as already accepted or already processed.
- Return `2xx` for duplicate keys when the original work was accepted.
- Avoid using `X-MailWebhook-Signature` as a deduplication key because the signature includes a timestamp and can change between attempts.

## Replay behavior

Replay queues a new delivery attempt for an existing event.

Replay is useful after you fix a receiver outage, update endpoint credentials, correct a route pipeline, or want to resend a stored message through the event's route.

Replay uses:

- The stored message for the event.
- The event's current route.
- The route's current pipeline configuration.
- The route's current endpoint URL and endpoint headers.
- The route signing secret identified by the route's key id, or `kid`.

MailWebhook builds a fresh HTTP request for replay. It does not reuse old captured request bytes. The request body is rebuilt from the stored message and current pipeline, and `X-MailWebhook-Signature` is generated again for the new raw body bytes.

Replay ignores any existing `next_attempt_at` delay. If the replay attempt fails with a retryable result, MailWebhook can schedule the next retry through the normal retry policy.

## Replay from the UI

Use the UI path for most replay work.

1. Open **Events**.
2. Select the event you want to inspect.
3. Review **Event Details** and **Delivery Attempts History**.
4. Fix the receiver, endpoint, or route issue.
5. Click **Replay Event**.

The UI queues a `replay_event` job and shows **Replay queued**.

## Replay from the API

You can also queue replay with the Events API:

```bash
curl -X POST "https://app.mailwebhook.com/v1/events/{event_id}/replay" \
  -H "X-API-Key: <project_api_key>"
```

A successful response has this shape:

```json
{
  "queued": true,
  "event_id": "00000000-0000-0000-0000-000000000000"
}
```

The replay endpoint validates that the event belongs to the authenticated project before queueing work.

## Inspect attempts from the API

Use these endpoints when you need delivery state outside the UI:

```bash
curl "https://app.mailwebhook.com/v1/events/{event_id}" \
  -H "X-API-Key: <project_api_key>"
```

```bash
curl "https://app.mailwebhook.com/v1/events/{event_id}/attempts" \
  -H "X-API-Key: <project_api_key>"
```

The event detail includes status, attempt count, last error, message summary, and route snapshot. The attempts endpoint returns attempt rows with delivery status, HTTP status, latency, and captured request or response previews when available.

## Common issues

- A receiver performs a side effect and then times out. MailWebhook retries because it did not receive a `2xx`; use `X-Idempotency-Key` to avoid duplicate side effects.
- A temporary authentication dependency returns `401` or `403`. MailWebhook treats these as terminal failures; return a retryable status only when another attempt can help.
- A replay after a route or pipeline edit sends a different body than the original failed attempt. Replay uses current route configuration.
- A receiver returns `2xx` before durable acceptance. MailWebhook considers the event delivered and will not retry automatically.
- An endpoint failure is fixed after the event becomes `dead`. Queue **Replay Event** to create another attempt.

## Related docs

- [Endpoints]
- [API keys and attachment downloads]
- [Verify signed webhook deliveries]
- [Webhook payload reference]
- [Send a test email and inspect the payload]
- [Receive your first inbound email webhook]

[Endpoints]: {% link docs/endpoints.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
[Verify signed webhook deliveries]: {% link docs/delivery/signatures.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
