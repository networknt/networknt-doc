# Kafka Streams Health Check

When operating a Kafka sidecar that runs Kafka Streams applications, it is crucial to monitor the health of these streams. The `SidecarHealthHandler` provides a health check endpoint that verifies if all registered Kafka Streams instances are in `RUNNING` or `REBALANCING` state.

## Registering Kafka Streams

To enable health monitoring for your Kafka Streams application, you must verify that your streams instance is registered with the `KafkaStreamsRegistry`.

### Usage

In your streams application startup hook or initialization logic, after starting the `KafkaStreams` instance, register it:

```java
import com.networknt.kafka.streams.KafkaStreamsRegistry;
import org.apache.kafka.streams.KafkaStreams;

// ... initialize streams ...

streams.start();

// Register the streams instance for health checks
KafkaStreamsRegistry.register("my-streams-app", streams);
```

The `SidecarHealthHandler` will automatically discover all registered streams and include them in the health check. If any registered stream is not in a healthy state, the health check endpoint will return `ERROR` (status 500 equivalent logic, though specifically returning a string).

### Example

Here is an example from `WordCountStreams`:

```java
    @Override
    public void start(String ip, int port) {
        // ... configuration ...
        wordCountStreams = new KafkaStreams(topology.build(), streamsProps);
        // ...
        wordCountStreams.start();
        
        // Registration
        KafkaStreamsRegistry.register("WordCountStreams", wordCountStreams);
    }

```

## Example Applications

There are two example applications available in the `light-example-4j` repository that demonstrate how to implement Kafka Streams with health check registration:

1. **Kafka Streams DSL (WordCount)**

   This example demonstrates a simple word count application using the high-level Kafka Streams DSL.
   
   - [Source Code](https://github.com/networknt/light-example-4j/tree/master/kafka-sidecar/kafka-streams-dsl)
   - [Health Check Implementation](https://github.com/networknt/light-example-4j/blob/master/kafka-sidecar/kafka-streams-dsl/src/main/java/com/networknt/kafka/WordCountStreams.java)

2. **Kafka Streams Processor API (UserQuery)**
   
   This example demonstrates a more complex application using the lower-level Processor API for querying user data.
   
   - [Source Code](https://github.com/networknt/light-example-4j/tree/master/kafka-sidecar/kafka-streams-processor-api)
   - [Health Check Implementation](https://github.com/networknt/light-example-4j/blob/master/kafka-sidecar/kafka-streams-processor-api/src/main/java/com/networknt/kafka/UserQueryStreams.java)

Both examples show how to:
- Configure the Kafka Streams application using `KafkaStreamsConfig`.
- Start the `KafkaStreams` instance.
- Register the instance with `KafkaStreamsRegistry` to enable health monitoring via `SidecarHealthHandler`.

