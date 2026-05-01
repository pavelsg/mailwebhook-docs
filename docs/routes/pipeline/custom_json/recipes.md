---
title: Recipes
parent: Custom JSON
grand_parent: Pipeline
nav_order: 2
---

# Recipes
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

These recipes are copy-pasteable `args` objects for `map.custom_json`. Put one
inside the final `map.custom_json` pipeline step.

For operator details, see the [JsonLogic-style DSL] reference.

## Extract the new reply only

Use when: you want the new reply body without quoted history.

Config:

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
    "message_id": { "var": "message.message_id" },
    "subject": { "var": "message.subject" },
    "from": { "var": "message.from[0].email" },
    "reply_text": { "var": "vars.reply_segments.text.reply_content" },
    "has_quoted_content": { "var": "vars.reply_segments.text.has_quoted_content" },
    "quoted_depth": { "var": "vars.reply_segments.text.max_depth" }
  }
}
```

Emits:

```json
{
  "message_id": "abc123@example.com",
  "subject": "Re: Contract update",
  "from": "alice@example.com",
  "reply_text": "Looks good to me.",
  "has_quoted_content": true,
  "quoted_depth": 1
}
```

Notes:

* Use `sources: ["html"]` when downstream needs preserved HTML reply content.
* Use `include_signature: true` if signatures should be available in `segments`.

## Extract all links

Use when: you need a normalized list of links from text or HTML email.

Config:

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "links",
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
    "links": {
      "map": {
        "over": { "var": "vars.links" },
        "as": "link",
        "do": {
          "url": { "var": "link.url" },
          "label": { "var": "link.title" },
          "source_element": { "var": "link.element" }
        }
      }
    }
  }
}
```

Emits:

```json
{
  "subject": "Your report is ready",
  "links": [
    {
      "url": "https://example.com/report",
      "label": "View report",
      "source_element": "a"
    }
  ]
}
```

Notes:

* HTML mode extracts from link-like attributes first, then falls back to text mode when no links are found.
* Markdown links in text mode include both URL and title.

## Extract key/value fields

Use when: emails contain short labels such as `Order ID: AZ-123` or HTML table rows that should become stable webhook fields.

Config:

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
    "customer_email": { "var": "vars.kv.values.customer_email" },
    "fields": {
      "map": {
        "over": { "var": "vars.kv.items" },
        "as": "pair",
        "do": {
          "key": { "var": "pair.normalized_key" },
          "value": { "var": "pair.value" },
          "source": { "var": "pair.source" },
          "line": { "var": "pair.line" }
        }
      }
    }
  }
}
```

Emits:

```json
{
  "subject": "Order received",
  "order_id": "AZ-123",
  "all_order_ids": ["AZ-123"],
  "customer_email": "ada@example.com",
  "fields": [
    {
      "key": "order_id",
      "value": "AZ-123",
      "source": "text",
      "line": 1
    },
    {
      "key": "customer_email",
      "value": "ada@example.com",
      "source": "text",
      "line": 2
    }
  ]
}
```

Notes:

* Use `values.<key>` for the first value and `groups.<key>` when duplicate labels matter.
* Reference extracted fields by normalized key name: `Order ID` becomes `vars.kv.values.order_id`, `Customer Email` becomes `vars.kv.values.customer_email`, and `Ticket-Type` becomes `vars.kv.values.ticket_type`.
* Duplicate labels are available through `groups`, for example `vars.kv.groups.order_id`.
* Keys normalize whitespace and hyphens to underscores.
* HTML keys and values are always returned as text only.
* Separators must be single characters. The defaults are `:` and `=`.

## Extract table cells by row and column

Use when: emails contain order, invoice, inventory, or status tables that should
become stable webhook fields without parsing table text yourself.

Config:

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
    "gadget_total": {
      "var": "vars.tables.tables[0].lookup.by_column.total.gadget"
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
    },
    "source": { "var": "vars.tables.tables[0].source" }
  }
}
```

Emits:

```json
{
  "subject": "Order summary",
  "table_count": 1,
  "widget_qty": "2",
  "gadget_total": "$15.00",
  "row_totals": [
    {
      "item": "widget",
      "total": "$10.00"
    },
    {
      "item": "gadget",
      "total": "$15.00"
    }
  ],
  "source": "html_table"
}
```

Notes:

* Use `lookup.by_row.<row_key>.<column_key>` for direct cell access.
* Use `lookup.by_column.<column_key>.<row_key>` when the column is the primary lookup dimension.
* Use `rows` and `cols` with `map`, `filter`, or `find` for repeated data.
* Header keys are normalized, so `Order Total` becomes `order_total`; duplicate headers get suffixes such as `total_2`.
* Set `include_html_grids: true` for known senders that use ARIA or CSS table-like grids instead of native `<table>` markup.
* Returned cell values are text-only. Raw HTML is not returned.

## Extract repeated DOM cards

Use when: an HTML email uses repeated visual cards or nested layout tables and
you need one structured object per card.

Config:

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
          "max_matches": 16,
          "fields": {
            "outlet": {
              "selector": ".//td[@width='90']//img[1]",
              "value": "attr",
              "attr": "alt"
            },
            "title": {
              "selector": ".//td[@width='90']/following-sibling::td[1]/p[2]/a[1]",
              "value": "text",
              "required": true
            },
            "submit_by": {
              "selector": ".//p[contains(normalize-space(.), 'Submit By:')]",
              "value": "text"
            },
            "description": {
              "selector": ".//tr[2]/td/p[not(.//img)]",
              "value": "text",
              "many": true
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
    "subject": { "var": "message.subject" },
    "opportunity_count": { "var": "vars.opportunities.summary.item_count" },
    "opportunities": {
      "map": {
        "over": { "var": "vars.opportunities.items" },
        "as": "opportunity",
        "do": {
          "outlet": { "var": "opportunity.values.outlet" },
          "title": { "var": "opportunity.values.title" },
          "submit_by": { "var": "opportunity.values.submit_by" },
          "description": { "var": "opportunity.fields.description.values" },
          "pitch_url": { "var": "opportunity.values.pitch_url" }
        }
      }
    }
  }
}
```

Emits:

```json
{
  "subject": "New media requests",
  "opportunity_count": 2,
  "opportunities": [
    {
      "outlet": "Attentus Technologies",
      "title": "CFOs & IT leaders on in-house vs managed IT",
      "submit_by": "Submit By: 4 May 7:17PM CEST (in 3 days)",
      "description": [
        "Looking for insights from CFOs, CIOs, and IT decision-makers..."
      ],
      "pitch_url": "https://example.com/pitch"
    }
  ]
}
```

Notes:

* CSS selectors are usually enough for stable links and attributes. XPath is better when the email needs text anchors such as `Submit By:` or section headings.
* Field XPath selectors must be relative to each card, for example `.//p[contains(normalize-space(.), 'Submit By:')]`.
* Use `fields.<name>.values` when a field has `many: true`; use `values.<name>` for first-value shortcuts.
* Tracking URLs are returned as they appear in the email. Normalize or unwrap them downstream when needed.

## Turn a bullet list into tasks

Use when: an email contains a checklist or next-steps list that should become structured task objects.

Config:

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "lists",
      "expr": {
        "call.extract.bullet_list": {
          "html": { "var": "message.html" },
          "text": { "var": "message.text" },
          "nested": true
        }
      }
    }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "tasks": {
      "map": {
        "over": { "var": "vars.lists[0].bullets" },
        "as": "bullet",
        "do": {
          "vars": [
            {
              "name": "parts",
              "expr": {
                "string.split": {
                  "value": { "var": "bullet.text" },
                  "sep": ":"
                }
              }
            }
          ],
          "output": {
            "title": { "string.trim": { "var": "vars.parts[0]" } },
            "detail": { "string.trim": { "var": "vars.parts[1]" } },
            "children": { "var": "bullet.child.bullets" }
          }
        }
      }
    }
  }
}
```

Emits:

```json
{
  "subject": "Next steps",
  "tasks": [
    {
      "title": "Legal",
      "detail": "Review the updated terms",
      "children": null
    }
  ]
}
```

Notes:

* The recipe expects bullet text in `Title: detail` form.
* Missing `detail` values evaluate to `null`.

## Create a compact alert payload

Use when: you want a small, stable payload for monitoring, ticketing, or notification systems.

Config:

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
      "name": "urgent",
      "expr": {
        "regex.match": {
          "value": { "var": "message.subject" },
          "pattern": "(?i)urgent|failure|down|error"
        }
      }
    }
  ],
  "output": {
    "event_id": { "var": "ctx.event_id" },
    "route_id": { "var": "ctx.route_id" },
    "message_id": { "var": "message.message_id" },
    "from": { "var": "message.from[0].email" },
    "subject": { "var": "message.subject" },
    "priority": { "if": [{ "var": "vars.urgent" }, "high", "normal"] },
    "snippet": { "substr": [{ "var": "vars.text" }, 0, 240] }
  }
}
```

Emits:

```json
{
  "event_id": "6ff49aa1-7050-4ad1-95d9-2711f2ca7e88",
  "route_id": "2f3713bf-88cc-46c6-aaa3-ea9d6e9d20f3",
  "message_id": "abc123@example.com",
  "from": "monitor@example.com",
  "subject": "Service failure",
  "priority": "high",
  "snippet": "The service failed its health check..."
}
```

Notes:

* Adjust the regex pattern to match your own alert vocabulary.
* `snippet` is plain text because the recipe normalizes HTML first.

## Build a Slack-compatible webhook body

Use when: you want a custom Slack incoming-webhook payload instead of the built-in simple chat mapper.

Config:

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
    }
  ],
  "output": {
    "text": {
      "cat": [
        "*",
        { "var": "message.subject" },
        "*\nFrom: ",
        { "var": "message.from[0].email" },
        "\n",
        { "substr": [{ "var": "vars.text" }, 0, 700] }
      ]
    },
    "blocks": [
      {
        "type": "section",
        "text": {
          "type": "mrkdwn",
          "text": {
            "cat": [
              "*",
              { "var": "message.subject" },
              "*\n",
              { "substr": [{ "var": "vars.text" }, 0, 700] }
            ]
          }
        }
      },
      {
        "type": "context",
        "elements": [
          {
            "type": "mrkdwn",
            "text": {
              "cat": [
                "From ",
                { "var": "message.from[0].email" },
                " via MailWebhook"
              ]
            }
          }
        ]
      }
    ]
  }
}
```

Emits:

```json
{
  "text": "*Invoice received*\nFrom: vendor@example.com\nPlease review invoice #123...",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Invoice received*\nPlease review invoice #123..."
      }
    }
  ]
}
```

Notes:

* Send this to an HTTP endpoint configured for Slack incoming webhooks.
* For Slack or Telegram route-pair onboarding, prefer the built-in chat mapper unless you need a custom body.

## Normalize invoice metadata

Use when: you want predictable invoice/vendor fields for downstream accounting or CRM systems.

Config:

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "sender",
      "expr": { "string.lower": { "var": "message.from[0].email" } }
    },
    {
      "name": "vendor_domain",
      "expr": {
        "regex.replace": {
          "value": { "var": "vars.sender" },
          "pattern": "^.*@([^>\\s]+)$",
          "with": "\\1"
        }
      }
    },
    {
      "name": "is_invoice",
      "expr": {
        "regex.match": {
          "value": { "var": "message.subject" },
          "pattern": "(?i)invoice|receipt|payment"
        }
      }
    }
  ],
  "output": {
    "kind": { "if": [{ "var": "vars.is_invoice" }, "invoice", "message"] },
    "vendor": {
      "email": { "var": "vars.sender" },
      "domain": { "var": "vars.vendor_domain" }
    },
    "subject": { "var": "message.subject" },
    "first_attachment": {
      "filename": { "var": "message.attachments[0].filename" },
      "content_type": { "var": "message.attachments[0].content_type" },
      "size": { "var": "message.attachments[0].size" },
      "sha256": { "var": "message.attachments[0].sha256" }
    }
  }
}
```

Emits:

```json
{
  "kind": "invoice",
  "vendor": {
    "email": "billing@vendor.example",
    "domain": "vendor.example"
  },
  "subject": "Invoice 1234",
  "first_attachment": {
    "filename": "invoice-1234.pdf",
    "content_type": "application/pdf",
    "size": 48192,
    "sha256": "..."
  }
}
```

Notes:

* Missing attachments produce `null` values inside `first_attachment`.
* Adjust `is_invoice` to match the terms your vendors use.

[JsonLogic-style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
