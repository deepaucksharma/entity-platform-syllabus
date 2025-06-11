# 📚 Kafka Entity Platform: Introduction and Fundamentals

<div align="center">

![Kafka](https://img.shields.io/badge/Apache%20Kafka-231F20?style=for-the-badge&logo=apache-kafka&logoColor=white)
![New Relic](https://img.shields.io/badge/New%20Relic-008C99?style=for-the-badge&logo=new-relic&logoColor=white)
![Documentation](https://img.shields.io/badge/Part_1_of_4-Fundamentals-blue?style=for-the-badge)

**Transform your Kafka monitoring from raw metrics to intelligent entity-based observability**

</div>

---

## 📑 Document Series Navigation

<table>
<tr>
<td width="25%" align="center" bgcolor="#e3f2fd">

### 📘 Part 1 (This Doc)
**Fundamentals**
- Introduction
- Platform Basics
- Core Concepts
- Entity Hierarchy
- Lifecycle & Flow

</td>
<td width="25%" align="center">

### [📗 Part 2](KAFKA_ENTITY_IMPLEMENTATION.md)
**Implementation**
- Synthesis Engine
- Golden Metrics
- Relationships
- Providers
- Dashboards

</td>
<td width="25%" align="center">

### [📙 Part 3](KAFKA_ENTITY_OPERATIONS.md)
**Operations**
- Configuration
- Testing
- Excellence
- Troubleshooting
- Performance

</td>
<td width="25%" align="center">

### [📕 Part 4](KAFKA_ENTITY_ADVANCED.md)
**Advanced**
- Best Practices
- Integration
- Security
- Future
- Reference

</td>
</tr>
</table>

---

## 📖 Table of Contents

1. [Introduction: Understanding the Big Picture](#introduction)
2. [Entity Platform Fundamentals](#entity-platform-fundamentals)
3. [Core Concepts and Architecture](#core-concepts)
4. [The Kafka Entity Hierarchy](#kafka-hierarchy)
5. [Entity Lifecycle and Data Flow](#entity-lifecycle)

---

## 1. Introduction: Understanding the Big Picture {#introduction}

### What is the Entity Platform?

The New Relic Entity Platform is a sophisticated distributed system that transforms raw telemetry data into intelligent, actionable insights about your infrastructure and applications. Think of it as the "brain" that understands what each component in your system is, how it relates to other components, and what its current state means for your business.

### Why Kafka Entity Definitions Matter

Apache Kafka has become the backbone of modern event-driven architectures. However, monitoring Kafka effectively requires understanding its complex hierarchy of clusters, brokers, topics, partitions, and consumer groups. This PR introduces a comprehensive framework that:

- **Transforms Raw Metrics** into intelligent entities with lifecycle management
- **Maps Relationships** between Kafka components and your applications automatically
- **Provides Unified Monitoring** across different Kafka providers (self-managed, AWS MSK, Confluent Cloud)
- **Enables Smart Alerting** based on Kafka-specific health calculations

### The Business Impact

<table>
<tr>
<td width="50%">

#### 🚫 Before this implementation:

```
- Fragmented monitoring across different Kafka deployments
- Manual correlation between Kafka issues and application impact
- Provider-specific monitoring tools and dashboards
- No unified view of Kafka infrastructure health
```

</td>
<td width="50%">

#### ✅ After this implementation:

```
- Single pane of glass for all Kafka infrastructure
- Automatic impact analysis when Kafka issues occur
- Provider-agnostic monitoring and alerting
- Intelligent health scoring based on Kafka best practices
```

</td>
</tr>
</table>

### How to Use This Guide

This guide is structured to serve multiple audiences:

<table>
<tr>
<th width="25%">👩‍💻 Audience</th>
<th width="35%">📚 Focus Areas</th>
<th width="40%">🎯 Key Sections</th>
</tr>
<tr>
<td><b>Beginners</b></td>
<td>Foundational understanding</td>
<td>Start with sections 1-4 in this document</td>
</tr>
<tr>
<td><b>Developers</b></td>
<td>Implementation details</td>
<td>Focus on [Part 2](KAFKA_ENTITY_IMPLEMENTATION.md) for synthesis and metrics</td>
</tr>
<tr>
<td><b>Operators</b></td>
<td>Operational aspects</td>
<td>[Part 3](KAFKA_ENTITY_OPERATIONS.md) covers testing and troubleshooting</td>
</tr>
<tr>
<td><b>Architects</b></td>
<td>Design patterns and future</td>
<td>[Part 4](KAFKA_ENTITY_ADVANCED.md) discusses patterns and roadmap</td>
</tr>
</table>

Each section builds upon previous concepts while remaining self-contained enough to serve as a reference.

---

## 2. Entity Platform Fundamentals {#entity-platform-fundamentals}

### The Entity Concept

An **entity** in New Relic represents a monitored component with:

```yaml
Entity:
  identity:
    - guid: Globally unique identifier
    - name: Human-readable name
    - type: Component classification
  properties:
    - tags: Key-value metadata
    - goldenTags: Most important searchable attributes
  telemetry:
    - metrics: Performance measurements
    - events: State changes and activities
  relationships:
    - contains: Hierarchical ownership
    - hosts: Physical hosting relationships
    - produces_to: Data flow relationships
    - consumes_from: Data consumption relationships
```

### The Platform Architecture

The Entity Platform consists of several layers working in concert:

```
┌─────────────────────────────────────────────────────────┐
│                    External Systems                      │
│        (Kafka Integrations, APM, Infrastructure)        │
└─────────────────────────────────────────────────────────┘
                            │
                            v
┌─────────────────────────────────────────────────────────┐
│                   Ingestion Layer                        │
│    - Schema Validation                                   │
│    - Rate Limiting                                       │
│    - Initial Routing                                     │
└─────────────────────────────────────────────────────────┘
                            │
                            v
┌─────────────────────────────────────────────────────────┐
│              Stream Processing Layer                     │
│    - Deduplication                                       │
│    - Merging                                            │
│    - Synthesis                                          │
│    - Enrichment                                         │
└─────────────────────────────────────────────────────────┘
                            │
                            v
┌─────────────────────────────────────────────────────────┐
│               Storage & Indexing Layer                   │
│    - Elasticsearch (Primary Storage)                     │
│    - PostgreSQL (Configuration)                          │
│    - Redis (Caching)                                     │
└─────────────────────────────────────────────────────────┘
                            │
                            v
┌─────────────────────────────────────────────────────────┐
│                    API Layer                             │
│    - GraphQL (Flexible Queries)                          │
│    - REST (CRUD Operations)                              │
│    - WebSocket (Real-time Updates)                       │
└─────────────────────────────────────────────────────────┘
```

### Event-Driven Architecture

The platform uses Apache Kafka as its messaging backbone:

<div style="background-color: #e3f2fd; border-radius: 8px; padding: 20px; margin: 20px 0;">

**Kafka Topics Used:**
- `entity-raw`: Initial entity data
- `entity-merged`: Consolidated entities
- `entity-deduplicated`: Cleaned entity stream
- `entity-definitions-v3`: Configuration updates
- `relationship-proposed`: Discovered relationships
- `relationship-validated`: Confirmed relationships

</div>

### Key Design Principles

<table>
<tr>
<th width="30%">🎯 Principle</th>
<th width="70%">📝 Description</th>
</tr>
<tr>
<td><b>Event Sourcing</b></td>
<td>All state changes are captured as immutable events</td>
</tr>
<tr>
<td><b>CQRS</b></td>
<td>Separate paths for writes (Kafka) and reads (Elasticsearch)</td>
</tr>
<tr>
<td><b>Eventually Consistent</b></td>
<td>Embraces distributed system realities</td>
</tr>
<tr>
<td><b>Idempotent Operations</b></td>
<td>Safe to retry without side effects</td>
</tr>
<tr>
<td><b>Schema Evolution</b></td>
<td>Backward-compatible changes supported</td>
</tr>
</table>

---

## 3. Core Concepts and Architecture {#core-concepts}

### Domains and Types

Every entity belongs to a hierarchical classification system:

```yaml
Domain: INFRA
├── Type: HOST
├── Type: CONTAINER  
├── Type: MESSAGE_QUEUE_CLUSTER
│   └── Subtypes: Kafka, RabbitMQ, SQS, etc.
├── Type: MESSAGE_QUEUE_BROKER
├── Type: MESSAGE_QUEUE_TOPIC
└── Type: MESSAGE_QUEUE_PARTITION
```

<table>
<tr>
<th width="30%">🏷️ Classification</th>
<th width="70%">📝 Description</th>
</tr>
<tr>
<td><b>Domain</b></td>
<td>
Broad category representing the area of monitoring:

- `INFRA`: Infrastructure components
- `APM`: Application services
- `BROWSER`: Frontend applications
- `MOBILE`: Mobile applications
- `SYNTH`: Synthetic monitors
</td>
</tr>
<tr>
<td><b>Type</b></td>
<td>
Specific kind of entity within a domain. For Kafka, we use the `MESSAGE_QUEUE_*` prefix to support future message queue types.
</td>
</tr>
</table>

### Entity Identification

Each entity has a unique GUID (Globally Unique Identifier) generated deterministically:

<div style="background-color: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0;">

```
GUID = base64(hash(accountId + domain + type + identifier))

Example:
- accountId: 12345678
- domain: INFRA
- type: MESSAGE_QUEUE_CLUSTER
- identifier: "prod-kafka-cluster"
- GUID: "MTIzNDU2NzhJTkZSQU1FU1NBR0VfUVVFVUVfQ0xVU1RFUnByb2Qta2Fma2E"
```

**This deterministic approach ensures:**
- Same entity always gets same GUID
- No database lookups needed
- Works across distributed systems
- Supports entity merging

</div>

### Golden Tags

Golden tags are the primary attributes used for searching and filtering:

```yaml
goldenTags:
  - kafka.cluster.name      # Human-friendly cluster name
  - kafka.cluster.id        # Unique cluster identifier
  - kafka.cluster.arn       # AWS ARN for MSK clusters
  - cloud.provider          # aws, gcp, azure, self-managed
  - cloud.region           # Geographic region
  - provider               # Provider type constant
  - integration.type       # How data is collected
```

<div style="background-color: #e8f5e9; border-radius: 8px; padding: 15px; margin: 20px 0;">

**Golden tags appear in:**
- ✓ Entity search interfaces
- ✓ Filter dropdowns
- ✓ Entity lists
- ✓ Relationship displays

</div>

### Synthesis Rules

Synthesis rules are the "recipes" that create entities from raw telemetry:

```yaml
synthesis:
  rules:
    - identifier: clusterName           # Unique ID within account
      name: clusterName                 # Display name
      encodeIdentifierInGUID: true      # Include in GUID generation
      conditions:                       # When to apply this rule
        - attribute: eventType
          value: KafkaClusterSample
        - attribute: clusterName
          present: true
      tags:                            # Extract these attributes
        clusterName:
          entityTagName: kafka.cluster.name
          ttl: P30D                    # Tag expires after 30 days
```

### Tag TTL (Time To Live)

Tags can have expiration times for dynamic attributes:

<table>
<tr>
<th width="30%">⏱️ TTL Type</th>
<th width="35%">📝 Example</th>
<th width="35%">🎯 Use Case</th>
</tr>
<tr>
<td><b>Short (Minutes)</b></td>
<td>

```yaml
consumer.lag.sum:
  ttl: P5M  # 5 minutes
```

</td>
<td>Rapidly changing metrics</td>
</tr>
<tr>
<td><b>Medium (Days)</b></td>
<td>

```yaml
kafka.version:
  ttl: P30D # 30 days
```

</td>
<td>Configuration that rarely changes</td>
</tr>
<tr>
<td><b>Permanent</b></td>
<td>

```yaml
provider:
  # No TTL
```

</td>
<td>Static attributes</td>
</tr>
</table>

### Configuration Settings

Entity behavior is controlled through configuration:

```yaml
configuration:
  alertable: true                    # Can create alerts on this entity
  entityExpirationTime: EIGHT_DAYS   # Auto-cleanup after inactivity
  isContainer: true                  # Contains other entities
```

<table>
<tr>
<th width="30%">⏰ Expiration Time</th>
<th width="70%">🎯 Use Case</th>
</tr>
<tr>
<td><code>FOUR_HOURS</code></td>
<td>High-cardinality entities (partitions)</td>
</tr>
<tr>
<td><code>EIGHT_DAYS</code></td>
<td>Standard infrastructure</td>
</tr>
<tr>
<td><code>THIRTY_DAYS</code></td>
<td>Long-lived entities (consumer groups)</td>
</tr>
<tr>
<td><code>NEVER</code></td>
<td>Permanent entities</td>
</tr>
</table>

---

## 4. The Kafka Entity Hierarchy {#kafka-hierarchy}

### Visual Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  MESSAGE_QUEUE_CLUSTER                       │
│  Properties: version, controller count, health status        │
│  Metrics: throughput, partition count, broker count          │
├─────────────────────────────────────────────────────────────┤
│                        Contains                              │
├──────────────┬────────────────────┬─────────────────────────┤
│              │                    │                         │
▼              ▼                    ▼                         ▼
BROKER         TOPIC               CONSUMER_GROUP           PRODUCER
│              │                    │
│              Contains             Contains
│              │                    │
│              ▼                    ▼
│              PARTITION            CONSUMER
│              │
│              Hosted by
└──────────────┘
```

### Detailed Entity Specifications

#### 🏢 MESSAGE_QUEUE_CLUSTER

**Purpose**: Represents the entire Kafka deployment

<table>
<tr>
<th width="30%">🔍 Aspect</th>
<th width="70%">📝 Details</th>
</tr>
<tr>
<td><b>Identifiers</b></td>
<td>

- `clusterName` (self-managed)
- `kafka.cluster.arn` (AWS MSK)
- `confluent.kafka.cluster.id` (Confluent)

</td>
</tr>
<tr>
<td><b>Health Indicators</b></td>
<td>

- `activeControllerCount` (must be 1)
- `offlinePartitionsCount` (must be 0)
- `underReplicatedPartitions` (should be 0)

</td>
</tr>
<tr>
<td><b>Performance Metrics</b></td>
<td>

- `throughputInBytesPerSec`
- `throughputOutBytesPerSec`
- `totalBrokers`
- `totalTopics`

</td>
</tr>
<tr>
<td><b>Provider-Specific</b></td>
<td>

- `clusterLoadPercent` (Confluent)
- `hotPartitionCount` (Confluent)

</td>
</tr>
</table>

**Health Calculation**:
```sql
CASE 
  WHEN activeControllerCount = 1 AND offlinePartitionsCount = 0 
    THEN 'Healthy'
  WHEN activeControllerCount != 1 OR offlinePartitionsCount > 0 
    THEN 'Critical'
  ELSE 'Unknown'
END
```

#### 🖥️ MESSAGE_QUEUE_BROKER

**Purpose**: Individual Kafka server within a cluster

<table>
<tr>
<th width="30%">🔍 Aspect</th>
<th width="70%">📝 Details</th>
</tr>
<tr>
<td><b>Identifiers</b></td>
<td>

- `broker.id`
- `hostname`

</td>
</tr>
<tr>
<td><b>Resource Metrics</b></td>
<td>

- `cpuPercent`
- `memoryUsedPercent`
- `diskUsedPercent`
- `networkUtilization`

</td>
</tr>
<tr>
<td><b>Performance Indicators</b></td>
<td>

- `requestHandlerAvgIdlePercent` (>70% is healthy)
- `networkProcessorAvgIdlePercent` (>70% is healthy)
- `underReplicatedPartitions` (per broker)

</td>
</tr>
<tr>
<td><b>Workload Metrics</b></td>
<td>

- `partitionCount`
- `leaderCount`
- `bytesInPerSec`
- `bytesOutPerSec`

</td>
</tr>
</table>

**Performance Thresholds**:
- CPU < 80%: Healthy
- Disk < 85%: Healthy
- Request Handler Idle > 70%: Healthy
- Network Processor Idle > 70%: Healthy

#### 📬 MESSAGE_QUEUE_TOPIC

**Purpose**: Logical message channel

<table>
<tr>
<th width="30%">🔍 Aspect</th>
<th width="70%">📝 Details</th>
</tr>
<tr>
<td><b>Configuration</b></td>
<td>

- `partitions` (partition count)
- `replicationFactor`
- `retentionMs`
- `cleanupPolicy`

</td>
</tr>
<tr>
<td><b>Activity Metrics</b></td>
<td>

- `messagesInPerSec`
- `messagesOutPerSec`
- `bytesInPerSec`
- `bytesOutPerSec`

</td>
</tr>
<tr>
<td><b>Consumer Health</b></td>
<td>

- `maxConsumerLag`
- `consumerGroupCount`
- `activeProducers`

</td>
</tr>
<tr>
<td><b>Performance</b></td>
<td>

- `fetchRequestRate`
- `produceRequestRate`

</td>
</tr>
</table>

#### 📁 MESSAGE_QUEUE_PARTITION

**Purpose**: Physical storage unit within a topic

<div style="background-color: #fff3e0; border-radius: 8px; padding: 15px; margin: 20px 0;">

⚠️ **Special Considerations:**
- High cardinality entity
- 4-hour TTL (vs 8 days for others)
- Not alertable

</div>

<table>
<tr>
<th width="30%">🔍 Aspect</th>
<th width="70%">📝 Details</th>
</tr>
<tr>
<td><b>Identity</b></td>
<td>

- `partition.id`
- `topic.name`
- `leader.broker.id`

</td>
</tr>
<tr>
<td><b>State Information</b></td>
<td>

- `inSyncReplicas`
- `replicas`
- `highWatermark`
- `logEndOffset`
- `logStartOffset`
- `size`

</td>
</tr>
</table>

#### 👥 MESSAGE_QUEUE_CONSUMER_GROUP

**Purpose**: Coordinated group of consumers

<table>
<tr>
<th width="30%">🔍 Aspect</th>
<th width="70%">📝 Details</th>
</tr>
<tr>
<td><b>Lag Metrics</b></td>
<td>

- `totalLag` (sum across all partitions)
- `maxLag` (highest single partition)
- `lagTrend` (increasing/decreasing)

</td>
</tr>
<tr>
<td><b>Stability Metrics</b></td>
<td>

- `memberCount`
- `rebalanceRate`
- `state` (active/rebalancing/dead)

</td>
</tr>
<tr>
<td><b>Performance</b></td>
<td>

- `messagesConsumedPerSec`
- `bytesConsumedPerSec`
- `assignedPartitions`

</td>
</tr>
</table>

**Lag Health Scoring**:
- Lag < 1000: Healthy
- Lag < 10000: Warning  
- Lag >= 10000: Critical

#### 📤📥 MESSAGE_QUEUE_PRODUCER / CONSUMER

**Purpose**: Application-level entities

<div style="background-color: #e8f5e9; border-radius: 8px; padding: 15px; margin: 20px 0;">

**Sources:**
- Kafka integration metrics
- APM distributed tracing
- OpenTelemetry spans

**Enables:**
- Application impact analysis
- End-to-end latency tracking
- Service dependency mapping

</div>

---

## 5. Entity Lifecycle and Data Flow {#entity-lifecycle}

### The Journey from Telemetry to Entity

```
Step 1: Data Collection
├── Kafka Integration polls JMX metrics
├── CloudWatch collects AWS MSK metrics
├── Confluent API provides cloud metrics
└── APM agents capture application spans

Step 2: Ingestion
├── Data arrives at entity-ingest service
├── Schema validation ensures data quality
├── Events published to entity-raw topic
└── Initial deduplication window opens

Step 3: Stream Processing
├── entity-merger consolidates updates
├── entity-deduplicator removes duplicates
├── entity-synthesis-engine applies rules
└── Enrichment adds calculated fields

Step 4: Entity Creation
├── GUID generated from attributes
├── Entity document created/updated
├── Tags applied with TTLs
└── Relationships discovered

Step 5: Storage & Indexing
├── core-entity-indexer receives entity
├── Shard assignment determined
├── Bulk indexing to Elasticsearch
└── Cache invalidation triggered

Step 6: API Availability
├── Entity available via GraphQL/REST
├── Appears in UI entity lists
├── Relationships visible in service maps
└── Metrics queryable via NRQL
```

### Deduplication Strategy

The platform implements sophisticated deduplication:

<table>
<tr>
<th width="30%">🕐 Window Type</th>
<th width="35%">⏱️ Duration</th>
<th width="35%">🎯 Purpose</th>
</tr>
<tr>
<td><b>Short Window</b></td>
<td>1 minute</td>
<td>Catch rapid duplicates</td>
</tr>
<tr>
<td><b>Medium Window</b></td>
<td>5 minutes</td>
<td>Handle delays</td>
</tr>
<tr>
<td><b>Long Window</b></td>
<td>1 hour</td>
<td>Cross-region sync</td>
</tr>
</table>

**Deduplication Keys:**
- Entity GUID
- Timestamp
- Mutation type (create/update/delete)

### State Management

Entities maintain state through event accumulation:

<div style="background-color: #f5f5f5; border-radius: 8px; padding: 20px; margin: 20px 0;">

```
Initial State: {}
Event 1: {cluster: "prod", brokers: 3}
Event 2: {brokers: 4}
Event 3: {health: "critical"}

Final State: {
  cluster: "prod",
  brokers: 4,
  health: "critical"
}
```

</div>

### Cross-Cell Replication

For global visibility, entities replicate across regions:

```
Primary Cell (US-East)
    │
    ├── entity-hydration-mirror
    │
    v
Secondary Cells (EU, APAC)
```

<table>
<tr>
<th width="30%">📋 Rule Type</th>
<th width="70%">📝 Description</th>
</tr>
<tr>
<td><b>Entity Selection</b></td>
<td>Only specific entity types replicate</td>
</tr>
<tr>
<td><b>Conflict Resolution</b></td>
<td>Timestamp-based resolution</td>
</tr>
<tr>
<td><b>Attribute Filtering</b></td>
<td>Selective attribute replication</td>
</tr>
<tr>
<td><b>Optimization</b></td>
<td>Bandwidth optimization through compression</td>
</tr>
</table>

---

## 🎯 Next Steps

You've completed Part 1 of the Kafka Entity Platform documentation! You now understand:

- ✅ What the Entity Platform is and why it matters
- ✅ Core concepts like entities, domains, and types
- ✅ The Kafka entity hierarchy
- ✅ How entities are created and managed

<div align="center">

### 📚 Continue Your Journey

<table>
<tr>
<td width="33%" align="center">

**[📗 Part 2: Implementation](KAFKA_ENTITY_IMPLEMENTATION.md)**

Deep dive into:
- Entity synthesis rules
- Golden metrics design
- Relationship mapping
- Provider configurations
- Dashboard creation

</td>
<td width="33%" align="center">

**[📙 Part 3: Operations](KAFKA_ENTITY_OPERATIONS.md)**

Master:
- Configuration management
- Testing strategies
- Operational excellence
- Troubleshooting
- Performance tuning

</td>
<td width="33%" align="center">

**[📕 Part 4: Advanced Topics](KAFKA_ENTITY_ADVANCED.md)**

Explore:
- Best practices
- Platform integration
- Security & compliance
- Future roadmap
- Complete glossary

</td>
</tr>
</table>

</div>

---

<div align="center">

### 🆘 Need Help?

[📖 Entity Platform Docs](#) • [💬 Community Forum](#) • [🐛 Report Issues](#) • [📧 Contact Support](#)

---

*Entity Platform for Kafka • Part 1 of 4 • Last Updated: January 2025*

</div>