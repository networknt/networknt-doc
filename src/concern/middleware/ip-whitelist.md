# IP Whitelist

The `ip-whitelist` middleware handler is designed to secure specific endpoints (e.g., health checks, metric endpoints, admin screens) by allowing access only from trusted IP addresses. This is often used for endpoints that cannot be easily secured via OAuth 2.0 or other standard authentication mechanisms, or as an additional layer of security.

It serves as a critical component in scenarios where services like Consul, Prometheus, or internal orchestration tools need access to technical endpoints without user-level authentication.

## Requirements

The handler supports:
*   **Per-Path Configuration:** Rules are defined for specific url path prefixes.
*   **IPv4 and IPv6:** Full support for both IP versions.
*   **Flexible matching:**
    *   **Exact match:** e.g., `127.0.0.1` or `FE45:00:00:000:0:AAA:FFFF:0045`
    *   **Wildcard match:** e.g., `10.10.*.*` or `FE45:00:00:000:0:AAA:FFFF:*`
    *   **CIDR Notation (Slash):** e.g., `127.0.0.48/30` or `FE45:00:00:000:0:AAA:FFFF:01F4/127`
*   **Default Policy:** Configurable default behavior (allow or deny) when no specific rule is matched.

## Configuration

The configuration is managed via `whitelist.yml`.

### Configuration Options

*   **`enabled`**: (boolean) Enable or disable the handler globally. Default `true`.
*   **`defaultAllow`**: (boolean) Determines the behavior for the IP addresses listed in the configuration.
    *   **`true` (Whitelist Mode - Most Common):** If `defaultAllow` is true, the IPs listed for a path are **ALLOWED**. Any IP accessing that path that is *not* in the list is **DENIED**. If a path is *not* defined in the config, access is **ALLOWED** globally for that path.
    *   **`false` (Blacklist Mode):** If `defaultAllow` is false, the IPs listed for a path are **DENIED**. Any IP accessing that path that is *not* in the list is **ALLOWED**. If a path is *not* defined in the config, access is **DENIED** globally for that path.
*   **`paths`**: A map where keys are request path prefixes and values are lists of IP patterns.

### Example `whitelist.yml` (Template)

```yaml
# IP Whitelist configuration

# Indicate if this handler is enabled or not.
enabled: ${whitelist.enabled:true}

# Default allowed or denied behavior.
defaultAllow: ${whitelist.defaultAllow:true}

# List of path prefixes and their access rules.
paths: ${whitelist.paths:}
```

### Configuring via `values.yml`

You can define the rules in `values.yml` using either YAML or JSON format.

#### YAML Format
Suitable for file-based configuration.

```yaml
whitelist.enabled: true
whitelist.defaultAllow: true
whitelist.paths:
  /health:
    - 127.0.0.1
    - 10.10.*.*
  /prometheus:
    - 192.168.1.5
    - 10.10.*.*
  /admin:
    - 127.0.0.1
```

#### JSON Format
Suitable for config servers or environment variables where a single string is required.

```yaml
whitelist.paths: {"/health":["127.0.0.1","10.10.*.*"],"/prometheus":["192.168.1.5"]," /admin":["127.0.0.1"]}
```

## Logic Flow

1.  **Extract IP:** The handler extracts the source IP address from the request.
2.  **Match Path:** It checks if the request path starts with any of the configured prefixes.
3.  **Find ACL:** If a matching path prefix is found, it retrieves the corresponding Access Control List containing rules (IP patterns).
4.  **Evaluate Rules:**
    *   It iterates through the rules for the IP version (IPv4/IPv6).
    *   If a rule matches the source IP:
        *   It returns `!rule.isDeny()`. (In `defaultAllow=true` mode, rules are allow rules, so `deny` is false, returning true).
    *   If no rule matches the source IP:
        *   It returns `!defaultAllow`. (In `defaultAllow=true` mode, if IP is not in the allow list, it returns false (deny)).
5.  **No Path Match:** If the request path is not configured:
    *   It returns `defaultAllow`. (In `defaultAllow=true` mode, unknown paths are open).

## Error Handling

If access is denied, the handler terminates the exchange and returns an error.

*   **Status Code:** 403 Forbidden
*   **Error Code:** `ERR10049`
*   **Message:** `INVALID_IP_FOR_PATH`
*   **Description:** `Peer IP %s is not in the whitelist for the endpoint %s`

## Usage

1.  Add the `ip-whitelist` module dependency to your `pom.xml`.
2.  Add `com.networknt.whitelist.WhitelistHandler` to your `handler.yml` chain.
3.  Configure `whitelist.yml` (or `values.yml`) with your specific IP rules.
