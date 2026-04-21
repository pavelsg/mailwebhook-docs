---
title: JsonLogic-Style DSL
parent: Custom JSON
grand_parent: Pipeline
nav_order: 1
---

# JsonLogic-Style DSL
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

The Custom JSON mapper uses a JsonLogic-style expression language. It is not a
complete JsonLogic implementation: MailWebhook supports a focused operator set,
adds MailWebhook-specific helpers under `call.*`, and reserves scoped blocks for
local variables inside templates.

## Expressions

An expression is usually a single-key object:

```json
{ "var": "message.subject" }
```

Expression values can also be plain literals:

```json
"normal"
```

Any value inside a Custom JSON `output` template can be:

* a scalar literal
* an array of template values
* a plain object template
* a single-key expression
* a scoped block whose exact shape is `{ "vars": [...], "output": ... }`

Single-key objects with recognized operator names are evaluated as expressions.
Unknown operator-like keys are rejected. Other objects are treated as plain JSON
templates and their values are evaluated recursively.

## Variable lookup

Use `var` to read from the current scope or the Custom JSON evaluation root:

```json
{ "var": "message.from[0].email" }
```

Path rules:

* Dots traverse object keys.
* `[index]` traverses arrays.
* Missing paths return `null`.
* Bracket syntax supports numeric indexes only.

Lookup order:

1. Current lexical scope head segment first.
2. Iterator aliases such as `item`, a custom `as` alias, `current`, and `accumulator`.
3. Nearest visible `vars.*`, then outer `vars.*`, then root `vars.*`.
4. Root fallback for `message.*`, `ctx.*`, `meta.*`, and top-level `vars.*`.

Examples:

```json
{ "var": "message.subject" }
```

```json
{ "var": "vars.text" }
```

```json
{ "var": "bullet.text" }
```

```json
{ "var": "current.amount" }
```

## Core operators

| Operator | Form | Result |
|----------|------|--------|
| `if` | `{ "if": [condition, thenValue, elseValue] }` | `thenValue` when condition is truthy; otherwise `elseValue` or `null`. |
| `and` | `{ "and": [a, b, ...] }` | First falsey evaluated value, or the last evaluated value. |
| `or` | `{ "or": [a, b, ...] }` | First truthy evaluated value, or the last evaluated value. |
| `!` | `{ "!": expr }` | Boolean negation. |
| `==`, `!=`, `===`, `!==` | `{ "==": [a, b] }` | Equality comparison. Strict and non-strict forms behave the same in `v1`. |
| `>`, `>=`, `<`, `<=` | `{ ">": [a, b] }` | Comparison, or `null` when values cannot be compared. |
| `in` | `{ "in": [needle, haystack] }` | Membership test, or `null` when haystack cannot be searched. |
| `cat` | `{ "cat": [a, b, ...] }` | String concatenation. `null` parts become empty strings. |
| `substr` | `{ "substr": [value, start, length] }` | String slice starting at `start` with optional `length`. |
| `merge` | `{ "merge": [objectA, objectB] }` | Shallow object merge. Later objects win. |

`substr` also accepts object form:

```json
{
  "substr": {
    "value": { "var": "message.subject" },
    "start": 0,
    "end": 80
  }
}
```

## String helpers

| Operator | Form | Result |
|----------|------|--------|
| `string.lower` | `{ "string.lower": expr }` | Lowercase string, or `null`. |
| `string.upper` | `{ "string.upper": expr }` | Uppercase string, or `null`. |
| `string.trim` | `{ "string.trim": expr }` | Trimmed string, or `null`. |
| `string.slice` | `{ "string.slice": { "value": expr, "start": 0, "end": 120 } }` | Slice by start and optional end indexes. |
| `string.split` | `{ "string.split": { "value": expr, "sep": "," } }` | Array of string parts, or `null`. |
| `string.join` | `{ "string.join": { "items": exprArray, "sep": ", " } }` | Joined string, or `null` when `items` is not an array. |
| `string.replace` | `{ "string.replace": { "value": expr, "find": "x", "with": "y", "count": 1 } }` | Exact substring replacement. |

## Regex helpers

| Operator | Form | Result |
|----------|------|--------|
| `regex.match` | `{ "regex.match": { "value": expr, "pattern": "(?i)invoice" } }` | Boolean match result, or `null` for invalid input or invalid pattern. |
| `regex.replace` | `{ "regex.replace": { "value": expr, "pattern": "\\s+", "with": " " } }` | Replaced string, or `null` for invalid input or invalid pattern. |

Regex replacement uses Python-style replacement semantics. Captured groups use
backreferences such as `"\\1"`, not JavaScript-style `$1`.

Invalid regex patterns return `null`. Regex timeouts raise a mapper error and
stop route delivery.

## Array helpers

Array helpers accept object form and, for compact configs, list form. Object form
is clearer and recommended for new configs.

| Operator | Object form | List form | Empty-array result |
|----------|-------------|-----------|--------------------|
| `map` | `{ "map": { "over": exprArray, "as": "item", "do": expr } }` | `{ "map": [exprArray, expr] }` | `[]` |
| `filter` | `{ "filter": { "over": exprArray, "as": "item", "where": expr } }` | `{ "filter": [exprArray, expr] }` | `[]` |
| `find` | `{ "find": { "over": exprArray, "as": "item", "where": expr } }` | `{ "find": [exprArray, expr] }` | `null` |
| `reduce` | `{ "reduce": { "over": exprArray, "do": expr, "start": initial } }` | `{ "reduce": [exprArray, expr, initial] }` | `start`, or `null` when omitted |
| `some` | `{ "some": { "over": exprArray, "as": "item", "where": expr } }` | `{ "some": [exprArray, expr] }` | `false` |
| `all` | `{ "all": { "over": exprArray, "as": "item", "where": expr } }` | `{ "all": [exprArray, expr] }` | `true` |
| `none` | `{ "none": { "over": exprArray, "as": "item", "where": expr } }` | `{ "none": [exprArray, expr] }` | `true` |

Rules:

* `map` returns the evaluated `do` result for each item.
* `filter` returns the original items whose `where` expression is truthy.
* `find` returns the first original item whose `where` expression is truthy.
* `reduce` binds `current` and `accumulator` for each item.
* `some`, `all`, and `none` evaluate `where` as a predicate and short-circuit.
* Non-array `over` values return `null`.
* `as` defaults to `item` when omitted.
* `as` must match `^[A-Za-z_][A-Za-z0-9_]*$`.
* `as` must not shadow reserved scope names: `message`, `ctx`, `meta`, `vars`, `current`, `accumulator`.

Example:

```json
{
  "map": {
    "over": { "var": "vars.lists[0].bullets" },
    "as": "bullet",
    "do": { "var": "bullet.text" }
  }
}
```

Reducer example:

```json
{
  "reduce": {
    "over": [{ "amount": 10 }, { "amount": 15 }],
    "start": 0,
    "do": {
      "cat": [
        { "var": "accumulator" },
        { "var": "current.amount" }
      ]
    }
  }
}
```

## Scoped blocks

Scoped blocks let you define local `vars` anywhere a template value is allowed:

```json
{
  "vars": [
    { "name": "text", "expr": { "var": "bullet.text" } },
    {
      "name": "parts",
      "expr": {
        "string.split": {
          "value": { "var": "vars.text" },
          "sep": ":"
        }
      }
    }
  ],
  "output": { "string.trim": { "var": "vars.parts[1]" } }
}
```

Rules:

* Detection is exact: an object with exactly `vars` and `output` is evaluated as a scoped block.
* Any extra key keeps the object a plain JSON template.
* Block `vars` use the same `{ "name": "...", "expr": ... }` contract as top-level vars.
* Block `vars` evaluate top-to-bottom.
* Later block vars can read earlier block vars through `vars.<name>`.
* Inner `vars.foo` shadows outer `vars.foo`.

If you need to emit a literal object with keys named `vars` and `output`, build
it indirectly:

```json
{
  "merge": [
    { "vars": [1, 2, 3] },
    { "output": "value" }
  ]
}
```

## Function calls

Function calls expose deterministic MailWebhook helpers. They can be written in
generic form:

```json
{
  "call": {
    "fn": "extract.urls",
    "args": {
      "text": "See https://example.com"
    }
  }
}
```

Or prefixed form:

```json
{
  "call.extract.urls": {
    "text": "See https://example.com"
  }
}
```

Allowed helpers:

* `transform.html_to_text`
* `extract.urls`
* `extract.bullet_list`
* `extract.reply_segments`
* `extract.key_value_pairs`

Unknown helper names are rejected.

### `call.transform.html_to_text`

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `html` | none | HTML string to convert. |
| `text` | none | Plain text fallback. When present and truthy, it is returned directly. |

Returns a string or `null`.

### `call.extract.urls`

Extracts URLs from text or HTML. In HTML mode, it reads `href`, `src`,
`action`, and `formaction` attributes from link, image, form, script, iframe,
and source-like tags. In text mode, Markdown links include both URL and title.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `mode` | `"html"` unless only `text` is provided | `"html"` or `"text"`. |
| `html` | `message.html` | HTML source. |
| `text` | `message.text` | Text source. |
| `deduplicate` | `false` | Drop duplicate `(url, title, element)` entries. |

Returns an array of objects:

```json
[
  {
    "url": "https://example.com",
    "title": "Optional title",
    "element": "a"
  }
]
```

### `call.extract.bullet_list`

Extracts bullet and numbered lists from HTML or text content. HTML mode falls
back to text mode when no lists are found.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `mode` | `"html"` | `"html"` or `"text"`. |
| `html` | `message.html` | HTML source. |
| `text` | `message.text` | Text source. |
| `nested` | `false` | Preserve nested list structure. |
| `sanitize` | `true` | Remove empty lines in text mode. |
| `numbered` | `true` | Include numbered lists. |

Returns an array of list objects:

```json
[
  {
    "title": "Optional heading",
    "bullets": [
      {
        "text": "Item text",
        "child": {
          "title": "Item text",
          "bullets": []
        }
      }
    ]
  }
]
```

### `call.extract.reply_segments`

Splits reply content from quoted content, forwarded content, and signatures.
Use `text.reply_content` or `html.reply_content` for the common "just the new
reply" case.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `text` | `message.text` | Text source. |
| `html` | `message.html` | HTML source. |
| `sources` | `["text", "html"]` | Sources to analyze. |
| `include_signature` | `false` | Include detected signatures as segments. |
| `split_quoted_by_depth` | `false` | Split quoted content by quote depth. |
| `min_confidence` | `0.0` | Filter segments below this confidence. |
| `max_segments_per_source` | runtime capped | Soft per-call segment cap. |
| `max_input_chars_per_source` | runtime capped | Soft per-call input-size cap. |

Returns an object with source-specific results:

```json
{
  "text": {
    "format": "text/plain",
    "reply_content": "Thanks for the update.",
    "has_quoted_content": true,
    "max_depth": 1,
    "detected_vendors": [],
    "segments": [
      {
        "kind": "reply_content",
        "depth": 0,
        "content": "Thanks for the update.",
        "confidence": 1.0,
        "detectors": ["plain.leading_unquoted"],
        "vendor": null
      }
    ]
  },
  "html": null
}
```

Useful fields:

* `text.reply_content` and `html.reply_content`
* `text.has_quoted_content` and `html.has_quoted_content`
* `segments[*].kind`
* `segments[*].depth`
* `segments[*].content`

### `call.extract.key_value_pairs`

Extracts conservative key/value pairs from message body text or HTML. Use this
helper when downstream config needs direct lookup fields such as
`vars.kv.values.order_id`, while still retaining ordered `items` for
`map`/`filter` workflows.

Arguments:

| Arg | Default | Description |
|-----|---------|-------------|
| `mode` | `"auto"` | `"auto"`, `"html"`, or `"text"`. Auto prefers HTML when present and falls back to text when HTML yields no pairs. |
| `html` | `message.html` | HTML source. |
| `text` | `message.text` | Text source. |
| `separators` | `[":", "="]` | Ordered single-character separators for text candidates. |
| `max_key_words` | `5` | Maximum accepted key word count before a candidate is treated as prose. |
| `allow_prose` | `false` | Allow small prose-like keys that are normally rejected. |
| `allow_html_adjacent_blocks` | `false` | Allow generic adjacent HTML block pairing. Disabled by default because it can be noisy. |

Returns:

```json
{
  "items": [
    {
      "key": "Order ID",
      "normalized_key": "order_id",
      "value": "AZ-123",
      "separator": ":",
      "source": "text",
      "line": 1
    }
  ],
  "values": {
    "order_id": "AZ-123"
  },
  "groups": {
    "order_id": ["AZ-123"]
  }
}
```

Useful fields:

* `items[*].key`
* `items[*].normalized_key`
* `items[*].value`
* `items[*].separator`
* `items[*].source`
* `items[*].line`
* `values.<normalized_key>` for the first value
* `groups.<normalized_key>` for all values in encounter order

Referencing normalized keys:

```text
Order ID: AZ-123
Order ID: AZ-124
Customer Email: ada@example.com
Ticket-Type: billing
```

The labels above produce these lookup keys:

| Label | Normalized key | First value | All values |
|-------|----------------|-------------|------------|
| `Order ID` | `order_id` | `vars.kv.values.order_id` | `vars.kv.groups.order_id` |
| `Customer Email` | `customer_email` | `vars.kv.values.customer_email` | `vars.kv.groups.customer_email` |
| `Ticket-Type` | `ticket_type` | `vars.kv.values.ticket_type` | `vars.kv.groups.ticket_type` |

Use `values` when the first value is enough, and `groups` when duplicate labels
should be preserved:

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "kv",
      "expr": {
        "call.extract.key_value_pairs": {
          "mode": "text"
        }
      }
    }
  ],
  "output": {
    "first_order_id": { "var": "vars.kv.values.order_id" },
    "all_order_ids": { "var": "vars.kv.groups.order_id" },
    "customer_email": { "var": "vars.kv.values.customer_email" },
    "ticket_type": { "var": "vars.kv.values.ticket_type" }
  }
}
```

Text extraction rules:

* Keys must start with an alphabetic character.
* Keys may contain letters, digits, whitespace, and hyphen only.
* Hyphens and whitespace in keys normalize to `_`, so `Ticket-Type` becomes `ticket_type`.
* Separators must be single non-whitespace characters.
* Candidate lines ending with `?` or `!` are ignored.
* Values preserve internal whitespace after edge trimming.
* One trailing `.` is trimmed from values.

HTML extraction rules:

* HTML keys and values are always returned as text only.
* Structured extraction covers common table rows, definition lists, labels, and bold or strong key hints.
* HTML-to-text fallback handles separator lines after structural extraction.
* Generic adjacent block pairing requires `allow_html_adjacent_blocks: true`.

## Runtime limits and errors

The DSL is deterministic and does not perform network IO. Helper calls are
limited to MailWebhook's registered helpers.

Runtime limits:

* Maximum expression depth: `50`
* Maximum nodes evaluated: `10,000`
* Regex timeout: about `50 ms` per operation
* `transform.html_to_text`, `extract.urls`, `extract.bullet_list`, and `extract.key_value_pairs`: about `200 ms` per helper call
* `extract.reply_segments`: bounded by reply-segmentation runtime guardrails

Most operator-level failures return `null`. Hard guard failures raise a mapper
error and stop route delivery.

## Unsupported features

The DSL intentionally does not support:

* arbitrary functions
* network calls
* mutation
* loops outside array helpers

[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
