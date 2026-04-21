# Posh – Data Architecture Interview Prep
### Interview: Cat & Ethan — "Walk us through what you see, your thoughts, and how you'd build the pipelines"

> Dense, concrete, AWS-specific. Built for the Posh Senior Software Engineer, Personalization role. Updated with confirmed intel from prior interview rounds. No fluff — real trade-offs, real failure modes, real names.

---

## What You Know Going In (Confirmed Intel)

This is not speculation. You learned this in prior rounds. Walk into tomorrow knowing:

| Fact | Implication |
|---|---|
| Personalization is **heuristics-based today** — not ML | You're not improving a model. You're building the foundation that makes ML viable. |
| **Redshift** is being queried directly for personalization | Wrong tool for serving. This is the most urgent architectural problem. |
| **Rudderstack** is their event tracking layer | Good news: adding Personalize/Kinesis is mostly a routing configuration, not new infra. |
| They only track **clicks and RSVPs** — not impressions | The fundamental measurement gap. You can't compute CTR without impression data. |
| **MongoDB** is the application DB, search runs on top of it | Two problems: operational DB + search competing for resources. Won't scale. |
| They had a **production outage during a large event** | Almost certainly Redshift concurrency collapse under traffic spike. |
| They are **open to AWS Personalize and SageMaker** | They want you to be the person who builds this transition, not just evaluate it. |
| No confirmed event streaming primitive (Kinesis/Kafka) | First infrastructure gap to close before ML becomes viable. |

---

## Table of Contents

1. [What This Interview Actually Tests](#1-what-this-interview-actually-tests)
2. [How to Read Their Architecture When They Show It To You](#2-how-to-read-their-architecture-when-they-show-it-to-you)
3. [Posh's Confirmed Current Stack](#3-poshs-confirmed-current-stack)
4. [The Two Adjacent Problems: Redshift Limits + MongoDB Search Scaling](#4-the-two-adjacent-problems-redshift-limits-mongodb-search-scaling)
5. [The AWS Personalize Mental Model — Know It Cold](#5-the-aws-personalize-mental-model-know-it-cold)
6. [The AWS SageMaker Mental Model — Know It Cold](#6-the-aws-sagemaker-mental-model-know-it-cold)
7. [Data Pipeline Patterns — Batch vs. Streaming vs. Lambda](#7-data-pipeline-patterns-batch-vs-streaming-vs-lambda)
8. [Feature Stores — The Bridge Between Pipelines and Models](#8-feature-stores-the-bridge-between-pipelines-and-models)
9. [Productionalizing ML Models — The Hard Part They Need Help With](#9-productionalizing-ml-models-the-hard-part-they-need-help-with)
10. [Event Tracking Architecture — Rudderstack as the Foundation](#10-event-tracking-architecture-rudderstack-as-the-foundation)
11. [Model Serving Infrastructure](#11-model-serving-infrastructure)
12. [Data Quality, Observability, and Feedback Loops](#12-data-quality-observability-and-feedback-loops)
13. [A/B Testing Infrastructure for Personalization](#13-ab-testing-infrastructure-for-personalization)
14. [The Full Production Blueprint — How You'd Build It](#14-the-full-production-blueprint-how-youd-build-it)
15. [Questions to Ask Cat and Ethan During the Walkthrough](#15-questions-to-ask-cat-and-ethan-during-the-walkthrough)
16. [How to Present Your Observations Without Being Arrogant](#16-how-to-present-your-observations-without-being-arrogant)
17. [Interview Drill — Scenarios They Might Throw At You](#17-interview-drill-scenarios-they-might-throw-at-you)

---

## 1. What This Interview Actually Tests

This is not a quiz. This is a structured conversation where they want to see how you think when you encounter a real system. They will show you their current setup and then ask: *"What do you see? What would you do?"*

They are testing four things simultaneously:

**1. Can you diagnose where their heuristics-based system is breaking down?**
They're not using ML yet. They're using hand-coded rules querying Redshift. You need to articulate *why* that's the right place to start but the wrong place to stay — and what breaks as they scale: query latency, Redshift concurrency limits, no feedback loop, no personalization for new users.

**2. Do you understand the data foundation that has to exist before ML becomes viable?**
You can't drop AWS Personalize onto a system with no impression logging, no streaming primitive, and training data living only in a data warehouse. Before the first model, you need: a durable event log, impression tracking, and a clean interaction dataset. Do you know how to build that on their existing Rudderstack + Redshift stack without a full rewrite?

**3. Can you propose improvements that fit their constraints?**
Posh is 65 people. They are not building a Netflix-scale recommendation infrastructure. Your proposals need to respect their team size, their existing AWS bet, and the fact that they need to ship. "Rewrite everything in Flink" is the wrong answer. "Extend your Rudderstack routing to add Kinesis as a destination, build an impression logging contract, and connect that to Personalize's Event Tracker" is the right answer.

**4. Do you see the full picture — not just personalization, but the adjacent problems?**
They explicitly want help with MongoDB search scalability, not just recommendations. The engineer who walks in and only addresses the ML pipeline is leaving half the job undone. Show you can hold both problems simultaneously and propose coherent solutions that share infrastructure (e.g., an OpenSearch cluster that serves both search and semantic event retrieval for cold-start).

---

## 2. How to Read Their Architecture When They Show It To You

When they present their current setup, do this before you say anything:

### Trace the data flow end to end

Ask yourself: where does a raw interaction (a user clicking on an event, buying a ticket) enter the system? Where does it land? When does a model see it? When does a user feel the effect in their recommendations?

The gap between "event happens" and "model uses that event" is one of the most revealing signals of a system's maturity. If the answer is "we batch import to S3 once a day, then retrain weekly," that's a very different system than "we stream events to Kinesis, PutEvents to Personalize in real time."

### Look for these five structural questions

| Question | Why It Matters |
|---|---|
| How does training data get created? | Determines freshness, schema correctness, label leakage risk |
| Where are features stored? | If features are recomputed at request time you have consistency problems |
| How does a model get updated? | Manual redeploys → stale models. Automated pipelines → operational risk |
| How does the recommendation surface in product? | Sync API call in the critical path → latency risk. Pre-computed cache → staleness risk |
| How do they know if it's working? | No offline eval + no A/B infra → flying blind |

### Red flags to note silently (then address diplomatically)

You now know several of these are confirmed. Frame them as opportunities, not criticisms:

- **Redshift in the recommendation serving path** ← confirmed. This is the root cause of their outage. Redshift is an analytical warehouse; it wasn't designed for sub-100ms serving under concurrent user traffic.
- **No impression logging** ← confirmed. They only track clicks and RSVPs. Without knowing what was shown, you cannot compute CTR, you cannot identify what's *not* working, and you cannot build a proper feedback loop.
- **No streaming primitive** ← likely. Without Kinesis or equivalent, all data is batch. The model (when it exists) will always be learning from yesterday's behavior.
- **Search running on MongoDB** ← confirmed. MongoDB text indexes were designed for operational queries, not relevance-ranked full-text search. Atlas Search helps but still competes for resources with the operational DB.
- Data pipeline is a Redshift query pattern rather than an event-sourced log
- Features are recomputed at serving time from Redshift queries, not stored/versioned
- No monitoring on recommendation quality — only infrastructure monitoring

---

## 3. Posh's Confirmed Current Stack

This is not speculation — you learned this in prior rounds. Know it cold so nothing they show you surprises you.

### The Confirmed Architecture Today

```
Mobile / Web App (TypeScript frontend)
        |
        | clicks, RSVPs (NOT impressions — confirmed gap)
        |
        v
  Rudderstack (event routing / CDP layer)
  ┌─────────────────────────────────────────────────────┐
  │  Collects events from web + mobile SDKs             │
  │  Routes to configured destinations                  │
  └─────────────────────────────────────────────────────┘
        |
        | (current primary destination)
        v
  Amazon Redshift (data warehouse)
        |
        | SQL queries for heuristic personalization
        | e.g. "top events in user's city last 30 days"
        |      "popular events matching user's past categories"
        v
  Personalization API (Lambda / backend service)
        |
        v
  Product Surface (event feed, discovery page)

  Separately:
  MongoDB (application DB)
    ├─ Serves all application reads/writes
    └─ Search indexes (Atlas Search or native text indexes)
           → event search by name, category, organizer
           → worried about scaling — confirmed pain point
```

### What This Architecture Tells You

**The heuristics are Redshift queries.** They're computing recommendation candidates by querying the warehouse at request time or on a schedule. This is common for early-stage personalization — fast to build, interpretable, requires no ML expertise. But it has hard ceilings:

- **Latency**: Redshift queries take 100ms–several seconds. Fine for async/batch, catastrophic in a synchronous API call.
- **Concurrency**: Standard Redshift provisioned clusters have a default concurrency limit of 5 queries. Redshift Serverless scales better but still isn't designed for thousands of concurrent sub-100ms reads.
- **No learning**: heuristics don't improve from user behavior. They're static rules. If a user buys three jazz events in a row, the heuristic for "jazz events" is the same for them as for a new user with no history.
- **No feedback loop**: there is no mechanism to measure which recommendations led to ticket purchases.

**The outage during the large event was almost certainly this.** A high-profile event drives a traffic spike. Thousands of users hit the app simultaneously. Each feed load triggers a Redshift query. Redshift's concurrency queue fills up. Queries start timing out. The recommendation endpoint goes down or returns errors. Without recommendations, the fallback (if one exists) kicks in — or the feed breaks entirely.

### Confirmed Gaps

| Gap | Severity | Fix |
|---|---|---|
| No impression logging | Critical | Can't measure CTR or evaluate what's working |
| Redshift in serving path | Critical | Will fail again at the next large event |
| No streaming primitive | High | All data is batch; model will always learn from stale data |
| No ML model (heuristics only) | High | Heuristics don't scale with catalog size or user diversity |
| MongoDB doing search | Medium | Search competing with operational reads for DB resources |
| No A/B testing infrastructure | Medium | Can't measure if changes are improving outcomes |
| Cold start undefined | Medium | New users and new events have no explicit strategy |

---

## 4. The Two Adjacent Problems: Redshift Limits + MongoDB Search Scaling

These came up in prior rounds as explicit concerns. You need crisp answers for both. Neither is the personalization problem, but both are blocking problems that affect the same data infrastructure you're building.

---

### 4.1 Redshift: Why It's the Wrong Tool for Personalization Serving

Redshift is a columnar analytical warehouse built for complex aggregations over large datasets — think: "what was our gross revenue by city last quarter?" It is not built for:

- Sub-100ms single-record lookups
- Thousands of concurrent queries from live user traffic
- Real-time updates (rows are not updated in-place; they're marked deleted and re-inserted)

**The Concurrency Problem**

| Redshift cluster type | Default concurrency limit | With Concurrency Scaling |
|---|---|---|
| Provisioned (dc2.large, 2 nodes) | 5 queries | Up to 10 with auto concurrency scaling (adds cost) |
| Provisioned (ra3.xlplus) | 15 queries | Higher, but still finite |
| Redshift Serverless | Per-RPU auto-scaling | Better, but query startup latency still ~100ms+ |

When a large event goes live and 500 users simultaneously load their event feed, 500 queries hit Redshift. The queue fills immediately. The first queries take 200ms; by the time the queue backs up, queries are waiting 2–5 seconds. Then they time out. The feed breaks. This is exactly what happened in their outage.

**The Right Role for Redshift**

Keep Redshift — it's valuable for:
- BI dashboards (revenue by organizer, ticket sales by category, cohort retention)
- Training data preparation (batch queries to build the ML training dataset)
- A/B test analysis (querying the annotated event log)

Remove it from the serving path. Pre-compute recommendation results and serve from low-latency stores.

**Migration Path Off Redshift Serving**

```
TODAY (wrong):
  User requests feed → Lambda → Redshift query (100–500ms, limited concurrency) → response

PHASE 1 (safe transition):
  Background job (hourly) → Redshift query → write top-N recs per user → DynamoDB
  User requests feed → Lambda → DynamoDB GetItem (~3ms, unlimited concurrency) → response
  
  Cost: DynamoDB on-demand is ~$1.25 per million reads. 
        At 7M users × 5 feed loads/day = 35M reads/day = ~$44/day → ~$1,300/month.
        Acceptable. Use TTL on DynamoDB items (24h) to avoid stale recs piling up.

PHASE 2 (after Personalize is live):
  Personalize BatchInferenceJob (nightly) → DynamoDB
  Personalize Campaign (real-time) → for high-signal moments (post-follow, post-purchase)
  Redshift → training data only, not serving
```

**Talking point for the interview:**
> "Redshift is doing great work for your analytics, and I wouldn't remove it — but it's load-bearing in a place it wasn't designed for. The outage is a symptom of that. The fix isn't replacing Redshift; it's moving recommendation *serving* to DynamoDB with a pre-compute step, which also lowers your p99 latency from hundreds of milliseconds to under 5ms. Redshift stays in the picture as the training data source."

---

### 4.2 MongoDB Search Scaling — What's Breaking and How to Fix It

**Why search on MongoDB is a problem at scale**

MongoDB's text search (Atlas Search or native `$text` index) runs on the same cluster handling all your operational reads and writes. Event creates, ticket purchases, user profile updates — all competing for the same I/O and CPU as search queries.

As the catalog grows (more events, more organizers, more cities), search queries get more expensive. Atlas Search uses Lucene under the hood and is actually capable, but:
- It runs on the same Atlas nodes as your operational workload (no resource isolation)
- It doesn't scale independently — you scale Atlas for search, you scale it for everything
- Atlas Search has limitations for semantic/vector search that matter for personalized search

**The Two Options**

**Option A: Amazon OpenSearch Service (recommended for Posh)**

Fully managed Elasticsearch/OpenSearch cluster, completely decoupled from MongoDB. Events are indexed in OpenSearch at create/update time. All search traffic hits OpenSearch, not MongoDB.

```
Event created/updated (MongoDB)
  → MongoDB Change Streams or application-level dual-write
  → Lambda
  → OpenSearch index (events-v1)
        Fields: title, description, organizer_name, category, city,
                date_start, price_min, is_free, tags

Search query:
  User types "jazz NYC this weekend"
  → API Gateway → Lambda
  → OpenSearch query:
      {
        "query": {
          "bool": {
            "must": { "multi_match": { "query": "jazz", "fields": ["title^3", "description", "tags"] } },
            "filter": [
              { "term": { "city": "New York" } },
              { "range": { "date_start": { "gte": "now", "lte": "now+3d" } } }
            ]
          }
        }
      }
  → Return ranked results

Key advantage: OpenSearch also supports k-NN vector search.
  → Index event embeddings (from Bedrock/SageMaker) as knn_vector fields
  → Power semantic search ("events like this one") without a separate vector DB
```

**Option B: MongoDB Atlas Search (lower migration cost, fewer operational benefits)**

If they're already on MongoDB Atlas, Atlas Search is available as an add-on without adding a new service. Good for a short-term fix. Limitations:
- Runs on the same Atlas tier, still competes for resources
- More limited vector search capabilities than OpenSearch
- No independence in scaling: adding search capacity means adding Atlas capacity

**Recommendation for Posh:**
OpenSearch is the right long-term call because (a) it's fully decoupled from MongoDB, (b) it supports vector search which you'll need for semantic event discovery, and (c) it's an AWS service that fits the rest of the stack. The migration cost is one event-indexing Lambda and a sync job for the existing catalog. The operational cost is running an OpenSearch domain (t3.small.search at ~$50/month for dev, m6g.large for prod at ~$180/month).

**Talking point for the interview:**
> "The search concern is actually connected to the personalization work. If you move to OpenSearch for search, you get vector search capability as a side effect — which means semantic retrieval for cold-start personalization (new users, new events) comes almost for free. You'd be solving two problems with the same infrastructure decision."

---

## 5. The AWS Personalize Mental Model — Know It Cold

This is the managed service they're open to adopting. You need to understand it at the level of someone who has operated it in production, not just read the docs.

### Core Concepts

**Dataset Group** — the namespace for all your Personalize resources. Everything lives inside one. You can have multiple for different use cases (event discovery vs. follow recommendations).

**Datasets** — three types, and the schema requirements are strict:
- `Interactions` — required. Minimum: `USER_ID`, `ITEM_ID`, `TIMESTAMP`. Optionally: `EVENT_TYPE`, `EVENT_VALUE`. This is the most important dataset — Personalize builds almost everything from it.
- `Items` — metadata about your items (events). Supports categorical and numerical fields. Used for cold start and filtering.
- `Users` — metadata about users. Useful for contextual personalization but has a weaker effect than interaction history.

**Solution** — a trained model. You pick a Recipe (algorithm), provide config, and create a Solution Version.

**Recipe types** — know these:
| Recipe | Use Case |
|---|---|
| `aws-user-personalization` | Personalized top-N items per user. Main recipe for event discovery. |
| `aws-user-personalization-v2` | Improved version with automatic exploration. Prefer this. |
| `aws-popularity-count` | Baseline: globally popular items. Use as fallback / cold start. |
| `aws-similar-items` | Item-to-item similarity. "Events like this one." |
| `aws-personalized-ranking` | Re-rank a candidate list you provide. Good for blending with rule-based filtering. |
| `SIMS` | Similar items via collaborative filtering. |

**Campaign** — a deployed Solution Version. You query it with `GetRecommendations`. Campaigns have a `minProvisionedTPS` setting — this is a cost driver. Set too high and you're paying for idle capacity 24/7.

**Event Tracker** — the mechanism for real-time interaction ingestion. You call `PutEvents` from your application with a `trackingId`. Personalize immediately uses these events to adjust recommendations for existing users (no retraining required). New users see popular items until they have interaction history.

### The Key Operational Trade-offs

**Managed = fast, but constrained**
AWS Personalize is a black box. You cannot inspect the model weights, modify the architecture, or override the ranking function. This is fine for baseline personalization, but it's limiting when you want to:
- Incorporate business logic (boost events from followed organizers)
- Use custom features not supported in the schema
- Run sophisticated multi-stage retrieval + ranking pipelines

**Retraining cadence**
- Incremental retraining (recommended): trains on new data using existing solution as warm start. Takes 30-60 min.
- Full retraining: trains from scratch. Takes hours. Only needed if you've significantly changed your dataset schema or recipe.
- Best practice: trigger retraining after you've accumulated enough new interactions (roughly 1-5% new data relative to total). Weekly retraining is often right at Posh's scale.

**Cold start**
- New users: Personalize immediately starts personalizing once you stream even 1-2 events via PutEvents. Without that, they get popularity-based recommendations.
- New items (new events): items must be in the Items dataset to be recommended. There's a delay between when an organizer creates an event and when it becomes eligible for recommendations if you do batch imports. This is a real product problem.

**Filters**
Personalize supports filter expressions applied to results. This is how you implement: "don't show events the user already purchased," "only show events in their city," "only show upcoming events." Filters are evaluated post-recommendation and can exclude items — so request more than you need (e.g., ask for 50, filter down to 10).

### Common Production Mistakes with Personalize

1. **Importing data with wrong TIMESTAMP format** — must be Unix epoch (seconds). Nanoseconds or ISO strings silently produce bad training data.
2. **Using EVENT_TYPE inconsistently** — mixing "view," "View," "EVENT_VIEW" in the same dataset confuses the model. Normalize event types rigorously.
3. **Not excluding test/internal users** from training data — Posh employees testing the app will skew interaction patterns.
4. **Not tracking negative signals** — if you only track purchases, the model doesn't know about events the user browsed and rejected. Impressions with no click are useful negative training signal. Personalize supports `EVENT_VALUE` to encode implicit feedback.
5. **minProvisionedTPS too high** — a campaign with minProvisionedTPS=10 costs ~$744/month even at zero traffic. At Posh's scale, minProvisionedTPS=1 is likely fine; use auto-scaling.
6. **Single campaign for all surfaces** — the "Discover Events" feed and the "Similar Events" widget need different recipes. Don't force one campaign to serve both.

---

## 6. The AWS SageMaker Mental Model — Know It Cold

The job description says "translate experimental outputs from AWS SageMaker/Personalize into production-ready systems." At Posh, there are no experiments yet — you're the person who will *start* them. Your job is to build the infrastructure that makes experimenting viable and turns results into production pipelines.

### The Experiment-to-Production Gap (This Is The Core Problem)

At Posh today, they're not even at the "notebook experiment" stage — they're using SQL heuristics against Redshift. The gap you're closing is bigger than the typical experiment-to-production gap: it starts one step earlier.

```
TODAY:
  Redshift SQL heuristic → personalization output
  No model. No training data. No offline evaluation.

WHAT NEEDS TO EXIST BEFORE SAGEMAKER IS USEFUL:
  1. A durable interaction event log (clicks, RSVPs, impressions)
  2. A clean Items dataset (events with structured metadata)
  3. A Users dataset (with enough signal to personalize)
  4. A training data generation pipeline
  5. An offline evaluation framework (what does "good" look like?)

THEN: the classic experiment-to-production problem kicks in:
  6. Notebook experiment (someone trains a model, evaluates it)
  7. "It looks good!" — but it never makes it to production
  8. Or it makes it to production in a non-reproducible way
```

This is the gap you're being hired to close in full. Understand its components:

### SageMaker Components You Need to Know

**SageMaker Pipelines** — the orchestration layer for ML workflows. A Pipeline is a DAG of steps: `ProcessingStep` (feature engineering), `TrainingStep`, `EvaluationStep`, `ConditionStep` (only deploy if metrics pass threshold), `RegisterStep`. This is how you turn a notebook experiment into a reproducible, automated pipeline.

**SageMaker Feature Store** — a managed feature store with two storage tiers:
- **Online store** (DynamoDB-backed): low-latency reads for real-time inference (~5ms p99). Stores the latest feature value per record.
- **Offline store** (S3-backed Parquet): full time-series of feature values for training. Automatically keeps history.
Feature groups have a schema you define once. Features are written with a `RecordIdentifierValue` (user ID or item ID) and `EventTime`.

**SageMaker Model Registry** — versioned catalog of trained models. Models go through: `PendingManualApproval → Approved → Deployed`. This is how you create a promotion workflow: experiments train models into the registry, engineers review quality metrics and approve, automation deploys.

**SageMaker Endpoints** — managed real-time inference. Deploy a model artifact + container to an endpoint. Supports:
- Real-time endpoints: synchronous, low-latency, always-on
- Serverless inference: scales to zero, cold starts (~1s). Good for low-traffic use cases.
- Async inference: queue-based, for large payloads or batch-like workloads

**SageMaker Experiments** — tracks runs, hyperparameters, metrics. Lets you compare model versions systematically instead of relying on notebook notes.

### The Productionalization Checklist

When you're asked "how would you take this SageMaker experiment to production," here is your answer structure:

```
1. REPRODUCIBILITY
   - Move data pull from ad-hoc SQL to a versioned, timestamped S3 snapshot
   - Move feature engineering from notebook to a SageMaker ProcessingJob (Python script in a container)
   - Pin library versions in a requirements.txt / Docker image
   - Store the exact dataset version used for training alongside the model artifact

2. AUTOMATION
   - Build a SageMaker Pipeline: ProcessingStep → TrainingStep → EvaluationStep → ConditionStep → RegisterStep
   - Trigger via EventBridge: weekly schedule, or on new data arrival in S3
   - Use Step Functions if you need cross-service orchestration (e.g., trigger Personalize reimport after SageMaker training)

3. VALIDATION GATE
   - Define a minimum quality threshold (e.g., Precision@10 ≥ 0.15, NDCG@10 ≥ 0.20)
   - ConditionStep in the pipeline checks this before registering the model
   - Never auto-deploy a model that hasn't passed the quality gate

4. DEPLOYMENT
   - Register approved model in SageMaker Model Registry
   - Deploy to SageMaker Endpoint with traffic shifting (blue/green or canary)
   - Or: batch score and write to DynamoDB/ElastiCache for low-latency serving

5. MONITORING
   - SageMaker Model Monitor: detect data drift (input distribution shift)
   - SageMaker Clarify: bias detection and feature attribution
   - Custom CloudWatch metrics: track recommendation quality in production (CTR, conversion)
   - Alert on drift, trigger retraining
```

---

## 7. Data Pipeline Patterns — Batch vs. Streaming vs. Lambda

The personalization system's quality is bounded by the freshness and completeness of its data. This is a core architectural decision.

### Current Pipeline (Rudderstack → Redshift, confirmed)

```
App events (clicks, RSVPs only)
  → Rudderstack SDK
  → Rudderstack server (routing layer)
  → Redshift (primary destination today)

Personalization:
  → Scheduled SQL query on Redshift
  → Heuristic scoring (popularity, recency, city filter)
  → API returns results
```

**The problems with this pipeline for personalization:**
- Redshift is the serving layer — wrong for reasons detailed in Section 4.1
- Only clicks and RSVPs: the training signal is incomplete. No impressions means you can't distinguish "user saw this and chose not to click" from "user never saw this."
- No event type for views/saves/follows — the model has no gradient between "mild interest" and "strong intent"
- Rudderstack is routing only to Redshift today — no S3 archive, no streaming, no ML destination

### The Transition: Extending Rudderstack to New Destinations

The good news: Rudderstack already handles event ingestion. Getting events to AWS Personalize, S3, and Kinesis is mostly **a routing configuration problem**, not a new infrastructure build.

Rudderstack supports destinations including:
- **Amazon S3** (via native S3 destination) — raw event archive for training data
- **Amazon Kinesis Data Streams** (via native Kinesis destination) — real-time fan-out
- **Amazon Redshift** (existing) — keep for analytics, remove from serving path

Rudderstack does **not** have a native Amazon Personalize destination. You bridge via Kinesis → Lambda → `PutEvents`. This is a ~50-line Lambda.

```
EXTENDED PIPELINE:

App events (clicks, RSVPs + NEW: impressions, views, saves, follows)
  → Rudderstack SDK
  → Rudderstack server
      ├─ → Redshift (analytics / BI — keep)
      ├─ → S3 (raw event archive — add as new destination)
      │       → Glue / Athena → ML training data
      └─ → Kinesis Data Streams (add as new destination)
              → Lambda (transform to Personalize schema)
                  → Personalize PutEvents (real-time rec updates)
              → Firehose → S3 (Parquet, partitioned — backup for batch training)
              → Lambda → SageMaker Feature Store (user feature updates)

Recommendation serving (separate path):
  Background job → Personalize BatchInferenceJob
  → DynamoDB (pre-computed top-N per user)
  → Lambda serves from DynamoDB, not Redshift
```

### Near Real-Time Pipeline (Once Rudderstack Routes to Kinesis)

```
Application (TypeScript/Java)
  → Rudderstack → Kinesis Data Streams
      → Lambda → Personalize PutEvents   ← immediate rec update
      → Firehose → S3 (Parquet)          ← training data archive
```

This is the architecture change with the highest ROI-to-effort ratio. PutEvents means new interactions influence recommendations within seconds, not hours.

**How PutEvents works:**
```python
personalize_events_client.put_events(
    trackingId='your-event-tracker-id',
    userId='user_123',
    sessionId='session_abc',
    eventList=[{
        'eventId': 'uuid',
        'eventType': 'ticket_purchase',  # or 'event_view', 'event_save', 'rsvp'
        'sentAt': datetime.now(),
        'itemId': 'event_456',
        'properties': json.dumps({'eventValue': 1.0})
    }]
)
```

### Full Streaming Pipeline (for when you've outgrown Personalize)

If Posh eventually needs more control over the ranking model than Personalize allows:

```
Application Events
  → Kinesis Data Streams (partitioned by user_id)
      → Kinesis Data Analytics (Flink) or Lambda
          → Feature computation (windowed aggregations)
          → SageMaker Feature Store (online + offline)
      → Firehose → S3 → Glue Catalog → Athena/Redshift (analytics)
  
Training Pipeline (daily/weekly):
  SageMaker Pipeline
    → Read from Feature Store offline store
    → ProcessingJob (feature engineering)
    → TrainingJob (custom model)
    → EvaluationJob
    → Model Registry

Serving Pipeline (real-time):
  API Gateway → Lambda
    → Feature Store online read (user features, item features)
    → SageMaker Endpoint (inference)
    → Results cache (ElastiCache/DynamoDB)
    → Response
```

### The Lambda Architecture Hybrid (Pragmatic for Posh's Scale)

Don't jump straight to full streaming. For a 7M user platform with a 65-person team, this hybrid is right:

| Layer | Technology | Refresh Rate | Use |
|---|---|---|---|
| Batch layer | Rudderstack → S3 + Personalize import | Daily | Full retraining with all historical data |
| Speed layer | Rudderstack → Kinesis → PutEvents | Real-time | Recent interactions influence recs immediately |
| Serving layer | Personalize BatchInferenceJob → DynamoDB | Sub-10ms | Product-facing API (replaces Redshift queries) |

The key insight: Rudderstack already sits in front of all event data. You're not adding event capture infrastructure — you're adding routing destinations and a serving layer.

---

## 8. Feature Stores — The Bridge Between Pipelines and Models

This is the concept that separates senior engineers from junior engineers when discussing ML systems. Feature stores solve the **training/serving skew** problem and the **feature reuse** problem.

### The Problem Without a Feature Store

In most early ML systems, features are computed differently in two places:
- **Training time:** a SQL query or pandas transformation that runs once when you train the model
- **Inference time:** some code in a Lambda or API handler that recomputes the feature on the fly

These two computations drift. A feature "user's purchase rate in the last 30 days" computed by a SQL query at training time won't always match the same feature computed from a DynamoDB lookup at inference time because:
- The schema diverged
- The definition of "30 days" is calculated differently
- One pipeline includes cancelled orders, the other doesn't
- Timezone handling differs

This produces **training/serving skew**: the model was trained on data that doesn't match the data it sees at prediction time. Performance degrades mysteriously and is hard to debug.

### How SageMaker Feature Store Solves This

You define a Feature Group (schema + config). Features are ingested once via the `put_record` API:

```python
featurestore_runtime.put_record(
    FeatureGroupName='user-engagement-features',
    Record=[
        {'FeatureName': 'user_id', 'ValueAsString': '123'},
        {'FeatureName': 'purchase_count_30d', 'ValueAsString': '5'},
        {'FeatureName': 'avg_event_price_30d', 'ValueAsString': '42.50'},
        {'FeatureName': 'last_event_category', 'ValueAsString': 'music'},
        {'FeatureName': 'event_time', 'ValueAsString': str(int(time.time()))},
    ]
)
```

Now both training and serving read from the same source:
- Training reads the **offline store** (S3 Parquet with full history) using a batch query
- Serving reads the **online store** (DynamoDB, ~5ms) via the GetRecord API

Same feature definition. Same values (accounting for time). Skew eliminated.

### Feature Groups Posh Needs

If you're proposing a feature store in the interview, have concrete feature groups in mind:

**`user-interaction-features`**
- `purchase_count_7d`, `purchase_count_30d`
- `event_view_count_7d`
- `avg_ticket_price_purchased`
- `top_event_category` (most-attended category)
- `has_social_graph` (bool: does user follow anyone)
- `follow_count`, `followed_by_count`
- `city_primary` (inferred from purchase history)

**`event-content-features`**
- `event_category` (music, sports, arts, etc.)
- `price_tier` (free, low, medium, high)
- `days_until_event`
- `organizer_follower_count`
- `similar_event_purchase_rate` (popularity signal)
- `event_embedding` (dense vector, computed by SageMaker)

**`user-event-interaction-features`** (cross features, real-time)
- `user_has_purchased_from_organizer` (bool)
- `user_follows_organizer` (bool)
- `category_affinity_score` (user's historical engagement with this category)

---

## 9. Productionalizing ML Models — The Hard Part They Need Help With

The job description says "translate experimental outputs into production-ready systems, ensuring efficiency and reproducibility." This is the core of why they're hiring you. Know this cold.

### The Five Failure Modes of ML Experiments That Never Ship

**1. Notebook-to-production gap**
Experiment lives in a Jupyter notebook. No one can re-run it with new data. Key transformations exist only in-memory. Fix: enforce that all experiments are backed by versioned scripts in git, not notebooks. Notebooks for exploration only; `.py` files for anything that needs to run again.

**2. Data leakage in offline evaluation**
The experiment looks great offline but fails in production because future information leaked into training features. Example: using `purchase_count_all_time` as a feature includes purchases that happened *after* the event being evaluated. Fix: point-in-time correct feature extraction. Only use features available at the time of the interaction.

**3. No production-equivalent evaluation set**
Offline evaluation uses a random train/test split. Production traffic has a time dimension and selection bias (you can only evaluate recommendations that were shown). Fix: temporal train/test split (train on first 80% of time, test on last 20%). Track Precision@K, NDCG@K, and coverage.

**4. Serving performance never validated**
The model was evaluated for quality but not for latency. A 500ms inference time is fine in a notebook, catastrophic in the critical path of an event discovery API. Fix: load test the serving infrastructure before declaring the model production-ready. P99 < 100ms for synchronous recommendation APIs.

**5. No rollback plan**
Model is deployed, something goes wrong (latency spike, recommendation quality collapse), and no one knows how to roll back because the previous model artifact was overwritten. Fix: SageMaker Model Registry with versioned model packages. Never overwrite — always create a new version. Keep the N-1 version deployed in shadow mode for 24h.

### The Production Pipeline Template

When Cat and Ethan ask how you'd build the pipeline, anchor to this structure:

```
[Data Collection] → [Feature Engineering] → [Training] → [Evaluation] → [Registry] → [Deployment] → [Monitoring] → [Feedback Loop]
```

Each arrow is a hand-off with a defined interface. Every step should be:
- **Idempotent**: run it twice, same output
- **Versioned**: data snapshot, model artifact, feature definitions all have versions
- **Observable**: each step emits metrics to CloudWatch
- **Gated**: quality checks before promotion

### Concretely for Posh

```
Step 1: DATA COLLECTION
  Trigger: EventBridge (daily at 3am UTC, off-peak)
  Source: S3 event log (from Kinesis Firehose delivery)
  Output: timestamped Parquet snapshot in S3
  Schema validation: Great Expectations or custom Lambda check

Step 2: FEATURE ENGINEERING
  SageMaker ProcessingJob (Python, Spark via EMR Serverless for large jobs)
  Computes feature groups, writes to SageMaker Feature Store
  Writes training dataset (feature matrix + labels) to S3

Step 3: TRAINING
  SageMaker TrainingJob
  Container: XGBoost built-in, or custom PyTorch container
  Hyperparameter tuning: SageMaker HyperParameter Tuning Jobs (Bayesian search)
  Artifacts: model.tar.gz → S3 model artifacts bucket

Step 4: EVALUATION
  SageMaker ProcessingJob (evaluation script)
  Computes: Precision@10, NDCG@10, coverage, popularity bias
  Writes metrics JSON to S3

Step 5: QUALITY GATE
  SageMaker Pipeline ConditionStep
  Condition: NDCG@10 >= threshold AND coverage >= 0.3
  On pass → RegisterModel
  On fail → SNS notification to #ml-alerts Slack channel

Step 6: MODEL REGISTRY
  SageMaker Model Registry: status = "PendingManualApproval"
  Automated approval for weekly retrains if quality delta < 5%
  Manual approval required if quality delta > 5% (human review)

Step 7: DEPLOYMENT
  On approval trigger → Lambda → update SageMaker Endpoint (blue/green)
  Or: batch score top-N recs per user → write to DynamoDB
  Shadow deployment for 24h alongside previous model version

Step 8: MONITORING
  SageMaker Model Monitor: detect input data drift
  Custom Lambda: daily sample of recs → human spot-check
  CloudWatch: track CTR, conversion, recommendation diversity
  Alert threshold: if CTR drops > 15% vs. 7-day average → trigger investigation
```

---

## 10. Event Tracking Architecture — Rudderstack as the Foundation

Posh already has Rudderstack. This is significant — event capture infrastructure exists and is in production. The gap isn't "we need to track events," it's "we're only tracking two event types and routing them to one destination."

### What Rudderstack Is (Know This Cold)

Rudderstack is a Customer Data Platform (CDP) — it sits between your application and all the places your event data needs to go. Think of it as an event router with SDK libraries for web/mobile.

```
SDK (web / iOS / Android)
  → Rudderstack Data Plane (cloud or self-hosted)
  → Configured Destinations (Redshift, S3, Kinesis, Amplitude, etc.)
```

Rudderstack handles: batching, retries, schema transformation (to match destination schemas), dead letter queuing for failed deliveries.

**What they track today:** `track("Clicked Event")`, `track("RSVP'd")` — two event types.

**What's missing (and you need to add):**

| User Action | Event Type | Signal Value | Currently Tracked? |
|---|---|---|---|
| View event detail page | `Event Viewed` | 0.2 | No |
| Save / bookmark event | `Event Saved` | 0.6 | No |
| Purchase ticket | `Ticket Purchased` | 1.0 | Possibly (RSVP may cover this) |
| RSVP (free event) | `Event RSVP'd` | 0.8 | Yes |
| Follow organizer | `Organizer Followed` | 0.7 | Unknown |
| Share event | `Event Shared` | 0.8 | No |
| **Scroll past without clicking** | `Event Impression` | -0.1 | **No — most critical gap** |
| Cancel / refund ticket | `Ticket Cancelled` | -0.5 | No |
| Click on recommendation | `Recommendation Clicked` | N/A | No (attribution gap) |

### The Impression Logging Gap — Why It's the Most Critical

Impressions are the denominator. CTR = clicks / impressions. Without impressions:
- You cannot compute CTR
- You cannot distinguish "user ignored this event" from "user never saw it"
- You cannot train a model that learns from non-engagement
- You cannot measure whether any recommendation strategy is working

**How to add impression logging with Rudderstack:**

In the event feed component (React Native / web), when items render on screen:

```typescript
// When an event card becomes visible in the viewport
const onEventImpression = (eventId: string, position: number, surfaceId: string) => {
  rudderClient.track('Event Impression', {
    event_id: eventId,
    position: position,         // rank position in the feed (1-indexed)
    surface: surfaceId,         // 'discovery_feed', 'similar_events', 'search_results'
    recommendation_run_id: runId, // which rec output was shown (for attribution)
  });
};
```

Impression events must include:
- `event_id` — what was shown
- `user_id` (from Rudderstack session)
- `position` — where in the feed it appeared
- `surface` — which recommendation surface
- `recommendation_run_id` — ties the impression back to the specific model output

Without `recommendation_run_id`, you can't do proper attribution when the user later purchases.

### Rudderstack Destination Architecture (Target State)

```
App events (all types: view, save, purchase, rsvp, impression, follow)
  → Rudderstack SDK (web + mobile)
  → Rudderstack Data Plane
      │
      ├─ Destination 1: Amazon Redshift (KEEP — analytics + BI)
      │    → Revenue dashboards, cohort analysis, A/B test queries
      │
      ├─ Destination 2: Amazon S3 (ADD)
      │    → Raw event archive (JSON, partitioned by date/event_type)
      │    → Glue Catalog → Athena queries
      │    → ML training data source
      │
      └─ Destination 3: Amazon Kinesis Data Streams (ADD)
           → Consumer 1: Lambda → Personalize PutEvents (real-time rec updates)
           → Consumer 2: Firehose → S3 Parquet (structured training data)
           → Consumer 3: Lambda → SageMaker Feature Store (user feature updates)
```

Rudderstack's native Kinesis destination takes ~1 hour to configure. Adding S3 as a destination is similarly straightforward. This is configuration, not engineering — the engineering work is downstream (the Kinesis consumers).

### What to Validate in the Interaction Log

These data quality problems silently poison recommendation quality:

1. **Duplicate events** — a user's single purchase recorded 3 times due to retry logic. Rudderstack has `messageId` (UUID) on every event — use this for deduplication in the Kinesis consumer before feeding to Personalize.
2. **Stale timestamps** — mobile apps queue events offline and flush them hours later. Rudderstack uses `originalTimestamp` (when the event happened on device) and `sentAt` (when it was sent). For Personalize, use `originalTimestamp`. For real-time PutEvents, both are fine.
3. **Bot / internal traffic** — filter out Posh employee accounts and load testing before events enter training data. Maintain a blocklist of internal user IDs in the Lambda consumer.
4. **Missing item IDs** — if `event_id` in Rudderstack refers to an event that doesn't exist in the Personalize Items dataset, Personalize silently ignores that interaction. Monitor the ratio of "interactions with missing items" — this reveals catalog sync lag.
5. **Event type name inconsistency** — Rudderstack uses human-readable event names ("Ticket Purchased"). Personalize uses `EVENT_TYPE` — normalize these at the Lambda consumer layer. Pick a canonical mapping and enforce it in code, not documentation.

---

## 11. Model Serving Infrastructure

This is where experiments become product. The serving layer must be fast, reliable, and decoupled from the training pipeline.

### Two Serving Patterns — Know When to Use Each

**Pattern 1: Online / Real-Time Serving**
```
User requests event feed
→ API Gateway → Lambda (recommendation service)
    → Personalize GetRecommendations API  (or SageMaker Endpoint)
    → Apply business rule filters (upcoming events, city, followed organizers boosted)
    → Return ranked list
→ App renders feed
```
**Latency budget:** Personalize GetRecommendations p99 ≈ 50-80ms. Add Lambda overhead and you're at ~100-150ms. For a mobile feed, that's acceptable if it's not in the critical path of initial render.

**Pattern 2: Pre-computed / Cached Serving**
```
Background job (every 1-6 hours, or triggered by retraining):
  → Personalize BatchInferenceJob: score top-100 recs for all active users
  → Write results to DynamoDB: {userId → [eventId1, eventId2, ...eventId100]}
  
User requests event feed:
  → Lambda → DynamoDB GetItem (single key lookup)  ← ~3ms
  → Apply real-time filters (sold out, already purchased)
  → Return ranked list
```
**Tradeoff:** Near-zero latency for serving, but recommendations can be stale (minutes to hours). For an events app, this is usually fine — users don't expect their feed to change the instant they view an event. Use this pattern for the main discovery feed.

**Hybrid:** Use pre-computed for the main feed (speed), real-time for "Similar Events" widget (freshness matters because the context is a specific event the user just opened).

### Caching Strategy

Never call Personalize in the critical render path without a cache. Use:

- **ElastiCache (Redis):** cache `GetRecommendations` results per user. TTL of 15-60 minutes. Invalidate on significant user action (purchase, follow). Cost: ~$50-200/month for a cache.cluster.r6g.large.
- **DynamoDB:** better for pre-computed recs (persistent, not ephemeral). Store top-N recs per user as a list attribute.

### Business Rule Layer

Never serve raw model output directly. Always pass through a business rule layer:

```python
def get_recs(user_id: str, n: int = 20) -> List[EventId]:
    # Step 1: get raw recs from model
    raw_recs = personalize.get_recommendations(userId=user_id, numResults=50)
    
    # Step 2: enrich with metadata (from cache or DB)
    events = fetch_event_metadata(raw_recs.item_list)
    
    # Step 3: apply hard filters
    events = [e for e in events if e.is_upcoming]
    events = [e for e in events if e.id not in user_purchased_event_ids(user_id)]
    events = [e for e in events if e.city in user_preferred_cities(user_id)]
    
    # Step 4: apply soft boosts (re-rank)
    for event in events:
        if event.organizer_id in user_followed_organizers(user_id):
            event.score *= 1.5  # boost followed organizers
        if event.is_trending:
            event.score *= 1.2
    events.sort(key=lambda e: e.score, reverse=True)
    
    return [e.id for e in events[:n]]
```

This layer handles the business logic that the model doesn't know about. It's also where you can A/B test specific ranking adjustments without retraining.

---

## 12. Data Quality, Observability, and Feedback Loops

Most companies monitor their infrastructure (uptime, latency). Very few monitor their data and model quality. This is where you add the most value.

### The Three Layers of Observability

**Layer 1: Infrastructure observability** (they probably have this)
- API latency, error rates, Lambda duration, DynamoDB read/write capacity
- Standard CloudWatch dashboards

**Layer 2: Data pipeline observability** (they probably don't have this)
- Row count at each pipeline stage (events captured → events validated → events imported → interactions in training set)
- Schema drift detection: alert if a new event type appears in the log
- Feature freshness: track the `event_time` of the most recent record per feature group
- Missing data rate: % of users with < 10 interactions (model performs poorly for them)

**Layer 3: Recommendation quality observability** (they almost certainly don't have this)
- **Offline metrics** (computed on held-out data): Precision@10, NDCG@10, coverage, catalog coverage, novelty
- **Online proxy metrics** (from production): recommendation CTR, post-recommendation conversion (ticket purchase rate), mean position of clicked items
- **Business metrics** (what actually matters): session ticket purchase rate for users who saw personalized recs vs. baseline, revenue attributable to personalization
- **Diversity and freshness**: are the same 20 events showing up for everyone? What % of the catalog gets recommended in a given week?

### The Feedback Loop Architecture

A personalization system without a feedback loop is not a learning system — it's a static filter.

```
User sees recommendations
    ↓
User interacts (or doesn't)
    ↓
Events captured (PutEvents + Kinesis)
    ↓
Features updated (Feature Store)
    ↓
Retraining triggered (when sufficient new data)
    ↓
Quality gate passed
    ↓
New model deployed
    ↓
User sees better recommendations
    ↓
(repeat)
```

The critical missing pieces in most early-stage systems:
1. **Impression logging** — you can't compute CTR if you don't know what was shown. Log every recommendation that was shown to a user, with position.
2. **Attribution** — track not just whether a user purchased, but whether they purchased an event from the recommendation surface (vs. direct search, vs. organizer link)
3. **Counterfactual evaluation** — the recs that weren't shown can't give you feedback. This is the exploration problem. AWS Personalize v2 has automatic exploration (5% of traffic gets exploratory recs) — make sure this is enabled.

---

## 13. A/B Testing Infrastructure for Personalization

You cannot improve personalization without the ability to measure the effect of changes. If there's no A/B test infrastructure, every change is a leap of faith.

### Why Personalization A/B Tests Are Harder Than Feature A/B Tests

**Interference (spillover):** In a typical feature A/B test, user A in the control group doesn't affect user B in the treatment group. In a social/events platform, this breaks: if a popular event gets boosted in treatment group recommendations, it sells out faster, and then control group users can't buy it either. This inflates the treatment effect.

**Novelty effects:** Users in the treatment group may engage more simply because recommendations feel "different," not because they're better. Wait 1-2 weeks before declaring a winner.

**Long feedback loops:** A user who sees a recommendation today might buy a ticket 3 days later. Your experiment needs a conversion window, not just a click-through window.

### The Minimum Viable A/B Test Infrastructure for Personalization

```
User bucket assignment:
  → Hash(user_id) % 100 → assignment to experiment group
  → Store in: Redis or DynamoDB (userId → {experimentId, variant, assignedAt})
  → Assignment must be stable: same user always gets same variant

Recommendation service:
  if user.variant == 'control':
      recs = personalize_campaign_control.get_recommendations(userId)
  elif user.variant == 'treatment':
      recs = personalize_campaign_treatment.get_recommendations(userId)
  
Event logging:
  All events must be annotated with experiment variant:
  {userId, eventType, itemId, timestamp, experimentId, variant}

Analysis:
  → Redshift or Athena query on annotated event log
  → Primary metric: ticket purchase rate (treatment vs. control, Mann-Whitney U)
  → Guardrail metrics: page load time, error rate (must not regress)
  → Statistical significance: 80% power, p < 0.05, minimum 2-week runtime
```

### AWS Personalize's Built-In A/B Support

Personalize supports running multiple campaigns simultaneously and splitting traffic. You can also use `GetPersonalizedRanking` to re-rank a blended candidate list and A/B test the ranker separately from the retriever.

---

## 14. The Full Production Blueprint — How You'd Build It

When Cat and Ethan ask "how would you build the pipelines with our architecture?" — give them this. The core principle: **don't propose a greenfield rewrite. Build on Rudderstack and Redshift they already have. Every phase should be independently deployable and immediately valuable.**

### Phase 1: Fix the Serving Layer + Instrument Impressions (Weeks 1-3)
*Goal: prevent the next outage and create the measurement foundation. Do this before any ML work.*

1. **Pre-compute heuristic recs to DynamoDB** — take the Redshift heuristic SQL that's currently in the serving path and move it to a background job. Output writes to DynamoDB (userId → [eventIds]). The API now reads from DynamoDB, not Redshift. Same recommendations, no Redshift in the critical path. This directly addresses the outage.
   ```
   EventBridge (hourly schedule) → Lambda → Redshift query → DynamoDB write
   API path: Lambda → DynamoDB GetItem (~3ms) → response
   ```

2. **Add impression logging to the event feed.** Add a Rudderstack `track("Event Impression")` call in the frontend feed component when event cards render on screen. Include `event_id`, `position`, `surface`, and a `run_id` that ties back to which pre-computed batch produced this result. This is the single most important instrument you can add — without it, nothing else is measurable.

3. **Add additional Rudderstack event types**: `Event Viewed`, `Event Saved`, `Organizer Followed`, `Event Shared`, `Ticket Cancelled`. These fill out the interaction signal vocabulary before you build the ML pipeline.

4. **Add Rudderstack → S3 destination.** Configure raw event archive in S3, partitioned by date and event type. This creates the durable event log that ML training will pull from.

5. **CloudWatch dashboard**: recommendation DynamoDB read latency, Rudderstack delivery success rate, Redshift query duration (for the now-offline background job).

### Phase 2: Build the Data Foundation for ML (Weeks 4-8)
*Goal: create the clean interaction dataset and event catalog that AWS Personalize requires.*

1. **Add Rudderstack → Kinesis destination.** From Kinesis, fan out to: (a) Lambda → Personalize PutEvents, (b) Firehose → S3 Parquet. This gives you real-time updates to Personalize and a structured training archive.

2. **Build the Personalize Items dataset sync.** When an event is created or updated in MongoDB, trigger an upsert to the Personalize Items dataset. Fields: `category`, `city`, `price_tier`, `days_until_event`, `organizer_id`, `is_free`. This is where OpenSearch indexing also happens (same trigger, same event, two destinations).

3. **Build the initial Personalize Interactions dataset.** Pull 6-12 months of historical clicks and RSVPs from Redshift, transform to Personalize schema (USER_ID, ITEM_ID, TIMESTAMP, EVENT_TYPE, EVENT_VALUE), validate schema, import to Personalize. This is the training seed.

4. **Set up the first Personalize solution.** Use `aws-user-personalization-v2`. Train on the historical dataset. Do not deploy yet — evaluate offline first (coverage, NDCG proxy via temporal split on held-out data).

5. **Deploy OpenSearch.** Migrate event search off MongoDB. Index events in OpenSearch at create/update time. A/B test search quality (not personalization yet — just relevance). This resolves the MongoDB search scaling concern independently.

### Phase 3: Deploy ML Personalization (Weeks 9-16)
*Goal: replace the heuristic with the model. Measure the difference.*

1. **A/B test infrastructure.** User bucketing service, annotated event log, Athena analysis queries. Deploy the Personalize campaign to a treatment group (50% of users). Control group continues to see pre-computed heuristic recs from DynamoDB.

2. **Deploy Personalize campaign.** `GetRecommendations` → DynamoDB write (BatchInferenceJob, nightly). Same serving architecture as heuristics — DynamoDB, not Personalize in the critical path. Only difference: the content of DynamoDB changes from heuristic results to model results.

3. **Add business rule layer** on top of raw Personalize output: filter already-purchased events, boost events from followed organizers, cap events more than 90 days out.

4. **First A/B test.** Run heuristic vs. Personalize for 2 weeks. Primary metric: ticket purchase rate per recommendation surface visit. Guardrail: page load time, error rate. If Personalize wins → roll out. If not → investigate what signals the heuristic uses that Personalize doesn't have (city? recency?) and add them to the Items/Users dataset.

5. **Cold start strategy.** New users: onboarding interest picker (3-5 categories) → store as synthetic interactions → immediate cold-start personalization. New events: on-create trigger adds to Personalize Items dataset immediately (no batch delay).

### Phase 4: Deepen the Signal (Weeks 17–24)
*Goal: make the model learn from everything, not just clicks and RSVPs.*

1. **SageMaker Feature Store** for the most important cross-features: `user_follows_organizer`, `user_category_affinity_score`, `organizer_avg_attendance`. These features aren't available inside Personalize — they feed a custom re-ranker.

2. **Custom re-ranker in SageMaker.** Use Personalize `GetPersonalizedRanking` on a candidate set (top-100 from Personalize). XGBoost re-ranker adds features from the Feature Store (social graph, organizer quality, contextual signals). Deployed via SageMaker Endpoint with pre-computation to DynamoDB.

3. **Semantic search + cold start via OpenSearch k-NN.** Generate event embeddings (Amazon Bedrock Titan Embeddings or SageMaker). Index as k-NN vectors in OpenSearch. New user with no history → k-NN search over user interest tags → personalized feed without collaborative filtering.

4. **Automated retraining pipeline.** SageMaker Pipeline: Rudderstack S3 archive → ProcessingJob → TrainingJob → EvaluationJob (quality gate: NDCG@10 ≥ 0.20) → Model Registry → Personalize import. EventBridge trigger: weekly + data-volume threshold.

---

## 15. Questions to Ask Cat and Ethan During the Walkthrough

You already know the broad strokes. These questions go deeper and signal that you've done your homework. Don't ask things you already know — ask things that refine what you know into a plan.

### On the Outage — Understand the Root Cause
- "When the outage happened during the large event — was the recommendation endpoint part of the failure, or was it something else? I want to understand whether my instinct about Redshift concurrency is right, or if there's a different bottleneck I should be thinking about."
- "After the outage, what changes were made? And what's still on the table?"

### On Redshift and the Serving Path
- "Is the Redshift query for personalization running synchronously in the API path, or is it pre-computed on a schedule and cached somewhere?"
- "What's the rough shape of the heuristics today — is it something like 'top events in the user's city in the last 30 days,' or is it more complex than that? I want to understand what the ML model will need to beat."
- "Are there any business rules layered on top — things like 'always show events from followed organizers' or 'filter sold-out events'? I want to understand what the model needs to preserve."

### On Rudderstack and Event Data
- "What are the current Rudderstack destinations — is it primarily Redshift, or are there others?"
- "Are you currently streaming events anywhere outside of Redshift — S3, Kinesis, anything like that?"
- "What does the event schema look like for clicks and RSVPs today? I'm thinking about what needs to change to get this into Personalize's interaction format."
- "Is there any event data older than Rudderstack — like historical interaction data in MongoDB or Redshift that predates the Rudderstack integration? That history matters for the initial training dataset."

### On MongoDB and Search
- "For the search use case — are you currently on MongoDB Atlas with Atlas Search, or is it native MongoDB text indexes?"
- "What are the search queries that are causing the most concern performance-wise — is it full-text search on event titles, geo-filtered search, or something else?"
- "Is the concern purely about search performance, or is it also about relevance quality — users not finding what they're looking for?"

### On Personalization Goals and Measurement
- "Since personalization is heuristics-based today — is there any measurement of whether it's helping? Any proxy metric like click rate on recommended events versus non-recommended events?"
- "When you imagine what 'great personalization' looks like on Posh, what does it enable for a user that they can't do today? That helps me understand where to focus first."
- "Are there surfaces beyond the discovery feed — like 'similar events' on an event detail page, or email recommendations — that you'd want personalization to power eventually?"

### On Team and Constraints
- "How much of the personalization stack is owned by one person today versus spread across the team? I want to understand how much institutional knowledge I'll be absorbing."
- "Are there infrastructure constraints I should know about — things Rudderstack can't route to, AWS services that aren't available in your account setup, cost guardrails?"
- "What does success look like for this role at 30 days, 90 days, and 6 months?"

---

## 16. How to Present Your Observations Without Being Arrogant

You will see gaps. You will see things that could be better. The interview is a test of both your technical insight *and* your judgment about how to communicate it.

### The Framework: Observe → Hypothesize → Validate → Propose

You know several gaps going in. You cannot walk in and immediately recite them — that reads as dismissive of the work they've done. Frame everything as building on what exists.

**On the Redshift + outage:**

Don't say:
> "Redshift shouldn't be in the serving path — that's why you had the outage."

Say:
> "The outage context was really helpful to hear about. My instinct from looking at the architecture is that the Redshift queries in the recommendation path were probably under contention during that traffic spike — is that roughly what happened? The good news is the fix doesn't require replacing Redshift at all. We pre-compute the same heuristics you have today into DynamoDB on a schedule, and the API reads from there instead. Redshift keeps doing what it's good at."

**On the missing impressions:**

Don't say:
> "You can't measure anything without impression logging — this is a fundamental gap."

Say:
> "One thing I'd want to close early is impression logging — tracking which events are actually shown to each user in the feed, with position. Right now you can see who clicked, but not what they passed on. That's the data that lets you compute click-through rate and eventually train a model that learns from non-engagement, not just engagement. It's a small Rudderstack event to add, but it unlocks a lot."

**On heuristics vs. ML:**

Don't say:
> "Heuristics don't scale. You need ML."

Say:
> "The heuristics approach makes total sense as a starting point — it's fast, interpretable, and you don't need a training dataset to ship. The question is: at what point do the heuristics start to plateau? When the catalog is large enough that city + category isn't enough signal. When you want to differentiate between two users in the same city who like jazz differently. That's when the model starts paying off. My approach would be to keep the heuristics running in parallel as the control group while we build and validate the model alongside it."

### Signal Understanding, Then Ask Before Proposing

You know a lot — but not everything. Before proposing solutions:
- "Before I get into what I'd change about the pipeline — can you tell me more about how the Rudderstack routing is set up today? Are there other destinations beyond Redshift?"
- The answer might reveal: they already have S3 as a destination, or they tried Kinesis and it was too complex. That changes your proposal.

### Tie Everything Back to the Product Outcome

Architecture decisions are means to an end. The end is: users discover events they love and buy tickets. Every architecture proposal should connect to that outcome:
- "The reason I'd prioritize impression logging before anything else isn't because it's best practice — it's because right now there's no way to know whether the current heuristics are actually driving ticket purchases, or whether users are buying tickets in spite of recommendations. That measurement is what lets you justify the ML investment and steer it."

---

## 17. Interview Drill — Scenarios They Might Throw At You

Practice these until the answers come naturally.

---

### Scenario 1: "We're seeing stale recommendations — a user buys a ticket and we still show them that event."

**Root cause:** Purchased events are not being filtered from recommendation results. The model has no way to know a ticket was bought.

**Solution:**
```
Short term: Add a Personalize filter expression.
  FilterArn filter expression:
  EXCLUDE itemId WHERE interactions.event_type = "ticket_purchase"
  
  This tells Personalize to exclude any item the user has a "ticket_purchase" interaction for.
  Apply this filter to all GetRecommendations calls.

Medium term: If using cached recs, purge the cache on purchase. 
  On ticket_purchase event → Lambda → invalidate DynamoDB/Redis cache for that user.
```

---

### Scenario 2: "We have a new event published 10 minutes ago — how quickly does it show up in recommendations?"

**If batch import only:** Up to 24 hours. The event won't enter the Items dataset until the next scheduled import.

**Fix:**
```
1. On event publish (application webhook or DynamoDB Stream)
   → Lambda triggers
   → Upsert the new event to the Personalize Items dataset via dataset import job
      (Personalize supports incremental item imports, not just full re-imports)
   → Or: if using a custom model, add the item to the Feature Store immediately
   
2. For cold start on the new item:
   → Tag it with is_new_event = true
   → Business rule layer: for first 48h, add new events as "spotlight" candidates
      that bypass the model ranking and appear in a "New on Posh" section
   → Track engagement on these new events; use the signal for future training
```

---

### Scenario 3: "Our recommendation quality seems to have gotten worse after last week's retraining."

**Diagnosis framework:**
1. Did the training data change? (New event types, schema changes, missing data)
2. Did the feature distribution shift? (Seasonality — the week before a major holiday has different interaction patterns)
3. Did coverage drop? (Model is only recommending a small subset of the catalog)
4. Is it actually worse, or just different? (Check offline metrics, not just product intuition)

**Mitigation:**
```
1. Rollback immediately: SageMaker Model Registry → reactivate previous version
   (This is why you never overwrite model versions.)

2. Compare training data between this run and last run:
   → Row counts per event_type
   → Distribution of TIMESTAMP values
   → Ratio of new users to returning users

3. Compare offline metrics: NDCG@10, Precision@10, coverage
   If metrics regressed past the quality gate, the gate wasn't strict enough.

4. Add a shadow evaluation: before replacing the production model,
   run both models on the same test set and diff the output.
   If the new model recommends significantly different items for the same users,
   investigate before deploying.
```

---

### Scenario 4: "How would you add our social graph (follows) to improve recommendations?"

**The problem:** Personalize's standard recipes don't natively support "boost events from organizers the user follows."

**Three approaches (choose based on maturity):**

**Approach A: Personalize Filters (quick win)**
```python
# When fetching recs, pass filterValues for followed organizers
response = personalize_runtime.get_recommendations(
    campaignArn=CAMPAIGN_ARN,
    userId=user_id,
    numResults=20,
    filterArn=FOLLOWED_ORGANIZER_BOOST_FILTER_ARN,
    filterValues={
        'FOLLOWED_ORGANIZER_IDS': json.dumps(user_followed_organizer_ids)
    }
)
```
Note: Personalize filters support `IN` expressions with injected values. Works up to 10 item metadata values.

**Approach B: Business Rule Post-Processing (more flexible)**
```python
# Fetch 50 raw recs, re-rank with follow boost in application code
raw_recs = get_personalize_recs(user_id, n=50)
recs_with_scores = enrich_with_metadata(raw_recs)
for rec in recs_with_scores:
    if rec.organizer_id in get_followed_organizers(user_id):
        rec.score *= 1.5
return sorted(recs_with_scores, key=lambda r: r.score)[:20]
```

**Approach C: Custom Ranking Model (highest quality, most work)**
```
Build a SageMaker LambdaRank model:
Features: 
  - Personalize raw score (from GetPersonalizedRanking)
  - is_followed_organizer (bool, from social graph DB)
  - organizer_follower_count (popularity of organizer)
  - user_organizer_purchase_history (have they bought from this organizer before)
  - social_proof (how many of user's followees attended similar events)

Training signal: did the user purchase after seeing this event in this position?

This model re-ranks Personalize's candidate list and naturally learns
the optimal weight for social graph signals from data.
```

---

### Scenario 5: "A new user just signed up. What do they see?"

**Cold start strategy:**

```
Tier 1: Brand new user, no interactions, no profile
  → Show globally popular upcoming events in their city (detected from IP at signup)
  → Onboarding quiz: "What kinds of events do you love?" (3-5 categories)
  → Store quiz answers as synthetic interactions or user metadata
  → Personalize uses these for immediate context

Tier 2: User has 1-5 interactions (first session)
  → PutEvents in real-time; Personalize immediately adjusts
  → aws-user-personalization-v2 starts personalizing after the first interaction
  → Show "Because you viewed X..." explanations to make personalization legible

Tier 3: User has 10+ interactions
  → Full personalization, model has sufficient signal
  → Social graph features kick in (follow recommendations, event recommendations from followees)
```

---

### Scenario 6: "How would you handle LLM integration for event classification or personalization?"

**Classification use case (most immediate value):**
Events are created by organizers who write free-form titles and descriptions. You need structured categories (music, sports, arts, food & drink, fitness, etc.) for filtering and personalization. Fine-tune or prompt an LLM to classify events at creation time.

```
Event created by organizer
  → Lambda trigger
  → Call LLM (Bedrock Claude or SageMaker-hosted model)
     Prompt: "Classify this event into one of [categories] based on title and description.
              Title: {title}. Description: {description}. Respond with JSON."
  → Store category in Items dataset
  → Enrich Personalize item metadata
  → Use as feature in custom ranker
```

**Semantic similarity use case (for cold start):**
Generate embeddings for events and users. Use cosine similarity for retrieval when collaborative filtering has no signal.

```
Event publish → generate text embedding (title + description + organizer bio)
             → store in vector DB (Pinecone, OpenSearch k-NN, or pgvector)

New user with only quiz answers:
  → Generate query embedding from quiz topics
  → k-NN search in vector DB
  → Return top-K semantically similar events
  → Blend with popularity signal for final ranking
```

This is a natural complement to Personalize, not a replacement. Personalize handles warm users with interaction history well. Embeddings handle cold users and semantic matching.

---

### Scenario 7: "We had an outage during a large event last year. How would you prevent that?"

**Diagnosing what happened:**

The most likely failure chain: A high-profile event goes live → traffic spikes → each user feed load triggers a Redshift personalization query → Redshift concurrency limit hit → queries queue, then time out → recommendation endpoint returns 5xx → feed breaks for users trying to buy tickets at the exact moment they're most engaged.

**Confirm your hypothesis by asking:** "Was the recommendation endpoint specifically failing, or was it a broader application issue? And roughly what traffic levels triggered it?"

**The fix (before you even touch ML):**

```
Root cause: Redshift is in the synchronous API serving path.
Fix: Remove it.

Step 1: Pre-compute heuristic recs to DynamoDB
  Background Lambda (runs every 30-60 min, or triggered on large-event flag):
    → Redshift query (same heuristics as today)
    → Write results: DynamoDB { pk: "USER#{user_id}", recs: [eventId1...eventId50], ttl: +2h }
  
  API path change:
    BEFORE: Lambda → Redshift query (~200-500ms, limited concurrency)
    AFTER:  Lambda → DynamoDB GetItem (~3ms, scales to millions of concurrent reads)

Step 2: For large event spikes specifically — pre-warm the cache
  When a high-profile event is published (organizer has > X followers):
    → Trigger immediate DynamoDB refresh for all users who follow that organizer
    → This ensures their feed is fresh before the traffic hits

Step 3: Circuit breaker on the recommendation path
  If DynamoDB read fails for any reason, fall through to:
    → Popularity-based fallback (top 20 events in user's city this week, hardcoded in Lambda)
    → Never let a recommendation failure break the feed render

Step 4: Load test the new architecture before the next large event
  Use Locust or k6 to simulate 5,000 concurrent users loading their feeds.
  DynamoDB should handle this trivially. Verify p99 < 50ms.
```

**Why this matters beyond reliability:**
Pre-computed DynamoDB also unlocks faster iteration on the recommendation algorithm. You can A/B test heuristic v1 vs. v2 by writing to two DynamoDB keys and routing traffic — no Redshift query changes needed.

---

### Scenario 8: "Our users are complaining they can't find events through search. How would you approach this?"

**First, clarify the problem type:** Search quality issues are different from search performance issues.
- **Performance** (searches are slow, timeouts): MongoDB/Atlas Search under resource pressure → OpenSearch
- **Relevance** (searches return wrong results, ranked poorly): needs query tuning, better schema, fuzzy matching
- **Discovery** (can't find events that exist, need synonym/semantic match): needs embedding-based search

**Ask:** "When you say they can't find events — is it that search is slow, or that they search for 'jazz concert' and the right events don't surface?"

**The architecture answer:**

```
Move event search off MongoDB → Amazon OpenSearch Service

Migration plan:
1. Deploy OpenSearch domain (m6g.large.search, 1 node to start — ~$180/month)

2. Dual-write during transition:
   Event created/updated in MongoDB
     → application writes to MongoDB (existing)
     → Lambda also indexes to OpenSearch (new)
   
3. OpenSearch index schema for events:
   {
     "event_id": "keyword",
     "title": "text (analyzed, with synonym expansion)",
     "description": "text (analyzed)",
     "organizer_name": "text + keyword",
     "category": "keyword",
     "tags": ["keyword"],
     "city": "keyword",
     "neighborhood": "keyword",
     "date_start": "date",
     "price_min": "float",
     "is_free": "boolean",
     "title_embedding": "knn_vector (1536 dims)"  ← for semantic search
   }

4. Query pattern for "jazz NYC this weekend":
   {
     "query": {
       "bool": {
         "must": {
           "multi_match": {
             "query": "jazz",
             "fields": ["title^3", "description", "tags^2"],
             "fuzziness": "AUTO"
           }
         },
         "filter": [
           { "term": { "city": "New York" } },
           { "range": { "date_start": { "gte": "now", "lte": "now+3d/d" } } }
         ],
         "should": [
           { "term": { "is_free": false } }  ← slight boost for paid events (more committed organizers)
         ]
       }
     },
     "sort": [
       { "_score": "desc" },
       { "date_start": "asc" }  ← tie-break by soonest
     ]
   }

5. Semantic / fallback search (zero-results recovery):
   If keyword search returns 0 results:
     → Generate query embedding (Bedrock Titan Embeddings)
     → k-NN search against title_embedding field
     → Return semantically similar events
     → "We didn't find exact matches for 'burlesque', but here are similar events..."

6. Traffic cutover:
   Feature flag in the search API: 10% → 50% → 100% of traffic to OpenSearch
   Monitor: result count per query, zero-result rate, click-through on results
   Keep MongoDB text search as fallback for 2 weeks, then remove
```

**The shared infrastructure point:**
> "The OpenSearch cluster you build for search is the same one you'd use for semantic event retrieval in the personalization cold-start problem. k-NN search on event embeddings powers both 'find events like this one' in search and 'here are events matching your interests' for new users. One cluster, two features."

---

### Scenario 9: "Walk us through how you'd think about migrating from heuristics to ML without disrupting what's working."

This is the core scenario. They're nervous about disrupting working heuristics. The answer is a no-big-bang migration with parallel running.

**The principle: never deprecate before you validate.**

```
PHASE 1: Run Personalize in shadow mode (no user impact)
  Deploy Personalize campaign
  For every API call:
    → Serve heuristic result to user (no change)
    → Also call Personalize in the background (async, non-blocking)
    → Log both result sets to DynamoDB/S3
  
  After 2 weeks:
    → Compare: overlap between heuristic and Personalize results
    → If < 30% overlap, the model is learning something different — investigate
    → Check offline metrics: NDCG@10, coverage
    → No user impact, full observability

PHASE 2: A/B test
  Bucket users: 80% control (heuristics), 20% treatment (Personalize)
  Same serving architecture — DynamoDB, pre-computed
  Annotate all events with experiment bucket
  
  Run for 2 weeks minimum. Watch:
    → Primary: ticket purchase rate (treatment vs. control)
    → Secondary: recommendation CTR, event discovery (% of users discovering
                 events from organizers they haven't engaged with before)
    → Guardrail: page load time (DynamoDB latency should be equivalent)

PHASE 3: Graduated rollout
  If treatment wins: 20% → 50% → 100% over 2 weeks
  Keep heuristic pipeline running for 4 weeks post-migration (rollback capability)
  
  If treatment is neutral: investigate features. Add city, follow graph, 
  recency signals that the heuristic uses. Retrain. Re-test.
  
  If treatment loses: the heuristic has signal the model doesn't. Debug.

PHASE 4: Deprecate heuristic
  Only after Personalize has been at 100% for 4 weeks with no regressions.
  Remove the Redshift heuristic query job.
  Redshift now serves analytics only.
```

The key message: you're not replacing the heuristic with a bet. You're running both in parallel, measuring the difference, and only cutting over when you have evidence. The heuristic stays live as a rollback until you're confident.

---

*This guide covers the architecture, AWS stack, pipeline design, and critical thinking frameworks for the Posh data architecture interview. The interview is a conversation, not a test — listen more than you talk, ask before you prescribe, and anchor every technical proposal to making it easier for people to discover events they love.*
