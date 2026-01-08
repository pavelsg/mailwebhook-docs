---
title: Pipeline
parent: Routes
nav_order: 2
---

# Pipeline
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Overview

A pipeline is the per-route transformation chain that runs only after a route
rule matches an incoming email. It consumes the parsed message plus route
context and emits a JSON payload; that payload is posted verbatim to the route's
endpoint.

The default step is `map.generic_json`, which emits the deterministic
`mailwebhook.generic@1` shape described in [Generic JSON]. Attachments stay out
of the HTTP body and only their metadata is included, keeping delivery fast and
reliable.

For custom outputs, `map.custom_json_mapper` lets you define a JsonLogic-based
mapper validated by JSON Schema, producing whatever JSON shape your downstream
system expects. See [Custom JSON] for the mapper spec, supported operators, and
runtime limits.

[Generic JSON]: {% link docs/routes/pipeline/generic_json.md %}
[Custom JSON]: {% link docs/routes/pipeline/custom_json.md %}
