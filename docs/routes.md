---
title: Routes
nav_order: 4
description: "Define MailWebhook routes that match inbound email, run a transformation pipeline, and deliver the resulting JSON to an endpoint."
---

# Routes
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Name

Friendly name for a route

## Endpoint

Select pre-defined [Endpoint].

## Rule & Pipeline (JSON)

Both rule and pipeline defined using JSON for now.

Schema defines both [Rules] and a [Pipeline].

To draft or review the JSON with your coding agent, use the [MailWebhook agent skill](#author-routes-with-an-ai-agent).

Default schema pre-filled for new routes (see below). `map.generic_json` pipeline transforms email into default deterministic shape.

Pipeline reminders:

* `pipeline.steps` must contain at least one step.
* Exactly one `map.*` step must exist.
* The `map.*` step must be the final step.
* Chat mappers use provider-specific destination arguments such as `channel` for Slack and `chat_id` for Telegram.

```json
{
  "rule": {
    "to_contains": [],
    "from_domains": [],
    "subject_contains": [],
    "headers_equals": {}
  },
  "pipeline": {
    "steps": [
      {
        "name": "map.generic_json",
        "args": {}
      }
    ]
  }
}
```

## Author routes with an AI agent

The [MailWebhook agent skill](https://github.com/mailwebhookhq/agent-skills/),
`author-mailwebhook-route-json`, helps your coding agent write, explain, review,
and repair matching rules, transform pipelines, and Custom JSON mappers. It can
return a complete route configuration or just the rule, pipeline, or mapper you
need.

With Node.js and npm installed, run this from your project directory:

```sh
npx skills add mailwebhookhq/agent-skills --skill author-mailwebhook-route-json
```

Follow the CLI prompts to select your coding agent. See the
[repository README](https://github.com/mailwebhookhq/agent-skills/#install) for
agent selection and installation across projects.

Ask your agent to use `author-mailwebhook-route-json`. Include which emails
should match, representative email content, and the JSON your endpoint expects.
For updates, include the existing configuration; for a complete API route body, include
the destination endpoint ID.

For example, to draft the **Rule & Pipeline (JSON)** configuration:

```text
Use author-mailwebhook-route-json to build a rule and pipeline for emails
to invoices@example.com from billing@example.com whose subject contains
"Invoice". Emit Custom JSON with message_id, subject, sender, and text.
Return one JSON object containing rule and pipeline for the route editor. Explain
which emails should match and how missing fields are handled.
```

Review the generated JSON, save it in the route editor, and use
**Mailbox Preview** to check matching and output against representative emails.
Then [send a test email and inspect the payload]. Generating configuration with
the skill does not publish a route or send a webhook.

[Endpoint]: {% link docs/endpoints.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
[send a test email and inspect the payload]: {% link docs/quickstart/send-test-email-inspect-payload.md %}
