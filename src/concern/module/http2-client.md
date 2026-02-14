# Http2 Client

The `Http2Client` module provided by `light-4j` is a high-performance, asynchronous HTTP client that supports both HTTP/1.1 and HTTP/2. It is built on top of Undertow's client capabilities and is designed to be efficient, lightweight, and easy to use in microservices architectures.

## Features

-   **HTTP/1.1 and HTTP/2 Support**: Transparently handles both protocol versions.
-   **Asynchronous & Non-blocking**: Uses callbacks and futures to handle requests without blocking threads.
-   **Connection Pooling**: Built-in connection pooling for efficient resource management, especially for HTTP/1.1.
-   **OAuth 2.0 Integration**: Built-in support for handling OAuth 2.0 tokens (client credentials, authorization code, etc.).
-   **Service Discovery**: Integrates with light-4j service discovery to resolve service IDs to URLs.
-   **TLS/SSL Support**: Comprehensive TLS configuration including two-way SSL.

> [!IMPORTANT]
> **Migration Notice**: The `borrowConnection`/`returnConnection` methods are deprecated. Migrate to the new `borrow()`/`restore()` SimplePool API for better metrics, health checks, and pool management. See the [SimplePool Migration Guide](simplepool-migration.md) for details.

## Configuration

The client is configured via `client.yml`.

### Key Configuration Sections

#### TLS (`tls`)

Configures Transport Layer Security settings.

| Property | Description | Default |
| :--- | :--- | :--- |
| `verifyHostname` | Verify the hostname against the certificate. | `true` |
| `loadTrustStore` | Load the trust store. | `true` |
| `trustStore` | Path to the trust store file. | `client.truststore` |
| `loadKeyStore` | Load the key store (for 2-way SSL). | `false` |
| `tlsVersion` | TLS version to use (e.g., TLSv1.2, TLSv1.3). | `TLSv1.2` |

#### Request (`request`)

Configures request behavior and connection pooling.

| Property | Description | Default |
| :--- | :--- | :--- |
| `timeout` | Request timeout in milliseconds. | `3000` |
| `enableHttp2` | Enable HTTP/2 support. | `true` |
| `connectionPoolSize` | Max connections per host in the pool. | `1000` |
| `maxReqPerConn` | Max requests per connection before closing. | `1000000` |
| `poolMetricsEnabled` | Enable connection pool metrics. | `false` |
| `healthCheckEnabled` | Enable background connection health checks. | `true` |
| `healthCheckIntervalMs` | Health check interval in milliseconds. | `30000` |

#### OAuth (`oauth`)

Configures interactions with OAuth 2.0 providers for token management. Includes sections for `token` (acquiring tokens), `sign` (signing requests), and `key` (fetching JWKs).

### Example `client.yml`

```yaml
tls:
  verifyHostname: true
  loadTrustStore: true
  trustStore: client.truststore
  trustStorePass: password
request:
  timeout: 3000
  enableHttp2: true
  connectionPoolSize: 100
  poolMetricsEnabled: true
  healthCheckEnabled: true
oauth:
  token:
    server_url: https://localhost:6882
    client_credentials:
      client_id: my-client
      client_secret: my-secret
```

## Usage

### Getting the Client Instance

The `Http2Client` is a singleton.

```java
import com.networknt.client.Http2Client;

Http2Client client = Http2Client.getInstance();
```

### Making a Request (Recommended - SimplePool)

Use `borrow()` to get a connection token and `restore()` to return it.

```java
import com.networknt.client.simplepool.SimpleConnectionState.ConnectionToken;
import io.undertow.client.ClientConnection;
import io.undertow.client.ClientRequest;
import io.undertow.util.Methods;
import java.net.URI;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicReference;

// ...

ConnectionToken token = null;
try {
    // Borrow a connection token
    token = client.borrow(
        new URI("https://example.com"),
        Http2Client.WORKER,
        Http2Client.BUFFER_POOL,
        true  // HTTP/2 enabled
    );
    
    ClientConnection connection = token.connection().getRawConnection();

    // Create the request
    ClientRequest request = new ClientRequest().setMethod(Methods.GET).setPath("/api/v1/data");
    request.getRequestHeaders().put(Headers.HOST, "example.com");

    // Latch to wait for response
    final CountDownLatch latch = new CountDownLatch(1);
    final AtomicReference<ClientResponse> reference = new AtomicReference<>();

    // Send the request
    connection.sendRequest(request, client.createClientCallback(reference, latch));

    // Wait for the response
    latch.await();

    // Process response
    ClientResponse response = reference.get();
    int statusCode = response.getResponseCode();
    String body = response.getAttachment(Http2Client.RESPONSE_BODY);

    System.out.println("Status: " + statusCode);
    System.out.println("Body: " + body);

} catch (Exception e) {
    e.printStackTrace();
} finally {
    // Always restore the token
    if (token != null) {
        client.restore(token);
    }
}
```

### Handling Token Injection

The client can automatically inject OAuth tokens.

```java
// Inject Client Credentials token
client.addCcToken(request);

// Inject Authorization Code token (Bearer token passed from caller)
client.addAuthToken(request, "bearer_token_string");
```

### Propagating Headers

For service-to-service calls, you often need to propagate headers like correlation ID and traceability ID.

```java
// Propagate headers from an existing HttpServerExchange
client.propagateHeaders(request, exchange);
```

### Pool Warm-Up

Pre-establish connections to reduce first-request latency:

```java
// Warm up connections at application startup
client.warmUpPool(URI.create("https://api.example.com:443"));
```

### Pool Metrics

Access connection pool statistics:

```java
SimplePoolMetrics metrics = client.getPoolMetrics();
if (metrics != null) {
    logger.info(metrics.getSummary());
}
```

### Shutdown

Stop background threads during application shutdown:

```java
client.shutdown();
```

## Best Practices

1.  **Use SimplePool API**: Prefer `borrow()`/`restore()` over deprecated `borrowConnection()`/`returnConnection()`.
2.  **Release Connections**: Always restore tokens in a `finally` block to avoid leaking connections.
3.  **Timeouts**: Always set timeouts for your requests to prevent indefinite hanging.
4.  **HTTP/2**: Enable HTTP/2 for better performance (multiplexing) if your infrastructure supports it.
5.  **Singleton**: Use the singleton `Http2Client.getInstance()` rather than creating new instances.
6.  **Shutdown**: Call `client.shutdown()` during application shutdown to stop health check threads.

## Related Documentation

- [SimplePool Migration Guide](simplepool-migration.md)
- [SimplePool Design](../../design/client-simplepool.md)
