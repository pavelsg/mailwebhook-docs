---
title: Route rule did not match
parent: Troubleshooting
nav_order: 3
description: "Troubleshoot MailWebhook route rules that do not match by checking enabled routes, mailbox preview, exact address filters, header matching, regexes, attachment MIME patterns, and boolean logic."
permalink: /docs/troubleshooting/route-rule-did-not-match/
---

# Troubleshoot route rules that do not match
{: .no_toc }

Use this guide when MailWebhook receives an email, but the expected route does not create an event or webhook delivery.

The fastest path is to confirm the route is enabled, preview the message against the route, then compare the route rule with the normalized message fields that the matcher actually evaluates.

{: .note }
Routing email into webhooks? See [Email to webhook](https://www.mailwebhook.com/email-to-webhook) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

MailWebhook creates events only for enabled routes whose rule matches the normalized inbound email.

Route rules use top-level AND semantics. If a rule contains `to_emails`, `from_domains`, and `subject_contains`, the message must satisfy all three. Empty rule fields add no constraint, and an empty rule matches every ingested message.

Use **Mailbox Preview** to test a stored message against a route. If the preview says **Selected message does not match this route**, the route predicate returned false. Fix the rule, then preview the same message again or send a new test email.

## Symptoms

Common symptoms include:

- The mailbox receives the email, but no event appears for the expected route.
- **Webhook Preview** stays empty after a test email.
- **Mailbox Preview** says **Selected message does not match this route**.
- A route works for one sender or mailbox, then fails for another.
- A header, subject, or attachment rule never matches.
- A rule with several fields works only after some fields are removed.
- A replay is unavailable because no event was created for the message and route.

## Most common causes

### The route is disabled

The ingestion worker loads enabled routes only. A disabled route is skipped before rule matching.

Open **Routes** and confirm the route is enabled. If the route was created through onboarding, confirm the saved route still exists and is enabled after any setup changes.

### The message reached a different mailbox

Rules evaluate the parsed message that belongs to the project mailbox that ingested the email.

Check the mailbox that actually received the message. For hosted mailboxes, confirm the alias is enabled and the email was sent to the exact hosted address. For Gmail, Microsoft 365, and IMAP, confirm the provider connection is active and the message is visible in the mailbox source MailWebhook polls or watches.

### The rule is narrower than expected

Each populated top-level matcher must pass.

This rule requires the recipient, sender domain, and subject to match:

```json
{
  "to_emails": ["alerts@example.com"],
  "from_domains": ["vendor.com"],
  "subject_contains": ["invoice"]
}
```

If the sender is `billing.vendor.com`, the `from_domains` matcher fails because it checks exact domains. Use the exact domain from the parsed `From` address, or use `from_contains` when a substring rule is appropriate.

### Exact address rules use normalized email addresses

`to_emails` and `from_emails` perform exact matching against normalized addresses.

Check for:

- Alias addresses.
- Plus addressing.
- Distribution lists.
- Forwarding addresses.
- A display name copied into the rule instead of the email address.
- The message being addressed through `Cc` while the rule checks `to_emails`.

Use `to_contains` or `from_contains` when the route should match a family of addresses.

### Header equality is too strict

Header names are matched case-insensitively. Header values for `headers_equals` must equal the normalized header value exactly after string conversion and trimming.

Use `headers_contains` when only part of a header value matters.

For example, this requires an exact value:

```json
{
  "headers_equals": {
    "x-priority": "high"
  }
}
```

This accepts a longer value that contains the substring:

```json
{
  "headers_contains": {
    "x-priority": "high"
  }
}
```

### The regex does not match the normalized subject

`subject_regex` is tested against the message subject and uses case-insensitive matching.

Common issues:

- The pattern expects text that is not in the subject.
- The pattern anchors to the beginning or end too tightly.
- The message subject has provider-added prefixes such as `Re:` or `Fwd:`.
- The expression is invalid or blocked by safe regex checks.

Start with `subject_contains` to confirm the route can match, then tighten the regex.

### The attachment MIME rule checks content type, not filename

`attachments_mime` matches attachment content types with glob patterns. It does not match file names.

Examples:

```json
{
  "attachments_mime": ["application/pdf"]
}
```

```json
{
  "attachments_mime": ["image/*"]
}
```

If the message has no attachments, `attachments_mime` cannot match. To match messages without attachments, use `none` around a broad attachment MIME rule:

```json
{
  "none": [
    { "attachments_mime": ["*/*"] }
  ]
}
```

### The rule uses unsupported or misspelled keys

Unknown rule keys are accepted for storage but ignored by the matcher.

For example, `recipient_email`, `from_domain`, and `attachment_type` are not route rule keys. Use the supported names from the Rules reference.

### Empty boolean lists do not restrict matching

Empty `any`, `all`, or `none` lists do not make a route stricter.

This rule behaves like a match-all route:

```json
{
  "any": []
}
```

Omit empty boolean blocks when you want a cleaner rule, and add at least one child rule when the branch should enforce a condition.

## Step-by-step checks

### 1. Confirm the message was ingested

Open the mailbox source and confirm the message exists after the mailbox was connected.

Check:

- The email was sent after the mailbox was connected or refreshed.
- The mailbox is enabled.
- The provider setup is active.
- The message is in the folder or label that MailWebhook reads.
- The recipient address is the address configured in MailWebhook.

If the message never reached MailWebhook, fix mailbox setup before changing the route rule.

### 2. Confirm the route is enabled

Open **Routes** and check the expected route.

Verify:

- The route is enabled.
- The route belongs to the same project as the mailbox.
- The route still points to the intended endpoint.
- The route rule JSON is the current saved version.

Route matching runs before pipeline execution and delivery. Endpoint delivery attempts appear only after a route matches and creates an event.

### 3. Use Mailbox Preview

Open **Mailboxes**, choose the mailbox, and open **Mailbox Preview**.

Then:

1. Select the message.
2. Select the route.
3. Wait for the preview status.

Use the result:

| Preview result | Meaning | Next action |
| --- | --- | --- |
| **Selected message does not match this route** | The route predicate returned false. | Compare the rule with the message fields. |
| **Route matched. Preview payload is shown below** | Rule matching passed and the pipeline produced payload. | Check delivery or endpoint behavior. |
| **Route matched, but preview transform failed** | Rule matching passed, then the pipeline failed. | Fix the route pipeline or mapper. |
| **Preview unavailable** | The message could not be fetched or preview is unsupported for the source. | Use a recent message from a supported mailbox or send a new test email. |

### 4. Test with a simple rule

For diagnosis, temporarily reduce the rule to one clear condition.

To match a specific recipient:

```json
{
  "to_emails": ["alerts@example.com"]
}
```

To confirm that routing and delivery work at all:

```json
{}
```

An empty rule matches every ingested message for that project. Use it only as a temporary diagnostic rule unless a catch-all route is intended.

### 5. Compare each matcher with normalized message values

Use this table while inspecting the message and route rule:

| Rule key | Compared with | Check |
| --- | --- | --- |
| `subject_contains` | Lowercased subject string. | Use a plain substring from the actual subject. |
| `subject_regex` | Subject string with case-insensitive regex search. | Test anchors and escaping. |
| `to_contains` | Each normalized recipient address. | Use a substring that appears in the address. |
| `to_emails` | Normalized recipient addresses. | Use the exact recipient address. |
| `from_contains` | Each normalized sender address. | Use a substring that appears in the address. |
| `from_emails` | Normalized sender addresses. | Use the exact sender address. |
| `from_domains` | Domain after `@` in each sender address. | Use the exact sender domain. |
| `headers_equals` | Header value by lowercased header name. | Match the value exactly after trimming. |
| `headers_contains` | Header value by lowercased header name. | Use a case-insensitive substring. |
| `attachments_mime` | Lowercased attachment content types. | Use MIME types such as `application/pdf` or glob patterns such as `image/*`. |

### 6. Check boolean logic

Use `any`, `all`, `none`, and `negate` only when the route needs nested logic.

Example: match invoices from either of two sender domains:

```json
{
  "subject_contains": ["invoice"],
  "any": [
    { "from_domains": ["vendor.com"] },
    { "from_domains": ["billing.example"] }
  ]
}
```

Example: match invoices while excluding an autoresponder:

```json
{
  "subject_contains": ["invoice"],
  "none": [
    { "from_emails": ["bot@vendor.com"] }
  ]
}
```

Keep the first diagnostic rule simple. Add boolean branches only after the simple rule matches.

### 7. Save, retest, and choose the right retry path

After editing the rule, save the route and retest.

Use the right path:

- If Mailbox Preview can fetch the message, preview the same message again.
- If no event was created, send a new email after saving the route.
- If an event exists and the route matched, use **Replay Event** only for delivery or pipeline fixes.

Replay works on existing events. A message that never matched a route usually has no event to replay for that route.

## Expected result

After the rule is fixed:

- **Mailbox Preview** shows **Route matched. Preview payload is shown below** for the selected message and route.
- New matching inbound emails create events for the route.
- The route pipeline runs after matching.
- Delivery attempts appear for HTTP endpoints.
- If the endpoint returns `2xx`, the event is marked `delivered`.

## Related docs

- [Rules]
- [Routes]
- [Safe Regex]
- [Mailboxes]
- [Test routes with a loopback mailbox]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Troubleshoot failed webhook deliveries]

[Rules]: {% link docs/routes/rules.md %}
[Routes]: {% link docs/routes.md %}
[Safe Regex]: {% link docs/routes/safe_regex.md %}
[Mailboxes]: {% link docs/mailboxes.md %}
[Test routes with a loopback mailbox]: {% link docs/mailboxes/loopback-testing.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
