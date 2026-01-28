# GraphQL Common

The `graphql-common` module serves as the foundational utility library for the `light-graphql-4j` framework. It provides centralized configuration management, shared constants, and essential helper classes that are used across other GraphQL-related modules (like `graphql-router` and `graphql-validator`).

## Key Features

*   **Centralized Configuration**: Manages the `graphql.yml` configuration parsing and loading.
*   **Hot Reload Support**: Implements the framework's standardized pattern for dynamic configuration reloading.
*   **Module Registration**: Automatically registers itself with the `ModuleRegistry` for runtime observability.
*   **Shared Utilities**: key constants and helper methods for handling GraphQL operations.

## Core Components

### 1. `GraphqlConfig`
This is the primary configuration class. It maps the settings defined in `graphql.yml` to Java objects.

**Singleton Access:**
The class uses a thread-safe singleton pattern. You should always access the configuration via the static `load()` method:
```java
GraphqlConfig config = GraphqlConfig.load();
```

**Configuration Properties:**

| Property | Default | Description |
| :--- | :--- | :--- |
| `path` | `/graphql` | The primary endpoint for handling GraphQL queries and mutations (supports both GET and POST). |
| `subscriptionsPath` | `/subscriptions` | The endpoint specific to WebSocket-based GraphQL subscriptions. |
| `enableGraphiQL` | `true` | A flag to enable or disable the embedded GraphiQL IDE. Typically enabled in dev/test and disabled in prod. |

**Example `graphql.yml`:**
```yaml
path: /api/graphql
subscriptionsPath: /api/subscriptions
enableGraphiQL: false
```

### 2. `GraphqlConstants`
A utility class holding framework-wide constants, ensuring consistency in key names, attribute keys, and default values across the entire GraphQL execution chain.

### 3. `GraphqlUtil`
Provides static helper methods for common GraphQL tasks, such as schema parsing or request validation helpers.

### 4. `InstrumentationLoader` & `InstrumentationProvider`
These classes facilitate the loading of GraphQL instrumentation. They allow developers to inject custom logic (like tracing, logging, or metrics) into the GraphQL execution lifecycle.

## Integration

The `graphql-common` module is typically included automatically if you are using higher-level modules like `light-graphql-4j` router. However, if you are building a custom integration, you can add it directly:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>graphql-common</artifactId>
    <version>${version.light-graphql-4j}</version>
</dependency>
```

## Hot Reload
If the `graphql.yml` configuration is updated on the config server (or locally in dev), the `GraphqlConfig.reload()` method can be triggered to refresh the active configuration without restarting the service. This is particularly useful for toggling features like `enableGraphiQL` or changing endpoint paths on the fly.
