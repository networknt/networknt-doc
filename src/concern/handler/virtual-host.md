# Virtual Host Handler

The `VirtualHostHandler` is a middleware handler in `light-4j` that enables name-based virtual hosting. It allows a single server instance to serve different content or applications based on the `Host` header in the incoming HTTP request. This is a wrapper around Undertow's `NameVirtualHostHandler`.

### Features

- **Domain-Based Routing**: Route requests to different resource sets based on the domain name.
- **Flexible Mappings**: Each virtual host can define its own path, base directory, and performance settings.
- **Centralized Configuration**: All virtual hosts are defined in a single `virtual-host.yml` file.
- **Auto-Registration**: Automatically registers with `ModuleRegistry` for administrative oversight.

### Configuration (virtual-host.yml)

The configuration contains a list of host definitions.

```yaml
# Virtual Host Configuration
---
hosts:
  - domain: dev.example.com
    path: /
    base: /var/www/dev
    transferMinSize: 1024
    directoryListingEnabled: true
  - domain: prod.example.com
    path: /app
    base: /var/www/prod/dist
    transferMinSize: 1024
    directoryListingEnabled: false
```

#### Host Parameters:
- **`domain`**: The domain name to match (exact match against the `Host` header).
- **`path`**: The URL prefix path within that domain to serve resources from.
- **`base`**: The absolute filesystem path to the directory containing the static content.
- **`transferMinSize`**: Minimum file size for transfer optimization (in bytes).
- **`directoryListingEnabled`**: Whether to allow directory browsing for this specific host.

### Setup

#### 1. Add Dependency
Include the `resource` module in your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>resource</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

#### 2. Register Handler
In your `handler.yml`, register the `VirtualHostHandler`.

```yaml
handlers:
  - com.networknt.resource.VirtualHostHandler@virtualHost

chains:
  default:
    - ...
    - virtualHost
    - ...
```

### How it Works

When a request arrives, the `VirtualHostHandler`:
1.  Inspects the `Host` header of the request.
2.  Matches it against the `domain` entries in the configuration.
3.  If a match is found, it delegates the request to a subdomain-specific `PathHandler` and `ResourceHandler` configured with the corresponding `base` and `path`.
4.  If no match is found, the request typically proceeds down the chain or returns a 404 depending on the rest of the handler configuration.

### Operational Visibility

The `virtual-host` module registers itself with the `ModuleRegistry` during initialization. The full list of configured virtual hosts and their mapping settings can be inspected via the [Server Info](/concern/admin/server-info.md) endpoint.
