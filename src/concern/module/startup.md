# Service Startup and Configuration Injection

In the previous section, we introduced the `config` module of the light-4j platform and demonstrated how `values.yml` can be used to override default externalized configuration properties across all configuration files.

While using a local `values.yml` file is suitable for a standalone light-4j server, configuration can also be dynamically injected during startup via the `light-config-server` (part of the light-portal). To enable a light-4j service to leverage the `light-config-server`, specific bootstrap configuration files and environment variables must be configured.

## Prerequisites

### startup.yml

The `startup.yml` file is used to replace the default configuration loader with a customized implementation, such as one that fetches configuration from a remote server instead of the local filesystem.

Below is an example of a `startup.yml` file configured for an `ai-gateway` development environment. It specifies the `DefaultConfigLoader`, which attempts to load configuration from the light-portal and falls back to the local filesystem for cached versions.

```
# This is the config file to replace the default config loader to a customized one.
# For example, load the config files from the config server instead of filesystem.
# The following dummy entry is just to prevent the warning message during startup.
# dummy: dummyEntry
# For real config loader config, please follow the format below with your implementation.
# light-4j provides a DefaultConfigLoader that loads config files from the light-portal
# and fallback to local file systems for cached config files.
# The following is the config for the DefaultConfigLoader.
configLoaderClass: com.networknt.server.DefaultConfigLoader
# All variables below can be used to look up the config files for a particular service,
# but only the productId is required. If other variables are not provided, the default
# "current" value will be used from the config server.
host: dev.lightapi.net
serviceId: com.networknt.ai.gateway-1.0.0
envTag: dev
# Indicate to the config server we prefer YAML format over JSON format. default is JSON.
acceptHeader: application/yaml
# The connect timeout for bootstrap from the config server. Default is 3 seconds.
timeout: 3000

```

### Environment Variables

Several environment variables must be set before starting the service. These variables facilitate the bootstrap process, establish a connection to the configuration server, and manage the loading and caching of configuration files.

#### Standard Configuration
The following environment variables are typically required for a development instance:

```
LIGHT_PORTAL_AUTHORIZATION=Bearer eyJraWQiOiJBWnAwMk1DdWNydVpmRkpieXlGd3VnIiwiYWxnIjoiUlMyNTYifQ.eyJpc3MiOiJ1cm46Y29tOm5ldHdvcmtudDpvYXV0aDI6djEiLCJhdWQiOiJ1cm46Y29tLm5ldHdvcmtudCIsImV4cCI6MjA4ODEwODIyNywianRpIjoiUlc4X1g5UHNmTUIxcmpIeUZ5Z0JKUSIsImlhdCI6MTc3Mjc0ODIyNywibmJmIjoxNzcyNzQ4MTA3LCJ2ZXIiOiIxLjAiLCJjaWQiOiIwMTljOTI3My0yNjYzLTdhOWUtODJmNC05NGY5ZjVmNzljM2EiLCJzY3AiOlsicG9ydGFsLnciLCJwb3J0YWwuciJdLCJyb2xlcyI6ImFHOXpkQzFoWkcxcGJpQjFjMlZ5IiwidXNlcklkIjoiMDE5NjRiMDUtNTUzMi03Yzc5LThjZGUtMTkxZGNiZDQyMWI4IiwiaG9zdCI6IjAxOTY0YjA1LTU1MmEtN2M0Yi05MTg0LTY4NTdlN2YzZGM1ZiIsImVtYWlsIjoic3RldmUuaHVAc3VubGlmZS5jb20iLCJlaWQiOiJzaDM1In0.uM7iTEHnOZKVtZk6evtar-IGhoh615er9UjQ0ozHfyLlYEBsUZK8KIk4h4R4gwgPX_ldbNBLcGT7bn2pd6JOwlfBM_VDtRZbtVjHRefaK1uYHOdRg9Ckn8xTFdYa9HCQkge7cRlgvx-HsHZn3404BeKa1YjBiJKCGRVHe7QXkKT5Mhsgma3MpQtQFnCVdswrJL1QHP2M0mjwgEH6Y1FnuBPey5IZYXkl_I4ecN2D4XM5I15sv7bvGVGLpdczJJP_YP9L-wFE_lGomst6_2FHQvF-VSry2eDtFS0o_lvW2MKrBB2roeqwZTRf5H9RiHSLVNjb4vvbQTvVfWEG3y7zGQ;CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION=/home/steve/workspace/light-gateway/config/ai-gateway/config/bootstrap.truststore;CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD=password;CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME=false;LIGHT_ENV=dev
```

#### Containerized Environments (Docker)
When running within a Docker container, the following variables should be exported to the container environment:

```
export CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION=/config/bootstrap.truststore
export CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD=password
export CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME=false
export LIGHT_PORTAL_AUTHORIZATION=eyJraWQiOiIxMDAiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJ1cm46Y29tOm5ldHdvcmtudDpvYXV0aDI6djEiLCJhdWQiOiJ1cm46Y29tLm5ldHdvcmtudCIsImV4c
CI6MTk0MTEyMjc3MiwianRpIjoiTkc4NWdVOFR0SEZuSThkS2JsQnBTUSIsImlhdCI6MTYyNTc2Mjc3MiwibmJmIjoxNjI1NzYyNjUyLCJ2ZXJzaW9uIjoiMS4wIiwiY2xpZW50X2lkIjoiZjdkNDIzNDgtYzY0Ny
00ZWZiLWE1MmQtNGM1Nzg3NDIxZTcyIiwic2NvcGUiOiJwb3J0YWwuciBwb3J0YWwudyIsInNlcnZpY2UiOiIwMTAwIn0.Q6BN5CGZL2fBWJk4PIlfSNXpnVyFhK6H8X4caKqxE1XAbX5UieCdXazCuwZ15wxyQJg
WCsv4efoiwO12apGVEPxIc7gpvctPrRIDo59dmTjfWH0p3ja0Zp8tYLD-5Sh65WUtJtkvPQk0uG96JJ64Da28lU4lGFZaCvkaS-Et9Wn0BxrlCE5_ta66Qc9t4iUMeAsAHIZJffOBsREFhOpC0dKSXBAyt9yuLDuD
t9j7HURXBHyxSBrv8Nj_JIXvKhAxquffwjZF7IBqb3QRr-sJV0auy-aBQ1v8dYuEyIawmIP5108LH8QdH-K8NkI1wMnNOz_wWDgixOcQqERmoQ_Q3g
export LIGHT_ENV=dev
export LIGHT_4J_CONFIG_DIR=/config
export LIGHT_CONFIG_SERVER_URI=https://localhost:8443
```

#### Orchestrated Environments (Kubernetes)
In Kubernetes, these variables should be defined within the deployment manifest. 

#### Local Development
For local development, it is often preferred to manage these variables outside of global profiles (like `.profile` or `.bashrc`) to avoid conflicts between different light-4j projects. However, a common baseline configuration might look like this:

```
export CONFIG_SERVER_CLIENT_TRUSTSTORE_LOCATION=/home/steve/networknt/light-4j/server/src/main/resources/config/bootstrap.truststore
export CONFIG_SERVER_CLIENT_TRUSTSTORE_PASSWORD=password
export CONFIG_SERVER_CLIENT_VERIFY_HOST_NAME=false
export LIGHT_PORTAL_AUTHORIZATION=Bearer eyJraWQiOiIxMDAiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJ1cm46Y29tOm5ldHdvcmtudDpvYXV0aDI6djEiLCJhdWQiOiJ1cm46Y29tLm5ldHdvcmtudCIsImV4c
CI6MTk0MTEyMjc3MiwianRpIjoiTkc4NWdVOFR0SEZuSThkS2JsQnBTUSIsImlhdCI6MTYyNTc2Mjc3MiwibmJmIjoxNjI1NzYyNjUyLCJ2ZXJzaW9uIjoiMS4wIiwiY2xpZW50X2lkIjoiZjdkNDIzNDgtYzY0Ny
00ZWZiLWE1MmQtNGM1Nzg3NDIxZTcyIiwic2NvcGUiOiJwb3J0YWwuciBwb3J0YWwudyIsInNlcnZpY2UiOiIwMTAwIn0.Q6BN5CGZL2fBWJk4PIlfSNXpnVyFhK6H8X4caKqxE1XAbX5UieCdXazCuwZ15wxyQJg
WCsv4efoiwO12apGVEPxIc7gpvctPrRIDo59dmTjfWH0p3ja0Zp8tYLD-5Sh65WUtJtkvPQk0uG96JJ64Da28lU4lGFZaCvkaS-Et9Wn0BxrlCE5_ta66Qc9t4iUMeAsAHIZJffOBsREFhOpC0dKSXBAyt9yuLDuD
t9j7HURXBHyxSBrv8Nj_JIXvKhAxquffwjZF7IBqb3QRr-sJV0auy-aBQ1v8dYuEyIawmIP5108LH8QdH-K8NkI1wMnNOz_wWDgixOcQqERmoQ_Q3g
export LIGHT_ENV=dev

```

### JVM Options

Project-specific variables are best managed using Java Virtual Machine (JVM) `-D` system properties passed via the command line.

```
-Dlight-4j-config-dir=config/ai-gateway/config -Dlight-config-server-uri=https://local.lightapi.net
```

- `-Dlight-4j-config-dir`: Points directly to the externalized configuration folder for a specific service.
- `-Dlight-config-server-uri`: Enables or disables the configuration server bootstrap process for the service.

### bootstrap.truststore

To securely connect to a `light-portal` instance running with a self-signed certificate, the appropriate `bootstrap.truststore` must be present in the local configuration folder. Using the example JVM options above, this file should reside in the `config/ai-gateway/config` directory alongside the `startup.yml` file.

## Startup Sequence

When a light-4j service starts with the `DefaultConfigLoader` configured in `startup.yml`, it follows a specific sequence to ensure all necessary configurations and assets are loaded before the server becomes operational.

1.  **Environment Identification**: The loader first identifies the execution environment (`light-env`) by checking system properties, environment variables, or the `envTag` in `startup.yml`. It defaults to `dev` if no value is found.
2.  **Bootstrap Context Initialization**: If a `light-config-server-uri` is detected, the loader initializes a secure HTTP client using the `bootstrap.truststore`. This allows the service to communicate securely with the light-portal.
3.  **Remote Configuration Fetching**:
    *   **Configuration Logic**: The service calls the configuration server's contexts (e.g., `/config-server/configs`) using query parameters defined in `startup.yml` (e.g., `productId`, `serviceId`, `envTag`).
    *   **Value Injection**: The retrieved values (typically from a centralized `values.yml` on the portal) are injected into the core `Config` module. This process includes clearing existing caches and ensuring encrypted values are decrypted.
4.  **Asset Retrieval**: The loader subsequently fetches static assets such as certificates (`/config-server/certs`) and other required files (`/config-server/files`), saving them directly into the externalized configuration directory (`targetConfigsDirectory`).
5.  **Local Persistence (Caching)**: To support offline restarts, the loader saves the fetched `values.yml` and static files to the local file system.
6.  **Logging Reconfiguration**: If a custom Logback configuration file is provided via the `logback.configurationFile` property, the loader resets the logging context and reapplies the fetched or local configuration.

## Fallback Mechanism

The `DefaultConfigLoader` is designed with a robust fallback mechanism to ensure service availability even if the `light-config-server` is unreachable.

*   **Config Server Missing**: If `light-config-server-uri` is not provided, the service immediately defaults to loading configurations from the local file system.
*   **Connection Failure**: If the service fails to connect to the configuration server (due to a timeout or network error), it attempts to use the locally cached `values.yml` in the `light-4j-config-dir`.
*   **Missing Cache**: If the config server is unreachable and no local cache exists, the service will throw a `RuntimeException` and terminate the startup process to prevent running with incomplete or incorrect configurations.
