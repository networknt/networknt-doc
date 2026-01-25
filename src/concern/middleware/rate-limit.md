# Rate Limit

The `rate-limit` middleware handler is designed to protect your APIs from being overwhelmed by too many requests. It can be used for two primary purposes:
1.  **DDoS Protection:** Limiting concurrent requests from the public internet to prevent denial-of-service attacks.
2.  **Throttling:** Protecting slow backend services or managing API quotas for different clients or users.

This handler impacts performance slightly, so it is typically not enabled by default. It should be placed early in the middleware chain, typically after `ExceptionHandler` and `MetricsHandler`, to fail fast and still allow metrics to capture rate-limiting events.

## Dependency

To use the handler, add the following dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>rate-limit</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```

Note: The configuration class is located in the `limit-config` module.

## Handler Configuration

Register the handler in `handler.yml`:

```yaml
handlers:
  - com.networknt.limit.LimitHandler@limit

chains:
  default:
    - exception
    - metrics
    - limit
    - ...
```

## Configuration (limit.yml)

The handler is configured via `limit.yml`.

```yaml
# Rate Limit Handler Configuration

# Enable or disable the handler
enabled: ${limit.enabled:false}

# Error code returned when limit is reached. 
# Use 503 for DDoS protection (to trick attackers) or 429 for internal throttling.
errorCode: ${limit.errorCode:429}

# Default rate limit: e.g., 10 requests per second and 10000 quota per day.
rateLimit: ${limit.rateLimit:10/s 10000/d}

# If true, rate limit headers are always returned, even for successful requests.
headersAlwaysSet: ${limit.headersAlwaysSet:false}

# Key of the rate limit: server, address, client, user
# server: Shared limit for the entire server.
# address: Limit per IP address.
# client: Limit per client ID (from JWT).
# user: Limit per user ID (from JWT).
key: ${limit.key:server}

# Custom limits can be defined for specific keys
server: ${limit.server:}
address: ${limit.address:}
client: ${limit.client:}
user: ${limit.user:}

# Key Resolvers
clientIdKeyResolver: ${limit.clientIdKeyResolver:com.networknt.limit.key.JwtClientIdKeyResolver}
addressKeyResolver: ${limit.addressKeyResolver:com.networknt.limit.key.RemoteAddressKeyResolver}
userIdKeyResolver: ${limit.userIdKeyResolver:com.networknt.limit.key.JwtUserIdKeyResolver}
```

### Rate Limit Keys

#### 1. Server Key
The limit is shared across all incoming requests. You can define specific limits for different path prefixes:

```yaml
limit.server:
  /v1/address: 10/s
  /v2/address: 1000/s
```

#### 2. Address Key
The source IP address is used as the key. Each IP gets its own quota.

```yaml
limit.address:
  192.168.1.100: 10/h 1000/d
  192.168.1.102:
    /v1: 10/s
```

#### 3. Client Key
The `client_id` from the JWT token is used as the key. **Note:** `JwtVerifierHandler` must be in the chain before the `limit` handler.

```yaml
limit.client:
  my-client-id: 100/m 10000/d
```

#### 4. User Key
The `user_id` from the JWT token is used as the key. **Note:** `JwtVerifierHandler` must be in the chain before the `limit` handler.

```yaml
limit.user:
  user@example.com: 10/m 10000/d
```

## Rate Limit Headers

When a limit is reached (or if `headersAlwaysSet` is `true`), the following headers are returned:

*   **X-RateLimit-Limit:** The configured limit (e.g., `10/s`).
*   **X-RateLimit-Remaining:** The number of requests remaining in the current time window.
*   **X-RateLimit-Reset:** The number of seconds until the current window resets.
*   **Retry-After:** (Only when limit is reached) The timestamp when the client can retry.

## Module Registration

The `limit-config` module automatically registers itself with the `ModuleRegistry` when `LimitConfig.load()` is called. This allows the configuration to be visible via the [Server Info](/concern/admin/server-info.md) endpoint and supports hot reloading of configurations.
