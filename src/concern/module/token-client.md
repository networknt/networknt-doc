# Token Client

The `token-client` is a Light-4j module designed to securely exchange sensitive Personally Identifiable Information (PII) for format-preserved proxy tokens (e.g., swapping a Social Security Number for a reversible proxy token like `TK-1234`). 

It serves as the highly optimized bridge between your Light-4j microservices and the persistent `light-tokenization` vault service.

## Architecture & Caching Strategy

Because Tokenization translations execute continuously during API workloads (especially when integrated within edge gateways or MCP routers), retrieving tokens rapidly is critical to preventing network I/O congestion.

To guarantee hyper-low latency without requiring complex, redundant cache code, the `token-client` fundamentally relies on a **Decorator Pattern** that natively wraps the core Light-4j `cache-manager`.

### 1. The Fallback Interface: `HttpTokenClient` 
The base engine is the `HttpTokenClient`. Using Light-4j's `Http2Client` multiplexing and `SimplePool` resource handlers, it establishes secure asynchronous connections outbound to the actual `tokenization.lightapi.net` REST endpoint to query the persistent database vault.

### 2. The Transparent Multi-Tier Decorator: `CacheTokenClient` 
Instead of forcing you to deploy a localized Cache vs a Distributed Cache, the `CacheTokenClient` simply proxies cache lookups to `CacheManager.getInstance().get("token_vault_cache")`.

This means the Token Client natively adopts your Microservice's active caching topology!
* If your `service.yml` configures `Caffeine`, the Token Client automatically operates an L1 heap-memory cache.
* If your `service.yml` configures `Hazelcast` or `Redis`, the Token Client instantly shares its resolving proxy maps across your entire Kubernetes node cluster!

## Bi-Directional Mapping Performance

When a cache miss occurs, the `HttpTokenClient` retrieves the mapping from the persistence vault. Before returning the result, the `CacheTokenClient` intercepts the exact translation and persists it **bi-directionally** back into the target `cache-manager` system:

1. `cleartext -> proxy_token` *(Accelerates future `tokenize()` requests)*
2. `reverse:proxy_token -> cleartext` *(Accelerates future `detokenize()` requests)*

Because of this symmetric pre-caching approach, nearly all subsequent token lookups bypass external network walls entirely, returning payloads in `<0.01ms`.

## Example Integration

Include the module in your target API `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>token-client</artifactId>
    <version>${project.version}</version>
</dependency>
```

Initialize the client with the fallback decorator in your handlers:

```java
import com.networknt.token.TokenClient;
import com.networknt.token.HttpTokenClient;
import com.networknt.token.CacheTokenClient;

// 1. Initialize the network client out to the persistence vault
TokenClient httpClient = new HttpTokenClient("https://tokenization.lightapi.net");

// 2. Wrap it in the Cache Manager system
TokenClient tokenClient = new CacheTokenClient(httpClient);

// 3. Execute! If not cached, it will fetch from HTTP.
String secureProxy = tokenClient.tokenize("987-65-4321", 2);

// 4. Reverse! Executes entirely in-memory if already cached.
String ssn = tokenClient.detokenize(secureProxy);
```
