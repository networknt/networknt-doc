# Data Source

The `data-source` module provides a flexible way to configure and manage JDBC data sources in light-4j applications. It wraps [HikariCP](https://github.com/brettwooldridge/HikariCP) to provide high-performance connection pooling.

## Overview

Unlike the simple `javax.sql.DataSource` configuration in `service.yml`, the `data-source` module offers:
-   Support for multiple data sources.
-   Detailed HikariCP configuration.
-   Encrypted password support.
-   Separation of concerns (DataSource config vs. Service config).

## Dependency

Add the following dependency to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>data-source</artifactId>
    <version>${version.light-4j}</version>
</dependency>
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
</dependency>
<!-- Add your JDBC driver dependency here (e.g., mysql-connector-java, postgresql) -->
```

## Configuration

Configuration is primarily handled in `datasource.yml`.

### `datasource.yml`

You can define multiple data sources by giving them unique names (e.g., `PostgresDataSource`, `H2DataSource`).

```yaml
# Example Configuration
PostgresDataSource:
  DriverClassName: org.postgresql.ds.PGSimpleDataSource
  jdbcUrl: jdbc:postgresql://postgresdb:5432/mydb
  username: postgres
  # Password can be encrypted using the light-4j decryptor
  password: CRYPT:747372737...
  maximumPoolSize: 10
  connectionTimeout: 30000
  # Optional HikariCP settings
  settings:
    autoCommit: true
  # Optional driver parameters
  parameters:
    ssl: 'false'

H2DataSource:
  DriverClassName: org.h2.jdbcx.JdbcDataSource
  jdbcUrl: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
  username: sa
  password: sa
```

### `service.yml`

You need to register the data sources in `service.yml` so they can be injected or retrieved via `SingletonServiceFactory`.

```yaml
singletons:
  # Register the Decryptor for encrypted passwords
  - com.networknt.utility.Decryptor:
      - com.networknt.decrypt.AESDecryptor

  # Register specific Data Sources
  - com.networknt.db.GenericDataSource:
      - com.networknt.db.GenericDataSource:
          - java.lang.String: PostgresDataSource
      - com.networknt.db.GenericDataSource:
          - java.lang.String: H2DataSource

  # Or specific implementations if available
  - javax.sql.DataSource:
      - com.networknt.db.GenericDataSource::getDataSource:
          - java.lang.String: PostgresDataSource
```

## Usage

### Getting a DataSource

You can retrieve the `DataSource` instance using `SingletonServiceFactory`.

```java
import com.networknt.service.SingletonServiceFactory;
import com.networknt.db.GenericDataSource;
import javax.sql.DataSource;

public class MyDao {
    public void query() {
        // Option 1: Get the GenericDataSource wrapper
        GenericDataSource genericDs = SingletonServiceFactory.getBean(GenericDataSource.class); 
        // Note: If you have multiple, you might need getBean(Class, name) or check how it's registered.
        
        // Option 2: If registered as javax.sql.DataSource
        DataSource ds = SingletonServiceFactory.getBean(DataSource.class);
        
        try (Connection conn = ds.getConnection()) {
            // ... use connection
        }
    }
}
```

### Multiple Data Sources

If you have multiple data sources, it is best to register them by name in `service.yml` and retrieve them by name or class if you created specific subclasses.

```java
// Retrieving by name if registered as such (requires custom registration logic or specific class beans)
GenericDataSource postgres = (GenericDataSource) SingletonServiceFactory.getBean("PostgresDataSource");
```

## Features

### Encrypted Passwords

You can use the `light-4j` encryption tool to encrypt database passwords. Prefix the encrypted string with `CRYPT:` in `datasource.yml` or `secret.yml`.

### Setting Configuration

The `settings` map in `datasource.yml` matches the `setXxx` methods of `HikariDataSource`. For example:

```yaml
settings:
  autoCommit: false
  idleTimeout: 60000
```
