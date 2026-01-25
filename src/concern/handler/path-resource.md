# Path Resource Handler

The `PathResourceHandler` is a middleware handler in `light-4j` that provides an easy way to serve static content from a specific filesystem path. It wraps Undertow's `PathHandler` and `ResourceHandler`, allowing you to expose local directories via HTTP with simple configuration.

### Features

- **External Configuration**: Define path mapping and base directory in a YAML file.
- **Path Matching**: Supports both exact path matching and prefix-based matching.
- **Directory Listing**: Optional support for listing directory contents.
- **Performance**: Integrated with Undertow's `PathResourceManager` for efficient file serving.
- **Auto-Registration**: Automatically registers with `ModuleRegistry` for runtime visibility.

### Configuration (path-resource.yml)

The behavior of the handler is controlled by `path-resource.yml`.

```yaml
# Path Resource Configuration
---
# The URL path at which the static content will be exposed (e.g., /static)
path: ${path-resource.path:/}

# The absolute filesystem path to the directory containing the static files
base: ${path-resource.base:/var/www/html}

# If true, the handler matches all requests starting with the 'path'.
# If false, it only matches exact hits on the 'path'.
prefix: ${path-resource.prefix:true}

# The minimum file size for transfer optimization (in bytes).
transferMinSize: ${path-resource.transferMinSize:1024}

# Whether to allow users to see a listing of files if they access a directory.
directoryListingEnabled: ${path-resource.directoryListingEnabled:false}
```

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
In your `handler.yml`, register the `PathResourceHandler` and add it to your handler chain or path mappings.

```yaml
handlers:
  - com.networknt.resource.PathResourceHandler@pathResource

chains:
  default:
    - ...
    - pathResource
    - ...
```

Or, if you want it on a specific path:

```yaml
paths:
  - path: '/static'
    method: 'GET'
    exec:
      - pathResource
```

### Operational Visibility

The `path-resource` module integrates with the `ModuleRegistry`. You can verify the active `path` and `base` directory mapping at runtime via the [Server Info](/concern/admin/server-info.md) endpoint.
