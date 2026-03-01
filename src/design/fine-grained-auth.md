# Fine-Grained Authorization (FGA) Design

In the Light framework ecosystems (`light-rest-4j`, `light-hybrid-4j`, and `light-graphql-4j`), Coarse-Grained Authorization revolves around OAuth 2.0 JWT token scopes, primarily ensuring that users possess top-level access to broad endpoints. 

However, **Fine-Grained Authorization (FGA)** applies dynamic, rule-based business logic to requests—operating at the row or column level—to selectively filter attributes, enforce ownership constraints, or permit conditionally authorized actions based on arbitrary token claims or request payloads.

This document outlines the architecture, components, and the integration flows supporting FGA.

## Core Components

The FGA system relies on the interplay of several core components structured to provide high performance (via local caching) while maintaining centralized policy definitions natively managed within the `light-portal`.

### 1. `RuleEngine`
The `RuleEngine` (housed in `yaml-rule`) evaluates incoming contextual payloads against a predefined set of YAML rules. The rules contain conditions structured around the business domain, granting binary (pass/fail) results or mutating payload structures.

### 2. `RuleLoaderStartupHook`
To prevent introducing network latency during live authorization checks, FGA operates completely in-memory. 

The `RuleLoaderStartupHook` (in `light-4j`) runs when a framework server starts up. It connects to the `light-portal` (acting as the control plane) to download all applicable YAML rules and API permissions specific to the host, API ID, and API version. 
- It caches all endpoint mapped rules natively inside `RuleLoaderStartupHook.endpointRules`.
- It caches the YAML schemas dynamically parsed as `Rule` objects inside `RuleLoaderStartupHook.rules`.
- It initializes the `RuleEngine` Singleton.

### 3. `AccessControlHandler`
The `AccessControlHandler` operates as a crucial middleware in both `light-rest-4j` and `light-hybrid-4j` architectures. As a middleware handler injected after authentication mechanisms (like `JwtVerifyHandler`), it builds a contextual Request Payload encapsulating headers, query parameters, method types, and the deserialized JWT claims (found in `auditInfo`).

It behaves iteratively:
1. **Lookup**: Resolves the exact endpoint being requested.
2. **Match**: Fetches the associated rules from the `RuleLoaderStartupHook` mappings for that endpoint.
3. **Execute**: Pipes the contextual payload through the `RuleEngine`.
4. **Enforce**: Depending on the server configuration (`accessRuleLogic`: `all` vs. `any`), the handler will either advance the HTTP Exchange to the next middleware or abort the request, dumping an `ERR10067` or `ERR10069` Status code indicating an access control failure or missing rule fallback.

## Refactoring and Integration Gaps

Historically, the `RuleLoaderStartupHook` depended on the `market` ecosystem to retrieve API endpoint rules. However, to modernize the control plane and decouple rule queries from the legacy marketplace components, Rule/Permission persistence has been shifted to specialized domain queries.

### API Migration Map
The `market` actions were deprecated and substituted with distinct domain boundaries:
- **`getServiceRule`**: Migrated out of `market` and is now natively handled by `GetServiceRule.java` within the `service-query` module mapped to the `service` target endpoint.
- **`getApiPermission`**: Migrated out of `market` and natively handled by `GetApiPermission.java` within the `service-query` module. (Note: `GetRuleByApiId.java` inside `rule-query` serves simple rule mappings, whereas the startup hook specifically requires the enriched unified schema containing `ruleBodies` provided by the structured `GetServiceRule`).

### Updates Required
Framework integrations must ensure their startup loops target the updated schema formats. Specifically, the `RuleLoaderStartupHook` payloads strictly hit the modernized endpoints:
```json
{
  "host": "lightapi.net",
  "service": "service",
  "action": "getServiceRule",   // or getApiPermission
  "version": "0.1.0",
  "data": { ... }
}
```

## Testing Strategy

Validating FGA logic avoids reliance on complex networked setups by supplying the `RuleLoaderStartupHook` instances with mock configurations at test-time.

For unit tests targeting `AccessControlHandler`:
1. **Initialization Bypass**: Tests artificially seed `RuleLoaderStartupHook.rules` and `RuleLoaderStartupHook.endpointRules` with predictable mapping behaviors before rule engine execution.
2. **Path Skip Asserts**: `AccessControlHandler` allows endpoints strictly matching the `skipPathPrefixes` config variable to bypass FGA (e.g., standard `/health` endpoints).
3. **Failure Overrides**: If rules are omitted, the explicit verification of `config.isDefaultDeny()` governs whether missing policies act as permit-all traps or deny-all blocks. Tests must assert that HTTP 403s are accurately rendered when rule configurations lack explicit definition.
