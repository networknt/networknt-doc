# Common Module

The `common` module provides shared utilities, constants, and enumerations that are used across both the client and server components of the Light-4j framework.

## Components

### ContentType

`ContentType` is an enumeration that defines standard HTTP `Content-Type` header values. It simplifies code maintenance by replacing hardcoded strings with strongly typed constants.

**Supported Types:**
-   `application/json`
-   `text/xml`
-   `application/xml`
-   `application/yaml`
-   `application/x-www-form-urlencoded`
-   `multipart/form-data`
-   ... and more.

It also includes a helper method `toContentType(String value)` to convert a string header value into the corresponding enum constant.

### SecretConstants

This class defines standard keys for sensitive values found in `secret.yml` or other configuration files. These constants ensure consistency when accessing secrets across different modules.

**Common Constants:**
-   **Certificates:** `serverKeystorePass`, `serverTruststorePass`, `clientKeystorePass`, etc.
-   **OAuth2:** `authorizationCodeClientSecret`, `clientCredentialsClientSecret`, etc.
-   **Infrastructure:** `consulToken`, `emailPassword`.

### DecryptUtil

`DecryptUtil` is a critical utility for handling encrypted sensitive values within configuration files (like `secret.yml`). It enables the "Encrypt-Once, Deploy-Everywhere" pattern where secrets are stored in version control in an encrypted format and decrypted only at runtime.

#### How it works

1.  **Iterative Decryption:** The `decryptMap` method recursively iterates through a configuration map (nested maps and lists).
2.  **Detection:** It identifies values that start with the `CRYPT` prefix.
3.  **Decryption:** When an encrypted value is found, it uses the configured `Decryptor` implementation to decrypt it.

#### Usage

The framework typically handles this automatically when loading `secret.yml`. However, if you are loading custom configuration files with encrypted values, you can use it directly:

```java
Map<String, Object> myConfig = Config.getInstance().getJsonMapConfig("my-config");
if(myConfig != null) {
    // recursively decrypt all values in the map
    DecryptUtil.decryptMap(myConfig);
}
```

#### Decryptor Configuration

`DecryptUtil` relies on a `Decryptor` implementation being registered in `service.yml`. The default implementation typically uses AES encryption.

For example, in `service.yml`:

```yaml
singletons:
  - com.networknt.decrypt.Decryptor:
      - com.networknt.decrypt.AESDecryptor
```

For more details on generating keys and encrypting values, refer to the [Decryptor Module](../module/decryptor.md).
