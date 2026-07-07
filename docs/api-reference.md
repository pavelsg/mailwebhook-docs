---
title: API Reference
nav_order: 8
layout: default
permalink: /api-reference/
has_toc: false
search_exclude: true
description: "Browse the MailWebhook OpenAPI reference with Swagger UI."
---

# API Reference
{: .no_toc }

Browse the MailWebhook OpenAPI reference, or open the raw [OpenAPI JSON].

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.32.8/swagger-ui.css">

<style>
  .swagger-ui-shell {
    margin-top: 1.5rem;
  }

  #swagger-ui {
    width: 100%;
  }

  #swagger-ui .swagger-ui .wrapper {
    max-width: 100%;
    padding-right: 0;
    padding-left: 0;
  }

  #swagger-ui .swagger-ui {
    font-family: inherit;
  }

  #swagger-ui .swagger-ui .scheme-container {
    padding: 1rem 0;
    box-shadow: none;
  }
</style>

<div class="swagger-ui-shell">
  <div id="swagger-ui" aria-label="MailWebhook API Reference"></div>
</div>

<noscript>
  <p>
    JavaScript is required to render Swagger UI. The OpenAPI document is available at
    <a href="{{ '/assets/json/public-openapi.v1.json' | relative_url }}">/assets/json/public-openapi.v1.json</a>.
  </p>
</noscript>

<script src="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.32.8/swagger-ui-bundle.js"></script>
<script>
  window.ui = SwaggerUIBundle({
    url: "{{ '/assets/json/public-openapi.v1.json' | relative_url }}",
    dom_id: "#swagger-ui",
    deepLinking: true,
    displayRequestDuration: true,
    docExpansion: "list",
    supportedSubmitMethods: [],
    validatorUrl: null,
    presets: [SwaggerUIBundle.presets.apis],
    layout: "BaseLayout",
  })
</script>

[OpenAPI JSON]: {{ '/assets/json/public-openapi.v1.json' | relative_url }}
