---
title: CHANGELOG
layout: default
---

# CHANGELOG

All notable user-facing changes to this project are documented in this file.

## Product Update: April 2026

Highlights:
* Added `extract.reply_segments` for [Custom JSON] routes to split new reply content from quoted history, forwarded content, and signatures.
* Added `extract.key_value_pairs` for [Custom JSON] routes to extract structured fields from text or HTML into ordered `items`, first-value `values`, and duplicate-preserving `groups`.
* Documented `extract.tables` for [Custom JSON] routes, including normalized row and column lookups.

### Documentation

* Added a dedicated [JsonLogic-style DSL] reference for Custom JSON operators, helper arguments, output shapes, and runtime limits.
* Added [Custom JSON recipes] covering reply extraction, key/value field extraction, table cell extraction, links, bullet lists, alerts, Slack payloads, and invoice metadata.
* Expanded pipeline and DSL examples for transform steps, normalized key lookups, `values` versus `groups`, table row and column lookups, array helpers, string and regex helpers, and extractor workflows.

## Product Update: March 2026

Highlights:
* Added native Gmail mailbox support
* Added native Microsoft 365 mailbox support for Outlook 365 work mailboxes
* Added temporary loopback mailboxes for onboarding and testing

### Documentation

* Updated mailbox documentation for Gmail, Microsoft 365, loopback, and OAuth fallback guidance

## Release v0.10.2

Features:
* Added 'Backfill' option to supported mailboxes.

## Release v0.10.2

Features:
* Added billing and paid plans
* Added hosted mailbox support for paid plans
* Added Gmail mailbox support

## Release v0.10.1

Hi folks! This is initial MVP release of the MailWebhook.

Highlights:
* Completed most of the server-side work for IMAP protocol
* Completed most of user interface work
* Completed 1st version of custom transformations for incoming emails using [Custom JSON] mapper
* Preview transformation results in user interface
* Attachment download end-point
* Basic sign-up/verification flow

### Bugfixes

N/A

### Documentation

Documentation site launched.

[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[JsonLogic-style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
[Custom JSON recipes]: {% link docs/routes/pipeline/custom_json/recipes.md %}
