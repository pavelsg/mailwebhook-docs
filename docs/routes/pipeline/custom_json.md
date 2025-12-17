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


* **JSON Schema 2020-12** to define the mapper document shape and validate configs.
* **JsonLogic 2.0** for expressions, conditionals, and array ops inside a mapper.

Below is a **normative spec** for `map.custom_json` using those standards, plus the minimal extensions.
For the authoritative [schema definition](#schema-definition) see below.
The mapper *output* is free-form and validated only by the permissive [custom_json_output@1](#custom-json-output-validation).

## How to instantiate

Using Rule & Pipeline (JSON):

```json
{
  "rule": {
    ...
  },
  "pipeline": {
    "steps": [
      {
        "name": "map.custom_json_mapper",
        "args": {
          mapper code here
        }
      }
    ]
  }
}
```

**Inputs available at evaluation time**

Use `{"var": "message.subject"}` etc. The evaluator provides:

* `message` object with parsed email fields: `message_id`, `subject`, `from`, `to`, `text`, `html`, `headers`, `attachments`.
* `ctx` object: `project_id`, `route_id`, `source_type`, `now` (RFC3339).
* `meta` object: any pipeline-provided metadata.

## Evaluation rules

* Evaluate `vars` entries sequentially, top to bottom. Each entry has `name` and `expr`. Later vars can reference earlier ones using `{"var":"vars.some_name"}`.
* Build `output` by recursively evaluating any JsonLogic expressions found in the template.
* The environment seen by `var` includes `message`, `ctx`, `meta`, and `vars`.
* Only array form is supported for `vars`; each entry must include `name` and `expr`.
* Ordering-sensitive behaviors (like array traversal) and runtime limits (depth, nodes, regex timeouts) are enforced by the evaluator.
* Errors inside expressions yield `null` unless noted below. Hard failures abort with a mapper runtime error.

## Supported JsonLogic operators

Use standard JsonLogic 2.0 semantics for these:

* Control and logic: `if`, `and`, `or`, `!`
* Comparators: `==`, `!=`, `===`, `!==`, `>`, `>=`, `<`, `<=`, `in`
* Data access: `var` with dotted and bracket paths. Examples:

  * `{"var":"message.subject"}`
  * `{"var":"message.attachments[0].filename"}`
* Strings and arrays:

  * `cat` (concatenate)
  * `substr` (start, length)
  * `map`, `filter`, `reduce`, `some`, `all`, `none`
* Objects:

  * `merge` (shallow object merge)

## Custom extensions

All extensions are prefixed to avoid collisions and are implemented as custom JsonLogic operators. Unknown operators or `call.fn` values are rejected by schema validation; the generic `call` must use one of: `transform.html_to_text`, `extract.urls`, `extract.bullet_list`.

### Path helpers

* You can rely on `var` for traversal. Arrays support `[index]`. Dots split object keys.

### String helpers

* `string.lower`: `{"string.lower": expr}`
* `string.upper`: `{"string.upper": expr}`
* `string.trim`:  `{"string.trim": expr}`
* `string.slice`: `{"string.slice": {"value": expr, "start": 0, "end": 120}}`
* `string.split`: `{"string.split": {"value": expr, "sep": ","}}`
* `string.join`:  `{"string.join": {"items": exprArray, "sep": ", "}}`
* `string.replace`: exact substring replace

  * `{"string.replace":{"value":expr,"find":"x","with":"y","count":1}}`

### Regex helpers

* `regex.match`

  * Input: `{"regex.match":{"value": expr, "pattern":"(?i)invoice|receipt"}}`
  * Output: boolean
* `regex.replace`

  * `{"regex.replace":{"value":expr,"pattern":"\\s+","with":" "}}`

Regex engine is the host regex with a timeout. On timeout or invalid pattern, operator returns `null`.

### Function calls to internal library


{: .note }
Those are the first few I came up with and might change/extend later.

Encoded as operator names under `call.*`. Arguments are named objects.

* `call.transform.html_to_text`

  * Args: `{"html": string}` or `{"html": string, "text": string}`
  * Returns: string
* `call.extract.urls`

  * Args: `{"text": string}` or `{"html": string}`
  * Returns: array of `{"url": string, "context": string}`
* `call.extract.bullet_list`

  * Args: `{"text": string}` or `{"html": string}`
  * Returns: array of `{"text": string, "level": integer}`

Results are plain JSON and can be traversed with `var`. If you assign results to a var, reference as `{"var":"vars.urls[0].url"}`.

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
        "with": "$1"
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

## Runtime, limits, and error policy

* Deterministic, pure evaluation. No IO except registered `call.*` functions.
* Guards:

  * Max expression depth: 50
  * Max nodes evaluated: 10k per mapper
  * Max output size: 1 MiB
  * Regex timeout: 50 ms per op (timeout raises a mapper error)
  * Function call timeout: 100 ms per call (timeout raises a mapper error)
* Operator error returns `null`. Critical errors raise a mapper runtime error and stop execution.


## Schema definition

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
