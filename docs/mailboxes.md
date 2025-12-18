---
title: Mailboxes
nav_order: 1
---

# Mailboxes
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Mailbox providers

### IMAP

Configure IMAP mailbox:
* `Email/Username`: specify IMAP login username.
* `Password / App Password`: specify IMAP password for authentication.
* `Host`: specify IMAP server hostname.
* `Port`: specify IMAP server port.
* `Use SSL/TLS`: enable/disable TLS for IMAP connection.
* `Folder`: specify mailbox folder to monitor (e.g., `INBOX`).

You can use "Auto-discover" feature to automatically fill in IMAP server settings for providers properly configured SRV records in DNS.

### Gmail

Coming soon.

### Office365

Coming soon.

## Message preview

For each mailbox you can see JSON output preview for specific message. 
Click "Preview" in action menu of a mailbox to open preview modal.
Select [route] to use for transformation and message from mailbox to preview.

Preview will be available only for messages matching [rules] for the selected [route].

You can click "Reply" button to run specific message through the [route] and 
see resulting JSON delivered to the [endpoint] linked to selected [route].

{: .note }
Limitation: no attachments are included in preview.

[Route]: {% link docs/routes.md %}
[Endpoint]: {% link docs/endpoints.md %}
[Rules]: {% link docs/routes/rules.md %}