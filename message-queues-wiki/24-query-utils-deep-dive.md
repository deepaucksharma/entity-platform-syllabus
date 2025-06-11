# Query Utils Deep Dive

## Overview

This document provides an in-depth analysis of the query-utils module, which is the core query building and data processing engine for the Message Queues monitoring system. It covers the architecture, implementation details, optimization strategies, and usage patterns.

## Query Utils Implementation (Actual)

### Core Imports and Dependencies

```typescript
// From query-utils.ts
import NRQLModel from '@datanerd/nrql-model';
import { ngql } from 'nr1';

import {
  MSK_PROVIDER_POLLING,
  CONFLUENT_CLOUD_PROVIDER,
  confluentCloudTopicWhereCond,
  DEFAULT_METRIC_VALUE,
  getAwsStreamWhere,
  getConditionMapping,
  keyMapping,
  LIMIT_20,
  MAX_LIMIT,
  METRIC_IDS,
  MSK_PROVIDER,
  PROVIDERS_ID_MAP,
  METRIC_TAG_ATTRIBUTES,
} from '../config/constants';
import { QueryModel, QueryOptions, StaticInfo } from '../types/types';
```

### Entity Query Filters (Actual Implementation)

```typescript
// AWS Query Filters
export const AWS_CLUSTER_QUERY_FILTER = 
  "domain IN ('INFRA') AND type='AWSMSKCLUSTER'";

export const AWS_TOPIC_QUERY_FILTER = 
  "domain IN ('INFRA') AND type='AWSMSKTOPIC'";

export const AWS_BROKER_QUERY_FILTER = 
  "domain IN ('INFRA') AND type='AWSMSKBROKER'";

// Confluent Cloud Query Filters  
export const CONFLUENT_CLOUD_QUERY_FILTER_CLUSTER =
  "domain IN ('INFRA') AND type='CONFLUENTCLOUDCLUSTER'";

export const CONFLUENT_CLOUD_QUERY_FILTER_TOPIC =
  "domain IN ('INFRA') AND type='CONFLUENTCLOUDKAFKATOPIC'";

// Combined filters
export const COUNT_TOPIC_QUERY_FILTER = 
  `domain IN ('INFRA') AND type IN ('AWSMSKTOPIC', 'CONFLUENTCLOUDKAFKATOPIC')`;

// Dynamic filter functions
export const AWS_CLUSTER_QUERY_FILTER_FUNC = (searchName: string) => {
  return `${AWS_CLUSTER_QUERY_FILTER} ${searchName ? `AND name IN (${searchName})` : ''}`;
};

export const CONFLUENT_CLOUD_QUERY_FILTER_CLUSTER_FUNC = (searchName: string) => {
  return `${CONFLUENT_CLOUD_QUERY_FILTER_CLUSTER} ${searchName ? `AND name IN (${searchName})` : ''}`;
};
```

## GraphQL Query Templates (Actual Implementation)

### ALL_KAFKA_TABLE_QUERY

```typescript
export const ALL_KAFKA_TABLE_QUERY = ngql`
  query ALL_KAFKA_TABLE_QUERY(
    $awsQuery: String!, 
    $confluentCloudQuery: String!, 
    $facet: EntitySearchCountsFacet!,
    $orderBy: EntitySearchOrderBy!
  ) {
    actor {
      awsEntitySearch: entitySearch(query: $awsQuery) {
        count
        facetedCounts(facets: {facetCriterion: {facet: $facet}, orderBy: $orderBy}) {
          counts {
            count
            facet
          }
        }
        results {
          accounts {
            id
            name
            reportingEventTypes(filter: "AwsMskBrokerSample")
          }
        }
      }
      confluentCloudEntitySearch: entitySearch(query: $confluentCloudQuery) {
        count
        facetedCounts(facets: {facetCriterion: {facet: $facet}, orderBy: $orderBy}) {
          counts {
            count
            facet
          }
        }
        results {
          accounts {
            id
            name
          }
        }
      }
    }
  }
`;
```

## NRQL Query Patterns (Actual Implementation)

### AWS MSK Queries

```typescript
// Polling-based queries
const AWS_POLLING_QUERIES = {
  CLUSTER_HEALTH_QUERY: {
    select: `round(average(provider.globalPartitionCount.Average)) as 'Global partition count', average(provider.activeControllerCount.Sum) as 'Active Controller Count', average(provider.offlinePartitionsCount.Sum) as 'Offline Partitions', average(provider.globalTopicCount.Average) as 'Global Topics Count'`,
    from: 'AwsMskClusterSample',
    facet: 'provider.clusterName',
    limit: MAX_LIMIT,
    metricType: 'Cluster',
  },
  
  TOPIC_HEALTH_QUERY: {
    select: `sum(BytesInPerSec) as BytesInPerSec, sum(BytesOutPerSec) as BytesOutPerSec, sum(MessagesInPerSec) as MessagesInPerSec`,
    from: {
      select: [
        "latest(provider.bytesInPerSec.Average) as 'BytesInPerSec'",
        "latest(provider.bytesOutPerSec.Average) as 'BytesOutPerSec'",
        "latest(provider.messagesInPerSec.Average) as 'MessagesInPerSec'",
      ],
      from: 'AwsMskTopicSample',
      facet: ['displayName', 'provider.brokerId'],
      limit: MAX_LIMIT,
    },
    facet: 'displayName',
    limit: MAX_LIMIT,
    isNested: true,
  },
  
  BROKER_HEALTH_QUERY: {
    select: `round(max(provider.cpuUser.Average)+max(provider.cpuSystem.Average)) as 'CPU %', round(average(provider.memoryUsed.Average)) as 'Memory Used in %', round(average(provider.underReplicatedPartitions.Sum)) as 'Under replicated partitions'`,
    from: 'AwsMskBrokerSample',
    facet: 'provider.brokerId',
    limit: MAX_LIMIT,
    metricType: 'Broker',
  }
};

// Metric Stream queries
const AWS_METRIC_STREAM_QUERIES = {
  CLUSTER_HEALTH_QUERY: {
    select: `round(average(aws.kafka.GlobalPartitionCount)) as 'Global partition count', average(aws.kafka.ActiveControllerCount) as 'Active Controller Count', average(aws.kafka.OfflinePartitionsCount) as 'Offline Partitions', average(aws.kafka.GlobalTopicCount) as 'Global Topics Count'`,
    from: 'Metric',
    where: ["metricName like 'aws.kafka.%' AND aws.kafka.byTopic IS NULL"],
    facet: 'aws.kafka.ClusterName OR aws.msk.clusterName',
    limit: MAX_LIMIT,
    metricType: 'Cluster',
  }
};
```

### Confluent Cloud Queries

```typescript
const CONFLUENT_QUERIES = {
  CLUSTER_HEALTH_QUERY: {
    select: `(average(confluent_kafka_server_cluster_load_percent or confluent.kafka.server.cluster_load_percent)*100 or ${DEFAULT_METRIC_VALUE}) as 'Cluster load percent', filter(average(confluent_kafka_server_hot_partition_ingress or confluent.kafka.server.hot_partition_ingress), WHERE (confluent_kafka_server_hot_partition_ingress or confluent.kafka.server.hot_partition_ingress)=1 AND metricName=('confluent_kafka_server_hot_partition_ingress' or 'confluent.kafka.server.hot_partition_ingress')) as 'Hot partition Ingress', filter(average(confluent_kafka_server_hot_partition_egress or confluent.kafka.server.hot_partition_egress), WHERE (confluent_kafka_server_hot_partition_egress or confluent.kafka.server.hot_partition_egress)=1 AND metricName=('confluent_kafka_server_hot_partition_egress' or 'confluent.kafka.server.hot_partition_egress')) as 'Hot partition Egress'`,
    from: 'Metric',
    where: ["metricName like '%confluent%'"],
    facet: 'kafka.cluster_name or kafka.clusterName or confluent.clusterName',
    limit: MAX_LIMIT,
    metricType: 'Cluster',
  },
  
  TOPIC_HEALTH_QUERY: {
    select: `latest(\`Bytes In\`) as 'Bytes In', latest(\`Bytes Out\`) as 'Bytes Out'`,
    from: {
      select: `filter(uniqueCount(topic OR confluent.kafka.server.metric.topic), where metricName like '%confluent%'), filter(sum(\`confluent.kafka.server.received_bytes\`)/datapointCount(), where metricName='confluent.kafka.server.received_bytes') as 'Bytes In', filter(sum(\`aws.kafka.confluent.kafka.server.sent_bytes\`)/datapointCount(), where metricName='confluent.kafka.server.sent_bytes') as 'Bytes Out'`,
      from: 'Metric',
      where: ["metricName like '%confluent%'"],
      facet: ["\`displayName\` OR \`entity.name\` as 'topic'"],
      limit: MAX_LIMIT,
    },
    facet: 'topic',
    limit: MAX_LIMIT,
    isNested: true,
  }
};
```

### GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY

```typescript
export const GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY = ngql`
  query GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY(
    $awsTopicQuery: String!, 
    $confluentTopicQuery: String!
  ) {
    actor {
      awsTopicEntitySearch: entitySearch(query: $awsTopicQuery) {
        __typename
        polling: groupedResults(by: {tag: "aws.clusterName"}) {
          group
        }
        metrics: groupedResults(by: {tag: "aws.kafka.ClusterName"}) {
          group
        }
      }
      confluentTopicEntitySearch: entitySearch(query: $confluentTopicQuery) {
        groupedResults(by: {tag: "confluent.kafka.id"}) {
          group
        }
      }
    }
  }
  
  private buildWhereClause(condition: WhereCondition): string {
    if ('and' in condition) {
      return `(${condition.and.map(c => this.buildWhereClause(c)).join(' AND ')})`;
    }
    if ('or' in condition) {
      return `(${condition.or.map(c => this.buildWhereClause(c)).join(' OR ')})`;
    }
    if ('not' in condition) {
      return `NOT (${this.buildWhereClause(condition.not)})`;
    }
    
    // Simple condition
    const { field, operator, value } = condition as SimpleCondition;
    return `${field} ${operator} ${this.formatValue(value)}`;
  }
  
  // Dynamic time range handling
  since(timeRange: TimeRange | string): this {
    if (typeof timeRange === 'string') {
      this.query.since = timeRange;
    } else {
      this.query.since = this.formatTimeRange(timeRange);
    }
    return this;
  }
  
  // Subquery support
  subquery(builder: (qb: NRQLQueryBuilder) => NRQLQueryBuilder): this {
    const subqueryBuilder = new NRQLQueryBuilder();
    const subquery = builder(subqueryBuilder).build();
    
    this.query.select.push(`(${subquery}) as subqueryResult`);
    return this;
  }
  
  // Query composition
  compose(...queries: NRQLQueryBuilder[]): this {
    const composer = new QueryComposer();
    const composed = composer.compose(this, ...queries);
    this.query = composed.query;
    return this;
  }
  
  // Build final query with optimizations
  build(options: BuildOptions = {}): string {
    if (options.optimize) {
      this.optimize();
    }
    
    const parts: string[] = [];
    
    // SELECT clause
    if (this.query.select.length > 0) {
      parts.push(`SELECT ${this.query.select.join(', ')}`);
    }
    
    // FROM clause
    if (this.query.from.length > 0) {
      parts.push(`FROM ${this.query.from.join(', ')}`);
    }
    
    // WHERE clause
    if (this.query.where.length > 0) {
      parts.push(`WHERE ${this.query.where.join(' AND ')}`);
    }
    
    // FACET clause
    if (this.query.facet.length > 0) {
      parts.push(`FACET ${this.query.facet.join(', ')}`);
    }
    
    // Time range
    if (this.query.since) {
      parts.push(`SINCE ${this.query.since}`);
    }
    if (this.query.until) {
      parts.push(`UNTIL ${this.query.until}`);
    }
    
    // LIMIT
    if (this.query.limit) {
      parts.push(`LIMIT ${this.query.limit}`);
    }
    
    // TIMESERIES
    if (this.query.timeseries) {
      parts.push(`TIMESERIES ${this.query.timeseries}`);
    }
    
    // COMPARE WITH
    if (this.query.compare) {
      parts.push(`COMPARE WITH ${this.query.compare}`);
    }
    
    return parts.join(' ');
  }
  
  private optimize(): void {
    // Remove duplicate selections
    this.query.select = [...new Set(this.query.select)];
    
    // Optimize WHERE conditions
    this.query.where = this.optimizeWhereConditions(this.query.where);
    
    // Add sampling for large time ranges
    if (this.shouldAddSampling()) {
      this.query.limit = this.query.limit || 1000;
    }
  }
}
```

### Query Templates

```typescript
// query-utils/query-templates.ts
export class QueryTemplates {
  static readonly templates = {
    clusterHealth: {
      name: 'Cluster Health Query',
      description: 'Comprehensive cluster health metrics',
      builder: (params: ClusterHealthParams) => new NRQLQueryBuilder()
        .select(
          { expression: 'provider.activeControllers', aggregation: 'latest', alias: 'controllers' },
          { expression: 'provider.offlinePartitions', aggregation: 'latest', alias: 'offline' },
          { expression: 'provider.underReplicatedPartitions', aggregation: 'latest', alias: 'underReplicated' },
          { expression: 'provider.brokerCount', aggregation: 'latest', alias: 'brokers' }
        )
        .from('KafkaClusterSample')
        .where({ field: 'clusterName', operator: '=', value: params.clusterName })
        .since(params.timeRange || '1 hour ago')
    },
    
    topicThroughput: {
      name: 'Topic Throughput Analysis',
      description: 'Analyze topic throughput patterns',
      builder: (params: TopicThroughputParams) => new NRQLQueryBuilder()
        .select(
          { expression: 'bytesInPerSec', aggregation: 'rate', alias: 'inRate' },
          { expression: 'bytesOutPerSec', aggregation: 'rate', alias: 'outRate' },
          { expression: 'messagesInPerSec', aggregation: 'average', alias: 'msgRate' }
        )
        .from('KafkaTopicSample')
        .where({
          and: [
            { field: 'topicName', operator: '=', value: params.topicName },
            { field: 'clusterName', operator: '=', value: params.clusterName }
          ]
        })
        .facet('topicName')
        .since(params.timeRange || '1 hour ago')
        .timeseries(params.interval || '5 minutes')
    },
    
    consumerLagAnalysis: {
      name: 'Consumer Lag Analysis',
      description: 'Analyze consumer group lag patterns',
      builder: (params: ConsumerLagParams) => new NRQLQueryBuilder()
        .select(
          { expression: 'consumerLag', aggregation: 'max', alias: 'maxLag' },
          { expression: 'consumerLag', aggregation: 'average', alias: 'avgLag' },
          { expression: 'consumerLag', aggregation: 'percentile', args: [95], alias: 'p95Lag' }
        )
        .from('KafkaConsumerSample')
        .where({ field: 'consumerGroup', operator: '=', value: params.consumerGroup })
        .facet('topic', 'partition')
        .since(params.timeRange || '6 hours ago')
        .limit(100),
    
    brokerPerformance: {
      name: 'Broker Performance Metrics',
      description: 'Detailed broker performance analysis',
      builder: (params: BrokerPerformanceParams) => new NRQLQueryBuilder()
        .select(
          { expression: 'cpuUser + cpuSystem', alias: 'cpuTotal' },
          { expression: 'memoryUsedBytes / memoryTotalBytes * 100', alias: 'memoryPercent' },
          { expression: 'networkRxBytes', aggregation: 'rate', alias: 'networkIn' },
          { expression: 'networkTxBytes', aggregation: 'rate', alias: 'networkOut' }
        )
        .from('KafkaBrokerSample')
        .where({
          and: [
            { field: 'brokerHost', operator: '=', value: params.brokerHost },
            { field: 'clusterName', operator: '=', value: params.clusterName }
          ]
        })
        .since(params.timeRange || '1 hour ago')
        .timeseries('1 minute')
        .compare('1 day ago')
    }
  };
  
  static getTemplate(name: string): QueryTemplate {
    return this.templates[name];
  }
  
  static buildFromTemplate(name: string, params: any): string {
    const template = this.getTemplate(name);
    if (!template) {
      throw new Error(`Unknown query template: ${name}`);
    }
    
    return template.builder(params).build({ optimize: true });
  }
}
```

## GraphQL Query Builder

### NerdGraph Query Construction

```typescript
// query-utils/graphql-query-builder.ts
export class GraphQLQueryBuilder {
  private operation: GraphQLOperation = {
    type: 'query',
    name: null,
    variables: {},
    selections: []
  };
  
  query(name?: string): this {
    this.operation.type = 'query';
    this.operation.name = name;
    return this;
  }
  
  mutation(name?: string): this {
    this.operation.type = 'mutation';
    this.operation.name = name;
    return this;
  }
  
  variable(name: string, type: string, defaultValue?: any): this {
    this.operation.variables[name] = { type, defaultValue };
    return this;
  }
  
  select(field: string | FieldSelection): this {
    if (typeof field === 'string') {
      this.operation.selections.push({ field });
    } else {
      this.operation.selections.push(field);
    }
    return this;
  }
  
  // Nested selections
  nested(field: string, builder: (qb: GraphQLQueryBuilder) => void): this {
    const nestedBuilder = new GraphQLQueryBuilder();
    builder(nestedBuilder);
    
    this.operation.selections.push({
      field,
      selections: nestedBuilder.operation.selections,
      arguments: nestedBuilder.operation.variables
    });
    
    return this;
  }
  
  // Entity search helper
  entitySearch(params: EntitySearchParams): this {
    return this.nested('actor', qb => {
      qb.nested('entitySearch', search => {
        search.select('count');
        search.select('query');
        search.nested('results', results => {
          results.nested('entities', entities => {
            entities.select('guid');
            entities.select('name');
            entities.select('type');
            entities.select('domain');
            entities.select('entityType');
            entities.select('reporting');
            entities.select('alertSeverity');
            entities.nested('tags', tags => {
              tags.select('key');
              tags.select('values');
            });
            if (params.includeRelationships) {
              entities.nested('relationships', rel => {
                rel.select('source');
                rel.select('target');
                rel.select('type');
              });
            }
            if (params.includeGoldenMetrics) {
              entities.nested('goldenMetrics', gm => {
                gm.select('title');
                gm.select('query');
                gm.select('unit');
              });
            }
          });
          results.select('nextCursor');
        });
      }).argument('query', params.query)
        .argument('limit', params.limit || 200);
    }).argument('id', params.accountId);
    
    return this;
  }
  
  // Build the final query
  build(): string {
    const { type, name, variables, selections } = this.operation;
    
    let query = type;
    
    // Add operation name and variables
    if (name || Object.keys(variables).length > 0) {
      query += ' ';
      if (name) query += name;
      if (Object.keys(variables).length > 0) {
        query += '(' + this.buildVariables(variables) + ')';
      }
    }
    
    // Add selections
    query += ' {\n' + this.buildSelections(selections, 1) + '\n}';
    
    return query;
  }
  
  private buildSelections(selections: FieldSelection[], indent: number): string {
    return selections.map(sel => {
      const spaces = '  '.repeat(indent);
      let field = spaces + sel.field;
      
      if (sel.arguments) {
        field += '(' + this.buildArguments(sel.arguments) + ')';
      }
      
      if (sel.selections) {
        field += ' {\n' + this.buildSelections(sel.selections, indent + 1) + '\n' + spaces + '}';
      }
      
      return field;
    }).join('\n');
  }
}
```

## Data Transformation

### Metric Aggregation

```typescript
// query-utils/metric-aggregator.ts
export class MetricAggregator {
  aggregate(
    data: MetricData[],
    aggregation: AggregationType,
    options: AggregationOptions = {}
  ): AggregatedMetric {
    switch (aggregation) {
      case 'average':
        return this.calculateAverage(data, options);
      
      case 'sum':
        return this.calculateSum(data, options);
      
      case 'percentile':
        return this.calculatePercentile(data, options.percentile || 95);
      
      case 'rate':
        return this.calculateRate(data, options);
      
      case 'histogram':
        return this.calculateHistogram(data, options);
      
      case 'custom':
        return options.customAggregator!(data);
      
      default:
        throw new Error(`Unknown aggregation type: ${aggregation}`);
    }
  }
  
  private calculateRate(data: MetricData[], options: AggregationOptions): AggregatedMetric {
    if (data.length < 2) {
      return { value: 0, unit: 'per second' };
    }
    
    // Sort by timestamp
    const sorted = [...data].sort((a, b) => a.timestamp - b.timestamp);
    
    // Calculate rates between consecutive points
    const rates: number[] = [];
    for (let i = 1; i < sorted.length; i++) {
      const timeDiff = (sorted[i].timestamp - sorted[i-1].timestamp) / 1000; // seconds
      const valueDiff = sorted[i].value - sorted[i-1].value;
      rates.push(valueDiff / timeDiff);
    }
    
    // Apply smoothing if requested
    if (options.smoothing) {
      return {
        value: this.applySmoothing(rates, options.smoothing),
        unit: 'per second',
        metadata: { smoothing: options.smoothing }
      };
    }
    
    return {
      value: rates.reduce((a, b) => a + b, 0) / rates.length,
      unit: 'per second'
    };
  }
  
  private calculateHistogram(data: MetricData[], options: AggregationOptions): AggregatedMetric {
    const values = data.map(d => d.value);
    const min = Math.min(...values);
    const max = Math.max(...values);
    const buckets = options.buckets || 10;
    const bucketSize = (max - min) / buckets;
    
    const histogram = new Array(buckets).fill(0);
    
    values.forEach(value => {
      const bucketIndex = Math.min(
        Math.floor((value - min) / bucketSize),
        buckets - 1
      );
      histogram[bucketIndex]++;
    });
    
    return {
      value: histogram,
      unit: 'distribution',
      metadata: {
        min,
        max,
        bucketSize,
        buckets: histogram.map((count, i) => ({
          range: [min + i * bucketSize, min + (i + 1) * bucketSize],
          count
        }))
      }
    };
  }
}
```

### Entity Transformation

```typescript
// query-utils/entity-transformer.ts
export class EntityTransformer {
  transformToHierarchy(entities: Entity[]): EntityHierarchy {
    const hierarchy: EntityHierarchy = {
      accounts: new Map(),
      relationships: new Map()
    };
    
    // Group entities by type
    const grouped = this.groupByType(entities);
    
    // Build hierarchy
    grouped.clusters.forEach(cluster => {
      const accountId = cluster.tags.accountId;
      if (!hierarchy.accounts.has(accountId)) {
        hierarchy.accounts.set(accountId, {
          id: accountId,
          name: cluster.tags.accountName || accountId,
          clusters: new Map()
        });
      }
      
      const account = hierarchy.accounts.get(accountId)!;
      account.clusters.set(cluster.guid, {
        ...cluster,
        brokers: new Map(),
        topics: new Map()
      });
    });
    
    // Add brokers to clusters
    grouped.brokers.forEach(broker => {
      const clusterId = this.findParentCluster(broker, hierarchy);
      if (clusterId) {
        const cluster = this.getClusterFromHierarchy(clusterId, hierarchy);
        cluster?.brokers.set(broker.guid, broker);
      }
    });
    
    // Add topics to clusters
    grouped.topics.forEach(topic => {
      const clusterId = this.findParentCluster(topic, hierarchy);
      if (clusterId) {
        const cluster = this.getClusterFromHierarchy(clusterId, hierarchy);
        cluster?.topics.set(topic.guid, {
          ...topic,
          partitions: new Map(),
          consumers: new Map()
        });
      }
    });
    
    // Build relationship map
    entities.forEach(entity => {
      if (entity.relationships) {
        entity.relationships.forEach(rel => {
          if (!hierarchy.relationships.has(rel.type)) {
            hierarchy.relationships.set(rel.type, []);
          }
          hierarchy.relationships.get(rel.type)!.push({
            source: entity.guid,
            target: rel.targetGuid,
            metadata: rel.metadata
          });
        });
      }
    });
    
    return hierarchy;
  }
  
  transformForVisualization(
    entities: Entity[],
    visualizationType: VisualizationType
  ): VisualizationData {
    switch (visualizationType) {
      case 'honeycomb':
        return this.transformToHoneycomb(entities);
      
      case 'topology':
        return this.transformToTopology(entities);
      
      case 'table':
        return this.transformToTable(entities);
      
      case 'timeseries':
        return this.transformToTimeseries(entities);
      
      default:
        throw new Error(`Unknown visualization type: ${visualizationType}`);
    }
  }
  
  private transformToHoneycomb(entities: Entity[]): HoneycombData {
    const hexagons = entities.map((entity, index) => {
      const position = this.calculateHexPosition(index, entities.length);
      
      return {
        id: entity.guid,
        x: position.x,
        y: position.y,
        data: {
          name: entity.name,
          type: entity.type,
          health: entity.metrics?.healthScore || 100,
          metrics: this.summarizeMetrics(entity.metrics)
        },
        color: this.getHealthColor(entity.metrics?.healthScore),
        size: this.calculateHexSize(entity)
      };
    });
    
    return {
      hexagons,
      layout: 'packed',
      dimensions: this.calculateDimensions(hexagons)
    };
  }
}
```

## Query Optimization

### Query Cache Implementation

```typescript
// query-utils/query-cache.ts
export class QueryCache {
  private cache: Map<string, CacheEntry> = new Map();
  private lru: string[] = [];
  private maxSize: number = 1000;
  private defaultTTL: number = 300000; // 5 minutes
  
  async get<T>(
    key: string,
    fetcher: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const cached = this.cache.get(key);
    
    if (cached && !this.isExpired(cached)) {
      this.updateLRU(key);
      return cached.data as T;
    }
    
    // Fetch new data
    const data = await fetcher();
    
    // Store in cache
    this.set(key, data, options.ttl || this.defaultTTL);
    
    return data;
  }
  
  private set(key: string, data: any, ttl: number): void {
    // Remove oldest entry if cache is full
    if (this.cache.size >= this.maxSize && !this.cache.has(key)) {
      const oldest = this.lru.shift();
      if (oldest) {
        this.cache.delete(oldest);
      }
    }
    
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl,
      hits: 0
    });
    
    this.updateLRU(key);
  }
  
  private updateLRU(key: string): void {
    const index = this.lru.indexOf(key);
    if (index > -1) {
      this.lru.splice(index, 1);
    }
    this.lru.push(key);
  }
  
  // Invalidation strategies
  invalidate(pattern?: string | RegExp): void {
    if (!pattern) {
      this.cache.clear();
      this.lru = [];
      return;
    }
    
    const regex = typeof pattern === 'string' 
      ? new RegExp(pattern) 
      : pattern;
    
    const keysToDelete = Array.from(this.cache.keys())
      .filter(key => regex.test(key));
    
    keysToDelete.forEach(key => {
      this.cache.delete(key);
      const index = this.lru.indexOf(key);
      if (index > -1) {
        this.lru.splice(index, 1);
      }
    });
  }
  
  // Cache warming
  async warm(queries: WarmQuery[]): Promise<void> {
    const warmPromises = queries.map(async (query) => {
      const key = this.generateKey(query.query, query.variables);
      
      try {
        const data = await query.fetcher();
        this.set(key, data, query.ttl || this.defaultTTL);
      } catch (error) {
        console.error(`Failed to warm cache for ${key}:`, error);
      }
    });
    
    await Promise.all(warmPromises);
  }
}
```

### Query Batching

```typescript
// query-utils/query-batcher.ts
export class QueryBatcher {
  private pendingQueries: Map<string, PendingQuery[]> = new Map();
  private batchTimeout: number = 50; // ms
  private maxBatchSize: number = 20;
  
  async execute<T>(
    query: string,
    variables: any = {},
    options: BatchOptions = {}
  ): Promise<T> {
    const batchKey = options.batchKey || this.generateBatchKey(query);
    
    return new Promise((resolve, reject) => {
      // Add to pending queries
      if (!this.pendingQueries.has(batchKey)) {
        this.pendingQueries.set(batchKey, []);
        
        // Schedule batch execution
        setTimeout(() => {
          this.executeBatch(batchKey);
        }, options.batchTimeout || this.batchTimeout);
      }
      
      const pending = this.pendingQueries.get(batchKey)!;
      pending.push({ query, variables, resolve, reject });
      
      // Execute immediately if batch is full
      if (pending.length >= this.maxBatchSize) {
        this.executeBatch(batchKey);
      }
    });
  }
  
  private async executeBatch(batchKey: string): Promise<void> {
    const queries = this.pendingQueries.get(batchKey);
    if (!queries || queries.length === 0) return;
    
    this.pendingQueries.delete(batchKey);
    
    try {
      // Combine queries into a single request
      const batchedQuery = this.combinQueries(queries);
      const results = await this.executeQuery(batchedQuery);
      
      // Distribute results to individual promises
      queries.forEach((query, index) => {
        query.resolve(results[index]);
      });
    } catch (error) {
      // Reject all promises in the batch
      queries.forEach(query => {
        query.reject(error);
      });
    }
  }
  
  private combinQueries(queries: PendingQuery[]): BatchedQuery {
    if (queries.length === 1) {
      return queries[0];
    }
    
    // For NRQL queries, combine with UNION
    if (this.isNRQL(queries[0].query)) {
      return {
        query: queries.map((q, i) => `(${q.query}) as q${i}`).join(' UNION '),
        variables: {}
      };
    }
    
    // For GraphQL, create aliased queries
    if (this.isGraphQL(queries[0].query)) {
      return {
        query: `{
          ${queries.map((q, i) => `q${i}: ${q.query}`).join('\n')}
        }`,
        variables: Object.assign({}, ...queries.map(q => q.variables))
      };
    }
    
    throw new Error('Unknown query type for batching');
  }
}
```

## Query Utilities

### Query Validation

```typescript
// query-utils/query-validation.ts
export class QueryValidator {
  private static readonly NRQL_KEYWORDS = [
    'SELECT', 'FROM', 'WHERE', 'FACET', 'SINCE', 'UNTIL', 
    'LIMIT', 'TIMESERIES', 'COMPARE', 'WITH'
  ];
  
  private static readonly DANGEROUS_PATTERNS = [
    /DROP\s+TABLE/i,
    /DELETE\s+FROM/i,
    /UPDATE\s+SET/i,
    /INSERT\s+INTO/i
  ];
  
  static validateNRQL(query: string): ValidationResult {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    // Check for dangerous patterns
    for (const pattern of this.DANGEROUS_PATTERNS) {
      if (pattern.test(query)) {
        errors.push({
          type: 'security',
          message: 'Query contains potentially dangerous operations',
          position: query.search(pattern)
        });
      }
    }
    
    // Validate syntax
    const syntaxErrors = this.validateNRQLSyntax(query);
    errors.push(...syntaxErrors);
    
    // Check for performance issues
    const perfWarnings = this.checkPerformanceIssues(query);
    warnings.push(...perfWarnings);
    
    // Validate time range
    const timeRangeErrors = this.validateTimeRange(query);
    errors.push(...timeRangeErrors);
    
    return {
      valid: errors.length === 0,
      errors,
      warnings
    };
  }
  
  private static validateNRQLSyntax(query: string): ValidationError[] {
    const errors: ValidationError[] = [];
    
    // Must have SELECT and FROM
    if (!query.includes('SELECT')) {
      errors.push({
        type: 'syntax',
        message: 'Query must contain SELECT clause'
      });
    }
    
    if (!query.includes('FROM')) {
      errors.push({
        type: 'syntax',
        message: 'Query must contain FROM clause'
      });
    }
    
    // Check for balanced parentheses
    const openParens = (query.match(/\(/g) || []).length;
    const closeParens = (query.match(/\)/g) || []).length;
    if (openParens !== closeParens) {
      errors.push({
        type: 'syntax',
        message: 'Unbalanced parentheses in query'
      });
    }
    
    // Validate FACET clause
    if (query.includes('FACET')) {
      const facetMatch = query.match(/FACET\s+(.+?)(?:\s+LIMIT|\s+SINCE|\s+UNTIL|$)/i);
      if (facetMatch) {
        const facetFields = facetMatch[1].split(',').map(f => f.trim());
        if (facetFields.some(f => f.length === 0)) {
          errors.push({
            type: 'syntax',
            message: 'Empty field in FACET clause'
          });
        }
      }
    }
    
    return errors;
  }
  
  private static checkPerformanceIssues(query: string): ValidationWarning[] {
    const warnings: ValidationWarning[] = [];
    
    // Check for missing LIMIT
    if (!query.includes('LIMIT') && query.includes('FACET')) {
      warnings.push({
        type: 'performance',
        message: 'Consider adding LIMIT clause to FACET query',
        suggestion: 'LIMIT 100'
      });
    }
    
    // Check for SELECT *
    if (query.match(/SELECT\s+\*/)) {
      warnings.push({
        type: 'performance',
        message: 'Avoid using SELECT *, specify needed fields',
        suggestion: 'SELECT specific fields'
      });
    }
    
    // Check for large time ranges without sampling
    const timeRangeMatch = query.match(/SINCE\s+(\d+)\s+(hour|day|week|month)s?\s+ago/i);
    if (timeRangeMatch) {
      const amount = parseInt(timeRangeMatch[1]);
      const unit = timeRangeMatch[2].toLowerCase();
      const hours = this.convertToHours(amount, unit);
      
      if (hours > 24 && !query.includes('TIMESERIES')) {
        warnings.push({
          type: 'performance',
          message: `Large time range (${hours} hours) without TIMESERIES`,
          suggestion: 'Add TIMESERIES for better performance'
        });
      }
    }
    
    return warnings;
  }
}
```

### Query Helpers

```typescript
// query-utils/query-helpers.ts
export class QueryHelpers {
  // Time range helpers
  static getRelativeTimeRange(amount: number, unit: TimeUnit): string {
    return `${amount} ${unit}${amount > 1 ? 's' : ''} ago`;
  }
  
  static getAbsoluteTimeRange(start: Date, end: Date): string {
    return `SINCE ${start.getTime()} UNTIL ${end.getTime()}`;
  }
  
  // Field helpers
  static formatFieldList(fields: string[]): string {
    return fields.map(f => this.escapeField(f)).join(', ');
  }
  
  static escapeField(field: string): string {
    if (field.includes(' ') || field.includes('.')) {
      return `\`${field}\``;
    }
    return field;
  }
  
  // Value helpers
  static formatValue(value: any): string {
    if (value === null || value === undefined) {
      return 'NULL';
    }
    
    if (typeof value === 'string') {
      return `'${value.replace(/'/g, "\\'")}'`;
    }
    
    if (Array.isArray(value)) {
      return `(${value.map(v => this.formatValue(v)).join(', ')})`;
    }
    
    if (value instanceof Date) {
      return value.getTime().toString();
    }
    
    return value.toString();
  }
  
  // Query building helpers
  static buildWhereClause(conditions: Record<string, any>): string {
    const clauses = Object.entries(conditions)
      .filter(([_, value]) => value !== undefined)
      .map(([field, value]) => {
        if (Array.isArray(value)) {
          return `${this.escapeField(field)} IN ${this.formatValue(value)}`;
        }
        return `${this.escapeField(field)} = ${this.formatValue(value)}`;
      });
    
    return clauses.join(' AND ');
  }
  
  // Aggregation helpers
  static buildAggregation(
    field: string,
    aggregation: AggregationType,
    options?: AggregationOptions
  ): string {
    switch (aggregation) {
      case 'count':
        return `count(${field})`;
      
      case 'uniqueCount':
        return `uniqueCount(${field})`;
      
      case 'sum':
        return `sum(${field})`;
      
      case 'average':
        return `average(${field})`;
      
      case 'min':
        return `min(${field})`;
      
      case 'max':
        return `max(${field})`;
      
      case 'percentile':
        return `percentile(${field}, ${options?.percentile || 95})`;
      
      case 'rate':
        return `rate(sum(${field}), ${options?.interval || '1 minute'})`;
      
      case 'latest':
        return `latest(${field})`;
      
      case 'stddev':
        return `stddev(${field})`;
      
      default:
        throw new Error(`Unknown aggregation type: ${aggregation}`);
    }
  }
}
```

## Best Practices

### 1. Query Building
- Use the builder pattern for complex queries
- Leverage query templates for common patterns
- Always validate queries before execution
- Use parameterized queries to prevent injection

### 2. Performance
- Implement query caching with appropriate TTLs
- Use query batching for multiple similar queries
- Add sampling for large time ranges
- Optimize WHERE clauses for index usage

### 3. Error Handling
- Validate queries before execution
- Implement retry logic with backoff
- Provide meaningful error messages
- Log query performance metrics

### 4. Maintenance
- Keep query templates up to date
- Monitor query performance
- Regular cache invalidation
- Document complex query patterns

## Conclusion

The query-utils module provides a robust foundation for building, optimizing, and executing queries in the Message Queues monitoring system. By leveraging the builder pattern, caching, batching, and validation, it ensures efficient and reliable data retrieval while maintaining flexibility for complex query requirements.