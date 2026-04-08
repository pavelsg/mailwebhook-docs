---
title: >
    Custom JSON
parent: Pipeline
nav_order: 5
---

# Pipeline
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

* **JSON Schema 2020-12** defines the top-level mapper document shape.
* **JsonLogic-style expressions** define the evaluator behavior inside the mapper.

Below is the normative spec for `map.custom_json` as currently implemented in
`v1`, including the nested scoped-block behavior added by the `JSONEXP`
feature. For the authoritative [schema definition](#schema-definition), see
below. The mapper output remains free-form and is validated only by the
permissive [custom_json_output@1](#custom-json-output-validation).

## How to instantiate

Using Rule & Pipeline (JSON):

```json
{
  "pipeline": {
    "steps": [
      {
        "name": "map.custom_json",
        "args": {
          "version": "v1",
          "vars": [
            // ordered list of variable definitions
          ],
          "output": {
            // desired shape for emitted JSON
          },
          "meta": {
            // optional static metadata injected into the eval root
          }
        }
      }
    ]
  }
}
```

**Inputs available at evaluation time**

Use `{"var": "message.subject"}` etc. The evaluator provides:

* `message` with parsed email fields such as `message_id`, `subject`, `from`, `to`, `text`, `html`, `headers`, and `attachments`.
* `ctx` with route/runtime metadata such as `project_id`, `route_id`, `source_type`, and `now` (RFC3339).
* `meta` with any static mapper-provided metadata.
* `vars` with the computed top-level mapper vars.

## Evaluation model

Top-level evaluation proceeds in two phases:

* Evaluate `vars` entries sequentially, top to bottom. Later vars can reference earlier ones via `{"var":"vars.some_name"}`.
* Evaluate `output` as a JSON template by recursively evaluating any expressions it contains.

Any JSON template value may be:

* a literal scalar
* an array of template values
* an object template
* a JsonLogic-style single-key expression
* a nested scoped block whose exact shape is `{ "vars": [...], "output": ... }`

Only array form is supported for `vars`; each entry must include `name` and `expr`.

## Variable lookup order

`var` resolves names in lexical-scope-first order:

1. Current lexical scope head segment first.
2. Iterator aliases such as `item`, a custom `as` alias, `current`, and `accumulator`.
3. Nearest visible `vars.*`, then outer `vars.*`, then root `vars.*`.
4. Root fallback for `message.*`, `ctx.*`, `meta.*`, and top-level `vars.*`.

Examples:

* `{"var":"message.subject"}`
* `{"var":"vars.text"}`
* `{"var":"bullet.text"}`
* `{"var":"current.text"}`

## Supported JsonLogic operators

Use standard JsonLogic-style semantics for these:

* Control and logic: `if`, `and`, `or`, `!`
* Comparators: `==`, `!=`, `===`, `!==`, `>`, `>=`, `<`, `<=`, `in`
* Data access: `var` with dotted and bracket paths
* Strings and arrays:
  * `cat`
  * `substr`
  * `map`, `filter`, `find`, `reduce`, `some`, `all`, `none`
* Objects:
  * `merge`

## Custom extensions

All extensions are prefixed to avoid collisions and are implemented by the
custom evaluator. Unknown operators or unsupported `call.fn` values are rejected
by schema validation or evaluator semantic checks.

### Path helpers

* Arrays support `[index]`. Dots split object keys.
* Array helpers resolve local aliases before the root context, so `bullet.text`
  reads from the current scoped item when `as: "bullet"` is active.
* Custom `as` aliases must match `^[A-Za-z_][A-Za-z0-9_]*$` and must not shadow
  reserved scope names: `message`, `ctx`, `meta`, `vars`, `current`,
  `accumulator`.

### Scoped blocks

`v1` reserves the exact nested object shape below as a scoped expression block:

```json
{
  "vars": [
    { "name": "text", "expr": { "var": "bullet.text" } }
  ],
  "output": { "var": "vars.text" }
}
```

Scoped-block rules:

* Detection is exact: an object with exactly the keys `vars` and `output` is evaluated as a block.
* Any extra key keeps the object a plain JSON template.
* Nested `vars` entries follow the same `{name, expr}` contract as top-level `vars`.
* Nested `vars` evaluate top-to-bottom.
* Later nested vars can read earlier nested vars through `vars.<name>`.
* `output` is the value returned by the block.
* Inner `vars.foo` shadows outer `vars.foo`.

If you need to emit a literal object whose keys are `vars` and `output`, build
it indirectly, for example:

```json
{
  "merge": [
    { "vars": [1, 2, 3] },
    { "output": "value" }
  ]
}
```

### String helpers

* `string.lower`: `{"string.lower": expr}`
* `string.upper`: `{"string.upper": expr}`
* `string.trim`: `{"string.trim": expr}`
* `string.slice`: `{"string.slice": {"value": expr, "start": 0, "end": 120}}`
* `string.split`: `{"string.split": {"value": expr, "sep": ","}}`
* `string.join`: `{"string.join": {"items": exprArray, "sep": ", "}}`
* `string.replace`: exact substring replace

  * `{"string.replace":{"value":expr,"find":"x","with":"y","count":1}}`

### Regex helpers

* `regex.match`

  * Input: `{"regex.match":{"value": expr, "pattern":"(?i)invoice|receipt"}}`
  * Output: boolean
* `regex.replace`

  * `{"regex.replace":{"value":expr,"pattern":"\\s+","with":" "}}`

Regex replacement uses Python-style replacement semantics. Captured groups use
backreferences such as `"\\1"`, not JavaScript-style `$1`.

On invalid pattern or timeout, the regex operator returns `null`.

### Function calls to internal library


{: .note }
Allowed built-in helpers are currently `transform.html_to_text`,
`extract.urls`, and `extract.bullet_list`.

Function calls can be encoded either as the generic `call` form or as prefixed
operators under `call.*`.

Generic form:

```json
{ "call": { "fn": "extract.urls", "args": { "text": "See https://example.com" } } }
```

Prefixed form:

```json
{ "call.extract.urls": { "text": "See https://example.com" } }
```

* `call.transform.html_to_text`

  * Args: `{"html": string}` or `{"html": string, "text": string}`
  * Returns: string
* `call.extract.urls`

  Extracts URLs from the text or HTML. In HTML mode, extracts from href/src attributes and text content from the following tags: `<a>`, `<area>`, `<img>`, `<link>`, `<form>`, `<button>`, `<input>`, `<script>`, `<iframe>`, and `<source>`.

  In text mode, if Markdown links are found (e.g., `[title](url)`), extracts both URL and title.

  * Args are optional. Supported args:
    * `mode`: `"html"` or `"text"`.
    * `html`: HTML string source (default: `message.html`).
    * `text`: text string source (default: `message.text`).
    * `deduplicate`: boolean to drop duplicate `(url,title,element)` entries (default: `false`).
  * Mode selection:
    * If `mode` is provided and valid, honor it.
    * If only `text` is provided (no `html`), default to text mode.
    * Otherwise default to HTML mode; if HTML parsing errors or yields zero results, fall back to text mode.
  * Output item shape:
    * `{ "url": "<string>", "title": "<optional string>", "element": "<optional string>" }`

* `call.extract.bullet_list`

  Extracts bullet/numbered lists from HTML or text content. Tries to guess title from preceding text.

  * Args are optional. Supported keys:
    * `mode`: `"html"` or `"text"` (default: `"html"`).
    * `html`: HTML string source (default: `message.html`).
    * `text`: text string source (default: `message.text`).
    * `nested`: boolean to enable nested list extraction (default: `false`).
    * `sanitize`: boolean to remove empty lines in text mode (default: `true`).
    * `numbered`: boolean to include numbered lists (`<ol>` or `1.`/`1)`) (default: `true`).
  * Source selection:
    * In HTML mode: use `args.html` if provided, else `message.html`. If extraction yields zero results, fall back to text mode.
    * In text mode: use `args.text` if provided, else `message.text`.
  * Output shape:
    * List objects: `{"title": "<optional>", "bullets": [<bullet>, ...]}`.
    * Bullet objects: `{"text": "<item>", "child": <list|null>}` where `child` (if present) uses the same list object shape.

Results are plain JSON and can be traversed with `var`. If you assign results to
a var, reference them as `{"var":"vars.urls[0].url"}` or
`{"var":"vars.lists[0].bullets[0].text"}`.

Within array helpers, scoped aliases can also be traversed with `var`, for example:

```json
{
  "map": {
    "over": { "var": "vars.lists[0].bullets" },
    "as": "bullet",
    "do": { "var": "bullet.text" }
  }
}
```

## Examples

### Conditionals with text extraction

```json
{
  "version": "v1",
  "vars": [
    { "name": "text", "expr": { "call.transform.html_to_text": { "html": { "var": "message.html" } } } },
    { "name": "is_vendor", "expr": { "regex.match": { "value": { "var":"message.from[0].email" }, "pattern": "@vendor\\.com$" } } }
  ],
  "output": {
    "id": { "var": "message.message_id" },
    "source": { "var": "ctx.source_type" },
    "priority": { "if": [ { "var":"vars.is_vendor" }, "high", "normal" ] },
    "snippet": { "substr": [ { "var":"vars.text" }, 0, 200 ] }
  }
}
```

### URL extraction and formatting

```json
{
  "version": "v1",
  "vars": [
    { "name": "text", "expr": { "call.transform.html_to_text": { "html": { "var": "message.html" } } } },
    { "name": "urls", "expr": { "call.extract.urls": { "text": { "var": "vars.text" } } } }
  ],
  "output": {
    "subject": { "var": "message.subject" },
    "first_url": { "var": "vars.urls[0].url" },
    "domain": {
      "regex.replace": {
        "value": { "var":"vars.urls[0].url" },
        "pattern": "^https?://([^/]+)/.*$",
        "with": "\\1"
      }
    }
  }
}
```

### Slack-like text with joins

```json
{
  "version": "v1",
  "vars": [
    { "name": "text", "expr": { "call.transform.html_to_text": { "html": { "var":"message.html" } } } },
    { "name": "lines", "expr": { "string.split": { "value": { "var":"vars.text" }, "sep": "\n" } } }
  ],
  "output": {
    "text": {
      "cat": [
        "*", { "var":"message.subject" }, "*", "\n",
        "From: ", { "var":"message.from[0].email" }, "\n",
        { "string.join": { "items": { "var":"vars.lines" }, "sep": " " } }
      ]
    }
  }
}
```

### Nested scoped block inside `map`

```json
{
  "version": "v1",
  "vars": [
    {
      "name": "lists",
      "expr": {
        "call.extract.bullet_list": {
          "mode": "html",
          "html": { "var": "message.html" }
        }
      }
    }
  ],
  "output": {
    "next_steps": {
      "map": {
        "over": { "var": "vars.lists[0].bullets" },
        "as": "bullet",
        "do": {
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
            },
            {
              "name": "task",
              "expr": {
                "string.trim": { "var": "vars.parts[1]" }
              }
            }
          ],
          "output": { "var": "vars.task" }
        }
      }
    }
  }
}
```

## Runtime, limits, and error policy

* Deterministic, pure evaluation. No IO except registered `call.*` helpers.
* Guards:

  * Max expression depth: 50
  * Max nodes evaluated: 10k per mapper
  * Regex timeout: 50 ms per op (timeout raises a mapper error)
  * Function call timeout: about 200 ms per call
* Operator-level failures generally return `null`. Critical guard failures raise a mapper runtime error and stop execution.
* Route-save validation also enforces semantic guardrails not expressed directly in the raw JSON Schema, including invalid or reserved `as` aliases and malformed nested scoped blocks.

## Schema definition

The published JSON Schema defines the top-level mapper document. Nested scoped
blocks are an additional reserved evaluator form enforced semantically by, so they are not fully expressible in the raw
top-level schema alone.

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.mailwebhook.dev/transform/custom_json_mapper@1",
  "title": "mailwebhook custom_json mapper config v1",
  "description": "Validates the JsonLogic-style configuration passed to map.custom_json.",
  "type": "object",
  "required": ["version", "output"],
  "additionalProperties": false,
  "properties": {
    "version": { "const": "v1" },
    "vars": {
      "type": "array",
      "default": [],
      "items": {
        "type": "object",
        "required": ["name", "expr"],
        "additionalProperties": false,
        "properties": {
          "name": { "type": "string", "pattern": "^[A-Za-z_][A-Za-z0-9_]*$" },
          "expr": { "$ref": "#/$defs/expr" },
          "description": { "type": "string" }
        }
      }
    },
    "output": { "$ref": "#/$defs/jsonTemplate" },
    "meta": { "type": "object" }
  },
  "$defs": {
    "jsonTemplate": {
      "description": "Arbitrary JSON where any value may be a literal or an expression",
      "anyOf": [
        { "$ref": "#/$defs/expr" },
        { "type": ["string", "number", "boolean", "null"] },
        {
          "type": "array",
          "items": { "$ref": "#/$defs/jsonTemplate" }
        },
        {
          "type": "object",
          "additionalProperties": { "$ref": "#/$defs/jsonTemplate" }
        }
      ]
    },
    "expr": {
      "description": "JsonLogic-style expression or literal",
      "oneOf": [
        { "type": ["string", "number", "boolean", "null"] },
        {
          "type": "array",
          "items": { "$ref": "#/$defs/expr" }
        },
        {
          "type": "object",
          "minProperties": 1,
          "maxProperties": 1,
          "patternProperties": {
            "^(var|if|and|or|!|==|!=|===|!==|>|>=|<|<=|in|cat|substr|merge|string\\.lower|string\\.upper|string\\.trim|string\\.slice|string\\.split|string\\.join|string\\.replace|regex\\.match|regex\\.replace|map|filter|find|reduce|some|all|none)$": {
              "description": "Supported operators for map.custom_json",
              "type": ["array", "object", "string", "number", "boolean", "null"]
            },
            "^call$": {
              "type": "object",
              "required": ["fn"],
              "additionalProperties": false,
              "properties": {
                "fn": {
                  "type": "string",
                  "enum": [
                    "transform.html_to_text",
                    "extract.urls",
                    "extract.bullet_list"
                  ]
                },
                "args": { "type": "object" }
              }
            },
            "^call\\.(transform\\.html_to_text|extract\\.urls|extract\\.bullet_list)$": {
              "type": "object"
            }
          },
          "additionalProperties": false
        }
      ]
    }
  }
}
```

## Custom JSON output validation

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schemas.mailwebhook.dev/transform/custom_json_output@1",
  "title": "mailwebhook.custom_json output v1",
  "description": "Output envelope for map.custom_json. Intentionally permissive (user-defined shape).",
  "type": ["object", "array", "string", "number", "boolean", "null"],
  "additionalProperties": true
}
```
