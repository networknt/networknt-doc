# Config Reload

The `config-reload` module provides admin endpoints to reload configuration files for specific modules or plugins at runtime without restarting the server. This is particularly useful in shared environments like a gateway where restarting would impact multiple services, or for dynamic updates to policies and rules.

## Endpoints

The module typically exposes two operations on the same path (e.g., `/adm/modules`).

### Get Registered Modules

-   **Path**: `/adm/modules`
-   **Method**: `GET`
-   **Description**: Returns a list of all registered modules and plugins capable of being reloaded.
-   **Response**: A JSON array of class names (Strings).

### Trigger Config Reload

-   **Path**: `/adm/modules`
-   **Method**: `POST`
-   **Description**: Triggers a config reload for the specified list of modules or plugins.
-   **Request Body**: A JSON array of class names (Strings) to reload. If the list is empty or contains "ALL", it attempts to reload all registered modules.
-   **Response**: A JSON array of the class names that were successfully reloaded.

## Configuration

### Module Configuration (`configreload.yml`)

The module itself is configured via `configReload.yml` (or `configReload.yaml`/`json`).

| Property | Description | Default |
| :--- | :--- | :--- |
| `enabled` | Enable or disable the config reload endpoints. | `true` |

### Handler Configuration (`handler.yml`)

To expose the endpoints, you need to configure them in your `handler.yml`.

```yaml
paths:
  - path: '/adm/modules'
    method: 'get'
    exec:
      - admin
      - modules
  - path: '/adm/modules'
    method: 'post'
    exec:
      - admin
      - configReload
```

And in `values.yml` (if using aliases):

```yaml
handler.handlers:
  - com.networknt.config.reload.handler.ModuleRegistryGetHandler@modules
  - com.networknt.config.reload.handler.ConfigReloadHandler@configReload
```

## Security

These are sensitive admin endpoints. They should be protected by OAuth 2.0 (requiring a specific scope like `admin` or `${serviceId}/admin`) or restricted by IP whitelist in a sidecar/gateway deployment.

## Dependency

Add the following dependency to your `pom.xml`:

```xml
<dependency>
  <groupId>com.networknt</groupId>
  <artifactId>config-reload</artifactId>
  <version>${version.light-4j}</version>
</dependency>
```

## Usage Examples

### List Modules

```bash
curl -k https://localhost:8443/adm/modules
```

**Response:**
```json
[
    "com.networknt.limit.LimitHandler",
    "com.networknt.audit.AuditHandler",
    "com.networknt.router.middleware.PathPrefixServiceHandler",
    ...
]
```

### Reload Config

Reload configuration for the Limit Handler and Path Prefix Handler.

```bash
curl -k -X POST https://localhost:8443/adm/modules \
  -H 'Content-Type: application/json' \
  -d '[
    "com.networknt.limit.LimitHandler",
    "com.networknt.router.middleware.PathPrefixServiceHandler"
  ]'
```

**Response:**
```json
[
    "com.networknt.limit.LimitHandler",
    "com.networknt.router.middleware.PathPrefixServiceHandler"
]
```
