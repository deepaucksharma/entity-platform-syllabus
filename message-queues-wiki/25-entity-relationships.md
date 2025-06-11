# Entity Relationships

## Overview

This document provides a comprehensive guide to understanding and working with entity relationships in the Message Queues monitoring system. It covers the entity model, relationship types, synthesis rules, and how entities connect across the Kafka ecosystem.

## Entity Model Overview

### Entity Hierarchy

```
Kafka Entity Hierarchy
├── Account Level
│   └── AWS Account / Confluent Organization
├── Provider Level
│   └── AWS MSK / Confluent Cloud
├── Cluster Level
│   └── Kafka Cluster
├── Broker Level
│   └── Kafka Broker / Node
├── Topic Level
│   └── Kafka Topic
├── Partition Level
│   └── Topic Partition
├── Consumer Level
│   └── Consumer Group
└── Application Level
    └── Producer / Consumer Application
```

## Entity Types

### Kafka Cluster Entities

```typescript
interface KafkaClusterEntity {
  // Core attributes
  guid: string;
  name: string;
  type: 'AWSMSKCLUSTER' | 'CONFLUENTCLOUDCLUSTER';
  domain: 'INFRA';
  entityType: 'KAFKA_CLUSTER';
  
  // Provider-specific attributes
  provider: {
    type: 'AWS_MSK' | 'CONFLUENT_CLOUD';
    region?: string;
    accountId: string;
    clusterId: string;
    arn?: string; // AWS specific
    environmentId?: string; // Confluent specific
  };
  
  // Cluster configuration
  configuration: {
    version: string;
    brokerCount: number;
    storageMode: 'TIERED' | 'LOCAL';
    instanceType?: string;
    encryptionInTransit: boolean;
    encryptionAtRest: boolean;
  };
  
  // Relationships
  relationships: {
    contains: string[]; // Broker GUIDs
    monitors: string[]; // Topic GUIDs
    ownedBy: string; // Account GUID
  };
  
  // Metrics summary
  goldenMetrics: {
    throughput: GoldenMetric;
    availability: GoldenMetric;
    latency: GoldenMetric;
    errors: GoldenMetric;
  };
}
```

### Kafka Broker Entities

```typescript
interface KafkaBrokerEntity {
  guid: string;
  name: string;
  type: 'KAFKA_BROKER';
  domain: 'INFRA';
  
  // Broker identification
  broker: {
    id: number;
    host: string;
    port: number;
    rack?: string;
  };
  
  // Cluster relationship
  cluster: {
    guid: string;
    name: string;
    provider: string;
  };
  
  // Broker role and state
  state: {
    status: 'ONLINE' | 'OFFLINE' | 'DEGRADED';
    isController: boolean;
    isLeader: boolean;
    startTime: number;
  };
  
  // Resource metrics
  resources: {
    cpu: ResourceMetric;
    memory: ResourceMetric;
    disk: ResourceMetric;
    network: ResourceMetric;
  };
  
  // Relationships
  relationships: {
    partOf: string; // Cluster GUID
    hosts: string[]; // Partition GUIDs
    replicates: string[]; // Partition GUIDs
  };
}
```

### Kafka Topic Entities

```typescript
interface KafkaTopicEntity {
  guid: string;
  name: string;
  type: 'KAFKA_TOPIC';
  domain: 'INFRA';
  
  // Topic identification
  topic: {
    name: string;
    partitionCount: number;
    replicationFactor: number;
    minInSyncReplicas: number;
  };
  
  // Configuration
  configuration: {
    retentionMs: number;
    retentionBytes: number;
    segmentMs: number;
    compressionType: 'none' | 'gzip' | 'snappy' | 'lz4' | 'zstd';
    cleanupPolicy: 'delete' | 'compact' | 'delete,compact';
  };
  
  // Usage patterns
  usage: {
    producerCount: number;
    consumerGroupCount: number;
    messageRate: number;
    byteRate: number;
  };
  
  // Relationships
  relationships: {
    belongsTo: string; // Cluster GUID
    partitions: string[]; // Partition GUIDs
    producedBy: string[]; // Application GUIDs
    consumedBy: string[]; // Consumer Group GUIDs
  };
}
```

### Consumer Group Entities

```typescript
interface ConsumerGroupEntity {
  guid: string;
  name: string;
  type: 'KAFKA_CONSUMER_GROUP';
  domain: 'INFRA';
  
  // Group identification
  group: {
    id: string;
    state: 'STABLE' | 'PREPARING_REBALANCE' | 'COMPLETING_REBALANCE' | 'EMPTY' | 'DEAD';
    protocol: string;
    protocolType: string;
  };
  
  // Membership
  members: {
    count: number;
    activeCount: number;
    members: ConsumerMember[];
  };
  
  // Consumption patterns
  consumption: {
    topics: string[];
    partitionAssignments: PartitionAssignment[];
    totalLag: number;
    consumptionRate: number;
  };
  
  // Relationships
  relationships: {
    consumes: string[]; // Topic GUIDs
    runsOn: string; // Cluster GUID
    associatedWith: string[]; // Application GUIDs
  };
}
```

## Relationship Types

### Hierarchical Relationships

```typescript
enum HierarchicalRelationshipType {
  CONTAINS = 'CONTAINS',         // Parent contains child
  PART_OF = 'PART_OF',          // Child is part of parent
  BELONGS_TO = 'BELONGS_TO',    // Ownership relationship
  HOSTED_BY = 'HOSTED_BY',      // Infrastructure hosting
  MANAGES = 'MANAGES'           // Management relationship
}

interface HierarchicalRelationship {
  type: HierarchicalRelationshipType;
  source: string; // Parent entity GUID
  target: string; // Child entity GUID
  metadata?: {
    created: number;
    weight?: number;
    primary?: boolean;
  };
}
```

### Operational Relationships

```typescript
enum OperationalRelationshipType {
  PRODUCES_TO = 'PRODUCES_TO',      // Producer to topic
  CONSUMES_FROM = 'CONSUMES_FROM',  // Consumer from topic
  REPLICATES = 'REPLICATES',        // Broker replicates partition
  LEADS = 'LEADS',                  // Broker leads partition
  MONITORS = 'MONITORS',            // Monitoring relationship
  DEPENDS_ON = 'DEPENDS_ON'         // Dependency relationship
}

interface OperationalRelationship {
  type: OperationalRelationshipType;
  source: string;
  target: string;
  metadata?: {
    throughput?: number;
    latency?: number;
    errorRate?: number;
    sla?: string;
  };
}
```

### Data Flow Relationships

```typescript
interface DataFlowRelationship {
  type: 'DATA_FLOW';
  source: string; // Producer entity
  target: string; // Consumer entity
  via: string[];  // Intermediate entities (topics, brokers)
  
  flow: {
    volumePerSecond: number;
    messageRate: number;
    averageSize: number;
    peakThroughput: number;
  };
  
  quality: {
    successRate: number;
    errorRate: number;
    retryRate: number;
    averageLatency: number;
  };
}
```

## Entity Synthesis

### Synthesis Rules

```typescript
// Entity synthesis configuration
const entitySynthesisRules: SynthesisRule[] = [
  {
    // AWS MSK Cluster synthesis
    entityType: 'AWSMSKCLUSTER',
    sources: ['KafkaClusterSample'],
    identifier: (data) => `${data.provider.accountId}:${data.provider.region}:${data.provider.clusterName}`,
    synthesis: {
      guid: (data) => generateGuid('AWSMSKCLUSTER', data),
      name: (data) => data.provider.clusterName,
      tags: {
        'account.id': (data) => data.provider.accountId,
        'aws.region': (data) => data.provider.region,
        'aws.arn': (data) => data.provider.clusterArn,
        'kafka.version': (data) => data.provider.kafkaVersion,
        environment: (data) => data.tags?.Environment || 'production'
      },
      relationships: [
        {
          type: 'BELONGS_TO',
          target: (data) => `AWS_ACCOUNT:${data.provider.accountId}`
        },
        {
          type: 'CONTAINS',
          targets: (data) => data.brokers.map(b => `KAFKA_BROKER:${b.id}`)
        }
      ]
    }
  },
  
  {
    // Confluent Cloud Cluster synthesis
    entityType: 'CONFLUENTCLOUDCLUSTER',
    sources: ['ConfluentCloudClusterSample'],
    identifier: (data) => `${data.provider.organizationId}:${data.provider.environmentId}:${data.provider.clusterId}`,
    synthesis: {
      guid: (data) => generateGuid('CONFLUENTCLOUDCLUSTER', data),
      name: (data) => data.provider.clusterName,
      tags: {
        'confluent.org': (data) => data.provider.organizationId,
        'confluent.env': (data) => data.provider.environmentId,
        'confluent.cluster': (data) => data.provider.clusterId,
        'confluent.cloud': (data) => data.provider.cloud,
        'confluent.region': (data) => data.provider.region
      }
    }
  },
  
  {
    // Topic synthesis with relationships
    entityType: 'KAFKA_TOPIC',
    sources: ['KafkaTopicSample'],
    identifier: (data) => `${data.clusterGuid}:${data.topicName}`,
    synthesis: {
      relationships: [
        {
          type: 'BELONGS_TO',
          target: (data) => data.clusterGuid
        },
        {
          type: 'PRODUCED_BY',
          targets: (data) => data.producers || [],
          bidirectional: true
        },
        {
          type: 'CONSUMED_BY',
          targets: (data) => data.consumerGroups || [],
          bidirectional: true
        }
      ]
    }
  }
];
```

### Relationship Discovery

```typescript
class RelationshipDiscovery {
  async discoverRelationships(entities: Entity[]): Promise<Relationship[]> {
    const relationships: Relationship[] = [];
    
    // Discover hierarchical relationships
    relationships.push(...this.discoverHierarchical(entities));
    
    // Discover operational relationships
    relationships.push(...await this.discoverOperational(entities));
    
    // Discover data flow relationships
    relationships.push(...await this.discoverDataFlow(entities));
    
    // Discover inferred relationships
    relationships.push(...this.inferRelationships(entities));
    
    return this.deduplicateRelationships(relationships);
  }
  
  private discoverHierarchical(entities: Entity[]): Relationship[] {
    const relationships: Relationship[] = [];
    
    // Cluster -> Broker relationships
    const clusters = entities.filter(e => e.type.includes('CLUSTER'));
    const brokers = entities.filter(e => e.type === 'KAFKA_BROKER');
    
    clusters.forEach(cluster => {
      const clusterBrokers = brokers.filter(b => 
        b.tags?.clusterName === cluster.name ||
        b.tags?.clusterId === cluster.tags?.clusterId
      );
      
      clusterBrokers.forEach(broker => {
        relationships.push({
          type: 'CONTAINS',
          source: cluster.guid,
          target: broker.guid,
          metadata: { discovered: true }
        });
      });
    });
    
    // Topic -> Partition relationships
    const topics = entities.filter(e => e.type === 'KAFKA_TOPIC');
    const partitions = entities.filter(e => e.type === 'KAFKA_PARTITION');
    
    topics.forEach(topic => {
      const topicPartitions = partitions.filter(p => 
        p.tags?.topicName === topic.name
      );
      
      topicPartitions.forEach(partition => {
        relationships.push({
          type: 'CONTAINS',
          source: topic.guid,
          target: partition.guid
        });
      });
    });
    
    return relationships;
  }
  
  private async discoverOperational(entities: Entity[]): Promise<Relationship[]> {
    const relationships: Relationship[] = [];
    
    // Query for producer/consumer relationships
    const query = `
      FROM Transaction, KafkaProducerSample, KafkaConsumerSample
      SELECT 
        appId,
        entityGuid,
        kafkaCluster,
        kafkaTopic,
        operation
      WHERE operation IN ('produce', 'consume')
      SINCE 1 hour ago
    `;
    
    const results = await this.nrqlQuery(query);
    
    results.forEach(result => {
      if (result.operation === 'produce') {
        relationships.push({
          type: 'PRODUCES_TO',
          source: result.appId,
          target: result.kafkaTopic,
          metadata: {
            cluster: result.kafkaCluster
          }
        });
      } else if (result.operation === 'consume') {
        relationships.push({
          type: 'CONSUMES_FROM',
          source: result.appId,
          target: result.kafkaTopic,
          metadata: {
            cluster: result.kafkaCluster
          }
        });
      }
    });
    
    return relationships;
  }
}
```

## Relationship Queries

### GraphQL Relationship Queries

```graphql
# Query entity relationships
query GetEntityRelationships($guid: EntityGuid!) {
  actor {
    entity(guid: $guid) {
      guid
      name
      type
      
      # Direct relationships
      relationships {
        source {
          entity {
            guid
            name
            type
          }
        }
        target {
          entity {
            guid
            name
            type
          }
        }
        type
      }
      
      # Related entities with specific types
      relatedEntities(filter: {
        relationshipTypes: ["CONTAINS", "PRODUCES_TO", "CONSUMES_FROM"]
        entityTypes: ["KAFKA_TOPIC", "KAFKA_BROKER", "APM_APPLICATION"]
      }) {
        results {
          source {
            entity {
              guid
              name
              type
            }
          }
          target {
            entity {
              guid
              name
              type
              ... on ApmApplicationEntity {
                language
                settings {
                  appName
                }
              }
            }
          }
          type
        }
      }
    }
  }
}
```

### NRQL Relationship Analysis

```sql
-- Analyze producer to topic relationships
FROM Transaction
SELECT 
  count(*) as messageCount,
  average(duration) as avgLatency,
  uniqueCount(appId) as producerCount
WHERE operation = 'kafka.produce'
FACET kafka.topic, kafka.cluster
SINCE 1 hour ago

-- Consumer group lag by relationship
FROM KafkaConsumerSample
SELECT 
  max(consumerLag) as maxLag,
  average(consumerLag) as avgLag,
  latest(messageRate) as consumptionRate
FACET consumerGroup, topic, cluster
WHERE consumerLag > 0
SINCE 30 minutes ago

-- Broker to partition relationships
FROM KafkaPartitionSample
SELECT 
  uniqueCount(partition) as partitionCount,
  filter(uniqueCount(partition), WHERE isLeader = true) as leaderCount,
  filter(uniqueCount(partition), WHERE inSyncReplicas < replicationFactor) as underReplicatedCount
FACET broker, cluster
SINCE 1 hour ago
```

## Relationship Visualization

### Topology Mapping

```typescript
interface TopologyMap {
  nodes: TopologyNode[];
  edges: TopologyEdge[];
  layout: 'hierarchical' | 'force' | 'circular';
}

class TopologyBuilder {
  buildTopology(entities: Entity[], relationships: Relationship[]): TopologyMap {
    // Create nodes from entities
    const nodes = entities.map(entity => ({
      id: entity.guid,
      label: entity.name,
      type: entity.type,
      level: this.getHierarchyLevel(entity.type),
      metrics: {
        health: entity.alertSeverity === 'NOT_ALERTING' ? 100 : 50,
        throughput: this.getEntityThroughput(entity)
      },
      style: this.getNodeStyle(entity)
    }));
    
    // Create edges from relationships
    const edges = relationships.map(rel => ({
      id: `${rel.source}-${rel.target}`,
      source: rel.source,
      target: rel.target,
      type: rel.type,
      weight: this.calculateEdgeWeight(rel),
      style: this.getEdgeStyle(rel)
    }));
    
    return {
      nodes,
      edges,
      layout: this.determineOptimalLayout(nodes, edges)
    };
  }
  
  private getHierarchyLevel(entityType: string): number {
    const hierarchy = {
      'AWS_ACCOUNT': 0,
      'CONFLUENT_ORGANIZATION': 0,
      'AWSMSKCLUSTER': 1,
      'CONFLUENTCLOUDCLUSTER': 1,
      'KAFKA_BROKER': 2,
      'KAFKA_TOPIC': 3,
      'KAFKA_PARTITION': 4,
      'KAFKA_CONSUMER_GROUP': 3,
      'APM_APPLICATION': 2
    };
    
    return hierarchy[entityType] || 5;
  }
}
```

### Dependency Graph

```typescript
interface DependencyGraph {
  services: ServiceNode[];
  dependencies: Dependency[];
  criticalPaths: CriticalPath[];
}

class DependencyAnalyzer {
  analyzeDependencies(
    entities: Entity[],
    relationships: Relationship[]
  ): DependencyGraph {
    // Build service map
    const services = this.identifyServices(entities, relationships);
    
    // Calculate dependencies
    const dependencies = this.calculateDependencies(services, relationships);
    
    // Find critical paths
    const criticalPaths = this.findCriticalPaths(services, dependencies);
    
    return {
      services,
      dependencies,
      criticalPaths
    };
  }
  
  private calculateDependencies(
    services: ServiceNode[],
    relationships: Relationship[]
  ): Dependency[] {
    const dependencies: Dependency[] = [];
    
    services.forEach(service => {
      // Find all topics this service produces to
      const producedTopics = relationships
        .filter(r => r.type === 'PRODUCES_TO' && r.source === service.guid)
        .map(r => r.target);
      
      // Find all consumers of these topics
      producedTopics.forEach(topic => {
        const consumers = relationships
          .filter(r => r.type === 'CONSUMES_FROM' && r.target === topic)
          .map(r => r.source);
        
        consumers.forEach(consumer => {
          dependencies.push({
            source: service.guid,
            target: consumer,
            via: topic,
            strength: this.calculateDependencyStrength(service, consumer, topic),
            type: 'DATA_DEPENDENCY'
          });
        });
      });
    });
    
    return dependencies;
  }
}
```

## Relationship Management

### Relationship Lifecycle

```typescript
class RelationshipManager {
  async createRelationship(relationship: RelationshipCreate): Promise<Relationship> {
    // Validate entities exist
    await this.validateEntities(relationship.source, relationship.target);
    
    // Check for existing relationship
    const existing = await this.findRelationship(
      relationship.source,
      relationship.target,
      relationship.type
    );
    
    if (existing) {
      return this.updateRelationship(existing.id, relationship);
    }
    
    // Create new relationship
    const created = await this.persistRelationship(relationship);
    
    // Update entity caches
    await this.updateEntityCaches(created);
    
    // Trigger relationship events
    await this.emitRelationshipEvent('created', created);
    
    return created;
  }
  
  async deleteRelationship(relationshipId: string): Promise<void> {
    const relationship = await this.getRelationship(relationshipId);
    
    // Check if relationship can be deleted
    if (!this.canDelete(relationship)) {
      throw new Error('Cannot delete system-managed relationship');
    }
    
    // Delete relationship
    await this.removeRelationship(relationshipId);
    
    // Update caches
    await this.updateEntityCaches(relationship, 'delete');
    
    // Trigger events
    await this.emitRelationshipEvent('deleted', relationship);
  }
  
  async refreshRelationships(entityGuid: string): Promise<void> {
    // Get entity
    const entity = await this.getEntity(entityGuid);
    
    // Discover current relationships
    const discovered = await this.relationshipDiscovery.discover(entity);
    
    // Get stored relationships
    const stored = await this.getStoredRelationships(entityGuid);
    
    // Reconcile differences
    const toCreate = discovered.filter(d => 
      !stored.find(s => this.isSameRelationship(d, s))
    );
    
    const toDelete = stored.filter(s => 
      !discovered.find(d => this.isSameRelationship(d, s))
    );
    
    // Apply changes
    await Promise.all([
      ...toCreate.map(r => this.createRelationship(r)),
      ...toDelete.map(r => this.deleteRelationship(r.id))
    ]);
  }
}
```

## Best Practices

### 1. Entity Modeling
- Use consistent naming conventions
- Include all relevant metadata in tags
- Define clear entity boundaries
- Maintain entity lifecycle

### 2. Relationship Design
- Use appropriate relationship types
- Avoid circular dependencies
- Include relationship metadata
- Consider bidirectional relationships

### 3. Performance
- Cache relationship queries
- Use batch operations for updates
- Implement pagination for large sets
- Optimize relationship traversal

### 4. Maintenance
- Regular relationship validation
- Cleanup orphaned relationships
- Monitor relationship health
- Document relationship patterns

## Conclusion

Understanding and properly managing entity relationships is crucial for effective Kafka monitoring. The relationship model provides a powerful way to navigate and analyze the complex interactions within Kafka infrastructure, enabling better insights and more effective troubleshooting.