---
title: API Reference
nav_order: 8
layout: minimal
permalink: /api-reference/
has_toc: false
search_exclude: true
description: "Browse the MailWebhook OpenAPI reference with Swagger UI."
---

<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swagger-ui-dist@5.32.8/swagger-ui.css">

<style>
  .main-content-wrap,
  .main-content {
    max-width: none;
    padding: 0;
  }

  .main-content main {
    display: block;
    width: 100%;
  }

  #swagger-ui {
    min-height: calc(100vh - 60px);
  }

  #swagger-ui .swagger-ui {
    font-family: inherit;
  }
</style>

<div id="swagger-ui" aria-label="MailWebhook API Reference"></div>

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
