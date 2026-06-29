---
title: Recipes
nav_order: 9
has_children: true
description: "Apply MailWebhook recipes for common email automation workflows such as support tickets, alerts, CRM intake, invoices, Slack, and Telegram."
---

# Recipes
{: .no_toc }

Use recipes when you want a practical workflow pattern instead of a reference page. Each recipe combines mailbox intake, route matching, payload mapping, endpoint setup, and delivery verification for a common email automation job.

Notification recipes:

- [Send inbound email to Slack]
- [Send inbound email alerts to Telegram]

Workflow recipes:

- [Turn support emails into ticket webhooks]
- [Send lead emails to a CRM webhook]

References used across recipes:

- [Routes]
- [Webhook payload reference]
- [Endpoints]
- [Signed delivery]

[Send inbound email to Slack]: {% link docs/recipes/slack-email-webhooks.md %}
[Send inbound email alerts to Telegram]: {% link docs/recipes/telegram-email-alerts.md %}
[Turn support emails into ticket webhooks]: {% link docs/recipes/support-ticket-webhooks.md %}
[Send lead emails to a CRM webhook]: {% link docs/recipes/crm-lead-email-webhook.md %}
[Routes]: {% link docs/routes.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Signed delivery]: {% link docs/delivery/signatures.md %}
