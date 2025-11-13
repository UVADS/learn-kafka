# Learn Apache Kafka

<img src="https://us.ovhcloud.com/sites/default/files/styles/desktop_full_width/public/2024-01/kafka.webp" style="align:right;float:right;max-width:40%;" alt="Apache Kafka Logo" />

Learn how to work with Apache Kafka and distributed logs.

## Why use a distributed log?

Distributed logs (like Apache Kafka) provide several key advantages over traditional messaging systems:

### **Decoupling and Flexibility**
- **Producers and consumers are independent**: Data producers (like our Bluesky and Wikipedia streamers) don't need to know about or wait for consumers. They simply write to the log and continue.
- **Multiple consumers, same data**: Different consumers can read the same stream at their own pace, enabling multiple downstream systems to process the same events independently.

### **Durability and Replayability**
- **Persistent storage**: Events are stored on disk, not just in memory, so data survives system crashes and restarts.
- **Replay capability**: You can reprocess historical data by resetting consumer offsets, which is invaluable for debugging, testing, and recovering from processing errors.

### **Scalability and Performance**
- **High throughput**: Distributed logs are optimized for high-volume, sequential writes, making them ideal for event streaming workloads.
- **Horizontal scaling**: You can partition topics across multiple brokers to handle increasing load.

### **Fault Tolerance**
- **Replication**: With replication factors (like `replication_factor=3` in our examples), data is copied across multiple brokers, ensuring availability even if some brokers fail.
- **Consumer groups**: Multiple consumers in the same group can share the processing load, and if one fails, others automatically take over its partitions.

### **Real-time Processing**
- **Low latency**: Events can be processed as soon as they arrive, enabling real-time analytics, monitoring, and alerting.
- **Ordering guarantees**: Within a partition, messages maintain their order, which is crucial for event-sourced systems and maintaining data consistency.

In this project, we demonstrate these concepts by streaming real-time events from Bluesky and Wikipedia into Kafka, then processing them with independent consumers that can scale, replay, and handle failures gracefully.

## What do you store in Kafka?

Kafka is designed to store **events**—immutable records of something that happened at a specific point in time. Unlike traditional databases that store current state, Kafka stores a sequence of events that represent changes or occurrences.

### Common Use Cases:

- **Event Streams** - Real-time events from applications, services, or external systems (like Wikipedia edits or Bluesky posts in this project)
- **Change Data Capture (CDC)** - Records of changes to database tables, enabling replication and audit trails
- **Logs and Audit Trails** - Immutable logs of system activity, user actions, or transactions
- **Event Sourcing** - Storing all changes to application state as a sequence of events, allowing you to reconstruct state at any point in time
- **Metrics and Monitoring** - Time-series data from applications, infrastructure, or IoT devices
- **Message Queues** - Decoupling services by passing messages between producers and consumers

### What Makes Events Different?

Events are **immutable**—once written, they cannot be changed or deleted (within the retention period). This makes Kafka ideal for:
- **Audit compliance** - Creating tamper-proof records
- **Debugging** - Replaying events to understand what happened
- **Analytics** - Processing historical data to discover patterns
- **Replay** - Reprocessing events with new logic or fixing bugs

In this project, we store event streams: Wikipedia edit events and Bluesky post events. Each event represents something that happened (an edit was made, a post was created) and includes all relevant context in the message value.

## Basic Concepts

- **Brokers** - Kafka receives, stores, and distributes messages via a cluster of brokers, which essentially perform identical tasks. One broker is elected leader of the cluster.
- **Topics** - Kafka stores messages in a topic, which is identified by name. Topics can be partitioned across multiple brokers, can have replication across multiple brokers, can have a retention policy based on time or storage capacity.
- **Producers** - Applications that write messages to Kafka topics. Producers are responsible for choosing which partition to write to (via the message key) and can batch messages for better throughput. In this project, `wikipedia-producer.py` and `bluesky-producer.py` are examples of producers that stream real-time events.
- **Consumers** - Applications that read messages from Kafka topics. Consumers subscribe to one or more topics and process messages sequentially. They track their position in the log using offsets, allowing them to resume from where they left off after restarts.
- **Consumer Groups** - A collection of consumers that work together to consume messages from a topic. Kafka automatically distributes partitions among consumers in the same group, enabling parallel processing and load balancing. If a consumer fails, its partitions are automatically reassigned to other consumers in the group.

## Examples

### Setup

To get started:

1. **Create a Python virtual environment** (using `virtualenv` or `pipenv`).
2. **Run a local Kafka cluster using Docker**. The stack included in this repository creates a 3-node cluster, a web interface, and a Redis database.

    ```
    docker compose up -d
    ```

To see the web UI for Kafka, visit **http://127.0.0.1:8080/**

### Wikipedia Event Stream

The Wikipedia event stream has approximately 2-5 messages per second, and consists of 4 distinct event types.

To explore log messages from Wikipedia, run:

```
python wikipedia-producer.py
```

Let that run for a length of time, and explore the new topic created within "Topics" of the Kafka UI. Click through to see individual messages.

Note that every log entry in Kafka (regardless of topic) has three elements:

- `TIMESTAMP` - Datetime when the log entry was written. This is not a data field that can be modified or set when producing messages.
- `KEY` - (typically) a unique identifier for each message, although uniqueness is not required.
- `VALUE` - the actual text or JSON content of the log message.

Next, run the `wikipedia-consumer.py` script, which will ingest messages from the topic, and parse their event type. This script also uses the Redis database to store a count for each event type processed.

Let the script run for a few minutes (or let it catch up entirely with the produced messages) and then press Ctrl+C to break. You will then see an output from Redis of how many events were processed, by type.

### Bluesky Firehose

The Bluesky firehose is a much higher throughput data stream. With approximately 45M users, this open stream has approximately 3 event types. Generally it emits 40-100 messages per second, though this can raise significantly under user load.

To listen to the data stream and produce messages into Kafka, run the Bluesky producer:

```
python bluestream-producer.py`
```

Let it run for a few minutes. Note in the code (on line 61) that the event stream is already scoped to only POSTS within Bluesky, not other event families.

In a separate terminal (with virtual environment activated) you can then run the `bluesky-consumer.py` which will do the same sort of tracking by event type, which are stored in Redis.

Press Ctrl+C to interrupt the script, and examine the counts by event type `create`, `delete`, and `update`.

### Backpressure

When producing messages to Kafka from a realtime data stream, there is always the possibility of not keeping up with the data source. Imagine your data source (like the Bluesky firehose) emits 100 messages/sec, but your producer can only send 50 messages/sec to Kafka. This state of being "behind" is called backpressure, and it means that messages are accumulating in memory or being lost, which is clearly undesirable.

> Kafka backpressure is a flow control mechanism that prevents a producer from being overwhelmed when the data source sends messages faster than the producer can write them to Kafka.

To alleviate backpressure when producing messages, the developer or data scientist has several options:

### **1. Producer Batching**
- **Batch multiple messages**: Configure the Kafka producer to batch messages together before sending, reducing network overhead and increasing throughput.
- **Configuration**: Set `linger.ms` (how long to wait for more messages) and `batch.size` (maximum batch size in bytes) in producer configuration.
- **Example**: The `bluesky-producer.py` script uses `linger.ms: 5` and `batch.size: 100000` to batch messages efficiently.
- **Benefit**: Can significantly increase throughput by reducing per-message overhead.

### **2. Buffering and Queuing**
- **Decouple receipt from production**: Use an in-memory queue (like Python's `queue.Queue`) to buffer messages as they arrive from the data source, allowing the producer to continue receiving messages while sending them to Kafka.
- **Queue size limits**: Set maximum queue size to prevent memory issues. When the queue is full, you can either drop messages, slow down message receipt, or implement backpressure signals.
- **Example**: The `backpressure-threaded.py` script uses a queue with a maximum size of 10,000 messages to handle spikes in message arrival.

### **3. Parallel Production**
- **Multiple producer instances**: Run multiple producer instances, each handling a portion of the data stream.
- **Threading/Multiprocessing**: Use multiple worker threads or processes within a single producer to send messages concurrently.
- **Benefit**: Can significantly increase production throughput without requiring multiple producer instances.

### **4. Optimize Producer Configuration**
- **Performance tuning**: Configure producer settings for better throughput, such as increasing buffer sizes, enabling compression, or adjusting acknowledgment settings.
- **Remove bottlenecks**: Identify slow operations (serialization, network calls) and optimize them.
- **Async operations**: Use asynchronous I/O for network operations to avoid blocking the producer.

### **5. Handle Producer Errors Gracefully**
- **Retry logic**: Implement retry mechanisms for failed message sends.
- **Error handling**: Log and handle producer errors without stopping the entire stream.
- **Flush on shutdown**: Always call `producer.flush()` when shutting down to ensure all buffered messages are sent.


### **Practical Example**

The `bluesky-producer.py` script demonstrates several of these strategies:
- Uses producer batching with `linger.ms` and `batch.size` configuration
- Implements proper error handling and flush on shutdown
- Tracks message production rate to monitor throughput

The `backpressure-threaded.py` script shows a more advanced approach:
- Uses a message queue to buffer incoming messages from the data source
- Spawns multiple worker threads for parallel processing and production
- Tracks statistics to monitor receive rate vs. production rate
- Implements queue size limits to prevent memory exhaustion
- Handles backpressure by dropping messages when the queue is full (with logging)

To see backpressure in action, run a producer and observe the message rate. When the data source rate exceeds the production rate, you may see messages accumulating in producer buffers or queues, indicating backpressure.

## Cleaning Up

After working with these producers and consumers, be sure to take down your Kafka cluster with this Docker command:

```
docker compose down
```

## Resources

