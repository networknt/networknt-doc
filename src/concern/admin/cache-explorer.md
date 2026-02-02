# Cache Explorer

The `CacheExplorerHandler` is an admin endpoint that allows developers and operators to inspect the contents of the application's internal caches. It is particularly useful for debugging and verifying that data (such as JWKs, ensuring config reloading, etc.) is correctly cached.

## Overview

-   **Type**: Admin Handler
-   **Class**: `com.networknt.cache.CacheExplorerHandler`
-   **Source**: `light-4j/cache-explorer`

## Usage

The handler operates by retrieving a specific cache by name via the `CacheManager` singleton.

### Endpoint

The endpoint is typically accessible via the admin port (default 8473) if configured using the [admin-endpoint][] module.

**Method**: `GET`
**Path**: `/adm/cache` (or as configured in `handler.yml`)

### Parameters

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `name` | Query | Yes | The name of the cache to inspect (e.g., `jwk`, `jwt`, `myCache`). |

### Examples

#### Inspect JWK Cache

The `jwk` cache has special handling to format the output as a simple JSON map of strings.

Request:
```
GET /adm/cache?name=jwk
```

Response:
```json
{
  "key1": "value1",
  "key2": "value2"
}
```

#### Inspect Generic Cache

For other caches, it returns the JSON representation of the cache's key-value pairs.

Request:
```
GET /adm/cache?name=myCache
```

Response:
```json
{
  "user123": {
    "name": "John",
    "role": "admin"
  }
}
```

## Configuration

This module does not have a specific configuration file (e.g., `cache-explorer.yml`). It relies on the `CacheManager` being available and populated.

To use it, you must register the handler in `handler.yml` and add it to the admin chain (or another chain).

### `handler.yml` Registration

```yaml
handler.handlers:
  # ... other handlers ...
  - com.networknt.cache.CacheExplorerHandler@cache_explorer

handler.chains.default:
  # ...
  - cache_explorer
```

**Note**: In most standard `light-4j` templates, if you are using the unified `admin-endpoint`, this handler might not be wired by default and needs explicit registration if you want it exposed on the main port, or it might be auto-wired in the admin chain if the dependency is present (depending on the chassis version).

[admin-endpoint]: /concern/admin-endpoint/
