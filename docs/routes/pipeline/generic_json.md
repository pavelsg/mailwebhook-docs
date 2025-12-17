---
title: >
    Generic JSON
parent: Pipeline
nav_order: 2
---

# Generic JSON
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

This mapper emits payloads that conform to [`mailwebhook.generic@1`](#schema-definition). Attachments stay out of the HTTP body and no URLs are embedded.

## Shape

- Top-level keys: `schema`, `event`, `message`, `body`, `meta`, optional `envelope`.
- `schema`: `{ "name": "mailwebhook.generic", "version": "1" }`
- `event`: `{ id, project_id, route_id, created_at }` (all strings; `created_at` is RFC3339 UTC, no fractional seconds).
- `message`: includes `message_id`, `message_id_type` (`original|synthetic`), `subject`, `date`, people arrays (`from`, `to`, optional `reply_to|cc|bcc`), optional normalized `headers`.
- `body`: `attachments` (required array) plus optional `text`/`html`.
- `meta`: `source` (enum string), `raw_size_bytes`, `received_at` (RFC3339 UTC), optional `spam` (for hosted mailboxes only).
- `envelope` (optional): `mail_from` and `rcpt_to` (unique, sorted).

## Determinism rules enforced

- People arrays are sorted by email; emails are lowercased and trimmed.
- Attachments are sorted by `(filename, size)` and include `{id, filename, content_type, size, is_inline, content_id?, sha256?}`.
- Headers are lowercased, unfolded, trimmed; duplicates are joined with `", "`. Empty values are dropped.
- Times are UTC (`Z`), whole seconds.
- Optional fields are omitted when empty.
- Sorting/dedup behaviors above are enforced at runtime; JSON Schema cannot express ordering constraints (it enforces non-empty headers, unique attachments/rcpt_to).

## Schema definition

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.mailwebhook.dev/transform/generic@1",
  "title": "mailwebhook.generic v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema", "event", "message", "body", "meta"],
  "properties": {
    "schema": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name", "version"],
      "properties": {
        "name": { "const": "mailwebhook.generic" },
        "version": { "const": "1" }
      }
    },
    "event": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "project_id", "route_id", "created_at"],
      "properties": {
        "id": { "type": "string", "minLength": 1 },
        "project_id": { "type": "string", "minLength": 1 },
        "route_id": { "type": "string", "minLength": 1 },
        "created_at": {
          "type": "string",
          "format": "date-time",
          "pattern": "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}Z$"
        }
      }
    },
    "message": {
      "type": "object",
      "additionalProperties": false,
      "required": ["message_id", "message_id_type", "subject", "date", "from", "to"],
      "properties": {
        "message_id": { "type": "string", "minLength": 1 },
        "message_id_type": { "enum": ["original", "synthetic"] },
        "subject": { "type": "string" },
        "date": {
          "type": "string",
          "format": "date-time",
          "pattern": "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}Z$"
        },
        "from": { "$ref": "#/$defs/people" },
        "reply_to": { "$ref": "#/$defs/people" },
        "to": { "$ref": "#/$defs/people" },
        "cc": { "$ref": "#/$defs/people" },
        "bcc": { "$ref": "#/$defs/people" },
        "headers": {
          "type": "object",
          "patternProperties": {
            "^[a-z0-9_-]+$": {
              "type": "string",
              "minLength": 1,
              "pattern": "^\\S(.*\\S)?$"
            }
          },
          "additionalProperties": false
        }
      }
    },
    "envelope": {
      "type": "object",
      "additionalProperties": false,
      "required": ["mail_from", "rcpt_to"],
      "properties": {
        "mail_from": { "type": "string", "format": "idn-email" },
        "rcpt_to": {
          "type": "array",
          "items": { "type": "string", "format": "idn-email" },
          "uniqueItems": true,
          "minItems": 0
        }
      }
    },
    "body": {
      "type": "object",
      "additionalProperties": false,
      "required": ["attachments"],
      "properties": {
        "text": { "type": "string" },
        "html": { "type": "string" },
        "attachments": {
          "type": "array",
          "items": { "$ref": "#/$defs/attachment" },
          "uniqueItems": true,
          "minItems": 0
        }
      }
    },
    "meta": {
      "type": "object",
      "additionalProperties": false,
      "required": ["source", "raw_size_bytes", "received_at"],
      "properties": {
        "source": { "enum": ["imap", "hosted", "api", "cli"] },
        "raw_size_bytes": { "type": "integer", "minimum": 0 },
        "received_at": {
          "type": "string",
          "format": "date-time",
          "pattern": "^\\d{4}-\\d{2}-\\d{2}T\\d{2}:\\d{2}:\\d{2}Z$"
        },
        "spam": {
          "type": "object",
          "additionalProperties": false,
          "minProperties": 1,
          "properties": {
            "flag": { "type": "boolean" },
            "score": { "type": "number" },
            "quarantined": { "type": "boolean" },
            "status": { "type": "string" },
            "details": { "type": "object" }
          }
        }
      }
    }
  },
  "$defs": {
    "people": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["email"],
        "properties": {
          "name": { "type": "string" },
          "email": { "type": "string", "format": "idn-email" }
        }
      }
    },
    "attachment": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "filename", "content_type", "size", "is_inline"],
      "properties": {
        "id": { "type": "string", "minLength": 1 },
        "filename": { "type": "string" },
        "content_type": { "type": "string" },
        "size": { "type": "integer", "minimum": 0 },
        "is_inline": { "type": "boolean" },
        "content_id": { "type": "string" },
        "sha256": { "type": "string", "pattern": "^[a-f0-9]{64}$" }
      }
    }
  }
}
```

## Example

Find [example of default email] here.

[example of default email]: {% link docs/routes/pipeline/generic_json/example.md %}
