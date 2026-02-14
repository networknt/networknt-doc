# SimplePool Migration Guide

This guide helps migrate from deprecated `Http2Client` connection pooling methods to the new SimplePool API.

> [!IMPORTANT]
> The old `borrowConnection`/`returnConnection` methods are deprecated for removal. Migrate to `borrow()`/`restore()` for better pool management, metrics, and health checks.

## Quick Reference

| Old API (Deprecated) | New API (SimplePool) |
|---------------------|---------------------|
| `borrowConnection(uri, worker, pool, isHttp2)` | `borrow(uri, worker, pool, isHttp2)` |
| `returnConnection(connection)` | `restore(token)` |
| `ClientConnection` | `ConnectionToken` |

## Key Difference: Token-Based Pool Management

The SimplePool uses a **token pattern** instead of raw connections:

```java
// OLD: Returns ClientConnection directly
ClientConnection conn = client.borrowConnection(uri, worker, pool, true);
try {
    // use connection
} finally {
    client.returnConnection(conn);
}

// NEW: Returns ConnectionToken that wraps the connection
ConnectionToken token = client.borrow(uri, worker, pool, true);
try {
    ClientConnection conn = token.connection().getRawConnection();
    // use connection
} finally {
    client.restore(token);
}
```

**Why tokens?** Tokens track which specific borrow operation acquired a connection, enabling:
- Accurate metrics (borrow/restore counts per URI)
- Leak detection
- Proper HTTP/2 multiplexing management

---

## Migration Patterns

### Pattern 1: Simple Request

**Before:**
```java
Http2Client client = Http2Client.getInstance();
ClientConnection connection = null;
try {
    connection = client.borrowConnection(10, uri, Http2Client.WORKER, 
        Http2Client.BUFFER_POOL, OptionMap.create(UndertowOptions.ENABLE_HTTP2, true));
    
    // Send request
    connection.sendRequest(request, callback);
    
} finally {
    if (connection != null) {
        client.returnConnection(connection);
    }
}
```

**After:**
```java
Http2Client client = Http2Client.getInstance();
SimpleConnectionState.ConnectionToken token = null;
try {
    token = client.borrow(uri, Http2Client.WORKER, Http2Client.BUFFER_POOL, true);
    ClientConnection connection = token.connection().getRawConnection();
    
    // Send request
    connection.sendRequest(request, callback);
    
} finally {
    if (token != null) {
        client.restore(token);
    }
}
```

### Pattern 2: With OptionMap

**Before:**
```java
OptionMap options = OptionMap.builder()
    .set(UndertowOptions.ENABLE_HTTP2, true)
    .set(Options.KEEP_ALIVE, true)
    .getMap();
    
ClientConnection conn = client.borrowConnection(10, uri, worker, bufferPool, options);
```

**After:**
```java
OptionMap options = OptionMap.builder()
    .set(UndertowOptions.ENABLE_HTTP2, true)
    .set(Options.KEEP_ALIVE, true)
    .getMap();
    
ConnectionToken token = client.borrow(uri, worker, bufferPool, options);
ClientConnection conn = token.connection().getRawConnection();
```

### Pattern 3: HTTP/1.1 vs HTTP/2

```java
// HTTP/2 (multiplexed)
ConnectionToken token = client.borrow(uri, worker, bufferPool, true);

// HTTP/1.1 (non-multiplexed)
ConnectionToken token = client.borrow(uri, worker, bufferPool, false);
```

---

## New Features After Migration

After migrating to SimplePool, you gain access to:

### Pool Metrics
```java
SimplePoolMetrics metrics = client.getPoolMetrics();
if (metrics != null) {
    logger.info(metrics.getSummary());
}
```

### Pool Warm-Up
```java
// Pre-establish connections at startup
client.warmUpPool(URI.create("https://api.example.com:443"));
```

### Health Checks
Enable in configuration:
```yaml
client:
  request:
    healthCheckEnabled: true
    healthCheckIntervalMs: 30000
```

---

## Configuration

Enable SimplePool features in `client.yml`:

```yaml
client:
  request:
    # Pool sizing
    connectionPoolSize: 10
    connectionExpireTime: 1800000
    
    # Metrics (default: false)
    poolMetricsEnabled: true
    
    # Warm-up (default: false)
    poolWarmUpEnabled: true
    poolWarmUpSize: 2
    
    # Health checks (default: true)
    healthCheckEnabled: true
    healthCheckIntervalMs: 30000
```

---

## Backward Compatibility

During the transition period:
1. Deprecated methods internally redirect to SimplePool when possible
2. Old pool (`Http2ClientConnectionPool`) is still available as fallback
3. Both pools can coexist during migration

---

## AI Migration Instructions

For automated migration tools and AI assistants, use these patterns:

### SEARCH_PATTERN: borrowConnection call
```
\bclient\.borrowConnection\s*\([^)]*\)
```

### REPLACE_PATTERN: Convert to borrow
1. Change method name from `borrowConnection` to `borrow`
2. Change return type from `ClientConnection` to `ConnectionToken`
3. Extract connection: `token.connection().getRawConnection()`
4. Change `returnConnection(connection)` to `restore(token)`

### SEARCH_PATTERN: returnConnection call
```
\bclient\.returnConnection\s*\(\s*(\w+)\s*\)
```

### REPLACE_PATTERN: Convert to restore
```
client.restore($1Token)
```
where `$1Token` is the corresponding token variable.

### VARIABLE_RENAME
- `connection` → `token`
- `conn` → `token`
- Add `.connection().getRawConnection()` when accessing the raw connection

### IMPORT_CHANGES
Add:
```java
import com.networknt.client.simplepool.SimpleConnectionState;
```

---

## Troubleshooting

### Connection Not Returned to Pool
**Symptom:** Pool exhaustion, connections not being reused

**Cause:** `restore()` not called in finally block

**Fix:** Always use try-finally:
```java
ConnectionToken token = null;
try {
    token = client.borrow(...);
    // use connection
} finally {
    if (token != null) client.restore(token);
}
```

### Metrics Show Zero
**Cause:** Metrics not enabled

**Fix:** Add to configuration:
```yaml
client:
  request:
    poolMetricsEnabled: true
```

---

## Related Documentation

- [Http2Client Module](http2-client.md)
- [SimplePool Design](../../design/client-simplepool.md)
