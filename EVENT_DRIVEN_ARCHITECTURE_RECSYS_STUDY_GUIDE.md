# Event-Driven Architecture for Large-Scale Recommendation & Personalization Systems
### A Staff-Level Study Guide

---

## Table of Contents

1. [Foundations](#1-foundations)
2. [Core Concepts](#2-core-concepts)
3. [Architectural Patterns](#3-architectural-patterns)
4. [System Design Considerations](#4-system-design-considerations)
5. [Trade-offs and Pitfalls](#5-trade-offs-and-pitfalls)
6. [Infrastructure & Tooling](#6-infrastructure--tooling)
7. [Real-World System Design Examples](#7-real-world-system-design-examples)
8. [Python-Oriented Examples](#8-python-oriented-examples)
9. [Mental Models & Intuition](#9-mental-models--intuition)
10. [Advanced Topics](#10-advanced-topics)
11. [Staff-Level Decision Frameworks](#11-staff-level-decision-frameworks)
12. [Learning Roadmap](#12-learning-roadmap)

---

## 1. Foundations

### What is Event-Driven Architecture?

Event-Driven Architecture (EDA) is a software design paradigm where the **flow of data is driven by events** — discrete, immutable records of things that have happened — rather than by direct invocations between services.

A system is event-driven when:
- Services communicate by **emitting and consuming events**, not calling each other directly.
- An event represents a **fact about the past**: `user_clicked_item`, `recommendation_served`, `model_scored_request`.
- Producers and consumers are **decoupled in time and space** — the producer does not know who will consume its event, or when.

Three sub-paradigms often conflate:

| Pattern | Communication | Coupling | Durability |
|---|---|---|---|
| **Event Notification** | Fire-and-forget; consumers react | Loose | Ephemeral (usually) |
| **Event-Carried State Transfer** | Event contains full payload; no follow-up needed | Very loose | Varies |
| **Event Sourcing** | Log of events is the system of record | Structural | Permanent |

At Staff level, the key mental shift is: **events are not just messages — they are a shared, persistent, replayable log of truth.** This distinction drives most architectural decisions.

---

### Why EDA Is Well-Suited for Recommendation & Personalization Systems

**Real-time recommendations** require that user behavior — a click, a dwell, a purchase — is immediately propagated to feature computation, model serving, and feedback loops. A synchronous request-response chain cannot absorb this fan-out efficiently. Consider a single user click:

```
Click event → feature store update (user context)
            → session model update (online learning)
            → diversity/dedup filter update
            → A/B experiment logging
            → downstream analytics
```

In a request-response world, this fan-out requires either synchronous parallelism (tight coupling, cascade failures) or a synchronous "fire calls to N services" pattern that becomes a latency liability. EDA lets all these consumers react to the same event independently, at their own pace, with independent SLAs.

**User activity tracking** — clicks, impressions, dwell time, scroll depth — is naturally append-only. You never "update" an impression; you append new facts. A log-structured event system is architecturally aligned with this semantics. Furthermore:
- Volume is extremely high (billions of events/day at scale).
- Loss tolerance varies per event type (impression loss tolerable; purchase loss is not).
- You need replayability to backfill features or retrain models.

**Feature pipelines and online inference** are inherently streaming transformations. The user's feature vector is a derived view of their event history. Computing it in real time means transforming a stream of events into a state. This is precisely what stream processors do.

---

### When EDA Is Preferable to Request-Response in ML Systems

**Prefer EDA when:**

| Scenario | Why EDA wins |
|---|---|
| Fan-out writes (one event triggers N consumers) | Each consumer decouples its SLA |
| Audit trails and reproducibility (feature backfills, model retraining) | Log is replayable |
| High-volume, write-heavy workloads (user interaction streams) | Append-only log is cheap |
| Loose coupling across team/service boundaries | Schema contract replaces API contract |
| Temporal buffering needed (producer faster than consumer) | Broker absorbs burst |
| Debugging and root-cause analysis at scale | Every signal is logged; queryable |

**Do NOT default to EDA when:**

| Scenario | Why request-response wins |
|---|---|
| Synchronous user-facing request requiring immediate response | Eventual consistency is unacceptable |
| Simple CRUD where coupling is intentional and bounded | Overhead without benefit |
| Low event volume, simple fan-out | Kafka for a toy system is over-engineering |
| Strong transactional consistency requirements across services | Saga complexity exceeds benefit |
| Initial iteration / MVP | Operational overhead is prohibitive |

The clearest anti-pattern: using EDA for operations that **require an immediate, consistent response**. A model inference call at p50=10ms cannot be decomposed into async events unless you pre-compute and cache.

---

## 2. Core Concepts

### Events as Behavioral Signals

In recommendation systems, events are the raw substrate from which all understanding of users and items derives. Conceptually:

```
User intent
  → behavioral signal (event)
    → feature extraction
      → model input
        → ranking output
          → new event (impression served)
            → future behavioral signal
```

This is a **closed loop**: events generate features, features influence recommendations, recommendations generate new events. Understanding this cycle is critical for reasoning about feedback loops, popularity bias, and exploration/exploitation.

**Event taxonomy in recommendation systems:**

```
Interaction Events:
  - click, long_click, dwell (>N seconds), scroll_depth
  - add_to_cart, purchase, return
  - share, save, bookmark
  - explicit_rating, thumbs_up/down

Negative Signals:
  - skip, swipe_left, hide, block
  - short_dwell (<N seconds) on recommendation

System Events:
  - recommendation_served (impression)
  - recommendation_requested (query)
  - model_scored (internal, for debugging)

Contextual Events:
  - session_start, session_end
  - app_foreground, app_background
  - location_update (if applicable)
```

**The semantics matter enormously.** A "view" event may mean different things depending on how it's instrumented. Staff engineers push hard on event definition rigor because ambiguous events propagate ambiguity into every model trained on them.

---

### Producers and Consumers

**Producers** in ML systems:

- **Frontend clients** (web, mobile): User interaction events. These are unreliable producers — network issues, client-side bugs, ad blockers. Expect loss rates of 0.1–5%.
- **API Gateway / BFF (Backend-for-Frontend)**: Can emit server-side events as a reliability backstop.
- **Recommendation serving layer**: Emits impression events with full context (model version, features used, ranking scores) — critical for training data generation.
- **Batch jobs**: Emit derived or aggregated events after processing (e.g., "user_cluster_assignment_updated").
- **Feature pipelines**: Emit feature materialization events.

**Consumers** in ML systems:

- **Feature store writers**: Consume interaction events to update user feature vectors (online store).
- **Training data pipelines**: Consume labeled events (impression + outcome) for model training.
- **Ranking/scoring services**: May consume feature update events to refresh in-memory caches.
- **Real-time aggregation services**: Compute rolling counts (clicks in last 1h, 24h) from event streams.
- **Notification services**: Trigger personalized push notifications based on behavioral events.
- **Analytics/BI**: Long-tail consumer; less latency-sensitive but needs completeness.
- **Abuse/fraud detection**: Consume events for anomaly detection.

**A critical design choice: should consumers be stateful or stateless?**

Stateless consumers are easier to scale and restart, but force state into an external store (the feature store, a database). Stateful consumers (streaming jobs that maintain in-memory state) are powerful but require careful state management, checkpointing, and rebalancing strategies.

---

### Event Brokers: Queue vs. Log-Based Systems

This is one of the most important conceptual distinctions in EDA.

**Queue-based (e.g., RabbitMQ, SQS):**
- Message is consumed and **deleted**.
- Designed for work distribution: "process this task once."
- Naturally models task queues, job dispatch.
- Poor fit for ML systems where you need replay, multiple consumers, and audit trails.

**Log-based (e.g., Kafka, Kinesis, Pulsar):**
- Messages are written to an **append-only, durable, ordered log**.
- Consumers maintain their own offset (position in the log).
- Messages are **not deleted on consumption** — they are retained for a configured period (hours to weeks to forever via log compaction).
- Multiple independent consumer groups each get the full stream.
- Replay is first-class: reset your offset, reprocess the full history.

**Why log-based systems dominate ML pipelines:**

```
Feature backfill scenario:
  - A bug in your feature computation logic went unnoticed for 3 days.
  - 3 days of user feature vectors are corrupt.
  - With a queue: those events are gone. You cannot rebuild.
  - With a log: reset consumer offset to 3 days ago, replay all events,
    recompute features. Problem solved.

Training data generation:
  - Model v2 requires a new label signal that wasn't captured 6 months ago
    but the raw behavioral events were retained.
  - With a log: backfill the derived label by replaying events through
    new transformation logic.
  - With a queue: impossible.
```

The replay capability is not a nice-to-have — it is foundational to the operational model of ML systems. **Log-based event storage is to ML pipelines what Git is to code.**

---

### Topics, Partitions, Ordering, and Their Impact on ML

**Topics** are logical namespaces for events. Good topic design:

```
# Too coarse:
user_events                   (everything mixed, consumers filter, wastes IO)

# Too fine:
user_click_desktop_search_v2  (proliferation, hard to maintain)

# Balanced:
user_interactions             (clicks, dwell, saves — interaction class)
recommendation_impressions    (served recommendations with metadata)
item_updates                  (catalog changes)
model_outputs                 (scored requests, for offline debugging)
```

**Partitions** are the unit of parallelism and ordering. Within a partition, events are **strictly ordered** by offset. Across partitions, no ordering guarantee.

**Partition key selection is a consequential design decision in ML:**

| Key | Ordering guarantee | Pros | Cons |
|---|---|---|---|
| `user_id` | All events for a user go to same partition | Feature computation sees ordered user history | Hot partitions for power users |
| `item_id` | All events for an item are co-located | Useful for item popularity counters | Power-law distribution creates imbalance |
| Random | Even distribution | Best throughput | No ordering; hard to reason about user sessions |
| `session_id` | Session-level ordering | Good for session-based models | Sessions span partitions; cross-session ordering lost |

**Impact on feature correctness:**

Imagine computing a "clicks in last 30 minutes" feature for user U. If U's events are spread across partitions, your streaming job sees them out of order. You compute a potentially incorrect aggregate. This is especially dangerous at boundary conditions (events near the window edge).

**Impact on model consistency:**

If impression events and click events for the same (user, item, timestamp) tuple land in different partitions, and your join processor reads them at different rates, you may join the wrong click to the wrong impression. This poisons training labels. Partitioning impressions and clicks by the same key (e.g., `request_id` or `user_id`) and using windowed joins is the mitigation.

---

### Event Schemas and Contracts

Events are a team-to-team contract. When Team A produces events consumed by Team B's feature pipeline, a schema change is a breaking API change — except it's worse, because the breakage may be silent (old code processes new fields as null) and may poison weeks of training data before detection.

**Schema evolution principles:**

```
# v1 schema
{
  "event_type": "item_click",
  "user_id": "u123",
  "item_id": "i456",
  "timestamp_ms": 1746647000000
}

# v2: added context field — BACKWARD COMPATIBLE (consumers ignoring it still work)
{
  "event_type": "item_click",
  "user_id": "u123",
  "item_id": "i456",
  "timestamp_ms": 1746647000000,
  "context": {
    "surface": "homepage",
    "position": 3
  }
}

# v3: renamed user_id to viewer_id — BREAKING CHANGE
# Old consumers silently receive null for user_id
```

**Backward compatibility**: new consumers can read old events. Required when deploying new consumers against historical replays.

**Forward compatibility**: old consumers can read new events. Required for rolling deploys where old consumer code runs alongside new producer code.

**Governing this in practice:**

- **Schema registry** (Confluent Schema Registry, AWS Glue Schema Registry): Centralized schema store. Producers register schemas; consumers validate on read. Prevents incompatible schemas from being published.
- **Avro / Protobuf / Thrift** for binary-efficient, schema-aware serialization. Prefer over JSON at high volume (5–10x size reduction, built-in schema enforcement).
- **Schema versioning policy**: Additive changes only (new optional fields). Semantic changes (renaming, type changes) require a new topic with a migration period.

**Evolving feature definitions:**

One of the most insidious problems in ML EDA systems. Suppose `dwell_time_seconds` changes its instrumentation — the client now reports 0 for items that were previously not tracked. Old events mean "unknown"; new events mean "0 seconds dwell." If your feature pipeline treats both the same, your model gets a data distribution shift. Solutions:

- Version events semantically, not just structurally.
- Store raw events forever; derive features in versioned transformation jobs.
- Never mutate the semantics of an existing field; always add a new field.

---

### Idempotency and Deduplication in User-Event Streams

At high volume, **duplicate events are inevitable**. Sources:

- Client-side retry on timeout (the server processed the event but response was lost).
- Broker redelivery after consumer failure before offset commit.
- Network retransmission.
- Edge caching or CDN replay.

Duplicate events cause incorrect feature values:
- "User clicked item X twice" when they clicked once → inflated engagement signals.
- "Purchase recorded twice" → incorrect revenue attribution, corrupted training labels.

**Deduplication strategies:**

```
1. Event-level deduplication key:
   Every event carries a globally unique event_id (UUID or hash of key fields).
   The consumer maintains a dedup window (e.g., Redis set with TTL).

   Pseudocode:
     if event.event_id in dedup_cache:
       drop event
     else:
       dedup_cache.add(event.event_id, ttl=24h)
       process(event)

   Trade-off: memory scales with event volume × dedup window.

2. Idempotent writes:
   Design the downstream write to be inherently idempotent.
   "Set user_click_count[user, item] = max(current, new_count)"
   rather than incrementing.
   Works when you can express the operation as an upsert with a key.

3. Log-based dedup:
   Use the event log offset as the dedup key.
   If you replay from a specific offset, you get each event exactly once
   from that point. Works for batch reprocessing but not real-time dedup
   across producers.

4. Exactly-once in Kafka:
   Kafka 0.11+ supports idempotent producers and transactional APIs.
   Combined with a transactional consumer, achieves exactly-once
   end-to-end. High overhead; use selectively.
```

**Design principle**: make your consumers idempotent first; only add dedup infrastructure when the cost of duplicates exceeds the cost of dedup complexity.

---

## 3. Architectural Patterns

### Pub/Sub: Decoupling Producers and Consumers

Publish/Subscribe is the foundational pattern: producers publish events to a topic; consumers subscribe and receive them independently.

**In recommendation systems, Pub/Sub solves the fan-out problem:**

```
Without Pub/Sub (tight coupling):
  UserInteractionService.recordClick(event)
    → calls FeatureStoreService.updateUserFeatures(event)
    → calls TrainingDataService.logLabel(event)
    → calls AnalyticsService.recordEngagement(event)
    → calls NotificationService.evaluateTriggers(event)

  Problems:
    - UserInteractionService must know all consumers.
    - Any consumer failure degrades click recording.
    - Adding a new consumer requires changing UserInteractionService.

With Pub/Sub:
  UserInteractionService.publishEvent(user_interaction_topic, event)

  Independent consumers:
    FeatureStoreConsumer      → reads user_interaction_topic
    TrainingDataConsumer      → reads user_interaction_topic
    AnalyticsConsumer         → reads user_interaction_topic
    NotificationConsumer      → reads user_interaction_topic

  Properties:
    - UserInteractionService has zero knowledge of consumers.
    - Consumer failures are isolated; they catch up when healthy.
    - New consumers require zero changes to the producer.
```

**Pub/Sub is not a silver bullet.** The coupling moves from code to schema. The event schema becomes the shared contract, and schema changes can still break consumers — just silently and asynchronously instead of loudly and synchronously. You've traded compile-time coupling for runtime coupling. Staff engineers recognize this and invest heavily in schema governance.

---

### Event Streaming for Real-Time Feature Pipelines

The core pattern for real-time features in recommendation systems:

```
Raw events (Kafka)
  → Stream processor (Flink/Spark Streaming/custom)
    → Windowed aggregations (last 1h clicks, last 24h purchases)
    → User feature vectors
      → Online feature store (Redis/Cassandra)
        → Serving layer reads at inference time
```

**Windowing is where complexity lives:**

```
Tumbling window: [0:00–1:00], [1:00–2:00], ...
  - Non-overlapping, fixed-size. Simple. Good for hourly aggregates.
  - Issue: a click at 0:59 and 1:01 are in different windows.
    User activity near boundaries is split.

Sliding window: every 1m, compute last 60m.
  - Continuously updated. More computationally expensive.
  - Better for feature freshness.

Session window: group events within a session (gap-based).
  - Closes when idle for >N minutes.
  - Most semantically meaningful for recommendation features.
  - Complex to implement in distributed systems.
```

**The watermark problem:**

Stream processors need to know when a window is "done" so they can emit the aggregated result. But events arrive late (mobile clients with spotty connectivity, CDN buffering). The watermark is the processor's estimate of "how late can events be?" It's a trade-off:

- **Low watermark latency** (emit results quickly): risk including late events in the wrong window → incorrect features.
- **High watermark latency** (wait longer for late arrivals): feature freshness degrades → staleness.

In practice, for recommendation systems:
- Use a low watermark (1–5 second latency) for most features.
- Emit early results and handle updates gracefully.
- Log late events separately for offline analysis.

---

### Event Sourcing for User State Reconstruction

Event sourcing stores state as an **ordered sequence of events**, not a snapshot. Current state is derived by replaying events.

```
Traditional (snapshot-based):
  users_table: {user_id: u1, click_count: 47, last_seen: T3, ...}
  → fast reads, but history is lost

Event-sourced:
  user_events: [
    {user_id: u1, type: click, ts: T1, item: i1},
    {user_id: u1, type: click, ts: T2, item: i2},
    {user_id: u1, type: purchase, ts: T3, item: i3},
    ...
  ]
  current_state = reduce(user_events, initial_state, apply_event)
```

**Why event sourcing is powerful for recommendation systems:**

1. **Recompute any feature as of any point in time.** Critical for point-in-time correct training data — your model should see features as they existed at the moment of recommendation, not as they exist today.

2. **Model retraining with new feature definitions.** If you add a new feature, replay the event log with new transformation logic to backfill it.

3. **Debugging feature drift.** "Why did user U get recommendation R on date D?" Replay U's events up to D, recompute features, re-score model → exact reproducibility.

4. **A/B experiment analysis.** Assign treatment based on event history, then re-score a counterfactual recommendation policy.

**The snapshot problem at scale:**

Replaying millions of events to compute a user's current state is expensive. The solution is **periodic snapshotting**: take a state snapshot every N hours/days. To reconstruct current state, load the snapshot and replay only events since the snapshot.

```
State at T = apply(events[snapshot_ts → T], snapshot_at_ts)
```

This is the same logic used in database WAL-based recovery and CRDT systems.

**Operational reality**: full event sourcing is rarely adopted wholesale in ML systems. Instead, teams adopt event sourcing principles selectively:
- Raw events are always retained (the "event log").
- Derived states (feature vectors) are computed from events.
- The derived state can be recomputed on demand from the raw log.

---

### CQRS: Separating Write and Read Paths

CQRS (Command Query Responsibility Segregation) separates the write model (commands → events) from the read model (queries → materialized views). In recommendation systems, this maps cleanly:

**Write path (event ingestion):**
```
User action
  → Command (RecordClick)
    → Validation & enrichment
      → Event (ClickRecorded) published to log
        → No immediate read concern
```

**Read path (materialized views for serving):**
```
Online feature store (Redis):
  → user:{id}:features → {click_count_1h: 12, last_item: "i789", ...}
  → Updated by stream processor consuming the event log

Recommendation cache:
  → user:{id}:recs → [i1, i2, i3, ...]
  → Pre-computed based on latest features

Serving layer:
  → Reads from materialized views only
  → Never touches the event log directly
```

**Why CQRS matters in recommendation systems:**

The write path optimizes for throughput (append to log, minimal processing, low latency) while the read path optimizes for query performance (Redis lookups, pre-computed vectors). Without CQRS, you'd need a single data store that handles both — a significant technical compromise in either direction.

**The consistency implication**: there is a lag between event ingestion (write path) and feature update (read path). This is acceptable in most recommendation scenarios — a click 200ms ago not yet reflected in the feature vector is tolerable. But it must be explicitly reasoned about and monitored.

---

### Saga Pattern: Coordinating Multi-Step ML Pipelines

Sagas manage distributed transactions that span multiple services, without a distributed lock. Instead of two-phase commit, a Saga is a sequence of local transactions, each publishing an event that triggers the next step. If a step fails, compensating transactions undo the work.

**ML pipeline application — model deployment saga:**

```
Saga: deploy_new_model

Step 1: ModelTrainingComplete event
  → Trigger: run offline evaluation
  → Publish: EvaluationComplete (with metrics) or EvaluationFailed

Step 2: EvaluationComplete event
  → Trigger: shadow traffic test (run new model on X% of traffic, log but don't serve)
  → Publish: ShadowTestPassed or ShadowTestFailed

Step 3: ShadowTestPassed event
  → Trigger: canary deployment (serve new model to 1% of users)
  → Publish: CanaryHealthy or CanaryDegraded

Step 4a (success): CanaryHealthy event
  → Trigger: full rollout
  → Publish: ModelDeploymentComplete

Step 4b (failure): CanaryDegraded event
  → Trigger: rollback (compensating transaction)
  → Publish: ModelRolledBack
```

**Choreography vs. orchestration:**

```
Choreography: each service knows what to do when it sees an event.
  + Fully decoupled; services have no knowledge of the overall flow.
  - Hard to understand the overall flow from code; distributed debugging.
  - Risk of circular event loops.

Orchestration: a central "saga orchestrator" commands each step.
  + Flow is explicit and traceable from one place.
  - Orchestrator becomes a coupling point and potential single point of failure.
```

For ML pipelines with complex conditional logic (different rollout paths depending on metric thresholds), orchestration is often preferable. For simple fan-out workflows, choreography is simpler.

**A critical failure mode with Sagas**: compensating transactions in ML are often impossible or lossy. You cannot "un-train" a model. You cannot "un-serve" a recommendation. Design Sagas in ML systems around gating (don't advance until confident) rather than rollback.

---

## 4. System Design Considerations

### Throughput and Latency Trade-offs in Real-Time Ranking

A recommendation system has hard latency requirements (typically p99 < 100–200ms for the full stack). The event-driven components must fit within this budget.

**Latency budget decomposition for a recommendation request:**

```
Total SLA: 100ms p99

  API Gateway:           5ms
  Feature retrieval:    20ms   ← reads from online feature store
  Candidate retrieval:  15ms   ← ANN index lookup
  Scoring:              30ms   ← model inference (N candidates × scoring)
  Re-ranking & filters: 10ms
  Response serialization: 5ms
  Network (client):     15ms   ← not controllable
  ----
  Total:               100ms
```

**Events do not sit in the serving critical path.** The serving layer reads from materialized views (feature store, candidate index) that were populated asynchronously by the event-driven pipeline. This is the key architectural insight: **EDA enables low-latency serving by pre-computing and materializing state, not by being fast on the critical path.**

The question then becomes: how fresh are the materialized views? And what happens to recommendation quality when features are stale?

**Throughput considerations:**

At 100M active users generating 10 events/second average:
- Peak: 10^9 events/second is unrealistic; assume 1–10M events/second peak.
- At 1KB average event size: 1–10 GB/second ingestion throughput.
- Kafka can handle this with appropriate partitioning (thousands of partitions across a large cluster).
- The stream processor (Flink/Spark) must keep up with this rate; lag is your enemy.

**The consumer lag metric is your system's health heartbeat.** A growing lag means your pipeline is falling behind. Features become stale. Eventually, if lag exceeds your Kafka retention window, you lose data permanently.

---

### Handling Bursty Traffic

Recommendation and personalization systems face severe burstiness: a viral tweet, a breaking news story, a major sale event, a sports game ending. Traffic can spike 10–100x in seconds.

**Anatomy of a burst:**

```
T+0s:  Normal load: 100K events/sec
T+5s:  Viral event triggers: 2M events/sec spike
T+10s: Kafka ingestion buffer absorbs spike (events queue up)
T+30s: Stream processors lag grows (they can't process 2M/s)
T+5m:  If burst subsides, processors catch up
T+15m: If burst persists, lag grows to hours; features become stale
```

**Mitigation strategies:**

1. **Elastic consumer scaling**: Auto-scale your stream processing fleet based on consumer lag. The lag metric should trigger auto-scaling events. Challenge: Kafka partition count bounds the parallelism ceiling. You cannot have more consumers than partitions. Over-partition by 3–5x anticipated peak consumer count.

2. **Priority queues / topic separation**: Not all events are equally important. Serve impression events (needed for training labels) on a high-priority topic. Analytics events go to a separate, lower-priority topic. During bursts, you deprioritize analytics without affecting ML-critical pipelines.

3. **Load shedding at producers**: During extreme load, producers can selectively drop or batch low-value events. "Drop 50% of scroll events, never drop purchase events." Requires careful event importance classification.

4. **Pre-warmed capacity**: For predictable bursts (Black Friday, scheduled events), pre-scale the consumer fleet hours in advance.

5. **Backpressure propagation**: If the feature store is overwhelmed, its write throughput drops, creating backpressure into the stream processor, which slows consumption, increasing Kafka lag. This is usually preferable to crashing the feature store.

---

### Backpressure in Streaming Pipelines

Backpressure is the mechanism by which a slow consumer signals upstream components to slow down (or buffer more aggressively).

```
Event source → Kafka → Stream processor → Feature store writes
                                              ↑
                                     Feature store overwhelmed:
                                     writes take 50ms instead of 5ms

Stream processor slows:
  - Blocking writes slow consumption rate
  - Kafka consumer lag increases

Backpressure propagates:
  - If processor has bounded internal buffers, it stalls
  - If Kafka has sufficient retention, events queue safely

Resolution:
  Option A: Scale feature store (add replicas)
  Option B: Batch writes (reduce write amplification)
  Option C: Write to a write-ahead buffer, async flush to feature store
  Option D: Drop non-critical feature updates (graceful degradation)
```

**The bounded buffer problem**: stream processors (Flink, Spark Streaming) have internal operator queues. If a downstream sink is slow and the processor's internal buffers fill, the processor blocks. This can cascade upstream, eventually causing the Kafka consumer to stop polling, which causes rebalancing in the consumer group, which interrupts all consumers. Managing buffer sizes and timeouts is an operational art.

**Practical guidance**: build explicit SLOs for consumer lag per topic. Alert at lag > 30 seconds for ML-critical features. Define a runbook for lag growth: when to scale, when to shed load, when to declare an incident.

---

### Data Consistency

#### Online vs. Offline Feature Parity

This is one of the most persistent and painful challenges in production ML systems.

**The training-serving skew problem:**

```
Offline (training):
  - Features computed from a batch job on historical events
  - Events joined with time-series features using a point-in-time join
  - Features as of time T are used to label the outcome at T+Δ

Online (serving):
  - Features retrieved from online store in real time
  - Computed by a stream processor consuming the live event stream

If the feature computation logic differs between offline and online:
  - Model trained on feature set F_offline
  - Model served on feature set F_online
  - Distribution shift = degraded model performance
  - Bug is often subtle and discovered only through A/B testing or metric analysis
```

**Mitigation**: **Feature platform unification** — compute features using the same transformation logic offline and online. Tools like Tecton, Feast, and Hopsworks try to enforce this. The architecture:

```
Single feature definition:
  transform(events) → feature_vector

Used in two modes:
  Streaming mode: transform(live_event_stream) → online store
  Batch mode:     transform(historical_events) → offline store / training data
```

This works when transforms are pure functions of events — which they often are not. Handling time zones, null semantics, and aggregation windows consistently across batch and streaming is an ongoing engineering investment.

#### Eventual Consistency and Model Staleness

In an event-driven pipeline, features are eventually consistent with the latest events. Staleness has two dimensions:

1. **Feature staleness**: the online feature store reflects events from T-Δ, not T. Δ is your pipeline lag.
2. **Model staleness**: the model was trained on data from T-N days ago. Even with fresh features, the model's weights reflect old patterns.

**Reasoning about staleness impact on recommendation quality:**

```
High staleness sensitivity:
  - News/trending content: an article from 10 minutes ago that's going viral
    won't be recommended if the feature store is 30 minutes stale
  - Flash sales: a user's recent purchase intent signal must propagate quickly

Low staleness sensitivity:
  - Long-term taste features (genre preference, brand affinity)
    change over weeks/months; 1h staleness is irrelevant
  - Item quality scores (average rating) change slowly

Design implication:
  - Invest in low-latency pipelines for high-sensitivity features
  - High-sensitivity features should drive the freshness SLA
  - Low-sensitivity features can be batch-computed
```

---

### Fault Tolerance

#### Replay Strategies for Rebuilding Features

Kafka's durable log enables feature replay. The replay capability should be a first-class operational tool, not an emergency procedure.

**Operational replay scenarios:**

```
Scenario 1: Bug in feature computation logic (most common)
  Action: Fix the bug, reset consumer group offset to T_bug_start,
          replay events through corrected logic.
  Risk: Replay may take hours; during replay, old (incorrect) features
        are still being served. Two-phase strategy:
          Phase 1: Replay into a shadow feature store.
          Phase 2: Atomic cutover to the new store.

Scenario 2: New feature definition for a new model
  Action: Run a new consumer group from offset 0 (or earliest needed),
          computing the new feature into a new key namespace.
          Existing features untouched.

Scenario 3: Data corruption in feature store (hardware failure, bug)
  Action: Wipe affected keys. Replay from Kafka to rebuild.
  Risk: Requires Kafka retention ≥ the feature's lookback window.
        A "last 90 days activity" feature requires 90 days of retention.
  Design implication: set Kafka retention >= max feature lookback window.

Scenario 4: Bootstrap new region
  Action: Mirror event stream to new region (MirrorMaker/cross-region replication).
          Run feature pipeline in new region consuming mirrored events.
```

**The retention budget**: long retention is expensive (storage) and requires careful capacity planning. A common pattern: use tiered storage (Kafka Tiered Storage, or Kafka → S3 via compaction) to retain events indefinitely at low cost, with hot data on local disk for fast consumption and cold data in object storage for replay.

#### Handling Dropped or Duplicated Events

**Dropped events:**

- Client-side drops (network failure before delivery): accept some loss for high-volume, low-value events (scroll depth). For critical events (purchase, explicit feedback), implement client-side retry with idempotency keys.
- Broker drops: with proper replication factor (≥3) and acks=all, Kafka does not drop committed events under normal hardware failure.
- Consumer drops: processor crashes after consuming but before committing offset. On restart, events are reprocessed from the last committed offset. This is "at-least-once" delivery — the norm.

**The at-least-once contract:**

Most production EDA systems operate under "at-least-once" delivery semantics. Events may be processed more than once; the system must tolerate this. For ML systems, common tolerance mechanisms:

```
Feature computation:
  - Idempotent write: "set click_count = max(current, new)" instead of increment
  - Dedup window: track recent event_ids, skip duplicates (as described in §2)
  - Windowed aggregation with event_time: duplicate events with same timestamp
    in the same window are naturally handled if you deduplicate by event_id
    before aggregating

Training labels:
  - Deduplicate impression → outcome join keys before training
  - Use event_id as the unique training example ID; dedup in the batch job
```

---

### Observability

Debugging a distributed, asynchronous, event-driven ML system without observability is like debugging a race condition blind. Observability in EDA requires specific instrumentation beyond standard service metrics.

**Metrics to instrument:**

```
Pipeline health:
  - Consumer lag per topic/partition (primary health signal)
  - Events processed per second (throughput)
  - Processing latency (event_time to feature_store_write_time)
  - Error rate per consumer

Feature store freshness:
  - feature_last_updated_ts per key (or percentile of staleness across keys)
  - % of inference requests that had features > Δ old

Data quality:
  - Schema validation failure rate (should be 0)
  - Null/missing field rates per event type
  - Event volume per type (sudden drops → producer issues)
  - Cardinality anomalies (sudden new user_ids → possible ID mapping bug)

End-to-end tracing:
  - event_id → feature_update_ts → inference_request_id
  - Enables "why did user U get recommendation R?" debugging
```

**Debugging feature drift with events:**

Feature drift (model performance degrades due to changing input distributions) is common in production ML. EDA helps debug it:

```
Step 1: Alert fires: CTR dropped 15% over 24 hours.
Step 2: Check feature store freshness metrics. All green.
Step 3: Query feature distribution over last 7 days.
        Observe: click_count_1h feature mean dropped 40%.
Step 4: Trace back: find the event stream that feeds click_count_1h.
        Observe: click events dropped 30% in volume starting at T.
Step 5: Trace to producer: find the frontend deployment at T that
        introduced a bug causing some click events to be dropped.
Step 6: Fix producer bug. Replay events from T using server-side
        click recovery (if available) or accept the data gap.
```

This root-cause workflow is only possible if: (a) you have per-event-type volume metrics, (b) you can trace features back to their source events, and (c) you have schema validation that would have caught a structural change.

**Distributed tracing for event pipelines:**

Propagate a trace context through the event chain:

```
User click → event carries trace_id: "t-abc123"
  → Kafka message includes trace_id in headers
    → Stream processor logs: {trace_id: "t-abc123", step: "feature_computed", ts: ...}
      → Feature store write logs: {trace_id: "t-abc123", step: "feature_written", ts: ...}
        → Inference request carries impression_id linked to trace_id
```

This enables querying your log aggregation system for all events related to a single user interaction.

---

### Multi-Region Considerations

Global recommendation systems must operate across regions for latency (serving users from a nearby region) and resilience (one region fails, others continue). EDA complicates this significantly.

**Geo-partitioned event streams:**

Option A: **Regional streams, global aggregation**
```
US users → US Kafka cluster → US feature store → US serving
EU users → EU Kafka cluster → EU feature store → EU serving

Cross-region aggregation:
  - Global popularity signals require aggregating across regions
  - Cross-region Kafka MirrorMaker replication (adds latency, cost)
  - Eventual consistency between regions: acceptable for recommendation
```

Option B: **Global stream with geo-routing**
```
All users → Global Kafka cluster (multi-region)
  → Region-local consumers read the global stream
  → Only process events relevant to their region

Problems:
  - Data residency / GDPR: EU user data must not flow to US
  - Cross-ocean replication latency: 50–100ms
  - Single point of failure risk (global cluster)
```

**Recommendation**: geo-partition your event streams to comply with data residency requirements. Use cross-region replication only for non-PII signals (item popularity, global trend signals).

**Cross-region consistency:**

When a user travels between regions (US → EU) or requests are served from different regions:
- User feature state should be consistent across regions.
- Options: (a) synchronous cross-region writes (high latency, unacceptable for serving), (b) eventual consistency (user sees slightly stale recommendations in new region), (c) follow-the-sun writes (user's home region is authoritative, others are replicas).

Most large-scale systems choose eventual consistency with a known lag SLA (e.g., cross-region sync within 60 seconds). This is usually invisible to users.

---

## 5. Trade-offs and Pitfalls

### Hidden Coupling via Shared Event Schemas

EDA replaces service-to-service coupling with event schema coupling. This is subtler and more dangerous because it's invisible in code reviews and often not enforced by tooling.

**Manifestations:**

1. **Semantic drift**: Event `item_view` originally meant "the user saw the item." A new team starts producing `item_view` for server-side pre-fetches. Consumers don't know the semantics changed. Training data now includes "views" the user never saw.

2. **Implicit ordering assumptions**: Consumer A assumes `user_created` always precedes `user_interaction`. But in a distributed system during high load, `user_interaction` can arrive first. The consumer silently skips the event or creates a corrupt state.

3. **Undocumented invariants**: "field X is always populated for mobile events but optional for web events." This lives in someone's head, not the schema.

**Mitigations:**

- Schema registry as the single source of truth.
- Semantic versioning of event types (not just structural versioning).
- Consumer contract testing: consumers declare the subset of event fields they depend on and the constraints they expect. CI validates these contracts before producer deploys.
- Event ownership documentation: each event type has an owning team, a SLA for schema stability, and a changelog.

---

### Data Quality Issues Propagating Through Pipelines

EDA creates long, chained data pipelines. A data quality issue at the source propagates through every downstream consumer. By the time the issue is detected (often in model metrics), it has corrupted weeks of training data.

**Cascade failure pattern:**

```
Mobile client bug (T=0):
  → Sends item_id as string "null" instead of omitting the field

Stream processor (T=0 to T+3 days):
  → Passes item_id="null" through; no validation

Feature store (T=0 to T+3 days):
  → Happily stores features keyed on item_id="null"

Training data pipeline (T+7 days):
  → Creates training examples with item="null" — a phantom item
  → Model learns associations with phantom item

Model deployed (T+14 days):
  → Recommends item "null" to some users
  → 404 errors in serving layer, user experience degraded
```

**Prevention requires defense in depth:**

1. **Schema validation at ingestion**: reject events with null required fields at the broker boundary.
2. **Data quality monitoring per event type**: daily volume, field null rates, cardinality checks. Alert on anomalies.
3. **Feature distribution monitoring**: track mean/stddev of each feature. Alert on distribution shifts.
4. **Training data audits**: validate training dataset statistics before training runs.

The key principle: **data quality debt compounds through ML pipelines faster than in traditional systems**. A 1% event loss is a rounding error in a counting dashboard but can systematically bias a recommendation model if it's correlated with a specific user segment or item type.

---

### Debugging Distributed, Asynchronous Systems

This is the primary operational challenge of EDA. Failures are non-deterministic, non-local, and often time-delayed. A bug in production may cause symptoms hours or days after the root cause.

**Staff-level debugging mental model:**

```
1. Establish the symptom timeline.
   - When did the metric degrade?
   - Was the degradation gradual (data quality) or sudden (deployment)?

2. Isolate the pipeline stage.
   - Use per-stage metrics (lag, throughput, error rate) to identify which
     component deviated from normal.

3. Examine the data at the boundary.
   - What does the input data to the failing stage look like?
   - Is it structurally valid? Is the volume correct?

4. Replay a sample.
   - Take a small window of events and replay them through the pipeline
     in a non-production environment. Can you reproduce the issue?

5. Check recent deployments.
   - Schema change? New producer version? Consumer code change?
   - Event sourcing is your friend: the log is immutable, so you can
     always replay the original events even after fixing the bug.
```

**The asynchrony trap**: when you see a symptom, your instinct is to look at what changed recently. But in EDA, the root cause may be an event that was ingested 2 hours ago, processed 30 minutes ago, and materialized 10 minutes ago. The deployment that caused the issue may have happened 12 hours before the symptom appeared (e.g., a schema change that took time to propagate through a slow consumer).

---

### Event Ordering vs. Correctness in Feature Computation

Out-of-order event processing is the rule in distributed systems, not the exception. Events may arrive out of order due to network conditions, client-side batching, or consumer rebalancing.

**Impact on feature correctness:**

```
Scenario: Compute "last item clicked by user"

Events in processing order (out-of-order due to partition skew):
  T=100ms: user clicks item A (event emitted at T=90ms, partition 0, late)
  T=80ms:  user clicks item B (event emitted at T=80ms, partition 1, on time)

Processing order at stream processor:
  1. Process T=80ms click on B → last_clicked = B
  2. Process T=100ms click on A → last_clicked = A  ← wrong! B is later by wall clock

Correct behavior: last_clicked = B (highest event_time, not processing order)
```

**Mitigation**: always use **event time** (the timestamp embedded in the event by the producer) for ordering semantics, never **processing time** (when the event was consumed). This requires watermark-based processing and tolerating some latency for late arrivals.

**The global ordering fallacy**: it is impossible to achieve global, consistent ordering of events across producers in a distributed system without centralized coordination (which creates a throughput bottleneck). Accept this and design features that are robust to small ordering violations:

- Use time windows instead of global sequencing.
- Prefer set-semantics (has user interacted with item?) over order-semantics (what did user interact with last?).
- For order-sensitive features (session reconstruction), use partition keys that ensure co-location.

---

### Cost Implications of High-Volume Event Streams

At scale, event infrastructure is a significant cost center. A system ingesting 1M events/sec at 1KB average size processes ~86 TB/day.

**Cost vectors:**

```
Storage (Kafka retention):
  - 1M events/sec × 1KB × 86400s × 7 days = ~600 TB/week
  - At $0.02/GB, tiered storage: ~$12,000/week just for one event type
  - Mitigation: aggressive compression (snappy/zstd: 3–5x), tiered storage,
    log compaction for state events

Compute (stream processing):
  - Proportional to event volume × processing complexity
  - Aggregation with state (windowed features) is more expensive than stateless transforms
  - Over-provisioning for burst handling wastes money between bursts
  - Mitigation: auto-scaling, spot instances for non-critical consumers

Network:
  - Cross-region replication is expensive
  - Every consumer group replicates the full topic traffic
  - Mitigation: filter events before cross-region replication; only ship aggregated signals

Feature store:
  - High-write-throughput key-value stores (Redis) have high cost per GB
  - A feature store with 100M users × 1KB features = 100GB in memory ≈ expensive
  - Mitigation: feature compression, eviction policies, tiered storage (hot/warm/cold)
```

**Staff-level cost reasoning**: cost discussions often feel like premature optimization, but at scale, an architectural decision today (raw events vs. pre-aggregated, 7-day vs. 30-day retention) can mean the difference between a $500K/year and $5M/year infrastructure bill. Cost must be a first-class architectural input, not a post-hoc optimization.

---

## 6. Infrastructure & Tooling

### Log-Based vs. Queue-Based Systems

The conceptual distinction was introduced in §2. Here, the operational implications:

**Kafka-style log-based systems dominate ML pipelines because:**

1. **Multiple independent consumers**: every consumer group gets its own cursor in the log. Adding a new consumer (new model, new feature, new analytics) requires zero changes to the producer or other consumers.

2. **Replay capability**: reset a consumer group's offset to any point in the past and reprocess from there. Essential for feature backfills, training data regeneration.

3. **Temporal decoupling at scale**: consumers can fall behind and catch up. A queue's message is lost when consumed; a log's message persists until expiration.

4. **Event sourcing compatibility**: the log is the event source of record.

5. **High throughput**: Kafka's sequential disk I/O model is extremely efficient. Modern Kafka clusters sustain millions of messages/second per broker.

**When queue-based systems are appropriate in ML contexts:**

- **Task dispatch**: "score these 10K candidates for this user in the next 500ms." This is a work item to be processed once by one worker, not a shared event stream.
- **Notification delivery**: "send this push notification to user U." Once delivered, the task is done.
- **Rate-limited external API calls**: work queue with backpressure to external services.

The systems are complementary. A common pattern: Kafka as the event backbone, SQS/RabbitMQ as a work queue for derived tasks triggered by events.

---

### Stream Processing vs. Batch Processing: Lambda and Kappa

**Lambda Architecture:**

```
Incoming data
    │
    ├─→ Speed layer (streaming): low latency, approximate
    │     Computes approximate features in near real-time
    │
    └─→ Batch layer: high latency, exact
          Recomputes exact features periodically

Serving layer merges speed and batch results:
  feature = batch_feature (exact, older) + speed_delta (approximate, recent)
```

**Lambda trade-offs:**
- Pro: Speed layer handles latency; batch layer handles correctness.
- Con: You maintain **two codebases** for the same logic (streaming + batch). They diverge over time. Training-serving skew emerges. Operational complexity doubles.

**Kappa Architecture:**

```
Incoming data
    │
    └─→ Single streaming layer handles everything
          "Batch" = streaming over a bounded historical window
          Replay from Kafka = batch reprocessing
```

**Kappa trade-offs:**
- Pro: Single codebase; streaming logic is used for both real-time and historical reprocessing.
- Con: Stream processing is harder to reason about for complex aggregations. Some batch patterns (large joins, global sorts) are inefficient in streaming.
- Con: Latency of historical replay depends on cluster size and data volume.

**Modern recommendation systems trend toward Kappa** because:
- Feature unification (same code offline and online) is more achievable.
- Stream processing frameworks (Flink) are mature enough for complex aggregations.
- Kafka tiered storage makes historical replay practical.
- Lambda's dual-codebase burden is consistently underestimated.

---

### Schema Management Systems

A schema registry is as important to event-driven systems as a type system is to code.

**What a schema registry does:**

```
Producer publishes event:
  1. Looks up schema ID for "user_interaction_v3"
  2. Serializes event using Avro/Protobuf with schema ID embedded in message header
  3. Publishes to Kafka

Consumer reads event:
  1. Extracts schema ID from message header
  2. Fetches schema from registry
  3. Deserializes using the schema
  4. If schema has evolved, applies compatibility rules
     (new optional field → consumer sees null for old events)
```

**Schema compatibility modes:**

```
BACKWARD: new schema can read old data.
  New consumer can read events produced with old schema.
  Required for: deploying new consumers that will replay historical events.
  Allowed: add optional fields, remove optional fields.

FORWARD: old schema can read new data.
  Old consumer can read events produced with new schema.
  Required for: rolling deploys where old consumers run alongside new producers.
  Allowed: add optional fields, add required fields with defaults.

FULL: both BACKWARD and FORWARD.
  Most restrictive. Safest for shared event schemas with many consumers.

NONE: no compatibility enforcement.
  Dangerous for ML systems; only for internal, single-consumer topics.
```

**Operational practice**: enforce FULL compatibility on production topics with multiple consumers. Maintain a schema changelog. Require schema changes to go through a PR review process, same as code changes.

---

## 7. Real-World System Design Examples

### 7.1 Real-Time Recommendation Pipeline

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────────┐
│                        User-Facing Layer                            │
│   Mobile/Web Client → API Gateway / BFF                             │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────────┐
              │ (1) Rec Request│                   │ (2) Events
              ▼               │                   ▼
  ┌───────────────────┐       │    ┌──────────────────────────┐
  │  Recommendation   │       │    │    Event Ingestion        │
  │  Serving (Online) │       │    │    (Kafka Producers)      │
  └─────────┬─────────┘       │    └────────────┬─────────────┘
            │                 │                 │
  (3) Read  │                 │    ┌────────────▼─────────────┐
  features  │                 │    │   Kafka Topic:           │
            │                 │    │   user_interactions      │
  ┌─────────▼─────────┐       │    └────────────┬─────────────┘
  │   Online Feature  │◄──────┘                 │
  │   Store (Redis)   │            ┌────────────▼─────────────┐
  └───────────────────┘            │   Stream Processor        │
                                   │   (Flink)                │
  ┌────────────────────┐           │   - Windowed aggregations│
  │  Candidate Index   │           │   - Session features     │
  │  (ANN: FAISS/HNSW) │           │   - Recency signals      │
  └─────────┬──────────┘           └────────────┬─────────────┘
            │                                   │
  ┌─────────▼──────────┐          ┌─────────────▼────────────┐
  │  Ranking Model     │          │  Online Feature Store     │
  │  (score candidates)│          │  Write Path               │
  └─────────┬──────────┘          └──────────────────────────┘
            │
  ┌─────────▼──────────┐
  │  Response          │ ← ranked recommendations
  └────────────────────┘

                ┌─────────────────────────────────┐
                │         Offline Path             │
                │                                 │
                │  Kafka → S3 (raw event archive) │
                │  ↓                              │
                │  Batch feature computation       │
                │  ↓                              │
                │  Offline feature store          │
                │  ↓                              │
                │  Training data pipeline         │
                │  (impression + outcome join)    │
                │  ↓                              │
                │  Model training                 │
                │  ↓                              │
                │  Offline evaluation             │
                │  ↓                              │
                │  Model deployment               │
                └─────────────────────────────────┘
```

**Key design decisions:**

- **Impression events** carry the full context: model version, feature snapshot hash, ranking scores, candidate pool. This enables offline debugging and training data generation.
- **Feature store** has two write paths: (a) streaming path for low-latency features (last-N clicks, session activity), (b) batch path for expensive-to-compute features (long-term preferences, collaborative filtering embeddings).
- **Candidate index** is updated asynchronously from item events (new item, item popularity update). The index may be slightly stale; this is acceptable.

**Failure modes:**

1. **Stream processor falls behind** (lag > 10 minutes): online features become stale. Recommendation quality degrades for engagement-sensitive signals. Mitigation: alert on lag, scale consumers.

2. **Feature store unavailability**: serving falls back to default features (global popularity, item quality only). Quality degrades but system stays up.

3. **Training-serving skew creeps in**: feature computation logic diverges between streaming and batch paths. Manifests as unexplained offline/online metric gap. Mitigation: unified feature platform, automated distribution comparison.

4. **Feedback loop amplification**: popular items accumulate engagement, get recommended more, accumulate more engagement. EDA makes this feedback loop very fast. Mitigation: exploration injection at serving time, diversity constraints.

---

### 7.2 Search Ranking System with Click Feedback

**Architecture:**

```
┌──────────────────────────────────────────────────────────────────┐
│                     Search Query Flow                            │
│                                                                  │
│  User Query → Query Parsing → Retrieval → Re-ranking → Results  │
└──────────────────────────────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────────┐
                    │ (1) Log query  │ (2) Log impressions │
                    ▼                ▼                     │
             ┌────────────┐  ┌────────────────┐           │
             │search_query│  │search_impression│           │
             │   topic    │  │    topic       │           │
             └─────┬──────┘  └───────┬────────┘           │
                   │                 │                     │
                   └────────┬────────┘         ┌───────────▼────────┐
                            │                  │  Click/Skip events │
                            ▼                  │  search_interaction│
             ┌─────────────────────────────────┴─topic──────────────┐
             │                  Stream Processor                     │
             │                                                       │
             │  Impression-Click Join (windowed, 30-min window):     │
             │    impression(query, doc_id, position, ts)            │
             │  + click(query, doc_id, ts) or timeout(no click)      │
             │  → labeled_example(query, doc, features, clicked=0/1) │
             └──────────────────────┬────────────────────────────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                │
                    ▼               ▼                ▼
          ┌──────────────┐  ┌────────────┐  ┌──────────────┐
          │  Training    │  │  Query     │  │  Real-time   │
          │  Data Store  │  │  Feature   │  │  Feedback    │
          │  (S3/DWH)    │  │  Cache     │  │  for online  │
          └──────┬───────┘  └────────────┘  │  learning    │
                 │                          └──────────────┘
          ┌──────▼───────┐
          │  Batch model │
          │  training    │
          └──────┬───────┘
                 │
          ┌──────▼───────┐
          │  New ranking │
          │  model       │
          └──────────────┘
```

**The impression-click join** is the heart of search ranking data generation. Critical challenges:

1. **Position bias**: clicks are position-dependent. An impression at position 1 has 5x the click rate of position 5 for the same quality document. Your training data must account for position bias (IPS weighting, position-aware models, or inverse propensity scoring).

2. **Click assignment ambiguity**: a user queries "python tutorial," sees 10 results, clicks result 3, then clicks result 7. Did they find what they wanted at 3, or did 3 satisfy but 7 was also relevant? This is the "last click" problem.

3. **Abandonment signals**: no click after an impression is a negative signal — but only if the query session truly ended. A user who clicked a result but didn't click another may be satisfied with one result.

4. **Query reformulation**: user queries "python tutorial," doesn't click, reformulates to "python tutorial beginners," then clicks. The first query had a hidden negative signal that's only visible from the session context.

EDA enables capturing the full session context by streaming all query and interaction events for a session, allowing downstream analysis to reconstruct the implicit signals.

---

### 7.3 Feature Store Architecture

**A feature store is the materialized view layer of your EDA system.**

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Feature Store Architecture                    │
│                                                                     │
│  ┌─────────────┐     ┌────────────────────────────────────────┐    │
│  │  Event Log  │     │              Write Paths               │    │
│  │  (Kafka)    │────►│                                        │    │
│  └─────────────┘     │  Streaming path (Flink):               │    │
│                      │    user_interaction → user_click_1h    │    │
│  ┌─────────────┐     │    user_interaction → user_session_ctx │    │
│  │  Data       │     │                                        │    │
│  │  Warehouse  │────►│  Batch path (Spark):                   │    │
│  │  (Hive/BQ)  │     │    user_history_90d → user_taste_vec   │    │
│  └─────────────┘     │    item_stats → item_quality_score     │    │
│                      └──────────────────┬─────────────────────┘    │
│                                         │                           │
│                          ┌──────────────┼──────────────────┐       │
│                          ▼              ▼                   ▼       │
│                   ┌──────────┐   ┌──────────┐   ┌──────────────┐  │
│                   │  Online  │   │  Offline │   │  Nearline    │  │
│                   │  Store   │   │  Store   │   │  Store       │  │
│                   │  (Redis) │   │  (S3/DW) │   │  (Cassandra) │  │
│                   │  p99<5ms │   │ minutes  │   │  p99<20ms    │  │
│                   └─────┬────┘   └─────┬────┘   └──────┬───────┘  │
│                         │             │                │           │
│                    ┌────▼─────────────▼────────────────▼──────┐   │
│                    │              Read Paths                   │   │
│                    │                                           │   │
│                    │  Online inference: reads Online Store     │   │
│                    │  Training data: reads Offline Store       │   │
│                    │  Batch scoring: reads Nearline Store      │   │
│                    └───────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

**Three-tier feature store:**

| Tier | Latency | Storage | Feature types | Consistency |
|---|---|---|---|---|
| **Online** (Redis) | <5ms | Expensive (RAM) | Recent activity, session | Updated in seconds |
| **Nearline** (Cassandra) | <20ms | Moderate | Daily/weekly aggregates | Updated in minutes |
| **Offline** (S3/DW) | Minutes | Cheap | Historical, expensive features | Updated in hours |

**The consistency guarantee challenge:**

Training uses the offline store. Serving uses the online store. If they computed the same feature differently, you have training-serving skew. Mitigation: the offline store is computed from the same raw events as the online store, using the same (or equivalent) logic. Periodic distribution checks validate parity.

**Point-in-time correctness:**

Training data must use features as they existed at the time of the prediction, not as they exist today. This requires time-travel queries against the offline store:

```
For training example (user U, item I, event at T):
  features = offline_store.get(user_id=U, as_of=T)
  NOT
  features = offline_store.get(user_id=U)  ← uses current features, leaks future info
```

This is a subtle but critical correctness requirement. Without it, your model learns from features that include future information (e.g., a user's total purchases including purchases made after the training event), leading to optimistic offline metrics and degraded online performance.

---

### 7.4 Notification System with Event-Triggered Personalization

**Architecture:**

```
Event sources:
  - user_interaction (clicks, saves, purchases)
  - item_events (price drop, back-in-stock, trending)
  - social_events (someone liked your review, friend joined)

┌────────────────────────────────────────────────────────────────┐
│                  Notification Trigger Engine                   │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Rule Engine (event-driven):                            │  │
│  │    IF item_event(type=price_drop, item=I)               │  │
│  │    AND user_feature(saved_item=I)                       │  │
│  │    AND user_feature(notification_fatigue_score < 0.7)   │  │
│  │    AND user_feature(last_notified > 24h ago)            │  │
│  │    THEN trigger notification candidate                  │  │
│  └─────────────────────────────────────────────────────────┘  │
│                         │                                      │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │  Personalization Layer:                                 │  │
│  │    - Score notification value for this user             │  │
│  │    - Predict engagement probability                     │  │
│  │    - Apply frequency capping                            │  │
│  │    - Select optimal send time (time-of-day model)       │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                         │                                      │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │  Deduplication & Suppression:                           │  │
│  │    - Has user already received a notification for I     │  │
│  │      in the last N hours?                               │  │
│  │    - Is user in a do-not-disturb window?                │  │
│  │    - Has user unsubscribed from this notification type? │  │
│  └──────────────────────┬──────────────────────────────────┘  │
└───────────────────────  │ ─────────────────────────────────────┘
                          ▼
               ┌────────────────────┐
               │  Notification Queue│ (time-delayed if optimal send time ≠ now)
               └────────┬───────────┘
                        ▼
               ┌────────────────────┐
               │  Delivery Service  │ → Push, Email, SMS
               └────────┬───────────┘
                        │
               ┌────────▼───────────┐
               │  Delivery Event    │ → notification_delivered topic
               └────────┬───────────┘
                        │
               ┌────────▼───────────┐
               │  Engagement Event  │ → notification_clicked / notification_dismissed
               └────────────────────┘ → feeds back into notification model training
```

**Key design choices and trade-offs:**

1. **Fanout at trigger vs. at send**: when a popular item drops in price, millions of users may have saved it. Triggering a notification evaluation for all of them simultaneously creates a massive fanout. Options:
   - **Eager fanout** (evaluate immediately for all users): high compute burst, good for time-sensitive events.
   - **Lazy fanout** (queue evaluation, drain slowly): smoothed load, may miss the window of urgency.
   - **Hybrid**: tier users by notification likelihood score, process high-affinity users eagerly.

2. **Notification fatigue as a feature**: users who receive too many notifications disengage. The notification system must model fatigue as a real-time feature (how many notifications has this user received in the last 24h, what was their engagement rate?). This feature must be updated by the notification delivery events — a feedback loop within the notification system itself.

3. **Idempotency in delivery**: a notification delivery system must be idempotent — "send this notification once and only once." EDA makes at-least-once delivery easy; exactly-once requires a distributed dedup store. Use notification_id as a dedup key with a 48h TTL.

---

## 8. Python-Oriented Examples

### Minimal Event Streaming (Producer/Consumer)

```python
# Language-agnostic pseudocode with Python flavor
# Demonstrates core concepts without framework specifics

# --- Producer ---
class EventProducer:
    def __init__(self, broker_config, topic: str, schema: Schema):
        self.topic = topic
        self.schema = schema
        # Idempotent producer: broker will deduplicate retries
        self.client = KafkaProducer(
            broker_config,
            enable_idempotence=True,
            acks="all",           # Wait for all replicas to acknowledge
            compression="snappy", # ~40% size reduction
        )

    def emit(self, event: dict) -> None:
        # Assign idempotency key
        event["event_id"] = event.get("event_id") or generate_uuid()
        event["emitted_at_ms"] = current_time_ms()

        # Validate against schema before emit (fail fast)
        self.schema.validate(event)

        self.client.produce(
            topic=self.topic,
            key=event["user_id"],    # Partition by user_id for ordering
            value=serialize_avro(event, self.schema),
            on_delivery=self._delivery_callback,
        )

    def _delivery_callback(self, error, message):
        if error:
            # Log and alert; do not swallow delivery failures silently
            metrics.increment("event.delivery.failure", tags={"topic": self.topic})
            raise DeliveryException(error)
        metrics.increment("event.delivery.success")
```

---

### Async Event Processing with `asyncio`

```python
import asyncio
from collections import defaultdict

class AsyncEventConsumer:
    """
    Demonstrates async consumption pattern for I/O-bound consumers
    (e.g., writing to a feature store that has an async client).

    Key principle: async is valuable when the consumer is I/O-bound
    (waiting on network, disk). CPU-bound processing (model inference,
    heavy aggregation) does not benefit from async alone — use
    multiprocessing or distributed compute.
    """

    def __init__(self, topic: str, group_id: str, feature_store):
        self.topic = topic
        self.group_id = group_id
        self.feature_store = feature_store  # Async client
        self._dedup_cache: dict[str, float] = {}  # event_id → timestamp

    async def process(self, event: dict) -> None:
        # Step 1: Dedup check (in-memory for this example; use Redis at scale)
        if self._is_duplicate(event):
            metrics.increment("event.duplicate.dropped")
            return

        # Step 2: Feature computation (pure function of event)
        features = self._compute_features(event)

        # Step 3: Write to feature store (async I/O)
        await self.feature_store.update(
            key=f"user:{event['user_id']}",
            features=features,
        )

        # Step 4: Mark as processed (commit offset externally)
        self._mark_dedup(event)

    def _is_duplicate(self, event: dict) -> bool:
        event_id = event["event_id"]
        # Expire entries older than 24h to bound memory
        self._dedup_cache = {
            k: v for k, v in self._dedup_cache.items()
            if current_time_ms() - v < 86_400_000
        }
        return event_id in self._dedup_cache

    def _mark_dedup(self, event: dict) -> None:
        self._dedup_cache[event["event_id"]] = event["emitted_at_ms"]

    def _compute_features(self, event: dict) -> dict:
        # Stateless feature extraction from a single event
        return {
            "last_clicked_item": event.get("item_id"),
            "last_event_ts": event["emitted_at_ms"],
            "last_event_type": event["event_type"],
        }

    async def run(self, consumer_client) -> None:
        async for message in consumer_client.subscribe(self.topic, self.group_id):
            try:
                event = deserialize_avro(message.value)
                await self.process(event)
                await consumer_client.commit(message)  # Commit after successful processing
            except SchemaValidationError as e:
                # Schema errors are usually permanent; log and skip
                logger.error("schema_validation_failed", event_id=event.get("event_id"))
                await consumer_client.commit(message)  # Don't block on bad events
            except FeatureStoreUnavailable:
                # Transient errors: do NOT commit; retry by not advancing offset
                logger.warning("feature_store_unavailable; will retry")
                await asyncio.sleep(1)
```

---

### Idempotent Consumer Logic for Duplicate User Events

```python
class IdempotentClickCounter:
    """
    Counts unique clicks per (user, item) pair using idempotent writes.

    The naive approach: increment a counter on every click event.
    Problem: if the event is delivered twice (at-least-once), the counter
    is incremented twice.

    Idempotent approach: track which event_ids have been processed;
    only increment if not seen before. The counter reflects the true
    count of distinct click events, not deliveries.
    """

    def __init__(self, feature_store, dedup_store):
        self.feature_store = feature_store  # e.g., Redis
        self.dedup_store = dedup_store       # e.g., Redis SET with TTL

    def process_click(self, event: dict) -> None:
        event_id = event["event_id"]
        user_id = event["user_id"]
        item_id = event["item_id"]

        # Atomic check-and-set: returns True if event_id was newly added,
        # False if it already existed.
        is_new = self.dedup_store.set_if_absent(
            key=f"dedup:{event_id}",
            value=1,
            ttl_seconds=86400  # 24h dedup window
        )

        if not is_new:
            return  # Duplicate; skip

        # Idempotent increment: safe because we've guaranteed is_new above
        self.feature_store.increment(
            key=f"user:{user_id}:item:{item_id}:click_count",
            delta=1,
        )
        self.feature_store.increment(
            key=f"user:{user_id}:total_clicks",
            delta=1,
        )

    def get_click_count(self, user_id: str, item_id: str) -> int:
        return self.feature_store.get(
            f"user:{user_id}:item:{item_id}:click_count"
        ) or 0
```

---

## 9. Mental Models & Intuition

### Events as the Source of Truth vs. Derived State

The most important mental shift in event-driven thinking:

**Traditional model**: the database row is truth. Events (if they exist) are side effects.

**Event-driven model**: the event log is truth. The database row is a materialized cache of the log.

This inversion has profound implications:

```
Traditional: user_profile table
  → "User U has click_count = 47"
  → If the table is corrupted, the truth is lost

Event-driven: event log
  → "User U has emitted 47 click events"
  → click_count = 47 is a derived fact, computed on demand
  → If the derived store is corrupted, recompute from the log
  → If the computation logic was wrong, recompute with new logic
  → If you need a new fact, compute it from the existing log
```

**The log as the source of truth enables time-travel**: given the event log, you can reconstruct any state at any point in time. This is not possible with snapshot-based storage (without explicit audit tables).

For recommendation systems, this means:
- **Training data correctness**: always derive training features from the event log using point-in-time joins, not from a current snapshot.
- **Feature backfills**: when you add a new feature, backfill it from the log.
- **Debugging**: when a model behaves unexpectedly, replay its input from the log.

---

### Time, Causality, and Ordering in User Behavior Streams

Human behavior in time is not perfectly ordered in distributed event streams. Understanding this is critical for designing correct feature pipelines.

**Three time concepts:**

```
Event time: when the event actually happened (set by the client/producer)
Ingestion time: when the event arrived at the broker
Processing time: when the event was processed by the consumer

Event time is the "true" time and should drive all temporal logic.

Example:
  - User clicks on mobile at T=10:00:00 (event time)
  - Mobile is offline; event queued locally
  - Mobile reconnects at T=10:30:00 (ingestion time)
  - Stream processor processes it at T=10:31:00 (processing time)

A consumer using processing time would assign this event to the 10:30-10:31 window.
A consumer using event time assigns it to the 10:00 window — correct.
```

**Causality vs. correlation in user streams:**

A click on item B after viewing item A does not necessarily mean A caused the click on B. The user may have already intended to click B. But in recommendation feedback loops, this correlation is treated as causality: "users who viewed A clicked B → recommend B to viewers of A." This is the survivorship bias / popularity bias problem, and it is amplified by EDA's fast feedback loops.

Staff-level thinking: always ask whether your signal is capturing causal intent or correlated behavior. This is the difference between a model that learns genuine preferences and one that amplifies popularity bias.

---

### Designing Systems Around Data Flow, Not Request Flow

The request-response mental model is: "user asks for X, system computes and returns X." The data flow mental model is: "user generates event; event propagates through pipeline; state is pre-computed; when user asks, answer is ready."

**The key insight**: in a recommendation system, the "interesting computation" happens asynchronously, driven by events. By the time a user requests recommendations, most of the work is already done:

```
Request-response mental model:
  User requests recs → system computes everything → returns recs
  Latency: sum of all computation

Data flow mental model:
  User generates events → async feature computation → async candidate pre-computation
  User requests recs → read pre-computed state → light ranking → return recs
  Latency: just the read + ranking step (10–50ms)
```

The data flow mental model demands that you **separate the hot path from the computation path**. The serving layer should be thin and fast; the heavy lifting happens upstream in event-driven pipelines.

This mental model also changes how you design systems under load:
- In request-response: load spikes hit your serving layer directly.
- In data flow: load spikes hit the event pipeline (which buffers via Kafka lag) and the serving layer reads from pre-computed state (which is insulated from the spike).

---

### Thinking in Terms of Pipelines and Transformations

Every feature in a recommendation system can be expressed as a transformation pipeline over an event stream:

```
click_count_1h(user) = 
  COUNT(
    events WHERE event_type='click' AND user_id=user
    WINDOW SLIDING(1 HOUR)
  )

diversity_score(user, item) = 
  1 - JACCARD(
    recent_categories(user, last_10_clicks),
    item_categories(item)
  )

trending_score(item) =
  ROLLING_MEAN(
    events WHERE item_id=item AND event_type='click',
    WINDOW TUMBLING(1 HOUR)
  ) / GLOBAL_MEAN(clicks_per_item)
```

When you think this way, the architecture falls out naturally:
- Each feature is a streaming job with well-defined inputs (event types) and outputs (feature values).
- Features are composable: complex features are pipelines of simpler transforms.
- Testing is straightforward: apply the transform to a known sequence of events, assert the output.
- Replaying features means replaying the event sequence through the same transform.

---

## 10. Advanced Topics

### Exactly-Once Semantics vs. Practical Guarantees

The three delivery guarantees in distributed messaging:

- **At-most-once**: events may be dropped; never duplicated. Easy to implement (no retries), unacceptable for most ML use cases.
- **At-least-once**: events may be duplicated; never dropped. The default in Kafka. Requires idempotent consumers.
- **Exactly-once**: each event processed exactly once. Kafka supports this via transactions, but with significant overhead and limitations.

**Exactly-once in Kafka (since 0.11):**

```
Mechanism:
  1. Idempotent producer: broker assigns sequence number per producer.
     Duplicate messages (retries) are detected and dropped at the broker.

  2. Transactional producer: produces across multiple topics atomically.
     Either all messages commit, or none do.

  3. Read-process-write pattern:
     consumer.poll() → process → producer.send() → consumer.commit()
     All three operations are wrapped in a transaction.
     If the consumer crashes before commit, the transaction aborts.
     On restart, it reprocesses the event — but the producer won't
     re-emit because the transaction was aborted.

Limitations:
  - Only guarantees exactly-once within Kafka (read from Kafka → write to Kafka).
  - Does NOT guarantee exactly-once if the write target is an external system
    (Redis, database). Exactly-once with external sinks requires the external
    system to support idempotent writes (usually via upsert with a unique key).
  - Throughput overhead: ~20–40% reduction due to transaction coordination.
```

**Practical guidance for ML systems:**

For most features in recommendation systems, "at-least-once with idempotent consumers" achieves the same practical outcome as exactly-once, at lower cost and complexity. Reserve true exactly-once semantics for financial or compliance-critical pipelines where duplicates have irreversible consequences.

---

### Event Replay for Backfills and Model Retraining

Replay is EDA's superpower for ML teams. Understanding its mechanics and limits is essential.

**Replay mechanics:**

```
Consumer group offset management:
  - Each consumer group maintains an offset per partition
  - Kafka stores offsets in the __consumer_offsets internal topic
  - To replay: reset consumer group offsets to an earlier position

Two replay strategies:
  Option A: Reset existing consumer group
    Pro: Simple; same consumer group; no duplicate feature writes
    Con: Existing consumers stop processing new events while replaying old ones
    Use when: you can tolerate feature freshness degradation during replay

  Option B: Create a new consumer group
    Pro: Existing pipeline continues; replay runs in parallel
    Con: New features written to different namespace; cutover needed at end
    Use when: you cannot tolerate disruption to the production pipeline
```

**Replay for model retraining:**

When retraining a model on historical data:

```
Goal: Generate training examples (user, item, features, label) for the past 90 days

Event replay approach:
  1. Replay user_interaction events for T-90d to T
  2. For each impression event at time T_i:
     a. Fetch user features as_of T_i (point-in-time from offline store)
     b. Fetch item features as_of T_i
     c. Join with outcome event (click/no-click within T_i to T_i+30min)
  3. Emit (features, label) to training data store

Critical: the features must reflect the state at T_i, not the current state.
Without point-in-time correctness, your model sees features that include
future information → optimistic offline metrics → production degradation.
```

**Replay for backfills:**

```
Scenario: You add a new feature, "user_purchase_velocity_7d."
You need this feature for all users for training and initial serving.

Replay approach:
  1. Create new consumer group "feature_purchase_velocity_backfill"
  2. Set offset to T-90d (cover the full training window)
  3. Run consumer: processes purchase events, computes velocity, writes to feature store
  4. Consumer catches up to present → features are fully populated
  5. Hand off to production streaming consumer for ongoing updates
  6. Delete backfill consumer group
```

---

### Handling Late-Arriving and Out-of-Order Events

Late arrivals are events that arrive at the processor after their event-time window has already been emitted. Sources:
- Mobile apps with intermittent connectivity.
- CDN or proxy buffering.
- Clock skew between producers.
- Slow networks for specific user segments.

**Handling strategies:**

```
Strategy 1: Drop late events (simplest)
  - Set watermark = max(event_time_seen) - allowed_lateness
  - Events with event_time < watermark are dropped
  - Acceptable for high-volume, low-value signals (scroll depth, hover)
  - Not acceptable for critical signals (purchase, explicit feedback)

Strategy 2: Update already-emitted windows (complex)
  - Flink "allowed lateness" window: window remains open for T after watermark passes
  - Late events update the window result; downstream receives correction event
  - Requires downstream to handle "update" semantics (not just append)
  - Feature store must support conditional updates (update only if newer)

Strategy 3: Side output for late events
  - Late events go to a separate "late arrivals" topic
  - Batch job periodically processes late arrivals and reconciles with features
  - Simple to implement; introduces some latency but maintains correctness
  - Best practice for most ML systems
```

**Clock skew considerations:**

In distributed systems, clocks drift. A producer's event_time may be slightly in the future (clock ahead) or in the past (clock behind). Best practices:
- Server-side ingestion timestamp as a fallback when client-side event_time is anomalous.
- Reject events with event_time > server_time + 60s (likely clock error).
- Reject events with event_time < server_time - MAX_ALLOWED_LATENESS (too old to be useful).

---

### Data Lineage and Reproducibility in ML Systems

Data lineage is the ability to trace any model prediction back to the raw events that influenced it. This is required for:
- **Debugging** model misbehavior.
- **Compliance** and right-to-explanation requirements.
- **Reproducibility** of offline experiments.
- **Detecting** data quality issues after the fact.

**Lineage graph for a recommendation:**

```
Recommendation R served to user U at time T
  → Model version M_v42
    → Training data batch D_2026_Q1
      → Feature set F_v7 computed from events E_set in [T-90d, T-30d]
      → Labels L computed from impression-click join at T-30d
  → Features used at inference time:
    → user_click_1h: computed from click events in [T-1h, T]
    → user_taste_vec: computed from purchase events in [T-90d, T]
    → item_quality: computed from rating events in [T-180d, T]
```

With this lineage, you can answer:
- "Did user U's recommendation change because a click event was dropped?" → check user_click_1h for U at T.
- "Was model M_v42 trained with correct labels?" → re-run the label join from events and compare to D_2026_Q1.
- "Is this feature value anomalous?" → trace it to its source events.

**Implementing lineage:**

- Tag all events with a `trace_id` that propagates through the pipeline.
- Log feature computation metadata (which events were used, which logic version).
- Store model inference metadata (which feature version, which model version, which candidate pool).
- Use a lineage store (separate from the feature store) to record these relationships.

---

## 11. Staff-Level Decision Frameworks

### When to Adopt EDA in ML Systems vs. Simpler Architectures

Use this decision framework when evaluating EDA adoption:

```
Scoring framework (higher = more compelling for EDA):

1. Fan-out factor
   How many consumers need each event?
   1-2 consumers: 0 pts  |  3-5: 1pt  |  5+: 2pts

2. Latency tolerance of consumers
   All consumers need sync response: 0 pts
   Mix of sync/async: 1pt
   All consumers can be async: 2pts

3. Need for replay / historical processing
   No: 0pts  |  Possible future need: 1pt  |  Required today: 2pts

4. Team/service decoupling requirement
   Single team, single service: 0pts
   Multiple teams: 1pt
   Teams need independent deployment: 2pts

5. Volume characteristics
   Low volume (<10K events/min): 0pts
   Medium (10K–1M/min): 1pt
   High (>1M/min): 2pts

6. Data quality / auditability requirement
   Best-effort: 0pts  |  Auditable: 1pt  |  Regulatory: 2pts

Score interpretation:
  0-4:  Stick with simpler architecture (REST APIs, direct DB writes)
  5-7:  EDA beneficial for specific components; don't boil the ocean
  8-12: EDA is the right architectural choice
```

**Concrete example: should a small recommendation team adopt Kafka?**

- Fan-out: 4 consumers (features, training data, analytics, abuse detection) → 1pt
- Latency: all async except feature pipeline → 2pts
- Replay: needed for feature backfills (known requirement) → 2pts
- Decoupling: 2 teams share events → 1pt
- Volume: 500K events/min → 1pt
- Audit: wanted for debugging → 1pt
- **Total: 8pts → EDA recommended**

---

### Reasoning About Trade-offs

**Latency vs. Correctness:**

```
Trade-off axis:
  Lower latency → accept eventual consistency → risk stale features → risk suboptimal recommendations
  Higher correctness → wait for consistent state → higher latency → worse user experience

Resolution:
  1. Segment features by latency sensitivity:
     - "Did user click an item in the last 60 seconds?" → extremely latency-sensitive
     - "What are user's long-term genre preferences?" → correctness-sensitive, latency-tolerant
  2. Assign different pipeline latency SLOs per feature type.
  3. Never compromise on correctness for training labels (offline is not latency-sensitive).
  4. Accept short-term staleness in serving if it enables lower system complexity.
```

**Cost vs. Freshness:**

```
Freshness is expensive because:
  - High-frequency updates → higher compute (stream processor) cost
  - Low-latency storage (Redis) → higher storage cost per GB
  - Cross-region sync → network cost

Marginal value of freshness decreases:
  - 10-second-old features are almost as good as 1-second for most features
  - 5-minute-old trending signals are acceptable for most recommendations
  - 24-hour-old long-term preference features are fine

Framework: estimate the marginal improvement in click-through rate (CTR)
per unit of feature freshness. If going from 10-minute to 1-minute
freshness costs $50K/month in infrastructure but improves CTR by 0.1%
(worth ~$10K/month in revenue), the trade-off is clear.
```

---

### Designing Team Boundaries Around Event Streams

Event streams are excellent team interface boundaries because:
- They make data contracts explicit (schema).
- Producers and consumers can deploy independently.
- Teams can add new consumers without involving the producer team.

**Event-driven team organization for a recommendation system:**

```
Team: User Interaction Platform
  Owns: user_interactions, user_sessions topics
  Responsibility: reliable, schema-compliant event ingestion

Team: Feature Platform
  Consumes: user_interactions, item_events
  Owns: Online feature store, offline feature store
  Produces: feature_materialized events (for downstream lineage)

Team: Ranking & Relevance
  Consumes: Online/offline feature stores
  Owns: Ranking models, candidate index
  Produces: recommendation_served, recommendation_requested topics

Team: Training & Experimentation
  Consumes: recommendation_served, user_interactions (for labels)
  Owns: Training pipelines, experiment platform
  Produces: model_deployed events
```

**The team contract is the event schema.** Schema changes require cross-team coordination, just as API changes do. Make this explicit in your engineering process: schema changes require a PR against the schema registry, reviewed by consuming teams.

---

### Migration Strategies

**From batch pipelines to streaming:**

```
Phase 0 (baseline): full batch system
  - Daily feature refresh job: reads last N days of raw events from DWH
  - Slow, but simple to reason about

Phase 1: introduce event streaming without removing batch
  - Stand up Kafka; have producers write to Kafka AND DWH (dual-write)
  - Build streaming feature pipeline consuming Kafka
  - Run streaming and batch in parallel; validate agreement
  - Streaming features shadow batch features: serving still uses batch

Phase 2: cut over high-sensitivity features to streaming
  - Features where freshness matters (session context, recent clicks)
    switch to streaming-only
  - Features where freshness doesn't matter (long-term preferences) stay batch

Phase 3: streaming as primary, batch as backstop
  - All features primary from streaming
  - Batch still runs as a daily reconciliation (correctness backstop)
  - On streaming failure, fall back to batch features

Phase 4: Kappa architecture
  - Batch is eliminated; "batch" = replay streaming job over bounded window
  - Single codebase for all feature computation
```

**The key principle of migration**: never do a hard cutover. Always run parallel systems, validate agreement, then incrementally shift traffic. The cost of running two systems temporarily is lower than the cost of a failed cutover.

**From monolith to event-driven microservices:**

```
The Strangler Fig pattern:
  1. Identify the first bounded context to extract (e.g., user interaction tracking).
  2. New service emits events; monolith still handles the same function.
  3. Point some consumers to the new event stream.
  4. Validate equivalence.
  5. Migrate all consumers to new event stream.
  6. Remove the monolith code path.
  7. Repeat for next bounded context.

Anti-pattern: "big bang" migration to EDA. This always takes 3x longer
than estimated and often fails. Incremental strangler fig is the only
reliable approach.
```

---

## 12. Learning Roadmap

### Key Concepts to Master for Real-Time ML Systems

**Foundational (if not already solid):**
- [ ] Log-structured storage and why sequential I/O is fast
- [ ] Distributed consensus basics (Paxos/Raft) — not for implementation, but to understand why exactly-once is hard
- [ ] Hash ring / consistent hashing — how partitioning works at scale
- [ ] Time series data structures (ring buffers, sliding window algorithms)

**Event streaming core:**
- [ ] Kafka internals: producer batching, segment log format, consumer group rebalancing, compaction
- [ ] Exactly-once semantics: idempotent producers, transactions, their limits
- [ ] Stream processing windows: tumbling, sliding, session, global — and when each is appropriate
- [ ] Watermarks and late data handling in stateful stream processing
- [ ] State backends in stream processors (in-memory, RocksDB, remote) and their trade-offs

**ML-specific:**
- [ ] Point-in-time correct feature generation (train/serve parity)
- [ ] Online vs. offline feature stores: consistency, latency, and operational model
- [ ] Feedback loop dynamics in recommendation systems
- [ ] Label construction from event streams (delayed outcomes, position bias)
- [ ] Feature monitoring: distribution drift detection, staleness SLOs

**Distributed systems:**
- [ ] CAP theorem and its practical implications (CP vs AP systems for features)
- [ ] Two-phase commit and why it's problematic; Saga pattern
- [ ] CRDT (Conflict-free Replicated Data Types) — useful for distributed counters

---

### Suggested Hands-On Projects

**Project 1: Mini Real-Time Recommender**

Build a system that:
1. Simulates a stream of user interaction events (clicks, purchases).
2. Computes streaming features (click count per user per item, last 1h activity).
3. Writes features to a local Redis instance.
4. Serves simple recommendations based on features ("similar users also bought").
5. Logs recommendation impressions back as events.
6. Closes the loop: recommendation impressions are consumed by the training data pipeline.

Focus areas: partitioning strategy, consumer group management, idempotency, feature freshness measurement.

**Project 2: Feature Pipeline with Replay Capability**

Build a system that:
1. Generates a stream of "raw events" (a simple click stream).
2. Computes a feature (rolling 10-minute click count per user) from the stream.
3. Introduces a bug in the feature computation logic (deliberate).
4. Detects the bug via a feature distribution anomaly.
5. Fixes the bug and replays the event stream to recompute correct features.
6. Validates that the replayed features match the correct expected values.

Focus areas: offset management, replay mechanics, feature validation.

**Project 3: Schema Evolution Safety**

Build a system that:
1. Starts with schema v1 for a user interaction event.
2. Deploys consumer v1 that reads v1 events.
3. Evolves to schema v2 (add optional fields).
4. Validates that consumer v1 still works with v2 events (backward compatibility).
5. Evolves to schema v3 (rename a field — breaking change).
6. Observes the failure mode of consumer v1 reading v3 events.
7. Implements a schema registry to prevent v3 from being deployed without consumer migration.

Focus areas: Avro/Protobuf schema evolution, schema registry integration, compatibility modes.

---

## Appendix: Quick Reference

### Event Design Checklist

- [ ] Does every event have a globally unique `event_id`?
- [ ] Does every event have an `event_time` set by the producer (not ingestion time)?
- [ ] Is the event schema registered and versioned?
- [ ] Is the partition key chosen to ensure ordering for consumers that need it?
- [ ] Are required fields validated at ingestion?
- [ ] Is the event payload self-contained (event-carried state transfer) or does it require follow-up reads?
- [ ] Is the event semantically versioned (not just structurally)?

### Stream Processing Health Checklist

- [ ] Is consumer lag monitored with alerting thresholds?
- [ ] Are event volumes per topic monitored (sudden drops alert)?
- [ ] Is feature store freshness (staleness distribution) measured?
- [ ] Is there a runbook for lag growth?
- [ ] Is replay tested periodically (not just when there's an incident)?
- [ ] Are schema validation failure rates at 0?

### Training Data Quality Checklist

- [ ] Are training features computed with point-in-time correctness?
- [ ] Are duplicate training examples deduplicated before training?
- [ ] Is position bias accounted for in click labels?
- [ ] Are training data statistics (volume, label distribution, feature distributions) validated before training runs?
- [ ] Is there a test for online/offline feature parity?

---

*This guide is intended as a living reference. The field evolves; the mental models are stable.*
