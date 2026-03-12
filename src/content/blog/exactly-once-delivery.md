---
title: "Exactly-once semantics on at-least-once infrastructure"
date: 2026-03-12
description: "Exactly-once delivery is impossible at the transport layer. The pattern that gives you the semantics anyway: at-least-once delivery plus an idempotent writer."
tags: ["system design", "distributed systems", "data pipelines"]
ogImage: "/og-exactly-once.png"
draft: false
---

Your payment service processes a charge. The database write succeeds. The network drops before the producer receives the acknowledgment. The producer retries. The charge runs twice.

The naive fix — "just don't retry" — trades duplicates for data loss. The real fix: stop trying to eliminate retries and make them harmless instead. **At-least-once delivery** (retry until confirmed) plus an **idempotent writer** (duplicate writes have no effect) gives you exactly-once *semantics* without needing an impossible guarantee from the transport layer. The message may arrive more than once — that's by design. What matters is that its effect on the database happens exactly once.

## Why exactly-once at the transport layer is impossible

When a write times out, you face an ambiguity you cannot resolve: did the message arrive and the acknowledgment get lost, or did the message never arrive? You can't know. To avoid data loss, you retry. That's the [Two Generals Problem](https://en.wikipedia.org/wiki/Two_generals_problem) — certainty is not achievable over an unreliable channel.

This means any system that prioritizes no data loss is at-least-once by design. Duplicates aren't a failure mode — they're the price of guaranteed delivery. The question is what you do with them.

## Moving the guarantee to the write boundary

You can't control how many times a message is delivered. You *can* control what happens when the same message is written twice.

An **idempotent write** produces the same result whether it runs once or ten times. The mechanism is an **idempotency key** — a stable identifier attached to each logical operation. At the database level:

```sql
INSERT INTO events (id, device_id, metric, value, ts)
VALUES ($1, $2, $3, $4, $5)
ON CONFLICT (id) DO NOTHING;
```

The first time this runs, the row is inserted. Every subsequent time with the same `id`, nothing happens. The transport layer can retry as many times as it wants — the result is always exactly one row.

## What makes a good idempotency key

The key must be **deterministic** — the same event must always produce the same key, across retries, restarts, and reprocessing.

**Good sources:**
- Kafka message coordinates: `topic + partition + offset` — globally unique within a cluster, never changes on redelivery
- A hash of stable event fields: `SHA-256(device_id + event_type + event_timestamp)`
- A business-level natural key: `order_id`, `transaction_id`

**Bad sources:**
- `UUID.randomUUID()` — a new UUID on each retry defeats the entire point
- `System.currentTimeMillis()` — not stable across retries

The Kafka offset is usually the cleanest choice: it's already there, it's unique per partition, and it never changes on redelivery.

## The commit-after-write pattern

The ordering matters — write to the sink first, commit the offset only after. Using Kafka:

```java
@KafkaListener(...)
public void consume(List<ConsumerRecord<String, Event>> records,
                    Acknowledgment ack) {
    List<Row> rows = records.stream()
        .map(r -> Row.of(
            r.topic() + "_" + r.partition() + "_" + r.offset(), // idempotency key
            r.value()
        ))
        .collect(toList());

    database.upsertBatch(rows);  // ON CONFLICT (id) DO NOTHING
    ack.acknowledge();           // commit offset only after write succeeds
}
```

Now trace the two failure scenarios:

**Failure before commit:** The write succeeds but `ack.acknowledge()` never runs — a crash, a timeout, a network drop. Kafka redelivers the same messages with the same offsets. Same idempotency keys. `ON CONFLICT DO NOTHING`. The rows already exist — nothing changes.

**Failure before write:** The write fails before any rows land. No commit happens. Kafka redelivers. The write runs again.

## "But Kafka already has exactly-once — doesn't it?"

Sort of. Kafka's [transactional API](https://www.confluent.io/blog/transactions-apache-kafka/) (`transactional.id` + `enable.idempotence=true`) does give you exactly-once semantics — but only within the broker. Under the hood, each producer gets a unique Producer ID and per-partition sequence numbers; the broker uses these to deduplicate retried sends. `transactional.id` goes further: it fences zombie producers — when a new instance registers the same `transactional.id`, the broker bumps the epoch, and the old instance gets a `ProducerFencedException` on its next transaction attempt. It also lets a producer write to multiple partitions atomically via a transaction coordinator. On the consumer side, `isolation.level=read_committed` means the consumer only sees messages from committed transactions — aborted writes are invisible.

```
# Kafka transactions work here:
Topic A  →  consumer  →  [process]  →  Topic B
         └──────── atomic: all or nothing ────────┘

# They stop here:
Topic A  →  consumer  →  [process]  →  Postgres
         └── Kafka's guarantee ends ──┘    ↑
                                     you're on your own
```

For Kafka-to-Kafka pipelines — a stream topology reading from one topic and writing to another — this gives you genuine exactly-once. It's the right tool for that case.

The boundary is the broker. [Kafka explicitly does not support two-phase commit with external systems](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging). The moment your sink is a database, an object store, or any external service, Kafka's transaction coordinator has no authority there — the write happens outside the transaction, and retries are unavoidable regardless of your producer configuration.

Application-level idempotency is the more general solution precisely because it makes no assumption about the transport. It works for any sink, adds no broker-side coordination overhead, and keeps the correctness logic where it's easy to test and reason about — at the write.

## Where this breaks down

`ON CONFLICT DO NOTHING` is safe for inserts. Updates — incrementing counters, maintaining the latest value — are a different problem. Don't update in place; store events and derive aggregates at read time. The idea: append immutable events, reconstruct current state by replaying them — optionally snapshotting to avoid full log replay on every read. It's the right model for the problem, but a significant architectural commitment: schema evolution is painful, snapshot management adds operational overhead, and the tooling is immature. Not a casual swap.

There's also a subtler limit: **intentional reprocessing to correct data**. If you reset Kafka offsets to replay messages after fixing a bug in your transformation logic — or after a schema change that should alter how events map to rows — the same `topic + partition + offset` coordinates produce the same idempotency key. `ON CONFLICT DO NOTHING` preserves the old, incorrect data silently. The job completes with no errors; stale records stay. For correction-style reprocessing, switch to a content-based key — `SHA-256(device_id + event_type + event_timestamp)` — or a business-level natural key scoped to the logical event, not its position in the log. (Catching up on genuinely missed messages works fine — only existing rows are skipped, new ones insert correctly.)

It also doesn't help with **non-idempotent side effects**. If processing a message sends an email, charges a card, or calls an external API, you need idempotency support from that system too (Stripe's [idempotency keys](https://stripe.com/docs/api/idempotent_requests) are exactly this pattern applied to their payments API). The database write can be idempotent; the downstream call must be too.

---

The transport layer will always be unreliable. Work with it, not against it. Make every write idempotent, commit only after the write succeeds, and let the transport retry as often as it needs to. The result is exactly-once semantics built on at-least-once infrastructure. For systems that can't afford to lose data, it's the only approach that works.
