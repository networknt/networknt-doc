# DB Provider

The `db-provider` module is a lightweight database provider designed to simplify the initialization of a generic SQL/JDBC data source, typically for internal services or control plane components (like `light-portal` or `light-config-server`) that need a single, primary database connection.

## Overview

Unlike the more flexible `data-source` module which is designed for multiple, named data sources, `db-provider` focuses on providing a **single, global** `HikariDataSource` initialized at startup. It is often used in conjunction with a `StartupHookProvider` to ensure the database connection is ready before the server starts handling requests.

## Configuration

The module is configured via `db-provider.yml` (or `yaml`/`json`).

| Property | Description | Default |
| :--- | :--- | :--- |
| `driverClassName` | JDBC driver class name. | `org.postgresql.Driver` |
| `jdbcUrl` | JDBC connection URL. | `jdbc:postgresql://timescale:5432/configserver` |
| `username` | Database username. | `postgres` |
| `password` | Database password (supports encryption). | `secret` |
| `maximumPoolSize` | Max connections in the pool. | `3` |

**Example `db-provider.yml`:**
```yaml
driverClassName: org.h2.Driver
jdbcUrl: jdbc:h2:mem:testdb
username: sa
password: sa
maximumPoolSize: 5
```

## Startup Hook

To use this module, you typically register the `SqlDbStartupHook` in your `service.yml`. This hook reads the configuration, creates the `HikariDataSource`, and initializes the `CacheManager`.

**service.yml:**
```yaml
startupHooks:
  - com.networknt.db.provider.SqlDbStartupHook
```

## Usage

Once initialized by the startup hook, the data source is available via the static field in `SqlDbStartupHook`.

```java
import com.networknt.db.provider.SqlDbStartupHook;
import javax.sql.DataSource;
import java.sql.Connection;

public class MyService {
    public void doSomething() {
        DataSource ds = SqlDbStartupHook.ds;
        try (Connection conn = ds.getConnection()) {
            // ... perform DB operations
        } catch (SQLException e) {
            // handle exception
        }
    }
}
```

## Key Classes

-   **`DbProviderConfig`**: Loads configuration from `db-provider.yml`.
-   **`SqlDbStartupHook`**: Initializes the global `HikariDataSource` and `CacheManager`.
-   **`DbProvider`**: Interface for provider extensions (defaulting to `DbProviderImpl`).

## When to use

-   **Use `db-provider`** if you are building a simple service or a light-4j platform component (like Config Server) that needs a single, standard SQL connection setup globally at startup.
-   **Use `data-source`** if you need multiple data sources, named data sources, or more complex configuration that doesn't rely on a specific startup hook.
