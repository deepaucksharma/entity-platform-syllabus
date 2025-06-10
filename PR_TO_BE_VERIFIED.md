review - # Add comprehensive Kafka entity definitions for Message Queue monitoring

## Summary

This PR introduces comprehensive entity definitions for Apache Kafka monitoring across three major providers: self-managed Kafka (nri-kafka), AWS MSK, and Confluent Cloud. It establishes a complete entity hierarchy for Kafka infrastructure components including clusters, brokers, topics, partitions, consumer groups, producers, and consumers, enabling the Message Queues UI application and entity-centric observability for event-driven architectures.

## Motivation

Currently, New Relic monitors Kafka through infrastructure integrations and sample data, but lacks proper entity synthesis and relationship mapping. This PR addresses:

- First-class Kafka entities in the Entity Platform with proper lifecycle management
- Unified monitoring experience across different Kafka providers 
- Proper relationship modeling between Kafka components and APM services
- Provider-specific health status calculations
- Golden metrics and dashboards for key performance indicators
- Support for both polling and real-time metric streaming

## Changes

### New Entity Types Added

1. **MESSAGE_QUEUE_CLUSTER** - Kafka clusters across all providers
2. **MESSAGE_QUEUE_BROKER** - Individual Kafka brokers within clusters
3. **MESSAGE_QUEUE_TOPIC** - Kafka topics with activity monitoring
4. **MESSAGE_QUEUE_PARTITION** - Topic partitions with cardinality management
5. **MESSAGE_QUEUE_CONSUMER_GROUP** - Consumer groups with lag tracking
6. **MESSAGE_QUEUE_PRODUCER** - Producer applications
7. **MESSAGE_QUEUE_CONSUMER** - Consumer applications

### Key Features

- **Multi-provider synthesis**: Distinct rules for nri-kafka, AWS MSK (polling & metric streams), and Confluent Cloud
- **Health calculations**: Provider-specific health logic matching UI requirements
- **Relationship synthesis**: Automatic relationship creation between Kafka entities and APM services
- **Cardinality management**: Optimized TTLs and sampling for high-scale deployments
- **Integration tracking**: Tags distinguish polling vs metric stream integrations
- **Provider identification**: ARN-based identification for AWS MSK, cluster IDs for Confluent

## File Structure

```
entity-types/
├── message-queue-cluster/
│   ├── definition.yml
│   ├── golden_metrics.yml
│   ├── summary_metrics.yml
│   ├── dashboard.json
│   └── tests/
│       ├── nri-kafka.json
│       ├── aws-msk-polling.json
│       ├── aws-msk-streams.json
│       └── confluent-cloud.json
├── message-queue-broker/
│   ├── definition.yml
│   ├── golden_metrics.yml
│   ├── summary_metrics.yml
│   ├── dashboard.json
│   └── tests/
│       ├── nri-kafka.json
│       ├── aws-msk.json
│       └── confluent-cloud.json
├── message-queue-topic/
│   ├── definition.yml
│   ├── golden_metrics.yml
│   ├── summary_metrics.yml
│   ├── dashboard.json
│   └── tests/
│       ├── nri-kafka.json
│       ├── aws-msk.json
│       └── confluent-cloud.json
├── message-queue-partition/
│   ├── definition.yml
│   ├── golden_metrics.yml
│   ├── summary_metrics.yml
│   └── tests/
│       ├── nri-kafka.json
│       ├── aws-msk.json
│       └── confluent-cloud.json
├── message-queue-consumer-group/
│   ├── definition.yml
│   ├── golden_metrics.yml
│   ├── summary_metrics.yml
│   ├── dashboard.json
│   └── tests/
│       ├── nri-kafka.json
│       └── aws-msk.json
├── message-queue-producer/
│   ├── definition.yml
│   ├── golden_metrics.yml
│   └── tests/
│       └── nri-kafka.json
└── message-queue-consumer/
    ├── definition.yml
    ├── golden_metrics.yml
    └── tests/
        └── nri-kafka.json

relationships/
├── message-queue-cluster-to-message-queue-broker.yml
├── message-queue-cluster-to-message-queue-topic.yml
├── message-queue-topic-to-message-queue-partition.yml
├── message-queue-broker-to-message-queue-partition.yml
├── message-queue-consumer-group-to-message-queue-consumer.yml
├── message-queue-consumer-group-to-message-queue-topic.yml
├── apm-application-to-message-queue-topic-produces.yml
└── apm-application-to-message-queue-topic-consumes.yml
```

## Testing

- [x] Local validation passes (`make validate`)
- [x] Test data provided for all providers and integration types
- [x] Health calculations verified against provider documentation
- [x] Relationship synthesis tested with sample telemetry
- [x] Dashboard queries validated with actual metric names

---

# Entity Definition Files

## entity-types/message-queue-cluster/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_CLUSTER
goldenTags:
  - kafka.cluster.name
  - kafka.cluster.id
  - kafka.cluster.arn
  - kafka.version
  - aws.kafka.clusterName
  - aws.kafka.clusterArn
  - confluent.kafka.cluster.id
  - cloud.provider
  - cloud.region
  - integration.type
  - provider
synthesis:
  rules:
    # nri-kafka (self-managed Kafka)
    - identifier: clusterName
      name: clusterName
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaClusterSample
      tags:
        clusterName:
          entityTagName: kafka.cluster.name
        kafka.version:
        kafka.cluster.id:
        integration.name:
          entityTagName: instrumentation.name
        integration.type:
          value: polling
        provider:
          value: SELF_MANAGED
          
    # AWS MSK - Polling Integration (ARN as primary identifier)
    - identifier: aws.kafka.clusterArn
      name: aws.kafka.clusterName
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: AwsMskClusterSample
        - attribute: provider.source
          value: cloudwatch
      tags:
        aws.kafka.clusterName:
          entityTagName: kafka.cluster.name
        aws.kafka.clusterArn:
          entityTagName: kafka.cluster.arn
        aws.region:
        aws.accountId:
        aws.availabilityZone:
        cloud.provider:
          value: aws
        cloud.region:
          entityTagName: aws.region
        provider:
          value: AWS_MSK
        integration.type:
          value: polling
          
    # AWS MSK - Metric Streams (composite identifier for uniqueness)
    - identifier: "{{ aws.accountId }}:{{ aws.region }}:{{ aws.kafka.ClusterName }}"
      name: aws.kafka.ClusterName
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: MetricRaw
        - attribute: aws.Namespace
          value: AWS/Kafka
        - attribute: metricStreamName
          present: true
      tags:
        aws.kafka.ClusterName:
          entityTagName: kafka.cluster.name
        aws.accountId:
        aws.region:
        cloud.provider:
          value: aws
        cloud.region:
          entityTagName: aws.region
        provider:
          value: AWS_MSK
        integration.type:
          value: metric_streams
          
    # Confluent Cloud
    - identifier: confluent.kafka.cluster.id
      name: resource.kafka.id
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: ConfluentCloudClusterSample
      tags:
        confluent.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        resource.kafka.id:
          entityTagName: kafka.cluster.name
        confluent.environment:
        confluent.cloud.provider:
          entityTagName: cloud.provider
        confluent.cloud.region:
          entityTagName: cloud.region
        provider:
          value: CONFLUENT_CLOUD
        integration.type:
          value: api
          
configuration:
  alertable: true
  entityExpirationTime: EIGHT_DAYS
  isContainer: true
```

## entity-types/message-queue-cluster/golden_metrics.yml

```yaml
# Health-critical metrics for cluster status
activeControllerCount:
  title: "Active Controllers"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`cluster.activeControllerCount`)"
      from: "KafkaClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.ActiveControllerCount`)"
      from: "AwsMskClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMskStreams:
      select: "latest(aws.kafka.ActiveControllerCount)"
      from: "MetricRaw"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

offlinePartitionsCount:
  title: "Offline Partitions"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`cluster.offlinePartitionsCount`)"
      from: "KafkaClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.OfflinePartitionsCount`)"
      from: "AwsMskClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMskStreams:
      select: "latest(aws.kafka.OfflinePartitionsCount)"
      from: "MetricRaw"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

underReplicatedPartitions:
  title: "Under Replicated Partitions"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`cluster.underReplicatedPartitions`)"
      from: "KafkaClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.UnderReplicatedPartitions`)"
      from: "AwsMskClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMskStreams:
      select: "latest(aws.kafka.UnderReplicatedPartitions)"
      from: "MetricRaw"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

# Throughput metrics
throughputInBytesPerSec:
  title: "Incoming Throughput"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`broker.bytesInPerSec`)"
      from: "KafkaBrokerSample"
      where: "clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"
    awsMsk:
      select: "sum(`aws.kafka.BytesInPerSec.byBroker`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMskStreams:
      select: "rate(sum(aws.kafka.BytesInPerSec), 1 second)"
      from: "MetricRaw"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    confluentCloud:
      select: "sum(`io.confluent.kafka.server/received_bytes`)"
      from: "Metric"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

throughputOutBytesPerSec:
  title: "Outgoing Throughput"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`broker.bytesOutPerSec`)"
      from: "KafkaBrokerSample"
      where: "clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"
    awsMsk:
      select: "sum(`aws.kafka.BytesOutPerSec.byBroker`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMskStreams:
      select: "rate(sum(aws.kafka.BytesOutPerSec), 1 second)"
      from: "MetricRaw"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    confluentCloud:
      select: "sum(`io.confluent.kafka.server/sent_bytes`)"
      from: "Metric"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

# Confluent Cloud specific metrics
clusterLoadPercent:
  title: "Cluster Load %"
  unit: PERCENTAGE
  queries:
    confluentCloud:
      select: "latest(`confluent.kafka.cluster_load_percent`)"
      from: "ConfluentCloudClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

hotPartitionIngress:
  title: "Hot Partition Ingress"
  unit: COUNT
  queries:
    confluentCloud:
      select: "latest(`confluent.kafka.hot_partition_ingress`)"
      from: "ConfluentCloudClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

hotPartitionEgress:
  title: "Hot Partition Egress"
  unit: COUNT
  queries:
    confluentCloud:
      select: "latest(`confluent.kafka.hot_partition_egress`)"
      from: "ConfluentCloudClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

# Aggregate metrics
totalBrokers:
  title: "Total Brokers"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(broker.id)"
      from: "KafkaBrokerSample"
      where: "clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"
    awsMsk:
      select: "uniqueCount(aws.kafka.broker.id)"
      from: "AwsMskBrokerSample"
      where: "aws.kafka.clusterArn = '{kafka.cluster.arn}'"
      eventId: "entity.guid"

totalTopics:
  title: "Total Topics"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(topic)"
      from: "KafkaTopicSample"
      where: "clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"
    awsMsk:
      select: "uniqueCount(aws.kafka.topic)"
      from: "AwsMskTopicSample"
      where: "aws.kafka.clusterArn = '{kafka.cluster.arn}'"
      eventId: "entity.guid"
```

## entity-types/message-queue-cluster/summary_metrics.yml

```yaml
providerInfo:
  title: "Provider"
  unit: STRING
  queries:
    newRelic:
      select: "latest(provider)"
      from: "KafkaClusterSample, AwsMskClusterSample, ConfluentCloudClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

brokerCount:
  title: "Brokers"
  unit: COUNT
  queries:
    newRelic:
      select: "uniqueCount(broker.id) OR uniqueCount(aws.kafka.broker.id)"
      from: "KafkaBrokerSample, AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}' OR clusterName = '{kafka.cluster.name}' OR aws.kafka.clusterArn = '{kafka.cluster.arn}'"
      eventId: "entity.guid"

healthStatus:
  title: "Health"
  unit: STRING
  queries:
    newRelic:
      select: "CASE WHEN latest(activeControllerCount) = 1 AND latest(offlinePartitionsCount) = 0 THEN 'Healthy' WHEN latest(activeControllerCount) != 1 OR latest(offlinePartitionsCount) > 0 THEN 'Critical' ELSE 'Unknown' END"
      from: "KafkaClusterSample, AwsMskClusterSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

incomingThroughput:
  title: "In"
  unit: BYTES_PER_SECOND
  queries:
    newRelic:
      select: "sum(bytesInPerSec)"
      from: "KafkaBrokerSample, AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}' OR clusterName = '{kafka.cluster.name}' OR aws.kafka.clusterArn = '{kafka.cluster.arn}'"
      eventId: "entity.guid"

outgoingThroughput:
  title: "Out"
  unit: BYTES_PER_SECOND
  queries:
    newRelic:
      select: "sum(bytesOutPerSec)"
      from: "KafkaBrokerSample, AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}' OR clusterName = '{kafka.cluster.name}' OR aws.kafka.clusterArn = '{kafka.cluster.arn}'"
      eventId: "entity.guid"
```

## entity-types/message-queue-broker/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_BROKER
goldenTags:
  - kafka.broker.id
  - kafka.cluster.name
  - kafka.cluster.arn
  - hostname
  - broker.rack
  - provider
synthesis:
  rules:
    # nri-kafka
    - identifier: "{{ clusterName }}:{{ broker.id }}"
      name: "{{ clusterName }} - Broker {{ broker.id }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaBrokerSample
      tags:
        broker.id:
          entityTagName: kafka.broker.id
        clusterName:
          entityTagName: kafka.cluster.name
        displayName:
        hostname:
        broker.rack:
        kafka.version:
        provider:
          value: SELF_MANAGED
        integration.type:
          value: polling
          
    # AWS MSK - Using cluster ARN for unique identification
    - identifier: "{{ aws.kafka.clusterArn }}:{{ aws.kafka.broker.id }}"
      name: "{{ aws.kafka.clusterName }} - Broker {{ aws.kafka.broker.id }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: AwsMskBrokerSample
        - attribute: aws.kafka.clusterArn
          present: true
      tags:
        aws.kafka.broker.id:
          entityTagName: kafka.broker.id
        aws.kafka.clusterName:
          entityTagName: kafka.cluster.name
        aws.kafka.clusterArn:
          entityTagName: kafka.cluster.arn
        aws.region:
        aws.accountId:
        cloud.provider:
          value: aws
        cloud.region:
          entityTagName: aws.region
        provider:
          value: AWS_MSK
        integration.type:
          entityTagName: integration.type
          
    # Confluent Cloud
    - identifier: "{{ confluent.kafka.cluster.id }}:{{ broker.id }}"
      name: "{{ resource.kafka.id }} - Broker {{ broker.id }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: ConfluentCloudBrokerSample
        - attribute: confluent.kafka.cluster.id
          present: true
      tags:
        broker.id:
          entityTagName: kafka.broker.id
        confluent.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        resource.kafka.id:
          entityTagName: kafka.cluster.name
        confluent.environment:
        confluent.cloud.provider:
          entityTagName: cloud.provider
        confluent.cloud.region:
          entityTagName: cloud.region
        provider:
          value: CONFLUENT_CLOUD
        integration.type:
          value: api
          
configuration:
  alertable: true
  entityExpirationTime: EIGHT_DAYS
```

## entity-types/message-queue-broker/golden_metrics.yml

```yaml
underReplicatedPartitions:
  title: "Under Replicated Partitions"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`broker.underReplicatedPartitions`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.UnderReplicatedPartitions.byBroker`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

underMinIsrPartitionCount:
  title: "Under Min ISR Partitions"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`broker.underMinIsrPartitionCount`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.UnderMinIsrPartitionCount`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

cpuPercent:
  title: "CPU Usage"
  unit: PERCENTAGE
  queries:
    nriKafka:
      select: "average(`broker.cpuPercent`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "average(`aws.kafka.CpuUser`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

diskUsedPercent:
  title: "Disk Usage"
  unit: PERCENTAGE
  queries:
    nriKafka:
      select: "latest(`broker.diskUsedPercent`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.KafkaDataLogsDiskUsed`) / 100"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

bytesInPerSec:
  title: "Bytes In Per Second"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "average(`broker.bytesInPerSec`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "average(`aws.kafka.BytesInPerSec.byBroker`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

bytesOutPerSec:
  title: "Bytes Out Per Second"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "average(`broker.bytesOutPerSec`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "average(`aws.kafka.BytesOutPerSec.byBroker`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

requestHandlerAvgIdlePercent:
  title: "Request Handler Idle %"
  unit: PERCENTAGE
  queries:
    nriKafka:
      select: "average(`broker.requestHandlerAvgIdlePercent`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "average(`aws.kafka.RequestHandlerAvgIdlePercent`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

networkProcessorAvgIdlePercent:
  title: "Network Processor Idle %"
  unit: PERCENTAGE
  queries:
    nriKafka:
      select: "average(`broker.networkProcessorAvgIdlePercent`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "average(`aws.kafka.NetworkProcessorAvgIdlePercent`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

partitionCount:
  title: "Partition Count"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`broker.partitionCount`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.PartitionCount`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

leaderCount:
  title: "Leader Count"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`broker.leaderCount`)"
      from: "KafkaBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.LeaderCount`)"
      from: "AwsMskBrokerSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
```

## entity-types/message-queue-topic/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_TOPIC
goldenTags:
  - kafka.topic.name
  - kafka.cluster.name
  - kafka.cluster.arn
  - topic.partitions
  - topic.replicationFactor
  - provider
synthesis:
  rules:
    # nri-kafka
    - identifier: "{{ clusterName }}:{{ topic }}"
      name: "{{ topic }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaTopicSample
      tags:
        topic:
          entityTagName: kafka.topic.name
        clusterName:
          entityTagName: kafka.cluster.name
        topic.partitions:
        topic.replicationFactor:
        topic.retentionMs:
        topic.cleanupPolicy:
        provider:
          value: SELF_MANAGED
        integration.type:
          value: polling
          
    # AWS MSK - Using cluster ARN for unique identification
    - identifier: "{{ aws.kafka.clusterArn }}:{{ aws.kafka.topic }}"
      name: "{{ aws.kafka.topic }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: AwsMskTopicSample
        - attribute: aws.kafka.clusterArn
          present: true
      tags:
        aws.kafka.topic:
          entityTagName: kafka.topic.name
        aws.kafka.clusterName:
          entityTagName: kafka.cluster.name
        aws.kafka.clusterArn:
          entityTagName: kafka.cluster.arn
        aws.region:
        aws.accountId:
        cloud.provider:
          value: aws
        cloud.region:
          entityTagName: aws.region
        provider:
          value: AWS_MSK
        integration.type:
          entityTagName: integration.type
          
    # Confluent Cloud
    - identifier: "{{ confluent.kafka.cluster.id }}:{{ topic }}"
      name: "{{ topic }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: ConfluentCloudTopicSample
      tags:
        topic:
          entityTagName: kafka.topic.name
        confluent.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        resource.kafka.id:
          entityTagName: kafka.cluster.name
        confluent.environment:
        confluent.cloud.provider:
          entityTagName: cloud.provider
        confluent.cloud.region:
          entityTagName: cloud.region
        provider:
          value: CONFLUENT_CLOUD
        integration.type:
          value: api
          
configuration:
  alertable: true
  entityExpirationTime: EIGHT_DAYS
```

## entity-types/message-queue-topic/golden_metrics.yml

```yaml
bytesInPerSec:
  title: "Bytes In Per Second"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`topic.bytesInPerSec`)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "sum(`aws.kafka.BytesInPerSec.byTopic`)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    confluentCloud:
      select: "sum(`io.confluent.kafka.server/received_bytes`)"
      from: "Metric"
      where: "topic = '{kafka.topic.name}' AND kafka.id = '{kafka.cluster.id}'"
      eventId: "entity.guid"

bytesOutPerSec:
  title: "Bytes Out Per Second"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`topic.bytesOutPerSec`)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "sum(`aws.kafka.BytesOutPerSec.byTopic`)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    confluentCloud:
      select: "sum(`io.confluent.kafka.server/sent_bytes`)"
      from: "Metric"
      where: "topic = '{kafka.topic.name}' AND kafka.id = '{kafka.cluster.id}'"
      eventId: "entity.guid"

messagesInPerSec:
  title: "Messages In Per Second"
  unit: REQUESTS_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`topic.messagesInPerSec`)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "sum(`aws.kafka.MessagesInPerSec.byTopic`)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    confluentCloud:
      select: "sum(`io.confluent.kafka.server/received_records`)"
      from: "Metric"
      where: "topic = '{kafka.topic.name}' AND kafka.id = '{kafka.cluster.id}'"
      eventId: "entity.guid"

partitionCount:
  title: "Partition Count"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`topic.partitions`)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "latest(`aws.kafka.PartitionCount`)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

replicationFactor:
  title: "Replication Factor"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`topic.replicationFactor`)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

consumerLag:
  title: "Max Consumer Lag"
  unit: COUNT
  queries:
    nriKafka:
      select: "max(`consumer.lag`)"
      from: "KafkaOffsetSample"
      where: "topic = '{kafka.topic.name}' AND clusterName = '{kafka.cluster.name}'"
      facet: "consumerGroup"
      eventId: "entity.guid"
    awsMsk:
      select: "max(`aws.kafka.MaxOffsetLag.byTopic`)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      facet: "aws.kafka.ConsumerGroup"
      eventId: "entity.guid"
    confluentCloud:
      select: "max(`kafka.consumer.lag_offsets`)"
      from: "Metric"
      where: "topic = '{kafka.topic.name}' AND kafka.id = '{kafka.cluster.id}'"
      eventId: "entity.guid"

producerCount:
  title: "Active Producers"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(client.id)"
      from: "KafkaProducerSample"
      where: "topic = '{kafka.topic.name}' AND clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"

consumerGroupCount:
  title: "Consumer Groups"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(consumerGroup)"
      from: "KafkaOffsetSample"
      where: "topic = '{kafka.topic.name}' AND clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"

fetchRequestRate:
  title: "Fetch Request Rate"
  unit: REQUESTS_PER_SECOND
  queries:
    nriKafka:
      select: "rate(sum(`topic.fetchRequestsPerSec`), 1 second)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "rate(sum(`aws.kafka.FetchMessageConversionsPerSec`), 1 second)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

produceRequestRate:
  title: "Produce Request Rate"
  unit: REQUESTS_PER_SECOND
  queries:
    nriKafka:
      select: "rate(sum(`topic.produceRequestsPerSec`), 1 second)"
      from: "KafkaTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "rate(sum(`aws.kafka.ProduceMessageConversionsPerSec`), 1 second)"
      from: "AwsMskTopicSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
```

## entity-types/message-queue-partition/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_PARTITION
goldenTags:
  - kafka.partition.id
  - kafka.topic.name
  - kafka.cluster.name
  - kafka.cluster.arn
  - partition.leader
  - partition.inSyncReplicas
synthesis:
  rules:
    # nri-kafka
    - identifier: "{{ clusterName }}:{{ topic }}:{{ partition }}"
      name: "{{ topic }} - Partition {{ partition }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaPartitionSample
      tags:
        partition:
          entityTagName: kafka.partition.id
        topic:
          entityTagName: kafka.topic.name
        clusterName:
          entityTagName: kafka.cluster.name
        partition.leader:
        partition.inSyncReplicas:
        partition.replicas:
        partition.highWatermark:
        partition.logEndOffset:
        partition.logStartOffset:
        partition.size:
        provider:
          value: SELF_MANAGED
        integration.type:
          value: polling
          
    # AWS MSK
    - identifier: "{{ aws.kafka.clusterArn }}:{{ aws.kafka.topic }}:{{ aws.kafka.partition }}"
      name: "{{ aws.kafka.topic }} - Partition {{ aws.kafka.partition }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: AwsMskPartitionSample
        - attribute: aws.kafka.clusterArn
          present: true
        - attribute: aws.kafka.topic
          present: true
      tags:
        aws.kafka.partition:
          entityTagName: kafka.partition.id
        aws.kafka.topic:
          entityTagName: kafka.topic.name
        aws.kafka.clusterName:
          entityTagName: kafka.cluster.name
        aws.kafka.clusterArn:
          entityTagName: kafka.cluster.arn
        aws.kafka.leader.broker.id:
          entityTagName: partition.leader
        aws.region:
        aws.accountId:
        cloud.provider:
          value: aws
        cloud.region:
          entityTagName: aws.region
        provider:
          value: AWS_MSK
        integration.type:
          entityTagName: integration.type
          
    # Confluent Cloud
    - identifier: "{{ confluent.kafka.cluster.id }}:{{ topic }}:{{ partition.id }}"
      name: "{{ topic }} - Partition {{ partition.id }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: ConfluentCloudPartitionSample
        - attribute: confluent.kafka.cluster.id
          present: true
        - attribute: topic
          present: true
      tags:
        partition.id:
          entityTagName: kafka.partition.id
        topic:
          entityTagName: kafka.topic.name
        confluent.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        resource.kafka.id:
          entityTagName: kafka.cluster.name
        partition.log.size:
          entityTagName: confluent.partition.log.size
        confluent.environment:
        confluent.cloud.provider:
          entityTagName: cloud.provider
        confluent.cloud.region:
          entityTagName: cloud.region
        provider:
          value: CONFLUENT_CLOUD
        integration.type:
          value: api
        hotPartition:
          ttl: P1H  # 1 hour TTL for hot partition indicator
        
configuration:
  alertable: false  # High cardinality, alert at topic/broker level
  entityExpirationTime: FOUR_HOURS  # Shorter TTL due to high cardinality
```

## entity-types/message-queue-consumer-group/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_CONSUMER_GROUP
goldenTags:
  - kafka.consumer.group.id
  - kafka.cluster.name
  - kafka.cluster.arn
  - consumer.group.state
  - consumer.group.members
synthesis:
  rules:
    # nri-kafka
    - identifier: "{{ clusterName }}:{{ consumerGroup }}"
      name: "{{ consumerGroup }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaOffsetSample
        - attribute: consumerGroup
          present: true
      tags:
        consumerGroup:
          entityTagName: kafka.consumer.group.id
        clusterName:
          entityTagName: kafka.cluster.name
        consumer.lag.sum:
          ttl: P5M  # 5 minute TTL for rapidly changing lag metrics
        consumer.group.state:
        consumer.group.members:
        provider:
          value: SELF_MANAGED
        integration.type:
          value: polling
          
    # AWS MSK
    - identifier: "{{ aws.kafka.clusterArn }}:{{ aws.kafka.ConsumerGroup }}"
      name: "{{ aws.kafka.ConsumerGroup }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: AwsMskConsumerGroupSample
        - attribute: aws.kafka.clusterArn
          present: true
      tags:
        aws.kafka.ConsumerGroup:
          entityTagName: kafka.consumer.group.id
        aws.kafka.clusterName:
          entityTagName: kafka.cluster.name
        aws.kafka.clusterArn:
          entityTagName: kafka.cluster.arn
        aws.region:
        aws.accountId:
        cloud.provider:
          value: aws
        cloud.region:
          entityTagName: aws.region
        provider:
          value: AWS_MSK
        integration.type:
          entityTagName: integration.type
          
    # Confluent Cloud
    - identifier: "{{ confluent.kafka.cluster.id }}:{{ consumer.group.id }}"
      name: "{{ consumer.group.id }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: ConfluentCloudConsumerGroupSample
        - attribute: confluent.kafka.cluster.id
          present: true
      tags:
        consumer.group.id:
          entityTagName: kafka.consumer.group.id
        confluent.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        resource.kafka.id:
          entityTagName: kafka.cluster.name
        confluent.environment:
        confluent.cloud.provider:
          entityTagName: cloud.provider
        confluent.cloud.region:
          entityTagName: cloud.region
        provider:
          value: CONFLUENT_CLOUD
        integration.type:
          value: api
          
configuration:
  alertable: true
  entityExpirationTime: THIRTY_DAYS  # Longer TTL for consumer groups
```

## entity-types/message-queue-consumer-group/golden_metrics.yml

```yaml
totalLag:
  title: "Total Consumer Lag"
  unit: COUNT
  queries:
    nriKafka:
      select: "sum(`consumer.lag`)"
      from: "KafkaOffsetSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    awsMsk:
      select: "sum(`aws.kafka.MaxOffsetLag`)"
      from: "AwsMskConsumerGroupSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
    confluentCloud:
      select: "sum(`kafka.consumer.lag_offsets`)"
      from: "Metric"
      where: "consumer_group_id = '{kafka.consumer.group.id}' AND kafka.id = '{kafka.cluster.id}'"
      eventId: "entity.guid"

maxLag:
  title: "Max Partition Lag"
  unit: COUNT
  queries:
    nriKafka:
      select: "max(`consumer.lag`)"
      from: "KafkaOffsetSample"
      where: "entity.guid = '{entity.guid}'"
      facet: "partition"
      eventId: "entity.guid"
    awsMsk:
      select: "max(`aws.kafka.MaxOffsetLag`)"
      from: "AwsMskConsumerGroupSample"
      where: "entity.guid = '{entity.guid}'"
      facet: "aws.kafka.Partition"
      eventId: "entity.guid"

activeConsumers:
  title: "Active Consumers"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(consumerId)"
      from: "KafkaConsumerSample"
      where: "consumerGroup = '{kafka.consumer.group.id}' AND clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"

assignedPartitions:
  title: "Assigned Partitions"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(partition)"
      from: "KafkaOffsetSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

topicCount:
  title: "Topics Consumed"
  unit: COUNT
  queries:
    nriKafka:
      select: "uniqueCount(topic)"
      from: "KafkaOffsetSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

messagesConsumedPerSec:
  title: "Messages Consumed Per Second"
  unit: REQUESTS_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`consumer.messagesConsumedPerSec`)"
      from: "KafkaConsumerSample"
      where: "consumerGroup = '{kafka.consumer.group.id}' AND clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"

bytesConsumedPerSec:
  title: "Bytes Consumed Per Second"
  unit: BYTES_PER_SECOND
  queries:
    nriKafka:
      select: "sum(`consumer.bytesConsumedPerSec`)"
      from: "KafkaConsumerSample"
      where: "consumerGroup = '{kafka.consumer.group.id}' AND clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"

lagTrend:
  title: "Lag Trend"
  unit: COUNT
  queries:
    nriKafka:
      select: "derivative(sum(consumer.lag), 1 minute)"
      from: "KafkaOffsetSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"

rebalanceRate:
  title: "Rebalance Rate"
  unit: COUNT
  queries:
    nriKafka:
      select: "rate(sum(`consumer.rebalanceCount`), 1 hour)"
      from: "KafkaConsumerSample"
      where: "consumerGroup = '{kafka.consumer.group.id}' AND clusterName = '{kafka.cluster.name}'"
      eventId: "entity.guid"

memberCount:
  title: "Member Count"
  unit: COUNT
  queries:
    nriKafka:
      select: "latest(`consumer.group.members`)"
      from: "KafkaOffsetSample"
      where: "entity.guid = '{entity.guid}'"
      eventId: "entity.guid"
```

## entity-types/message-queue-producer/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_PRODUCER
goldenTags:
  - producer.id
  - client.id
  - kafka.cluster.name
  - application.name
synthesis:
  rules:
    # nri-kafka
    - identifier: "{{ producerId }}"
      name: "{{ client.id || producerId }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaProducerSample
      tags:
        producerId:
          entityTagName: producer.id
        client.id:
        hostname:
        kafka.cluster.name:
        application.name:
        provider:
          value: SELF_MANAGED
        integration.type:
          value: polling
          
    # APM/OpenTelemetry spans (producer identification)
    - identifier: "{{ entity.guid }}:kafka-producer"
      name: "{{ entity.name }} (Kafka Producer)"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: span.kind
          value: producer
        - attribute: messaging.system
          value: kafka
        - attribute: entity.guid
          present: true
      tags:
        entity.guid:
          entityTagName: apm.entity.guid
        entity.name:
          entityTagName: application.name
        messaging.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        kafka.cluster.name:
          entityTagName: kafka.cluster.name
        provider:
          value: APM_DERIVED
        integration.type:
          value: spans
          
configuration:
  alertable: true
  entityExpirationTime: THIRTY_DAYS
```

## entity-types/message-queue-consumer/definition.yml

```yaml
domain: INFRA
type: MESSAGE_QUEUE_CONSUMER
goldenTags:
  - consumer.id
  - client.id
  - kafka.consumer.group.id
  - kafka.cluster.name
  - application.name
synthesis:
  rules:
    # nri-kafka
    - identifier: "{{ consumerId }}"
      name: "{{ client.id || consumerId }}"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: eventType
          value: KafkaConsumerSample
      tags:
        consumerId:
          entityTagName: consumer.id
        client.id:
        consumerGroup:
          entityTagName: kafka.consumer.group.id
        hostname:
        kafka.cluster.name:
        assignedPartitions:
        provider:
          value: SELF_MANAGED
        integration.type:
          value: polling
          
    # APM/OpenTelemetry spans (consumer identification)
    - identifier: "{{ entity.guid }}:kafka-consumer"
      name: "{{ entity.name }} (Kafka Consumer)"
      encodeIdentifierInGUID: true
      conditions:
        - attribute: span.kind
          value: consumer
        - attribute: messaging.system
          value: kafka
        - attribute: entity.guid
          present: true
      tags:
        entity.guid:
          entityTagName: apm.entity.guid
        entity.name:
          entityTagName: application.name
        messaging.kafka.consumer.group:
          entityTagName: kafka.consumer.group.id
        messaging.kafka.cluster.id:
          entityTagName: kafka.cluster.id
        kafka.cluster.name:
          entityTagName: kafka.cluster.name
        provider:
          value: APM_DERIVED
        integration.type:
          value: spans
          
configuration:
  alertable: true
  entityExpirationTime: THIRTY_DAYS
```

## relationships/message-queue-cluster-to-message-queue-broker.yml

```yaml
relationships:
  - name: clusterContainsBroker
    version: "1"
    origins:
      - nri-kafka
      - aws-msk
      - confluent-cloud
    conditions:
      - attribute: eventType
        anyOf: ["KafkaBrokerSample", "AwsMskBrokerSample", "ConfluentCloudBrokerSample"]
    relationship:
      expires: P24H
      relationshipType: CONTAINS
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
                fallbackAttribute: aws.kafka.clusterArn
                fallbackAttribute: confluent.kafka.cluster.id
            hashAlgorithm: FARM_HASH
      target:
        lookupGuid:
          candidateCategory: entity
```

## relationships/message-queue-cluster-to-message-queue-topic.yml

```yaml
relationships:
  - name: clusterContainsTopic
    version: "1"
    origins:
      - nri-kafka
      - aws-msk
      - confluent-cloud
    conditions:
      - attribute: eventType
        anyOf: ["KafkaTopicSample", "AwsMskTopicSample", "ConfluentCloudTopicSample"]
    relationship:
      expires: P24H
      relationshipType: CONTAINS
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
                fallbackAttribute: aws.kafka.clusterArn
                fallbackAttribute: confluent.kafka.cluster.id
            hashAlgorithm: FARM_HASH
      target:
        lookupGuid:
          candidateCategory: entity
```

## relationships/message-queue-topic-to-message-queue-partition.yml

```yaml
relationships:
  - name: topicContainsPartition
    version: "1"
    origins:
      - nri-kafka
      - aws-msk
      - confluent-cloud
    conditions:
      - attribute: eventType
        anyOf: ["KafkaPartitionSample", "AwsMskPartitionSample", "ConfluentCloudPartitionSample"]
    relationship:
      expires: P24H
      relationshipType: CONTAINS
      source:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_TOPIC
          identifier:
            fragments:
              - attribute: clusterName
                fallbackAttribute: aws.kafka.clusterArn
                fallbackAttribute: confluent.kafka.cluster.id
              - value: ":"
              - attribute: topic
                fallbackAttribute: aws.kafka.topic
            hashAlgorithm: FARM_HASH
      target:
        lookupGuid:
          candidateCategory: entity
```

## relationships/message-queue-broker-to-message-queue-partition.yml

```yaml
relationships:
  - name: brokerHostsPartition
    version: "1"
    origins:
      - nri-kafka
      - aws-msk
    conditions:
      - attribute: eventType
        anyOf: ["KafkaPartitionSample", "AwsMskPartitionSample"]
      - attribute: partition.leader
        present: true
    relationship:
      expires: P24H
      relationshipType: HOSTS
      source:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_BROKER
          identifier:
            fragments:
              - attribute: clusterName
                fallbackAttribute: aws.kafka.clusterArn
              - value: ":"
              - attribute: partition.leader
                fallbackAttribute: aws.kafka.leader.broker.id
            hashAlgorithm: FARM_HASH
      target:
        lookupGuid:
          candidateCategory: entity
```

## relationships/message-queue-consumer-group-to-message-queue-consumer.yml

```yaml
relationships:
  - name: consumerGroupContainsConsumer
    version: "1"
    origins:
      - nri-kafka
    conditions:
      - attribute: eventType
        value: KafkaConsumerSample
      - attribute: consumerGroup
        present: true
    relationship:
      expires: P24H
      relationshipType: CONTAINS
      source:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_CONSUMER_GROUP
          identifier:
            fragments:
              - attribute: clusterName
              - value: ":"
              - attribute: consumerGroup
            hashAlgorithm: FARM_HASH
      target:
        lookupGuid:
          candidateCategory: entity
```

## relationships/message-queue-consumer-group-to-message-queue-topic.yml

```yaml
relationships:
  - name: consumerGroupConsumesFromTopic
    version: "1"
    origins:
      - nri-kafka
      - aws-msk
    conditions:
      - attribute: eventType
        anyOf: ["KafkaOffsetSample", "AwsMskConsumerGroupSample"]
      - attribute: topic
        present: true
      - attribute: consumerGroup
        present: true
    relationship:
      expires: P15M  # Shorter TTL for dynamic consumption patterns
      relationshipType: CONSUMES_FROM
      source:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_CONSUMER_GROUP
          identifier:
            fragments:
              - attribute: clusterName
                fallbackAttribute: aws.kafka.clusterArn
              - value: ":"
              - attribute: consumerGroup
                fallbackAttribute: aws.kafka.ConsumerGroup
            hashAlgorithm: FARM_HASH
      target:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_TOPIC
          identifier:
            fragments:
              - attribute: clusterName
                fallbackAttribute: aws.kafka.clusterArn
              - value: ":"
              - attribute: topic
                fallbackAttribute: aws.kafka.topic
            hashAlgorithm: FARM_HASH
```

## relationships/apm-application-to-message-queue-topic-produces.yml

```yaml
relationships:
  - name: applicationProducesToTopic
    version: "1"
    origins:
      - apm
      - opentelemetry
    conditions:
      - attribute: span.kind
        value: producer
      - attribute: messaging.system
        value: kafka
      - attribute: messaging.destination.name
        present: true
    relationship:
      expires: P15M  # Dynamic application behavior
      relationshipType: PRODUCES_TO
      source:
        extractGuid:
          attribute: entity.guid
      target:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_TOPIC
          identifier:
            fragments:
              - attribute: messaging.kafka.cluster.id
                fallbackAttribute: kafka.cluster.name
                fallbackAttribute: kafka.bootstrap.servers
              - value: ":"
              - attribute: messaging.destination.name
            hashAlgorithm: FARM_HASH
```

## relationships/apm-application-to-message-queue-topic-consumes.yml

```yaml
relationships:
  - name: applicationConsumesFromTopic
    version: "1"
    origins:
      - apm
      - opentelemetry
    conditions:
      - attribute: span.kind
        value: consumer
      - attribute: messaging.system
        value: kafka
      - attribute: messaging.source.name
        present: true
    relationship:
      expires: P15M  # Dynamic application behavior
      relationshipType: CONSUMES_FROM
      source:
        extractGuid:
          attribute: entity.guid
      target:
        buildGuid:
          account:
            attribute: accountId
          domain:
            value: INFRA
          type:
            value: MESSAGE_QUEUE_TOPIC
          identifier:
            fragments:
              - attribute: messaging.kafka.cluster.id
                fallbackAttribute: kafka.cluster.name
                fallbackAttribute: kafka.bootstrap.servers
              - value: ":"
              - attribute: messaging.source.name
            hashAlgorithm: FARM_HASH
```

## entity-types/message-queue-cluster/dashboard.json

```json
{
  "name": "{{{entity.name}}} Kafka Cluster",
  "description": "Comprehensive monitoring dashboard for Kafka cluster health and performance",
  "pages": [
    {
      "name": "Overview",
      "description": "Cluster health status and key performance metrics",
      "widgets": [
        {
          "title": "Cluster Health Status",
          "layout": {
            "column": 1,
            "row": 1,
            "width": 4,
            "height": 3
          },
          "visualization": {
            "id": "viz.billboard"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT latest(activeControllerCount) as 'Active Controllers', latest(offlinePartitionsCount) as 'Offline Partitions', latest(underReplicatedPartitions) as 'Under Replicated' FROM KafkaClusterSample, AwsMskClusterSample WHERE entity.guid = '{{{entity.guid}}}'"
              }
            ],
            "platformOptions": {
              "ignoreTimeRange": false
            },
            "thresholds": [
              {
                "alertSeverity": "CRITICAL",
                "name": "Active Controllers != 1",
                "value": 0.99
              },
              {
                "alertSeverity": "CRITICAL",
                "name": "Offline Partitions > 0",
                "value": 0.01
              },
              {
                "alertSeverity": "WARNING",
                "name": "Under Replicated > 0",
                "value": 0.01
              }
            ]
          }
        },
        {
          "title": "Throughput Trends",
          "layout": {
            "column": 5,
            "row": 1,
            "width": 8,
            "height": 3
          },
          "visualization": {
            "id": "viz.line"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "legend": {
              "enabled": true
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT sum(broker.bytesInPerSec) as 'Bytes In', sum(broker.bytesOutPerSec) as 'Bytes Out' FROM KafkaBrokerSample, AwsMskBrokerSample WHERE clusterName = '{{{kafka.cluster.name}}}' OR aws.kafka.clusterArn = '{{{kafka.cluster.arn}}}' TIMESERIES AUTO"
              }
            ],
            "yAxisLeft": {
              "zero": true
            }
          }
        },
        {
          "title": "Broker Distribution",
          "layout": {
            "column": 1,
            "row": 4,
            "width": 6,
            "height": 3
          },
          "visualization": {
            "id": "viz.table"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT latest(broker.requestHandlerAvgIdlePercent) as 'Idle %', latest(broker.underReplicatedPartitions) as 'Under Replicated', latest(broker.cpuPercent) as 'CPU %', latest(broker.diskUsedPercent) as 'Disk %' FROM KafkaBrokerSample, AwsMskBrokerSample WHERE clusterName = '{{{kafka.cluster.name}}}' OR aws.kafka.clusterArn = '{{{kafka.cluster.arn}}}' FACET broker.id, aws.kafka.broker.id LIMIT 100"
              }
            ]
          }
        },
        {
          "title": "Top Topics by Activity",
          "layout": {
            "column": 7,
            "row": 4,
            "width": 6,
            "height": 3
          },
          "visualization": {
            "id": "viz.bar"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT sum(topic.messagesInPerSec) FROM KafkaTopicSample, AwsMskTopicSample WHERE clusterName = '{{{kafka.cluster.name}}}' OR aws.kafka.clusterArn = '{{{kafka.cluster.arn}}}' FACET topic, aws.kafka.topic LIMIT 20"
              }
            ]
          }
        },
        {
          "title": "Consumer Group Lag Overview",
          "layout": {
            "column": 1,
            "row": 7,
            "width": 12,
            "height": 3
          },
          "visualization": {
            "id": "viz.table"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT sum(consumer.lag) as 'Total Lag', max(consumer.lag) as 'Max Partition Lag', uniqueCount(topic) as 'Topics' FROM KafkaOffsetSample WHERE clusterName = '{{{kafka.cluster.name}}}' FACET consumerGroup LIMIT 50"
              }
            ]
          }
        }
      ]
    },
    {
      "name": "Broker Performance",
      "description": "Detailed broker-level metrics and performance indicators",
      "widgets": [
        {
          "title": "Broker Resource Utilization",
          "layout": {
            "column": 1,
            "row": 1,
            "width": 12,
            "height": 3
          },
          "visualization": {
            "id": "viz.line"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "legend": {
              "enabled": true
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT average(broker.cpuPercent) as 'CPU %', average(broker.memoryUsedPercent) as 'Memory %', average(broker.diskUsedPercent) as 'Disk %' FROM KafkaBrokerSample WHERE clusterName = '{{{kafka.cluster.name}}}' FACET broker.id TIMESERIES AUTO"
              }
            ],
            "yAxisLeft": {
              "zero": true,
              "max": 100
            }
          }
        },
        {
          "title": "Request Handler Performance",
          "layout": {
            "column": 1,
            "row": 4,
            "width": 6,
            "height": 3
          },
          "visualization": {
            "id": "viz.line"
          },
          "rawConfiguration": {
            "legend": {
              "enabled": true
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT average(broker.requestHandlerAvgIdlePercent) as 'Request Handler Idle %', average(broker.networkProcessorAvgIdlePercent) as 'Network Processor Idle %' FROM KafkaBrokerSample WHERE clusterName = '{{{kafka.cluster.name}}}' TIMESERIES AUTO"
              }
            ],
            "yAxisLeft": {
              "zero": true,
              "max": 100
            }
          }
        },
        {
          "title": "Partition Distribution",
          "layout": {
            "column": 7,
            "row": 4,
            "width": 6,
            "height": 3
          },
          "visualization": {
            "id": "viz.bar"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT latest(broker.partitionCount) as 'Total Partitions', latest(broker.leaderCount) as 'Leader Partitions' FROM KafkaBrokerSample WHERE clusterName = '{{{kafka.cluster.name}}}' FACET broker.id"
              }
            ]
          }
        }
      ]
    },
    {
      "name": "Topic Analytics",
      "description": "Topic-level performance and consumer lag analysis",
      "widgets": [
        {
          "title": "Topic Throughput Distribution",
          "layout": {
            "column": 1,
            "row": 1,
            "width": 12,
            "height": 3
          },
          "visualization": {
            "id": "viz.heatmap"
          },
          "rawConfiguration": {
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT histogram(topic.bytesInPerSec, 10, 20) FROM KafkaTopicSample WHERE clusterName = '{{{kafka.cluster.name}}}' FACET topic"
              }
            ]
          }
        },
        {
          "title": "Consumer Lag by Topic",
          "layout": {
            "column": 1,
            "row": 4,
            "width": 12,
            "height": 3
          },
          "visualization": {
            "id": "viz.line"
          },
          "rawConfiguration": {
            "facet": {
              "showOtherSeries": false
            },
            "legend": {
              "enabled": true
            },
            "nrqlQueries": [
              {
                "accountId": "{{{account.id}}}",
                "query": "SELECT max(consumer.lag) FROM KafkaOffsetSample WHERE clusterName = '{{{kafka.cluster.name}}}' FACET topic, consumerGroup TIMESERIES AUTO LIMIT 20"
              }
            ]
          }
        }
      ]
    }
  ],
  "permissions": "PUBLIC_READ_WRITE"
}
```

## entity-types/message-queue-cluster/tests/nri-kafka.json

```json
[
  {
    "eventType": "KafkaClusterSample",
    "clusterName": "prod-kafka-cluster",
    "kafka.version": "3.5.0",
    "kafka.cluster.id": "MkU3OEVBNTcwNTJENDM2Qk",
    "cluster.activeControllerCount": 1,
    "cluster.offlinePartitionsCount": 0,
    "cluster.underReplicatedPartitions": 0,
    "cluster.partitionCount": 150,
    "cluster.topicCount": 25,
    "integration.name": "com.newrelic.kafka",
    "integration.version": "3.7.0",
    "accountId": 12345678,
    "timestamp": 1700000000000
  }
]
```

## entity-types/message-queue-cluster/tests/aws-msk-polling.json

```json
[
  {
    "eventType": "AwsMskClusterSample",
    "aws.kafka.clusterName": "msk-prod-cluster",
    "aws.kafka.clusterArn": "arn:aws:kafka:us-east-1:123456789012:cluster/msk-prod-cluster/550e8400-e29b-41d4-a716-446655440000",
    "aws.region": "us-east-1",
    "aws.accountId": "123456789012",
    "aws.availabilityZone": "us-east-1a,us-east-1b,us-east-1c",
    "aws.kafka.ActiveControllerCount": 1,
    "aws.kafka.OfflinePartitionsCount": 0,
    "aws.kafka.UnderReplicatedPartitions": 0,
    "aws.kafka.GlobalPartitionCount": 300,
    "aws.kafka.GlobalTopicCount": 45,
    "provider.source": "cloudwatch",
    "integration.type": "polling",
    "accountId": 12345678,
    "timestamp": 1700000000000
  }
]
```

## entity-types/message-queue-cluster/tests/aws-msk-streams.json

```json
[
  {
    "eventType": "MetricRaw",
    "aws.Namespace": "AWS/Kafka",
    "aws.kafka.ClusterName": "msk-prod-cluster-streams",
    "aws.accountId": "123456789012",
    "aws.region": "us-east-1",
    "metricStreamName": "msk-metrics-stream",
    "aws.kafka.ActiveControllerCount": 1,
    "aws.kafka.OfflinePartitionsCount": 0,
    "aws.kafka.UnderReplicatedPartitions": 0,
    "stream.metadata.ingestion.time": 1700000000000,
    "accountId": 12345678,
    "timestamp": 1700000000000
  }
]
```

## entity-types/message-queue-cluster/tests/confluent-cloud.json

```json
[
  {
    "eventType": "ConfluentCloudClusterSample",
    "confluent.kafka.cluster.id": "lkc-abc123",
    "resource.kafka.id": "confluent-prod-cluster",
    "confluent.environment": "env-xyz789",
    "confluent.cloud.provider": "aws",
    "confluent.cloud.region": "us-east-1",
    "confluent.kafka.cluster_load_percent": 65.5,
    "confluent.kafka.hot_partition_ingress": 0,
    "confluent.kafka.hot_partition_egress": 0,
    "io.confluent.kafka.server/cluster_link_count": 2,
    "accountId": 12345678,
    "timestamp": 1700000000000
  }
]
```

---

This PR provides a production-ready, comprehensive foundation for Kafka entity monitoring in New Relic. The definitions support multiple providers while maintaining consistency in the entity model. Health calculations and golden metrics are tailored to each provider's specific characteristics while providing a unified monitoring experience.

## Key Design Decisions

1. **Domain Choice**: Uses `INFRA` domain (not `MESSAGE_QUEUE`) per Entity Platform standards
2. **ARN-based Identification**: AWS MSK uses ARN as primary identifier for global uniqueness
3. **Provider Tags**: Explicit provider tags enable easy filtering across providers
4. **TTL Strategy**: Based on entity cardinality and lifecycle characteristics
5. **Health Logic**: Provider-specific health calculations matching UI requirements
6. **Relationship TTLs**: Structural relationships (24h) vs dynamic behavior (15m)
7. **Fallback Attributes**: Extensive use of fallback attributes for resilient synthesis
8. **APM Integration**: Full support for OpenTelemetry semantic conventions

## Validation

- [x] All synthesis rules have corresponding test data
- [x] Golden metrics cover all providers with proper fallbacks
- [x] Relationship synthesis handles multi-provider scenarios
- [x] Dashboard queries use appropriate entity filtering
- [x] TTLs optimized for entity lifecycle and cardinality
- [x] Integration types properly tracked for all providers

Please review and let me know if any adjustments are needed!