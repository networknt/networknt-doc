# Direct Registry

If you don't have Consul or the Light Portal deployed for service registry and discovery, you can use the built-in `DirectRegistry` as a temporary solution. It provides a simple way to define service-to-host mappings and allows for an easy transition to more robust enterprise registry solutions later.

There are two main ways to define the mapping between a `serviceId` and its corresponding hosts:
1.  **service.yml**: The original method, defined as part of the `URLImpl` parameters.
2.  **direct-registry.yml**: The recommended method for dynamic environments like `http-sidecar` or `light-gateway`.

---

### 1. Configuration via service.yml

Defining mappings in `service.yml` is straightforward but comes with a significant limitation: the configuration cannot be reloaded without restarting the server.

#### Single URL Mapping
A simple mapping from one `serviceId` to exactly one service instance.

```yaml
singletons:
- com.networknt.registry.URL:
  - com.networknt.registry.URLImpl:
      protocol: https
      host: localhost
      port: 8080
      path: direct
      parameters:
        com.networknt.apib-1.0.0: http://localhost:7002
        com.networknt.apic-1.0.0: http://localhost:7003
- com.networknt.registry.Registry:
  - com.networknt.registry.support.DirectRegistry
- com.networknt.balance.LoadBalance:
  - com.networknt.balance.RoundRobinLoadBalance
- com.networknt.cluster.Cluster:
  - com.networknt.cluster.LightCluster
```

#### Multiple URL Mapping
For a service with multiple instances, provide a comma-separated list of URLs.

```yaml
      parameters:
        com.networknt.apib-1.0.0: http://localhost:7002,http://localhost:7005
```

#### Using Environment Tags
To support [multi-tenancy](/design/multi-tenancy/), you can pass tags (e.g., environment) into the registered URLs.

```yaml
      parameters:
        com.networknt.portal.command-1.0.0: https://localhost:8440?environment=0000,https://localhost:8441?environment=0001
```

---

### 2. Configuration via direct-registry.yml (Recommended)

When using `DirectRegistry` in `http-sidecar` or `light-gateway`, it is highly recommended to use `direct-registry.yml`. This file supports the **config-reload** feature, allowing you to update service mappings without downtime.

To use this method, ensure that the `parameters` section in the `URLImpl` configuration within `service.yml` is either empty or commented out.

#### Example direct-registry.yml
This file maps `serviceId` (optionally with an environment tag) to host URLs.

```yaml
---
# direct-registry.yml
# Mapping between serviceId and hosts (comma-separated).
# If a tag is used, separate it from serviceId using a vertical bar |
directUrls:
  code: http://192.168.1.100:6881,http://192.168.1.101:6881
  token: http://192.168.1.100:6882
  com.networknt.test-1.0.0: http://localhost,https://localhost
  command|0000: https://192.168.1.142:8440
  command|0001: https://192.168.1.142:8441
```

Mappings can also be provided in **JSON** or **Map String** formats via environment variables or the config server within `values.yml`.

---

### Reloading Configuration

When `DirectRegistry` initializes, it automatically registers its configuration with the `ModuleRegistry`. You can use the [config-reload](/concern/config-reload/) API to trigger a reload of `direct-registry.yml` from the local filesystem or a config server.

#### Simultaneous Reloads
Because components like `RouterHandler` site on top of the registry and cache their own mappings, you must reload the related modules simultaneously to ensure consistency.

**Example Reload Request:**
```bash
curl --location --request POST 'https://localhost:8443/adm/modules' \
--header 'Content-Type: application/json' \
--header 'Authorization: Bearer <TOKEN>' \
--data-raw '[
    "com.networknt.router.middleware.PathPrefixServiceHandler",
    "com.networknt.router.RouterHandler",
    "com.networknt.registry.support.DirectRegistry"
]'
```

### Auto-Registration & Visibility

The `direct-registry` module now implements auto-registration via the `DirectRegistryConfig` singleton. The current active mappings and configuration parameters can be inspected at runtime via the [Server Info](/concern/admin/server-info.md) endpoint.

### Dependency

Add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>registry</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```
