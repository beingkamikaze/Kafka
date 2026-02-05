"""Patch applied: rework markdown structure for clarity and formatting."""
## Apache Kafka — Architecture, Internals & Mental Models

This document explains Apache Kafka from first principles: core concepts, internals, and mental models useful for engineers and interview preparation.

---

**Quick summary:** Kafka is a distributed, fault-tolerant, append-only commit log. Producers append records to partitions; consumers read sequentially using offsets.

---

## Table of Contents

- [What is Kafka?](#what-is-kafka)
- [High-level Architecture](#high-level-architecture)
- [Cluster and Brokers](#cluster-and-brokers)
- [Topics vs Partitions](#topics-vs-partitions)
- [Partition Internals](#partition-internals)
- [Replication, Leaders and ISR](#replication-leaders-and-isr)
- [Controller & Metadata](#controller--metadata)
- [Producer Architecture](#producer-architecture)
- [Partition Selection](#partition-selection)
- [Consumer Architecture](#consumer-architecture)
- [Consumer Groups & Rebalancing](#consumer-groups--rebalancing)
- [Offsets & Offset Management](#offsets--offset-management)
- [Delivery Semantics](#delivery-semantics)
- [Ordering Guarantees](#ordering-guarantees)
- [Fault Tolerance & Failure Scenarios](#fault-tolerance--failure-scenarios)
- [Scaling Kafka](#scaling-kafka)
- [Common Misconceptions (Interview Traps)](#common-misconceptions-interview-traps)
- [Mental Models Summary](#mental-models-summary)
- [Further Reading](#further-reading)

---

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