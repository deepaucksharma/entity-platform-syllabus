# Metric Definitions

## Overview

This document provides comprehensive definitions of all metrics collected and calculated by the Message Queues monitoring system. Each metric includes its source, calculation method, unit of measurement, and interpretation guidelines.

## Metric Categories

### Metric Taxonomy

```
Kafka Metrics
├── Cluster Metrics
│   ├── Health Indicators
│   ├── Capacity Metrics
│   └── Performance Metrics
├── Broker Metrics
│   ├── Resource Utilization
│   ├── Network Metrics
│   └── Replication Metrics
├── Topic Metrics
│   ├── Throughput Metrics
│   ├── Storage Metrics
│   └── Message Metrics
├── Consumer Metrics
│   ├── Lag Metrics
│   ├── Performance Metrics
│   └── Group Coordination
└── Producer Metrics
    ├── Rate Metrics
    ├── Latency Metrics
    └── Error Metrics
```

## Cluster Metrics

### Health Indicators

#### Active Controllers
- **Metric Name**: `kafka.controller.active.count`
- **Source**: AWS MSK: `ActiveControllerCount`, Confluent: `kafka.controller.active`
- **Type**: Gauge
- **Unit**: Count
- **Expected Value**: 1 (exactly one controller per cluster)
- **Description**: Number of active controllers in the cluster
- **Alert Threshold**: != 1 for > 60 seconds
- **NRQL Query**:
```sql
SELECT latest(provider.activeControllers)
FROM AwsMskClusterSample
WHERE provider.clusterName = '{clusterName}'
```

#### Offline Partitions
- **Metric Name**: `kafka.partition.offline.count`
- **Source**: AWS MSK: `OfflinePartitionsCount`, Confluent: `kafka.partition.offline`
- **Type**: Gauge
- **Unit**: Count
- **Expected Value**: 0
- **Description**: Number of partitions without an active leader
- **Alert Threshold**: > 0 (Critical)
- **Impact**: Data unavailability for affected partitions
- **NRQL Query**:
```sql
SELECT latest(provider.offlinePartitions)
FROM AwsMskClusterSample
WHERE provider.clusterName = '{clusterName}'
```

#### Under-Replicated Partitions
- **Metric Name**: `kafka.partition.underReplicated.count`
- **Source**: AWS MSK: `UnderReplicatedPartitions`, Confluent: `kafka.partition.under_replicated`
- **Type**: Gauge
- **Unit**: Count
- **Expected Value**: 0
- **Description**: Partitions with fewer in-sync replicas than configured
- **Alert Threshold**: 
  - Warning: > 5% of total partitions
  - Critical: > 10% of total partitions
- **NRQL Query**:
```sql
SELECT latest(provider.underReplicatedPartitions)
FROM AwsMskClusterSample
WHERE provider.clusterName = '{clusterName}'
```

### Capacity Metrics

#### Cluster Disk Usage
- **Metric Name**: `kafka.disk.used.percent`
- **Type**: Gauge
- **Unit**: Percentage
- **Calculation**: `(used_space / total_space) * 100`
- **Alert Threshold**:
  - Warning: > 75%
  - Critical: > 85%
- **NRQL Query**:
```sql
SELECT average(provider.kafkaDataLogsDiskUsed)
FROM AwsMskBrokerSample
WHERE provider.clusterName = '{clusterName}'
FACET provider.brokerId
```

#### Partition Count
- **Metric Name**: `kafka.partition.count`
- **Type**: Gauge
- **Unit**: Count
- **Description**: Total number of partitions in the cluster
- **Calculation**: Sum of partitions across all topics
- **NRQL Query**:
```sql
SELECT sum(partitionCount)
FROM (
  SELECT latest(partitionCount)
  FROM KafkaTopicSample
  FACET topicName
)
```

### Performance Metrics

#### Cluster Throughput
- **Metric Name**: `kafka.cluster.throughput.bytes`
- **Type**: Rate
- **Unit**: Bytes/second
- **Components**:
  - `kafka.cluster.throughput.in`: Incoming bytes
  - `kafka.cluster.throughput.out`: Outgoing bytes
- **Calculation**: Sum of all broker throughputs
- **NRQL Query**:
```sql
SELECT 
  sum(average(provider.bytesInPerSec.Average)) as bytesIn,
  sum(average(provider.bytesOutPerSec.Average)) as bytesOut
FROM AwsMskBrokerSample
WHERE provider.clusterName = '{clusterName}'
TIMESERIES 5 minutes
```

## Broker Metrics

### Resource Utilization

#### CPU Usage
- **Metric Name**: `kafka.broker.cpu.percent`
- **Type**: Gauge
- **Unit**: Percentage
- **Components**:
  - `kafka.broker.cpu.user`: User CPU time
  - `kafka.broker.cpu.system`: System CPU time
- **Calculation**: `user + system`
- **Alert Threshold**:
  - Warning: > 75% for 5 minutes
  - Critical: > 90% for 5 minutes
- **NRQL Query**:
```sql
SELECT 
  average(provider.cpuUser) as userCPU,
  average(provider.cpuSystem) as systemCPU,
  average(provider.cpuUser + provider.cpuSystem) as totalCPU
FROM AwsMskBrokerSample
FACET provider.brokerId
```

#### Memory Usage
- **Metric Name**: `kafka.broker.memory.used.percent`
- **Type**: Gauge
- **Unit**: Percentage
- **Calculation**: `(heap_used / heap_max) * 100`
- **Alert Threshold**:
  - Warning: > 80%
  - Critical: > 95%
- **Components**:
  - JVM Heap Memory
  - Off-heap Memory (Direct buffers)

#### Network Utilization
- **Metric Name**: `kafka.broker.network.bytes`
- **Type**: Rate
- **Unit**: Bytes/second
- **Components**:
  - `kafka.broker.network.rx`: Received bytes
  - `kafka.broker.network.tx`: Transmitted bytes
- **NRQL Query**:
```sql
SELECT 
  average(provider.networkRxPackets) * 1024 as rxBytes,
  average(provider.networkTxPackets) * 1024 as txBytes
FROM AwsMskBrokerSample
FACET provider.brokerId
TIMESERIES 5 minutes
```

### Replication Metrics

#### In-Sync Replicas (ISR)
- **Metric Name**: `kafka.isr.shrink.rate`
- **Type**: Rate
- **Unit**: Shrinks/second
- **Description**: Rate at which ISR shrinks (replicas falling out of sync)
- **Alert Threshold**: > 0.1/second sustained
- **Impact**: Potential data loss risk

#### Replication Lag
- **Metric Name**: `kafka.replica.lag.messages`
- **Type**: Gauge
- **Unit**: Messages
- **Description**: Number of messages follower replicas are behind the leader
- **Alert Threshold**: > 1000 messages
- **NRQL Query**:
```sql
SELECT max(provider.replicaLag)
FROM KafkaReplicaSample
WHERE provider.clusterName = '{clusterName}'
FACET provider.topic, provider.partition
```

## Topic Metrics

### Throughput Metrics

#### Topic Bytes In/Out Rate
- **Metric Name**: `kafka.topic.bytes.rate`
- **Type**: Rate
- **Unit**: Bytes/second
- **Components**:
  - `kafka.topic.bytes.in.rate`: Incoming byte rate
  - `kafka.topic.bytes.out.rate`: Outgoing byte rate
- **NRQL Query**:
```sql
SELECT 
  average(provider.bytesInPerSec.Average) as bytesIn,
  average(provider.bytesOutPerSec.Average) as bytesOut
FROM AwsMskTopicSample
WHERE provider.topic = '{topicName}'
TIMESERIES 5 minutes
```

#### Message Rate
- **Metric Name**: `kafka.topic.messages.in.rate`
- **Type**: Rate
- **Unit**: Messages/second
- **Description**: Rate of messages being produced to the topic
- **Calculation**: Based on record count metrics
- **NRQL Query**:
```sql
SELECT rate(sum(provider.messagesInPerSec.Average), 1 second)
FROM AwsMskTopicSample
WHERE provider.topic = '{topicName}'
```

### Storage Metrics

#### Topic Size
- **Metric Name**: `kafka.topic.size.bytes`
- **Type**: Gauge
- **Unit**: Bytes
- **Description**: Total size of all log segments for the topic
- **Calculation**: Sum across all partitions and replicas
- **Growth Rate**: Calculate trend over time

#### Retention Utilization
- **Metric Name**: `kafka.topic.retention.utilization`
- **Type**: Gauge
- **Unit**: Percentage
- **Description**: How close the topic is to retention limits
- **Calculation**: `(current_age / retention_time) * 100` or `(current_size / retention_size) * 100`

## Consumer Metrics

### Lag Metrics

#### Consumer Lag
- **Metric Name**: `kafka.consumer.lag.messages`
- **Type**: Gauge
- **Unit**: Messages
- **Description**: Number of messages consumer is behind the latest offset
- **Alert Threshold**:
  - Warning: > 10,000 messages
  - Critical: > 100,000 messages
- **NRQL Query**:
```sql
SELECT max(consumerLag)
FROM KafkaConsumerSample
WHERE consumerGroup = '{consumerGroup}'
FACET topic, partition
```

#### Lag Trend
- **Metric Name**: `kafka.consumer.lag.trend`
- **Type**: Derivative
- **Unit**: Messages/minute
- **Description**: Rate of change in consumer lag
- **Values**:
  - Positive: Lag increasing (falling behind)
  - Negative: Lag decreasing (catching up)
  - Zero: Stable lag
- **Calculation**: `derivative(lag, 1 minute)`

### Performance Metrics

#### Consumption Rate
- **Metric Name**: `kafka.consumer.records.rate`
- **Type**: Rate
- **Unit**: Records/second
- **Description**: Rate at which consumer is processing records
- **NRQL Query**:
```sql
SELECT rate(sum(recordsConsumed), 1 second)
FROM KafkaConsumerSample
WHERE consumerGroup = '{consumerGroup}'
FACET topic
```

#### Consumer Throughput
- **Metric Name**: `kafka.consumer.bytes.rate`
- **Type**: Rate
- **Unit**: Bytes/second
- **Description**: Rate of bytes consumed
- **Comparison**: Should roughly match producer rate for real-time processing

### Group Coordination

#### Rebalance Rate
- **Metric Name**: `kafka.consumer.rebalance.rate`
- **Type**: Rate
- **Unit**: Rebalances/hour
- **Description**: Frequency of consumer group rebalances
- **Alert Threshold**: > 5 per hour
- **Impact**: Processing interruptions during rebalance

#### Group State
- **Metric Name**: `kafka.consumer.group.state`
- **Type**: Enum
- **Values**: 
  - `Empty`: No members
  - `Stable`: All members connected
  - `PreparingRebalance`: Rebalance starting
  - `CompletingRebalance`: Rebalance in progress
  - `Dead`: Group inactive

## Producer Metrics

### Rate Metrics

#### Producer Request Rate
- **Metric Name**: `kafka.producer.request.rate`
- **Type**: Rate
- **Unit**: Requests/second
- **Description**: Rate of produce requests
- **Components**:
  - Total requests
  - Failed requests
  - Request size distribution

#### Producer Byte Rate
- **Metric Name**: `kafka.producer.bytes.rate`
- **Type**: Rate
- **Unit**: Bytes/second
- **Description**: Rate of bytes being produced
- **NRQL Query**:
```sql
SELECT rate(sum(bytesProduced), 1 second)
FROM KafkaProducerSample
WHERE appName = '{applicationName}'
```

### Latency Metrics

#### Request Latency
- **Metric Name**: `kafka.producer.request.latency.ms`
- **Type**: Timer
- **Unit**: Milliseconds
- **Percentiles**: p50, p75, p95, p99, p999
- **Description**: Time to complete a produce request
- **Alert Threshold**:
  - Warning: p95 > 100ms
  - Critical: p95 > 500ms

#### Record Queue Time
- **Metric Name**: `kafka.producer.record.queue.time.ms`
- **Type**: Timer
- **Unit**: Milliseconds
- **Description**: Time records spend in producer buffer before sending

### Error Metrics

#### Failed Send Rate
- **Metric Name**: `kafka.producer.failed.sends.rate`
- **Type**: Rate
- **Unit**: Failures/second
- **Description**: Rate of failed produce attempts
- **Alert Threshold**: > 1% of total sends

## Calculated Metrics

### Health Score
- **Metric Name**: `kafka.cluster.health.score`
- **Type**: Calculated
- **Range**: 0-100
- **Calculation**:
```typescript
healthScore = 100
if (offlinePartitions > 0) healthScore -= 50
if (underReplicatedPartitions > 0) healthScore -= 20
if (activeControllers !== 1) healthScore -= 30
if (avgBrokerCPU > 80) healthScore -= 10
if (avgBrokerDiskUsage > 80) healthScore -= 10
```

### Availability
- **Metric Name**: `kafka.cluster.availability`
- **Type**: Calculated
- **Unit**: Percentage
- **Calculation**: `((totalPartitions - offlinePartitions) / totalPartitions) * 100`

### Throughput Efficiency
- **Metric Name**: `kafka.throughput.efficiency`
- **Type**: Calculated
- **Unit**: Percentage
- **Description**: Ratio of actual to expected throughput
- **Calculation**: `(actualThroughput / configuredCapacity) * 100`

## Metric Aggregation

### Time-based Aggregations
```sql
-- Hourly aggregation
SELECT 
  average(value) as avg,
  max(value) as max,
  min(value) as min,
  stddev(value) as stddev,
  percentile(value, 95) as p95
FROM MetricSample
SINCE 1 hour ago
TIMESERIES 5 minutes

-- Daily patterns
SELECT 
  average(value) as avg,
  dayOfWeek(timestamp) as day,
  hourOf(timestamp) as hour
FROM MetricSample
SINCE 7 days ago
FACET day, hour
```

### Entity-based Aggregations
```sql
-- Cluster-wide aggregation
SELECT 
  sum(brokerValue) as clusterTotal,
  average(brokerValue) as brokerAverage,
  max(brokerValue) as brokerMax
FROM BrokerMetricSample
WHERE clusterName = '{cluster}'
FACET clusterName

-- Topic distribution
SELECT 
  count(*) as topicCount,
  sum(partitionCount) as totalPartitions,
  average(partitionCount) as avgPartitionsPerTopic
FROM TopicSample
FACET clusterName
```

## Metric Collection Best Practices

### 1. Collection Frequency
- **High-frequency** (10-30s): Critical health metrics
- **Medium-frequency** (1-5m): Performance metrics
- **Low-frequency** (5-15m): Capacity and trend metrics

### 2. Retention Policies
- **Raw metrics**: 7 days
- **5-minute aggregations**: 30 days
- **Hourly aggregations**: 13 months
- **Daily aggregations**: 25 months

### 3. Missing Data Handling
- Use `latest()` for point-in-time values
- Use `filter()` to handle sparse metrics
- Implement gap-filling for continuous metrics
- Alert on missing critical metrics

### 4. Metric Validation
- Range validation for gauges
- Rate limit checks for counters
- Anomaly detection for unusual patterns
- Cross-metric consistency checks

## Conclusion

This comprehensive metric catalog serves as the foundation for monitoring Kafka infrastructure. Understanding these metrics, their relationships, and proper interpretation is crucial for maintaining healthy Kafka deployments.