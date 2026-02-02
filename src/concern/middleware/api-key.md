# API Key Handler

For some legacy applications to migrate from a monolithic gateway to the light-gateway without changing any code on the consumer application, we need to support the API Key authentication on the light-gateway (LG) or client-proxy (LCP). The consumer application sends the API key in a header to authenticate itself on the light-gateway. Then the light-gateway will retrieve a JWT token to access the downstream API.

Only specific paths will have API Key set up, and the header name for each application might be different. To support all use cases, we add a list of maps to the configuration `apikey.yml` to `pathPrefixAuths` property.

Each config item will have `pathPrefix`, `headerName`, and `apiKey`. The handler will try to match the path prefix first and then get the input API Key from the header. After comparing with the configured API Key, the handler will return either `ERR10075` API_KEY_MISMATCH or pass the control to the next handler in the chain.

## Configuration

The configuration is managed by `ApiKeyConfig` and corresponds to `apikey.yml` (or `apikey.json`/`apikey.properties`).

### Configuration Properties

| Property | Type | Default | Description |
| :--- | :--- | :--- | :--- |
| `enabled` | boolean | `false` | Enable ApiKey Authentication Handler. |
| `hashEnabled` | boolean | `false` | If API key hash is enabled. The API key will be hashed with PBKDF2WithHmacSHA1 before it is stored in the config file. It is more secure than putting the clear text key into the config file. |
| `pathPrefixAuths` | list | `null` | A list of mappings between path prefix and the api key parameters. |

### Configuration Example (apikey.yml)

```yaml
# ApiKey Authentication Security Configuration for light-4j
# Enable ApiKey Authentication Handler, default is false.
enabled: ${apikey.enabled:false}

# If API key hash is enabled. The API key will be hashed with PBKDF2WithHmacSHA1 before it is
# stored in the config file. It is more secure than put the encrypted key into the config file.
# The default value is false. If you want to enable it, you need to use the light-hash command line tool.
hashEnabled: ${apikey.hashEnabled:false}

# path prefix to the api key mapping. It is a list of map between the path prefix and the api key
# for apikey authentication. In the handler, it loops through the list and find the matching path
# prefix. Once found, it will check if the apikey is equal to allow the access or return an error.
pathPrefixAuths: ${apikey.pathPrefixAuths:}
```

### Values Example (values.yml)

To enable the handler and configure paths, add the following to your `values.yml`:

```yaml
# apikey.yml
apikey.enabled: true
apikey.pathPrefixAuths:
  - pathPrefix: /test1
    headerName: x-gateway-apikey
    apiKey: abcdefg
  - pathPrefix: /test2
    headerName: x-apikey
    apiKey: mykey
```

JSON format example for `pathPrefixAuths` (useful for config server):

```json
[{"pathPrefix":"/test1","headerName":"x-gateway-apikey","apiKey":"abcdefg"},{"pathPrefix":"/test2","headerName":"x-apikey","apiKey":"mykey"}]
```

### Multiple Consumers for Same Path

Most services will have multiple consumers, and each consumer might have its own API key for authentication. You can define multiple entries for the same path prefix:

```yaml
apikey.pathPrefixAuths:
  - pathPrefix: /test1
    headerName: x-gateway-apikey
    apiKey: abcdefg
  # The same prefix has another apikey header and value.
  - pathPrefix: /test1
    headerName: authorization
    apiKey: xyz
```

## Security with Hash

Storing API keys in clear text is only recommended for testing. For production, enable hashing:

1.  Enable hash in `values.yml`:
    ```yaml
    apikey.hashEnabled: true
    ```
2.  Generate the hash of your API key using the [light-hash](https://github.com/networknt/light-hash) utility.
3.  Use the generated hash string as the `apiKey` value in your configuration.

## Error Response

If the request path matches a configured prefix but the API key verification fails, the handler returns error `ERR10075`.

**Status Code**: 401
**Code**: ERR10075
**Message**: API_KEY_MISMATCH
**Description**: APIKEY from header %s is not matched for request path prefix %s.

To prevent leaking sensitive information, the expected apiKey from the config file is not revealed in the error message.

## Usage

Register the `ApiKeyHandler` in your `handler.yml` chain.

```yaml
handler.handlers:
  .
  - com.networknt.apikey.ApiKeyHandler@apikey

handler.chains.default:
  .
  - apikey
  .
```

If you are using the Unified Security Handler, you can integrate the API Key handler there as well to support multiple authentication methods (ApiKey, Basic, OAuth2, SWT) simultaneously.
