---
title: Microsoft mailbox issues
parent: Troubleshooting
nav_order: 6
description: "Troubleshoot MailWebhook Microsoft 365, Office 365, and Outlook mailbox connection issues by checking OAuth consent, refresh tokens, shared mailbox access, folder targeting, Graph subscriptions, delta sync, backfill, route matching, and delivery events."
permalink: /docs/troubleshooting/microsoft-mailbox-issues/
---

# Troubleshoot Microsoft mailbox connection issues
{: .no_toc }

Use this guide when a Microsoft 365, Office 365, or Outlook mailbox does not connect, does not preview messages, or does not create webhook events after setup.

MailWebhook uses one **Microsoft 365** connector for Microsoft 365, Office 365, and Outlook mailbox workflows. The connector depends on Microsoft OAuth, Microsoft Graph mailbox permissions, folder targeting, Graph subscriptions, and Graph delta sync.

{: .note }
Connecting Microsoft mailboxes to webhooks? See [Microsoft 365 email to webhook](https://www.mailwebhook.com/microsoft-365-email-to-webhook) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Most Microsoft mailbox issues fall into one of four areas:

- OAuth did not complete with the permissions and refresh token MailWebhook needs.
- The signed-in account cannot read the selected mailbox or shared mailbox.
- The selected folder does not match the folder where the message arrived.
- The mailbox is connected, but the message is older than the live Graph delta cursor or the route did not match.

Reconnect the Microsoft mailbox when OAuth consent, shared mailbox access, or folder targeting changes. Send a brand-new email to the selected folder after setup to verify live sync. Use backfill for messages that arrived before setup or before a folder or target-mailbox change.

## Symptoms

Common symptoms include:

- Microsoft OAuth returns a connection error.
- The OAuth popup is blocked or closes without linking a mailbox.
- MailWebhook shows **Please verify your email address before connecting a mailbox**.
- The callback reports an invalid or expired OAuth state.
- Microsoft returns no refresh token.
- The selected shared mailbox cannot be read.
- The selected folder is not found.
- The mailbox appears in **Mailboxes**, but **Mailbox Preview** is empty.
- A route-level test works, but a real Microsoft mailbox email does not create an event.
- New mail stops arriving after changing the shared mailbox or folder.
- Existing mail in the folder never appears in events.
- Delivery settings look correct, but no Microsoft mailbox event exists.

## Most common causes

### The MailWebhook user email is not verified

MailWebhook requires a verified user email before starting the Microsoft mailbox connection.

If the user is not verified, the OAuth start and callback flows stop before the mailbox is linked. Verify the MailWebhook user email, then start the Microsoft connection again.

### The OAuth popup or state expired

Microsoft OAuth can run in a popup from onboarding, the dashboard, or the mailbox page.

If the popup is blocked, MailWebhook cannot complete the connection in that flow. Allow popups for MailWebhook and start the connection again.

OAuth state is also time-limited and tied to the MailWebhook project. If the callback says the state is invalid or expired, restart the Microsoft connection from the same project and finish the flow in one browser session.

### Microsoft did not grant the required Graph access

The Microsoft connector uses delegated Microsoft Graph access. The required Graph permissions include:

```text
User.Read
Mail.Read
Mail.Read.Shared
```

Some Microsoft organizations require admin approval before these permissions can be granted. If Microsoft blocks consent, ask the Microsoft admin to approve the app permissions, then reconnect the mailbox.

### Microsoft did not return a refresh token

MailWebhook needs a refresh token to keep the mailbox connected after the initial OAuth session.

If Microsoft returns no refresh token, retry the OAuth flow and approve consent when prompted. If the account or tenant policy blocks refresh-token issuance, the mailbox cannot stay connected until the policy is changed.

### The shared mailbox target cannot be read

Shared mailbox setup uses the same Microsoft 365 connector.

When **Connect shared mailbox** is enabled, the signed-in Microsoft account must have delegated access to the shared mailbox. MailWebhook requests `Mail.Read.Shared`, but Microsoft still decides whether the signed-in account can read the target mailbox and folder.

Check the shared mailbox email and confirm that the signed-in account can open that mailbox in Microsoft before reconnecting.

### The selected folder does not resolve

MailWebhook resolves the target folder during OAuth callback.

Folder targeting works this way:

- **Folder name** defaults to `Inbox`.
- Folder-name matching is case-insensitive.
- **Folder ID (optional)** takes precedence over folder name.
- The selected folder must exist in the target mailbox.

Use folder name for normal setup. Use folder ID when folder names are duplicated, frequently renamed, localized, or hard to identify.

### The connection starts from the current folder state

Normal Microsoft setup does not import existing mail.

After OAuth, MailWebhook resolves the folder, creates or renews a Graph subscription, and seeds a Microsoft Graph delta cursor for that folder. The first live sync starts from the current folder state and waits for newer changes.

If you reconnect to a different target mailbox or folder, MailWebhook resets the stored delta and subscription state for that mailbox target. The next sync seeds a fresh cursor. Use backfill for older messages from the new target.

### Graph subscription or delta sync needs attention

MailWebhook uses Microsoft Graph subscriptions for message `created,updated` changes and Graph delta sync for the selected folder.

During sync, MailWebhook refreshes the access token, ensures the Graph subscription is active, and walks the stored delta cursor. If Microsoft returns auth, throttling, or provider errors, live sync can stop until the token, permissions, provider state, or Microsoft availability issue is fixed.

Reconnect the mailbox after permission changes, password resets, tenant policy changes, revoked app access, or repeated Microsoft auth failures.

### The message was too large

Microsoft messages larger than the configured raw-message size limit are skipped. The default Microsoft raw-message limit is 10 MiB.

If a test message is large or has large attachments, send a smaller message first to prove the connection path.

### The message arrived, but no route matched

Microsoft mailbox sync stores the raw message and queues downstream processing. A webhook event appears only after an enabled route matches the normalized email.

If **Mailbox Preview** can read the message but no event appears, debug route matching before changing Microsoft OAuth or folder settings again.

## Step-by-step checks

### 1. Confirm the MailWebhook user is verified

Open the MailWebhook app and check whether the signed-in user has verified their email address.

If verification is pending, complete it first. Then restart the Microsoft mailbox connection.

### 2. Restart the OAuth flow cleanly

Start a new Microsoft connection from **Mailboxes**.

Check:

- Popups are allowed for MailWebhook.
- You complete the flow in the same browser session.
- You connect from the intended MailWebhook project.
- You select the Microsoft account that has access to the target mailbox.
- You approve requested Microsoft permissions when prompted.

If the callback mentions an invalid or expired state, start over from the MailWebhook mailbox page.

### 3. Check Microsoft consent and tenant policy

If Microsoft blocks the connection, check whether the organization requires admin consent.

The account or admin must allow delegated Graph access for:

- `User.Read`
- `Mail.Read`
- `Mail.Read.Shared`

After permissions are approved or tenant policy changes, reconnect the mailbox so MailWebhook stores fresh OAuth and Graph state.

### 4. Confirm the mailbox target

For the signed-in user's own mailbox, leave **Connect shared mailbox** off.

For a shared mailbox:

- Turn on **Connect shared mailbox**.
- Enter the shared mailbox address.
- Sign in with an account that has delegated access to that shared mailbox.
- Confirm that account can read the shared mailbox in Microsoft.

If the shared mailbox address or access changed, reconnect the mailbox.

### 5. Confirm folder targeting

Check the folder settings used during connection.

Use this table:

| Setup choice | What MailWebhook uses |
| --- | --- |
| Folder ID is set | The matching Graph folder ID |
| Folder ID is blank and folder name is set | The case-insensitive folder display-name match |
| Both are blank | Default `Inbox` |

If the message is in another folder, move your test workflow to the selected folder or reconnect with the correct folder target.

### 6. Send a brand-new email to the selected folder

Send a new email after Microsoft setup completes.

Use a simple message first:

- No large attachments.
- A unique subject.
- A recipient address your route can match.
- A message that lands in the selected folder.

Existing messages in the folder are not proof that live sync is broken. Normal setup starts from the current folder cursor.

### 7. Check Mailbox Preview

Open **Mailbox Preview** for the Microsoft mailbox.

If the new message appears, MailWebhook can read the selected Microsoft folder. Continue with route matching and delivery checks.

If the new message does not appear, focus on OAuth token state, mailbox access, folder targeting, Microsoft Graph availability, or message size.

### 8. Check Events before delivery settings

Open **Events** and search for the test message subject.

Use the result:

| What you see | Meaning | Next action |
| --- | --- | --- |
| No event | The message was not ingested or no route matched. | Check preview, folder, and route rule. |
| Event exists with no successful delivery | Route matched, but delivery failed. | Inspect delivery attempts. |
| Event is delivered | MailWebhook delivered the event. | Check the receiving application. |

Endpoint URL, custom headers, signatures, retries, and replay matter after an event exists.

### 9. Use backfill for older messages

Use mailbox backfill when you need messages that arrived before setup or before a mailbox target or folder change.

Microsoft backfill uses the same target mailbox and selected folder. It lists messages in UTC time windows, fetches their MIME content from Graph, stores new raw messages, and skips duplicates.

## Expected result

After the Microsoft mailbox state is correct:

- The MailWebhook user is verified.
- OAuth completes and stores a refresh token.
- The mailbox appears as a Microsoft 365 source.
- The target mailbox and folder are the intended ones.
- Graph subscription and delta sync state are active.
- A new email sent after setup appears in **Mailbox Preview**.
- The matching route creates an event.
- Delivery attempts appear for matched events.
- Backfill imports eligible historical messages from the selected Microsoft folder.

## Related docs

- [Microsoft 365 mailbox setup]
- [Mailboxes]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Routes]
- [Rules]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]
- [Webhook payload reference]
- [Webhook retries and replay]

[Microsoft 365 mailbox setup]: {% link docs/mailboxes/microsoft-365-setup.md %}
[Mailboxes]: {% link docs/mailboxes.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
