# Rule Loader

The `rule-loader` is a vital infrastructure module in `light-4j` that enables services to dynamically load and manage business logic rules during startup. It serves as the bridge between the [YAML Rule Engine](/library/yaml-rule/) and the specific service endpoints, allowing for flexible, rule-based request and response processing.

### Overview

Rules in the `light-4j` ecosystem are often shared across multiple lines of business or organizations. By centralizing these rules in the `light-portal`, applications can subscribe to them and have them automatically fetched and instantiated at runtime.

### Features

- **Dual Rule Sources**: Fetch rules from the `light-portal` for production environments or load them from a local `rules.yml` file for offline testing.
- **Startup Integration**: Uses a standard `StartupHookProvider` to ensuring all rules are ready before the server starts accepting traffic.
- **Endpoint-to-Rule Mapping**: Flexible mapping of endpoints to specific rule sets (e.g., access control, request transformation, response filtering).
- **Service Dependency Enrichment**: Automatically fetches service-specific permissions and enriches rule contexts.
- **Dynamic Action Loading**: Automatically discovery and instantiates action classes defined within the rules to prevent runtime errors.
- **Auto-Registration**: Automatically registers with `ModuleRegistry` for runtime configuration inspection.

### Configuration (rule-loader.yml)

The `rule-loader.yml` file defines how and where the rules are fetched.

```yaml
# Rule Loader Configuration
---
# A flag to enable the rule loader. Default is true.
enabled: ${rule-loader.enabled:true}

# Source of the rules: 'light-portal' or 'config-folder'.
# 'light-portal' fetches from a remote server.
# 'config-folder' expects a rules.yml file in the externalized config directory.
ruleSource: ${rule-loader.ruleSource:light-portal}

# The portal host URL (used when ruleSource is light-portal).
portalHost: ${rule-loader.portalHost:https://localhost}

# The authorization token for connecting to the light-portal.
portalHost: ${rule-loader.portalToken:}

# Endpoint to rules mapping (used when ruleSource is config-folder).
# endpointRules:
#   /v1/pets@get:
#     res-tra:
#       - ruleId: transform-pet-response
#   /v1/orders@post:
#     access-control:
#       - ruleId: check-order-permission
```

### Rule Sources

#### 1. Light Portal (`light-portal`)
In this mode, the loader interacts with the portal API to fetch the latest rules authorized for the service based on the `hostId`, `apiId`, and `apiVersion` defined in `values.yml` or `server.yml`. The fetched rules are cached locally in the target configuration directory as `rules.yml` for resilience.

#### 2. Config Folder (`config-folder`)
This mode is ideal for local development or air-gapped environments. The loader looks for a `rules.yml` file in the configuration directory and uses the `endpointRules` map defined in `rule-loader.yml` to link endpoints to rule IDs.

### Setup

#### 1. Add Dependency
Include the `rule-loader` in your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>rule-loader</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

#### 2. Configure Startup Hook
The `RuleLoaderStartupHook` must be registered in your `service.yml` or `handler.yml` under the startup hooks section.

```yaml
- com.networknt.server.StartupHookProvider:
  - com.networknt.rule.RuleLoaderStartupHook
```

### Operational Visibility

The `rule-loader` module utilizes the Singleton pattern for its configuration and automatically registers itself with the `ModuleRegistry`. You can inspect the current `ruleSource`, `portalHost`, and active `endpointRules` mappings at runtime via the [Server Info](/concern/admin/server-info.md) endpoint.

### Action Class discovery

During the initialization phase, the loader iterates through all actions defined in the loaded rules and attempts to instantiate their associated Java classes. This proactive step ensures that all required JAR files are present on the classpath and that the classes are correctly configured before the first request arrives.
