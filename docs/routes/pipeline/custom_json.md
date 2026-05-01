---
title: Custom JSON
parent: Pipeline
nav_order: 5
---

# Custom JSON
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

`map.custom_json` is the terminal mapper for routes that need a webhook payload
shape controlled by your own JSON configuration. The mapper document is
validated by JSON Schema, and values inside `vars` and `output` can use the
[JsonLogic-style DSL] for lookups, conditionals, string helpers, array helpers,
regex helpers, and MailWebhook extraction helpers.

Use this page for the mapper contract. Use the [JsonLogic-style DSL] reference
for operator syntax and expression semantics, and use [Recipes] for copy-paste
payload patterns.

## How to use it

Put `map.custom_json` as the final step in a route pipeline:

```json
{
  "pipeline": {
    "steps": [
      {
        "name": "map.custom_json",
        "args": {
          "version": "v1",
          "vars": [
            {
              "name": "plain_text",
              "expr": {
                "call.transform.html_to_text": {
                  "html": { "var": "message.html" },
                  "text": { "var": "message.text" }
                }
              }
            }
          ],
          "output": {
            "id": { "var": "message.message_id" },
            "subject": { "var": "message.subject" },
            "from": { "var": "message.from[0].email" },
            "text": { "var": "vars.plain_text" }
          }
        }
      }
    ]
  }
}
```

Pipeline rules still apply:

* `pipeline.steps` must contain at least one step.
* Exactly one `map.*` step must exist.
* The `map.*` step must be the final step.

## Mapper document

The `args` object passed to `map.custom_json` has this shape:

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "text",
      "expr": { "var": "message.text" }
    }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "text": { "var": "vars.text" }
  },
  "meta": {
    "source": "accounts-payable"
  }
}
```

| Field | Required | Description |
|-------|----------|-------------|
| `version` | yes | Must be `v1`. |
| `vars` | no | Ordered array of temporary values. Each entry is `{ "name": "...", "expr": ... }`. Later vars can read earlier vars through `vars.<name>`. |
| `output` | yes | JSON template that becomes the emitted webhook body. Any value can be a literal, array, object template, expression, or scoped block. |
| `meta` | no | Static object copied into the evaluation root as `meta`. |

`vars` names must match `^[A-Za-z_][A-Za-z0-9_]*$`. Duplicate names are rejected.

## Evaluation root

Expressions can read these root objects with `{"var": "..."}`.

| Root | Paths | Description |
|------|-------|-------------|
| `message` | `message_id`, `message_id_is_synthetic`, `subject`, `date`, `received_at` | Message identifiers and timestamps. |
| `message` | `from`, `to`, `reply_to`, `cc`, `bcc` | Address arrays. Each item has `email` and optional `name`. Use `message.from[0].email` for the first sender. |
| `message` | `headers`, `headers_multi` | Lowercase header map, plus duplicate-preserving header values when repeated headers exist. |
| `message` | `text`, `html` | Parsed text and HTML bodies. |
| `message` | `attachments` | Attachment metadata array with `id`, `filename`, `content_type`, `size`, `blob_key`, `sha256`, `is_inline`, and optional `content_id`. |
| `ctx` | `event_id`, `project_id`, `route_id`, `source_type`, `raw_size_bytes`, `now` | Runtime context. `now` is RFC3339 UTC. |
| `meta` | any key you provide | Static metadata from the mapper config. |
| `vars` | computed var names | Top-level vars evaluated before `output`. |

`message.from_` remains accepted for existing configs, but new configs should use
the customer-facing `message.from` path.

## Output behavior

The evaluated `output` value is sent as the webhook body for the route. The shape
is intentionally free-form and can be an object, array, string, number, boolean,
or `null`. If the top-level `output` evaluates to `null`, MailWebhook emits an
empty object.

Attachments are not inlined into the HTTP body. Use `message.attachments` for
metadata and the attachment download API for file content.

## Built-in helpers

`map.custom_json` supports these helper calls in addition to the core DSL
operators:

| Helper | Use it for |
|--------|------------|
| `call.transform.html_to_text` | Normalize HTML email into plain text. |
| `call.extract.urls` | Extract URLs from HTML attributes, HTML link text, plain text, or Markdown links. |
| `call.extract.bullet_list` | Extract bullet and numbered lists from HTML or text. |
| `call.extract.reply_segments` | Split replies from quoted content, forwarded content, and signatures. |
| `call.extract.key_value_pairs` | Extract conservative key/value fields from text or HTML. |
| `call.extract.tables` | Extract structured tables from text, native HTML tables, or opt-in HTML grids. |
| `call.extract.dom` | Extract scalar values or repeated structured records from stable HTML with CSS or XPath selectors. |

Helpers can be called in either form:

```json
{
  "call": {
    "fn": "extract.urls",
    "args": {
      "text": { "var": "message.text" }
    }
  }
}
```

```json
{
  "call.extract.urls": {
    "text": { "var": "message.text" }
  }
}
```

See [function calls] in the DSL reference for helper arguments and output shapes.

## Examples

### Vendor priority payload

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "text",
      "expr": {
        "call.transform.html_to_text": {
          "html": { "var": "message.html" },
          "text": { "var": "message.text" }
        }
      }
    },
    {
      "name": "is_vendor",
      "expr": {
        "regex.match": {
          "value": { "var": "message.from[0].email" },
          "pattern": "@vendor\\.com$"
        }
      }
    }
  ],
  "output": {
    "id": { "var": "message.message_id" },
    "source": { "var": "ctx.source_type" },
    "priority": { "if": [{ "var": "vars.is_vendor" }, "high", "normal"] },
    "snippet": { "substr": [{ "var": "vars.text" }, 0, 200] }
  }
}
```

### URL extraction

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "urls",
      "expr": {
        "call.extract.urls": {
          "html": { "var": "message.html" },
          "text": { "var": "message.text" },
          "deduplicate": true
        }
      }
    }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "first_url": { "var": "vars.urls[0].url" },
    "first_url_title": { "var": "vars.urls[0].title" }
  }
}
```

### Reply body only

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "reply_segments",
      "expr": {
        "call.extract.reply_segments": {
          "sources": ["text"],
          "split_quoted_by_depth": true
        }
      }
    }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "reply_text": { "var": "vars.reply_segments.text.reply_content" },
    "has_quoted_content": { "var": "vars.reply_segments.text.has_quoted_content" }
  }
}
```

### Key/value extraction

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "kv",
      "expr": {
        "call.extract.key_value_pairs": {
          "mode": "auto"
        }
      }
    }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "order_id": { "var": "vars.kv.values.order_id" },
    "all_order_ids": { "var": "vars.kv.groups.order_id" },
    "fields": {
      "map": {
        "over": { "var": "vars.kv.items" },
        "as": "pair",
        "do": {
          "key": { "var": "pair.normalized_key" },
          "value": { "var": "pair.value" },
          "source": { "var": "pair.source" }
        }
      }
    }
  }
}
```

### Table extraction

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "tables",
      "expr": {
        "call.extract.tables": {
          "mode": "auto"
        }
      }
    }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "table_count": { "var": "vars.tables.summary.table_count" },
    "widget_qty": {
      "var": "vars.tables.tables[0].lookup.by_row.widget.qty"
    },
    "row_totals": {
      "map": {
        "over": { "var": "vars.tables.tables[0].rows" },
        "as": "row",
        "do": {
          "item": { "var": "row.lookup_key" },
          "total": { "var": "row.values.total" }
        }
      }
    }
  }
}
```

`extract.tables` returns a `tables` array plus `summary.table_count`. Each table
includes normalized row and column headers, row-oriented and column-oriented
lookup maps, iterable `rows` and `cols`, and the rectangular text-only `matrix`.

### DOM extraction

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "opportunities",
      "expr": {
        "call.extract.dom": {
          "selector_type": "xpath",
          "selector": "//h1[contains(normalize-space(.), 'Matches Based')]/following::table[1]//tr[td[contains(@style, 'padding:20px 0;')]/table//p[contains(normalize-space(.), 'Submit By:')]]",
          "value": "text",
          "fields": {
            "outlet": {
              "selector": ".//td[@width='90']//img[1]",
              "value": "attr",
              "attr": "alt"
            },
            "title": {
              "selector": ".//td[@width='90']/following-sibling::td[1]/p[2]/a[1]",
              "value": "text"
            },
            "submit_by": {
              "selector": ".//p[contains(normalize-space(.), 'Submit By:')]",
              "value": "text"
            },
            "pitch_url": {
              "selector": ".//a[contains(normalize-space(.), 'Learn More') and contains(normalize-space(.), 'Pitch')]",
              "value": "attr",
              "attr": "href"
            }
          }
        }
      }
    }
  ],
  "output": {
    "opportunity_count": { "var": "vars.opportunities.summary.item_count" },
    "opportunities": {
      "map": {
        "over": { "var": "vars.opportunities.items" },
        "as": "opportunity",
        "do": { "var": "opportunity.values" }
      }
    }
  }
}
```

`extract.dom` is useful for stable sender templates where the fields are stored
in repeated HTML cards or deeply nested layout tables. CSS selectors are best
for simple stable attributes and links. XPath selectors are better for
label-relative or text-anchored email layouts.

## Validation and limits

The mapper config is validated when the route is saved and again before mapper
execution. Validation rejects malformed `vars`, unknown operators, unsupported
helper names, invalid array-helper aliases, and malformed scoped blocks.

Runtime limits:

* Max expression depth: `50`
* Max nodes evaluated: `10,000` per mapper
* Regex timeout: about `50 ms` per regex operation
* `transform.html_to_text`, `extract.urls`, `extract.bullet_list`, `extract.key_value_pairs`, `extract.tables`, and `extract.dom`: about `200 ms` per helper call
* `extract.reply_segments`: bounded by dedicated reply-segmentation runtime guardrails

Most operator-level failures return `null`. Hard guard failures, such as timeout
or depth/node-limit failures, raise a mapper error and stop route delivery.

## Schemas

The canonical mapper schema is published at
[`custom_json_mapper@1`](https://schemas.mailwebhook.dev/transform/custom_json_mapper@1).
The emitted output is intentionally permissive and is validated by
[`custom_json_output@1`](https://schemas.mailwebhook.dev/transform/custom_json_output@1).

[JsonLogic-style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
[Recipes]: {% link docs/routes/pipeline/custom_json/recipes.md %}
[function calls]: {% link docs/routes/pipeline/custom_json/dsl.md %}#function-calls
