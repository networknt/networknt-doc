# Kafka Consumer

The `kafka-consumer` module provides a RESTful interface for consuming records from Kafka topics. It abstracts the complexity of the native Kafka Consumer API, handling instance management, thread pooling, and record serialization/deserialization.

## Core Components

### KafkaConsumerManager
Manages the lifecycle of Kafka consumers.
*   **Instance Management**: Creates and caches `KafkaConsumer` instances based on configuration.
*   **Threading**: Uses `KafkaConsumerThreadPoolExecutor` to handle concurrent read operations.
*   **Task Scheduling**: Manages long-polling read tasks using a `DelayQueue` and `ReadTaskSchedulerThread` to efficiently handle `poll()` operations without blocking threads unnecessarily.
*   **Auto-Cleanup**: A background thread (`ExpirationThread`) automatically closes idle consumers to reclaim resources.

### LightConsumer
An interface defining the contract for consumer implementations, potentially allowing for different underlying consumer strategies (though `KafkaConsumerManager` is the primary implementation).

### KafkaConsumerReadTask
Encapsulates a single read request. It iterates over the Kafka consumer records, buffering them until the response size criteria (min/max bytes) are met or a timeout occurs.

## Configuration

This module relies on `kafka-consumer.yml`, managed by `KafkaConsumerConfig` (from the `kafka-common` module).

**Key Settings:**
*   `maxConsumerThreads`: Controls the thread pool size for consumer operations.
*   `server.id`: Unique identifier for the server instance, used for consumer naming.
*   `consumer.instance.timeout.ms`: Idle timeout for consumer instances.

## Usage

This module is typically used by the `kafka-sidecar` or other microservices that need to expose Kafka consumption over HTTP/REST.

The `KafkaConsumerManager` is usually initialized at application startup:

```java
KafkaConsumerConfig config = KafkaConsumerConfig.load();
KafkaConsumerManager manager = new KafkaConsumerManager(config);
```
