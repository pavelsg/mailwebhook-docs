---
title: Migration
nav_order: 11
has_children: true
description: "Plan MailWebhook migrations from inbound email parsers, legacy webhook handlers, SendGrid Inbound Parse, Zapier email workflows, and custom IMAP jobs."
---

# Migration
{: .no_toc }

Use migration docs when you are replacing an existing email intake or inbound parse workflow with MailWebhook.

Each guide compares the old workflow with the MailWebhook setup path, payload differences, delivery behavior, and rollout checks.

Migration guides:

- [Replace an IMAP polling script with MailWebhook]
- [Migrate from SendGrid Inbound Parse to MailWebhook]
- [Migrate from an email parser tool to MailWebhook]

Reference pages for migration work:

- [Webhook payload reference]
- [Custom JSON]
- [Signed delivery]
- [Webhook retries and replay]
- [API keys and attachment downloads]

[Replace an IMAP polling script with MailWebhook]: {% link docs/migration/replace-imap-polling.md %}
[Migrate from SendGrid Inbound Parse to MailWebhook]: {% link docs/migration/sendgrid-inbound-parse.md %}
[Migrate from an email parser tool to MailWebhook]: {% link docs/migration/email-parser-tool-migration.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[Signed delivery]: {% link docs/delivery/signatures.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
[API keys and attachment downloads]: {% link docs/api/api-keys-and-attachments.md %}
