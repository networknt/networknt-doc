# Kafka Streams

The `kafka-streams` module provides a lightweight wrapper around the Kafka Streams API, simplifying common tasks such as configuration loading, audit logging, and Dead Letter Queue (DLQ) integration.

## Core Components

### LightStreams
An interface that helps bootstrap a Kafka Streams application. It typically includes:
*   **startStream()**: Initializes the `KafkaStreams` instance with the provided topology and configuration. It automatically adds audit and exception handling sinks to the topology if configured.
*   **getKafkaValueByKey()**: A utility to query state stores (interactive queries) with retry logic for handling rebalances.
*   **getAllKafkaValue()**: Queries all values from a state store.

## Configuration

This module relies on `kafka-streams.yml`, managed by `KafkaStreamsConfig` (from the `kafka-common` module).

**Key Settings:**
*   `application.id`: The unique identifier for the streams application.
*   `bootstrap.servers`: Kafka cluster connection string.
*   `cleanUp`: If set to `true`, the application usually performs a local state store cleanup on startup (useful for resetting state).
*   `auditEnabled`: When true, an "AuditSink" is added to the topology to capture audit events.
*   `deadLetterEnabled`: When true, automatically configures DLQ sinks for error handling based on provided metadata.

## Usage

To use this module, implement `LightStreams` or call its default methods from your startup logic:

```java
// Load config
KafkaStreamsConfig config = KafkaStreamsConfig.load();

// Build Topology
Topology topology = ...;

// Start Stream
KafkaStreams streams = startStream(ip, port, topology, config, dlqMap, auditParentNames);
```

The module automatically handles the registration of the configuration module with the server for runtime observability.
