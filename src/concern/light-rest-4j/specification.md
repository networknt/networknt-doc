---
description: Specification Module
---

# Specification Module

The `specification` module in `light-rest-4j` provides a set of handlers to serve and display the API specification (Swagger/OpenAPI) of the service. This is particularly useful for exposing documentation endpoints directly from the running service.

## Overview

Exposing the API contract via the service itself ensures that the documentation is always in sync with the deployed version. The `specification` module provides handlers for:
- Serving the raw specification file (e.g., `openapi.yaml`).
- Rendering a Swagger UI instance to interact with the API.
- Serving a favicon for the UI.

## Components

### SpecDisplayHandler
Serves the raw content of the specification file. It supports different content types (defaulting to `text/yaml`) and loads the file directly from the filesystem or configuration folder.

### SpecSwaggerUIHandler
Renders a simple HTML page that embeds Swagger UI (via CDN). It is pre-configured to point to the `/spec.yaml` endpoint (served by `SpecDisplayHandler`) to load the API definition.

### FaviconHandler
A utility handler that serves a `favicon.ico` file, commonly requested by browsers when accessing the Swagger UI.

## Configuration

The module is configured via `specification.yml`.

### Properties

| Property | Default | Description |
| --- | --- | --- |
| `fileName` | `openapi.yaml` | The path and name of the specification file to be served. |
| `contentType` | `text/yaml` | The MIME type to be used when serving the specification file. |

## Features

### Hot Reload support
The `specification` module fully supports standardized hot-reloading. If the `specification.yml` is updated, the handlers will automatically refresh their internal configuration without requiring a server restart.

### Integration with ModuleRegistry
All handlers in this module register themselves with the `ModuleRegistry` on startup. This allows administrators to verify the loaded configuration via the `/server/info` endpoint.

## Usage

To use these handlers, you need to register them in your `handler.yml`.

### Example `handler.yml` registration:

```yaml
handlers:
  - com.networknt.specification.SpecDisplayHandler@spec
  - com.networknt.specification.SpecSwaggerUIHandler@swagger
  - com.networknt.specification.FaviconHandler@favicon

paths:
  - path: '/spec.yaml'
    method: 'get'
    handler:
      - spec
  - path: '/specui'
    method: 'get'
    handler:
      - swagger
  - path: '/favicon.ico'
    method: 'get'
    handler:
      - favicon
```

## Security Note
Since the specification handlers expose internal API details, it is recommended to protect these endpoints using the `AccessControlHandler` or similar security mechanisms if the documentation should not be publicly accessible.
