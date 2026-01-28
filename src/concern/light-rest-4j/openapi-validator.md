---
description: OpenAPI Validator Module
---

# OpenAPI Validator

The `openapi-validator` module provides comprehensive request and response validation against an OpenAPI Specification 3.0. It ensures that incoming requests and outgoing responses adhere to the schemas, parameters, and constraints defined in your API specification.

## Overview

In a contract-first development approach, the OpenAPI specification serves as the "source of truth." The `openapi-validator` middleware automates the enforcement of this contract, reducing the need for manual validation logic in your business handlers and improving API reliability.

## Core Components

### ValidatorHandler
The main middleware entry point. It identifies whether validation is required for the current request and coordinates the `RequestValidator` and `ResponseValidator`.

### RequestValidator
Validates all aspects of an incoming HTTP request:
- **Path Parameters**: Checks if path variables match the spec.
- **Query Parameters**: Validates presence, type, and constraints of query strings.
- **Header Parameters**: Validates required headers and their values.
- **Cookie Parameters**: Validates cookie values if defined.
- **Request Body**: Validates the JSON payload against the operation's requestBody schema.

### ResponseValidator
Validates the outgoing response from the server. This is typically disabled in production for performance reasons but is invaluable during development and testing to ensure the server respects its own contract.

### SchemaValidator
The underlying engine (built on `networknt/json-schema-validator`) that performs the actual JSON Schema validation for bodies and complex parameters.

## Configuration

The module is configured via `openapi-validator.yml`.

### Key Properties

| Property | Default | Description |
| --- | --- | --- |
| `enabled` | `true` | Globally enables or disables the validator. |
| `logError` | `true` | If true, validation errors are logged to the console/file. |
| `legacyPathType` | `false` | If true, uses the legacy dot-separated path format in error messages instead of JSON Pointers. |
| `skipBodyValidation` | `false` | Useful for gateways or proxies that want to validate headers/parameters but pass the body through without parsing. |
| `validateResponse` | `false` | Enables validation of outgoing responses. |
| `handleNullableField` | `true` | If true, treats fields explicitly marked as `nullable: true` in the spec correctly. |
| `skipPathPrefixes` | `[]` | A list of path prefixes to skip validation for (e.g., `/health`, `/info`). |

## Features

### Hot Reload Support
The validator supports hot-reloading. If the `openapi-validator.yml` or the underlying `openapi.yml` specification is updated on the filesystem, the handler will automatically detect the change and re-initialize the internal validators without a server restart.

### Multiple Specification Support
For gateway use cases where a single server might handle multiple APIs, the validator can maintain separate validation contexts for different path prefixes, each associated with its own OpenAPI specification.

### Integration with BodyHandler
The `RequestValidator` automatically retrieves the parsed body from the `BodyHandler`. To validate the request body, ensure that `BodyHandler` is placed before `ValidatorHandler` in your handler chain.

## Error Handling

When validation fails, the handler returns a standardized error response with a `400 Bad Request` status (for requests) or logs an error (for responses). The error body follows the standard `light-4j` status format, including a unique error code and a descriptive message pointing to the specific field that failed validation.

## Best Practices

1. **Development vs. Production**: Always enable `validateResponse` during development and CI/CD testing, but consider disabling it in production for high-throughput services.
2. **Contract-First**: Keep your `openapi.yml` accurate. The validator is only as good as the specification it follows.
3. **Gateway Optimization**: Use `skipBodyValidation: true` in gateways if the backend service is also performing validation, to save on CPU cycles spent parsing large JSON payloads twice.
