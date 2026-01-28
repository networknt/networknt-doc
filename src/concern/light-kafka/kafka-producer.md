---
description: Kafka Producer Module
---

# Kafka Producer

The `kafka-producer` module provides an abstraction for key features of the `light-kafka` ecosystem, including auditing, schema validation, and header injection. It supports publishing to Kafka with both key and value serialization via Confluent Schema Registry (Avro, JSON Schema, Protobuf).

## Core Components

### SidecarProducer
The primary producer implementation.
*   **Schema Integration**: Integrated with `SchemaRegistryClient` to handle serialization of keys and values. Caches schema lookups for performance.
*   **Audit Integration**: Automatically generates and sends audit records (success/failure) for each produced message if configured.
*   **Asynchronous**: Returns `CompletableFuture<ProduceResponse>` for non-blocking operation.
*   **Headers**: Propagates `traceabilityId` and `correlationId` into Kafka message headers for end-to-end tracing.

### NativeLightProducer
An interface extension which exposes the underlying `KafkaProducer` instance.

### SerializedKeyAndValue
Helper class that holds the serialized bytes for key and value along with target partition and headers.

## Configuration

This module relies on `kafka-producer.yml`, managed by `KafkaProducerConfig` (from the `kafka-common` module).

**Key Settings:**
*   `topic`: Default target topic.
*   `keyFormat` / `valueFormat`: Serialization format (e.g., `jsonschema`, `avro`, `string`).
*   `auditEnabled`: Toggle for audit logging.
*   `injectOpenTracing`: Optional integration with OpenTracing.

## Usage

Initialize `SidecarProducer` at application startup. It automatically loads configuration and registers itself.

```java
NativeLightProducer producer = new SidecarProducer();
producer.open();

// Usage
ProduceRequest request = ...;
producer.produceWithSchema(topic, serviceId, partition, request, headers, auditList);
```
