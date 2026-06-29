---
title: IMAP folder and UID issues
parent: Troubleshooting
nav_order: 5
description: "Troubleshoot MailWebhook IMAP folder and UID issues by checking the selected folder, live UID cursor, folder edits, oversized messages, backfill, route matching, and delivery events."
permalink: /docs/troubleshooting/imap-folder-and-uid-issues/
---

# Troubleshoot IMAP folder and UID issues
{: .no_toc }

Use this guide when an IMAP mailbox connects successfully, but expected messages do not appear in MailWebhook events, previews, or webhook deliveries.

The fastest path is to confirm which folder MailWebhook is polling, then separate live polling from backfill. Live polling is UID-based and starts after the mailbox is connected. Backfill is the path for older messages.

{: .note }
Connecting a generic IMAP mailbox to webhooks? See [IMAP to webhook](https://www.mailwebhook.com/imap-to-webhook) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

MailWebhook polls one configured IMAP folder per mailbox. The default folder is `INBOX`.

For a new IMAP mailbox, the first live poll records the current highest UID in that folder and does not import existing messages. Later live polls fetch messages with UIDs after that stored cursor.

Use a new test email to verify live polling. Use mailbox backfill when you need messages that were already in the folder before the mailbox was connected, or when you need historical messages after changing your setup.

If you changed the folder on an existing IMAP mailbox, remember that the saved `last_uid` cursor is retained. Because IMAP UIDs are scoped to a folder, a changed folder can appear quiet until that folder produces UIDs higher than the retained cursor. For a reliable folder change, create a new IMAP mailbox for the new folder or reset the cursor with support, then use backfill for older messages.

## Symptoms

Common symptoms include:

- **Test Connection** succeeds, but no new events appear.
- The message exists in the mailbox provider, but not in **Events**.
- **Mailbox Preview** does not show the expected message.
- **Webhook Preview** stays empty after sending a test email.
- Messages that were already in the folder before setup never arrive.
- Messages stop arriving after the configured IMAP folder is changed.
- Moving or copying a message between folders does not create the expected webhook.
- The mailbox item shows a `last_uid`, but the expected message has a lower UID.
- Route and endpoint settings look correct, but no route event is created.

## Most common causes

### MailWebhook is polling a different folder

IMAP folders are provider-specific. MailWebhook selects the configured folder during connection tests, preview, live polling, message fetch, and backfill.

The default is `INBOX`. If the provider stores the message in another folder, label, archive view, spam folder, or custom folder, a mailbox configured for `INBOX` will not read it.

Check the exact folder name in the mailbox settings. Folder names are often case-sensitive and can include provider-specific prefixes.

### The message existed before live polling started

Normal IMAP polling does not import existing messages on first connection.

When `last_uid` is empty, MailWebhook opens the configured folder, reads the current highest UID, stores it, and waits for newer UIDs. This prevents an existing mailbox from unexpectedly sending old mail to webhooks during setup.

Use backfill for historical messages.

### The folder was changed but the UID cursor was retained

IMAP UIDs are scoped to the selected folder. The same mailbox can have one UID sequence in `INBOX` and a different sequence in another folder.

MailWebhook stores a `last_uid` cursor on the IMAP mailbox. Updating the folder setting does not reset that cursor. If the new folder's UID sequence is lower than the retained cursor, live polling will search for UIDs after the old cursor and may not find new messages in the new folder for a long time.

For a clean switch to another folder, create a new IMAP mailbox that points at the intended folder, or ask support to reset the IMAP cursor for the existing mailbox. Use backfill when you also need historical messages from that folder.

### The message was moved or copied between folders

IMAP UIDs belong to a folder, not to the message globally.

Moving or copying a message can create a different UID in the destination folder. MailWebhook only polls the configured folder and only fetches UIDs after the stored cursor for that folder.

Use a brand-new email that lands directly in the configured folder when verifying live polling.

### The message was larger than the IMAP raw-message limit

MailWebhook skips IMAP messages larger than the configured raw-message size limit. The default limit is 10 MiB.

When an oversized message has a newer UID, MailWebhook can still advance the cursor beyond that UID. That means a later retry of the same oversized message will not be picked up by normal live polling.

If size is the issue, reduce the email size and send a new message.

### The mailbox is not active or enabled

The poller loads active, enabled mailboxes. Hosted and loopback mailboxes are not part of IMAP polling.

Check that the mailbox is enabled, active, and has a valid IMAP configuration. Repeated unhandled poller failures can disable a mailbox after the configured failure limit.

### The message was ingested but no route matched

IMAP polling only stores the raw message and queues downstream processing. A webhook event appears only after an enabled route matches the normalized email.

If the message appears in preview but no event or delivery exists, debug the route rule next.

### Backfill is needed instead of live polling

Backfill is separate from live polling. For IMAP, backfill searches the selected folder by UTC day windows, stores messages by IMAP UID, and skips messages that were already ingested.

Use backfill for older messages, for setup validation with existing mail, and for recovery after choosing the wrong folder.

## Step-by-step checks

### 1. Confirm the configured folder

Open **Mailboxes**, select the IMAP mailbox, and check **Folder**.

Verify:

- The folder is the one that contains the test message.
- The folder name is exact, including case and provider-specific prefixes.
- The message is not only visible through an archive, search, or label view outside the configured folder.
- **Test Connection** succeeds with the saved folder.

If the message is not in that folder, move your test workflow to the configured folder or update the mailbox to the correct folder.

### 2. Send a brand-new test email

Send a new email after the IMAP mailbox is saved and enabled.

Use a simple message first:

- No large attachments.
- A unique subject.
- A recipient address that your route can match.
- A message that lands directly in the configured folder.

Do not use an old message, a forwarded provider search result, or a message moved from another folder as the first test.

### 3. Wait for the poll interval

IMAP mailboxes are polled on an interval.

Check:

- The saved poll interval.
- Any plan-level polling floor that may make polling less frequent.
- Whether the mailbox was recently disabled or re-enabled.

After the interval passes, check **Events** and **Mailbox Preview**.

### 4. Check Mailbox Preview

Open **Mailbox Preview** for the IMAP mailbox.

If the test message appears, MailWebhook can read the configured folder. Continue to route troubleshooting if no event appears.

If the test message does not appear, the issue is still in mailbox setup, folder selection, provider placement, UID cursor state, or message size.

### 5. Compare live polling with backfill

Use this decision table:

| Situation | Use |
| --- | --- |
| Message arrived after setup and has a new UID in the configured folder | Live polling |
| Message was already present before setup | Backfill |
| You changed the folder and need older messages from the new folder | Backfill |
| You changed the folder and need reliable future live polling | New IMAP mailbox or support cursor reset |
| The message was moved or copied between folders | Send a new message directly to the configured folder |

Backfill uses the same IMAP host, port, SSL/TLS, username, password, and folder as the mailbox.

### 6. Check the UID cursor expectation

If the mailbox API or UI shows `last_uid`, treat it as the live-polling checkpoint for the configured IMAP mailbox.

Live polling fetches from:

```text
last_uid + 1
```

If the message you expect has a UID lower than or equal to `last_uid`, normal live polling will not fetch it. Use backfill for historical messages, or create a new IMAP mailbox for a corrected folder setup.

### 7. Check route matching after ingest

When Mailbox Preview can read the message but no event appears, open the route and test the message against it.

Check:

- The route is enabled.
- The route belongs to the same project.
- The rule matches the normalized sender, recipient, subject, headers, and attachment metadata.
- The route pipeline uses the payload mapper you expect.

If the route does not match, use the route-rule troubleshooting guide before changing IMAP settings again.

### 8. Check delivery only after an event exists

Endpoint delivery happens after a route creates an event.

If there is no event, delivery settings are not the first problem. Continue with mailbox, folder, UID, and route checks.

If an event exists but delivery failed, inspect the delivery attempts for HTTP status, network errors, signatures, retries, and replay.

## Expected result

After the folder and UID state are correct:

- **Test Connection** succeeds for the configured folder.
- A new email sent after setup appears in **Mailbox Preview**.
- The matching route creates an event.
- The webhook payload uses the configured route pipeline.
- Delivery attempts appear for matched events.
- Backfill imports eligible historical messages from the selected folder without changing the live `last_uid` cursor.

## Related docs

- [IMAP mailbox configuration]
- [Mailboxes]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Routes]
- [Rules]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]
- [Webhook payload reference]
- [Webhook retries and replay]

[IMAP mailbox configuration]: {% link docs/mailboxes/imap-configuration.md %}
[Mailboxes]: {% link docs/mailboxes.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
