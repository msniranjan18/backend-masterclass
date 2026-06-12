# 9.1 · Distributed Systems Design

## Overview

Distributed systems involve multiple independent nodes communicating over a network. Designing them requires understanding consistency trade-offs, failure modes, consensus, and coordination primitives.

---

## Core Concepts

### CAP Theorem
A distributed data store can provide at most **2 of 3** guarantees:
- **C**onsistency — every read receives the most recent write
- **A**vailability — every request receives a (non-error) response
- **P**artition Tolerance — system continues despite network partitions

> In practice, partitions are unavoidable → choose CP or AP per use-case.

### Consistency Models (weakest → strongest)
| Model | Description | Example |
|-------|-------------|---------|
| Eventual | Replicas converge over time | DNS, Cassandra |
| Monotonic Read | Once you read a value, never read older | |
| Read-your-writes | You always see your own writes | |
| Causal | Operations that are causally related are ordered | |
| Linearizability | Strongest — operations appear instantaneous | etcd |

### PACELC
Extension of CAP: even without partitions, there's a trade-off between **L**atency and **C**onsistency.

### Consensus Algorithms
- **Raft** — leader election + log replication; used by etcd, CockroachDB
- **Paxos** — original consensus; complex to implement correctly
- Key properties: Safety (never return incorrect result) + Liveness (eventually return result)

### Distributed Transactions
- **2PC (Two-Phase Commit)** — coordinator asks all nodes to prepare, then commit; blocking on coordinator failure
- **Saga Pattern** — sequence of local transactions with compensating transactions for rollback
  - Choreography: events trigger next step
  - Orchestration: central coordinator directs steps

### Failure Modes
- **Crash failure** — node stops responding
- **Byzantine failure** — node behaves arbitrarily/maliciously
- **Network partition** — nodes cannot communicate
- **Split-brain** — two nodes both believe they are the leader

### Key Distributed Primitives
- **Leader Election** — one node coordinates; others follow (etcd lease, ZooKeeper)
- **Distributed Lock** — Redis SETNX + expire, etcd transactions
- **Idempotency Keys** — prevent duplicate processing on retry
- **Vector Clocks / Lamport Timestamps** — ordering events without synchronized clocks
- **Gossip Protocol** — eventual dissemination of state (used by Consul, Cassandra)

### Reliability Patterns
- **Circuit Breaker** — stop calling a failing service; fail fast
- **Retry with exponential backoff + jitter** — avoid thundering herd
- **Bulkhead** — isolate failure domains (separate thread pools / connection pools)
- **Timeout** — never wait forever
- **Health checks** — liveness vs readiness vs startup

---

## Key Commands / Code Snippets

```go
// Idempotency with Redis
func processWithIdempotency(ctx context.Context, rdb *redis.Client, idempotencyKey string, fn func() error) error {
    set, err := rdb.SetNX(ctx, "idem:"+idempotencyKey, "processing", 24*time.Hour).Result()
    if err != nil { return err }
    if !set { return nil } // already processed
    return fn()
}

// Circuit breaker (conceptual)
type CircuitBreaker struct {
    failures  int
    threshold int
    state     string // closed, open, half-open
    mu        sync.Mutex
}
```

```
Saga Orchestration Flow:
  Orchestrator → OrderService.Create
              → PaymentService.Reserve
              → InventoryService.Reserve
              → OrderService.Confirm

On failure at any step → compensating transactions in reverse
```

---

## Common Interview Questions

**Q: Explain the CAP theorem with a practical example.**
> During a network partition between two DB replicas: if we serve reads from both (A), they may return stale data (sacrificing C). If we refuse reads until in sync (C), we sacrifice A. Systems like Cassandra choose AP; etcd/Zookeeper choose CP.

**Q: How does Raft achieve consensus?**
> Raft elects a leader via randomized timeouts. The leader accepts writes, appends to its log, and replicates to a majority of followers before committing. This ensures only one leader and consistent log ordering.

**Q: What is the Saga pattern and when do you use it?**
> Saga breaks a distributed transaction into a sequence of local transactions. On failure, compensating transactions undo completed steps. Use it when 2PC's blocking coordinator is unacceptable (e.g., across microservices or external APIs).

**Q: How would you implement a distributed lock?**
> Redis `SET key value NX PX ttl` — acquire lock only if absent, with TTL to prevent deadlock. On success: work → delete key. Use unique value (UUID) to avoid releasing another client's lock.

**Q: What is idempotency and why does it matter?**
> An idempotent operation produces the same result regardless of how many times it's executed. Critical for safe retries — use idempotency keys in payment APIs, message deduplication IDs in queues.

---

## Gotchas & Best Practices

- Never assume network calls succeed — design for partial failure
- Eventual consistency requires conflict resolution strategy (last-write-wins, CRDT, application-level merge)
- Distributed locks do not guarantee mutual exclusion in all failure scenarios (use fencing tokens)
- Retry without jitter causes thundering herd — always add random jitter
- 2PC is blocking — prefer sagas for cross-service transactions
- Log sequence numbers / vector clocks are needed for causality in event-driven systems

---

## Resources

- [ ] [Designing Data-Intensive Applications (Kleppmann)](https://dataintensive.net/)
- [ ] [The Raft Consensus Algorithm](https://raft.github.io/)
- [ ] [AWS Builder's Library — Avoiding cascading failures](https://aws.amazon.com/builders-library/)
- [ ] [CAP Twelve Years Later (Brewer)](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)

---

## My Notes

<!-- ADD YOUR PERSONAL NOTES HERE -->

