---
title: "System Design Fundamentals: High Availability, Load Balancing, and Background Jobs"
excerpt: "Part 2 covers the patterns that keep systems running when things go wrong — redundancy, failover, load balancing strategies, and offloading work to background jobs."
tags:
  - Architecture
  - Backend
  - DevOps
date: "2026-02-26"
featured: false
coverEmoji: "🏗️"
---

In Part 1, I covered the theoretical foundations — scalability, latency vs throughput, and the CAP theorem. This part is about the patterns you actually implement to turn those concepts into systems that stay up. These are the patterns I wish I had understood before my first on-call rotation.

## What High Availability Actually Requires

High availability (HA) is usually expressed as a percentage of uptime — 99.9% ("three nines") means about 8.7 hours of downtime per year. 99.99% means about 52 minutes. Every additional nine is roughly 10x harder to achieve than the last.

The foundational principle of HA is **eliminating single points of failure**. A single point of failure (SPOF) is any component whose failure brings down the system. A load balancer sitting in front of two application servers is not HA if that load balancer is a single instance — you have just moved the SPOF from the app tier to the networking tier.

The two mechanisms for achieving HA are **active-active** and **active-passive** redundancy.

In an **active-active** setup, multiple nodes handle traffic simultaneously. When one fails, the others absorb its traffic. This gives you better resource utilisation and zero failover latency. The trade-off is complexity: all active nodes need to share state, and writes need to be coordinated or routed consistently.

In an **active-passive** setup, a standby node replicates the primary's state but does not serve traffic. When the primary fails, the passive node promotes itself and takes over. Failover takes seconds to minutes (depending on your health check interval and promotion process). You pay for a node that sits idle most of the time, but writes are simple — they all go to the primary.

For stateless services — API servers, worker processes — active-active is straightforward. For stateful services — databases, session stores, message brokers — the choice is harder and usually comes down to how much write coordination complexity you can tolerate.

## Availability Patterns in Practice

Beyond redundancy, there are a set of patterns that prevent cascading failures and keep your system responsive even when downstream dependencies degrade.

**Failover with health checks** is the baseline. Your load balancer polls each instance on a health endpoint. If an instance fails three consecutive checks, it is removed from rotation. If it recovers, it is added back. The key tunable is the check interval and failure threshold — too aggressive and you flap on momentary blips; too conservative and you route traffic to a broken node for too long.

**Circuit breakers** protect a service from hammering a dependency that is already struggling. When failure rate to a downstream service exceeds a threshold, the circuit "opens" and subsequent calls fail immediately without actually making the network request. After a timeout, the circuit enters a half-open state and allows one probe request through. If it succeeds, the circuit closes. If not, it opens again.

```js
// circuit-breaker.js
class CircuitBreaker {
  constructor(threshold = 5, timeout = 30000) {
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = Date.now();
  }

  async call(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN — fast failing');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}
```

Without circuit breakers, a slow database or third-party API will cause your thread pool to fill with waiting requests, your response times to climb, your error rate to spike, and your entire application to become unresponsive — even for endpoints that have nothing to do with the failing dependency.

**Retry with exponential backoff and jitter** handles transient failures. A simple immediate retry after a network timeout often just adds to the load on an already struggling service. Exponential backoff spaces retries out: wait 100ms, then 200ms, then 400ms. Jitter adds randomness to prevent a thundering herd of clients all retrying at the same moment after a shared outage.

```js
// retry.js
async function retryWithBackoff(fn, maxRetries = 3, baseDelay = 100) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const jitter = Math.random() * baseDelay;
      const delay = baseDelay * Math.pow(2, attempt) + jitter;
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

## Load Balancing Strategies

A load balancer is not just "round robin between N servers." The routing strategy has a significant impact on both performance and availability.

**Round robin** distributes requests sequentially across all healthy instances. Simple and effective when all instances are identical and requests are roughly equal in cost. Falls apart if some requests are 100x more expensive than others — a slow request on one node does not prevent more requests from being routed to it.

**Least connections** routes each new request to the instance with the fewest active connections. Better for workloads with variable request duration. Most hardware and software load balancers (NGINX, HAProxy, AWS ALB) support this.

**IP hash** consistently routes a given client IP to the same backend instance. Used when you need soft session affinity — the user's requests should land on the same server to avoid cache misses or state synchronisation. The downside is uneven distribution when a small number of IPs generate most of the traffic.

**Weighted routing** assigns a weight to each instance. Send 80% of traffic to the primary instance and 20% to a new version during a canary deploy. Indispensable for zero-downtime rollouts.

| Strategy | Best for | Watch out for |
| --- | --- | --- |
| Round robin | Stateless, uniform cost | Variable request duration |
| Least connections | Long-lived or variable requests | Overhead on high-throughput systems |
| IP hash | Session affinity | Hotspots from a few large IPs |
| Weighted | Canary deploys, gradual rollouts | Manual weight management |

## Offloading Work with Background Jobs

One of the highest-leverage architectural decisions you can make is identifying which work in your request path does not need to happen synchronously — and moving it out.

Consider a user registration flow. The user submits the form. Your API needs to create the account record, hash the password, and return a 200. It does **not** need to send the welcome email, provision their default workspace, index them in your search engine, or fire analytics events — at least not before the response. Those can all be background jobs.

The pattern is: write a job record to a queue (Redis, SQS, RabbitMQ), return the response immediately, and let a separate worker process consume and execute the job asynchronously.

```js
// routes/auth.js — the API handler
router.post('/register', async (req, res) => {
  const user = await User.create({
    email: req.body.email,
    passwordHash: await bcrypt.hash(req.body.password, 12),
  });

  // Enqueue background jobs — do not await
  await queue.add('send-welcome-email', { userId: user.id });
  await queue.add('provision-workspace', { userId: user.id });

  res.status(201).json({ id: user.id });
});
```

```js
// workers/email.js — runs in a separate process
queue.process('send-welcome-email', async (job) => {
  const user = await User.findById(job.data.userId);
  await emailService.sendWelcome(user.email);
});
```

The benefits are significant. Your API response time drops because you have eliminated the email send latency (often 100–500ms). Your system is more resilient — if the email service is down, the job stays in the queue and retries; it does not fail the registration. And your workers can be scaled independently of your API tier.

The important design considerations for background jobs:

- **Idempotency** — your job handler must produce the same result if it runs twice. Jobs can and will be retried. Sending two welcome emails is worse than sending zero.
- **Dead letter queues** — jobs that fail after all retries should go to a dead letter queue for inspection, not silently vanish.
- **Observability** — you need visibility into queue depth, job processing rate, and failure rate. A backed-up queue is often the first sign that a downstream dependency is degrading.

## Key Takeaways

- **Eliminate SPOFs at every tier** — your redundant app servers do not help if your load balancer is a single instance.
- **Active-active for stateless services, active-passive for stateful** — session stores and databases need careful coordination before going active-active.
- **Circuit breakers prevent cascading failures** — fast-failing when a dependency is down protects your entire service, not just the affected endpoint.
- **Retry with jitter, not immediately** — thundering herds after an outage make recovery slower, not faster.
- **Move non-critical work out of the request path** — background jobs cut latency, improve resilience, and decouple failure domains.
- **Idempotency is non-negotiable for background jobs** — at-least-once delivery is the default; design for it.
