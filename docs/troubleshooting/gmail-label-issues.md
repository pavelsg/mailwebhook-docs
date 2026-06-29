---
title: Gmail label issues
parent: Troubleshooting
nav_order: 7
description: "Troubleshoot MailWebhook Gmail label filtering by checking the Gmail label ID, live watch setup, history cursor, mailbox preview behavior, backfill, route matching, and delivery events."
permalink: /docs/troubleshooting/gmail-label-issues/
---

# Troubleshoot Gmail label filtering
{: .no_toc }

Use this guide when a Gmail mailbox connects successfully, but messages with the expected Gmail label do not appear in MailWebhook events, previews, or webhook deliveries.

MailWebhook uses Google OAuth and Gmail history sync for the native Gmail connector. When **Gmail label ID (optional)** is set, MailWebhook stores that value as the mailbox `label_filter` and applies it to Gmail watch setup, live history sync, and Gmail backfill.

{: .note }
Connecting Gmail to webhooks? See [Gmail to webhook](https://www.mailwebhook.com/gmail-to-webhook) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Gmail label filtering uses a Gmail label ID, not the visible label display name.

Use `INBOX` when you want messages that carry Gmail's inbox label. Use the exact Gmail label ID for a custom label. Leave the field blank on a new connection when you do not want to restrict the Gmail mailbox to one label.

Normal Gmail setup starts from the current Gmail history cursor. It does not import older labeled mail. Send a new email after setup, or move a new message so it has the intended label after setup, when verifying live sync. Use backfill for older messages.

Mailbox Preview is not the source of truth for custom Gmail label filtering. The current Gmail preview path lists `INBOX` messages for display and does not apply the saved `label_filter`. Use Events, route checks, and provider-real test mail to verify label-filtered ingestion.

## Symptoms

Common symptoms include:

- Gmail OAuth completes, but labeled messages do not create events.
- A route-level test works, but a real Gmail message with a label does not arrive.
- Messages that already had the label before setup never appear.
- Messages in **All Mail**, archive, spam, trash, or another label do not appear for an `INBOX` filter.
- A custom Gmail label name was entered instead of the label ID.
- **Mailbox Preview** shows inbox messages instead of the custom label you configured.
- Reconnecting a Gmail mailbox with the label field blank still behaves like a previous label filter is active.
- A Gmail message appears in MailWebhook, but no route event or delivery attempt is created.

## Most common causes

### The label value is a display name instead of a label ID

MailWebhook sends the configured value to Gmail as a label ID.

For the inbox, the label ID is:

```text
INBOX
```

For a custom Gmail label, use the exact label ID from Gmail, not the visible display name. Display names can be duplicated, renamed, nested, or localized. The Gmail API uses stable label IDs.

### The message does not have the label MailWebhook is watching

When a Gmail label filter is set, MailWebhook includes that label in Gmail watch setup and live Gmail history reads.

If you set `INBOX`, the message must have Gmail's inbox label. Messages that skip the inbox because of Gmail filters, archive rules, or provider placement are outside an `INBOX`-filtered mailbox.

For custom labels, send or move a new message so it has the label after setup. Then verify with a unique subject.

### The message existed before the Gmail history cursor

Normal Gmail setup stores the current Gmail history cursor during OAuth callback and watch creation.

Messages that already existed before setup are not imported by live sync. Use Gmail backfill for older labeled mail.

### Mailbox Preview is showing Gmail inbox preview data

Gmail Mailbox Preview currently lists recent `INBOX` messages for preview display. It does not apply the saved Gmail `label_filter`.

Use **Events**, **Webhook Preview**, route checks, and a new provider-real Gmail message to validate label-filtered ingestion. Do not treat an inbox-only preview result as proof that the custom Gmail label filter is wrong.

### The existing mailbox kept an older label filter

When an existing Gmail mailbox is reconnected with a new label value, MailWebhook updates the stored Gmail account state and establishes a new Gmail watch.

Operational note:

- Reconnect with the intended label ID when changing a filter.
- If you are trying to remove a previously saved label filter and the mailbox still behaves filtered after reconnecting with the field blank, create a fresh Gmail mailbox connection or ask support to clear the stored Gmail label filter.

### Gmail watch or history state needs a fresh connection

Gmail push notifications enqueue a Gmail sync job for the mailbox. Live sync then uses the stored Gmail history cursor and saved label filter to fetch new messages.

Reconnect the Gmail mailbox after OAuth consent changes, revoked Google access, repeated Gmail watch failures, or provider-side account policy changes.

If Gmail reports that the stored history cursor is no longer valid, MailWebhook clears the cursor and seeds from the current Gmail profile history ID on the next sync. Use backfill if you need messages from the gap.

### The message was too large

Gmail messages larger than the configured raw-message size limit are skipped. The default Gmail raw-message limit is 10 MiB.

Use a small, simple test message first:

- Plain subject.
- Short body.
- No large attachments.
- A recipient address your route can match.

### The message was ingested, but no route matched

Gmail sync stores the raw message and queues downstream processing. A webhook event appears only after an enabled route matches the normalized email.

If the Gmail connection is correct but no event appears, debug route matching before changing the Gmail label again.

## Step-by-step checks

### 1. Confirm the intended label mode

Check the Gmail mailbox connection settings.

Use this table:

| Setup choice | What MailWebhook watches |
| --- | --- |
| **Gmail label ID (optional)** is blank on a new connection | Default Gmail mailbox scope configured by the connector |
| `INBOX` | Gmail messages that have Gmail's inbox label |
| Custom Gmail label ID | Gmail messages that have that exact Gmail label ID |

If the setup uses a custom label, confirm the value is the Gmail label ID.

### 2. Reconnect with the exact label ID

Start a clean Gmail connection from **Mailboxes** with the intended label value.

Check:

- Your MailWebhook user email is verified.
- Popups are allowed if the flow opens Google OAuth in a popup.
- You complete OAuth in the same browser session.
- You choose the Google account that owns the mailbox.
- You approve the Google consent prompt.
- The label field contains `INBOX`, a custom Gmail label ID, or is intentionally blank.

If you are removing a label filter from an existing Gmail mailbox and behavior still looks filtered, create a new Gmail mailbox connection or ask support to clear the stored label filter.

### 3. Send or move a new labeled test message

Send a new email after Gmail setup completes.

Use a message that:

- Has a unique subject.
- Is small enough to avoid the raw-message size limit.
- Has the Gmail label you configured.
- Uses a sender or recipient your route can match.

For an `INBOX` filter, make sure the message lands in the inbox. For a custom label, make sure the message has the custom label after setup.

### 4. Check Events before delivery settings

Open **Events** and search for the unique test subject.

Use the result:

| What you see | Meaning | Next action |
| --- | --- | --- |
| No event | The message was not ingested or no route matched. | Check label ID, mailbox state, message size, and route rule. |
| Event exists with no successful delivery | Route matched, but delivery failed. | Inspect delivery attempts. |
| Event is delivered | MailWebhook delivered the event. | Check the receiving application. |

Endpoint URL, headers, signatures, retries, and replay matter after an event exists.

### 5. Treat Mailbox Preview carefully

Open **Mailbox Preview** only as a Gmail-read sanity check.

For custom label troubleshooting, rely on a provider-real test message and **Events**. Gmail preview currently lists inbox messages and can look unrelated to the configured label filter.

### 6. Check route matching

If the Gmail message is ingested but no event appears, check the route.

Verify:

- The route is enabled.
- The route belongs to the same project.
- The rule matches the normalized sender, recipient, subject, headers, or attachment metadata you expect.
- The route pipeline uses the mapper you expect.

Use the route-rule troubleshooting guide before changing Gmail OAuth or label settings again.

### 7. Use backfill for older labeled mail

Use Gmail backfill when you need messages that existed before setup or before a label-filter change.

Gmail backfill uses the same stored Gmail label filter as the mailbox. It searches UTC day windows, asks Gmail for messages that match the date range and label, fetches raw message content, stores new messages, and skips duplicates.

## Expected result

After Gmail label state is correct:

- The Gmail mailbox is connected and enabled.
- The saved label value is blank, `INBOX`, or the intended custom Gmail label ID.
- Gmail watch and live history sync use the saved label filter.
- A new test message with that label is eligible for ingestion.
- The matching route creates an event.
- Delivery attempts appear for matched events.
- Gmail backfill imports eligible older messages for the saved label filter.

## Related docs

- [Gmail setup]
- [Mailboxes]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Routes]
- [Rules]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]
- [Webhook payload reference]
- [Webhook retries and replay]

[Gmail setup]: {% link docs/mailboxes/gmail-setup.md %}
[Mailboxes]: {% link docs/mailboxes.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
