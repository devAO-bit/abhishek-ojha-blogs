---
title: "System Design Fundamentals: Scalability, Latency, and the CAP Theorem"
excerpt: "Before you can design systems that survive real traffic, you need to deeply understand scalability, latency vs throughput, and the CAP theorem — the three concepts that underpin every distributed architecture decision."
tags:
  - Architecture
  - Backend
  - Performance
date: "2026-02-26"
featured: true
coverEmoji: "🏗️"
---

I used to treat system design as a collection of buzzwords — "just throw Kafka at it" or "add a cache layer" — until a production incident made it painfully clear that I had been pattern-matching without understanding the underlying trade-offs. This two-part series is my attempt to document what I actually learned.

## What Scalability Really Means

Scalability is not the same as performance. A system is performant if it responds quickly under its current load. A system is scalable if it can **continue** to respond quickly as load grows — without requiring a redesign.

There are two axes of scaling. **Vertical scaling** means adding more CPU, RAM, or disk to an existing machine. It is simple, requires no code changes, and hits a hard ceiling fast — both financially and physically. **Horizontal scaling** means adding more machines and distributing the workload across them. It is cheaper at scale and theoretically unbounded, but it introduces the complexity that makes distributed systems hard: coordination, consistency, and partial failure.

The key insight from Werner Vogels' writing on scalability is this: an increase in load should be handleable by adding resources proportionally, without any other changes. If doubling your traffic requires you to rewrite your data layer, your system is not scalable — it is fragile.

Before choosing a scaling strategy, you need to understand your bottleneck. Is it CPU? I/O? Memory? Network bandwidth? A system that is CPU-bound scales differently than one that is I/O-bound. Throwing more servers at a database that has a single write leader will not help — the bottleneck is the write path, not the compute.

## Latency vs Throughput

These two metrics are frequently confused, and optimizing for the wrong one is a common source of architectural mistakes.

**Latency** is how long a single operation takes — the time between sending a request and receiving a response. **Throughput** is how many operations a system can handle per unit of time. They are related but not the same, and they often trade off against each other.

A batch processing pipeline optimized for throughput might buffer writes and flush them every 500ms. That is great for throughput — you amortize the cost of I/O over many operations — but your latency for any individual write is now at least 500ms. A real-time API cannot make that trade.

The numbers that matter for most web systems (roughly, from lowest to highest latency):

| Operation | Approximate Latency |
| --- | --- |
| L1 cache read | ~1 ns |
| RAM read | ~100 ns |
| SSD read | ~100 µs |
| Network round-trip (same DC) | ~1 ms |
| Database query (indexed) | ~1–10 ms |
| Network round-trip (cross-region) | ~100 ms |

These numbers matter because they set your intuition. If a single user request makes 10 sequential database calls, your minimum latency is 10–100ms from the DB alone — before any application logic. That is why N+1 query problems kill performance at scale.

The goal for most systems is to **maximise throughput while keeping latency within an acceptable percentile bound** — usually p99 or p999, not average. Averages hide the tail, and it is the tail that makes users close the tab.

## The CAP Theorem — And Why You Only Really Choose One Thing

The CAP theorem states that a distributed system can guarantee at most two of three properties: **Consistency**, **Availability**, and **Partition Tolerance**.

- **Consistency** — every read returns the most recent write
- **Availability** — every request receives a response (not an error or timeout)
- **Partition Tolerance** — the system continues operating when network partitions occur

Here is the part most explanations skip: **you do not actually get to choose between all three**. Network partitions happen. Your inter-service network will fail. Your cross-AZ link will drop packets. Partition tolerance is not optional in any distributed system — it is a fact of the environment.

So the real choice is: **when a partition occurs, do you sacrifice Consistency or Availability?**

**CP systems** (Consistency + Partition Tolerance) return an error or timeout when they cannot guarantee the response reflects the latest write. HBase, Zookeeper, and most relational databases in their default configuration lean CP. Use CP when data correctness is non-negotiable — financial transactions, inventory counts, anything where stale data causes real harm.

**AP systems** (Availability + Partition Tolerance) return a response even if it might be stale. Cassandra, CouchDB, and DNS are AP. Use AP when it is acceptable to serve slightly old data in exchange for staying online — user profile reads, product catalog, feature flags.

This is not a permanent architectural decision baked into your whole system. Individual services and individual data stores make this trade independently. Your payment service should be CP. Your product recommendation service can be AP.

## Consistency Patterns

Once you have accepted that some systems will be AP, you need a mental model for how consistency is maintained over time.

**Strong consistency** means that after a write completes, all subsequent reads — from any node — will return that value. This requires coordination between nodes on every write, which is expensive. It is what you get from a single-leader relational database with synchronous replication.

**Eventual consistency** means that if no new writes occur, all replicas will converge to the same value — eventually. The window of inconsistency is typically milliseconds to seconds under normal operation, but can widen during failures. DynamoDB in its default mode is eventually consistent. Most DNS changes are eventually consistent.

**Weak consistency** makes no guarantees. After a write, reads may or may not reflect it. VoIP and live video streaming use weak consistency — a dropped frame is acceptable; blocking to retrieve it is not.

For most application data, eventual consistency is the right default. The question to ask is: what happens in your application if a user reads a value that is 200ms stale? If the answer is "nothing significant," you can use an AP store with eventual consistency and gain meaningful availability and performance benefits.

## Key Takeaways

- **Scalability is not performance** — a system is scalable if adding resources handles additional load without redesign.
- **Know your bottleneck before scaling** — throwing servers at an I/O-bound problem does not help.
- **Optimise for p99 latency, not average** — averages hide the tail that hurts real users.
- **Partition tolerance is mandatory** — CAP is really a choice between Consistency and Availability under failure.
- **CP vs AP is a per-service decision** — your payment flow and your recommendations feed have different requirements.
- **Eventual consistency is the right default** for most read-heavy application data — understand the staleness window and design around it.
