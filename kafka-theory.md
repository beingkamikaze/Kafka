"""Patch applied: rework markdown structure for clarity and formatting."""
## Apache Kafka — Architecture, Internals & Mental Models

This document explains Apache Kafka from first principles: core concepts, internals, and mental models useful for engineers and interview preparation.

---

**Quick summary:** Kafka is a distributed, fault-tolerant, append-only commit log. Producers append records to partitions; consumers read sequentially using offsets.

---

## Table of Contents

- [Apache Kafka — Architecture, Internals & Mental Models](#apache-kafka-architecture-internals-mental-models)
- [What is Kafka?](#what-is-kafka)
- [High-level Architecture](#high-level-architecture)
- [Cluster and Brokers](#cluster-and-brokers)
- [Topics vs Partitions](#topics-vs-partitions)
- [Partition Internals](#partition-internals)
- [Replication, Leaders and ISR](#replication-leaders-and-isr)
- [Controller & Metadata](#controller-metadata)
- [Producer Architecture](#producer-architecture)
- [Partition Selection](#partition-selection)
- [Consumer Architecture](#consumer-architecture)
- [Consumer Groups & Rebalancing](#consumer-groups-rebalancing)
- [Offsets & Offset Management](#offsets-offset-management)
- [Delivery Semantics](#delivery-semantics)
- [Ordering Guarantees](#ordering-guarantees)
- [Fault Tolerance & Failure Scenarios](#fault-tolerance-failure-scenarios)
- [Scaling Kafka](#scaling-kafka)
- [Common Misconceptions (Interview Traps)](#common-misconceptions-interview-traps)
- [Mental Models Summary](#mental-models-summary)
- [Further Reading](#further-reading)
- [License](#license)

- [Kafka Topics & Partitions — Creation and Producer Partition Logic](#kafka-topics-partitions-creation-and-producer-partition-logic)
  - [Who should read this](#who-should-read-this)
  - [1. How are partitions created?](#1-how-are-partitions-created)
  - [2. How does the producer select a partition?](#2-how-does-the-producer-select-a-partition)

- [Kafka Consumer — Basics (From Zero)](#kafka-consumer-basics-from-zero)
  - [Quick summary](#quick-summary)
  - [1 — What is a Kafka consumer?](#1-—-what-is-a-kafka-consumer)
  - [2 — What does a consumer actually read?](#2-—-what-does-a-consumer-actually-read)
  - [3 — Why do consumers need offsets?](#3-—-why-do-consumers-need-offsets)
  - [4 — Offsets are NOT global](#4-—-offsets-are-not-global)
  - [5 — Consumer groups (core concept)](#5-—-consumer-groups-core-concept)
  - [6 — Partition/consumer count scenarios](#6-—-partitionconsumer-count-scenarios)
  - [7 — Consumer startup flow (important)](#7-—-consumer-startup-flow-important)
  - [8 — Minimal Spring Boot consumer example](#8-—-minimal-spring-boot-consumer-example)
  - [9 — Where to start reading (offset resolution)](#9-—-where-to-start-reading-offset-resolution)
  - [10 — Consumer poll loop (internal view)](#10-—-consumer-poll-loop-internal-view)
  - [11 — Key guarantees (at this stage)](#11-—-key-guarantees-at-this-stage)
  - [12 — Common beginner confusions](#12-—-common-beginner-confusions)
  - [13 — Mental model (lock this in)](#13-—-mental-model-lock-this-in)

- [Offset in Kafka — From Creation to Consumption](#offset-in-kafka-from-creation-to-consumption)
  - [1 — What is an offset?](#1-—-what-is-an-offset)
  - [2 — When and how is an offset CREATED?](#2-—-when-and-how-is-an-offset-created)
  - [3 — Does the producer use the offset?](#3-—-does-the-producer-use-the-offset)
  - [4 — Why do consumers need offsets?](#4-—-why-do-consumers-need-offsets-1)
  - [5 — Where are offsets stored?](#5-—-where-are-offsets-stored)
  - [6 — How does a consumer GET the offset?](#6-—-how-does-a-consumer-get-the-offset)
  - [7 — What if no offset exists?](#7-—-what-if-no-offset-exists)
  - [8 — How consumers use offsets while reading](#8-—-how-consumers-use-offsets-while-reading)
  - [9 — What does committing an offset mean?](#9-—-what-does-committing-an-offset-mean)
  - [10 — Crash scenarios](#10-—-crash-scenarios)

- [Offset Commit Strategies](#offset-commit-strategies)
  - [1 — What does committing an offset mean?](#1-—-what-does-committing-an-offset-mean-1)
  - [2 — Where are offsets committed?](#2-—-where-are-offsets-committed)
  - [3 — Two core strategies: Auto Commit vs Manual Commit](#3-—-two-core-strategies-auto-commit-vs-manual-commit)
  - [4 — Sync vs Async manual commits](#4-—-sync-vs-async-manual-commits)
  - [5 — Which offset to commit?](#5-—-which-offset-to-commit)
  - [6 — Crash ordering examples](#6-—-crash-ordering-examples)
  - [7 — Exactly-once vs at-least-once](#7-—-exactly-once-vs-at-least-once)
  - [8 — Spring Boot example (manual commit)](#8-—-spring-boot-example-manual-commit)

- [How a Kafka Consumer Decides Partition, Gets Offset, and Commits It (Java View)](#how-a-kafka-consumer-decides-partition-gets-offset-and-commits-it-java-view)
  - [Quick summary](#quick-summary-1)
  - [1 — Who decides partitions?](#1-—-who-decides-partitions)
  - [2 — Partition assignment flow (internal steps)](#2-—-partition-assignment-flow-internal-steps)
  - [3 — How the consumer gets the offset](#3-—-how-the-consumer-gets-the-offset)
  - [4 — Polling records (Java view)](#4-—-polling-records-java-view)
  - [5 — Committing offsets (best practice)](#5-—-committing-offsets-best-practice)
  - [6 — Crash scenarios (must understand)](#6-—-crash-scenarios-must-understand)
  - [7 — Java mental model (one-liner)](#7-—-java-mental-model-one-liner)
  - [8 — Interview traps (avoid these)](#8-—-interview-traps-avoid-these)

- [Consumer Group — What It Is & Why We Need It](#consumer-group-what-it-is-why-we-need-it)
  - [Quick summary](#quick-summary-2)
  - [1 — What is a consumer group?](#1-—-what-is-a-consumer-group)
  - [2 — Why do we need consumer groups?](#2-—-why-do-we-need-consumer-groups)
  - [3 — What happens without consumer groups?](#3-—-what-happens-without-consumer-groups)
  - [4 — Core rules (memorize)](#4-—-core-rules-memorize)
  - [5 — Consumer group vs partitions (visual)](#5-—-consumer-group-vs-partitions-visual)
  - [6 — Offsets and consumer groups](#6-—-offsets-and-consumer-groups)
  - [7 — Internal flow (high level)](#7-—-internal-flow-high-level)
  - [8 — Java / Spring: how groups are created and scaled](#8-—-java-spring-how-groups-are-created-and-scaled)
  - [9 — What NOT to do](#9-—-what-not-to-do)
  - [10 — Same `group.id` across microservices — why it's dangerous](#10-—-same-groupid-across-microservices-why-its-dangerous)
  - [11 — One-line interview answer](#11-—-one-line-interview-answer)

- [How Parallel Processing Is Achieved in Kafka](#how-parallel-processing-is-achieved-in-kafka)
  - [Core idea](#core-idea)
  - [Formula](#formula)
  - [Scenarios](#scenarios)
  - [Where parallelism runs](#where-parallelism-runs)
  - [Why Kafka does NOT parallelize within a partition](#why-kafka-does-not-parallelize-within-a-partition)
  - [Interview-ready one-liner](#interview-ready-one-liner)

- [Consumer Rebalancing — Mechanics & Purpose](#consumer-rebalancing-mechanics-purpose)
  - [Quick summary](#quick-summary-3)
  - [1 — What is rebalancing?](#1-—-what-is-rebalancing)
  - [2 — Why rebalancing is required](#2-—-why-rebalancing-is-required)
  - [3 — When does a rebalance occur?](#3-—-when-does-a-rebalance-occur)
  - [4 — Who coordinates rebalances?](#4-—-who-coordinates-rebalances)
  - [5 — Step-by-step rebalance flow](#5-—-step-by-step-rebalance-flow)
  - [6 — Partition assignor strategies (short)](#6-—-partition-assignor-strategies-short)
  - [7 — What happens to offsets during rebalance?](#7-—-what-happens-to-offsets-during-rebalance)
  - [8 — Why rebalancing is painful (real-world impact)](#8-—-why-rebalancing-is-painful-real-world-impact)
  - [9 — How Kafka reduces rebalance impact](#9-—-how-kafka-reduces-rebalance-impact)
  - [10 — Operational recommendations](#10-—-operational-recommendations)
  - [11 — Mental model (lock this)](#11-—-mental-model-lock-this)
  - [12 — Interview one-liner](#12-—-interview-one-liner)

- [Resuming Consumption After Rebalancing — What Happens?](#resuming-consumption-after-rebalancing-what-happens)
- [What Does "Commit Offset" Mean? — Practical Explanation](#what-does-commit-offset-mean-practical-explanation)

- [What Happens to Messages After Retention Time?](#what-happens-to-messages-after-retention-time)
- [Handling Duplicates in Consumers](#handling-duplicates-in-consumers)

## What is Kafka?

Apache Kafka is a distributed, fault-tolerant, append-only event streaming platform (a distributed commit log).

- Producers append events
- Consumers pull events sequentially
- Data is stored durably on disk
- Designed for horizontal scale and high throughput

Common use cases: event-driven systems, streaming ETL, log aggregation, integration, and real-time analytics.

---

## High-level Architecture

A Kafka deployment exposes a single logical system backed by a cluster of brokers.

```
Kafka Cluster
├─ Broker A
├─ Broker B
└─ Broker C
```

---

## Cluster and Brokers

- Cluster: manages metadata and coordinates brokers; does not itself hold partition data.
- Broker: a Kafka server that stores partition segments on disk and serves client requests.

Key broker facts:
- Unique `broker.id`
- Stores partitions on disk and participates in leader election

---

## Topics vs Partitions

- Topic: a logical namespace and configuration boundary (retention, replication factor, cleanup). Think of it as a contract.
- Partition: the physical, ordered, append-only log where records are stored. Ordering is guaranteed only within a partition.

Example offsets within a partition:

- Offset 0 → Event A
- Offset 1 → Event B
- Offset 2 → Event C

---

## Partition Internals

Partitions are implemented as a sequence of log segments on disk for efficient I/O and retention.

```
partition/
  000000000000.log
  000000000000.index
  000000000123.log
  000000000123.index
```

Benefits: sequential writes, compact indices, and simpler segment eviction.

---

## Replication, Leaders and ISR

- Each partition has one leader and N−1 follower replicas.
- Leader serves reads and writes; followers replicate the leader.
- ISR (In-Sync Replicas) = followers that have caught up; elections pick leaders from the ISR to avoid data loss.

---

## Controller & Metadata

One broker acts as the controller and is responsible for partition assignment, leader election, and detecting broker failures. Modern Kafka can run without ZooKeeper (KRaft), but the controller role remains.

---

## Producer Architecture

- Producers write records to topics and choose a partition for each record.
- Producers send data only to partition leaders; they do not manage offsets nor replicate to followers.

---

## Partition Selection

Partition selection order:
1. Custom partitioner (if configured)
2. If key present → `hash(key) % partition_count`
3. No key → sticky or round-robin partitioner

Key-based partitioning ensures ordering for the same key.

---

## Consumer Architecture

- Consumers pull data from partition leaders and track progress via offsets.
- Kafka is pull-based; brokers do not push data to consumers.

---

## Consumer Groups & Rebalancing

- Consumers in a group share partitions so each partition is processed by at most one consumer in the group.
- Rebalances occur when consumers join/leave or partition counts change.

Rules: one consumer per partition per group; max parallelism = number of partitions.

---

## Offsets & Offset Management

- Offsets identify positions in a partition and are stored per consumer group (in `__consumer_offsets`).
- Offsets enable replay and crash recovery.

---

## Delivery Semantics

Supported modes:
- At-most-once
- At-least-once
- Exactly-once (via transactions)

Semantics depend on producer `acks`, `min.insync.replicas`, and consumer commit strategies.

---

## Ordering Guarantees

- Guaranteed within a single partition or for all messages with the same key.
- No ordering guarantees across partitions or topics.

---

## Fault Tolerance & Failure Scenarios

Kafka handles broker failures (leader elections), consumer failures (rebalances), and transient network issues. Durability is controlled by replication factor, `acks`, and `min.insync.replicas`.

---

## Scaling Kafka

Kafka scales via partitioning (parallelism), sequential disk I/O, and independent brokers. Scale by adding brokers and partitions.

---

## Common Misconceptions (Interview Traps)

- ❌ "Topics store data" — Topics are logical; partitions store data.
- ❌ "Producers manage offsets" — Offsets are consumer-side.
- ❌ "Ordering across topics/partitions is guaranteed" — Not true; ordering is per-partition.
- ❌ "Consumers read from followers" — Consumers read from leaders.

---

## Mental Models Summary

- Topic = WHAT (logical stream)
- Partition = HOW (parallelism & ordering)
- Broker = WHERE (physical storage)
- Key = ROUTING
- Offset = CONSUMPTION POSITION

---

## Further Reading

- Official docs: https://kafka.apache.org
- Confluent blog posts and talks on internals and KRaft

---

## License

Educational and interview-preparation material.
 
---

## Kafka Topics & Partitions — Creation and Producer Partition Logic

This section consolidates two focused topics:
- How partitions are created (who creates them, defaults, code vs infra)
- How producers select partitions by default (key / no key / internal logic)

The focus is on architecture and default behavior rather than application code.

### Who should read this

- Kafka beginners building mental models
- Backend engineers preparing for interviews
- Platform/infra engineers deciding partitioning strategy

---

### 1. How are partitions created?

Key principle

> Partitions are created when a topic is created. Application code does not create partitions.

Partitions are an infrastructure concern managed by the Kafka cluster or platform tooling.

When partitions are created

1. Explicit topic creation (recommended)

```bash
kafka-topics.sh --create \
  --topic orders \
  --partitions 6 \
  --replication-factor 3 \
  --bootstrap-server localhost:9092
```

This creates topic `orders` with partitions `orders-0` .. `orders-5`. Each partition is assigned leaders and replicas by the cluster controller.

2. Auto-topic creation (development default — not recommended for production)

If Kafka is configured with `auto.create.topics.enable=true`, sending a message to a non-existing topic causes Kafka to auto-create the topic with default settings.

Defaults used when auto-created

| Setting | Default |
|---|---:|
| Number of partitions | `num.partitions` (often 1) |
| Replication factor | 1 |

Warning: auto-creation can lead to surprising partition counts and should be disabled in production.

Who decides partition count

| Layer | Responsibility |
|---|---|
| Platform / Infra | Decide partition count and retention policies |
| Kafka Cluster | Create partitions and assign leaders/replicas |
| Application (Producer/Consumer) | Consume/send using topic name only; does not create partitions |

Can partition count be increased later?

Yes. You can increase partition count, e.g.:

```bash
kafka-topics.sh --alter \
  --topic orders \
  --partitions 12 \
  --bootstrap-server localhost:9092
```

Effects of increasing partitions:
- Existing data is NOT rebalanced across new partitions.
- Consumers will rebalance and may receive different partition assignments.
- Key-based ordering may be disrupted; plan partition counts early.

---

### 2. How does the producer select a partition?

Important clarification

There is no single global "default partition" such as partition 0. The producer chooses a partition at send-time using the partitioner logic.

Producer partition selection flow

1. Producer fetches topic metadata (partition count, leaders).
2. Producer runs the partitioner logic:
   - If a custom partitioner is configured → use it.
   - Else if a key is present → `partition = hash(key) % partition_count`.
   - Else (no key) → use the sticky partitioner (modern default) which batches to a chosen partition until the batch is full, then switches.
3. Producer sends the record to the chosen partition leader. Brokers only append and assign offsets.

Case A — Message WITH a key (recommended when ordering by key matters)

Example:

```java
kafkaTemplate.send("orders", "order-123", event);
```

Default behavior:

```
partition = hash(key) % number_of_partitions
```

Properties:
- Same key → same partition → ordering guaranteed per key.
- Deterministic routing across producer restarts (if partition count unchanged).

Case B — Message WITHOUT a key

Example:

```java
kafkaTemplate.send("orders", event);
```

Default modern behavior:
- Uses the sticky partitioner: the producer selects one partition for the current batch, sends multiple records to it, and switches when the batch is full or after a short timeout.

Properties:
- Improved throughput due to batching.
- Not a pure round-robin; ordering across messages without keys is not guaranteed.

Who selects the partition?

- The producer selects the partition (client-side) before sending.
- Brokers do NOT decide which partition a record goes to; they only append to the chosen partition and assign offsets.

Common misconceptions (interview traps)

- ❌ "Producer creates partitions" — No. Topics/partitions are created by the cluster or infra tooling.
- ❌ "Partition 0 is the default" — No fixed default; selection depends on key and partitioner.
- ❌ "Broker chooses partition" — The producer chooses; brokers append.

Final mental model (summary)

- Topic creation → partitions created (by infra/Kafka).
- Producer send → partition selected client-side (key-hash or sticky).
- Broker append → offset assigned.
- Consumer read → offset committed.

One-line

Partitions are created with topics (by infra/Kafka); producers select partitions before sending (key-based hashing if key present, sticky partitioner if not); brokers store data and assign offsets.

---

If you'd like, I can also add failure scenarios, offset commit strategies, or convert this into a diagram-rich README or separate `README.md` file.

---

## Kafka Consumer — Basics (From Zero)

This section explains consumers from first principles: what they read, why offsets exist, and how consumer groups enable parallelism and fault-tolerance.

### Quick summary

- Producers write data; consumers read data — always from partitions, never from topics directly.
- Consumers track progress using offsets; offsets are scoped to (consumer-group, topic, partition).

---

### 1 — What is a Kafka consumer?

- Pulls records from Kafka (consumer initiates reads)
- Reads data sequentially from partition leaders
- Tracks progress using offsets

Kafka is pull-based: the consumer requests data; Kafka does not push data to clients.

---

### 2 — What does a consumer actually read?

Consumers read from partitions, not the topic as a whole.

Example (topic `orders` with 3 partitions):

- Partition 0 → offsets 0,1,2,...
- Partition 1 → offsets 0,1,2,...
- Partition 2 → offsets 0,1,2,...

Each partition is ordered and independent; consumers consume each partition sequentially.

---

### 3 — Why do consumers need offsets?

Offsets answer: “From where should I continue reading?”

They enable restart safety, failure recovery, replay, and delivery guarantees. An offset is simply a position in a partition log.

---

### 4 — Offsets are NOT global

Offsets are tracked per (consumer-group, topic, partition).

Example:

- Group A: orders-0 → 120, orders-1 → 95
- Group B: orders-0 → 10,  orders-1 → 10

Same data, different progress per group.

---

### 5 — Consumer groups (core concept)

- A consumer group is a set of consumers cooperating to consume data and share partitions.
- Rules: one partition → one consumer per group; different groups can read the same data independently.

Benefits: parallelism, scalability, and automatic failover.

---

### 6 — Partition/consumer count scenarios

- Consumers > partitions: some consumers idle (parallelism limited by partition count).
- Partitions > consumers: consumers own multiple partitions (work distributed across fewer consumers).

---

### 7 — Consumer startup flow (important)

When a consumer starts:
1. Joins the consumer group (JoinGroup)
2. Receives assigned partitions from the coordinator
3. Fetches last committed offsets
4. Begins polling and processing

---

### 8 — Minimal Spring Boot consumer example

```java
@KafkaListener(
    topics = "orders",
    groupId = "order-service"
)
public void consume(OrderEvent event) {
    System.out.println("Received order: " + event);
}
```

Spring hides the consumer lifecycle: it creates the `KafkaConsumer`, joins the group, polls, heartbeats, and manages partition assignment.

---

### 9 — Where to start reading (offset resolution)

- If committed offsets exist → start from last committed offset + 1.
- If no offset exists (first-time) → controlled by `auto.offset.reset` (`earliest` or `latest`).

| Value | Behavior |
|---|---|
| `earliest` | Start from offset 0 (beginning) |
| `latest` | Start from end (new messages only) |

---

### 10 — Consumer poll loop (internal view)

Simplified loop:

```text
while (running) {
  poll()
  process(records)
  commitOffsets()
}
```

`poll()` fetches records, sends heartbeats, and detects rebalances.

---

### 11 — Key guarantees (at this stage)

- Ordering within a partition ✅
- At-least-once by default ✅
- Parallelism via partitions ✅
- Exactly-once — requires transactions and idempotency ❌

---

### 12 — Common beginner confusions

- ❌ Consumer reads from topic (false)
- ❌ Offset is global (false)
- ❌ Multiple consumers in same group can read same partition (false)
- ❌ Kafka pushes messages (false)

✅ Correct: consumers read partitions; offsets are per group+partition; Kafka is pull-based.

---

### 13 — Mental model (lock this in)

Producer → Partition → Offset
Consumer → Offset → Processing

---

## Offset in Kafka — From Creation to Consumption

An offset is a sequential index for a record inside a partition. This section explains where offsets are created, how clients see them, and how consumers use them.

### 1 — What is an offset?

Offset = sequential index of a record inside a partition (unique only within that partition).

Example (partition 0):
- Offset 0 → Order A
- Offset 1 → Order B
- Offset 2 → Order C

Key properties: monotonic per partition, no meaning across partitions.

---

### 2 — When and how is an offset CREATED?

- Offsets are assigned by the broker (leader) when appending the record to the partition log.

Flow:
1. Producer sends record to chosen partition leader.
2. Broker appends record and assigns next offset (e.g., `next++`).
3. Broker returns the offset in the produce response.

Note: offset is assigned after partition selection, never before.

---

### 3 — Does the producer use the offset?

No — producers do not manage offsets. They may receive offsets in responses for logging/debugging only; offsets are a consumer concern.

---

### 4 — Why do consumers need offsets?

Offsets allow consumers to resume, recover, replay, and provide delivery guarantees — without them consumers could re-read everything or lose track of progress.

---

### 5 — Where are offsets stored?

Committed offsets are stored in the internal compacted topic `__consumer_offsets`, which is replicated and durable.

---

### 6 — How does a consumer GET the offset?

On startup: the consumer joins the group, receives partition assignments, and fetches the last committed offsets from `__consumer_offsets`. It then starts reading from `committed_offset + 1`.

---

### 7 — What if no offset exists?

Controlled by `auto.offset.reset` (`earliest` or `latest`) — applies only when no committed offset exists for the group+partition.

---

### 8 — How consumers use offsets while reading

Typical loop:

1. `poll()` → receive records with offsets
2. Process records
3. Commit the last successfully processed offset

Example: read offsets 100,101,102; process; commit 102 → next read = 103.

---

### 9 — What does committing an offset mean?

Committing an offset declares group progress. It does NOT delete data or inform producers; it simply records "we are done up to X" for that consumer group.

---

### 10 — Crash scenarios

- Read up to 200, committed up to 195, crash → on restart resume from 196 → offsets 196–200 reprocessed (at-least-once).

---

## Offset Commit Strategies

Offset commit strategy determines delivery semantics: at-most-once vs at-least-once vs exactly-once (with transactions).

### 1 — What does committing an offset mean?

It means the consumer group records progress ("I have processed up to X"). It does not delete messages or ack producers.

### 2 — Where are offsets committed?

In `__consumer_offsets` (durable, replicated Kafka topic), keyed by `(groupId, topic, partition)`.

### 3 — Two core strategies: Auto Commit vs Manual Commit

#### Auto Commit (simple, risky)
- `enable.auto.commit=true`, `auto.commit.interval.ms=5000`
- Kafka periodically commits the latest offset independent of processing.

Risk: app may crash after commit but before processing → data loss (at-most-once semantics).

When acceptable: metrics, logs, fire-and-forget pipelines.

#### Manual Commit (recommended)
- `enable.auto.commit=false`
- Application explicitly commits after successful processing. Guarantees at-least-once.

### 4 — Sync vs Async manual commits

- `commitSync()` blocks until Kafka confirms commit (safer, slower).
- `commitAsync()` is non-blocking (faster, may fail silently).
- Common pattern: `commitAsync()` during normal flow, `commitSync()` on shutdown.

### 5 — Which offset to commit?

Commit the last successfully processed offset (e.g., after processing offsets 10..12, commit 12 → resume at 13).

### 6 — Crash ordering examples

- Crash BEFORE commit → reprocess uncommitted messages (duplicates possible).
- Commit BEFORE processing → risk of data loss.

### 7 — Exactly-once vs at-least-once

- Commit after processing → at-least-once.
- Commit before processing → at-most-once.
- Transactions + idempotent producers → exactly-once semantics (end-to-end).

### 8 — Spring Boot example (manual commit)

```java
@KafkaListener(
  topics = "orders",
  groupId = "order-service",
  containerFactory = "manualAckContainerFactory"
)
public void consume(OrderEvent event, Acknowledgment ack) {
    process(event);   // business logic
    ack.acknowledge(); // commit offset
}
```

Spring handles offset tracking, commit calls, and rebalance safety when configured appropriately.

---

### Common interview traps (offsets & consumers)

- ❌ "Commit deletes messages" — No; commit records progress only.
- ❌ "Offset is producer-side" — No; offset is assigned by broker and consumed by consumer groups.
- ❌ "Auto commit is safest" — No; auto-commit can cause data loss.

✅ Correct: commit after processing → at-least-once; `__consumer_offsets` stores durable commits per group+partition.

---

If you'd like, say "offset commit strategies next" to continue deeper into rebalancing, failure scenarios, or exactly-once patterns with examples and diagrams.

---

## How a Kafka Consumer Decides Partition, Gets Offset, and Commits It (Java View)

This section maps consumer theory to concrete Java/Spring behavior and shows the exact order Kafka executes consumer-side operations.

### Quick summary

- Consumers do not choose partitions in normal operation — Kafka assigns partitions when consumers join a group.
- After assignment the consumer fetches committed offsets from `__consumer_offsets`, polls records (which include offsets), processes them, then commits offsets.

---

### 1 — Who decides partitions?

Important rule (memorize): Consumers do NOT choose partitions; Kafka assigns them via the group coordinator.

In Spring/Java the annotation indicates intent but not assignment:

```java
@KafkaListener(
  topics = "orders",
  groupId = "order-service"
)
public void consume(OrderEvent event) { ... }
```

This tells Kafka the consumer belongs to `order-service` and wants `orders`; the coordinator returns partition assignments automatically.

---

### 2 — Partition assignment flow (internal steps)

When a consumer starts:
1. Connects and sends `JoinGroup` with `groupId`.
2. Kafka elects a group coordinator.
3. Coordinator runs the partition assignor and returns assignments.
4. Consumer receives assigned partitions and fetches committed offsets.

Example (orders: 4 partitions, consumers C1,C2):

- C1 → P0,P1
- C2 → P2,P3

Note: `consumer.assign(...)` bypasses group management (no rebalances) and is used only in advanced cases.

---

### 3 — How the consumer gets the offset

After assignment Kafka answers "where should this consumer start?" by reading committed offsets from `__consumer_offsets` and returning offsets per partition.

- If an offset exists → start from `committed_offset + 1`.
- If no offset exists → controlled by `auto.offset.reset` (`earliest` or `latest`).

---

### 4 — Polling records (Java view)

Plain Java consumer loop:

```java
ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
for (ConsumerRecord<String, Order> r : records) {
  // r.topic(), r.partition(), r.offset(), r.value()
  process(r.value());
}
consumer.commitSync(); // commit after processing
```

Spring's `@KafkaListener` wraps this: it joins group, polls, heartbeats, and calls your handler.

---

### 5 — Committing offsets (best practice)

Rule: commit offsets only AFTER successful processing.

Spring manual ack example:

```java
@KafkaListener(
  topics = "orders",
  groupId = "order-service",
  containerFactory = "manualAckContainerFactory"
)
public void consume(OrderEvent event, Acknowledgment ack) {
  process(event);
  ack.acknowledge(); // commits the last processed offset
}
```

Plain Java manual commit:

```java
consumer.poll(...);
process(records);
consumer.commitSync();
```

Committing means: "this group has finished processing up to offset X" — next read will start at `X+1`.

---

### 6 — Crash scenarios (must understand)

- Safe: Process → Commit → Crash → no data loss.
- Risky: Commit → Process → Crash → data loss.

Prefer commit-after-processing (manual commit) for correctness in production.

---

### 7 — Java mental model (one-liner)

`@KafkaListener` → JoinGroup → Kafka assigns partitions → fetch committed offsets → `poll()` reads records with offsets → process → commit offsets.

---

### 8 — Interview traps (avoid these)

- ❌ "Consumer chooses partition" — false.
- ❌ "Offset is global" — false.
- ❌ "Commit deletes message" — false.
- ❌ "Kafka pushes messages" — false.

✅ Correct: Kafka assigns partitions; offsets are per group+partition; commit = progress marker; Kafka is pull-based.

---

If you'd like, I can add a Mermaid diagram showing JoinGroup → Assign → FetchOffsets → Poll → Commit (Java and Spring variants).

## Consumer Group — What It Is & Why We Need It

This section explains consumer groups from first principles, why they exist, how they behave in Java/Spring, and how they enable parallel processing.

### Quick summary

- A consumer group is a logical set of consumers identified by a `group.id`.
- Within a group, each partition is processed by exactly one consumer; different groups read the same data independently.

---

### 1 — What is a consumer group?

- Identified by `group.id`.
- Contains one or more consumers that share partitions.
- Tracks progress using group-specific offsets (`(groupId, topic, partition) -> offset`).

Example: `group.id = "order-service"` — all consumers with this id cooperate as one logical application.

---

### 2 — Why do we need consumer groups?

Consumer groups solve three problems:

- Scalability (parallel processing): partitions are units of work distributed across consumers.
- Fault tolerance (automatic failover): if a consumer crashes, remaining members take over its partitions.
- Independent consumption (multiple views): different services use distinct group ids to consume the same topic independently.

Example (orders, 4 partitions):

- C1 → P0,P1
- C2 → P2,P3

Parallel processing and automatic failover follow from this assignment.

---

### 3 — What happens without consumer groups?

- No parallelism: every consumer would read everything (duplicates everywhere).
- No failover: crashes would leave partitions unprocessed.
- No isolation: one service’s progress would affect others.

Consumer groups solve all three issues.

---

### 4 — Core rules (memorize)

1. One partition → at most one consumer in a group.
2. A consumer may read multiple partitions.
3. Max parallelism = number of partitions.
4. Different groups read independently.

---

### 5 — Consumer group vs partitions (visual)

Topic: `payments` (3 partitions)

Group A:
  C1 → P0
  C2 → P1
  C3 → P2

Group B:
  C4 → P0,P1,P2

Group A and Group B are independent consumers of the same data.

---

### 6 — Offsets and consumer groups

Offsets are stored per `(groupId, topic, partition)`. Same partition — different groups → independent offsets and progress.

---

### 7 — Internal flow (high level)

1. Consumer starts with `group.id`.
2. Kafka elects a group coordinator.
3. Consumers join the group and the coordinator assigns partitions.
4. Offsets are fetched for assigned partitions.
5. Consumers poll and commit offsets per group.

---

### 8 — Java / Spring: how groups are created and scaled

Key point: There is no explicit "create group" API — a group is created implicitly when the first consumer with a `group.id` starts.

Spring Boot example (application level):

```yaml
spring:
  kafka:
    consumer:
      group-id: order-service
```

Consumer code:

```java
@KafkaListener(topics = "orders")
public void consume(OrderEvent event) { process(event); }
```

To scale consumers: run multiple instances of the same application (pods/containers/JVMs). Kafka assigns partitions across instances.

---

### 9 — What NOT to do

- Do NOT create multiple listeners in the same JVM expecting better parallelism — Kafka scales at the process/instance level, not thread level.
- Avoid sharing a `group.id` across different microservices with different business logic — this will split partitions across services and break correctness.

Bad pattern:

```java
@KafkaListener(topics = "orders", groupId = "order-service")
public void c1(...) {}

@KafkaListener(topics = "orders", groupId = "order-service")
public void c2(...) {}
```

This runs in a single JVM and does not provide intended independent scaling.

---

### 10 — Same `group.id` across microservices — why it's dangerous

- If two different services share the same `group.id`, Kafka treats them as one logical consumer application and splits partitions across both services.
- This results in each service receiving only a subset of messages (incorrect behavior) and is a common production bug.

Correct approach: each independent service must use its own `group.id`.

---

### 11 — One-line interview answer

Consumer group = a logical application identity (group.id). Multiple instances with the same `group.id` cooperate as workers; partitions are divided across them to enable parallelism and failover.

---

## How Parallel Processing Is Achieved in Kafka

Parallelism is a direct consequence of partitions + consumer groups.

### Core idea

Partition = unit of parallelism. Kafka assigns partitions to consumers in a group; each partition is processed by only one consumer in that group.

### Formula

Effective parallelism = min(number_of_partitions, number_of_consumers_in_group)

### Scenarios

- Partitions=6, Consumers=1 → Parallelism=1 (one consumer handles all partitions)
- Partitions=6, Consumers=6 → Parallelism=6 (ideal)
- Partitions=3, Consumers=6 → Parallelism=3 (3 consumers idle)
- Partitions=8, Consumers=3 → Parallelism=3 (each consumer handles multiple partitions)

### Where parallelism runs

At process/instance level (JVM, container, pod), not inside Kafka. Scale by increasing application instances or partition count.

### Why Kafka does NOT parallelize within a partition

To preserve strict ordering and offset correctness, Kafka enforces one consumer per partition per group.

---

### Interview-ready one-liner

Kafka achieves parallel processing by partitioning topics and distributing those partitions across consumers in a consumer group; effective parallelism equals the minimum of partitions and consumers.

---

If you'd like, I can (a) extract this into `CONSUMER-GROUPS.md`, (b) add diagrams (mermaid) for group join & rebalancing, or (c) produce a one-page interview cheat-sheet. Which would you prefer?

---

## Consumer Rebalancing — Mechanics & Purpose

Consumer rebalancing is the process Kafka uses to redistribute partition ownership within a consumer group when group membership or topic topology changes. It guarantees correct, balanced consumption but introduces a short disruption window.

### Quick summary

- Rebalance = pause consumers + revoke/assign partitions + resume.
- Triggered on joins, leaves, failures, subscription changes, or partition increases.

---

### 1 — What is rebalancing?

- During a rebalance, partition ownership moves between group members so each partition is owned by at most one consumer in the group.
- Consumption is briefly paused while ownership is exchanged and offsets are reconciled.

---

### 2 — Why rebalancing is required

1. Fault tolerance — when a consumer fails, others must take over its partitions.
2. Scalability — when new consumers join, partitions are redistributed to balance load.
3. Topology changes — partition increases or subscription changes require fresh assignments.

Rebalancing enforces correctness: one consumer per partition per group and independent progress via offsets.

---

### 3 — When does a rebalance occur?

- Consumer joins a group
- Consumer leaves or crashes
- Missed heartbeats
- Topic partition count increases
- Consumer subscription changes

---

### 4 — Who coordinates rebalances?

The Group Coordinator (a broker) tracks membership, runs the assignor, computes assignments, and notifies members.

---

### 5 — Step-by-step rebalance flow

1. Trigger detected (join/leave/failure/topology change).
2. Stop-the-world phase: consumers stop fetching new records.
3. Revoke: current consumers relinquish owned partitions (should commit offsets first).
4. Assign: coordinator runs the assignor algorithm and computes new ownership.
5. Resume: consumers start fetching from committed offsets for new partitions.

Key detail: committing offsets before revocation preserves at-least-once semantics.

---

### 6 — Partition assignor strategies (short)

- RangeAssignor — groups partitions by topic ranges (can be unbalanced).
- RoundRobinAssignor — spreads partitions evenly across consumers.
- StickyAssignor (default) — minimizes partition movement to reduce churn.

Sticky assignor + cooperative strategies minimize partition movement and rebalance cost.

---

### 7 — What happens to offsets during rebalance?

- Offsets are durable in `__consumer_offsets` and are not lost.
- Consumers should commit offsets before revocation; new owners resume from the committed offset (`committed + 1`).

---

### 8 — Why rebalancing is painful (real-world impact)

- Consumption pause → latency spikes and throughput drop.
- In-flight processing may be duplicated when consumers restart processing after reassignment.
- Frequent rebalances harm stability and SLOs.

---

### 9 — How Kafka reduces rebalance impact

- Sticky partition assignor: keeps partitions on the same consumers when possible.
- Cooperative rebalancing (incremental) avoids full stop-the-world rebalances.
- Tune consumer timeouts: `session.timeout.ms`, `heartbeat.interval.ms`, `max.poll.interval.ms`.

---

### 10 — Operational recommendations

- Minimize frequent consumer restarts and churning deployments.
- Use sticky assignor and cooperative rebalancing when available.
- Ensure consumers commit offsets promptly and handle revoke callbacks to flush progress.

---

### 11 — Mental model (lock this)

Rebalance = load redistribution + recovery. Change in group or topology → rebalance → safe continuation from committed offsets.

---

### 12 — Interview one-liner

Consumer rebalancing reassigns partitions across group members on membership or topology changes to preserve correctness and availability; it temporarily pauses consumption and must be managed (sticky assignor, cooperative rebalancing, and proper commit handling) to limit impact.

---

## Resuming Consumption After Rebalancing — What Happens?

Short answer: ownership moves, new owners read the last committed position, and processing resumes from there.

Sequence of events (practical view):
- Rebalance triggered (join/leave/failure/partition change).
- Consumers stop fetching new records and should commit in-flight progress.
- Coordinator computes new assignments and notifies members.
- Consumers fetch committed offsets for their newly assigned partitions (the stored position is the next offset to read).
- Consumers `seek()` to the committed position and resume polling.

Key operational notes:
- Commit before revocation: during `onPartitionsRevoked`, commit processed offsets to avoid duplicates.
- On assignment: seek to committed offsets (or `earliest`/`latest` based on policy) before calling `poll()`.
- Cooperative rebalancing reduces stop-the-world pauses by making rebalances incremental.

Java example (rebalance listener pattern):

```java
consumer.subscribe(topics, new ConsumerRebalanceListener() {
  @Override
  public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
    // commit current progress synchronously to avoid duplicates
    consumer.commitSync(currentOffsets);
  }

  @Override
  public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
    // resume from committed position (or decide earliest/latest)
    for (TopicPartition tp : partitions) {
      OffsetAndMetadata committed = consumer.committed(tp);
      if (committed != null) {
        consumer.seek(tp, committed.offset());
      } else {
        consumer.seekToBeginning(Collections.singleton(tp));
      }
    }
  }
});
```

---

## What Does "Commit Offset" Mean? — Practical Explanation

Definition: committing an offset records the group's progress for a topic-partition in the internal topic `__consumer_offsets`. The committed value represents the position the group will resume from.

Important details:
- The committed offset is the offset of the next record to read (i.e., if you processed up to `102`, committing `103` means the consumer will resume at `103`).
- Committing does NOT delete data, acknowledge producers, or affect retention — it only records progress for the consumer group.
- Common practice: commit AFTER successful processing to achieve at-least-once semantics.

Commit APIs and patterns:
- `commitSync()` — blocks until commit is acknowledged (safer during shutdown).
- `commitAsync()` — non-blocking; often used for throughput with a final `commitSync()` on shutdown.
- Frameworks (Spring Kafka) provide `Acknowledgment.acknowledge()` that delegates to the underlying commit logic.

Crash examples (illustrative):
- Processed up to 200, committed 195, crash → resume at 196 → offsets 196–200 may be reprocessed (duplicate processing).
- Committed 200, then process crashed before side-effect → resume at 200 (or 201 depending on convention) → potential data loss if commit occurred before processing.

One-liner: commit = durable progress marker (resume position) stored in `__consumer_offsets`; commit the next offset to read after successful processing to avoid data loss.

---

## What Happens to Messages After Retention Time?
![Kafka retention configuration](images/kafka-retention-configuration-hierarchy%20%281%29.png)
![Kafka time-index configuration](images/2021-12-17-timeindex.png)
Short answer: Kafka deletes data by removing entire log segments when retention conditions are met; this is independent of consumer offsets.

1) What is retention?

- Retention controls how long Kafka keeps data on disk for a topic. It is independent of consumers and offsets.

2) Common retention settings

| Setting | Meaning |
|---|---|
| `retention.ms` | Time-based retention (e.g., 7 days) |
| `retention.bytes` | Size-based retention per partition |
| `cleanup.policy` | `delete` (default) or `compact` |

Rules: Kafka deletes data when any retention condition is met (time OR size).

3) How deletion actually happens

- Kafka deletes whole log segments, not individual messages. Each segment contains many record batches/messages and covers a contiguous offset range.
- When a segment exceeds retention limits it is removed in full. This is efficient for sequential disk I/O and retention management.

4) What if consumers haven't read the data?

- Kafka does not wait for consumers. If the segment containing a committed offset is deleted, the consumer will not find that offset on restart. Kafka applies `auto.offset.reset` (`earliest` or `latest`) to decide where to resume.

Example:

- Committed offset = 5, segment with offset 5 deleted → on restart consumer will use `auto.offset.reset` to determine next start (earliest/latest).

5) Compaction (special-case cleanup)

- `cleanup.policy=compact` keeps only the latest record per key and removes older records with the same key. Compaction is used for changelogs and state topics. Even with compaction, retention and offsets behave independently.

One-liner (retention): Kafka removes old data by deleting log segments according to retention policies (time/size/cleanup), regardless of consumer progress or committed offsets.

---

## Handling Duplicates in Consumers

![Kafka time-index configuration](images\messageSemantics.png)

Short answer: duplicates are expected under Kafka’s default at-least-once delivery; consumers must be implemented idempotently or use DB constraints/transactions.

1) Why duplicates occur

- Kafka provides at-least-once delivery by default. Duplicates happen when a consumer processes a record but crashes before committing the offset, on rebalances, or due to transient commit failures.

2) Principle: consumers must be idempotent

- Design processing so that reprocessing the same message is safe.

3) Practical strategies

- Unique event ID: include an `eventId` (UUID) and persist it (skip if already processed).
- Database constraints: `UNIQUE(event_id)` or upsert semantics (`INSERT ... ON CONFLICT DO NOTHING`).
- Upserts/state updates: use idempotent updates rather than blind inserts.
- Exactly-once (advanced): use idempotent producers + transactions or Kafka Streams for end-to-end EOS, but this increases complexity.

4) What NOT to rely on

- In-memory caches for deduplication are unsafe alone (lost on restart/rebalance). Persisted dedup state or DB constraints are reliable.

5) Offset commit timing

- Commit AFTER processing → at-least-once (duplicates possible).
- Commit BEFORE processing → at-most-once (risk of data loss).

6) Interview one-liner (duplicates)

Kafka delivers at-least-once by default; consumers must handle duplicates using idempotency patterns such as unique event IDs, DB constraints, upserts, or by adopting Kafka transactions for stricter semantics.

---