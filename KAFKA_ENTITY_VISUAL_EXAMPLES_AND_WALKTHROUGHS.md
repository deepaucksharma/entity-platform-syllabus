# Kafka Entity Definitions: Visual Examples and Code Walkthroughs

## Interactive Examples and Real-World Scenarios

This companion guide provides hands-on examples and visual walkthroughs to complement the main wiki.

---

## Example 1: Complete Entity Creation Flow

### Scenario: New Kafka Cluster Deployment

Let's follow data from a freshly deployed Kafka cluster through the entire Entity Platform:

#### Step 1: Raw Telemetry Arrives

```json
// Incoming event from nri-kafka integration
{
  "eventType": "KafkaClusterSample",
  "clusterName": "orders-prod-kafka",
  "kafka.version": "3.5.0",
  "kafka.cluster.id": "MkU3OEVBNTcwNTJENDM2Qk",
  "cluster.activeControllerCount": 1,
  "cluster.offlinePartitionsCount": 0,
  "cluster.underReplicatedPartitions": 0,
  "integration.name": "com.newrelic.kafka",
  "accountId": 12345678,
  "timestamp": 1700000000000
}
```

#### Step 2: Synthesis Rule Matches

```yaml
# This rule from definition.yml matches our event
- identifier: clusterName
  name: clusterName
  encodeIdentifierInGUID: true
  conditions:
    - attribute: eventType
      value: KafkaClusterSample  # ✓ Matches!
```

#### Step 3: Entity Creation

```
Entity Created:
- GUID: MTIzNDU2NzhJTkZSQU1FU1NBR0VfUVVFVUVfQ0xVU1RFUm9yZGVycy1wcm9kLWthZmth
- Domain: INFRA
- Type: MESSAGE_QUEUE_CLUSTER
- Name: orders-prod-kafka
- Tags:
  - kafka.cluster.name: orders-prod-kafka
  - kafka.cluster.id: MkU3OEVBNTcwNTJENDM2Qk
  - kafka.version: 3.5.0
  - provider: SELF_MANAGED
  - integration.type: polling
```

#### Step 4: Golden Metrics Populate

```sql
-- Query executed for activeControllerCount metric
SELECT latest(`cluster.activeControllerCount`)
FROM KafkaClusterSample
WHERE entity.guid = 'MTIzNDU2NzhJTkZSQU1FU1NBR0VfUVVFVUVfQ0xVU1RFUm9yZGVycy1wcm9kLWthZmth'

-- Result: 1 (Healthy!)
```

---

## Example 2: Multi-Provider Entity Handling

### Scenario: Same Kafka Topic Across Providers

Here's how the same logical topic appears differently across providers:

#### Self-Managed Kafka
```json
{
  "eventType": "KafkaTopicSample",
  "clusterName": "prod-kafka",
  "topic": "user-events",
  "topic.partitions": 12,
  "topic.replicationFactor": 3
}
// Creates entity: prod-kafka:user-events
```

#### AWS MSK
```json
{
  "eventType": "AwsMskTopicSample",
  "aws.kafka.clusterArn": "arn:aws:kafka:us-east-1:123456789012:cluster/msk-prod/uuid",
  "aws.kafka.topic": "user-events",
  "aws.kafka.PartitionCount": 12
}
// Creates entity: arn:aws:kafka:us-east-1:123456789012:cluster/msk-prod/uuid:user-events
```

#### Confluent Cloud
```json
{
  "eventType": "ConfluentCloudTopicSample",
  "confluent.kafka.cluster.id": "lkc-abc123",
  "topic": "user-events",
  "resource.kafka.id": "confluent-prod"
}
// Creates entity: lkc-abc123:user-events
```

**Result**: Three different entities, but all have:
- Type: MESSAGE_QUEUE_TOPIC
- Golden tag: kafka.topic.name = "user-events"
- Provider-specific identifiers

---

## Example 3: Relationship Discovery in Action

### Scenario: Application Producing to Kafka

#### Step 1: APM Span Data
```json
{
  "span.kind": "producer",
  "messaging.system": "kafka",
  "messaging.destination.name": "order-events",
  "messaging.kafka.cluster.id": "prod-cluster",
  "entity.guid": "MTIzNDU2NzhBUE1BUFBMSUNBVElPTm9yZGVyLXNlcnZpY2U",
  "entity.name": "order-service",
  "accountId": 12345678
}
```

#### Step 2: Relationship Rule Triggers
```yaml
- name: applicationProducesToTopic
  conditions:
    - attribute: span.kind
      value: producer          # ✓
    - attribute: messaging.system  
      value: kafka            # ✓
```

#### Step 3: Relationship Creation
```
Relationship:
- Type: PRODUCES_TO
- Source: APM|APPLICATION|order-service (existing entity)
- Target: INFRA|MESSAGE_QUEUE_TOPIC|prod-cluster:order-events
- TTL: 15 minutes (behavioral relationship)
```

#### Step 4: Visible in Service Map
```
[order-service] ----PRODUCES_TO----> [order-events topic]
                                           |
                                    CONTAINED_IN
                                           |
                                    [prod-cluster]
```

---

## Example 4: Complex Health Calculation

### Scenario: Cluster Health Degradation

Let's trace how a broker failure affects cluster health:

#### Time T0: Healthy State
```json
{
  "eventType": "KafkaClusterSample",
  "clusterName": "prod-kafka",
  "cluster.activeControllerCount": 1,
  "cluster.offlinePartitionsCount": 0,
  "cluster.underReplicatedPartitions": 0
}
```

**Health Calculation**:
```sql
CASE 
  WHEN latest(activeControllerCount) = 1      -- ✓ True
   AND latest(offlinePartitionsCount) = 0     -- ✓ True
  THEN 'Healthy'                              -- Result!
```

#### Time T1: Broker Fails
```json
{
  "eventType": "KafkaClusterSample",
  "clusterName": "prod-kafka",
  "cluster.activeControllerCount": 1,
  "cluster.offlinePartitionsCount": 3,        -- Changed!
  "cluster.underReplicatedPartitions": 15     -- Changed!
}
```

**Health Calculation**:
```sql
CASE 
  WHEN latest(activeControllerCount) = 1      -- ✓ True
   AND latest(offlinePartitionsCount) = 0     -- ✗ False
  THEN 'Healthy'
  WHEN latest(activeControllerCount) != 1     -- False
   OR latest(offlinePartitionsCount) > 0     -- ✓ True
  THEN 'Critical'                             -- Result!
```

**Entity Update**:
- Health Status: Healthy → Critical
- Summary Metric: "3 Offline Partitions"
- Alert triggered (if configured)

---

## Example 5: Consumer Lag Tracking

### Scenario: Consumer Group Falling Behind

#### Step 1: Consumer Group Entity Creation
```json
{
  "eventType": "KafkaOffsetSample",
  "clusterName": "prod-kafka",
  "consumerGroup": "payment-processor",
  "topic": "payment-events",
  "partition": 0,
  "consumer.lag": 1500
}
```

**Creates/Updates**:
- Entity: MESSAGE_QUEUE_CONSUMER_GROUP "payment-processor"
- Identifier: "prod-kafka:payment-processor"

#### Step 2: Lag Metrics Calculation
```sql
-- Total lag across all partitions
SELECT sum(`consumer.lag`)
FROM KafkaOffsetSample
WHERE entity.guid = '{payment-processor-guid}'
-- Result: 15,000 (across 10 partitions)

-- Max partition lag
SELECT max(`consumer.lag`)
FROM KafkaOffsetSample
WHERE entity.guid = '{payment-processor-guid}'
FACET partition
-- Result: 3,500 (partition 7 is furthest behind)
```

#### Step 3: Lag Trend Analysis
```sql
-- Derivative shows lag growing at 100 messages/minute
SELECT derivative(sum(consumer.lag), 1 minute)
FROM KafkaOffsetSample
WHERE entity.guid = '{payment-processor-guid}'
-- Result: +100 (getting worse!)
```

#### Step 4: Relationship to Topic
```yaml
relationship:
  type: CONSUMES_FROM
  source: payment-processor (consumer group)
  target: payment-events (topic)
  expires: 15 minutes
  metadata:
    totalLag: 15000
    partitionsConsumed: 10
```

---

## Example 6: Dashboard in Action

### Scenario: Investigating Topic Performance

Here's how the dashboard helps diagnose issues:

#### Widget 1: Topic Throughput
```json
{
  "title": "Message Rate by Topic",
  "visualization": "viz.bar",
  "query": "SELECT sum(topic.messagesInPerSec) 
           FROM KafkaTopicSample 
           WHERE clusterName = 'prod-kafka' 
           FACET topic 
           LIMIT 20"
}
```

**Shows**:
```
payment-events:     5,000 msg/sec  ████████████████████
order-events:       3,500 msg/sec  ██████████████
user-events:        2,000 msg/sec  ████████
inventory-updates:    500 msg/sec  ██
```

#### Widget 2: Consumer Lag Heatmap
```json
{
  "visualization": "viz.heatmap",
  "query": "SELECT histogram(consumer.lag, 10, 20) 
           FROM KafkaOffsetSample 
           WHERE clusterName = 'prod-kafka' 
           FACET consumerGroup, topic"
}
```

**Reveals**: payment-processor group has high lag on payment-events topic

#### Widget 3: Broker Resource Usage
```json
{
  "visualization": "viz.line",
  "query": "SELECT average(broker.cpuPercent) as 'CPU %',
                   average(broker.diskUsedPercent) as 'Disk %' 
           FROM KafkaBrokerSample 
           WHERE clusterName = 'prod-kafka' 
           FACET broker.id 
           TIMESERIES AUTO"
}
```

**Shows**: Broker 2 CPU spiking to 95% during peak hours

---

## Example 7: TTL and Lifecycle Management

### Scenario: Partition Scaling Event

Let's see how TTLs affect entity lifecycle during a partition increase:

#### Time T0: Original State
```
Topic: user-events
Partitions: 6
Entities:
- user-events-partition-0 (created 3 hours ago)
- user-events-partition-1 (created 3 hours ago)
- ...
- user-events-partition-5 (created 3 hours ago)
```

#### Time T1: Scale to 12 Partitions
```
New partition entities created:
- user-events-partition-6 (new)
- user-events-partition-7 (new)
- ...
- user-events-partition-11 (new)
```

#### Time T4: Four Hours Later
```yaml
# Partition entity configuration
entityExpirationTime: FOUR_HOURS  # Shorter TTL

# Result:
- Partitions 0-5: Still exist (data within 4 hours)
- Partitions 6-11: Still exist (recently created)
```

#### Time T8: If Scaled Back Down
```
Topic scaled back to 6 partitions
- Partitions 0-5: Still receiving data (entities remain)
- Partitions 6-11: No new data (will expire after 4 hours)
```

**Key Learning**: Short TTLs for high-cardinality entities prevent accumulation of obsolete entities.

---

## Example 8: Error Handling and Edge Cases

### Scenario: Incomplete Data Handling

#### Case 1: Missing Required Attribute
```json
{
  "eventType": "KafkaClusterSample",
  // Missing clusterName!
  "cluster.activeControllerCount": 1
}
```

**Result**: No entity created (identifier required)

#### Case 2: Fallback Attributes
```json
{
  "eventType": "AwsMskBrokerSample",
  "aws.kafka.clusterArn": "arn:aws:kafka...",
  // aws.kafka.clusterName is missing
  "displayName": "msk-prod-cluster"  // But this exists
}
```

```yaml
# Fallback chain in action
aws.kafka.clusterName:
  entityTagName: kafka.cluster.name
  fallbackAttribute: clusterName      # Not found
  fallbackAttribute: displayName      # Found! Used as kafka.cluster.name
```

#### Case 3: Conditional Tag with TTL
```json
{
  "eventType": "ConfluentCloudPartitionSample",
  "partition.id": 0,
  "hotPartition": true,  // Temporary spike
  "timestamp": 1700000000000
}
```

```yaml
tags:
  hotPartition:
    ttl: P1H  # Only retained for 1 hour
```

**Result**: hotPartition=true tag expires after 1 hour if not refreshed

---

## Example 9: Query Pattern Examples

### Common Query Patterns for Different Use Cases

#### 1. Find Unhealthy Clusters
```sql
FROM KafkaClusterSample, AwsMskClusterSample
SELECT latest(clusterName), 
       latest(activeControllerCount),
       latest(offlinePartitionsCount)
WHERE activeControllerCount != 1 
   OR offlinePartitionsCount > 0
SINCE 5 minutes ago
```

#### 2. Top Topics by Volume
```sql
FROM KafkaTopicSample
SELECT sum(topic.bytesInPerSec) as 'Throughput'
FACET topic
SINCE 1 hour ago
LIMIT 10
```

#### 3. Consumer Groups with Growing Lag
```sql
FROM KafkaOffsetSample
SELECT derivative(sum(consumer.lag), 1 minute) as 'Lag Growth Rate'
WHERE derivative(sum(consumer.lag), 1 minute) > 0
FACET consumerGroup
SINCE 30 minutes ago
```

#### 4. Cross-Provider Cluster Count
```sql
SELECT uniqueCount(clusterName) as 'Self-Managed',
       uniqueCount(aws.kafka.clusterArn) as 'AWS MSK',
       uniqueCount(confluent.kafka.cluster.id) as 'Confluent'
FROM KafkaClusterSample, AwsMskClusterSample, ConfluentCloudClusterSample
SINCE 1 day ago
```

---

## Example 10: Building Custom Alerts

### Based on Entity Golden Metrics

#### Alert 1: Cluster Health
```yaml
Name: Kafka Cluster Unhealthy
Query: |
  SELECT latest(activeControllerCount) as 'controllers',
         latest(offlinePartitionsCount) as 'offline'
  FROM KafkaClusterSample
  WHERE entity.guid = '{cluster-guid}'
  
Condition: controllers != 1 OR offline > 0
For: 5 minutes
```

#### Alert 2: Consumer Lag Threshold
```yaml
Name: Consumer Group Lag Critical
Query: |
  SELECT sum(consumer.lag) as 'totalLag'
  FROM KafkaOffsetSample
  WHERE entity.guid = '{consumer-group-guid}'
  
Condition: totalLag > 100000
For: 10 minutes
```

#### Alert 3: Broker Resource Exhaustion
```yaml
Name: Kafka Broker High CPU
Query: |
  SELECT average(broker.cpuPercent) as 'cpu'
  FROM KafkaBrokerSample
  WHERE entity.guid = '{broker-guid}'
  
Condition: cpu > 90
For: 15 minutes
```

---

## Putting It All Together: Complete Monitoring Scenario

### Initial State
- 1 Kafka cluster with 3 brokers
- 10 topics with varying partition counts
- 5 consumer groups
- 3 producer applications

### Entity Creation Flow
1. **43 infrastructure entities** created:
   - 1 cluster
   - 3 brokers  
   - 10 topics
   - ~30 partitions (varies by topic)
   - 5 consumer groups

2. **6 application entities** discovered:
   - 3 producers (from APM spans)
   - 3 consumers (from APM spans)

3. **~25 relationships** formed:
   - Cluster CONTAINS brokers/topics
   - Brokers HOST partitions
   - Consumer groups CONSUME_FROM topics
   - Applications PRODUCE_TO/CONSUME_FROM topics

### Monitoring Dashboard Shows
- Cluster health: GREEN
- Total throughput: 50K messages/sec
- Average broker CPU: 45%
- Max consumer lag: 5,000 messages
- Active relationships in service map

### When Issues Occur
1. Broker 2 fails
2. Cluster health → CRITICAL
3. Offline partitions: 10
4. Under-replicated partitions: 30
5. Consumer lag starts growing
6. Alerts fire based on golden metrics
7. Service map shows impacted data flows

This complete example demonstrates how all components work together to provide comprehensive Kafka monitoring through the Entity Platform.

---

*This visual guide complements the main wiki with practical examples and real-world scenarios.*