# LDAP Utility

The `ldap-util` is a utility module in the light-4j framework that provides a helper class `LdapUtil` to interact with LDAP servers for authentication and authorization. It simplifies common tasks such as binding (verifying credentials) and searching for user attributes (like `memberOf` for group-based authorization).

## Features

*   **Authentication**: Authenticates users against an LDAP server using simple binding.
*   **Authorization**: Retrieves user attributes (specifically `memberOf`) from the LDAP server to support group-based access control.
*   **Secure Connection**: Supports both standard LDAP and LDAPS (LDAP over SSL) using a custom socket factory.
*   **Configuration Driven**: Behavior is controlled via externalized configuration (`ldap.yml`).

## Configuration

The `ldap-util` module uses a configuration file named `ldap.yml`.

### Configuration Options

The following properties can be defined in `ldap.yml`:

*   **`uri`**: The LDAP server URI (e.g., `ldap://localhost:389` or `ldaps://localhost:636`).
*   **`domain`**: The LDAP domain name.
*   **`principal`**: The user principal (DN or username) used for binding to perform searches.
*   **`credential`**: The password for the binding principal.
*   **`searchFilter`**: The search filter used to find user entries (e.g., `(&(objectClass=user)(sAMAccountName={0}))`). The `{0}` placeholder is replaced by the username during runtime.
*   **`searchBase`**: The base DN (Distinguished Name) where searches will start.

### Example `ldap.yml`

```yaml
# LDAP server connection and search settings.
---
# The LDAP server uri.
uri: ${ldap.uri:ldap://localhost:389}
# The LDAP domain name.
domain: ${ldap.domain:}
# The user principal for binding (authentication).
principal: ${ldap.principal:cn=admin,dc=example,dc=org}
# The user credential (password) for binding.
credential: ${ldap.credential:admin}
# The search filter (e.g., (&(objectClass=user)(sAMAccountName={0}))).
searchFilter: ${ldap.searchFilter:(&(objectClass=person)(uid={0}))}
# The search base DN (Distinguished Name).
searchBase: ${ldap.searchBase:dc=example,dc=org}
```

## Usage

The primary interface is the `com.networknt.ldap.LdapUtil` class.

### Authentication

To verify a user's password:

```java
boolean isAuthenticated = LdapUtil.authenticate("username", "password");
```

This method first searches for the user's DN using the configured `principal`, and then attempts to bind with the found DN and the provided password.

### Authorization

To retrieve a user's groups (`memberOf` attribute):

```java
Set<String> groups = LdapUtil.authorize("username");
```

This method uses the configured `principal` to search for the user and extracts all values from the `memberOf` attribute.

## Integration

The `ldap-util` is used by other modules in the light platform that require LDAP integration, such as the `ldap-security` handler for protecting endpoints with LDAP credentials.

Whenever `LdapConfig.load()` is called, the module automatically registers itself with the `ModuleRegistry` so that its configuration can be viewed via the [Server Info](/concern/admin/server-info.md) endpoint.
