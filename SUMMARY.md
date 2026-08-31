# Table of Contents

Complete index of all chapters. Every chapter contains a 🧒 **ELI5** section.

[README](README.md) — how to use this book, study plans, and the one-page mental model.

## Part 1 — Introduction

### Basics

- [1.1.1 — What Is a System Design Interview?](01-introduction/01-basics/01-what-is-a-system-design-interview.md)
- [1.1.2 — The Interview Template](01-introduction/01-basics/02-interview-template.md)
- [1.1.3 — Core Challenges in Web-scale System Design (and How to Tackle Them)](01-introduction/01-basics/03-core-challenges.md)
- [1.1.4 — How to Scale a System](01-introduction/01-basics/04-how-to-scale-a-system.md)
- [1.1.5 — Study Guide](01-introduction/01-basics/05-study-guide.md)

### Api Design

- [1.2.1 — API Design Intro](01-introduction/02-api-design/01-api-design-intro.md)
- [1.2.2 — API Design Example](01-introduction/02-api-design/02-api-design-example.md)
- [1.2.3 — API Design: Pagination](01-introduction/02-api-design/03-pagination.md)
- [1.2.4 — API Authentication](01-introduction/02-api-design/04-api-authentication.md)
- [1.2.5 — API Authorization](01-introduction/02-api-design/05-api-authorization.md)
- [1.2.6 — API Gateway](01-introduction/02-api-design/06-api-gateway.md)

### Non Functional Requirements

- [1.3.1 — Non-functional Requirements](01-introduction/03-non-functional-requirements/01-non-functional-requirements.md)
- [1.3.2 — System Design Components](01-introduction/03-non-functional-requirements/02-system-design-components.md)
- [1.3.3 — High Availability](01-introduction/03-non-functional-requirements/03-high-availability.md)
- [1.3.4 — How to Achieve High Availability](01-introduction/03-non-functional-requirements/04-how-to-achieve-high-availability.md)
- [1.3.5 — Tech Stacks to Achieve High Availability](01-introduction/03-non-functional-requirements/05-tech-stacks-for-high-availability.md)
- [1.3.6 — Latency](01-introduction/03-non-functional-requirements/06-latency.md)
- [1.3.7 — Throughput](01-introduction/03-non-functional-requirements/07-throughput.md)

### Resource Estimation

- [1.4.1 — Back-of-the-envelope Resource Estimation](01-introduction/04-resource-estimation/01-back-of-envelope-estimation.md)
- [1.4.2 — QPS and System Design](01-introduction/04-resource-estimation/02-qps-and-system-design.md)
- [1.4.3 — Resource Estimation: Real World Examples](01-introduction/04-resource-estimation/03-real-world-examples.md)

### Microservices

- [1.5.1 — Microservices and Monolithic Architecture](01-introduction/05-microservices/01-microservices-vs-monolithic.md)

## Part 2 — Microservices & Data Flow

### Synchronous Communication

- [2.1.1 — Microservice Communication](02-microservices-and-dataflow/01-synchronous-communication/01-microservice-communication.md)
- [2.1.2 — Synchronous Communication](02-microservices-and-dataflow/01-synchronous-communication/02-synchronous-communication.md)
- [2.1.3 — Implementing Synchronous Communication](02-microservices-and-dataflow/01-synchronous-communication/03-implementation.md)
- [2.1.4 — Failure Handling in Synchronous Communication](02-microservices-and-dataflow/01-synchronous-communication/04-failure-handling.md)
- [2.1.5 — Timeout](02-microservices-and-dataflow/01-synchronous-communication/05-timeout.md)
- [2.1.6 — Retries](02-microservices-and-dataflow/01-synchronous-communication/06-retries.md)
- [2.1.7 — Circuit Breaker](02-microservices-and-dataflow/01-synchronous-communication/07-circuit-breaker.md)
- [2.1.8 — Fallbacks](02-microservices-and-dataflow/01-synchronous-communication/08-fallbacks.md)
- [2.1.9 — Service Discovery](02-microservices-and-dataflow/01-synchronous-communication/09-service-discovery.md)

### Asynchronous Communication

- [2.2.1 — Asynchronous Communication Through Messaging](02-microservices-and-dataflow/02-asynchronous-communication/01-async-communication.md)
- [2.2.2 — Message Queues in System Design](02-microservices-and-dataflow/02-asynchronous-communication/02-message-queues.md)
- [2.2.3 — Message Queue Use Cases and Patterns](02-microservices-and-dataflow/02-asynchronous-communication/03-message-queue-patterns.md)
- [2.2.4 — Redis Queue Tutorial](02-microservices-and-dataflow/02-asynchronous-communication/04-redis-queue-tutorial.md)
- [2.2.5 — Log-based Message Queues](02-microservices-and-dataflow/02-asynchronous-communication/05-log-based-message-queues.md)
- [2.2.6 — Introduction to Kafka](02-microservices-and-dataflow/02-asynchronous-communication/06-introduction-to-kafka.md)
- [2.2.7 — Kafka Exercise](02-microservices-and-dataflow/02-asynchronous-communication/07-kafka-exercise.md)

## Part 3 — Scaling Services

### Horizontal Scaling

- [3.1.1 — Evolution of Computing Environments](03-scaling-services/01-horizontal-scaling/01-evolution-of-computing-environments.md)
- [3.1.2 — Evolution of a Web App: Stateless vs Stateful](03-scaling-services/01-horizontal-scaling/02-stateless-vs-stateful.md)
- [3.1.3 — Evolution of a Web App: Single Server to Scaled System](03-scaling-services/01-horizontal-scaling/03-single-to-scaling.md)
- [3.1.4 — Load Balancer](03-scaling-services/01-horizontal-scaling/04-load-balancer.md)
- [3.1.5 — Load Balancing Codelab](03-scaling-services/01-horizontal-scaling/05-load-balancing-codelab.md)
- [3.1.6 — Auto Scaling](03-scaling-services/01-horizontal-scaling/06-auto-scaling.md)

### Read Write Separation

- [3.2.1 — Read-Write Separation](03-scaling-services/02-read-write-separation/01-read-write-separation.md)
- [3.2.2 — CQRS (Command Query Responsibility Segregation)](03-scaling-services/02-read-write-separation/02-cqrs.md)

### Caching

- [3.3.1 — Caching: The Mental Model](03-scaling-services/03-caching/01-caching-mental-model.md)
- [3.3.2 — The Multi-Layer Defense](03-scaling-services/03-caching/02-caching-tiers.md)
- [3.3.3 — The Edge: CDN vs Application Cache](03-scaling-services/03-caching/03-cdn-vs-app-cache.md)
- [3.3.4 — Cache Key Design](03-scaling-services/03-caching/04-cache-key-design.md)
- [3.3.5 — Read Patterns: Fetching Data](03-scaling-services/03-caching/05-read-patterns.md)
- [3.3.6 — 🧪 Lab: The Read Drill](03-scaling-services/03-caching/06-lab-read-drill.md)
- [3.3.7 — Write Patterns: Mutating Data](03-scaling-services/03-caching/07-write-patterns.md)
- [3.3.8 — The Consistency Problem](03-scaling-services/03-caching/08-consistency.md)
- [3.3.9 — 🧪 Lab: The Write Drill](03-scaling-services/03-caching/09-lab-write-drill.md)
- [3.3.10 — Invalidation & Freshness](03-scaling-services/03-caching/10-invalidation.md)
- [3.3.11 — Eviction & Sizing](03-scaling-services/03-caching/11-eviction-and-sizing.md)
- [3.3.12 — Distributed Caching](03-scaling-services/03-caching/12-distributed-caching.md)
- [3.3.13 — Cache High Availability](03-scaling-services/03-caching/13-cache-high-availability.md)
- [3.3.14 — Cold Start](03-scaling-services/03-caching/14-cold-start.md)
- [3.3.15 — Failure Modes](03-scaling-services/03-caching/15-failure-modes.md)
- [3.3.16 — 🧪 Lab: The Disaster Drill](03-scaling-services/03-caching/16-lab-disaster-drill.md)
- [3.3.17 — Security & Observability](03-scaling-services/03-caching/17-security-and-observability.md)
- [3.3.18 — Interview Walkthrough](03-scaling-services/03-caching/18-interview-walkthrough.md)

### Dataflow

- [3.4.1 — Dataflow Overview](03-scaling-services/04-dataflow/01-dataflow-overview.md)
- [3.4.2 — Push vs Pull](03-scaling-services/04-dataflow/02-push-vs-pull.md)
- [3.4.3 — Push vs Pull in the Twitter Timeline](03-scaling-services/04-dataflow/03-push-vs-pull-twitter-timeline.md)

## Part 4 — Data Storage

### Data Structures Behind Databases

- [4.1.1 — Data Structures Behind Databases](04-data-storage/01-data-structures-behind-databases/01-data-structures-behind-databases.md)
- [4.1.2 — B-tree](04-data-storage/01-data-structures-behind-databases/02-b-tree.md)
- [4.1.3 — SSTable](04-data-storage/01-data-structures-behind-databases/03-sstable.md)
- [4.1.4 — LSM Tree](04-data-storage/01-data-structures-behind-databases/04-lsm-tree.md)

### Storage

- [4.2.1 — Introduction to Storage](04-data-storage/02-storage/01-introduction-to-storage.md)
- [4.2.2 — SQL Database](04-data-storage/02-storage/02-sql-database.md)
- [4.2.3 — Introduction to NoSQL Databases](04-data-storage/02-storage/03-nosql-database.md)
- [4.2.4 — Key-value Database](04-data-storage/02-storage/04-key-value-database.md)
- [4.2.5 — Document Database](04-data-storage/02-storage/05-document-database.md)
- [4.2.6 — Full-text Search Database](04-data-storage/02-storage/06-full-text-search-database.md)
- [4.2.7 — OLTP or OLAP?](04-data-storage/02-storage/07-oltp-vs-olap.md)
- [4.2.8 — Blob / Object Storage](04-data-storage/02-storage/08-blob-object-storage.md)
- [4.2.9 — SQL vs NoSQL](04-data-storage/02-storage/09-sql-vs-nosql.md)

## Part 5 — Scaling Data Storage

### Data Replication

- [5.1.1 — How to Scale Databases](05-scaling-data-storage/01-data-replication/01-database-scaling.md)
- [5.1.2 — Database Replication: Fundamentals and Algorithms](05-scaling-data-storage/01-data-replication/02-database-replication.md)
- [5.1.3 — Implementing Database Replication: Practical Guide and Failover Strategies](05-scaling-data-storage/01-data-replication/03-implementing-replication.md)
- [5.1.4 — 🧪 Data Replication Tutorial](05-scaling-data-storage/01-data-replication/04-replication-codelab.md)
- [5.1.5 — Change Data Capture](05-scaling-data-storage/01-data-replication/05-change-data-capture.md)

### Data Partitioning

- [5.2.1 — Database Partitioning](05-scaling-data-storage/02-data-partitioning/01-database-partitioning.md)
- [5.2.2 — Advanced Database Partitioning Techniques and Key Selection](05-scaling-data-storage/02-data-partitioning/02-advanced-partitioning.md)
- [5.2.3 — Consistent Hashing](05-scaling-data-storage/02-data-partitioning/03-consistent-hashing.md)
- [5.2.4 — 🧪 Database Partition Tutorial](05-scaling-data-storage/02-data-partitioning/04-partition-codelab.md)

## Part 6 — Big Data

### Overview

- [6.1.1 — Batch & Stream Processing: Overview](06-big-data/01-overview/01-batch-and-stream-overview.md)

### Batch Processing

- [6.2.1 — Unix Pipelines](06-big-data/02-batch-processing/01-unix-pipelines.md)
- [6.2.2 — The MapReduce Model](06-big-data/02-batch-processing/02-mapreduce-model.md)
- [6.2.3 — Distributed File Systems](06-big-data/02-batch-processing/03-distributed-file-systems.md)
- [6.2.4 — Modern Batch: Spark](06-big-data/02-batch-processing/04-modern-batch-spark.md)

### Stream Processing

- [6.3.1 — What Is Stream Processing?](06-big-data/03-stream-processing/01-stream-processing-intro.md)
- [6.3.2 — Windowing Patterns](06-big-data/03-stream-processing/02-windowing-patterns.md)
- [6.3.3 — Event Time & Watermarks](06-big-data/03-stream-processing/03-event-time-watermarks.md)
- [6.3.4 — Delivery Guarantees](06-big-data/03-stream-processing/04-delivery-guarantees.md)
- [6.3.5 — Modern Stream: Flink & Kafka Streams](06-big-data/03-stream-processing/05-modern-stream-flink.md)

### Hybrid Architectures

- [6.4.1 — Lambda Architecture](06-big-data/04-hybrid-architectures/01-lambda-architecture.md)
- [6.4.2 — Kappa Architecture](06-big-data/04-hybrid-architectures/02-kappa-architecture.md)
- [6.4.3 — Unified Processing](06-big-data/04-hybrid-architectures/03-batch-stream-unification.md)

### Pipeline Operations

- [6.5.1 — ETL vs ELT](06-big-data/05-pipeline-operations/01-etl-vs-elt.md)
- [6.5.2 — Backfill & Reprocessing](06-big-data/05-pipeline-operations/02-backfill-reprocessing.md)
- [6.5.3 — Error Handling](06-big-data/05-pipeline-operations/03-pipeline-error-handling.md)

### Realtime And Analytics

- [6.6.1 — Materialized Views](06-big-data/06-realtime-and-analytics/01-materialized-views-streaming.md)
- [6.6.2 — Time-Series Patterns](06-big-data/06-realtime-and-analytics/02-time-series-patterns.md)
- [6.6.3 — Analytics Architecture](06-big-data/06-realtime-and-analytics/03-streaming-analytics-architecture.md)

## Part 7 — Patterns & Templates

### Patterns

- [7.1.1 — Database Optimization Techniques](07-patterns-and-templates/01-patterns/01-database-optimization.md)
- [7.1.2 — Cache-First Pattern](07-patterns-and-templates/01-patterns/02-cache-always.md)
- [7.1.3 — Pre-Computing Pattern](07-patterns-and-templates/01-patterns/03-pre-computing-pattern.md)
- [7.1.4 — Database Per Microservice](07-patterns-and-templates/01-patterns/04-database-per-service.md)
- [7.1.5 — Multi-System Data Sync](07-patterns-and-templates/01-patterns/05-multi-system-data-sync.md)
- [7.1.6 — Unique ID Generators](07-patterns-and-templates/01-patterns/06-unique-id-generators.md)
- [7.1.7 — Rate Limiting Patterns](07-patterns-and-templates/01-patterns/07-rate-limiting-patterns.md)
- [7.1.8 — The Two-Stage Processing Pattern](07-patterns-and-templates/01-patterns/08-two-stage-processing.md)
- [7.1.9 — Fan-Out / Fan-In Pattern](07-patterns-and-templates/01-patterns/09-fan-out-fan-in.md)
- [7.1.10 — Saga Pattern](07-patterns-and-templates/01-patterns/10-saga-pattern.md)

### Template

- [7.2.1 — The Design Template](07-patterns-and-templates/02-template/01-design-template.md)
- [7.2.2 — Template Application: A Social Media Comment System](07-patterns-and-templates/02-template/02-template-application-comments.md)

---

**120 chapters.** Hands-on labs: Redis queues, Kafka, load balancing, caching read/write/disaster drills, replication, and partitioning.
