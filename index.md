---
title: Home
layout: home
nav_order: 1
description: "MailWebhook turns received email messages into deterministic JSON delivered to your backend."
permalink: /
---

# MailWebhook Documentation
{: .fs-9 }

MailWebhook turns received email messages into deterministic JSON delivered to your backend.
{: .fs-6 .fw-300 }

{: .warning }
This project is a work-in-progress, so information on this page will update frequently.
We try to maintain backward compatibility though.

## Quick Start

### General concepts

MailWebhook operates on 3 core concepts:
- [Mailboxes]
- [Routes]
- [Endpoints]

[Mailboxes] are polled at pre-defined intervals. All ingested emails are pushed to all [Routes]. [Routes] do the matching, transform matched emails and deliver those to connected webhook [Endpoints].

```mermaid
graph TD;
    accTitle: Conceptual diagram
    accDescr: a graph with four nodes: A points to B and C, while B and C both point to D
    Mailbox1-->Route1;
    Mailbox1-->Route2;
    Mailbox2-->Route1;
    Mailbox2-->Route2;
    Route1-->Endpoint1;
    Route2-->Endpoint2;
```

### 1. Connect the mailbox.

From Dashboard or [Mailboxes] screens click "+Add Mailbox" button. Only IMAP connector is supported for now, Gmail and Outlook365 is on the way.

### 2. Connect endpoint

From Dashboard or [Endpoints] screens click "+Add Endpoint" button. No auth supported at the moment, header-based auth is coming soon.

### 3. Define route

Each email received from all configured mailboxes hit all [Routes]. Routes composed of 3 components:

1. [Rules] match incoming emails to defined set of criteria
2. [Pipeline] transform email into desired shape
3. [Endpoints] Attach route to the Endpoint.

---

[Mailboxes]: {% link docs/mailboxes.md %}
[Routes]: {% link docs/routes.md %}
[Endpoints]: {% link docs/endpoints.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
