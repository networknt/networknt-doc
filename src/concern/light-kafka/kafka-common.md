# Kafka Common

The `kafka-common` module is a core shared library for `light-kafka` and its related microservices (like the kafka-sidecar). It encapsulates common configuration management, serialization/deserialization utilities, and shared constants.

## Key Features

*   **Centralized Configuration**:
    *   `KafkaProducerConfig`: Manages configuration for Kafka producers (topic defaults, serialization formats, audit settings).
    *   **Hot Reload**: Supports dynamic configuration updates for `kafka-producer.yml` and others via `Config.reload()`.
    *   **Module Registry**: Automatically registers configuration modules with the server's `ModuleRegistry` for runtime observability.
*   **Shared Utilities**:
    *   `LightSchemaRegistryClient`: A lightweight client for interacting with the Confluent Schema Registry.
    *   `AvroConverter` & `AvroDeserializer`: Helpers for handling Avro data formats.
    *   `KafkaConfigUtils`: Utilities for parsing and mapping configuration properties.

## Configuration Classes

### KafkaProducerConfig
Responsible for loading `kafka-producer.yml`. Key properties include:
*   `topic`: Default topic name.
*   `auditEnabled`: Whether to send audit logs.
*   `auditTarget`: Topic or logfile for audit data.
*   `injectOpenTracing`: Whether to inject OpenTracing headers.

### KafkaConsumerConfig
Responsible for loading `kafka-consumer.yml`.
*   `maxConsumerThreads`: Concurrency settings.
*   `topic`: List of topics to subscribe to.
*   `deadLetterEnabled`: DLQ configuration.

### KafkaStreamsConfig
Responsible for loading `kafka-streams.yml`.
*   `cleanUp`: State store cleanup settings.
*   `applicationId`: Streams application ID.

## Integration

Include `kafka-common` in your service to leverage shared Kafka capabilities:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>kafka-common</artifactId>
    <version>${version.light-kafka}</version>
</dependency>
```

## Hot Reload

Configurations in this module invoke `ModuleRegistry.registerModule` upon load and reload. This ensures that any changes pushed from the config server are immediately reflected in the application state and visible via the server's info endpoints.
