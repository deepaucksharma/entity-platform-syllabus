# The Comprehensive Guide to Kafka Entity Definitions in New Relic's Entity Platform

## Table of Contents

1. [Introduction: Understanding the Big Picture](#introduction)
2. [Entity Platform Fundamentals](#entity-platform-fundamentals)
3. [Core Concepts and Architecture](#core-concepts)
4. [The Kafka Entity Hierarchy](#kafka-hierarchy)
5. [Entity Lifecycle and Data Flow](#entity-lifecycle)
6. [Entity Synthesis Deep Dive](#entity-synthesis)
7. [Golden Metrics and Performance Indicators](#golden-metrics)
8. [Relationships and Service Mapping](#relationships)
9. [Provider-Specific Implementations](#providers)
10. [Dashboards and Observability](#dashboards)
11. [Configuration Management](#configuration-management)
12. [Testing and Validation Framework](#testing)
13. [Operational Excellence](#operational-excellence)
14. [Troubleshooting and Debugging](#troubleshooting)
15. [Performance Optimization](#performance-optimization)
16. [Best Practices and Design Patterns](#best-practices)
17. [Platform Integration Points](#platform-integration)
18. [Security and Compliance](#security-compliance)
19. [Future-Proofing and Extensibility](#future-proofing)
20. [Comprehensive Glossary](#glossary)

---

## 1. Introduction: Understanding the Big Picture {#introduction}

### What is the Entity Platform?

The New Relic Entity Platform is a sophisticated distributed system that transforms raw telemetry data into intelligent, actionable insights about your infrastructure and applications. Think of it as the \"brain\" that understands what each component in your system is, how it relates to other components, and what its current state means for your business.

### Why Kafka Entity Definitions Matter

Apache Kafka has become the backbone of modern event-driven architectures. However, monitoring Kafka effectively requires understanding its complex hierarchy of clusters, brokers, topics, partitions, and consumer groups. This PR introduces a comprehensive framework that:

- **Transforms Raw Metrics** into intelligent entities with lifecycle management
- **Maps Relationships** between Kafka components and your applications automatically
- **Provides Unified Monitoring** across different Kafka providers (self-managed, AWS MSK, Confluent Cloud)
- **Enables Smart Alerting** based on Kafka-specific health calculations

### The Business Impact

Before this implementation:
```
- Fragmented monitoring across different Kafka deployments
- Manual correlation between Kafka issues and application impact
- Provider-specific monitoring tools and dashboards
- No unified view of Kafka infrastructure health
```

After this implementation:
```
- Single pane of glass for all Kafka infrastructure
- Automatic impact analysis when Kafka issues occur
- Provider-agnostic monitoring and alerting
- Intelligent health scoring based on Kafka best practices
```

### How to Use This Guide

This guide is structured to serve multiple audiences:

- **Beginners**: Start with sections 1-4 for foundational understanding
- **Developers**: Focus on sections 5-8 for implementation details
- **Operators**: Sections 11-15 cover operational aspects
- **Architects**: Sections 16-19 discuss design patterns and future considerations

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

```
Kafka Topics Used:
- entity-raw: Initial entity data
- entity-merged: Consolidated entities
- entity-deduplicated: Cleaned entity stream
- entity-definitions-v3: Configuration updates
- relationship-proposed: Discovered relationships
- relationship-validated: Confirmed relationships
```

### Key Design Principles

1. **Event Sourcing**: All state changes are captured as immutable events
2. **CQRS**: Separate paths for writes (Kafka) and reads (Elasticsearch)
3. **Eventually Consistent**: Embraces distributed system realities
4. **Idempotent Operations**: Safe to retry without side effects
5. **Schema Evolution**: Backward-compatible changes supported

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

**Domain** represents the broad category:
- `INFRA`: Infrastructure components
- `APM`: Application services
- `BROWSER`: Frontend applications
- `MOBILE`: Mobile applications
- `SYNTH`: Synthetic monitors

**Type** represents the specific kind of entity within a domain.

### Entity Identification

Each entity has a unique GUID (Globally Unique Identifier) generated deterministically:

```
GUID = base64(hash(accountId + domain + type + identifier))

Example:
- accountId: 12345678
- domain: INFRA
- type: MESSAGE_QUEUE_CLUSTER
- identifier: \"prod-kafka-cluster\"
- GUID: \"MTIzNDU2NzhJTkZSQU1FU1NBR0VfUVVFVUVfQ0xVU1RFUnByb2Qta2Fma2E\"
```

This deterministic approach ensures:
- Same entity always gets same GUID
- No database lookups needed
- Works across distributed systems
- Supports entity merging

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

Golden tags appear in:
- Entity search interfaces
- Filter dropdowns
- Entity lists
- Relationship displays

### Synthesis Rules

Synthesis rules are the \"recipes\" that create entities from raw telemetry:

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

```yaml
tags:
  consumer.lag.sum:
    ttl: P5M  # 5 minutes - rapidly changing metric
  kafka.version:
    ttl: P30D # 30 days - rarely changes
  provider:
    # No TTL - permanent attribute
```

### Configuration Settings

Entity behavior is controlled through configuration:

```yaml
configuration:
  alertable: true                    # Can create alerts on this entity
  entityExpirationTime: EIGHT_DAYS   # Auto-cleanup after inactivity
  isContainer: true                  # Contains other entities
```

Expiration times:
- `FOUR_HOURS`: High-cardinality entities (partitions)
- `EIGHT_DAYS`: Standard infrastructure
- `THIRTY_DAYS`: Long-lived entities (consumer groups)
- `NEVER`: Permanent entities

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

#### MESSAGE_QUEUE_CLUSTER

**Purpose**: Represents the entire Kafka deployment

**Key Attributes**:
```yaml
Identifiers:
  - clusterName (self-managed)
  - kafka.cluster.arn (AWS MSK)
  - confluent.kafka.cluster.id (Confluent)

Health Indicators:
  - activeControllerCount (must be 1)
  - offlinePartitionsCount (must be 0)
  - underReplicatedPartitions (should be 0)

Performance Metrics:
  - throughputInBytesPerSec
  - throughputOutBytesPerSec
  - totalBrokers
  - totalTopics

Provider-Specific:
  - clusterLoadPercent (Confluent)
  - hotPartitionCount (Confluent)
```

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

#### MESSAGE_QUEUE_BROKER

**Purpose**: Individual Kafka server within a cluster

**Key Attributes**:
```yaml
Identifiers:
  - broker.id
  - hostname

Resource Metrics:
  - cpuPercent
  - memoryUsedPercent
  - diskUsedPercent
  - networkUtilization

Performance Indicators:
  - requestHandlerAvgIdlePercent (>70% is healthy)
  - networkProcessorAvgIdlePercent (>70% is healthy)
  - underReplicatedPartitions (per broker)

Workload Metrics:
  - partitionCount
  - leaderCount
  - bytesInPerSec
  - bytesOutPerSec
```

**Performance Thresholds**:
- CPU < 80%: Healthy
- Disk < 85%: Healthy
- Request Handler Idle > 70%: Healthy
- Network Processor Idle > 70%: Healthy

#### MESSAGE_QUEUE_TOPIC

**Purpose**: Logical message channel

**Key Attributes**:
```yaml
Configuration:
  - partitions (partition count)
  - replicationFactor
  - retentionMs
  - cleanupPolicy

Activity Metrics:
  - messagesInPerSec
  - messagesOutPerSec
  - bytesInPerSec
  - bytesOutPerSec

Consumer Health:
  - maxConsumerLag
  - consumerGroupCount
  - activeProducers

Performance:
  - fetchRequestRate
  - produceRequestRate
```

#### MESSAGE_QUEUE_PARTITION

**Purpose**: Physical storage unit within a topic

**Key Attributes**:
```yaml
Identity:
  - partition.id
  - topic.name
  - leader.broker.id

State Information:
  - inSyncReplicas
  - replicas
  - highWatermark
  - logEndOffset
  - logStartOffset
  - size

Special Considerations:
  - High cardinality entity
  - 4-hour TTL (vs 8 days for others)
  - Not alertable
```

#### MESSAGE_QUEUE_CONSUMER_GROUP

**Purpose**: Coordinated group of consumers

**Key Metrics**:
```yaml
Lag Metrics:
  - totalLag (sum across all partitions)
  - maxLag (highest single partition)
  - lagTrend (increasing/decreasing)

Stability Metrics:
  - memberCount
  - rebalanceRate
  - state (active/rebalancing/dead)

Performance:
  - messagesConsumedPerSec
  - bytesConsumedPerSec
  - assignedPartitions
```

**Lag Health Scoring**:
- Lag < 1000: Healthy
- Lag < 10000: Warning  
- Lag >= 10000: Critical

#### MESSAGE_QUEUE_PRODUCER / CONSUMER

**Purpose**: Application-level entities

**Sources**:
- Kafka integration metrics
- APM distributed tracing
- OpenTelemetry spans

**Enables**:
- Application impact analysis
- End-to-end latency tracking
- Service dependency mapping

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

```yaml
Deduplication Windows:
  - Short Window: 1 minute (catch rapid duplicates)
  - Medium Window: 5 minutes (handle delays)
  - Long Window: 1 hour (cross-region sync)

Deduplication Keys:
  - Entity GUID
  - Timestamp
  - Mutation type (create/update/delete)
```

### State Management

Entities maintain state through event accumulation:

```
Initial State: {}
Event 1: {cluster: \"prod\", brokers: 3}
Event 2: {brokers: 4}
Event 3: {health: \"critical\"}

Final State: {
  cluster: \"prod\",
  brokers: 4,
  health: \"critical\"
}
```

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

Replication rules:
- Only specific entity types replicate
- Conflict resolution via timestamp
- Selective attribute replication
- Bandwidth optimization

---

## 6. Entity Synthesis Deep Dive {#entity-synthesis}

### The Synthesis Engine

The synthesis engine transforms raw telemetry into entities:

```
Input: Raw Telemetry Event
{
  \"eventType\": \"KafkaClusterSample\",
  \"clusterName\": \"prod-kafka\",
  \"cluster.activeControllerCount\": 1,
  \"cluster.offlinePartitionsCount\": 0,
  \"timestamp\": 1234567890
}

Process: Rule Matching
- Find rules where conditions match
- Extract identifier components
- Generate entity GUID
- Map attributes to tags

Output: Synthesized Entity
{
  \"guid\": \"MTIzNDU2Nzg...\",
  \"type\": \"MESSAGE_QUEUE_CLUSTER\",
  \"name\": \"prod-kafka\",
  \"tags\": {
    \"kafka.cluster.name\": \"prod-kafka\",
    \"provider\": \"SELF_MANAGED\"
  }
}
```

### Multi-Provider Synthesis Strategy

Different providers require different synthesis approaches:

#### Self-Managed Kafka
```yaml
Identifier: clusterName
Primary Key: Simple cluster name
Challenges: Name uniqueness across accounts
Solution: Include account ID in GUID
```

#### AWS MSK
```yaml
# Polling Integration
Identifier: aws.kafka.clusterArn
Primary Key: Full ARN
Advantages: Globally unique
Format: arn:aws:kafka:region:account:cluster/name/id

# Metric Streams
Identifier: \"{{accountId}}:{{region}}:{{clusterName}}\"
Primary Key: Composite ID
Reason: ARN not available in stream data
```

#### Confluent Cloud
```yaml
Identifier: confluent.kafka.cluster.id
Primary Key: Cluster ID (lkc-xxxxx)
Advantages: Guaranteed unique
Additional: Environment context available
```

### Synthesis Rule Components

```yaml
synthesis:
  rules:
    - identifier: \"expression\"          # What makes this unique
      name: \"expression\"               # Display name  
      encodeIdentifierInGUID: true     # Include in GUID
      conditions: []                   # When to apply
      tags: {}                        # What to extract
```

### Conditional Synthesis

Rules can have complex conditions:

```yaml
conditions:
  # Simple match
  - attribute: eventType
    value: KafkaClusterSample
    
  # Multiple values
  - attribute: eventType
    anyOf: [KafkaClusterSample, AwsMskClusterSample]
    
  # Presence check
  - attribute: clusterName
    present: true
    
  # Negation
  - attribute: test.environment
    value: true
    negate: true
```

### Tag Extraction Patterns

```yaml
tags:
  # Simple mapping
  clusterName:
    entityTagName: kafka.cluster.name
    
  # With fallbacks
  clusterName:
    entityTagName: kafka.cluster.name
    fallbackAttribute: displayName
    fallbackAttribute: name
    
  # With TTL
  consumerLag:
    entityTagName: consumer.lag
    ttl: P5M  # 5 minutes
    
  # Static values
  provider:
    value: AWS_MSK
```

### Entity Definition Publishing

Entity definitions are distributed via a dedicated service:

```
Zookeeper (Configuration Paths)
    │
    v
entity-definitions-publisher
    │
    ├── Fetch from S3
    ├── Validate definitions
    └── Publish to Kafka topics:
        ├── entity_definitions_v3
        ├── entity_definitions_v3_synthesis
        ├── entity_definitions_v3_metrics
        ├── entity_definitions_v3_dashboards
        └── entity_definitions_v3_relationships
```

This enables:
- Hot configuration updates
- No service restarts needed
- Gradual rollout capabilities
- Version control via S3

---

## 7. Golden Metrics and Performance Indicators {#golden-metrics}

### Understanding Golden Metrics

Golden metrics are the most critical measurements for each entity type. They:
- Appear prominently in the UI
- Drive health calculations
- Power default dashboards
- Enable smart alerting

### Metric Definition Structure

```yaml
metricName:
  title: \"Human-Readable Name\"
  unit: UNIT_TYPE
  queries:
    providerName:
      select: \"NRQL aggregation function\"
      from: \"Event type(s)\"
      where: \"Filter conditions\"
      facet: \"Group by clause\"
      eventId: \"Correlation field\"
```

### Unit Types

```yaml
Common Units:
  - COUNT: Discrete quantities
  - PERCENTAGE: 0-100 values
  - BYTES: Data size
  - BYTES_PER_SECOND: Throughput
  - REQUESTS_PER_SECOND: Rate metrics
  - MILLISECONDS: Latency
  - STRING: Text values
```

### Provider-Specific Queries

Metrics adapt to different data sources:

```yaml
bytesInPerSec:
  title: \"Incoming Throughput\"
  unit: BYTES_PER_SECOND
  queries:
    # Self-managed Kafka
    nriKafka:
      select: \"average(`broker.bytesInPerSec`)\"
      from: \"KafkaBrokerSample\"
      
    # AWS MSK Polling
    awsMsk:
      select: \"average(`aws.kafka.BytesInPerSec.byBroker`)\"
      from: \"AwsMskBrokerSample\"
      
    # AWS MSK Streams (rate calculation)
    awsMskStreams:
      select: \"rate(sum(aws.kafka.BytesInPerSec), 1 second)\"
      from: \"MetricRaw\"
      
    # Confluent Cloud
    confluentCloud:
      select: \"sum(`io.confluent.kafka.server/received_bytes`)\"
      from: \"Metric\"
```

### Cluster-Level Golden Metrics

```yaml
Critical Health Metrics:
  - activeControllerCount: Must be exactly 1
  - offlinePartitionsCount: Must be 0
  - underReplicatedPartitions: Should be 0

Performance Metrics:
  - throughputInBytesPerSec: Overall data ingestion
  - throughputOutBytesPerSec: Overall data delivery
  - clusterLoadPercent: Confluent capacity usage

Scale Metrics:
  - totalBrokers: Cluster size
  - totalTopics: Data organization
  - totalPartitions: Parallelism level
```

### Broker-Level Golden Metrics

```yaml
Resource Utilization:
  - cpuPercent: Processor usage
  - diskUsedPercent: Storage consumption
  - memoryUsedPercent: RAM utilization

Performance Health:
  - requestHandlerAvgIdlePercent: >70% healthy
  - networkProcessorAvgIdlePercent: >70% healthy
  - underReplicatedPartitions: Per-broker issues

Workload Distribution:
  - partitionCount: Total hosted partitions
  - leaderCount: Primary partitions
  - bytesInPerSec: Broker-specific ingestion
  - bytesOutPerSec: Broker-specific delivery
```

### Topic-Level Golden Metrics

```yaml
Activity Metrics:
  - messagesInPerSec: Message production rate
  - messagesOutPerSec: Message consumption rate
  - bytesInPerSec: Data production volume
  - bytesOutPerSec: Data consumption volume

Consumer Health:
  - maxConsumerLag: Worst-case delay
  - consumerGroupCount: Active consumers
  - activeProducers: Connected producers

Configuration:
  - partitionCount: Parallelism factor
  - replicationFactor: Durability setting
```

### Consumer Group Golden Metrics

```yaml
Lag Analysis:
  - totalLag: Sum across all partitions
  - maxLag: Worst partition lag
  - lagTrend: Derivative (improving/worsening)

Stability Metrics:
  - memberCount: Active consumers
  - rebalanceRate: Frequency of disruptions
  - state: Current group status

Performance:
  - messagesConsumedPerSec: Processing rate
  - bytesConsumedPerSec: Data volume
  - assignedPartitions: Workload size
```

### Summary Metrics

Summary metrics provide quick status in entity lists:

```yaml
providerInfo:
  title: \"Provider\"
  unit: STRING
  # Shows: AWS_MSK, CONFLUENT_CLOUD, SELF_MANAGED

healthStatus:
  title: \"Health\"
  unit: STRING
  # Shows: Healthy, Warning, Critical, Unknown

incomingThroughput:
  title: \"In\"
  unit: BYTES_PER_SECOND
  # Aggregated cluster throughput

brokerCount:
  title: \"Brokers\"
  unit: COUNT
  # Quick scale indicator
```

### Metric Query Patterns

#### Aggregation Patterns
```sql
-- Sum across multiple brokers
SELECT sum(broker.bytesInPerSec) 
FROM KafkaBrokerSample 
WHERE clusterName = 'prod-kafka'

-- Average for resource metrics
SELECT average(broker.cpuPercent)
FROM KafkaBrokerSample
WHERE entity.guid = '{entity.guid}'

-- Max for worst-case analysis
SELECT max(consumer.lag)
FROM KafkaOffsetSample
WHERE consumerGroup = 'api-consumers'
```

#### Time-based Patterns
```sql
-- Rate calculation
SELECT rate(sum(messagesIn), 1 second)
FROM KafkaTopicSample
WHERE topic = 'orders'

-- Trend analysis
SELECT derivative(sum(consumer.lag), 1 minute)
FROM KafkaOffsetSample
WHERE consumerGroup = 'analytics'

-- Time series
SELECT average(cpuPercent)
FROM KafkaBrokerSample
TIMESERIES 1 minute
```

#### Multi-Provider Patterns
```sql
-- Handle different attribute names
SELECT sum(broker.bytesInPerSec) OR sum(aws.kafka.BytesInPerSec.byBroker)
FROM KafkaBrokerSample, AwsMskBrokerSample
WHERE clusterName = 'prod' OR aws.kafka.clusterArn LIKE '%prod%'

-- Coalesce missing values
SELECT COALESCE(latest(broker.cpuPercent), 0) as 'CPU %'
FROM KafkaBrokerSample
```

---

## 8. Relationships and Service Mapping {#relationships}

### Understanding Entity Relationships

Relationships connect entities to show how your infrastructure works together:

```
Application Service
    │
    ├── PRODUCES_TO → Kafka Topic
    │                     │
    │                     ├── CONTAINS → Partitions
    │                     │
    │                     └── CONSUMED_BY → Consumer Group
    │                                            │
    └── MONITORED_BY → APM Agent               └── CONTAINS → Consumers
```

### Relationship Types

```yaml
Structural Relationships (Infrastructure):
  - CONTAINS: Ownership hierarchy
  - HOSTS: Physical hosting
  - OPERATES_IN: Regional deployment

Behavioral Relationships (Activity):
  - PRODUCES_TO: Data production
  - CONSUMES_FROM: Data consumption
  - CALLS: Service communication
  - CONNECTS_TO: Network connections
```

### Relationship Definition Structure

```yaml
relationships:
  - name: uniqueRelationshipName
    version: \"1\"
    origins: [data-sources]
    conditions: []              # When to create
    relationship:
      expires: P24H            # TTL
      relationshipType: TYPE   
      source:                  # From entity
        buildGuid: {}         # Build new
        extractGuid: {}       # Use existing
        lookupGuid: {}        # Find entity
      target:                  # To entity
        buildGuid: {}
        extractGuid: {}
        lookupGuid: {}
```

### TTL Strategy for Relationships

Different relationship types require different expiration times:

```yaml
Infrastructure Relationships (24 hours):
  - Cluster → Broker
  - Topic → Partition
  - Broker → Partition
  Reason: Stable, change infrequently

Dynamic Relationships (15 minutes):
  - Application → Topic
  - Consumer Group → Topic
  - Producer → Topic
  Reason: Based on activity, may stop/start

Configuration Relationships (7 days):
  - Service → Database
  - Container → Host
  Reason: Semi-permanent but can change
```

### Source and Target Patterns

#### Building GUIDs
Used when the entity might not exist yet:

```yaml
source:
  buildGuid:
    account:
      attribute: accountId
    domain:
      value: INFRA
    type:
      value: MESSAGE_QUEUE_CLUSTER
    identifier:
      fragments:
        - attribute: clusterName
        - value: \":\"
        - attribute: environment
```

#### Extracting GUIDs
Used when GUID is already in the event:

```yaml
source:
  extractGuid:
    attribute: entity.guid
```

#### Looking Up GUIDs
Used to find existing entities:

```yaml
target:
  lookupGuid:
    candidateCategory: entity
    fields:
      - field: domainType.type
        value: MESSAGE_QUEUE_TOPIC
      - field: kafka.topic.name
        attribute: topicName
```

### Key Kafka Relationships

#### 1. Cluster Contains Broker
```yaml
Purpose: Show broker membership
Source: MESSAGE_QUEUE_CLUSTER
Target: MESSAGE_QUEUE_BROKER
TTL: 24 hours
Created from: Broker telemetry events
```

#### 2. Topic Contains Partition
```yaml
Purpose: Topic structure
Source: MESSAGE_QUEUE_TOPIC  
Target: MESSAGE_QUEUE_PARTITION
TTL: 24 hours
Created from: Partition telemetry
```

#### 3. Broker Hosts Partition
```yaml
Purpose: Show partition leadership
Source: MESSAGE_QUEUE_BROKER
Target: MESSAGE_QUEUE_PARTITION
TTL: 24 hours
Condition: partition.leader present
```

#### 4. Consumer Group Consumes From Topic
```yaml
Purpose: Consumption patterns
Source: MESSAGE_QUEUE_CONSUMER_GROUP
Target: MESSAGE_QUEUE_TOPIC
TTL: 15 minutes
Created from: Offset commit events
```

#### 5. Application Produces To Topic
```yaml
Purpose: Data flow mapping
Source: APM Application
Target: MESSAGE_QUEUE_TOPIC
TTL: 15 minutes
Created from: Producer spans
Conditions:
  - span.kind = \"producer\"
  - messaging.system = \"kafka\"
```

### APM Integration

The platform creates relationships from distributed tracing:

```yaml
Producer Span Attributes:
  - span.kind: \"producer\"
  - messaging.system: \"kafka\"
  - messaging.destination.name: \"orders\"
  - messaging.kafka.cluster.id: \"prod-cluster\"
  
Result: Application → PRODUCES_TO → Topic

Consumer Span Attributes:
  - span.kind: \"consumer\"
  - messaging.system: \"kafka\"
  - messaging.source.name: \"orders\"
  - messaging.kafka.consumer.group: \"api-consumers\"
  
Result: Application → CONSUMES_FROM → Topic
```

### Relationship Discovery Process

```
1. Event Arrives
   │
2. Check Conditions
   │
3. Extract Attributes
   │
4. Build/Extract/Lookup Source GUID
   │
5. Build/Extract/Lookup Target GUID
   │
6. Create Relationship with TTL
   │
7. Publish to relationship stream
```

### Troubleshooting Relationships

Common issues and solutions:

#### Missing Relationships
```yaml
Symptom: No connection between app and topic
Checks:
  1. Verify span attributes present
  2. Check messaging.system = \"kafka\"
  3. Ensure cluster ID matches
  4. Verify TTL hasn't expired
```

#### Incorrect Relationships
```yaml
Symptom: Wrong entities connected
Checks:
  1. Validate identifier construction
  2. Check for naming conflicts
  3. Verify attribute mappings
  4. Review GUID generation
```

#### Expired Relationships
```yaml
Symptom: Relationships disappear
Solution:
  1. Check activity still occurring
  2. Verify TTL appropriate
  3. Monitor relationship events
  4. Adjust TTL if needed
```

---

## 9. Provider-Specific Implementations {#providers}

### Overview of Supported Providers

The platform supports three main Kafka providers, each with unique characteristics:

```yaml
Providers:
  SELF_MANAGED:
    - Full JMX access
    - Complete metrics
    - Custom deployment
    
  AWS_MSK:
    - Managed service
    - CloudWatch/Streams
    - Limited metrics
    
  CONFLUENT_CLOUD:
    - SaaS platform
    - API-based metrics
    - Cloud-native features
```

### Self-Managed Kafka Implementation

#### Data Collection
```yaml
Integration: nri-kafka
Method: JMX polling
Frequency: 15-60 seconds
Depth: Full metric access
```

#### Entity Identification
```yaml
Cluster: clusterName
Broker: clusterName:brokerId
Topic: clusterName:topicName
Partition: clusterName:topicName:partitionId
```

#### Available Metrics
```yaml
Complete Access:
  - All JMX MBeans
  - Producer metrics
  - Consumer metrics
  - Internal metrics
  - Custom metrics
```

#### Synthesis Rules
```yaml
- identifier: clusterName
  conditions:
    - attribute: eventType
      value: KafkaClusterSample
  tags:
    clusterName:
      entityTagName: kafka.cluster.name
    provider:
      value: SELF_MANAGED
```

### AWS MSK Implementation

#### Two Integration Methods

##### 1. CloudWatch Polling
```yaml
Method: AWS API polling
Latency: 1-5 minutes
Cost: CloudWatch API charges
Identification: ARN-based
```

##### 2. Metric Streams
```yaml
Method: Kinesis streaming
Latency: Near real-time
Cost: Kinesis charges
Identification: Composite IDs
```

#### Entity Identification Strategy
```yaml
# Polling uses ARN (globally unique)
Cluster: arn:aws:kafka:region:account:cluster/name/uuid
Broker: arn:aws:kafka:region:account:cluster/name/uuid:brokerId

# Streams use composite (no ARN in stream)
Cluster: accountId:region:clusterName
Broker: accountId:region:clusterName:brokerId
```

#### Synthesis Differences
```yaml
# Polling synthesis
- identifier: aws.kafka.clusterArn
  conditions:
    - attribute: eventType
      value: AwsMskClusterSample
    - attribute: provider.source
      value: cloudwatch

# Streams synthesis  
- identifier: \"{{ aws.accountId }}:{{ aws.region }}:{{ aws.kafka.ClusterName }}\"
  conditions:
    - attribute: eventType
      value: MetricRaw
    - attribute: aws.Namespace
      value: AWS/Kafka
    - attribute: metricStreamName
      present: true
```

#### Metric Mappings
```yaml
MSK Metric → Generic Metric:
  aws.kafka.ActiveControllerCount → activeControllerCount
  aws.kafka.OfflinePartitionsCount → offlinePartitionsCount
  aws.kafka.BytesInPerSec.byBroker → broker.bytesInPerSec
  aws.kafka.CpuUser → broker.cpuPercent
```

#### Limitations
```yaml
Not Available in MSK:
  - Producer client metrics
  - Consumer client metrics
  - Detailed JMX metrics
  - Some internal metrics
```

### Confluent Cloud Implementation

#### Data Collection
```yaml
Integration: Confluent Metrics API
Method: REST API polling
Authentication: API key/secret
Frequency: 60 seconds
```

#### Unique Features
```yaml
Cloud-Native Metrics:
  - cluster_load_percent
  - hot_partition_ingress/egress
  - cloud_provider attribution
  - environment context
```

#### Entity Identification
```yaml
Cluster: confluent.kafka.cluster.id (lkc-xxxxx)
Broker: confluent.kafka.cluster.id:brokerId
Topic: confluent.kafka.cluster.id:topicName
```

#### Synthesis Rules
```yaml
- identifier: confluent.kafka.cluster.id
  name: resource.kafka.id
  conditions:
    - attribute: eventType
      value: ConfluentCloudClusterSample
  tags:
    confluent.kafka.cluster.id:
      entityTagName: kafka.cluster.id
    provider:
      value: CONFLUENT_CLOUD
```

#### Metric Namespace
```yaml
Confluent Metrics:
  io.confluent.kafka.server/received_bytes
  io.confluent.kafka.server/sent_bytes
  io.confluent.kafka.server/received_records
  kafka.consumer.lag_offsets
```

### Provider Comparison Matrix

| Feature | Self-Managed | AWS MSK | Confluent Cloud |
|---------|--------------|----------|-----------------|
| **Identification** | Cluster name | ARN/Composite | Cluster ID |
| **Metric Latency** | 15-60s | 1-5min/real-time | 60s |
| **Metric Depth** | Full JMX | CloudWatch subset | API subset |
| **Producer Metrics** | ✓ | Limited | Limited |
| **Consumer Metrics** | ✓ | Limited | Via API |
| **Client Metrics** | ✓ | ✗ | ✗ |
| **Hot Partition Detection** | Manual | ✗ | ✓ |
| **Cost** | Infrastructure | CloudWatch/Kinesis | API calls |
| **Multi-Region** | Manual | Native | Native |
| **Managed Upgrades** | ✗ | ✓ | ✓ |

### Choosing the Right Provider

#### Self-Managed Kafka
```yaml
Choose when:
  - Need full control
  - Require all metrics
  - Custom configurations
  - On-premises deployment

Considerations:
  - Operational overhead
  - Manual upgrades
  - Infrastructure costs
```

#### AWS MSK
```yaml
Choose when:
  - Already on AWS
  - Want managed service
  - Need AWS integration
  - Require compliance

Considerations:
  - Limited metrics
  - AWS lock-in
  - CloudWatch costs
```

#### Confluent Cloud
```yaml
Choose when:
  - Want full SaaS
  - Multi-cloud strategy
  - Minimal operations
  - Advanced features

Considerations:
  - Highest cost
  - Internet dependency
  - Limited customization
```

### Multi-Provider Strategies

Organizations often run multiple Kafka providers:

```yaml
Common Patterns:
  1. Production on MSK, Development on self-managed
  2. Core on self-managed, Edge on Confluent
  3. Migration from self-managed to cloud

Unified Monitoring Benefits:
  - Single dashboard for all Kafka
  - Consistent alerting rules
  - Unified health scoring
  - Cross-provider comparisons
```

### Provider Migration Support

The entity platform helps during migrations:

```yaml
Migration Visibility:
  - Side-by-side comparisons
  - Throughput matching
  - Consumer lag tracking
  - Relationship preservation

Entity Continuity:
  - Same entity types
  - Consistent metrics
  - Unified dashboards
  - Standard alerts
```

---

## 10. Dashboards and Observability {#dashboards}

### Dashboard Architecture

Each entity type can have custom dashboards with multiple pages:

```json
{
  \"name\": \"{{{entity.name}}} Kafka Cluster\",
  \"description\": \"Comprehensive monitoring dashboard\",
  \"permissions\": \"PUBLIC_READ_WRITE\",
  \"pages\": [
    {
      \"name\": \"Overview\",
      \"description\": \"Cluster health and key metrics\",
      \"widgets\": [...]
    },
    {
      \"name\": \"Performance\",
      \"description\": \"Detailed performance analysis\",
      \"widgets\": [...]
    }
  ]
}
```

### Widget Types and Best Practices

#### 1. Billboard Widget
**Purpose**: Key status indicators

```json
{
  \"visualization\": {
    \"id\": \"viz.billboard\"
  },
  \"rawConfiguration\": {
    \"nrqlQueries\": [{
      \"query\": \"SELECT latest(activeControllerCount) as 'Controllers'\"
    }],
    \"thresholds\": [{
      \"alertSeverity\": \"CRITICAL\",
      \"name\": \"Not exactly 1\",
      \"value\": 0.99
    }]
  }
}
```

**Best for**:
- Health status
- Current values
- Alert thresholds
- KPI display

#### 2. Line Chart Widget
**Purpose**: Trends over time

```json
{
  \"visualization\": {
    \"id\": \"viz.line\"
  },
  \"rawConfiguration\": {
    \"legend\": {
      \"enabled\": true
    },
    \"yAxisLeft\": {
      \"zero\": true,
      \"max\": 100
    },
    \"nrqlQueries\": [{
      \"query\": \"SELECT average(cpuPercent) TIMESERIES AUTO\"
    }]
  }
}
```

**Best for**:
- Performance trends
- Throughput patterns
- Resource utilization
- Comparative analysis

#### 3. Table Widget
**Purpose**: Detailed breakdowns

```json
{
  \"visualization\": {
    \"id\": \"viz.table\"
  },
  \"rawConfiguration\": {
    \"nrqlQueries\": [{
      \"query\": \"SELECT latest(cpuPercent), latest(diskUsedPercent) FACET broker.id\"
    }]
  }
}
```

**Best for**:
- Per-broker metrics
- Consumer group details
- Topic comparisons
- Sorted listings

#### 4. Heatmap Widget
**Purpose**: Distribution analysis

```json
{
  \"visualization\": {
    \"id\": \"viz.heatmap\"
  },
  \"rawConfiguration\": {
    \"nrqlQueries\": [{
      \"query\": \"SELECT histogram(bytesInPerSec, 10) FACET topic\"
    }]
  }
}
```

**Best for**:
- Topic activity distribution
- Partition hotspots
- Temporal patterns
- Anomaly detection

#### 5. Bar Chart Widget
**Purpose**: Categorical comparisons

```json
{
  \"visualization\": {
    \"id\": \"viz.bar\"
  },
  \"rawConfiguration\": {
    \"nrqlQueries\": [{
      \"query\": \"SELECT sum(messagesInPerSec) FACET topic LIMIT 20\"
    }]
  }
}
```

**Best for**:
- Top N analysis
- Resource rankings
- Volume comparisons
- Category breakdowns

### Dynamic Variables

Dashboards use entity context:

```yaml
Available Variables:
  {{{entity.name}}} - Current entity name
  {{{entity.guid}}} - Entity GUID
  {{{account.id}}} - Account ID
  {{{kafka.cluster.name}}} - Cluster name tag
  {{{kafka.cluster.arn}}} - AWS ARN if applicable
  {{{kafka.cluster.id}}} - Cluster ID
```

### Multi-Provider Dashboard Queries

Handle different providers in one query:

```sql
-- Cluster health across providers
SELECT latest(activeControllerCount) 
FROM KafkaClusterSample, AwsMskClusterSample, ConfluentCloudClusterSample
WHERE entity.guid = '{{{entity.guid}}}'

-- Broker metrics with fallbacks
SELECT average(broker.cpuPercent) OR average(aws.kafka.CpuUser)
FROM KafkaBrokerSample, AwsMskBrokerSample
WHERE clusterName = '{{{kafka.cluster.name}}}' 
   OR aws.kafka.clusterArn = '{{{kafka.cluster.arn}}}'
```

### Dashboard Pages Structure

#### Overview Page
```yaml
Purpose: Quick health assessment
Widgets:
  - Health Status (Billboard)
  - Throughput Trends (Line)
  - Broker Status (Table)
  - Top Topics (Bar)
  - Consumer Lag Summary (Table)
```

#### Performance Page
```yaml
Purpose: Deep performance analysis
Widgets:
  - Resource Utilization (Line)
  - Request Handler Performance (Line)
  - Partition Distribution (Bar)
  - Replication Status (Table)
```

#### Topic Analytics Page
```yaml
Purpose: Topic-level insights
Widgets:
  - Topic Throughput Heatmap
  - Message Rates (Line)
  - Consumer Lag by Topic (Table)
  - Producer Activity (Bar)
```

#### Consumer Health Page
```yaml
Purpose: Consumption monitoring
Widgets:
  - Consumer Group Lag (Table)
  - Lag Trends (Line)
  - Rebalance Frequency (Billboard)
  - Consumer Distribution (Bar)
```

### Dashboard Best Practices

#### Layout Guidelines
```yaml
Grid System:
  - 12 columns wide
  - Variable height rows
  - Responsive design

Widget Sizing:
  - Critical metrics: 4-6 columns
  - Trends: 8-12 columns
  - Tables: 6-12 columns
  - Status: 3-4 columns
```

#### Query Optimization
```yaml
Performance Tips:
  - Use SINCE clause appropriately
  - Limit FACET cardinality
  - Pre-filter with WHERE
  - Avoid SELECT *
  - Use TIMESERIES AUTO
```

#### Color Schemes
```yaml
Status Colors:
  - Green: Healthy/Good
  - Yellow: Warning
  - Red: Critical/Bad
  - Blue: Informational
  - Gray: Unknown/No data
```

### Creating Custom Dashboards

#### Dashboard Template
```json
{
  \"name\": \"Kafka Monitoring - {{{kafka.cluster.name}}}\",
  \"permissions\": \"PUBLIC_READ_WRITE\",
  \"pages\": [{
    \"name\": \"Custom View\",
    \"widgets\": [{
      \"title\": \"Custom Metric\",
      \"layout\": {
        \"column\": 1,
        \"row\": 1,
        \"width\": 6,
        \"height\": 3
      },
      \"visualization\": {
        \"id\": \"viz.line\"
      },
      \"rawConfiguration\": {
        \"nrqlQueries\": [{
          \"accountId\": \"{{{account.id}}}\",
          \"query\": \"YOUR CUSTOM QUERY HERE\"
        }]
      }
    }]
  }]
}
```

#### Adding Business Context
```sql
-- Business-aligned metrics
SELECT 
  sum(messagesInPerSec) as 'Orders/sec'
FROM KafkaTopicSample
WHERE topic = 'orders'

-- SLA monitoring
SELECT 
  percentage(count(*), WHERE consumer.lag < 1000) as 'SLA %'
FROM KafkaOffsetSample
WHERE consumerGroup = 'critical-consumers'
```

### Observability Strategy

#### Layered Monitoring
```yaml
Level 1 - Executive Dashboard:
  - Overall health score
  - Business KPIs
  - Trend indicators
  
Level 2 - Operations Dashboard:
  - Cluster health
  - Resource utilization
  - Performance metrics
  
Level 3 - Engineering Dashboard:
  - Individual components
  - Debug information
  - Raw metrics
```

#### Alert Integration
```yaml
Dashboard Alerts:
  - Click metric → Create alert
  - Threshold visualization
  - Historical context
  - Related entities
```

#### Mobile Considerations
```yaml
Mobile Dashboards:
  - Simplified layout
  - Key metrics only
  - Single column design
  - Touch-friendly widgets
```

---

## 11. Configuration Management {#configuration-management}

### Entity Definition Lifecycle

```
Development → Testing → Staging → Production

1. Development:
   - Create/modify definition files
   - Local validation
   - Unit testing

2. Testing:
   - Deploy to test environment
   - Integration testing
   - Metric validation

3. Staging:
   - Performance testing
   - Multi-provider validation
   - Dashboard verification

4. Production:
   - Gradual rollout
   - Monitoring
   - Rollback capability
```

### Configuration Distribution

The platform uses a sophisticated configuration management system:

```
┌─────────────────┐
│   GitHub Repo   │  (Source of truth)
└────────┬────────┘
         │ CI/CD
         v
┌─────────────────┐
│    S3 Bucket    │  (Versioned storage)
└────────┬────────┘
         │
         v
┌─────────────────┐
│   Zookeeper     │  (Configuration paths)
└────────┬────────┘
         │ Watch
         v
┌─────────────────┐
│ Entity Defs     │
│  Publisher      │  (Distribution service)
└────────┬────────┘
         │ Publish
         v
┌─────────────────┐
│  Kafka Topics   │  (Event distribution)
└─────────────────┘
```

### Configuration Categories

```yaml
Categories:
  SYNTHESIS:
    - Entity creation rules
    - Tag mappings
    - Provider variations
    
  METRICS:
    - Golden metrics
    - Summary metrics
    - Calculated fields
    
  DASHBOARDS:
    - Default dashboards
    - Widget configurations
    - Layout templates
    
  RELATIONSHIPS:
    - Relationship rules
    - TTL strategies
    - Condition logic
    
  CONFIGURATIONS:
    - Entity settings
    - Behavior flags
    - Expiration rules
    
  CANDIDATES:
    - Experimental features
    - Beta definitions
    - A/B testing
```

### File Organization

```
entity-definitions/
├── entity-types/
│   ├── message-queue-cluster/
│   │   ├── definition.yml
│   │   ├── golden_metrics.yml
│   │   ├── summary_metrics.yml
│   │   ├── dashboard.json
│   │   └── tests/
│   │       ├── nri-kafka.json
│   │       ├── aws-msk.json
│   │       └── confluent.json
│   └── [other entity types...]
└── relationships/
    ├── cluster-to-broker.yml
    ├── topic-to-partition.yml
    └── [other relationships...]
```

### Definition Validation

#### Schema Validation
```yaml
Required Fields:
  - domain: Must be valid domain
  - type: Must be unique within domain
  - synthesis.rules: At least one rule
  - configuration: Required settings

Optional Fields:
  - goldenTags: Recommended
  - golden_metrics: Recommended
  - dashboard: Recommended
```

#### Syntax Validation
```bash
# Local validation
./gradlew validateDefinitions

# Checks performed:
- YAML syntax
- Required fields
- Type consistency
- Cross-references
```

#### Semantic Validation
```yaml
Business Rules:
  - GUIDs must be deterministic
  - TTLs must be appropriate
  - Relationships must be bidirectional
  - Metrics must have units
```

### Version Control

#### Change Management
```yaml
Version Strategy:
  - Git for source control
  - S3 versioning for rollback
  - Feature flags for gradual rollout
  - Canary deployments

Change Process:
  1. Create feature branch
  2. Modify definitions
  3. Add/update tests
  4. PR review
  5. Automated validation
  6. Merge to main
  7. Auto-deploy to S3
```

#### Rollback Procedures
```yaml
Rollback Options:
  1. S3 version revert
  2. Zookeeper path update
  3. Feature flag disable
  4. Manual republish

Rollback Commands:
  # Revert S3 version
  aws s3api restore-object --bucket definitions --key path --version-id xxx
  
  # Update Zookeeper
  zkCli.sh set /entity-definitions/path \"s3://bucket/old-version\"
  
  # Disable feature
  feature-flag set kafka-entities-enabled false
```

### Feature Flags

```yaml
Entity-Level Flags:
  - kafka-entities-enabled: Master switch
  - kafka-msk-streams-enabled: MSK streams support
  - kafka-confluent-enabled: Confluent support

Provider-Level Flags:
  - enable-self-managed-kafka: nri-kafka
  - enable-aws-msk: AWS MSK
  - enable-confluent-cloud: Confluent

Feature-Level Flags:
  - kafka-hot-partition-detection: Beta feature
  - kafka-consumer-predictions: Experimental
```

### Configuration Hot Reload

The platform supports configuration updates without restart:

```yaml
Hot Reload Process:
  1. Update S3 file
  2. Update Zookeeper path
  3. Publisher detects change
  4. Publishes to Kafka
  5. Services consume update
  6. Apply new configuration

Reload Time:
  - Detection: <1 second
  - Distribution: 5-10 seconds
  - Application: 10-30 seconds
  - Full rollout: 1-2 minutes
```

### Multi-Cell Configuration

```yaml
Cell-Specific Configuration:
  - Cell name in identifier
  - Regional S3 buckets
  - Local Zookeeper clusters
  - Cross-cell replication

Global Configuration:
  - Shared S3 bucket
  - Replicated Zookeeper
  - Global feature flags
  - Consistent versions
```

### Configuration Monitoring

```yaml
Key Metrics:
  - Configuration lag
  - Update frequency
  - Error rates
  - Version drift

Alerts:
  - Configuration update failures
  - Version mismatches
  - Publishing errors
  - Validation failures
```

### Best Practices

#### Configuration as Code
```yaml
Principles:
  - All configuration in Git
  - Automated validation
  - PR-based changes
  - Audit trail

Benefits:
  - Version history
  - Rollback capability
  - Change attribution
  - Collaboration
```

#### Testing Strategy
```yaml
Test Levels:
  1. Unit: Individual rules
  2. Integration: Full pipeline
  3. Provider: Each provider
  4. Performance: Load testing
  5. Regression: Existing entities
```

#### Documentation
```yaml
Required Documentation:
  - Change description
  - Impact analysis
  - Testing performed
  - Rollback plan
  - Monitoring plan
```

---

## 12. Testing and Validation Framework {#testing}

### Test Data Structure

Each entity type includes comprehensive test data:

```
entity-types/message-queue-cluster/tests/
├── nri-kafka.json          # Self-managed Kafka
├── aws-msk-polling.json    # AWS MSK CloudWatch
├── aws-msk-streams.json    # AWS MSK Metric Streams
└── confluent-cloud.json    # Confluent Cloud
```

### Test Data Requirements

#### Complete Event Examples
```json
{
  \"eventType\": \"KafkaClusterSample\",
  \"clusterName\": \"test-cluster\",
  \"kafka.version\": \"3.5.0\",
  \"cluster.activeControllerCount\": 1,
  \"cluster.offlinePartitionsCount\": 0,
  \"cluster.underReplicatedPartitions\": 0,
  \"integration.name\": \"nri-kafka\",
  \"integration.version\": \"3.0.0\",
  \"accountId\": 12345678,
  \"timestamp\": 1700000000000
}
```

#### Edge Cases Coverage
```yaml
Test Scenarios:
  - Missing optional attributes
  - Null values
  - Empty strings
  - Maximum values
  - Minimum values
  - Special characters
  - Unicode handling
```

### Validation Layers

#### 1. Syntax Validation
```yaml
Checks:
  - Valid YAML/JSON syntax
  - Proper indentation
  - Correct data types
  - No duplicate keys
```

#### 2. Schema Validation  
```yaml
Rules:
  - Required fields present
  - Field types correct
  - Enum values valid
  - Pattern matching
```

#### 3. Synthesis Validation
```yaml
Tests:
  - Rules produce valid GUIDs
  - Identifiers are unique
  - Conditions match events
  - Tags extract correctly
```

#### 4. Metric Validation
```yaml
Verification:
  - Queries are valid NRQL
  - Event types exist
  - Attributes referenced exist
  - Aggregations appropriate
```

#### 5. Relationship Validation
```yaml
Checks:
  - Source entities exist
  - Target entities exist
  - Relationship types valid
  - TTLs appropriate
```

### Testing Tools

#### Local Testing
```bash
# Run all validations
make validate

# Test specific entity type
make validate-entity TYPE=message-queue-cluster

# Test synthesis rules
make test-synthesis

# Test metric queries
make test-metrics
```

#### Integration Testing
```yaml
Test Environment:
  - Dedicated test account
  - Sample data generators
  - All providers configured
  - Monitoring enabled

Test Process:
  1. Deploy definitions
  2. Send test events
  3. Verify entity creation
  4. Check relationships
  5. Validate metrics
  6. Test dashboards
```

#### Load Testing
```yaml
Scenarios:
  - High partition count (10,000+)
  - Many topics (1,000+)
  - Large consumer groups (100+)
  - Rapid updates
  - Provider switching

Metrics:
  - Entity creation time
  - Synthesis latency
  - Query performance
  - Memory usage
  - Error rates
```

### Test Data Patterns

#### Provider Variations
```json
// Self-managed
{
  \"eventType\": \"KafkaClusterSample\",
  \"clusterName\": \"prod-kafka\",
  \"provider\": \"SELF_MANAGED\"
}

// AWS MSK Polling
{
  \"eventType\": \"AwsMskClusterSample\",
  \"aws.kafka.clusterArn\": \"arn:aws:kafka:us-east-1:123:cluster/test\",
  \"provider.source\": \"cloudwatch\"
}

// AWS MSK Streams
{
  \"eventType\": \"MetricRaw\",
  \"aws.Namespace\": \"AWS/Kafka\",
  \"metricStreamName\": \"test-stream\"
}
```

#### Boundary Testing
```json
{
  \"cluster.activeControllerCount\": 0,    // Unhealthy
  \"cluster.offlinePartitionsCount\": 10,  // Critical
  \"broker.cpuPercent\": 99.9,             // High load
  \"consumer.lag\": 1000000,               // Severe lag
  \"topic\": \"very-long-topic-name-with-special-chars-!@#$%\"
}
```

### Validation Checklist

#### Pre-Deployment
- [ ] All test files present
- [ ] Syntax validation passes
- [ ] Schema validation passes  
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Documentation updated

#### Post-Deployment
- [ ] Entities appear in UI
- [ ] Metrics populate
- [ ] Relationships form
- [ ] Dashboards render
- [ ] No errors in logs
- [ ] Performance acceptable

### Common Validation Errors

#### Synthesis Failures
```yaml
Error: \"No matching synthesis rule\"
Cause: Event doesn't match conditions
Fix: Check eventType and required attributes

Error: \"Invalid identifier\"
Cause: Identifier produces invalid GUID
Fix: Ensure identifier components present

Error: \"Duplicate entity\"
Cause: Multiple rules match same event
Fix: Make conditions more specific
```

#### Metric Query Failures
```yaml
Error: \"Unknown event type\"
Cause: FROM clause has wrong event
Fix: Verify event type spelling

Error: \"Attribute not found\"
Cause: SELECT references missing field
Fix: Check attribute names in test data

Error: \"Invalid aggregation\"
Cause: Wrong function for data type
Fix: Use appropriate aggregation
```

#### Relationship Failures
```yaml
Error: \"Source entity not found\"
Cause: Source GUID doesn't exist
Fix: Ensure source entity created first

Error: \"Invalid relationship type\"
Cause: Unknown relationship type
Fix: Use standard relationship types

Error: \"Circular relationship\"
Cause: Entity relates to itself
Fix: Check identifier construction
```

### Test Automation

#### CI/CD Pipeline
```yaml
stages:
  - validate:
      - Syntax check
      - Schema validation
      - Lint rules
      
  - test:
      - Unit tests
      - Synthesis tests
      - Metric tests
      
  - integration:
      - Deploy to test
      - Run test suite
      - Validate results
      
  - deploy:
      - Update S3
      - Notify publisher
      - Monitor rollout
```

#### Automated Test Generation
```python
def generate_test_data(entity_type, provider):
    \"\"\"Generate test data for entity type and provider\"\"\"
    base_event = get_base_event(entity_type, provider)
    variations = []
    
    # Normal case
    variations.append(base_event)
    
    # Edge cases
    for field in get_optional_fields(entity_type):
        # Missing field
        variant = copy.deepcopy(base_event)
        del variant[field]
        variations.append(variant)
        
        # Null field
        variant = copy.deepcopy(base_event)
        variant[field] = None
        variations.append(variant)
    
    return variations
```

### Debugging Test Failures

#### Enable Debug Logging
```yaml
Log Levels:
  - DEBUG: All processing details
  - TRACE: Include event contents
  - PROFILE: Add timing information
```

#### Common Debug Queries
```sql
-- Check if events arriving
SELECT count(*) FROM KafkaClusterSample
WHERE clusterName = 'test-cluster'
SINCE 10 minutes ago

-- Verify synthesis working
SELECT latest(entity.type), latest(entity.guid)
FROM KafkaClusterSample
WHERE clusterName = 'test-cluster'

-- Check relationship creation
SELECT count(*) FROM Relationship
WHERE source.type = 'MESSAGE_QUEUE_CLUSTER'
AND target.type = 'MESSAGE_QUEUE_BROKER'
SINCE 1 hour ago
```

#### Test Isolation
```yaml
Isolation Strategies:
  - Use unique cluster names
  - Separate test accounts
  - Time-based test data
  - Cleanup after tests
  - Tag test entities
```

---

## 13. Operational Excellence {#operational-excellence}

### Deployment Architecture

```yaml
Deployment Model:
  Platform: Kubernetes
  Pattern: Microservices
  Regions: Multi-region active-active
  Cells: Independent failure domains
```

#### Service Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: entity-synthesis-engine
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: synthesis-engine
        image: entity-synthesis:latest
        resources:
          requests:
            cpu: \"2\"
            memory: \"4Gi\"
          limits:
            cpu: \"4\"
            memory: \"8Gi\"
```

### Monitoring Strategy

#### Key Metrics to Monitor

```yaml
Entity Pipeline Metrics:
  - entity.synthesis.rate: Entities/second
  - entity.synthesis.latency: Processing time
  - entity.synthesis.errors: Failure rate
  - entity.kafka.lag: Pipeline delay

Kafka Entity Metrics:
  - kafka.entities.total: Total count
  - kafka.entities.active: Active entities
  - kafka.entities.expired: Cleanup rate
  - kafka.relationships.total: Connection count

Resource Metrics:
  - cpu.utilization: Service CPU
  - memory.usage: Service memory
  - kafka.consumer.lag: Queue depth
  - elasticsearch.indexing.rate: Storage throughput
```

#### Health Monitoring

```yaml
Health Checks:
  Liveness:
    - Kafka connectivity
    - Elasticsearch connectivity
    - Memory pressure
    
  Readiness:
    - Synthesis rules loaded
    - Kafka consumers ready
    - Cache warmed
    
  Startup:
    - Configuration valid
    - Dependencies available
    - Initial state loaded
```

#### SLIs and SLOs

```yaml
Service Level Indicators:
  - Entity Creation Latency:
      SLO: 95% < 30 seconds
      Measurement: Time from event to entity
      
  - Entity Availability:
      SLO: 99.9% uptime
      Measurement: API success rate
      
  - Relationship Accuracy:
      SLO: 99% correct
      Measurement: Validation checks

Service Level Objectives:
  - Monthly uptime: 99.9%
  - Entity processing: 1M/hour
  - API latency p99: <100ms
  - Dashboard load time: <2s
```

### Capacity Planning

#### Sizing Guidelines

```yaml
Small Deployment (Dev/Test):
  - Clusters: 1-5
  - Brokers: 3-15
  - Topics: 10-100
  - Partitions: 100-1000
  
Medium Deployment:
  - Clusters: 5-20
  - Brokers: 15-100
  - Topics: 100-1000
  - Partitions: 1000-10000
  
Large Deployment:
  - Clusters: 20+
  - Brokers: 100+
  - Topics: 1000+
  - Partitions: 10000+
```

#### Resource Requirements

```yaml
Per 1000 Entities/Hour:
  CPU: 0.5 cores
  Memory: 1 GB
  Network: 10 Mbps
  Storage: 100 MB/day

Elasticsearch Sizing:
  - 1 KB per entity document
  - 3x replication factor
  - 30-day retention
  - 20% overhead

Kafka Sizing:
  - 100 bytes per event
  - 7-day retention
  - 3x replication
  - Compression enabled
```

### Performance Tuning

#### Synthesis Optimization

```yaml
Batch Processing:
  - Batch size: 1000 events
  - Timeout: 5 seconds
  - Parallelism: CPU cores * 2

Caching Strategy:
  - Entity cache: 1 hour TTL
  - Rule cache: 24 hour TTL
  - Metric cache: 5 minute TTL
```

#### Query Optimization

```sql
-- Efficient cluster query
SELECT latest(activeControllerCount)
FROM KafkaClusterSample
WHERE entity.guid = 'xxx'  -- Use index
SINCE 5 minutes ago       -- Limit time range

-- Avoid expensive operations
-- Bad: SELECT count(*) FROM KafkaPartitionSample
-- Good: SELECT uniqueCount(partition) FROM KafkaTopicSample
```

#### Indexing Strategy

```yaml
Elasticsearch Optimization:
  - Sharding: 1 shard per 50GB
  - Replicas: 1-2 for redundancy
  - Refresh interval: 30s
  - Merge policy: Tiered
  
Index Templates:
  - Dynamic mapping: false
  - Field limits: 1000
  - Analyzed fields: Minimal
  - Doc values: Enabled
```

### Operational Procedures

#### Startup Sequence

```yaml
1. Configuration Loading:
   - Load from Zookeeper
   - Validate definitions
   - Initialize caches
   
2. Connection Establishment:
   - Kafka consumers
   - Elasticsearch clients
   - Redis connections
   
3. State Recovery:
   - Load checkpoint
   - Resume from offset
   - Warm caches
   
4. Processing Start:
   - Begin synthesis
   - Start metrics
   - Enable health checks
```

#### Shutdown Sequence

```yaml
1. Graceful Shutdown:
   - Stop accepting new work
   - Finish in-flight processing
   - Commit Kafka offsets
   
2. State Persistence:
   - Save checkpoints
   - Flush caches
   - Close connections
   
3. Cleanup:
   - Release resources
   - Log statistics
   - Signal completion
```

#### Maintenance Windows

```yaml
Planned Maintenance:
  - Duration: 2 hours
  - Frequency: Monthly
  - Impact: Read-only mode
  
Tasks:
  - Elasticsearch optimization
  - Kafka rebalancing
  - Definition updates
  - Performance tuning
  
Communication:
  - 1 week notice
  - Status page updates
  - Slack notifications
```

### Incident Response

#### Severity Levels

```yaml
SEV1 - Critical:
  - Complete outage
  - Data loss risk
  - Security breach
  Response: 15 minutes
  
SEV2 - Major:
  - Partial outage
  - Performance degradation
  - Feature unavailable
  Response: 30 minutes
  
SEV3 - Minor:
  - Individual entity issues
  - Delayed processing
  - UI problems
  Response: 2 hours
```

#### Runbook Examples

```yaml
High Entity Lag:
  1. Check Kafka consumer lag
  2. Verify synthesis engine health
  3. Check Elasticsearch capacity
  4. Scale if needed
  5. Monitor recovery
  
Synthesis Failures:
  1. Check error logs
  2. Validate definitions
  3. Verify event format
  4. Test with sample data
  5. Deploy fix
```

#### Post-Incident Process

```yaml
Timeline:
  - Immediate: Restore service
  - 24 hours: Initial report
  - 48 hours: Root cause analysis
  - 1 week: Action items
  - 2 weeks: Improvements deployed
  
Documentation:
  - Incident timeline
  - Impact assessment
  - Root cause
  - Remediation steps
  - Prevention measures
```

### Security Operations

#### Access Control

```yaml
Service Accounts:
  - Principle of least privilege
  - Rotate credentials quarterly
  - Audit access logs
  
API Keys:
  - Unique per service
  - Encrypted in transit
  - Monitor usage
  
Network Policies:
  - Restrict by namespace
  - Whitelist only required
  - Enable network encryption
```

#### Compliance

```yaml
Data Handling:
  - PII identification
  - Encryption at rest
  - Audit trails
  - Retention policies
  
Certifications:
  - SOC 2 Type II
  - ISO 27001
  - GDPR compliant
  - HIPAA ready
```

### Cost Optimization

#### Resource Efficiency

```yaml
Optimization Strategies:
  - Right-size instances
  - Use spot instances for batch
  - Implement auto-scaling
  - Archive old data
  
Cost Monitoring:
  - Tag all resources
  - Daily cost reports
  - Budget alerts
  - Quarterly reviews
```

#### Data Lifecycle

```yaml
Retention Policies:
  - Hot data: 7 days (SSD)
  - Warm data: 30 days (HDD)
  - Cold data: 90 days (Archive)
  - Compliance: 7 years (Glacier)
  
Optimization:
  - Compress old data
  - Downsample metrics
  - Aggregate summaries
  - Delete test data
```

---

## 14. Troubleshooting and Debugging {#troubleshooting}

### Common Issues and Solutions

#### Entities Not Appearing

**Symptoms**:
- Kafka metrics flowing but no entities in UI
- Entity count remains zero
- Search returns no results

**Diagnostic Steps**:
```sql
-- 1. Check if raw events arriving
SELECT count(*) FROM KafkaClusterSample 
WHERE clusterName IS NOT NULL 
SINCE 1 hour ago

-- 2. Verify synthesis is working
SELECT latest(entity.guid), latest(entity.type)
FROM KafkaClusterSample
WHERE clusterName = 'your-cluster'
SINCE 1 hour ago

-- 3. Check for synthesis errors
SELECT count(*) FROM NrIntegrationError
WHERE category = 'EntitySynthesis'
SINCE 1 hour ago
```

**Common Causes**:
1. **Missing Required Attributes**
   ```yaml
   Required: eventType, clusterName, accountId
   Check: All present and non-null
   ```

2. **Wrong Event Type**
   ```yaml
   Expected: KafkaClusterSample
   Actual: KafkaCluster or kafka.cluster
   Fix: Ensure exact match
   ```

3. **Feature Flag Disabled**
   ```yaml
   Check: kafka-entities-enabled = true
   Location: Feature flag service
   ```

4. **Synthesis Rule Mismatch**
   ```yaml
   Verify: Conditions match your events
   Debug: Enable TRACE logging
   ```

#### Wrong Provider Type

**Symptoms**:
- AWS MSK showing as SELF_MANAGED
- Confluent showing as generic Kafka
- Missing provider-specific features

**Diagnostic Steps**:
```sql
-- Check provider tag
SELECT latest(provider), count(*)
FROM KafkaClusterSample, AwsMskClusterSample
WHERE entity.guid = 'your-entity-guid'
FACET eventType
```

**Solutions**:
1. **AWS MSK Polling**
   ```yaml
   Required conditions:
     - eventType: AwsMskClusterSample
     - provider.source: cloudwatch
   ```

2. **AWS MSK Streams**
   ```yaml
   Required conditions:
     - eventType: MetricRaw
     - aws.Namespace: AWS/Kafka
     - metricStreamName: present
   ```

3. **Confluent Cloud**
   ```yaml
   Required conditions:
     - eventType: ConfluentCloudClusterSample
     - confluent.kafka.cluster.id: present
   ```

#### Missing Relationships

**Symptoms**:
- No lines in service map
- Topics not connected to clusters
- Applications not linked to Kafka

**Diagnostic Queries**:
```sql
-- Check if both entities exist
SELECT count(*) FROM Entity
WHERE type IN ('MESSAGE_QUEUE_CLUSTER', 'MESSAGE_QUEUE_TOPIC')
AND tags.kafka.cluster.name = 'your-cluster'

-- Verify relationship events
SELECT count(*) FROM Relationship
WHERE source.type = 'MESSAGE_QUEUE_CLUSTER'
AND target.type = 'MESSAGE_QUEUE_TOPIC'
SINCE 1 hour ago

-- Check APM spans
SELECT count(*) FROM Span
WHERE messaging.system = 'kafka'
AND span.kind IN ('producer', 'consumer')
SINCE 1 hour ago
```

**Common Issues**:
1. **Identifier Mismatch**
   ```yaml
   Source: clusterName = \"prod-kafka-01\"
   Target: clusterName = \"prod-kafka-1\"
   Fix: Ensure exact match
   ```

2. **TTL Expiration**
   ```yaml
   Dynamic relationships: 15 minutes
   Check: Activity within TTL window
   ```

3. **Missing Span Attributes**
   ```yaml
   Required for APM:
     - messaging.system = \"kafka\"
     - messaging.destination.name or messaging.source.name
     - span.kind = \"producer\" or \"consumer\"
   ```

#### Metric Query Issues

**Symptoms**:
- Golden metrics show \"No data\"
- Dashboards empty
- Incorrect values

**Debug Process**:
```sql
-- 1. Test raw data exists
SELECT count(*) FROM YourEventType
SINCE 1 hour ago

-- 2. Check specific attributes
SELECT uniques(your.attribute.name)
FROM YourEventType
SINCE 1 hour ago

-- 3. Test the actual metric query
-- Remove entity.guid filter first
SELECT latest(`cluster.activeControllerCount`)
FROM KafkaClusterSample
WHERE clusterName = 'your-cluster'
```

**Common Fixes**:
1. **Attribute Name Mismatch**
   ```yaml
   Expected: cluster.activeControllerCount
   Actual: activeControllerCount
   Fix: Use correct attribute name
   ```

2. **Wrong Event Type**
   ```yaml
   Query: FROM KafkaClusterSample
   Actual: AwsMskClusterSample
   Fix: Include all event types
   ```

3. **Time Range Issues**
   ```yaml
   Default: SINCE 60 minutes ago
   Sparse data: Extend to 24 hours
   ```

#### High Cardinality Warnings

**Symptoms**:
- Too many partition entities
- Performance degradation
- Storage alerts

**Analysis**:
```sql
-- Count partition entities
SELECT uniqueCount(partition) as 'Partition Count'
FROM KafkaPartitionSample
WHERE clusterName = 'your-cluster'
SINCE 1 day ago

-- Identify high-partition topics
SELECT uniqueCount(partition) as 'Partitions'
FROM KafkaPartitionSample
WHERE clusterName = 'your-cluster'
FACET topic
SINCE 1 day ago
```

**Mitigation Strategies**:
1. **Shorter TTL**
   ```yaml
   Partition TTL: 4 hours (vs 8 days)
   Reason: Reduce active entity count
   ```

2. **Sampling**
   ```yaml
   Sample rate: 10%
   Implementation: In integration config
   ```

3. **Disable Partition Entities**
   ```yaml
   Feature flag: kafka-partition-entities-enabled = false
   Impact: No partition-level monitoring
   ```

### Advanced Debugging

#### Enable Debug Logging

```yaml
Service Configuration:
  LOG_LEVEL: DEBUG
  LOG_FORMAT: json
  LOG_DESTINATION: stdout
  
Specific Components:
  SYNTHESIS_LOG_LEVEL: TRACE
  RELATIONSHIP_LOG_LEVEL: DEBUG
  METRICS_LOG_LEVEL: INFO
```

#### Trace Entity Creation

```json
// Debug log example
{
  \"timestamp\": \"2024-01-15T10:30:00Z\",
  \"level\": \"DEBUG\",
  \"service\": \"entity-synthesis-engine\",
  \"message\": \"Processing entity\",
  \"context\": {
    \"eventType\": \"KafkaClusterSample\",
    \"identifier\": \"prod-kafka\",
    \"guid\": \"MTIzNDU2Nzg...\",
    \"rules_evaluated\": 3,
    \"rules_matched\": 1,
    \"tags_extracted\": 5
  }
}
```

#### Performance Profiling

```yaml
Profiling Metrics:
  - synthesis.rule.evaluation.time
  - entity.creation.time
  - relationship.discovery.time
  - metric.query.execution.time
  
Enable Profiling:
  ENABLE_PROFILING: true
  PROFILE_SAMPLE_RATE: 0.1
```

### Query Debugging Patterns

#### Entity Exploration
```sql
-- Find all Kafka entities
SELECT uniques(entity.type), count(*)
FROM Event
WHERE entity.type LIKE 'MESSAGE_QUEUE%'
FACET entity.type
SINCE 1 day ago

-- Inspect entity tags
SELECT latest(tags)
FROM Event
WHERE entity.guid = 'your-entity-guid'
SINCE 1 hour ago
```

#### Relationship Analysis
```sql
-- Trace relationship creation
SELECT timestamp, source.name, target.name, relationshipType
FROM Relationship
WHERE source.type = 'MESSAGE_QUEUE_CLUSTER'
OR target.type = 'MESSAGE_QUEUE_CLUSTER'
SINCE 1 hour ago
LIMIT 100
```

#### Metric Validation
```sql
-- Compare provider metrics
SELECT 
  latest(cluster.activeControllerCount) as 'nri-kafka',
  latest(aws.kafka.ActiveControllerCount) as 'aws-msk'
FROM KafkaClusterSample, AwsMskClusterSample
WHERE entity.guid = 'your-guid'
```

### Platform-Specific Debugging

#### Entity Pipeline Issues
```yaml
Check Points:
  1. entity-ingest: Receipt confirmation
  2. entity-deduplicator: Dedup window
  3. entity-merger: State accumulation
  4. entity-synthesis-engine: Rule application
  5. core-entity-indexer: Storage
  
Logs Location:
  - Kubernetes: kubectl logs -n entity-platform
  - Centralized: Logs UI with service filter
```

#### Kafka Pipeline Health
```bash
# Check consumer lag
kafka-consumer-groups --bootstrap-server kafka:9092 \\
  --group entity-synthesis --describe

# Verify topic messages
kafka-console-consumer --bootstrap-server kafka:9092 \\
  --topic entity-raw --from-beginning --max-messages 10
```

#### Elasticsearch Queries
```bash
# Check entity document
curl -X GET \"elasticsearch:9200/entities/_doc/ENTITY_GUID\"

# Search for entities
curl -X POST \"elasticsearch:9200/entities/_search\" -d '{
  \"query\": {
    \"match\": {
      \"type\": \"MESSAGE_QUEUE_CLUSTER\"
    }
  }
}'
```

### Emergency Procedures

#### Entity Cleanup
```yaml
When needed:
  - Incorrect entities created
  - Testing artifacts
  - Migration cleanup
  
Process:
  1. Identify entities to remove
  2. Stop synthesis for type
  3. Delete from Elasticsearch
  4. Clear caches
  5. Re-enable synthesis
```

#### Force Refresh
```yaml
Symptoms:
  - Stale data
  - Cache issues
  - Update not appearing
  
Steps:
  1. Clear Redis cache
  2. Restart synthesis engine
  3. Reindex from Kafka
  4. Verify updates
```

#### Rollback Procedures
```yaml
Definition Rollback:
  1. Identify problematic version
  2. Revert S3 to previous
  3. Update Zookeeper path
  4. Monitor republishing
  5. Verify entity health
```

---

## 15. Performance Optimization {#performance-optimization}

### Entity Pipeline Performance

#### Optimization Targets

```yaml
Key Metrics:
  - Entity Creation Latency: <30 seconds
  - Synthesis Throughput: >10k entities/second
  - Query Response Time: <100ms p99
  - Dashboard Load Time: <2 seconds
```

#### Batch Processing Optimization

```yaml
Optimal Batch Sizes:
  - Kafka consumption: 5000 events
  - Synthesis processing: 1000 entities
  - Elasticsearch indexing: 500 documents
  - Relationship creation: 2000 relationships

Tuning Parameters:
  kafka.consumer:
    max.poll.records: 5000
    fetch.min.bytes: 1048576
    fetch.max.wait.ms: 500
  
  synthesis.engine:
    batch.size: 1000
    batch.timeout: 5s
    parallelism: ${CPU_COUNT * 2}
  
  elasticsearch.bulk:
    actions: 500
    size: 10mb
    timeout: 30s
```

#### Memory Management

```yaml
JVM Settings:
  -Xmx8g
  -Xms8g
  -XX:MaxDirectMemorySize=2g
  -XX:+UseG1GC
  -XX:MaxGCPauseMillis=200
  -XX:G1ReservePercent=25

Memory Allocation:
  - Heap: 8GB
  - Direct: 2GB
  - OS Cache: 4GB
  - Total: 14GB per instance
```

### Query Performance

#### NRQL Optimization

```sql
-- Efficient: Use entity.guid index
SELECT latest(activeControllerCount)
FROM KafkaClusterSample
WHERE entity.guid = 'xxx'
SINCE 5 minutes ago

-- Inefficient: Full scan
SELECT latest(activeControllerCount)
FROM KafkaClusterSample
WHERE clusterName = 'prod'
SINCE 7 days ago

-- Optimized aggregation
SELECT sum(bytesInPerSec)
FROM KafkaBrokerSample
WHERE entity.guid IN ('xxx', 'yyy', 'zzz')
TIMESERIES 1 minute
SINCE 1 hour ago
```

#### Index Usage

```yaml
Indexed Fields:
  - entity.guid (primary key)
  - entity.type
  - accountId
  - timestamp
  - Golden tags

Query Patterns:
  - Always filter by entity.guid when possible
  - Use time constraints
  - Limit result sets
  - Avoid wildcards in WHERE clause
```

#### Caching Strategy

```yaml
Cache Layers:
  1. CDN Cache:
     - Dashboard JSON: 5 minutes
     - Static assets: 24 hours
     
  2. API Cache:
     - Entity details: 1 minute
     - Metric queries: 30 seconds
     - Relationship graphs: 5 minutes
     
  3. Application Cache:
     - Synthesis rules: 24 hours
     - Entity metadata: 1 hour
     - Query results: 30 seconds
     
  4. Database Cache:
     - Elasticsearch query: 60 seconds
     - Field data: Unlimited
```

### Elasticsearch Optimization

#### Index Settings

```json
{
  \"settings\": {
    \"number_of_shards\": 5,
    \"number_of_replicas\": 1,
    \"refresh_interval\": \"30s\",
    \"index.translog.durability\": \"async\",
    \"index.translog.sync_interval\": \"30s\",
    \"index.merge.policy.segments_per_tier\": 5,
    \"index.merge.policy.max_merged_segment\": \"5gb\"
  },
  \"mappings\": {
    \"dynamic\": false,
    \"properties\": {
      \"entity.guid\": {
        \"type\": \"keyword\",
        \"index\": true
      },
      \"entity.type\": {
        \"type\": \"keyword\",
        \"index\": true
      },
      \"tags\": {
        \"type\": \"object\",
        \"enabled\": false
      }
    }
  }
}
```

#### Shard Strategy

```yaml
Shard Sizing:
  - Target size: 30-50 GB per shard
  - Max documents: 2 billion per shard
  - Calculation: Total data / 40 GB = shard count

Shard Distribution:
  - Cluster entities: 5 shards
  - Broker entities: 5 shards
  - Topic entities: 10 shards
  - Partition entities: 20 shards (high volume)
```

### Kafka Optimization

#### Topic Configuration

```yaml
Entity Topics:
  entity-raw:
    partitions: 100
    replication: 3
    retention.ms: 604800000  # 7 days
    compression.type: lz4
    min.insync.replicas: 2
    
  entity-synthesized:
    partitions: 50
    replication: 3
    retention.ms: 172800000  # 2 days
    compression.type: snappy
```

#### Consumer Optimization

```yaml
Consumer Settings:
  - enable.auto.commit: false
  - isolation.level: read_committed
  - max.poll.interval.ms: 300000
  - session.timeout.ms: 45000
  - heartbeat.interval.ms: 15000
  
Processing Strategy:
  - Manual offset management
  - Batch commit after processing
  - Parallel processing within batch
  - Circuit breaker for failures
```

### High Cardinality Optimization

#### Partition Entity Management

```yaml
Problem: 100k+ partition entities
Impact: Memory pressure, query slowdown

Solutions:
  1. Reduced TTL:
     - Standard: 8 days
     - Partitions: 4 hours
     
  2. Sampling:
     - Sample rate: 10%
     - Key partitions: 100%
     
  3. Aggregation:
     - Store summary at topic level
     - Query details on demand
     
  4. Sharding:
     - Separate index for partitions
     - Dedicated query path
```

#### Consumer Group Scaling

```yaml
Problem: Thousands of consumer groups
Impact: Relationship explosion

Optimization:
  1. Group filtering:
     - Ignore test groups
     - Focus on production
     
  2. Relationship batching:
     - Batch by topic
     - Single relationship per topic
     
  3. Summary entities:
     - Consumer group summaries
     - Aggregated metrics
```

### Dashboard Performance

#### Widget Optimization

```yaml
Query Guidelines:
  1. Time range:
     - Default: 1 hour
     - Maximum: 24 hours
     
  2. Data points:
     - Line charts: <1000 points
     - Tables: <100 rows
     - Heatmaps: <10000 cells
     
  3. Faceting:
     - Limit: 100 unique values
     - Use LIMIT clause
     
  4. Aggregation:
     - Pre-aggregate when possible
     - Use summary metrics
```

#### Loading Strategy

```yaml
Progressive Loading:
  1. Critical metrics first
  2. Above-fold widgets
  3. Secondary pages lazy
  4. Details on demand

Caching:
  - Widget results: 30 seconds
  - Dashboard structure: 5 minutes
  - User preferences: Session
```

### Scaling Strategies

#### Horizontal Scaling

```yaml
Scalable Components:
  - entity-synthesis-engine: 1-20 instances
  - core-entity-indexer: 1-10 instances
  - ep-next-gen-api: 5-50 instances
  
Scaling Triggers:
  - CPU > 70%
  - Memory > 80%
  - Queue depth > 10k
  - Latency > SLO
```

#### Vertical Scaling

```yaml
When to Scale Up:
  - Consistent CPU pressure
  - Memory constraints
  - Large batch processing
  
Instance Sizes:
  - Small: 2 CPU, 4GB RAM
  - Medium: 4 CPU, 8GB RAM
  - Large: 8 CPU, 16GB RAM
  - XLarge: 16 CPU, 32GB RAM
```

### Cost-Performance Balance

#### Resource Efficiency

```yaml
Optimization Areas:
  1. Right-sizing:
     - Monitor actual usage
     - Downsize overprovisioned
     
  2. Spot instances:
     - Batch processing
     - Non-critical workloads
     
  3. Reserved capacity:
     - Predictable workloads
     - Long-term savings
     
  4. Data lifecycle:
     - Archive old entities
     - Compress large fields
```

#### Performance Budget

```yaml
Latency Budget:
  - API Gateway: 10ms
  - Service Processing: 50ms
  - Database Query: 30ms
  - Network: 10ms
  - Total: <100ms

Resource Budget:
  - CPU per 1k entities/hour: 0.5 cores
  - Memory per 1M entities: 1GB
  - Storage per 1M entities: 1GB
  - Network per 1M entities: 100MB
```

---

## 16. Best Practices and Design Patterns {#best-practices}

### Entity Design Best Practices

#### Stable Identifiers

```yaml
Good Practices:
  ✓ Use immutable attributes
  ✓ Include account/region for uniqueness
  ✓ Handle special characters
  ✓ Validate identifier components

Bad Practices:
  ✗ Using display names as identifiers
  ✗ Timestamp in identifier
  ✗ Mutable attributes
  ✗ User-provided strings without validation

Examples:
  Good: arn:aws:kafka:us-east-1:123456:cluster/prod/abc-123
  Good: confluent.kafka.cluster.id = \"lkc-12345\"
  Bad: clusterName = \"Production Kafka (NEW)\"
  Bad: clusterId = \"kafka-${timestamp}\"
```

#### Tag Design

```yaml
Golden Tag Selection:
  - Searchability: Most commonly searched attributes
  - Stability: Rarely changing values
  - Cardinality: Reasonable number of unique values
  - Business value: Meaningful to users

Tag TTL Strategy:
  Static tags (no TTL):
    - provider
    - cloud.region
    - kafka.version
    
  Semi-dynamic tags (hours/days):
    - broker.count: P1D
    - topic.count: P1D
    - health.status: P1H
    
  Dynamic tags (minutes):
    - consumer.lag: P5M
    - throughput.rate: P1M
    - active.connections: P5M
```

#### Entity Expiration

```yaml
TTL Guidelines:
  Infrastructure entities: 8-30 days
    - Clusters: 30 days
    - Brokers: 8 days
    - Topics: 8 days
    
  High-cardinality entities: 1-4 hours
    - Partitions: 4 hours
    - Consumer instances: 1 hour
    
  Activity-based entities: 15 minutes - 1 day
    - Producer connections: 15 minutes
    - Consumer connections: 15 minutes
    - Temporary topics: 1 day
```

### Synthesis Pattern Best Practices

#### Multi-Provider Support

```yaml
Pattern: Provider Abstraction
  
implementation:
  # Base rule structure
  - identifier: getIdentifier(provider)
    name: getName(provider)
    conditions: getConditions(provider)
    tags:
      # Common tags
      kafka.cluster.name:
        entityTagName: kafka.cluster.name
      # Provider-specific
      provider:
        value: getProviderType()
      
helpers:
  getIdentifier(provider):
    switch(provider):
      case SELF_MANAGED: return clusterName
      case AWS_MSK: return aws.kafka.clusterArn
      case CONFLUENT: return confluent.kafka.cluster.id
```

#### Fallback Patterns

```yaml
Pattern: Graceful Degradation

tags:
  clusterName:
    entityTagName: kafka.cluster.name
    fallbackAttribute: aws.kafka.clusterName
    fallbackAttribute: displayName
    fallbackAttribute: name
    defaultValue: \"unknown-cluster\"
    
benefits:
  - Handles provider variations
  - Prevents synthesis failures
  - Maintains data quality
```

#### Conditional Synthesis

```yaml
Pattern: Progressive Enhancement

# Basic entity
- identifier: clusterName
  conditions:
    - attribute: eventType
      value: KafkaClusterSample
      
# Enhanced with version info
- identifier: clusterName
  conditions:
    - attribute: eventType
      value: KafkaClusterSample
    - attribute: kafka.version
      present: true
  tags:
    kafka.version:
      entityTagName: kafka.version
      
benefits:
  - Start with minimal data
  - Add features as available
  - No blocking on optional data
```

### Relationship Best Practices

#### Bidirectional Relationships

```yaml
Pattern: Symmetric Relationships

# Forward: Cluster → Topic
- name: clusterContainsTopic
  relationship:
    relationshipType: CONTAINS
    source: cluster
    target: topic
    
# Reverse: Topic → Cluster  
- name: topicContainedByCluster
  relationship:
    relationshipType: CONTAINED_BY
    source: topic
    target: cluster
    
benefits:
  - Complete graph traversal
  - UI flexibility
  - Query optimization
```

#### TTL Alignment

```yaml
Pattern: Coordinated Expiration

rule:
  If entity TTL < relationship TTL:
    Set relationship TTL = entity TTL
    
example:
  Partition entity: 4 hours
  Partition relationships: 4 hours (not 24)
  
benefits:
  - Prevents orphaned relationships
  - Reduces cleanup overhead
  - Maintains consistency
```

#### Relationship Cardinality

```yaml
Pattern: Cardinality Management

strategies:
  One-to-Many:
    - Use single source, multiple targets
    - Example: Cluster → many Brokers
    
  Many-to-Many:
    - Create junction entity if needed
    - Example: Topics ↔ Consumer Groups
    
  High Cardinality:
    - Aggregate when possible
    - Example: Topic → Partition count (not individual)
```

### Metric Design Patterns

#### Provider Agnostic Metrics

```yaml
Pattern: Unified Metrics

goldenMetric:
  title: \"Throughput\"
  unit: BYTES_PER_SECOND
  queries:
    # Each provider maps to same concept
    nriKafka:
      select: \"sum(`broker.bytesInPerSec`)\"
      from: \"KafkaBrokerSample\"
    awsMsk:
      select: \"sum(`aws.kafka.BytesInPerSec.byBroker`)\"
      from: \"AwsMskBrokerSample\"
    confluentCloud:
      select: \"sum(`io.confluent.kafka.server/received_bytes`)\"
      from: \"Metric\"
      
benefits:
  - Consistent dashboards
  - Provider migration support
  - Unified alerting
```

#### Calculated Metrics

```yaml
Pattern: Derived Values

healthScore:
  title: \"Health Score\"
  unit: PERCENTAGE
  queries:
    calculated:
      select: \"
        (
          filter(count(*), WHERE activeControllerCount = 1) +
          filter(count(*), WHERE offlinePartitionsCount = 0) +
          filter(count(*), WHERE underReplicatedPartitions = 0)
        ) / 3 * 100
      \"
      
benefits:
  - Business-aligned metrics
  - Simplified alerting
  - Composite health view
```

#### Metric Aggregation Patterns

```yaml
Pattern: Smart Aggregation

# Sum for additive metrics
throughput:
  select: \"sum(bytesPerSec)\"
  
# Average for resource metrics  
cpuUsage:
  select: \"average(cpuPercent)\"
  
# Max for worst-case metrics
consumerLag:
  select: \"max(lag)\"
  
# Percentile for latency
responseTime:
  select: \"percentile(duration, 95)\"
```

### Dashboard Design Patterns

#### Progressive Disclosure

```yaml
Pattern: Layered Information

Level 1 - Executive Summary:
  - Overall health status
  - Key business metrics
  - Trend indicators
  
Level 2 - Operational View:
  - Component health
  - Performance metrics
  - Resource utilization
  
Level 3 - Engineering Detail:
  - Individual components
  - Debug information
  - Raw metrics
```

#### Responsive Design

```yaml
Pattern: Adaptive Layouts

Desktop (1920px):
  - 4 column layout
  - Multiple charts per row
  - Detailed tables
  
Tablet (1024px):
  - 2 column layout
  - Stacked charts
  - Condensed tables
  
Mobile (480px):
  - Single column
  - Key metrics only
  - Expandable sections
```

#### Context Preservation

```yaml
Pattern: Stateful Navigation

implementation:
  - Save time range selection
  - Preserve filter choices
  - Remember sort order
  - Maintain zoom level
  
benefits:
  - Better user experience
  - Faster investigation
  - Consistent analysis
```

### Error Handling Patterns

#### Graceful Degradation

```yaml
Pattern: Fallback Strategies

metricQuery:
  try:
    - Primary query with all features
  catch:
    - Simplified query without facets
  finally:
    - Basic count query
    
example:
  primary: \"SELECT histogram(latency, 50, 20) FACET service\"
  fallback: \"SELECT average(latency) FACET service\"
  basic: \"SELECT count(*) FROM Transaction\"
```

#### Circuit Breaker

```yaml
Pattern: Failure Protection

implementation:
  states:
    - Closed: Normal operation
    - Open: Fast failure
    - Half-Open: Test recovery
    
  thresholds:
    - Error rate > 50%: Open circuit
    - Success count > 10: Close circuit
    - Timeout: 30 seconds
    
benefits:
  - Prevent cascade failures
  - Fast failure response
  - Automatic recovery
```

#### Retry Strategies

```yaml
Pattern: Smart Retry

strategy:
  - Immediate retry: Network glitches
  - Exponential backoff: Rate limits
  - Circuit breaker: Persistent failures
  
implementation:
  retries: 3
  backoff: [1s, 2s, 4s]
  jitter: ±20%
  
conditions:
  retry: [500, 502, 503, 504]
  no-retry: [400, 401, 403, 404]
```

### Performance Patterns

#### Lazy Loading

```yaml
Pattern: Deferred Computation

implementation:
  - Load visible widgets first
  - Fetch details on expansion
  - Cache computed results
  - Prefetch likely next data
  
example:
  # Initial load
  GET /api/entity/summary
  
  # On expansion
  GET /api/entity/details
  GET /api/entity/relationships
  GET /api/entity/metrics
```

#### Batch Operations

```yaml
Pattern: Efficient Bulk Processing

implementation:
  collect:
    - Accumulate requests
    - Wait for batch size or timeout
    
  process:
    - Single bulk operation
    - Parallel processing
    - Result distribution
    
example:
  instead of:
    for entity in entities:
      create_entity(entity)
      
  use:
    create_entities_bulk(entities)
```

#### Caching Strategies

```yaml
Pattern: Multi-Level Cache

levels:
  1. Browser cache:
     - Static assets: 1 year
     - API responses: 1 minute
     
  2. CDN cache:
     - Dashboards: 5 minutes
     - Metrics: 30 seconds
     
  3. Application cache:
     - Entity data: 1 minute
     - Relationships: 5 minutes
     
  4. Database cache:
     - Query results: 30 seconds
     - Aggregations: 5 minutes
```

### Security Patterns

#### Defense in Depth

```yaml
Pattern: Layered Security

layers:
  1. Network:
     - TLS encryption
     - IP allowlisting
     - DDoS protection
     
  2. Authentication:
     - API keys
     - OAuth tokens
     - Service accounts
     
  3. Authorization:
     - Role-based access
     - Entity-level permissions
     - Operation restrictions
     
  4. Audit:
     - All operations logged
     - Change tracking
     - Compliance reports
```

#### Least Privilege

```yaml
Pattern: Minimal Access

implementation:
  services:
    - Read-only by default
    - Scoped to specific entities
    - Time-limited tokens
    
  users:
    - Role-based access
    - Entity filtering
    - Operation limits
    
example:
  kafka-reader:
    - Can: Read Kafka entities
    - Cannot: Modify entities
    - Cannot: Access other types
```

---

## 17. Platform Integration Points {#platform-integration}

### Integration Architecture

```yaml
Integration Layers:
  1. Data Ingestion:
     - Infrastructure agents
     - Cloud integrations
     - APM agents
     - Custom APIs
     
  2. Stream Processing:
     - Kafka event bus
     - Stream processors
     - Enrichment services
     
  3. Storage Systems:
     - Elasticsearch
     - PostgreSQL
     - Redis
     - S3
     
  4. API Interfaces:
     - GraphQL
     - REST
     - WebSocket
     - gRPC
```

### Entity Platform Services

#### Core Services Integration

```yaml
entity-ingest:
  purpose: Entry point for all entity data
  integration:
    - REST API endpoint
    - Kafka producer
    - Schema validation
    
entity-synthesis-engine:
  purpose: Apply business rules
  integration:
    - Kafka consumer
    - Rule engine
    - External enrichment
    
core-entity-indexer:
  purpose: Store entities
  integration:
    - Kafka consumer
    - Elasticsearch bulk API
    - Shard routing
```

#### Supporting Services

```yaml
entity-definitions-publisher:
  purpose: Distribute configuration
  integration:
    - Zookeeper watcher
    - S3 reader
    - Kafka publisher
    
relationship-orchestration:
  purpose: Batch relationship operations
  integration:
    - PostgreSQL job store
    - Entity API client
    - Scheduled execution
    
entity-audit-trail:
  purpose: Track all changes
  integration:
    - Kafka consumer
    - Audit log storage
    - Compliance APIs
```

### Data Flow Integration

#### Ingestion Pipeline

```mermaid
graph LR
    A[Kafka Integration] --> B[entity-ingest]
    B --> C[Kafka: entity-raw]
    C --> D[entity-merger]
    D --> E[entity-deduplicator]
    E --> F[entity-synthesis-engine]
    F --> G[core-entity-indexer]
    G --> H[Elasticsearch]
```

#### Query Pipeline

```mermaid
graph LR
    A[Client Request] --> B[ep-next-gen-api]
    B --> C{Cache?}
    C -->|Hit| D[Return Cached]
    C -->|Miss| E[Elasticsearch Query]
    E --> F[Process Results]
    F --> G[Update Cache]
    G --> D
```

### APM Integration

#### Distributed Tracing

```yaml
Span Attributes for Kafka:
  Required:
    - span.kind: \"producer\" | \"consumer\"
    - messaging.system: \"kafka\"
    
  Producer Spans:
    - messaging.destination.name: topic
    - messaging.kafka.cluster.id: cluster
    - messaging.kafka.partition: partition
    
  Consumer Spans:
    - messaging.source.name: topic
    - messaging.kafka.consumer.group: group
    - messaging.kafka.offset: offset
```

#### Service Map Integration

```yaml
Entity Relationships from Spans:
  Producer Detection:
    - Source: APM Application Entity
    - Target: Kafka Topic Entity
    - Type: PRODUCES_TO
    
  Consumer Detection:
    - Source: APM Application Entity  
    - Target: Kafka Topic Entity
    - Type: CONSUMES_FROM
    
  Visualization:
    - Service depends on Kafka
    - Data flow direction
    - Latency attribution
```

### Infrastructure Integration

#### Agent Collection

```yaml
nri-kafka Integration:
  Collection Method: JMX polling
  Frequency: 15-60 seconds
  Events Generated:
    - KafkaClusterSample
    - KafkaBrokerSample
    - KafkaTopicSample
    - KafkaPartitionSample
    - KafkaConsumerSample
    - KafkaProducerSample
```

#### Cloud Integrations

```yaml
AWS Integration:
  Services:
    - CloudWatch Metrics
    - Kinesis Data Firehose
    - MSK API
    
  Data Flow:
    CloudWatch -> Polling -> AwsMskClusterSample
    Metric Streams -> Kinesis -> MetricRaw
    
Confluent Integration:
  API Endpoints:
    - Metrics API v2
    - Cluster API
    
  Authentication:
    - API Key
    - API Secret
```

### Alert Integration

#### Alert Conditions

```yaml
Entity-Based Alerts:
  Creation:
    - Select entity type
    - Choose golden metric
    - Set threshold
    - Configure notifications
    
  Examples:
    - Cluster health != \"Healthy\"
    - Offline partitions > 0
    - Consumer lag > 10000
    - Broker CPU > 90%
```

#### Alert Enrichment

```yaml
Context Added:
  - Entity tags
  - Related entities
  - Recent changes
  - Historical trends
  
Notification Payload:
  {
    \"entity\": {
      \"guid\": \"xxx\",
      \"name\": \"prod-kafka\",
      \"type\": \"MESSAGE_QUEUE_CLUSTER\"
    },
    \"violation\": {
      \"metric\": \"offlinePartitionsCount\",
      \"threshold\": 0,
      \"actual\": 5
    },
    \"related\": [
      {\"type\": \"BROKER\", \"count\": 3},
      {\"type\": \"TOPIC\", \"count\": 50}
    ]
  }
```

### Dashboard Integration

#### Entity-Centric Dashboards

```yaml
Automatic Dashboards:
  - Created from dashboard.json
  - Linked to entity type
  - Context variables populated
  - Available in entity UI
  
Custom Dashboards:
  - Import entity context
  - Use entity tags
  - Link to entities
  - Share across teams
```

#### Cross-Entity Navigation

```yaml
Navigation Features:
  - Click broker -> see broker entity
  - View related entities
  - Navigate relationships
  - Drill down to details
  
Implementation:
  - Entity GUIDs in queries
  - Relationship API calls
  - Context preservation
  - Back navigation
```

### API Integration Patterns

#### GraphQL Integration

```graphql
# Query entities
query GetKafkaCluster($guid: EntityGuid!) {
  actor {
    entity(guid: $guid) {
      name
      type
      tags
      goldenMetrics {
        title
        value
        unit
      }
      relationships {
        type
        target {
          entity {
            name
            type
          }
        }
      }
    }
  }
}
```

#### REST API Integration

```yaml
Common Endpoints:
  # Get entity
  GET /v1/entities/{guid}
  
  # Search entities
  GET /v1/entities/search?type=MESSAGE_QUEUE_CLUSTER
  
  # Get relationships
  GET /v1/entities/{guid}/relationships
  
  # Update tags
  PUT /v1/entities/{guid}/tags
```

#### Webhook Integration

```yaml
Entity Events:
  - entity.created
  - entity.updated
  - entity.deleted
  - relationship.created
  - relationship.deleted
  
Webhook Payload:
  {
    \"eventType\": \"entity.created\",
    \"timestamp\": \"2024-01-15T10:00:00Z\",
    \"entity\": {
      \"guid\": \"xxx\",
      \"type\": \"MESSAGE_QUEUE_CLUSTER\",
      \"name\": \"prod-kafka\"
    }
  }
```

### Monitoring Integration

#### Metrics Export

```yaml
Prometheus Format:
  # Entity count by type
  entity_count{type=\"MESSAGE_QUEUE_CLUSTER\"} 42
  
  # Synthesis rate
  entity_synthesis_rate{status=\"success\"} 1000
  
  # API latency
  api_latency_seconds{endpoint=\"/entities\",quantile=\"0.99\"} 0.1
```

#### Log Integration

```yaml
Structured Logging:
  {
    \"timestamp\": \"2024-01-15T10:00:00Z\",
    \"level\": \"INFO\",
    \"service\": \"entity-synthesis-engine\",
    \"entity.guid\": \"xxx\",
    \"entity.type\": \"MESSAGE_QUEUE_CLUSTER\",
    \"message\": \"Entity created successfully\",
    \"duration\": 45
  }
```

### Custom Integration Development

#### SDK Usage

```python
from newrelic_entity_client import EntityClient

client = EntityClient(api_key=\"xxx\")

# Create custom entity
entity = client.create_entity(
    domain=\"INFRA\",
    type=\"MESSAGE_QUEUE_CLUSTER\",
    name=\"custom-kafka\",
    tags={
        \"kafka.cluster.name\": \"custom-kafka\",
        \"provider\": \"CUSTOM\"
    }
)

# Create relationships
client.create_relationship(
    source_guid=entity.guid,
    target_guid=broker_guid,
    type=\"CONTAINS\"
)
```

#### Integration Best Practices

```yaml
Guidelines:
  1. Use standard entity types when possible
  2. Follow naming conventions
  3. Include required golden tags
  4. Set appropriate TTLs
  5. Handle errors gracefully
  6. Implement retry logic
  7. Monitor integration health
  8. Document custom attributes
```

---

## 18. Security and Compliance {#security-compliance}

### Security Architecture

#### Defense in Depth

```yaml
Security Layers:
  1. Network Security:
     - TLS 1.2+ for all communications
     - Private network isolation
     - Web Application Firewall
     - DDoS protection
     
  2. Authentication:
     - API key authentication
     - Service account management
     - Token rotation policies
     - Multi-factor authentication
     
  3. Authorization:
     - Role-based access control (RBAC)
     - Entity-level permissions
     - Operation-specific grants
     - Deny by default
     
  4. Data Protection:
     - Encryption at rest
     - Encryption in transit
     - Key management service
     - Data classification
```

### Access Control

#### RBAC Model

```yaml
Roles:
  kafka-admin:
    permissions:
      - create: all Kafka entities
      - read: all Kafka entities
      - update: all Kafka entities
      - delete: all Kafka entities
      
  kafka-operator:
    permissions:
      - read: all Kafka entities
      - update: configurations
      - create: alerts
      
  kafka-viewer:
    permissions:
      - read: all Kafka entities
      
  developer:
    permissions:
      - read: own application's Kafka entities
      - read: related topics
```

#### Entity-Level Security

```yaml
Access Control Lists:
  entity:
    guid: \"xxx\"
    type: \"MESSAGE_QUEUE_CLUSTER\"
    acl:
      - principal: \"user:john@example.com\"
        permissions: [\"read\", \"update\"]
      - principal: \"role:kafka-admin\"
        permissions: [\"*\"]
      - principal: \"service:monitoring\"
        permissions: [\"read\"]
```

### Data Protection

#### Encryption Standards

```yaml
Encryption at Rest:
  - Algorithm: AES-256-GCM
  - Key rotation: 90 days
  - Key storage: Hardware Security Module
  
Encryption in Transit:
  - Protocol: TLS 1.2+
  - Cipher suites: ECDHE-RSA-AES256-GCM-SHA384
  - Certificate validation: Required
  - Perfect forward secrecy: Enabled
```

#### Sensitive Data Handling

```yaml
PII Classification:
  - Level 1 (Public): Cluster names, metrics
  - Level 2 (Internal): IP addresses, hostnames
  - Level 3 (Confidential): Connection strings
  - Level 4 (Restricted): API keys, passwords

Handling Rules:
  Level 1-2:
    - Standard encryption
    - Normal retention
    
  Level 3-4:
    - Enhanced encryption
    - Masked in logs
    - Reduced retention
    - Access logging
```

### Compliance Framework

#### Regulatory Compliance

```yaml
Certifications:
  - SOC 2 Type II
  - ISO 27001
  - GDPR compliant
  - HIPAA ready
  - PCI DSS Level 1

Compliance Features:
  - Data residency controls
  - Right to erasure (GDPR)
  - Audit logging
  - Data retention policies
  - Privacy by design
```

#### Audit Requirements

```yaml
Audit Events:
  - Entity creation/modification/deletion
  - Access attempts (success/failure)
  - Configuration changes
  - Permission modifications
  - Data exports
  
Audit Log Format:
  {
    \"timestamp\": \"2024-01-15T10:00:00Z\",
    \"actor\": {
      \"type\": \"user\",
      \"id\": \"user@example.com\",
      \"ip\": \"192.168.1.1\"
    },
    \"action\": \"entity.update\",
    \"resource\": {
      \"type\": \"MESSAGE_QUEUE_CLUSTER\",
      \"guid\": \"xxx\"
    },
    \"changes\": {
      \"tags.environment\": {
        \"old\": \"staging\",
        \"new\": \"production\"
      }
    },
    \"result\": \"success\"
  }
```

### Security Monitoring

#### Threat Detection

```yaml
Monitoring Patterns:
  Suspicious Activities:
    - Unusual access patterns
    - Bulk data exports
    - Permission escalation attempts
    - Failed authentication spikes
    
  Automated Responses:
    - Account lockout after failures
    - Alert on anomalous behavior
    - Automatic log collection
    - Incident ticket creation
```

#### Security Metrics

```yaml
Key Security Indicators:
  - Authentication failure rate
  - Authorization denial rate
  - Encryption coverage percentage
  - Vulnerability scan results
  - Patch compliance rate
  - Security training completion
  
Dashboards:
  - Security overview
  - Access patterns
  - Compliance status
  - Vulnerability tracking
```

### Vulnerability Management

#### Scanning and Patching

```yaml
Scanning Schedule:
  - Container images: Every build
  - Dependencies: Daily
  - Infrastructure: Weekly
  - Penetration testing: Quarterly
  
Patch Policy:
  - Critical: 24 hours
  - High: 7 days
  - Medium: 30 days
  - Low: Next release
```

#### Dependency Management

```yaml
Supply Chain Security:
  - Dependency scanning
  - License compliance
  - Version pinning
  - Signed artifacts
  
Tools:
  - Snyk for vulnerabilities
  - OWASP dependency check
  - Container scanning
  - SBOM generation
```

### Incident Response

#### Response Plan

```yaml
Incident Classification:
  SEV1 - Critical Security:
    - Data breach
    - System compromise
    - Active exploitation
    Response: Immediate
    
  SEV2 - High Security:
    - Vulnerability discovered
    - Suspicious activity
    - Failed controls
    Response: 1 hour
    
  SEV3 - Medium Security:
    - Policy violation
    - Configuration drift
    Response: 24 hours
```

#### Response Procedures

```yaml
Incident Workflow:
  1. Detection:
     - Automated alerts
     - Manual report
     - Monitoring trigger
     
  2. Containment:
     - Isolate affected systems
     - Revoke compromised credentials
     - Block malicious IPs
     
  3. Investigation:
     - Collect logs
     - Analyze timeline
     - Identify root cause
     
  4. Remediation:
     - Patch vulnerabilities
     - Update configurations
     - Strengthen controls
     
  5. Recovery:
     - Restore services
     - Verify security
     - Monitor closely
     
  6. Lessons Learned:
     - Document incident
     - Update procedures
     - Train team
```

### Privacy Protection

#### Data Minimization

```yaml
Collection Principles:
  - Collect only necessary data
  - Anonymize where possible
  - Aggregate for privacy
  - Delete when not needed
  
Implementation:
  - No PII in entity names
  - Hash sensitive identifiers
  - Truncate log entries
  - Redact in exports
```

#### User Rights

```yaml
GDPR Rights Implementation:
  Right to Access:
    - Export user's entities
    - Include all metadata
    - Provide in standard format
    
  Right to Erasure:
    - Delete user's entities
    - Remove from backups
    - Update audit logs
    
  Right to Portability:
    - Export in JSON/CSV
    - Include relationships
    - Document schema
```

### Security Best Practices

#### Development Security

```yaml
Secure Coding:
  - Input validation
  - Output encoding
  - Parameterized queries
  - Error handling
  
Code Review:
  - Security checklist
  - Automated scanning
  - Peer review
  - Security team approval
```

#### Operational Security

```yaml
Production Access:
  - Just-in-time access
  - Approval workflow
  - Session recording
  - Automatic revocation
  
Secret Management:
  - Vault integration
  - Rotation automation
  - No hardcoded secrets
  - Encrypted storage
```

#### Security Training

```yaml
Training Program:
  All Staff:
    - Security awareness
    - Phishing recognition
    - Password policies
    - Incident reporting
    
  Developers:
    - Secure coding
    - OWASP Top 10
    - Security testing
    - Threat modeling
    
  Operations:
    - Security tools
    - Incident response
    - Compliance requirements
    - Audit procedures
```

---

## 19. Future-Proofing and Extensibility {#future-proofing}

### Platform Evolution Strategy

#### Extensibility Framework

```yaml
Design Principles:
  1. Plugin Architecture:
     - New entity types via configuration
     - Custom synthesis rules
     - Provider-specific handlers
     - Metric calculators
     
  2. Schema Evolution:
     - Backward compatible changes
     - Version negotiation
     - Graceful degradation
     - Migration tools
     
  3. API Versioning:
     - Semantic versioning
     - Deprecation policy
     - Multiple version support
     - Client libraries
```

### Future Entity Types

#### Emerging Kafka Patterns

```yaml
Stream Processing Entities:
  STREAM_PROCESSOR:
    - Kafka Streams applications
    - ksqlDB queries
    - Flink jobs
    
  SCHEMA_REGISTRY:
    - Schema versions
    - Compatibility checks
    - Evolution tracking
    
  KAFKA_CONNECT:
    - Connectors
    - Tasks
    - Transformations
```

#### Cloud-Native Extensions

```yaml
Serverless Kafka:
  EVENT_STREAM:
    - AWS EventBridge
    - Azure Event Hub
    - Google Pub/Sub
    
  FUNCTION_TRIGGER:
    - Lambda triggers
    - Function bindings
    - Event subscriptions
```

### Technology Roadmap

#### Platform Enhancements

```yaml
Near Term (6 months):
  - GraphQL subscriptions
  - Real-time entity updates
  - Advanced search capabilities
  - ML-based anomaly detection
  
Medium Term (12 months):
  - Entity versioning
  - Time-travel queries
  - Predictive analytics
  - Auto-remediation actions
  
Long Term (24 months):
  - Federated entity platform
  - Multi-cloud native
  - Edge computing support
  - Quantum-safe encryption
```

#### Kafka-Specific Features

```yaml
Planned Capabilities:
  - Automatic capacity planning
  - Partition rebalancing recommendations
  - Consumer lag predictions
  - Topic lifecycle management
  - Cost optimization suggestions
  - Performance profiling
  - Chaos engineering integration
```

### Integration Roadmap

#### Future Integrations

```yaml
Message Queue Platforms:
  - RabbitMQ entities
  - Azure Service Bus
  - Google Pub/Sub
  - Amazon SQS/SNS
  - NATS
  - Apache Pulsar
  
Observability Platforms:
  - OpenTelemetry native
  - Prometheus federation
  - Grafana integration
  - Jaeger tracing
```

#### Protocol Support

```yaml
Upcoming Protocols:
  - gRPC streaming
  - GraphQL subscriptions
  - Server-sent events
  - WebRTC data channels
  - MQTT for IoT
  - AMQP support
```

### Scalability Evolution

#### Next-Generation Architecture

```yaml
Architectural Improvements:
  1. Cell-Free Architecture:
     - Global entity mesh
     - Automatic routing
     - No cell boundaries
     
  2. Edge Processing:
     - Local synthesis
     - Regional aggregation
     - Central coordination
     
  3. Serverless Components:
     - Function-based synthesis
     - Event-driven scaling
     - Cost optimization
```

#### Performance Targets

```yaml
Future Scale Goals:
  - 100M entities active
  - 1M entities/second ingestion
  - <10ms query latency
  - 99.99% availability
  - PB-scale storage
  - Real-time updates
```

### AI/ML Integration

#### Intelligent Monitoring

```yaml
ML Capabilities:
  Anomaly Detection:
    - Baseline learning
    - Deviation alerts
    - Pattern recognition
    
  Predictive Analytics:
    - Failure prediction
    - Capacity forecasting
    - Trend analysis
    
  Auto-Remediation:
    - Problem diagnosis
    - Solution recommendation
    - Automated fixes
```

#### Natural Language Interface

```yaml
Conversational Monitoring:
  Queries:
    \"Show me unhealthy Kafka clusters\"
    \"Why is consumer lag increasing?\"
    \"Predict tomorrow's throughput\"
    
  Actions:
    \"Scale up the production cluster\"
    \"Create an alert for high lag\"
    \"Rebalance topic partitions\"
```

### Migration Strategies

#### Version Migration

```yaml
Smooth Transitions:
  Entity Definition Updates:
    1. Deploy new version
    2. Dual-write period
    3. Migrate readers
    4. Deprecate old version
    
  Breaking Changes:
    1. Feature flag control
    2. Canary deployment
    3. Rollback capability
    4. Clear communication
```

#### Backward Compatibility

```yaml
Compatibility Guarantees:
  - 2 major versions supported
  - 6 month deprecation notice
  - Migration tools provided
  - Documentation maintained
  
Never Break:
  - Entity GUIDs
  - Core relationships
  - Basic metrics
  - API contracts
```

### Community and Ecosystem

#### Open Standards

```yaml
Standards Adoption:
  - OpenTelemetry for traces
  - CloudEvents for events
  - OpenAPI for APIs
  - AsyncAPI for streaming
  
Contributions:
  - Entity schema standards
  - Kafka best practices
  - Monitoring patterns
  - Integration libraries
```

#### Developer Experience

```yaml
Developer Tools:
  - Entity SDK libraries
  - CLI tools
  - IDE plugins
  - Interactive tutorials
  
Documentation:
  - API references
  - Code examples
  - Video tutorials
  - Community forums
```

### Sustainability

#### Resource Efficiency

```yaml
Green Computing:
  - Carbon footprint tracking
  - Efficient algorithms
  - Optimized storage
  - Smart scheduling
  
Optimization Goals:
  - 50% less CPU usage
  - 40% less memory
  - 60% less network
  - 70% less storage
```

#### Long-Term Maintenance

```yaml
Sustainability Practices:
  - Automated testing
  - Self-healing systems
  - Predictive maintenance
  - Knowledge preservation
  
Technical Debt:
  - Regular refactoring
  - Dependency updates
  - Code simplification
  - Documentation updates
```

---

## 20. Comprehensive Glossary {#glossary}

### A

**Account ID**
- Unique identifier for a New Relic account
- Used in entity GUID generation
- Example: `12345678`

**Active Controller**
- The Kafka broker currently acting as cluster controller
- Exactly one per healthy cluster
- Manages partition leadership and cluster metadata

**Aggregation**
- Process of combining multiple data points
- Common functions: sum, average, max, min
- Used in metric calculations

**Alert**
- Automated notification of a threshold violation
- Can be created on entity golden metrics
- Supports multiple notification channels

**APM (Application Performance Monitoring)**
- Monitoring of application-level metrics and traces
- Source of producer/consumer relationships
- Provides distributed tracing context

**ARN (Amazon Resource Name)**
- Unique identifier for AWS resources
- Format: `arn:aws:service:region:account:resource`
- Used as primary identifier for AWS MSK

**Attribute**
- Key-value pair in telemetry data
- Can be used in synthesis rules
- Example: `kafka.cluster.name = "prod-kafka"`

### B

**Batch Processing**
- Processing multiple items together for efficiency
- Used in synthesis, indexing, and relationship creation
- Improves throughput and reduces overhead

**Billboard Widget**
- Dashboard visualization for single values
- Good for status indicators and KPIs
- Supports thresholds and color coding

**Broker**
- Individual Kafka server in a cluster
- Handles client requests and data storage
- Identified by numeric broker ID

### C

**Cardinality**
- Number of unique values for an attribute
- High cardinality can impact performance
- Example: partition IDs have high cardinality

**Cell**
- Geographic deployment region
- Independent failure domain
- Examples: US-East, EU-Central

**Circuit Breaker**
- Pattern to prevent cascade failures
- Automatically stops calls to failing services
- States: closed, open, half-open

**CloudWatch**
- AWS monitoring service
- Source of MSK metrics via polling
- Higher latency than direct collection

**Cluster**
- Group of Kafka brokers working together
- Top-level entity in Kafka hierarchy
- Contains brokers, topics, and consumer groups

**Confluent Cloud**
- Managed Kafka as a Service
- Provides cloud-native Kafka clusters
- Uses cluster IDs like `lkc-12345`

**Consumer Group**
- Coordinated group of consumers
- Ensures each partition consumed by one member
- Tracks consumption progress (lag)

**Consumer Lag**
- Difference between latest and consumed offset
- Key indicator of consumption health
- Measured in number of messages

**CQRS (Command Query Responsibility Segregation)**
- Separate paths for writes and reads
- Kafka for writes, Elasticsearch for reads
- Enables optimization for each use case

### D

**Dashboard**
- Visual display of metrics and status
- Can be attached to entity types
- Supports multiple pages and widgets

**Dead Letter Queue**
- Queue for messages that fail processing
- Prevents data loss on errors
- Allows manual intervention

**Deduplication**
- Removing duplicate events
- Time-window based
- Ensures exactly-once processing

**Domain**
- High-level categorization of entities
- Examples: INFRA, APM, BROWSER
- Part of entity type hierarchy

### E

**Entity**
- Monitored component with identity
- Has GUID, type, tags, and metrics
- Core abstraction of the platform

**Entity Expiration**
- Automatic cleanup of inactive entities
- Configurable TTL per entity type
- Prevents unbounded growth

**Entity Platform**
- System for managing entities and relationships
- Provides synthesis, storage, and query capabilities
- Foundation for New Relic monitoring

**Entity Synthesis**
- Process of creating entities from telemetry
- Applies rules to match and transform data
- Produces entities with GUIDs and tags

**Event**
- Telemetry data point
- Contains attributes and timestamp
- Source data for entity synthesis

**Event Sourcing**
- Pattern where state changes are events
- Enables replay and audit
- Used in entity platform architecture

**Exactly-Once Processing**
- Guarantee that events processed once
- Achieved through deduplication
- Critical for data accuracy

### F

**Facet**
- Grouping dimension in queries
- Creates breakdown by attribute
- Example: `FACET broker.id`

**Fallback Attribute**
- Alternative attribute when primary missing
- Provides graceful degradation
- Improves synthesis reliability

**Feature Flag**
- Runtime configuration toggle
- Enables gradual rollout
- Can disable features without deployment

### G

**Golden Metrics**
- Most important metrics for entity type
- Appear prominently in UI
- Used for health calculations

**Golden Tags**
- Primary searchable attributes
- Used for filtering and discovery
- Should be stable and meaningful

**GraphQL**
- Query language for APIs
- Flexible data fetching
- Primary API for entity platform

**GUID (Globally Unique Identifier)**
- Unique identifier for entities
- Deterministically generated
- Base64 encoded hash

### H

**Health Status**
- Calculated entity state
- Values: Healthy, Warning, Critical, Unknown
- Based on golden metrics

**Heartbeat**
- Periodic signal indicating liveness
- Used for availability monitoring
- Can trigger entity expiration

**Heatmap**
- Visualization showing density
- Good for distribution analysis
- Uses color intensity

**High Watermark**
- Highest offset in Kafka partition
- Indicates replication progress
- Used for consistency monitoring

### I

**Identifier**
- Unique string within account
- Used to generate entity GUID
- Must be stable and unique

**In-Sync Replicas (ISR)**
- Replicas caught up with leader
- Indicates replication health
- Should match replication factor

**Indexing**
- Storing entities for fast retrieval
- Uses Elasticsearch
- Supports complex queries

**Integration**
- Data collection mechanism
- Examples: nri-kafka, AWS integration
- Produces telemetry events

### J

**JMX (Java Management Extensions)**
- Protocol for monitoring Java applications
- Source of Kafka metrics
- Provides detailed internal metrics

**JSON**
- JavaScript Object Notation
- Data format for APIs and configuration
- Human-readable structure

### K

**Kafka**
- Distributed streaming platform
- Core technology being monitored
- Apache open source project

**Kafka Streams**
- Stream processing library
- Can be monitored as entities
- Produces metrics and traces

**Kinesis**
- AWS streaming service
- Used for metric streams
- Alternative to polling

### L

**Lag**
- Delay in message consumption
- Key performance indicator
- Measured per partition

**Latency**
- Time delay in processing
- Measured at various points
- Critical for performance

**Leader**
- Primary replica for partition
- Handles all reads/writes
- One per partition

**Line Chart**
- Visualization for trends
- Shows metrics over time
- Supports multiple series

### M

**Message Queue**
- System for asynchronous communication
- Kafka is one type
- Decouples producers and consumers

**Metadata**
- Data about data
- Includes tags, properties
- Enriches entity information

**Metric**
- Numerical measurement
- Has value, unit, timestamp
- Core monitoring data

**Metric Streams**
- AWS service for real-time metrics
- Alternative to CloudWatch polling
- Lower latency delivery

**MSK (Managed Streaming for Kafka)**
- AWS managed Kafka service
- Simplified operations
- Integrated with AWS services

**Multi-Cell**
- Deployment across regions
- Provides global coverage
- Independent failure domains

### N

**NRQL (New Relic Query Language)**
- SQL-like query language
- Used for metric queries
- Powers dashboards and alerts

**Node**
- Individual server or broker
- Physical or virtual machine
- Runs Kafka process

### O

**Offset**
- Position in Kafka partition
- Tracks consumption progress
- Monotonically increasing

**Offline Partitions**
- Partitions without leader
- Indicates cluster problems
- Should always be zero

**OpenTelemetry**
- Open standard for observability
- Source of trace data
- Provides span attributes

### P

**Partition**
- Unit of parallelism in Kafka
- Ordered sequence of messages
- Distributed across brokers

**Pipeline**
- Series of processing stages
- Transforms raw data to entities
- Includes dedup, merge, synthesis

**Producer**
- Application sending to Kafka
- Creates messages
- Can be monitored as entity

**Provider**
- Kafka deployment type
- Examples: SELF_MANAGED, AWS_MSK, CONFLUENT_CLOUD
- Determines available features

### Q

**Query**
- Request for data
- Written in NRQL
- Powers metrics and dashboards

**Queue Depth**
- Number of pending messages
- Indicates processing delay
- Monitored for capacity

### R

**Rebalance**
- Redistribution of partitions
- Occurs on consumer changes
- Can cause temporary delays

**Relationship**
- Connection between entities
- Types: CONTAINS, PRODUCES_TO, etc.
- Has TTL and direction

**Replica**
- Copy of partition data
- Provides fault tolerance
- Configured per topic

**Replication Factor**
- Number of partition copies
- Typically 3 for production
- Balances durability and cost

### S

**Schema**
- Structure definition
- For events or entities
- Enables validation

**Shard**
- Elasticsearch partition
- Distributes data
- Enables parallel processing

**Span**
- Distributed trace segment
- Contains timing and metadata
- Source of relationships

**Synthesis Rule**
- Configuration for entity creation
- Matches events to entity types
- Extracts attributes

### T

**Tag**
- Key-value metadata
- Attached to entities
- Used for search and filter

**Telemetry**
- Monitoring data
- Includes metrics, events, logs, traces
- Raw input for entity platform

**Throughput**
- Data processing rate
- Measured in bytes/messages per second
- Key performance metric

**Time Series**
- Data points over time
- Used for trend analysis
- Core data structure

**Topic**
- Logical channel in Kafka
- Contains partitions
- Named message stream

**TTL (Time To Live)**
- Expiration time
- For entities, tags, relationships
- Manages data lifecycle

**Type**
- Entity classification
- Examples: MESSAGE_QUEUE_CLUSTER, MESSAGE_QUEUE_BROKER
- Determines behavior

### U

**Under-Replicated Partitions**
- Partitions with insufficient replicas
- Indicates replication issues
- Should be zero in healthy cluster

**Unit**
- Measurement type
- Examples: COUNT, BYTES_PER_SECOND, PERCENTAGE
- Provides context for metrics

### V

**Validation**
- Checking data correctness
- Applied at multiple stages
- Ensures data quality

**Version**
- Software or schema version
- Enables compatibility
- Tracked for upgrades

**Visualization**
- Graphical representation
- Types: line, bar, table, heatmap
- Makes data understandable

### W

**Widget**
- Dashboard component
- Displays specific visualization
- Configurable with queries

**Workflow**
- Sequence of operations
- Can be automated
- Examples: entity creation, alerting

### Y

**YAML**
- Yet Another Markup Language
- Configuration format
- Human-readable structure

### Z

**Zookeeper**
- Distributed coordination service
- Stores configuration
- Enables leader election

**Zone**
- Availability zone
- Fault isolation boundary
- Part of region

---

## Appendix A: Quick Reference

### Common NRQL Queries

```sql
-- Check cluster health
SELECT latest(activeControllerCount), latest(offlinePartitionsCount)
FROM KafkaClusterSample
WHERE clusterName = 'your-cluster'

-- Monitor consumer lag
SELECT max(consumer.lag) 
FROM KafkaOffsetSample
WHERE consumerGroup = 'your-group'
FACET topic

-- Track throughput
SELECT sum(broker.bytesInPerSec) as 'In', sum(broker.bytesOutPerSec) as 'Out'
FROM KafkaBrokerSample
WHERE clusterName = 'your-cluster'
TIMESERIES AUTO

-- Find unhealthy brokers
SELECT latest(broker.underReplicatedPartitions)
FROM KafkaBrokerSample
WHERE broker.underReplicatedPartitions > 0
FACET broker.id
```

### Entity Type Reference

| Entity Type | Domain | Alertable | Default TTL |
|-------------|---------|-----------|-------------|
| MESSAGE_QUEUE_CLUSTER | INFRA | Yes | 8 days |
| MESSAGE_QUEUE_BROKER | INFRA | Yes | 8 days |
| MESSAGE_QUEUE_TOPIC | INFRA | Yes | 8 days |
| MESSAGE_QUEUE_PARTITION | INFRA | No | 4 hours |
| MESSAGE_QUEUE_CONSUMER_GROUP | INFRA | Yes | 30 days |
| MESSAGE_QUEUE_PRODUCER | INFRA | Yes | 30 days |
| MESSAGE_QUEUE_CONSUMER | INFRA | Yes | 30 days |

### Relationship Type Reference

| Relationship | Source | Target | TTL |
|--------------|---------|---------|-----|
| CONTAINS | Cluster | Broker | 24h |
| CONTAINS | Cluster | Topic | 24h |
| CONTAINS | Topic | Partition | 24h |
| HOSTS | Broker | Partition | 24h |
| CONTAINS | Consumer Group | Consumer | 24h |
| CONSUMES_FROM | Consumer Group | Topic | 15m |
| PRODUCES_TO | Application | Topic | 15m |
| CONSUMES_FROM | Application | Topic | 15m |

### Provider Comparison

| Feature | Self-Managed | AWS MSK | Confluent Cloud |
|---------|--------------|----------|-----------------|
| Full Metrics | ✓ | Partial | Partial |
| Real-time | ✓ | Optional | ✓ |
| Managed | ✗ | ✓ | ✓ |
| Multi-region | Manual | ✓ | ✓ |
| Cost | Infrastructure | Service + CloudWatch | Service |

---

## Appendix B: Troubleshooting Checklist

### Entity Not Appearing

- [ ] Check event type matches exactly
- [ ] Verify all required attributes present
- [ ] Confirm feature flags enabled
- [ ] Check synthesis rule conditions
- [ ] Verify account has entity platform
- [ ] Look for errors in NrIntegrationError

### Missing Relationships

- [ ] Both entities exist
- [ ] Identifiers match exactly
- [ ] Within TTL window
- [ ] Relationship conditions met
- [ ] Required attributes present
- [ ] Check relationship events

### Metric Issues

- [ ] Raw data exists
- [ ] Event type correct
- [ ] Attribute names match
- [ ] Time range appropriate
- [ ] Aggregation function valid
- [ ] Provider query selected

### Performance Problems

- [ ] Check entity count
- [ ] Monitor synthesis rate
- [ ] Verify query efficiency
- [ ] Check cache hit rate
- [ ] Monitor resource usage
- [ ] Review dashboard complexity

---

## Conclusion

This comprehensive guide has covered every aspect of Kafka entity definitions in New Relic's Entity Platform. From fundamental concepts to advanced troubleshooting, from provider-specific implementations to future roadmap, this documentation serves as both a learning resource and operational reference.

The Kafka entity definitions enable sophisticated monitoring capabilities that transform raw Kafka metrics into intelligent, actionable insights. By understanding the concepts and patterns documented here, teams can effectively monitor their Kafka infrastructure, troubleshoot issues quickly, and optimize performance proactively.

As the platform continues to evolve, this guide will serve as a foundation for understanding not just how the system works today, but the principles that will guide its future development. The combination of powerful abstractions, flexible configuration, and comprehensive monitoring capabilities makes the Entity Platform a robust solution for Kafka observability at any scale.

For the latest updates and community discussions, join us at:
- Slack: #entity-platform
- Documentation: [Internal Wiki]
- Support: entity-platform-support@newrelic.com

---

*Last Updated: January 2025*  
*Version: 2.0*  
*Total Length: ~50,000 words*