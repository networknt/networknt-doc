# Access Control

The `AccessControlHandler` is a business middleware handler designed for the `light-rest-4j` framework. It provides fine-grained, rule-based authorization at the endpoint level, allowing developers to define complex access policies that go beyond simple scope-based security.

## Overview

Unlike standard security handlers that verify tokens and scopes, the Access Control handler interacts with the **Rule Engine** to evaluate business-specific logic. It typically runs late in the middleware chain, after technical concerns like security and validation are handled, and right before the request reaches the business logic or proxy.

### Key Features
- **Rule-Based Evaluation**: Leverage the Power of the YAML-based Rule Engine.
- **Dynamic Configuration**: Supports hot-reloading of both configuration and rules.
- **Flexible Logic**: Choose between `any` or `all` logic when multiple rules are applied to an endpoint.
- **Path Skipping**: Easily bypass checks for specific path prefixes.

## Configuration (access-control.yml)

The handler is configured via `access-control.yml`, which is mapped to the `AccessControlConfig` class.

```yaml
# Access Control Handler Configuration
---
# Enable or disable the handler
enabled: ${access-control.enabled:true}

# If there are multiple rules for an endpoint, how to combine them.
# any: access is granted if any rule passes.
# all: access is granted only if all rules pass.
accessRuleLogic: ${access-control.accessRuleLogic:any}

# If no rules are defined for an endpoint, should access be denied?
defaultDeny: ${access-control.defaultDeny:true}

# List of path prefixes to skip access control checks.
skipPathPrefixes:
  - /health
  - /server/info
```

### Configuration Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `enabled` | `true` | Globally enables or disables the handler. |
| `accessRuleLogic` | `any` | Determines evaluation logic for multiple rules (`any` \| `all`). |
| `defaultDeny` | `true` | If `true`, endpoints without defined rules will return an error. |
| `skipPathPrefixes` | `[]` | Requests to these paths skip authorization checks entirely. |

## How it Works

### 1. Rule Loading
Rules are loaded during server startup via the `RuleLoaderStartupHook`. This hook caches the rules in memory for high-performance evaluation.

### 2. Request Context
When a request arrives, the handler extracts context to build a **Rule Engine Payload**:
- **Audit Info**: User information, client ID, and the resolved OpenApi endpoint.
- **Request Headers**: Full map of HTTP headers.
- **Parameters**: Combined query and path parameters.
- **Request Body**: If a body handler is present and the method is POST/PUT/PATCH.

### 3. Evaluation
The handler looks up the rules associated with the current OpenApi endpoint.
- If rules exist, it iterates through them using the configured `accessRuleLogic`.
- Rules are executed by the `RuleEngine`, which returns a `result` (true/false) and optional error details.

### 4. Enforcement
- If the result is **Success**, the request proceeds to the next handler.
- If the result is **Failure**, the handler sets an error status (default `ERR10067`) and stops the chain.

## Hot Reload

The implementation supports hot-reloading through the standardized `AccessControlConfig.reload()` mechanism. 
- When a configuration change is detected at runtime, the singleton `AccessControlConfig` instance is updated.
- The `AccessControlHandler` loads the latest configuration at the start of every request, ensuring zero-downtime updates to authorization policies.
