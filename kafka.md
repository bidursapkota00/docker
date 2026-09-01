# Apache Kafka Complete Guide

![Bidur Sapkota](https://www.bidursapkota.com.np/images/gravatar.webp "Bidur Sapkota - Developer")&nbsp;[Bidur Sapkota](https://www.bidursapkota.com.np/)

## Table of Contents

1. [Introducing Kafka](#introducing-kafka)
2. [Architecture & Components](#architecture--components)
3. [Installation & Setup](#installation--setup)
4. [Topics & Partitions](#topics--partitions)
5. [Producers](#producers)
6. [Consumers & Consumer Groups](#consumers--consumer-groups)
7. [Brokers & Clusters](#brokers--clusters)
8. [Message Offsets & Retention](#message-offsets--retention)
9. [Serialization & Schema Registry](#serialization--schema-registry)
10. [Kafka Connect](#kafka-connect)
11. [Kafka Streams](#kafka-streams)
12. [Docker Compose Setup](#docker-compose-setup)
13. [Kafka with Python](#kafka-with-python)
14. [Configuration & Tuning](#configuration--tuning)
15. [Security](#security)
16. [Monitoring & Debugging](#monitoring--debugging)

---

## Introducing Kafka

Apache Kafka is a distributed event streaming platform designed for high-throughput, fault-tolerant, real-time data pipelines. Unlike traditional message queues (RabbitMQ, ActiveMQ) that delete messages after consumption, Kafka persists messages to disk and allows multiple consumers to read the same data independently.

Kafka lets you publish and subscribe to streams of events, store events durably and reliably for as long as you want, process streams of events in real time or retrospectively, and decouple producers from consumers so systems can evolve independently.

Common use cases include real-time analytics pipelines, event-driven microservices, log aggregation, change data capture (CDC), activity tracking, and stream processing.

Building blocks of Kafka are:

- **Broker**: A single Kafka server that stores data and serves client requests.
- **Cluster**: A group of brokers working together for scalability and fault tolerance.
- **Topic**: A named category or feed to which messages are published. Think of it as a log file.
- **Partition**: A topic is split into partitions for parallelism. Each partition is an ordered, immutable sequence of messages.
- **Producer**: An application that publishes (writes) messages to topics.
- **Consumer**: An application that subscribes to (reads) messages from topics.
- **Consumer Group**: A group of consumers that coordinate to consume a topic, with each partition assigned to exactly one consumer in the group.
- **Offset**: A sequential ID for each message within a partition. Consumers track offsets to know where they left off.
- **ZooKeeper / KRaft**: Coordination service for cluster metadata. KRaft (Kafka Raft) is the built-in replacement for ZooKeeper, available since Kafka 3.3 and the default since Kafka 3.6.

---

## Architecture & Components

### How Kafka Works

A producer sends a message to a **topic**. The topic is divided into **partitions** spread across **brokers**. Each partition is an append-only log. Consumers read from partitions, tracking their position via **offsets**. Messages are not deleted after consumption — they remain until the configured retention period expires.

### Core Architecture

```
Producer ──► Broker 1 ──► Topic A ──► Partition 0 ──► Consumer Group 1
                                  ──► Partition 1 ──► Consumer Group 1
         ──► Broker 2 ──► Topic A ──► Partition 2 ──► Consumer Group 2
                      ──► Topic B ──► Partition 0 ──► Consumer Group 2
         ──► Broker 3  (replicas for fault tolerance)
```

### Key Concepts

| Concept           | Description                                                                          |
| ----------------- | ------------------------------------------------------------------------------------ |
| Broker            | A single Kafka server. Each broker stores a subset of partitions                      |
| Topic             | Logical channel for messages. Analogous to a database table                           |
| Partition         | Ordered, immutable log. Enables parallelism and is the unit of replication             |
| Replication Factor| Number of copies of each partition across brokers for fault tolerance                  |
| Leader            | The broker that handles all reads and writes for a partition                           |
| Follower          | A replica that passively replicates data from the leader                              |
| ISR               | In-Sync Replicas. Followers that are fully caught up with the leader                  |
| Offset            | Sequential position of a message within a partition (starts at 0)                     |
| Consumer Group    | Consumers that share the workload of consuming a topic                                |
| Commit Log        | The underlying storage model — data is appended sequentially and read by offset       |

### Message Anatomy

A Kafka message (record) contains:

| Field      | Description                                                        |
| ---------- | ------------------------------------------------------------------ |
| Key        | Optional. Determines which partition the message is routed to       |
| Value      | The actual data payload (bytes)                                     |
| Timestamp  | When the message was produced or ingested by the broker             |
| Headers    | Optional key-value metadata pairs                                   |
| Offset     | Assigned by the broker after the message is written to a partition  |

Messages with the same key always go to the same partition, guaranteeing ordering for that key.

---

## Installation & Setup

### Prerequisites

Kafka requires Java 11 or higher.

```bash
java -version                            # Verify Java is installed
```

### Download & Install Kafka

```bash
# Download Kafka (check https://kafka.apache.org/downloads for latest version)
curl -LO https://downloads.apache.org/kafka/3.9.0/kafka_2.13-3.9.0.tgz
tar -xzf kafka_2.13-3.9.0.tgz
cd kafka_2.13-3.9.0
```

`2.13` is the Scala version Kafka was compiled with. `3.9.0` is the Kafka version. The extracted directory contains `bin/` (scripts), `config/` (configuration files), and `libs/` (jar files).

### Start Kafka in KRaft Mode (No ZooKeeper)

KRaft is the modern, recommended way to run Kafka without the external ZooKeeper dependency.

```bash
# Generate a unique cluster ID
KAFKA_CLUSTER_ID="$(bin/kafka-storage.sh random-uuid)"

# Format the storage directory
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server.properties

# Start Kafka
bin/kafka-server-start.sh config/kraft/server.properties
```

`kafka-storage.sh random-uuid` generates a unique identifier for your cluster. `format` initializes the log directories with the cluster metadata. `kafka-server-start.sh` launches the broker. In KRaft mode, the broker handles its own metadata management.

### Start Kafka with ZooKeeper (Legacy)

```bash
# Terminal 1: Start ZooKeeper
bin/zookeeper-server-start.sh config/zookeeper.properties

# Terminal 2: Start Kafka
bin/kafka-server-start.sh config/server.properties
```

ZooKeeper manages broker metadata, topic configurations, and leader elections. It runs on port `2181` by default. Kafka connects to it on startup. ZooKeeper is deprecated starting Kafka 3.5 and will be fully removed in Kafka 4.0.

### Verify Installation

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

`--bootstrap-server localhost:9092` tells the CLI tool where to find the Kafka broker. Port `9092` is the default Kafka listener port. If this returns without error, Kafka is running.

### Install with Homebrew (macOS)

```bash
brew install kafka
# KRaft mode
brew services start kafka

# Or start manually
/opt/homebrew/opt/kafka/bin/kafka-server-start /opt/homebrew/etc/kafka/kraft/server.properties
```

---

## Topics & Partitions

A topic is a named stream of messages. Each topic is divided into partitions for parallel processing. Messages within a partition are strictly ordered by offset.

### Creating Topics

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic orders \
  --partitions 3 \
  --replication-factor 1
```

`--topic orders` names the topic. `--partitions 3` splits the topic into 3 partitions, allowing up to 3 consumers in a group to read in parallel. `--replication-factor 1` stores 1 copy of each partition (no redundancy). For production, use `--replication-factor 3` with at least 3 brokers. The replication factor cannot exceed the number of brokers.

### Listing Topics

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### Describing Topics

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders
```

`describe` shows the partition count, replication factor, leader broker for each partition, and the ISR (in-sync replicas) list.

### Modifying Topics

```bash
# Increase partitions (cannot decrease)
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter \
  --topic orders \
  --partitions 6
```

Increasing partitions changes the key-to-partition mapping, which can break ordering guarantees for keyed messages. You cannot reduce the number of partitions — this is an irreversible operation.

### Deleting Topics

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic orders
```

### Partition Assignment

| Scenario              | Partition Assignment                                           |
| --------------------- | -------------------------------------------------------------- |
| Key is `null`         | Round-robin across partitions (or sticky partitioning)          |
| Key is provided       | `hash(key) % num_partitions` — same key always goes to same partition |
| Custom partitioner    | You define the logic in your producer code                      |

Choosing the right number of partitions is important. More partitions enable higher throughput and more parallel consumers but increase memory usage on brokers, end-to-end latency, and recovery time after broker failures.

---

## Producers

A producer is a client application that publishes messages to one or more topics.

### Console Producer

```bash
# Start interactive producer
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic orders
> {"orderId": 1, "item": "laptop"}
> {"orderId": 2, "item": "phone"}
```

Each line you type becomes a message. Press `Ctrl+C` to exit. Messages are sent with a `null` key by default.

### Producer with Keys

```bash
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --property parse.key=true \
  --property key.separator=:
> user-1:{"orderId": 1, "item": "laptop"}
> user-2:{"orderId": 2, "item": "phone"}
> user-1:{"orderId": 3, "item": "tablet"}
```

`parse.key=true` tells the producer to split each line into key and value. `key.separator=:` uses `:` as the delimiter. Messages with key `user-1` always land in the same partition, preserving order for that user.

### Key Producer Configurations

| Property                     | Default     | Description                                                     |
| ---------------------------- | ----------- | --------------------------------------------------------------- |
| `bootstrap.servers`          | —           | Comma-separated list of broker addresses                         |
| `key.serializer`             | —           | Serializer class for the message key                             |
| `value.serializer`           | —           | Serializer class for the message value                           |
| `acks`                       | `all`       | Acknowledgment level: `0`, `1`, or `all` (-1)                   |
| `retries`                    | `2147483647`| Number of retry attempts on transient failures                   |
| `batch.size`                 | `16384`     | Maximum bytes to batch before sending (per partition)            |
| `linger.ms`                  | `0`         | Time to wait for additional messages before sending a batch      |
| `compression.type`           | `none`      | Compression: `none`, `gzip`, `snappy`, `lz4`, `zstd`           |
| `max.in.flight.requests.per.connection` | `5`  | Max unacknowledged requests per connection               |
| `enable.idempotence`         | `true`      | Ensures exactly-once delivery to a partition                     |

### Acknowledgment Levels

| acks   | Behavior                                                             | Durability | Performance |
| ------ | -------------------------------------------------------------------- | ---------- | ----------- |
| `0`    | Producer does not wait for any acknowledgment                         | Lowest     | Fastest     |
| `1`    | Producer waits for the leader to write the message                    | Medium     | Medium      |
| `all`  | Producer waits for all ISR replicas to acknowledge                    | Highest    | Slowest     |

`acks=all` combined with `min.insync.replicas=2` ensures a message is written to at least 2 replicas before being acknowledged, providing strong durability.

---

## Consumers & Consumer Groups

A consumer is a client application that reads messages from topics. Consumers operate within consumer groups to share the workload.

### Console Consumer

```bash
# Read new messages (from now)
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders

# Read all messages from the beginning
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders --from-beginning

# Show keys, values, and timestamps
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic orders \
  --from-beginning \
  --property print.key=true \
  --property print.timestamp=true
```

Without `--from-beginning`, the consumer reads only new messages arriving after it starts. `--from-beginning` resets the offset to 0 and reads all retained messages.

### Consumer Groups

```bash
# Start consumer in a group
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --group order-processors
```

`--group order-processors` assigns this consumer to the `order-processors` consumer group. Kafka distributes partitions among consumers in the same group. If the topic has 3 partitions and you start 3 consumers with the same group, each consumer gets 1 partition.

### Consumer Group Rules

| Consumers in Group | Partitions | Assignment                                         |
| ------------------- | ---------- | -------------------------------------------------- |
| 1                   | 3          | 1 consumer reads all 3 partitions                   |
| 2                   | 3          | 1 consumer reads 2 partitions, 1 reads 1            |
| 3                   | 3          | 1 consumer per partition (ideal)                     |
| 4+                  | 3          | 1 consumer per partition, extra consumers sit idle   |

Adding more consumers than partitions does not increase throughput. The maximum parallelism equals the number of partitions.

### Managing Consumer Groups

```bash
# List all consumer groups
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Describe a consumer group (show lag, offsets, assignments)
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors

# Reset offsets to the beginning
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-processors \
  --topic orders \
  --reset-offsets --to-earliest --execute

# Reset offsets to a specific timestamp
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group order-processors \
  --topic orders \
  --reset-offsets --to-datetime 2025-01-01T00:00:00.000 --execute

# Delete a consumer group
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --delete --group order-processors
```

`--describe` shows the current offset, log-end offset, and **lag** (how many messages behind the consumer is) per partition. `--reset-offsets` changes where the consumer group reads from. The group must be inactive (all consumers stopped) to reset offsets. `--to-earliest` resets to the beginning. `--to-latest` skips to the end. `--to-datetime` resets to a specific point in time.

### Key Consumer Configurations

| Property                    | Default         | Description                                               |
| --------------------------- | --------------- | --------------------------------------------------------- |
| `group.id`                  | —               | Consumer group identifier                                  |
| `auto.offset.reset`         | `latest`        | What to do when no committed offset exists: `earliest`, `latest`, `none` |
| `enable.auto.commit`        | `true`          | Automatically commit offsets periodically                   |
| `auto.commit.interval.ms`   | `5000`          | Interval between auto-commits                               |
| `max.poll.records`          | `500`           | Max records returned per poll                               |
| `session.timeout.ms`        | `45000`         | Time before a consumer is considered dead                   |
| `heartbeat.interval.ms`     | `3000`          | Heartbeat frequency to the group coordinator                |

`auto.offset.reset=earliest` reads from the beginning when the group has no committed offset (first time consuming). `latest` starts from the newest messages. `enable.auto.commit=false` is recommended when you need at-least-once or exactly-once processing, allowing you to commit offsets manually after successful processing.

---

## Brokers & Clusters

A broker is a single Kafka server. A cluster is a group of brokers that work together.

### Broker Basics

Each broker is identified by a unique `broker.id`. Brokers store partition data on disk, serve producer and consumer requests, and replicate data to other brokers. A single broker can handle thousands of partitions and millions of messages per second.

### Multi-Broker Cluster (Local)

To run a multi-broker cluster locally, create separate configuration files for each broker:

```bash
# Copy config for broker 1
cp config/kraft/server.properties config/kraft/server-1.properties
# Copy config for broker 2
cp config/kraft/server.properties config/kraft/server-2.properties
```

Edit `server-1.properties`:

```properties
node.id=1
listeners=PLAINTEXT://localhost:9093
log.dirs=/tmp/kraft-logs-1
```

Edit `server-2.properties`:

```properties
node.id=2
listeners=PLAINTEXT://localhost:9094
log.dirs=/tmp/kraft-logs-2
```

```bash
# Format storage for each broker
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server-1.properties
bin/kafka-storage.sh format -t $KAFKA_CLUSTER_ID -c config/kraft/server-2.properties

# Start each broker in separate terminals
bin/kafka-server-start.sh config/kraft/server-1.properties
bin/kafka-server-start.sh config/kraft/server-2.properties
```

Each broker needs a unique `node.id`, a unique `listeners` port, and a separate `log.dirs`. All brokers must share the same cluster ID.

### Replication

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic critical-events \
  --partitions 3 \
  --replication-factor 3
```

With `--replication-factor 3`, each partition has 3 copies across 3 different brokers. One is the **leader** (handles reads/writes), and the others are **followers** (replicate data). If the leader broker fails, a follower in the ISR is promoted to leader automatically.

### Key Broker Configurations

| Property                      | Default    | Description                                                  |
| ----------------------------- | ---------- | ------------------------------------------------------------ |
| `broker.id` / `node.id`      | —          | Unique identifier for each broker                             |
| `listeners`                   | —          | Address and port the broker listens on                        |
| `log.dirs`                    | `/tmp/...` | Directory where partition data is stored                      |
| `num.partitions`              | `1`        | Default partition count for new topics                        |
| `default.replication.factor`  | `1`        | Default replication factor for new topics                     |
| `min.insync.replicas`         | `1`        | Minimum ISR count to accept a write when `acks=all`           |
| `log.retention.hours`         | `168`      | How long messages are retained (7 days)                       |
| `log.retention.bytes`         | `-1`       | Max size per partition before old segments are deleted (-1 = unlimited) |
| `log.segment.bytes`           | `1073741824`| Size of a single log segment file (1 GB)                     |

`min.insync.replicas=2` with `acks=all` means at least 2 replicas (including the leader) must acknowledge a write. If fewer replicas are in sync, the producer receives an error rather than risking data loss.

---

## Message Offsets & Retention

### Offsets

Every message in a partition is assigned a sequential offset starting from 0. Offsets are:

- **Per partition**: Offset 5 in partition 0 is a completely different message from offset 5 in partition 1.
- **Immutable**: Once assigned, an offset never changes.
- **Consumer-tracked**: Each consumer group tracks its committed offset per partition.

### Retention Policies

Kafka offers two retention strategies:

| Policy          | Property                 | Default  | Description                                                  |
| --------------- | ------------------------ | -------- | ------------------------------------------------------------ |
| Time-based      | `log.retention.hours`    | `168`    | Delete messages older than 7 days                             |
| Size-based      | `log.retention.bytes`    | `-1`     | Delete oldest messages when partition exceeds a size limit     |
| Compact         | `log.cleanup.policy`     | `delete` | Keep only the latest value per key (used for changelogs)      |

### Configuring Retention

```bash
# Set retention to 3 days for a specific topic
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name orders \
  --add-config retention.ms=259200000

# Set retention to 1 GB per partition
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name orders \
  --add-config retention.bytes=1073741824

# View topic configuration overrides
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --describe --entity-type topics --entity-name orders
```

`retention.ms` is in milliseconds (259200000 ms = 3 days). Topic-level configs override broker-level defaults.

### Log Compaction

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic user-profiles \
  --partitions 3 \
  --replication-factor 1 \
  --config cleanup.policy=compact
```

With `cleanup.policy=compact`, Kafka periodically removes older messages with the same key, keeping only the latest value. This is ideal for changelog-style topics where you only need the current state per key (e.g., user profiles, configuration).

---

## Serialization & Schema Registry

### Serialization

Kafka stores messages as raw bytes. Producers serialize data before sending, and consumers deserialize after receiving. Common formats:

| Format     | Pros                                         | Cons                                      |
| ---------- | -------------------------------------------- | ----------------------------------------- |
| String     | Simple, human-readable                       | No schema, no type safety                  |
| JSON       | Human-readable, flexible                     | Verbose, no built-in schema enforcement    |
| Avro       | Compact binary, schema evolution support     | Requires Schema Registry                   |
| Protobuf   | Compact binary, strong typing, cross-language| Requires Schema Registry                   |

### Schema Registry

The Schema Registry (provided by Confluent) stores and serves Avro, JSON Schema, and Protobuf schemas. It ensures producers and consumers agree on the data format and supports safe schema evolution.

```
Producer ──► Schema Registry (validate schema) ──► Kafka Broker
                                                      │
Consumer ◄── Schema Registry (fetch schema)    ◄──────┘
```

### Key Schema Registry Concepts

| Concept              | Description                                                    |
| -------------------- | -------------------------------------------------------------- |
| Subject              | A named scope for schemas, typically `<topic>-key` or `<topic>-value` |
| Schema ID            | Unique numeric ID assigned when a schema is registered          |
| Compatibility        | Rules for schema evolution: `BACKWARD`, `FORWARD`, `FULL`, `NONE` |

`BACKWARD` compatibility means new schema can read data written with the old schema (safe to add optional fields, remove fields with defaults). `FORWARD` means old schema can read data written with the new schema. `FULL` is both backward and forward compatible.

### Schema Registry REST API

```bash
# List all subjects
curl http://localhost:8081/subjects

# Get latest schema for a subject
curl http://localhost:8081/subjects/orders-value/versions/latest

# Register a new schema
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "{\"type\": \"record\", \"name\": \"Order\", \"fields\": [{\"name\": \"orderId\", \"type\": \"int\"}, {\"name\": \"item\", \"type\": \"string\"}]}"}' \
  http://localhost:8081/subjects/orders-value/versions
```

The Schema Registry runs on port `8081` by default.

---

## Kafka Connect

Kafka Connect is a framework for streaming data between Kafka and external systems (databases, file systems, search indexes) without writing code. It uses connectors — plugins that define how to read from a source or write to a sink.

### Concepts

| Term               | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| Source Connector    | Reads data from an external system into Kafka topics          |
| Sink Connector      | Writes data from Kafka topics to an external system           |
| Task               | The unit of parallelism within a connector                    |
| Worker             | The JVM process that runs connectors and tasks                |
| Standalone Mode    | Single worker, suitable for development and testing           |
| Distributed Mode   | Multiple workers, production-grade with fault tolerance        |

### Standalone Mode

```bash
# Start Connect in standalone mode
bin/connect-standalone.sh config/connect-standalone.properties connector.properties
```

### Example: File Source Connector

```properties
# file-source.properties
name=file-source
connector.class=FileStreamSource
tasks.max=1
file=/tmp/input.txt
topic=file-topic
```

This reads lines from `/tmp/input.txt` and publishes each line as a message to `file-topic`.

### Example: File Sink Connector

```properties
# file-sink.properties
name=file-sink
connector.class=FileStreamSink
tasks.max=1
file=/tmp/output.txt
topics=file-topic
```

This writes messages from `file-topic` to `/tmp/output.txt`.

### Distributed Mode REST API

```bash
# Start Connect in distributed mode
bin/connect-distributed.sh config/connect-distributed.properties

# List active connectors
curl http://localhost:8083/connectors

# Create a connector
curl -X POST -H "Content-Type: application/json" \
  --data '{
    "name": "jdbc-source",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
      "connection.url": "jdbc:postgresql://localhost:5432/mydb",
      "connection.user": "user",
      "connection.password": "pass",
      "topic.prefix": "db-",
      "mode": "incrementing",
      "incrementing.column.name": "id"
    }
  }' \
  http://localhost:8083/connectors

# Check connector status
curl http://localhost:8083/connectors/jdbc-source/status

# Delete a connector
curl -X DELETE http://localhost:8083/connectors/jdbc-source
```

The Connect REST API runs on port `8083`. Distributed mode stores connector configs and offsets in internal Kafka topics, enabling fault tolerance and scalability.

### Popular Connectors

| Connector             | Type   | Description                                   |
| --------------------- | ------ | --------------------------------------------- |
| JDBC Source/Sink       | Both   | Stream data to/from relational databases       |
| Debezium              | Source | Change Data Capture from MySQL, PostgreSQL, MongoDB |
| Elasticsearch Sink    | Sink   | Index data into Elasticsearch                   |
| S3 Sink               | Sink   | Write data to AWS S3 as files                   |
| MongoDB Source/Sink    | Both   | Stream data to/from MongoDB                     |

---

## Kafka Streams

Kafka Streams is a client library for building stream processing applications that read from and write to Kafka topics. It runs inside your application (no separate cluster needed), unlike Apache Flink or Spark Streaming.

### Key Concepts

| Concept              | Description                                                      |
| -------------------- | ---------------------------------------------------------------- |
| KStream              | An unbounded, continuously updating stream of records             |
| KTable               | A changelog stream where each key's latest value is the current state |
| GlobalKTable         | Like KTable but replicated to all application instances            |
| Windowing            | Grouping events by time intervals (tumbling, hopping, session)     |
| State Store          | Local storage for intermediate processing results                  |
| Exactly-Once (EOS)   | Processing guarantee that prevents duplicates                      |

### Stream Operations

| Operation       | Description                                                |
| --------------- | ---------------------------------------------------------- |
| `filter`        | Keep only records matching a condition                      |
| `map` / `mapValues` | Transform keys and/or values                           |
| `flatMap`       | Transform each record into zero or more records             |
| `groupByKey`    | Group records by their existing key                         |
| `aggregate`     | Combine grouped records into a single result                |
| `join`          | Combine two streams/tables by key                           |
| `branch`        | Split a stream into multiple streams based on predicates    |
| `to`            | Write results to an output topic                            |

### Java Example

```java
StreamsBuilder builder = new StreamsBuilder();

KStream<String, String> orders = builder.stream("orders");

orders
    .filter((key, value) -> value.contains("laptop"))
    .mapValues(value -> value.toUpperCase())
    .to("laptop-orders");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

This reads from `orders`, filters for messages containing "laptop", transforms values to uppercase, and writes results to `laptop-orders`. Kafka Streams handles partitioning, fault tolerance, and state management automatically.

---

## Docker Compose Setup

The easiest way to run Kafka locally for development is with Docker Compose. This section provides a ready-to-use setup.

### Single Broker (KRaft)

```yaml
# compose.yaml
services:
  kafka:
    image: apache/kafka:3.9.0
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-data:/var/lib/kafka/data

volumes:
  kafka-data:
```

`KAFKA_PROCESS_ROLES: broker,controller` runs both roles in the same process (combined mode for development). `KAFKA_ADVERTISED_LISTENERS` is the address clients use to connect. `KAFKA_CONTROLLER_QUORUM_VOTERS` lists the controller nodes for KRaft consensus. Replication factors are set to `1` since there is only one broker.

### Multi-Broker with Kafka UI

```yaml
# compose.yaml
services:
  kafka-1:
    image: apache/kafka:3.9.0
    container_name: kafka-1
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-1-data:/var/lib/kafka/data

  kafka-2:
    image: apache/kafka:3.9.0
    container_name: kafka-2
    ports:
      - "9093:9092"
    environment:
      KAFKA_NODE_ID: 2
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-2-data:/var/lib/kafka/data

  kafka-3:
    image: apache/kafka:3.9.0
    container_name: kafka-3
    ports:
      - "9094:9092"
    environment:
      KAFKA_NODE_ID: 3
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka-1:9093,2@kafka-2:9093,3@kafka-3:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    volumes:
      - kafka-3-data:/var/lib/kafka/data

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3

volumes:
  kafka-1-data:
  kafka-2-data:
  kafka-3-data:
```

```bash
docker compose up -d                     # Start the cluster
docker compose ps                        # Verify all services are running
docker compose down -v                   # Stop and remove everything
```

The Kafka UI is accessible at `http://localhost:8080` and provides a web interface to browse topics, messages, consumer groups, and brokers.

---

## Kafka with Python

The `confluent-kafka` library is the recommended Python client. It wraps the high-performance C library `librdkafka`.

### Install

```bash
pip install confluent-kafka
```

### Producer

```python
from confluent_kafka import Producer
import json

config = {
    "bootstrap.servers": "localhost:9092",
    "client.id": "python-producer"
}

producer = Producer(config)

def delivery_report(err, msg):
    if err:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()} [{msg.partition()}] @ offset {msg.offset()}")

# Produce messages
for i in range(10):
    data = {"orderId": i, "item": f"product-{i}"}
    producer.produce(
        topic="orders",
        key=str(i),
        value=json.dumps(data),
        callback=delivery_report
    )

# Wait for all messages to be delivered
producer.flush()
```

`Producer(config)` creates a producer instance. `produce()` is asynchronous — it queues the message internally. `callback` is called once the broker acknowledges the message. `flush()` blocks until all queued messages are delivered. Always call `flush()` before exiting.

### Consumer

```python
from confluent_kafka import Consumer
import json

config = {
    "bootstrap.servers": "localhost:9092",
    "group.id": "python-consumers",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": True
}

consumer = Consumer(config)
consumer.subscribe(["orders"])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            print(f"Error: {msg.error()}")
            continue

        key = msg.key().decode("utf-8") if msg.key() else None
        value = json.loads(msg.value().decode("utf-8"))
        print(f"Key: {key}, Value: {value}, Partition: {msg.partition()}, Offset: {msg.offset()}")

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

`consumer.subscribe(["orders"])` subscribes to the topic. `poll(timeout=1.0)` fetches a single message, waiting up to 1 second. `consumer.close()` commits final offsets and leaves the consumer group cleanly. Always close the consumer in a `finally` block.

### Manual Offset Commit

```python
config = {
    "bootstrap.servers": "localhost:9092",
    "group.id": "python-consumers",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": False            # Disable auto-commit
}

consumer = Consumer(config)
consumer.subscribe(["orders"])

try:
    while True:
        msg = consumer.poll(timeout=1.0)
        if msg is None:
            continue
        if msg.error():
            continue

        # Process the message
        process(msg)

        # Commit only after successful processing
        consumer.commit(asynchronous=False)

except KeyboardInterrupt:
    pass
finally:
    consumer.close()
```

Manual commits give you control over when offsets are saved. `asynchronous=False` blocks until the commit is confirmed. This provides at-least-once delivery: if processing succeeds but the commit fails, the message will be reprocessed.

---

## Configuration & Tuning

### Broker-Level Configuration

Edit `server.properties` or pass as environment variables in Docker.

```properties
# Basics
broker.id=0
listeners=PLAINTEXT://0.0.0.0:9092
log.dirs=/var/lib/kafka/data

# Topic defaults
num.partitions=3
default.replication.factor=3
min.insync.replicas=2

# Retention
log.retention.hours=168
log.retention.bytes=-1
log.segment.bytes=1073741824

# Performance
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
```

### Topic-Level Overrides

```bash
# Set topic-level configs
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name orders \
  --add-config retention.ms=86400000,max.message.bytes=10485760

# Remove a topic-level override (revert to broker default)
bin/kafka-configs.sh --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name orders \
  --delete-config retention.ms
```

Topic-level configs override broker defaults for that specific topic only.

### Performance Tuning Guidelines

| Goal                    | Configuration                                                      |
| ----------------------- | ------------------------------------------------------------------ |
| Higher throughput       | Increase `batch.size`, `linger.ms`, use `compression.type=lz4`     |
| Lower latency           | Set `linger.ms=0`, `acks=1`, reduce `batch.size`                   |
| Stronger durability     | Set `acks=all`, `min.insync.replicas=2`, `replication.factor=3`    |
| More parallel consumers | Increase topic partition count                                      |
| Reduce disk usage       | Enable compression, lower `retention.ms`                            |

Throughput and latency are a trade-off. Batching (`linger.ms > 0`) improves throughput but adds latency. Compression reduces network and disk usage at the cost of CPU. `lz4` offers the best balance of speed and compression ratio for most workloads.

---

## Security

### Authentication Methods

| Method    | Description                                                         |
| --------- | ------------------------------------------------------------------- |
| SASL/PLAIN | Username/password authentication. Simple but transmits in cleartext |
| SASL/SCRAM | Username/password with salted challenge-response (more secure)      |
| SSL/TLS    | Mutual TLS authentication using certificates                        |
| SASL/GSSAPI| Kerberos-based authentication for enterprise environments           |

### Enabling SSL/TLS

```properties
# Broker config
listeners=SSL://0.0.0.0:9093
ssl.keystore.location=/var/kafka/ssl/kafka.keystore.jks
ssl.keystore.password=keystorepass
ssl.key.password=keypass
ssl.truststore.location=/var/kafka/ssl/kafka.truststore.jks
ssl.truststore.password=truststorepass
ssl.client.auth=required
```

`ssl.client.auth=required` enforces mutual TLS — clients must present a certificate.

### ACLs (Authorization)

```bash
# Allow user "app-producer" to write to topic "orders"
bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:app-producer \
  --operation Write \
  --topic orders

# Allow user "app-consumer" to read from topic "orders" in a group
bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --add \
  --allow-principal User:app-consumer \
  --operation Read \
  --topic orders \
  --group order-processors

# List ACLs for a topic
bin/kafka-acls.sh --bootstrap-server localhost:9092 --list --topic orders

# Remove an ACL
bin/kafka-acls.sh --bootstrap-server localhost:9092 \
  --remove \
  --allow-principal User:app-producer \
  --operation Write \
  --topic orders
```

ACLs require an authorizer to be configured in the broker (`authorizer.class.name`). ACLs control which principals (users/services) can perform which operations (read, write, create, delete) on which resources (topics, consumer groups, cluster).

### Security Best Practices

- Always enable encryption (SSL/TLS) for production clusters
- Use SASL/SCRAM or mutual TLS for authentication
- Apply least-privilege ACLs for all producer and consumer applications
- Rotate credentials and certificates regularly
- Encrypt data at rest using filesystem-level or cloud-managed encryption
- Use separate credentials for each application

---

## Monitoring & Debugging

### Key Metrics to Monitor

| Metric                                 | What It Tells You                                    |
| -------------------------------------- | ---------------------------------------------------- |
| Under-replicated partitions             | Partitions with fewer ISR members than expected — risk of data loss |
| Consumer lag                            | How far behind a consumer group is from the latest message |
| Request latency (produce / fetch)       | Time brokers take to handle requests                  |
| Bytes in/out per second                 | Network throughput per broker                         |
| Active controller count                 | Should always be exactly 1 in the cluster              |
| ISR shrinks/expands per second          | Frequent changes indicate broker instability           |
| Log flush latency                       | Disk write performance                                |

### Checking Cluster Health

```bash
# Describe the cluster
bin/kafka-metadata.sh --snapshot /var/lib/kafka/data/__cluster_metadata-0/00000000000000000000.log --cluster-id

# List broker IDs
bin/kafka-broker-api-versions.sh --bootstrap-server localhost:9092

# Check under-replicated partitions
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# Check unavailable partitions
bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --unavailable-partitions
```

`--under-replicated-partitions` lists partitions where one or more replicas are not in sync. `--unavailable-partitions` lists partitions with no active leader — these cannot serve reads or writes.

### Consumer Lag

```bash
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
```

Output columns:

| Column          | Description                                     |
| --------------- | ----------------------------------------------- |
| TOPIC           | Topic name                                       |
| PARTITION       | Partition number                                 |
| CURRENT-OFFSET  | Last committed offset by the consumer             |
| LOG-END-OFFSET  | Latest offset in the partition                    |
| LAG             | Difference (LOG-END-OFFSET - CURRENT-OFFSET)     |
| CONSUMER-ID     | Identifier of the consumer reading this partition |

A growing LAG means consumers are not keeping up with producers. Fix by adding more consumers (up to the partition count), increasing `max.poll.records`, or optimizing processing logic.

### Monitoring Tools

| Tool             | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| Kafka UI         | Web UI for browsing topics, messages, and consumer groups     |
| Confluent Control Center | Enterprise monitoring and management (Confluent Platform) |
| Prometheus + JMX Exporter | Export Kafka JMX metrics to Prometheus            |
| Grafana          | Dashboard visualization for Kafka metrics from Prometheus     |
| Burrow           | LinkedIn's consumer lag monitoring tool                       |
| AKHQ             | Open-source Kafka GUI with topic browsing and management      |

### JMX Monitoring

Kafka exposes metrics via JMX (Java Management Extensions). To enable:

```bash
KAFKA_JMX_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false" \
bin/kafka-server-start.sh config/kraft/server.properties
```

Use the Prometheus JMX Exporter to scrape these metrics and visualize them in Grafana dashboards.

### Debugging Common Issues

| Issue                        | Cause                                               | Solution                                      |
| ---------------------------- | --------------------------------------------------- | --------------------------------------------- |
| Messages not appearing       | Wrong topic name, producer errors, serialization     | Check producer logs, verify topic exists        |
| Consumer not receiving       | Wrong group ID, offset reset, partition assignment    | Check `--describe --group`, reset offsets       |
| High consumer lag             | Slow processing, too few consumers                   | Add consumers, optimize logic, increase partitions |
| Broker unreachable            | Network issues, broker crash, wrong `advertised.listeners` | Check broker logs, verify connectivity   |
| Rebalance storms              | Consumers dying frequently, long processing          | Increase `session.timeout.ms`, reduce `max.poll.records` |
| Disk full                     | Retention too long, high throughput                   | Reduce `retention.ms`, add disk, enable compression |
