---
title: Custom JSON mapper errors
parent: Troubleshooting
nav_order: 8
description: "Troubleshoot MailWebhook Custom JSON mapper errors by checking save-time schema validation, unsupported operators, helper arguments, variable paths, null output, runtime limits, route events, and replay."
permalink: /docs/troubleshooting/custom-json-mapper-errors/
---

# Troubleshoot Custom JSON mapper errors
{: .no_toc }

Use this guide when a route that uses `map.custom_json` does not save, does not create the payload you expected, or creates an event that fails before webhook delivery.

Custom JSON is a terminal route mapper. It runs after a route matches an inbound email and before MailWebhook signs and sends the webhook request. A mapper error stops delivery before the endpoint receives a request.

{: .note }
Building structured email payloads? See [Email to JSON](https://www.mailwebhook.com/email-to-json) for product context.

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Short answer

Start by separating three cases:

| What you see | Where the problem is | First check |
| --- | --- | --- |
| The route will not save and the UI says **Invalid pipeline** | Custom JSON config shape, schema, helper name, operator, or alias | Route pipeline JSON |
| The route creates an event with `route_transform_failed` | Runtime mapper evaluation for that message | Event `last_error` and route pipeline |
| The webhook arrives, but fields are `null` or missing | Variable paths, message content, helper output, or mapper logic | Delivered request body and provider-real message |

Custom JSON paths are read from the mapper evaluation root. Use `message.*`, `ctx.*`, `meta.*`, and `vars.*`. Missing paths return `null`; unsupported operators and invalid helper configuration raise mapper errors.

Use **Mailbox Preview** or **Webhook Preview** to test the route output. Use **Events** when a real message already created an event. After fixing a mapper, send a new test email or replay an existing event.

## Symptoms

Common symptoms include:

- The route editor shows **Invalid pipeline**.
- The error mentions `schema_validation`, `bad_config`, `operator.unknown`, `depth`, `nodes`, or `timeout`.
- The error includes a path such as `/output/customer/call/fn` or `/vars/0/expr`.
- A saved route creates an event, but no delivery attempt appears.
- Event `last_error` includes `[map.custom_json]`.
- Event or notification failure code is `route_transform_failed`.
- A Custom JSON payload contains `null` where a value was expected.
- Array helpers return `null` or an empty array.
- Attachment fields are missing from a Custom JSON payload.
- Replay sends a different body after the route mapper is changed.

## Most common causes

### The pipeline does not end with `map.custom_json`

Every route pipeline must contain exactly one terminal `map.*` step, and that mapper must be the final step.

For Custom JSON, the smallest valid pipeline shape is:

```json
{
  "steps": [
    {
      "name": "map.custom_json",
      "args": {
        "version": "v1",
        "output": {
          "subject": { "var": "message.subject" }
        }
      }
    }
  ]
}
```

If another transform runs after `map.custom_json`, the pipeline fails validation.

### The mapper document is missing required fields

The `args` object for `map.custom_json` must include:

```json
{
  "version": "v1",
  "output": {}
}
```

`vars` is optional. `meta` is optional. Extra top-level keys in the mapper config are rejected by schema validation.

### A helper or operator name is unsupported

Custom JSON supports a focused operator set and a fixed helper list.

Supported helper calls are:

- `transform.html_to_text`
- `extract.urls`
- `extract.bullet_list`
- `extract.reply_segments`
- `extract.key_value_pairs`
- `extract.tables`
- `extract.dom`

Unknown helper names such as `extract.tables_v2` or `extract.reply_segments_v2` fail schema validation. Unknown operator-like keys such as `unknown.op` fail as mapper errors.

### A variable name or array alias is invalid

Top-level `vars` entries and scoped block vars must use names that match:

```text
^[A-Za-z_][A-Za-z0-9_]*$
```

Duplicate var names are rejected in the same `vars` array.

Array helper aliases in `map`, `filter`, `find`, `some`, `all`, and `none` follow the same name rule. They also cannot use reserved names:

- `message`
- `ctx`
- `meta`
- `vars`
- `current`
- `accumulator`

For example, use `pair` instead of `message`:

```json
{
  "map": {
    "over": { "var": "vars.kv.items" },
    "as": "pair",
    "do": { "var": "pair.value" }
  }
}
```

### The mapper uses the wrong evaluation root

Custom JSON does not read from the delivered Generic JSON shape.

Use these roots:

| Root | Use it for |
| --- | --- |
| `message` | Parsed email fields, body text, body HTML, headers, recipients, attachments |
| `ctx` | Event id, project id, route id, source type, raw size, current timestamp |
| `meta` | Static mapper metadata from the route config |
| `vars` | Values computed earlier in the mapper |

Common valid paths include:

- `message.message_id`
- `message.subject`
- `message.from[0].email`
- `message.to[0].email`
- `message.headers.x_priority`
- `message.attachments[0].filename`
- `ctx.event_id`
- `ctx.source_type`
- `vars.text`

Paths such as `body.text`, `body.attachments`, `event.id`, and `runtime.source` are not Custom JSON evaluation roots.

### A path is missing and evaluates to `null`

Missing paths do not raise mapper errors. They evaluate to `null`.

This can happen when:

- The email has no HTML body and the mapper reads `message.html`.
- The email has no attachments and the mapper reads `message.attachments[0].id`.
- The route-level test message lacks provider-real headers or attachment metadata.
- A helper returned no rows, links, tables, or key/value pairs.
- A `vars` entry name is misspelled in `output`.

Use a provider-real test email when the mapper depends on headers, attachments, Gmail labels, Microsoft folders, or IMAP provider behavior.

### Helper arguments are invalid

Some helpers validate their argument objects at runtime.

Examples:

- `extract.reply_segments` rejects unsupported argument keys and invalid `sources`.
- `extract.key_value_pairs` rejects invalid option values such as unsupported separator settings.
- `extract.tables` rejects invalid table extraction modes.
- `extract.dom` rejects invalid selector options.

When this happens, the error code is usually `bad_config`, and the error details include the helper name.

### Runtime limits stopped evaluation

Custom JSON has runtime limits to keep webhook delivery predictable.

Important limits include:

- Expression depth limit.
- Expression node count limit.
- Regex timeout.
- Helper call timeout.

Depth or node errors usually mean the mapper is too large or too deeply nested. Regex and helper timeouts usually mean the expression is doing too much work on the message content.

### The output shape is valid, but not what the receiver expects

Custom JSON can emit only the fields you define. It does not automatically include Generic JSON fields.

If your receiver needs event id, message id, source, body text, or attachments, map them explicitly:

```json
{
  "version": "v1",
  "output": {
    "event_id": { "var": "ctx.event_id" },
    "message_id": { "var": "message.message_id" },
    "source": { "var": "ctx.source_type" },
    "subject": { "var": "message.subject" },
    "text": { "var": "message.text" },
    "attachments": { "var": "message.attachments" }
  }
}
```

If top-level `output` evaluates to `null`, MailWebhook emits an empty object.

### The route did not match before the mapper ran

Custom JSON runs only after route matching.

If no event exists for the route, debug route rules and mailbox ingestion before changing the mapper. A mapper error cannot happen until the route has matched an ingested message.

## Step-by-step checks

### 1. Identify save-time or runtime failure

Use this table:

| Location | What it means | Next step |
| --- | --- | --- |
| Route save error | The pipeline did not compile. | Fix schema, operators, helpers, vars, or pipeline shape. |
| Event `last_error` | The pipeline compiled, then failed for a message. | Inspect the event, message, and mapper logic. |
| Delivery attempt exists | The mapper produced a payload. | Debug endpoint delivery or receiver behavior. |
| No event exists | The route did not match, or the message was not ingested. | Debug mailbox and route matching. |

### 2. Check the pipeline shape

Open the route pipeline JSON.

Confirm:

- `steps` is a non-empty array.
- There is exactly one `map.*` step.
- `map.custom_json` is the final step.
- The Custom JSON mapper document is inside `args`.
- `args.version` is `v1`.
- `args.output` exists.

### 3. Use the error path

When an error includes a path, use it as a pointer into the mapper config.

Examples:

| Error path | Check |
| --- | --- |
| `/output/customer/call/fn` | Unsupported helper name in a generic `call`. |
| `/output/kv/call.extract.key_value_pairs_v2` | Unsupported prefixed helper name. |
| `/output/bad/map/as` | Invalid or reserved array alias. |
| `/output/block/vars/0/expr` | Scoped block var is missing `expr`. |
| `/vars/0/expr` | Top-level var expression is invalid. |

Fix the smallest block first, then save the route again.

### 4. Compare paths to the Custom JSON root

Check every `var` path against the Custom JSON root.

Use:

```json
{ "var": "message.text" }
```

Avoid Generic JSON output paths such as:

```json
{ "var": "body.text" }
```

If the field can be absent, use `or` or `if` to provide a fallback:

```json
{
  "summary": {
    "or": [
      { "var": "message.text" },
      { "call.transform.html_to_text": { "html": { "var": "message.html" } } },
      ""
    ]
  }
}
```

### 5. Reduce the mapper to a known-good output

Temporarily replace the Custom JSON output with a small diagnostic payload:

```json
{
  "version": "v1",
  "output": {
    "event_id": { "var": "ctx.event_id" },
    "source": { "var": "ctx.source_type" },
    "subject": { "var": "message.subject" },
    "from": { "var": "message.from[0].email" },
    "text": { "var": "message.text" }
  }
}
```

After this works, add one helper, one var, or one output field at a time.

### 6. Check helper output before indexing into it

Helpers can return empty arrays or empty result objects.

For a table extraction workflow, first expose the summary:

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
    "table_count": { "var": "vars.tables.summary.table_count" },
    "first_table_source": { "var": "vars.tables.tables[0].source" }
  }
}
```

Only add deeper paths such as `vars.tables.tables[0].lookup.by_row.widget.qty` after confirming the table exists.

### 7. Check route matching before mapper output

If no event exists, Custom JSON did not run.

Open **Mailbox Preview**, select the message and route, and check whether the route matched. If preview says the selected message does not match the route, fix the route rule first.

### 8. Check delivery only after mapper output exists

If the event has a delivery attempt, the mapper produced a payload. Continue with endpoint troubleshooting.

If the event has `route_transform_failed` and no delivery attempt, the endpoint was never called. Fix the route pipeline, then send a new test email or replay the event.

### 9. Replay after fixing a mapper

Replay rebuilds the request body from the stored message and the route's current pipeline.

Use replay when:

- The event exists.
- The message is still the right test case.
- You changed the Custom JSON mapper or endpoint.
- You want to send the same stored message through the corrected route.

Send a new email instead when the problem depends on mailbox source behavior, provider labels, folder placement, or attachment extraction.

## Expected result

After the Custom JSON mapper is fixed:

- The route saves without an **Invalid pipeline** error.
- The route has one final `map.custom_json` step.
- The mapper config uses `version: "v1"` and a valid `output`.
- Vars, helper names, aliases, and scoped blocks pass validation.
- Provider-real test messages produce the expected Custom JSON output.
- Events no longer show `route_transform_failed`.
- Delivery attempts appear after the mapper produces a payload.
- Replay uses the corrected route pipeline for existing events.

## Related docs

- [Custom JSON]
- [JsonLogic-style DSL]
- [Custom JSON recipes]
- [Pipeline]
- [Transform steps]
- [Webhook payload reference]
- [Send a test email and inspect the payload]
- [Troubleshoot route rules that do not match]
- [Troubleshoot failed webhook deliveries]
- [Webhook retries and replay]

[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
[JsonLogic-style DSL]: {% link docs/routes/pipeline/custom_json/dsl.md %}
[Custom JSON recipes]: {% link docs/routes/pipeline/custom_json/recipes.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[Transform steps]: {% link docs/routes/pipeline/transform_steps.md %}
[Webhook payload reference]: {% link docs/payloads/webhook-payload-reference.md %}
[Send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
[Troubleshoot route rules that do not match]: {% link docs/troubleshooting/route-rule-did-not-match.md %}
[Troubleshoot failed webhook deliveries]: {% link docs/troubleshooting/webhook-delivery-failed.md %}
[Webhook retries and replay]: {% link docs/delivery/retries-and-replay.md %}
