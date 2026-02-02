# Basic Auth

The Basic Authentication middleware provides a mechanism to authenticate users using the standard HTTP Basic Authentication scheme (Authorization header with `Basic base64(username:password)`).

While OAuth 2.0 (JWT) is the recommended security standard for microservices, Basic Auth is useful for:
-   IoT devices that do not support OAuth flows.
-   Integrating legacy systems.
-   Simple authentication requirements where a full OAuth provider is overkill.

## Configuration

The configuration is managed by `BasicAuthConfig` and corresponds to `basic-auth.yml` (or `basic-auth.json`/`basic-auth.properties`).
The settings are injected under the `basic` key in `values.yml`.

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `false` | Enable or disable the Basic Auth Handler. |
| `enableAD` | boolean | `true` | Enable LDAP (Active Directory) authentication if the password in config is empty. |
| `allowAnonymous` | boolean | `false` | Allow requests without Authorization header if the path matches the `anonymous` user's paths. |
| `allowBearerToken` | boolean | `false` | Allow requests with `Bearer` token to pass through if the path matches the `bearer` user's paths. Useful for proxying mixed auth traffic. |
| `users` | list/map | `null` | A list of user definitions containing `username`, `password`, and allowed `paths`. |

### User Definition

Each user object in the `users` list/map contains:
-   `username`: The login username.
-   `password`: The password (clear text or encrypted).
-   `paths`: A list of URL path prefixes this user is allowed to access.

Special usernames:
-   `anonymous`: Used when `allowAnonymous` is true. Defines paths accessible without authentication.
-   `bearer`: Used when `allowBearerToken` is true. Defines paths accessible with a Bearer token (verification delegated to downstream or another handler).

### Configuration Example (basic-auth.yml)

```yaml
# Basic Authentication Security Configuration for light-4j
enabled: ${basic.enabled:false}
enableAD: ${basic.enableAD:true}
allowAnonymous: ${basic.allowAnonymous:false}
allowBearerToken: ${basic.allowBearerToken:false}
users: ${basic.users:}
```

### Values Example (values.yml)

You typically configure users in `values.yml`:

```yaml
basic.enabled: true
basic.users:
  - username: user1
    password: user1pass
    paths:
      - /v1/address
  - username: user2
    # Encrypted password
    password: CRYPT:0754fbc37347c136be7725cbf62b6942:71756e13c2400985d0402ed6f49613d0
    paths:
      - /v2/pet
      - /v2/address
  # Paths allowed for anonymous access (if allowAnonymous: true)
  - username: anonymous
    paths:
      - /v1/party
      - /info
```

## Features

### Static User Authentication
Verify `username` and `password` against the configured list. If matched, checks if the request path matches one of the user's allowed `paths`.

### LDAP Authentication
If `enableAD` is true and a configured user has an **empty password**, the handler attempts to authenticate against the configured LDAP server (using `LdapUtil`). Context:
1.  User `user1` is defined in `basic.users` with `password: ""` (empty).
2.  Incoming request has `Basic base64(user1:secret)`.
3.  Handler skips local password check and calls `LdapUtil.authenticate("user1", "secret")`.
4.  If LDAP auth succeeds, it checks the allowed `paths` for `user1` in the config.

### Anonymous Access
If `allowAnonymous` is true, requests without an Authorization header are checked against the paths defined for the special `anonymous` user.
-   If path matches `anonymous.paths`, request proceeds.
-   If path does not match, returns `ERR10071` (NOT_AUTHORIZED_REQUEST_PATH) or `ERR10002` (MISSING_AUTH_TOKEN).

### Bearer Token Pass-through
If `allowBearerToken` is true, requests with `Authorization: Bearer <token>` are checked against the paths defined for the special `bearer` user.
-   If path matches `bearer.paths`, request proceeds (skipping Basic Auth check).
-   This is useful in a gateway/proxy where some endpoints use Basic Auth and others use OAuth2, and you want to route them through the same chain conditionally.

## Usage

Register the `BasicAuthHandler` in your `handler.yml` chain.

```yaml
handler.handlers:
  .
  - com.networknt.basicauth.BasicAuthHandler@basic

handler.chains.default:
  .
  - basic
  .
```

## Error Codes

-   `ERR10002`: MISSING_AUTH_TOKEN - The basic authentication header is missing (and anonymous not allowed).
-   `ERR10046`: INVALID_BASIC_HEADER - The format of the basic authentication header is not valid.
-   `ERR10047`: INVALID_USERNAME_OR_PASSWORD - Invalid username or password.
-   `ERR10071`: NOT_AUTHORIZED_REQUEST_PATH - Authenticated user is not authorized for the requested path.
-   `ERR10072`: BEARER_USER_NOT_FOUND - Bearer token allowed but `bearer` user configuration missing.
