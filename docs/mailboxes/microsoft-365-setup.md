---
title: Microsoft 365 setup
parent: Mailboxes
nav_order: 2
description: "Connect Microsoft 365, Office 365, or Outlook as a MailWebhook mailbox source with Microsoft OAuth, folder targeting, shared mailbox access, and webhook verification."
permalink: /docs/mailboxes/microsoft-365-setup/
---

# Microsoft 365 mailbox setup
{: .no_toc }

Use this guide to connect a Microsoft mailbox to MailWebhook and verify that new messages can reach your webhook route.

MailWebhook uses one **Microsoft 365** connector for Microsoft 365, Office 365, and Outlook mailbox workflows. Access depends on Microsoft OAuth, Microsoft Graph availability, and the mailbox permissions granted to the signed-in account.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Prerequisites

You need:

- A MailWebhook account with a verified user email.
- A MailWebhook project with connected-mailbox capacity available.
- A Microsoft 365, Office 365, or Outlook mailbox you can authorize through Microsoft OAuth.
- Access to the target mailbox and folder.
- A route and endpoint if you want to verify delivery immediately after setup.

The Microsoft connector uses delegated Microsoft Graph access. The required Graph scopes include:

```text
User.Read
Mail.Read
Mail.Read.Shared
```

Your Microsoft organization may require approval before these permissions can be granted.

## Connection steps

Open **Mailboxes** and click **Add Mailbox**.

1. Set **Provider** to **Microsoft 365**.
2. Leave **Connect shared mailbox** off for the signed-in user's own mailbox.
3. Turn on **Connect shared mailbox** only when you want to target a shared mailbox.
4. If shared mailbox mode is on, enter **Shared mailbox email (optional)**.
5. Set **Folder name**. The default is `Inbox`.
6. Optionally set **Folder ID (optional)** when you know the exact Microsoft Graph folder id.
7. Click **Add Mailbox** to start the Microsoft OAuth flow.
8. Complete the Microsoft consent flow.
9. Return to MailWebhook and confirm the mailbox appears in **Mailboxes**.

In onboarding or other modal flows, choosing **Microsoft 365** can open the Microsoft OAuth flow in a popup. If the popup is blocked, allow popups for MailWebhook and start the connection again.

## Microsoft-specific fields

| Field | Use |
| --- | --- |
| **Connect shared mailbox** | Enable this when the target mailbox is a shared mailbox instead of the signed-in user's own mailbox. |
| **Shared mailbox email (optional)** | Email address for the shared mailbox. The signed-in account must have delegated access. |
| **Folder name** | Folder display name to monitor. The default is `Inbox`, and matching is case-insensitive during OAuth callback. |
| **Folder ID (optional)** | Exact Microsoft Graph folder id. When both folder id and folder name are set, folder id takes precedence. |

Use folder name for normal setup. Use folder id when two folders have similar names, when a folder name is renamed often, or when a support workflow gives you the exact Graph folder id.

## Shared mailbox setup

Shared mailbox setup uses the same Microsoft 365 connector.

To connect a shared mailbox:

1. Choose **Microsoft 365** as the provider.
2. Enable **Connect shared mailbox**.
3. Enter the shared mailbox address in **Shared mailbox email (optional)**.
4. Choose the folder by **Folder name** or **Folder ID (optional)**.
5. Start the Microsoft OAuth flow with an account that has access to that shared mailbox.

MailWebhook requests `Mail.Read.Shared` so delegated shared-mailbox access can work when Microsoft grants it. Microsoft still controls whether the signed-in account can read the target mailbox and folder.

## What happens after consent

After Microsoft returns control to MailWebhook, MailWebhook:

- Resolves the signed-in Microsoft profile.
- Resolves the target mailbox. For shared mailbox setup, this is the shared mailbox address you entered.
- Resolves the target folder by folder id, folder name, or the default `Inbox`.
- Creates or reconnects the Microsoft 365 mailbox for the current project.
- Stores the OAuth refresh token securely.
- Creates or renews a Microsoft Graph subscription for message `created,updated` changes.
- Seeds a Microsoft Graph delta cursor for the target folder.

Initial setup starts from the current folder delta state. It does not import older mail during normal setup. Use mailbox backfill when you need messages that arrived before the Microsoft mailbox was connected.

## Setup checks

After the OAuth flow completes, check:

- The mailbox is listed in **Mailboxes**.
- The mailbox source is Microsoft 365.
- The mailbox is enabled.
- The stored mailbox email is the expected signed-in mailbox or shared mailbox.
- The selected folder is the folder you expect.
- Your route rule matches the Microsoft mailbox address or the recipients you expect.
- Your route pipeline uses the mapper you want, such as `map.generic_json`.
- Your endpoint returns `2xx` only after it accepts the request.

If you selected a custom folder, send or move a new message into that folder after setup. Live Microsoft sync uses the stored folder target.

## Verify payload and delivery

Send a new email to the connected Microsoft mailbox. Then open **Events** or **Webhook Preview**.

For a route that uses `map.generic_json`, the payload includes `meta.source` set to `ms365`:

```json
{
  "schema": {
    "name": "mailwebhook.generic",
    "version": "1"
  },
  "event": {
    "id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
    "project_id": "dca29061-c4a7-4687-a8dd-24d2f26548c7",
    "route_id": "2f3713bf-88cc-46c6-aaa3-ea9d6e9d20f3",
    "created_at": "2026-06-28T12:00:02Z"
  },
  "message": {
    "message_id": "<message-id@outlook.com>",
    "message_id_type": "original",
    "subject": "Microsoft mailbox test",
    "date": "2026-06-28T12:00:00Z",
    "from": [{ "email": "sender@example.com" }],
    "to": [{ "email": "connected@example.com" }]
  },
  "body": {
    "attachments": [],
    "text": "Test message"
  },
  "meta": {
    "source": "ms365",
    "raw_size_bytes": 2048,
    "received_at": "2026-06-28T12:00:00Z"
  }
}
```

The exact payload depends on your route pipeline. See [Webhook payload reference] for mapper choices and [Generic JSON] for the full default payload contract.

## Backfill older Microsoft messages

Normal Microsoft setup starts from the current folder cursor. To ingest older messages, open the mailbox actions menu and start **Backfill**.

Backfill runs in UTC day windows and is bounded by your plan's backfill limit. It is separate from live Microsoft Graph subscription and delta sync.

## Common errors

### Please verify your email address

MailWebhook requires a verified user email before connecting a Microsoft mailbox. Verify your MailWebhook user email and start the Microsoft connection again.

### Popup was blocked

Allow popups for MailWebhook and retry the Microsoft connection. This applies to modal flows that open Microsoft OAuth in a popup window.

### Microsoft 365 connection failed

Start the connection again and complete the Microsoft consent flow. If the error persists, check whether your Microsoft organization blocks the requested Graph permissions.

### No refresh token returned

The OAuth flow must return a refresh token so MailWebhook can keep the mailbox connected. Retry the flow and approve consent when Microsoft asks.

### Shared mailbox cannot be read

Confirm the signed-in account has delegated access to the shared mailbox. Also confirm the mailbox address in **Shared mailbox email (optional)** is the mailbox you want MailWebhook to monitor.

### Selected folder was not found

Check **Folder name** or **Folder ID (optional)**. Folder-name matching is case-insensitive, but the folder must exist for the target mailbox. If both folder id and folder name are set, folder id wins.

### Mailbox limit reached

Your project has no connected-mailbox capacity left. Remove an unused connected mailbox or change to a plan with more connected mailboxes.

### New mail is not arriving

Check these items:

- The Microsoft mailbox is enabled.
- The message arrived after Microsoft setup completed.
- The message is in the folder selected during setup.
- The route rule matches the message.
- The endpoint accepts the delivery with a `2xx` response.

If you need to inspect provider-real message fields, send a normal email to the Microsoft mailbox. Route-level test events are useful for route and endpoint checks, but they do not prove Microsoft folder targeting or shared-mailbox access.

## Related docs

- [Mailboxes]
- [Receive your first inbound email webhook]
- [Send a test email and inspect the payload]
- [Routes]
- [Rules]
- [Endpoints]
- [Webhook payload reference]
- [Generic JSON]
- [Webhook retries and replay]
- [Microsoft 365 email to webhook]

[Mailboxes]: {% link docs/mailboxes.md %}
[Receive your first inbound email webhook]: {% link docs/quickstart/first-webhook.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Routes]: {% link docs/routes.md %}
[Rules]: {% link docs/routes/rules.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[Microsoft 365 email to webhook]: https://www.mailwebhook.com/microsoft-365-email-to-webhook
