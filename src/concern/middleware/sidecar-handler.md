# Sidecar Handler

The `sidecar` module provides a set of middleware handlers designed to operate in a sidecar or gateway environment. It enables the separation of cross-cutting concerns from business logic, allowing the light-sidecar to act as a micro-gateway for both incoming (ingress) and outgoing (egress) traffic.

## Overview

In a sidecar deployment (typically within a Kubernetes Pod), the `sidecar` module manages traffic between the backend service and the external network. It allows the backend service—regardless of the language or framework it's built with—to benefit from light-4j features like security, discovery, and observability.

## Configuration (sidecar.yml)

The behavior of the sidecar handlers is controlled via `sidecar.yml`. This configuration is mapped to the `SidecarConfig` class.

### Example sidecar.yml

```yaml
# Light http sidecar configuration
---
# Indicator used to determine the condition for router traffic.
# Valid values: 'header' (default), 'protocol', or other custom indicators.
# If 'header', it looks for service_id or service_url in the request headers.
# If 'protocol', it treats HTTP traffic as egress and HTTPS as ingress (or vice versa depending on setup).
egressIngressIndicator: ${sidecar.egressIngressIndicator:header}
```

### Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `egressIngressIndicator` | `header` | Determines if a request is Ingress or Egress. |

## Ingress vs. Egress Logic

The `SidecarRouterHandler` and related middleware use the `egressIngressIndicator` to decide how to process a request:

1.  **Ingress (Incoming)**: Traffic coming from external clients to the backend API. The sidecar applies security (Token validation), logging, and then proxies the request to the backend API.
2.  **Egress (Outgoing)**: Traffic initiated by the backend API to call another service. The sidecar handles service discovery, load balancing, and token injection (via SAML or OAuth2) before forwarding the request to the target service.

## Sidecar Middleware Handlers

The sidecar module includes specialized versions of standard middleware that are aware of the traffic direction.

### SidecarTokenHandler
Validates tokens for **Ingress** traffic. If the request is identified as **Egress** (based on the indicator), it skips validation to allow the request to proceed to the external service.

### SidecarSAMLTokenHandler
Similar to the Token Handler, it handles SAML token validation for **Ingress** traffic and skips for **Egress**.

### SidecarServiceDictHandler
Applies service dictionary mapping for **Egress** traffic, ensuring the outgoing request is correctly mapped to a target service definition.

### SidecarPathPrefixServiceHandler
Identifies the target service ID based on the path prefix for **Egress** traffic. For **Ingress** traffic, it populates audit attachments and passes the request to the proxy.

## Implementation Details

The `sidecar` module follows the standard light-4j implementation patterns:

-   **Singleton + Caching**: `SidecarConfig` uses a cached singleton instance for performance, loaded via `SidecarConfig.load()`.
-   **Hot Reload**: Supports runtime configuration updates via the `reload()` method, which re-registers the module in the `ModuleRegistry`.
-   **Module Registry Integration**: Automatically registers itself with `/server/info` to provide visibility into the active sidecar configuration.
