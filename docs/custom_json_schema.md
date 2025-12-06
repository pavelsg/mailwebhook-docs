---
title: Generic JSON Schema
nav_order: 2
---

# Generic JSON Schema
{: .no_toc }

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
        "created_at": { "type": "string", "format": "date-time" }
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
        "date": { "type": "string", "format": "date-time" },
        "from": { "$ref": "#/$defs/people" },
        "reply_to": { "$ref": "#/$defs/people" },
        "to": { "$ref": "#/$defs/people" },
        "cc": { "$ref": "#/$defs/people" },
        "bcc": { "$ref": "#/$defs/people" },
        "headers": {
          "type": "object",
          "patternProperties": {
            "^[a-z0-9_-]+$": { "type": "string" }
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
          "uniqueItems": true
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
          "items": { "$ref": "#/$defs/attachment" }
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
        "received_at": { "type": "string", "format": "date-time" },
        "spam": {
          "type": "object",
          "additionalProperties": false,
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
