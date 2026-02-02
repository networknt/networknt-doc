# Decryptor

The `decryptor` module provides a mechanism to decrypt sensitive configuration values (like passwords, tokens, and keys) at runtime. This allows you to check in your configuration files with encrypted values, enhancing security.

## Configuration

The decryptor is configured in `config.yml`.

| Property | Description | Default |
| :--- | :--- | :--- |
| `decryptorClass` | The implementation class of the `Decryptor` interface. | `com.networknt.decrypt.AutoAESDecryptor` |

 **Note**: The default class name in newer versions might be `AutoAESSaltDecryptor`. The `Config` module loads this class to handle any value starting with `CRYPT:`.

## Implementations

The module provides several implementations of the `Decryptor` interface.

### `AutoAESSaltDecryptor`

This is the default and recommended implementation. It uses AES encryption with a salt.

-   **Key Source**: Checks the environment variable `LIGHT_4J_CONFIG_PASSWORD`.
-   **Fallback**: If the environment variable is not set, it defaults to the password `light` (useful for testing).
-   **Usage**: Locate the sensitive value in your config file (e.g., `values.yml` or `secret.yml`) and replace the plain text with `CRYPT:encoded_string`.

### `ManualAESSaltDecryptor`

This implementation prompts the user to enter the password in the console/terminal during server startup. This is useful for local development or environments where environment variables cannot be securely set.

## Encryption Utility

To encrypt your secrets, you can use the `light-encryptor` utility tool.

1.  Clone `https://github.com/networknt/light-encryptor`.
2.  Run the utility (Java jar) with your master password and the clear text string.
3.  The tool will output the encrypted string (e.g., `CRYPT:fa343a...`).
4.  Copy this string into your configuration file.

## Deployment

1.  **Generate Encrypted Values**: Use `light-encryptor` to generate `CRYPT:...` strings.
2.  **Update Config**: Put these strings in `secret.yml` or other config files.
3.  **Set Master Password**: In your deployment environment (Kubernetes, VM), set the `LIGHT_4J_CONFIG_PASSWORD` environment variable to your master password.
    -   In Kubernetes, use a Secret mapped to an environment variable.

```yaml
env:
  - name: LIGHT_4J_CONFIG_PASSWORD
    valueFrom:
      secretKeyRef:
        name: my-app-secret
        key: master-password
```

## Creating a Review Decryptor

If you need a custom encryption algorithm (e.g., integrating with a cloud KMS or HashiCorp Vault), you can implement the `com.networknt.decrypt.Decryptor` interface and update `config.yml` to point to your class.
