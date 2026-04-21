---
title: Pipeline
parent: Routes
nav_order: 2
---

# Pipeline
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

A pipeline is the per-route transformation chain that runs only after a route
rule matches an incoming email. It consumes the parsed message plus route
context and emits a JSON payload; that payload is posted verbatim to the route's
endpoint.

The default step is `map.generic_json`, which emits the deterministic
`mailwebhook.generic@1` shape described in [Generic JSON]. Attachments stay out
of the HTTP body and only their metadata is included, keeping delivery fast and
reliable.

For custom outputs, `map.custom_json` lets you define a JsonLogic-style mapper
validated by JSON Schema, producing whatever JSON shape your downstream system
expects. See [Custom JSON] for the mapper contract and [JsonLogic-style DSL] for
expression operators, helper calls, and runtime limits.

Slack and Telegram use terminal chat mappers:

* `map.slack_simple`
* `map.telegram_simple`

These mappers transform an inbound email into a short chat-ready message and
emit the final payload delivered to the route endpoint. If you need a richer
JSON body, use [Generic JSON] or [Custom JSON] with your own webhook endpoint
instead of the simple chat mappers.

## Pipeline contract

All route pipelines follow the same rules:

1. `pipeline.steps` must contain at least one step.
2. Exactly one `map.*` step must exist.
3. The `map.*` step must be the final step.

Chat mappers accept provider-specific destination arguments:

* Slack: `channel`
* Telegram: `chat_id`

Useful optional arguments supported by both chat mappers:

* `max_chars`
* `include_from`
* `prefix`

## Slack Simple

`map.slack_simple` emits a payload with the following shape:

```json
{
  "channel": "C0123456789",
  "text": "..."
}
```

### Smallest valid Slack pipeline

```json
{
  "steps": [
    {
      "name": "map.slack_simple",
      "args": {
        "channel": "C0123456789"
      }
    }
  ]
}
```

### Slack pipeline with HTML normalization

```json
{
  "steps": [
    {
      "name": "html_to_text",
      "args": {
        "prefer": "html",
        "width": 0
      }
    },
    {
      "name": "map.slack_simple",
      "args": {
        "channel": "C0123456789",
        "max_chars": 700,
        "include_from": true
      }
    }
  ]
}
```

### Slack pipeline with prefix

```json
{
  "steps": [
    {
      "name": "html_to_text",
      "args": {
        "prefer": "html"
      }
    },
    {
      "name": "map.slack_simple",
      "args": {
        "channel": "C0123456789",
        "prefix": "[MailWebhook] ",
        "max_chars": 500,
        "include_from": true
      }
    }
  ]
}
```

### Full route JSON example for Slack

```json
{
  "name": "Invoices to Slack",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["ap@company.com"],
    "from_domains": ["vendor.com"],
    "subject_contains": ["invoice", "receipt"]
  },
  "pipeline": {
    "steps": [
      {
        "name": "html_to_text",
        "args": {
          "prefer": "html",
          "width": 0
        }
      },
      {
        "name": "map.slack_simple",
        "args": {
          "channel": "C0123456789",
          "max_chars": 700,
          "include_from": true
        }
      }
    ]
  }
}
```

### Slack payload after transformation

```json
{
  "channel": "C0123456789",
  "text": "Invoice from vendor@example.com\nSubject: March invoice\n..."
}
```

## Telegram Simple

`map.telegram_simple` emits a payload with the following shape:

```json
{
  "chat_id": "-10012345",
  "text": "..."
}
```

### Smallest valid Telegram pipeline

```json
{
  "steps": [
    {
      "name": "map.telegram_simple",
      "args": {
        "chat_id": "-10012345"
      }
    }
  ]
}
```

### Telegram pipeline with HTML normalization

```json
{
  "steps": [
    {
      "name": "html_to_text",
      "args": {
        "prefer": "html",
        "width": 0
      }
    },
    {
      "name": "map.telegram_simple",
      "args": {
        "chat_id": "-10012345",
        "max_chars": 700,
        "include_from": true
      }
    }
  ]
}
```

### Telegram pipeline with prefix

```json
{
  "steps": [
    {
      "name": "html_to_text",
      "args": {
        "prefer": "html"
      }
    },
    {
      "name": "map.telegram_simple",
      "args": {
        "chat_id": "-10012345",
        "prefix": "[Inbox alert] ",
        "max_chars": 500,
        "include_from": true
      }
    }
  ]
}
```

### Full route JSON example for Telegram

```json
{
  "name": "Alerts to Telegram",
  "endpoint_id": "0f5a83e4-8220-4f0d-917e-6be20d9dc32d",
  "signing_secret_kid": "route-signing-prod",
  "enabled": true,
  "rule": {
    "to_emails": ["alerts@company.com"],
    "subject_contains": ["urgent", "down", "failure"]
  },
  "pipeline": {
    "steps": [
      {
        "name": "html_to_text",
        "args": {
          "prefer": "html",
          "width": 0
        }
      },
      {
        "name": "map.telegram_simple",
        "args": {
          "chat_id": "-10012345",
          "max_chars": 700,
          "include_from": true
        }
      }
    ]
  }
}
```

### Telegram payload after transformation

```json
{
  "chat_id": "-10012345",
  "text": "Alert from sender@example.com\nSubject: Service failure\n..."
}
```

[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[JsonLogic-style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
