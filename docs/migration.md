---
title: Migration
nav_order: 11
has_children: true
description: "Plan MailWebhook migrations from inbound email parsers, legacy webhook handlers, SendGrid Inbound Parse, Zapier email workflows, and custom IMAP jobs."
---

# Migration
{: .no_toc }

Use migration docs when you are replacing an existing email intake or inbound parse workflow with MailWebhook.

This section is reserved for migration guides that compare the old workflow, the MailWebhook setup path, payload differences, delivery behavior, and rollout checks.

Available migration guides:

- [Replace an IMAP polling script with MailWebhook]
- [Migrate from SendGrid Inbound Parse to MailWebhook]
- [Migrate from an email parser tool to MailWebhook]

Planned migration topics:

- Legacy webhook payload migration
- Staged rollout and validation

[Replace an IMAP polling script with MailWebhook]: {% link docs/migration/replace-imap-polling.md %}
[Migrate from SendGrid Inbound Parse to MailWebhook]: {% link docs/migration/sendgrid-inbound-parse.md %}
[Migrate from an email parser tool to MailWebhook]: {% link docs/migration/email-parser-tool-migration.md %}
