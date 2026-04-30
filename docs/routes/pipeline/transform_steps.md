---
title: Transform Steps
parent: Pipeline
nav_order: 1
---

# Transform Steps
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

Transform steps run before the final `map.*` step and return a modified message
for later steps. Use them to normalize HTML, redact fields, rewrite values, or
drop attachments before the payload is mapped.

Built-in non-terminal steps:

| Step | Use it for |
|------|------------|
| `html_to_text` | Convert message HTML into deterministic plain text. |
| `remove_fields` | Remove or clear selected message fields before mapping. |
| `replace_values` | Set subject, body, or header values from literals or small templates. |
| `strip_attachments_if` | Remove attachments that match configured MIME, size, and filename conditions. |

## `html_to_text`

`html_to_text` writes deterministic text to `message.text` from `message.html`.
If the message has no HTML, the step leaves the message unchanged.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `prefer` | `"html"` | Accepted values are `"html"` and `"text"`. The current step converts HTML when HTML is present. |
| `width` | `80` | Soft wrap column. Use `0` to disable wrapping. |
| `preserve_links` | `true` | Render anchors as `text (url)` when possible. |
| `collapse_whitespace` | `true` | Collapse runs of whitespace into single spaces. |
| `keep_tables` | `false` | Render HTML table cells as TSV-like rows instead of linearized text. |

Example:

```json
{
  "name": "html_to_text",
  "args": {
    "width": 0,
    "preserve_links": true,
    "keep_tables": true
  }
}
```

Use `width: 0` when downstream systems should receive unwrapped text. Use
`keep_tables: true` when table-like text should remain row-oriented before a
mapper runs.

## `remove_fields`

`remove_fields` clears message fields, removes headers, removes all
attachments, or clears selected attachment metadata fields.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `paths` | required | Array of field paths to remove or clear. |

Supported paths:

* `subject`
* `text`
* `html`
* `headers.<lowercase-key>`
* `attachments`
* `attachments[*].filename`
* `attachments[*].content_type`
* `attachments[*].size`
* `attachments[*].sha256`
* `attachments[*].blob_key`

Example:

```json
{
  "name": "remove_fields",
  "args": {
    "paths": [
      "headers.received",
      "html",
      "attachments[*].blob_key"
    ]
  }
}
```

Unknown attachment subfields are ignored. Attachment field removal clears safe
metadata fields while preserving attachment identity.

## `replace_values`

`replace_values` assigns literal or templated values to supported message
fields. String values can include `{{ path }}` placeholders.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `set` | required | Object mapping writable paths to literal or templated values. |

Supported writable paths:

* `subject`
* `text`
* `html`
* `headers.<lowercase-key>`

Supported template paths:

* `subject`, `text`, `html`
* `headers.<lowercase-key>`
* `from[<index>].email`, `from[<index>].name`
* `to[<index>].email`, `to[<index>].name`
* `ctx.route_id`, `ctx.project_id`, `ctx.source_type`, `ctx.raw_size_bytes`

Example:

```json
{
  "name": "replace_values",
  "args": {
    "set": {
      "subject": "Order {{ headers.x_order_id }}",
      "headers.x-route": "{{ ctx.route_id }}"
    }
  }
}
```

Unresolved placeholders are left unchanged.

## `strip_attachments_if`

`strip_attachments_if` removes attachments that satisfy all supplied conditions.
Provide at least one condition.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `mime_in` | `[]` | MIME glob patterns such as `"image/*"` or `"application/pdf"`. |
| `max_size_kb` | none | Remove attachments larger than this size. |
| `filename_regex` | none | Remove attachments whose filename matches this regex. Unsafe regexes are rejected. |

Example:

```json
{
  "name": "strip_attachments_if",
  "args": {
    "mime_in": ["application/octet-stream"],
    "max_size_kb": 1024,
    "filename_regex": "(?i)\\.(exe|scr)$"
  }
}
```

When multiple conditions are provided, an attachment must match all of them to
be removed.
