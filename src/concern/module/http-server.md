# HTTP Server

The `server` module is the entry point of the light-4j framework. It wraps the [Undertow](http://undertow.io/) core HTTP server and manages its entire lifecycle. This includes initializing configuration loaders, merging status codes, registering the server with the module registry, and orchestrating the request/response handler chain.

## Core Responsibilities

1.  **Lifecycle Management**: Controls server startup, initialization, and graceful shutdown.
2.  **Configuration Injection**: Orchestrates the loading of configurations from local files or the [light-config-server](https://doc.networknt.com/concerns/config-server/).
3.  **Handler Orchestration**: Wires together the middleware handler chain and the final business logic handler.
4.  **Service Registration**: Automatically registers the service instance with registries like Consul or Zookeeper when enabled.
5.  **Distributed Security**: Integrates with TLS and distributed policy enforcement at the edge.

## Server Configuration (server.yml)

The behavior of the server is primarily controlled through `server.yml`. This configuration is mapped to the `ServerConfig` class.

### Example server.yml

```yaml
# Server configuration
---
ip: ${server.ip:0.0.0.0}
httpPort: ${server.httpPort:8080}
enableHttp: ${server.enableHttp:false}
httpsPort: ${server.httpsPort:8443}
enableHttps: ${server.enableHttps:true}
enableHttp2: ${server.enableHttp2:true}
keystoreName: ${server.keystoreName:server.keystore}
keystorePass: ${server.keystorePass:password}
keyPass: ${server.keyPass:password}
enableTwoWayTls: ${server.enableTwoWayTls:false}
truststoreName: ${server.truststoreName:server.truststore}
truststorePass: ${server.truststorePass:password}
serviceId: ${server.serviceId:com.networknt.petstore-1.0.0}
enableRegistry: ${server.enableRegistry:false}
dynamicPort: ${server.dynamicPort:false}
minPort: ${server.minPort:2400}
maxPort: ${server.maxPort:2500}
environment: ${server.environment:dev}
buildNumber: ${server.buildNumber:latest}
shutdownGracefulPeriod: ${server.shutdownGracefulPeriod:2000}
bufferSize: ${server.bufferSize:16384}
ioThreads: ${server.ioThreads:4}
workerThreads: ${server.workerThreads:200}
backlog: ${server.backlog:10000}
alwaysSetDate: ${server.alwaysSetDate:true}
serverString: ${server.serverString:L}
allowUnescapedCharactersInUrl: ${server.allowUnescapedCharactersInUrl:false}
maxTransferFileSize: ${server.maxTransferFileSize:1000000}
maskConfigProperties: ${server.maskConfigProperties:true}
```

### Configuration Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `ip` | 0.0.0.0 | Binding address for the server. |
| `httpPort` | 8080 | Port for HTTP connections (if enabled). |
| `enableHttp` | false | Flag to enable HTTP. Recommended only for local testing. |
| `httpsPort` | 8443 | Port for HTTPS connections. |
| `enableHttps` | true | Flag to enable HTTPS. |
| `enableHttp2` | true | Flag to enable HTTP/2 (requires HTTPS). |
| `keystoreName` | server.keystore | Name of the server keystore file. |
| `serviceId` | | Unique identifier for the service. |
| `dynamicPort` | false | If true, the server binds to an available port in the [minPort, maxPort] range. |
| `bufferSize` | 16384 | Undertow buffer size. |
| `ioThreads` | (CPU * 2) | Number of IO threads for the XNIO worker. |
| `workerThreads` | 200 | Number of worker threads for blocking tasks. |
| `shutdownGracefulPeriod` | 2000 | Wait period in ms for in-flight requests during shutdown. |
| `maskConfigProperties` | true | If true, sensitive properties are masked in the `/server/info` response. |

## Hooks

The server provides a plugin mechanism via hooks, allowing developers to execute custom logic during the startup and shutdown phases.

### Startup Hooks
Used for initialization tasks such as setting up database connection pools, warming up caches, or initializing third-party clients.
Implement `com.networknt.server.StartupHookProvider` and register via `service.yml`.

### Shutdown Hooks
Used for cleanup tasks such as closing connections, releasing resources, or deregistering from a control plane.
Implement `com.networknt.server.ShutdownHookProvider` and register via `service.yml`.

## Advanced Features

### Dynamic Configuration Loading
Using `IConfigLoader` (and implementations like `DefaultConfigLoader`), the server can fetch configurations and secret values from a centralized config server or even a URL on startup.

### Performance Tuning
The server exposes low-level Undertow options like `backlog`, `ioThreads`, and `workerThreads`. The light-4j framework is highly optimized for performance and can handle thousands of concurrent requests with minimal memory footprint.

### Graceful Shutdown
When a shutdown signal (like SIGTERM) is received, the server:
1.  Unregisters itself from the service registry.
2.  Stops accepting new connections.
3.  Waits for the `shutdownGracefulPeriod` to allow in-flight requests to complete.
4.  Executes all registered shutdown hooks.

### Server Info Registry
The server automatically registers itself and its configuration (sensitive values masked) into the `ModuleRegistry`. This information is typically exposed via the `/server/info` endpoint if the `InfoHandler` is in the chain.

## Registry Integration
When `enableRegistry` is true, the server uses the `serviceId`, `ip`, and the bound port to register its location. If `dynamicPort` is active, it explores the port range until it finds an available one, then uses that specific port for registration, enabling seamless scaling in container orchestrators.
