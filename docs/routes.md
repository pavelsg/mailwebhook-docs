---
title: Routes
nav_order: 2
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

Default schema pre-filled for new routes (see below). `map.generic_json` pipeline transforms email into default deterministic shape.

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

[Endpoint]: {% link docs/endpoints.md %}
[Rules]: {% link docs/routes/rules.md %}
[Pipeline]: {% link docs/routes/pipeline.md %}
