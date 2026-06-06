# 5.3 · Apache Kafka

## Overview

Apache Kafka is a distributed, append-only log designed for high-throughput, fault-tolerant, real-time event streaming. It decouples producers from consumers and enables durable event replay.

---

## Core Concepts

### Architecture
- **Topic** — named category of records; divided into partitions
- **Partition** — ordered, immutable log; enables parallelism and scalability
- **Offset** — unique position of a record within a partition
- **Producer** — publishes records to topics
- **Consumer** — reads records from topics at its own pace
- **Consumer Group** — multiple consumers share partition assignment; each partition → one consumer per group
- **Broker** — a Kafka server node; manages partitions
- **ZooKeeper / KRaft** — cluster coordination (KRaft replaces ZK since Kafka 3.3)

### Replication
- Each partition has **1 leader + N replicas** (ISR = In-Sync Replicas)
- Producers write to leader; followers replicate
- `replication.factor` = durability; `min.insync.replicas` = min acks required

### Delivery Guarantees
| Mode | Setting | Trade-off |
|------|---------|-----------|
| At-most-once | `acks=0`, no retry | Possible loss, fastest |
| At-least-once | `acks=all`, retry on error | Possible duplicates |
| Exactly-once | Idempotent producer + transactions | Highest correctness |

### Producer Config (Key Settings)
- `acks` — `0`, `1`, `all` (ISR acknowledgement)
- `retries`, `retry.backoff.ms`
- `enable.idempotence=true` — exactly-once per partition
- `batch.size`, `linger.ms` — throughput vs latency
- `compression.type` — `snappy`, `lz4`, `zstd`

### Consumer Config (Key Settings)
- `group.id` — consumer group identifier
- `auto.offset.reset` — `earliest` | `latest`
- `enable.auto.commit` — manual vs auto offset commit
- `max.poll.records`, `max.poll.interval.ms`

### Log Compaction
- Retains only the **latest record per key** in a compacted topic
- Used for changelog / event sourcing scenarios (e.g., CDC)
- `cleanup.policy=compact` (vs `delete` for time-based retention)

### Kafka Streams & ksqlDB
- Kafka Streams: Java library for stateful stream processing (joins, aggregations, windowing)
- ksqlDB: SQL interface for stream processing

---

## Key Commands / Code Snippets

```bash
# Create topic
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders --partitions 6 --replication-factor 3

# List topics
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Describe topic
kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic orders

# Consume from beginning
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders --from-beginning --group debug-group

# Consumer group lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group my-consumer-group
```

```go
// Go producer (confluent-kafka-go)
p, _ := kafka.NewProducer(&kafka.ConfigMap{
    "bootstrap.servers":  "localhost:9092",
    "enable.idempotence": true,
    "acks":               "all",
})
p.Produce(&kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
    Key:            []byte("order-123"),
    Value:          []byte(`{"status":"created"}`),
}, nil)
p.Flush(5000)

// Go consumer
c, _ := kafka.NewConsumer(&kafka.ConfigMap{
    "bootstrap.servers":  "localhost:9092",
    "group.id":           "order-processor",
    "auto.offset.reset":  "earliest",
    "enable.auto.commit": false,
})
c.Subscribe([]string{"orders"}, nil)
for {
    msg, err := c.ReadMessage(10 * time.Second)
    if err != nil { continue }
    process(msg)
    c.CommitMessage(msg) // manual commit after processing
}
```

---

## Common Interview Questions

**Q: How does Kafka achieve high throughput?**
> Sequential disk writes (append-only log), batching (producer sends batches), zero-copy I/O (`sendfile`), and partition-level parallelism. Kafka can sustain millions of messages/sec on commodity hardware.

**Q: How do consumer groups work?**
> Partitions are distributed across consumers in a group — each partition is owned by exactly one consumer. Adding consumers up to the partition count increases parallelism. More consumers than partitions → idle consumers.

**Q: What is the difference between at-least-once and exactly-once in Kafka?**
> At-least-once: ack + retry can duplicate on failure. Exactly-once: idempotent producer (dedup by sequence number per partition) + transactions (atomic write to multiple partitions + offset commit).

**Q: What is log compaction used for?**
> Keeping only the latest value per key. Use for event sourcing / CDC where you want the current state of each entity, not the full history. A null value (tombstone) marks deletion.

**Q: How would you handle consumer lag?**
> Scale consumers (up to partition count), increase `max.poll.records`, optimize processing, or add more partitions (requires topic recreation or careful reassignment).

---

## Gotchas & Best Practices

- Partition count is the max parallelism — plan ahead (repartitioning is complex)
- Use meaningful partition keys for ordering guarantees (all events for an entity → same partition)
- Never decrease partition count
- Monitor consumer group lag — use `kafka-consumer-groups.sh` or Burrow
- `enable.auto.commit=false` + manual commit after processing = safe at-least-once
- Avoid small messages — batch at the application layer if needed
- Replication factor ≥ 3 for production; `min.insync.replicas=2`

---

## Resources

- [ ] [Kafka Documentation](https://kafka.apache.org/documentation/)
- [ ] [Confluent Kafka Go Client](https://github.com/confluentinc/confluent-kafka-go)
- [ ] [Kafka: The Definitive Guide (O'Reilly)](https://www.oreilly.com/library/view/kafka-the-definitive/9781491936153/)
- [ ] [Exactly-Once Semantics in Kafka](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

