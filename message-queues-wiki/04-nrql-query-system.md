# NRQL Query System

## Overview

The New Relic Query Language (NRQL) forms the backbone of data retrieval in the Message Queues monitoring system. This document provides a comprehensive guide to the query patterns, optimization strategies, and provider-specific implementations.

## NRQL Fundamentals for Kafka Monitoring

### Basic Query Structure

```sql
SELECT function(attribute) 
FROM EventType 
WHERE conditions 
FACET dimension 
SINCE timeRange 
LIMIT count
```

### Event Types and Data Sources

#### AWS MSK Event Types
```typescript
const AWS_MSK_EVENT_TYPES = {
  CLUSTER: 'AwsMskClusterSample',
  BROKER: 'AwsMskBrokerSample', 
  TOPIC: 'AwsMskTopicSample'
};

// Metric Streams event type
const METRIC_STREAM_EVENT = 'Metric';
```

#### Confluent Cloud Event Types
```typescript
const CONFLUENT_EVENT_TYPES = {
  ALL_METRICS: 'Metric' // All Confluent metrics use Metric event type
};
```

## Query Building System

### Central Query Builder Architecture

```typescript
// query-utils.ts core query builder
export function getQueryByProviderAndPreference(
  isPreferMetrics: boolean,
  provider: string,
  metricId: string, 
  filterSet: string,
  facet: string,
  timeRange: TimeRanges,
  staticInfo?: StaticInfo
): QueryOptions {
  // Provider-specific query selection
  const queries = getQueriesByProvider(provider, isPreferMetrics);
  const queryDefinition = queries[metricId];
  
  if (!queryDefinition) {
    throw new Error(`Query not found for ${provider}.${metricId}`);
  }
  
  // Build final query with filters
  return getQueryString(
    queryDefinition,
    provider,
    isPreferMetrics,
    filterSet,
    facet,
    timeRange,
    staticInfo
  );
}
```

### Query Definition Structure

```typescript
interface QueryModel {
  from: string | QueryModel;  // Event type or nested query
  select: string[];           // Metric selections
  facet?: string[];          // Grouping dimensions
  where?: string[];          // Filter conditions
  timeRange?: TimeRanges;    // Time window
  limit?: number | 'MAX';    // Result limit
  isNested?: boolean;        // Nested query flag
  isTimeseries?: boolean;    // Time series flag
  metricType?: string;       // Entity type context
}
```

## Provider-Specific Query Patterns

### AWS MSK Polling Queries

#### Cluster Health Query
```typescript
const DIM_QUERIES = {
  [METRIC_IDS.TOTAL_CLUSTERS]: {
    from: 'AwsMskClusterSample',
    select: ["uniqueCount(entity.guid) AS 'value'"],
    isNested: false
  },
  
  [METRIC_IDS.UNHEALTHY_CLUSTERS]: {
    from: {
      from: 'AwsMskClusterSample',
      select: [
        "latest(`provider.activeControllerCount.Sum`) AS 'activeControllers'",
        "latest(`provider.offlinePartitionsCount.Sum`) AS 'offlinePartitions'"
      ],
      facet: ['provider.clusterName as cluster'],
      limit: 'MAX'
    },
    select: ["uniqueCount(cluster) AS 'value'"],
    where: ['activeControllers != 1 OR offlinePartitions > 0'],
    isNested: true
  }
};
```

#### Throughput Aggregation Query
```sql
-- Incoming throughput by cluster
SELECT sum(bytesInPerSec)
FROM (
  SELECT average(provider.bytesInPerSec.Average) as 'bytesInPerSec'
  FROM AwsMskBrokerSample
  FACET provider.clusterName as cluster, provider.brokerId
  LIMIT MAX
)
WHERE provider.accountId = '${accountId}'
FACET cluster
TIMESERIES AUTO
```

### AWS MSK Metric Streams Queries

#### Metric Streams Health Query
```typescript
const MTS_QUERIES = {
  [METRIC_IDS.UNHEALTHY_CLUSTERS]: {
    from: {
      from: 'Metric',
      select: [
        "filter(sum(aws.kafka.ActiveControllerCount)/datapointCount(), " +
        "where metricName='aws.kafka.ActiveControllerCount') as 'Active Controllers'",
        "filter(sum(aws.kafka.OfflinePartitionsCount), " +
        "where metricName='aws.kafka.OfflinePartitionsCount') as 'Offline Partitions'"
      ],
      facet: ['aws.kafka.ClusterName OR aws.msk.clusterName AS cluster'],
      limit: 'MAX'
    },
    select: ["uniqueCount(cluster) AS 'value'"],
    where: ["`Active Controllers` != 1 OR `Offline Partitions` > 0"],
    isNested: true
  }
};
```

### Confluent Cloud Queries

#### Cluster Load Query
```sql
SELECT 
  (average(confluent_kafka_server_cluster_load_percent or 
           confluent.kafka.server.cluster_load_percent) * 100 or 0) 
    as 'Cluster load percent',
  filter(average(confluent_kafka_server_hot_partition_ingress or 
                 confluent.kafka.server.hot_partition_ingress), 
         WHERE (confluent_kafka_server_hot_partition_ingress or 
                confluent.kafka.server.hot_partition_ingress) = 1) 
    as 'Hot partition Ingress'
FROM Metric
WHERE metricName like '%confluent%'
FACET kafka.cluster_name or kafka.clusterName or confluent.clusterName
LIMIT MAX
```

## Complex Query Patterns

### Nested Aggregation Pattern

Used for multi-level aggregations:

```typescript
const nestedAggregationExample = {
  // Outer query - final aggregation
  from: {
    // Inner query - detailed data
    from: 'AwsMskBrokerSample',
    select: [
      'average(provider.bytesInPerSec.Average) as bytesIn',
      'average(provider.bytesOutPerSec.Average) as bytesOut'
    ],
    facet: ['provider.clusterName', 'provider.brokerId'],
    limit: 'MAX'
  },
  select: [
    'sum(bytesIn) as totalBytesIn',
    'sum(bytesOut) as totalBytesOut'
  ],
  facet: ['provider.clusterName'],
  isNested: true
};
```

### Dynamic Filter Application

```typescript
function getQueryString(
  queryDefinition: QueryModel,
  provider: string,
  isMetricStream: boolean,
  filterSet: string,
  facet?: string,
  timeRange?: TimeRanges,
  staticInfo?: StaticInfo
): QueryOptions {
  const model = new NRQLModel();
  
  // Build base query
  model.select(...queryDefinition.select);
  
  // Handle nested queries
  if (queryDefinition.isNested) {
    const innerQuery = buildInnerQuery(queryDefinition.from);
    model.from(`(${innerQuery})`);
  } else {
    model.from(queryDefinition.from);
  }
  
  // Apply filters
  const whereConditions = buildWhereConditions(
    queryDefinition,
    provider,
    isMetricStream,
    filterSet
  );
  
  if (whereConditions.length > 0) {
    model.where(whereConditions.join(' AND '));
  }
  
  // Apply facets
  if (facet || queryDefinition.facet) {
    model.facet(...(facet ? [facet] : queryDefinition.facet));
  }
  
  // Time series
  if (queryDefinition.isTimeseries) {
    model.timeseries();
  }
  
  // Time range
  if (timeRange) {
    model.since(timeRange);
  }
  
  return {
    query: model.toString(),
    ...staticInfo
  };
}
```

## Filter Processing System

### Provider-Specific Filter Functions

#### AWS Metric Streams Filter
```typescript
export const getAwsStreamWhere = (
  metricType: string,
  keyName: string,
  whereCond: string,
  isNavigator = false
) => {
  // Handle tag attributes specially
  if (METRIC_TAG_ATTRIBUTES.includes(keyName)) {
    return getClusterFilterCondition(isNavigator, whereCond);
  }
  
  // Entity type specific filtering
  if (metricType === 'Cluster') {
    if (['aws.msk.brokerId', 'aws.kafka.BrokerID'].includes(keyName)) {
      return `(aws.msk.brokerId OR aws.kafka.BrokerID) IN (
        SELECT ${isNavigator ? '' : 'uniques'}(
          aws.msk.brokerId OR aws.kafka.BrokerID
        ) 
        FROM Metric 
        WHERE ${whereCond}
        ${isNavigator ? 'LIMIT MAX' : ''}
      )`;
    }
  }
  
  return whereCond;
};
```

#### Confluent Cloud Filter
```typescript
export const confluentCloudTopicWhereCond = (
  whereCond: string,
  isNavigator = false,
  queryDefinition: any
) => {
  let whereClause = whereCond;
  
  if (!whereCond.includes("metricName like '%confluent%'")) {
    whereClause = `
      (kafka.cluster_name OR kafka.clusterName OR confluent.clusterName) 
      IN (
        SELECT ${isNavigator ? '' : 'uniques'}(
          kafka.cluster_name OR kafka.clusterName OR confluent.clusterName
        ) 
        FROM Metric 
        WHERE ${whereCond}
        ${isNavigator ? 'LIMIT MAX' : ''}
      )
    `;
    
    if (queryDefinition.metricType === 'Topic') {
      whereClause = `${whereCond} AND ${whereClause}`;
    }
  }
  
  return whereClause;
};
```

## Time Series Queries

### Throughput Over Time
```sql
-- Cluster throughput time series
SELECT 
  sum(bytesInPerSec) as 'Incoming',
  sum(bytesOutPerSec) as 'Outgoing'
FROM (
  SELECT 
    average(provider.bytesInPerSec.Average) as bytesInPerSec,
    average(provider.bytesOutPerSec.Average) as bytesOutPerSec
  FROM AwsMskBrokerSample
  FACET provider.clusterName, provider.brokerId
  LIMIT MAX
)
FACET provider.clusterName
TIMESERIES AUTO
SINCE 1 hour ago
```

### Message Rate Trends
```sql
-- Topic message rate over time
SELECT 
  average(provider.messagesInPerSec.Average) as 'Messages/sec'
FROM AwsMskTopicSample
FACET provider.topic
TIMESERIES 5 minutes
SINCE 24 hours ago
LIMIT 20
```

## Optimization Strategies

### 1. Query Structure Optimization

```typescript
const optimizationPatterns = {
  // Use nested queries for complex aggregations
  nestedAggregation: {
    good: `
      SELECT sum(metric) 
      FROM (
        SELECT average(metric) as metric 
        FROM Event 
        FACET dimension 
        LIMIT MAX
      )
    `,
    bad: `
      SELECT sum(average(metric)) 
      FROM Event
    `
  },
  
  // Limit data points for performance
  limitStrategy: {
    display: 'LIMIT 20',      // For UI display
    aggregation: 'LIMIT MAX', // For calculations
    pagination: 'LIMIT 100'   // For tables
  },
  
  // Strategic facet usage
  facetOptimization: {
    // Facet at appropriate level
    clusterLevel: 'FACET provider.clusterName',
    brokerLevel: 'FACET provider.clusterName, provider.brokerId',
    topicLevel: 'FACET provider.topic'
  }
};
```

### 2. Filter Optimization

```typescript
const filterOptimization = {
  // Use indexed fields
  indexed: [
    'entity.guid',
    'entity.type', 
    'provider.accountId',
    'eventType'
  ],
  
  // Avoid expensive operations
  avoid: [
    'LIKE with wildcards at start',
    'Complex regex patterns',
    'Large IN clauses'
  ],
  
  // Optimize WHERE conditions
  whereOptimization: (filters: Filter[]) => {
    // Put most selective filters first
    const sortedFilters = filters.sort((a, b) => 
      a.selectivity - b.selectivity
    );
    
    // Combine similar conditions
    const combinedFilters = combineFilters(sortedFilters);
    
    return combinedFilters.map(f => f.toWhereClause()).join(' AND ');
  }
};
```

### 3. Time Range Optimization

```typescript
const timeRangeOptimization = {
  // Default time ranges by query type
  defaults: {
    realtime: '5 minutes ago',
    operational: '1 hour ago',
    analytical: '24 hours ago',
    historical: '7 days ago'
  },
  
  // Adaptive time ranges based on data volume
  adaptive: (entityCount: number) => {
    if (entityCount > 1000) return '1 hour ago';
    if (entityCount > 100) return '3 hours ago';
    return '24 hours ago';
  },
  
  // Time bucket optimization
  timeBuckets: {
    '5 minutes': 'TIMESERIES 1 minute',
    '1 hour': 'TIMESERIES 5 minutes',
    '24 hours': 'TIMESERIES 1 hour',
    '7 days': 'TIMESERIES 6 hours'
  }
};
```

## Query Templates

### Dashboard Queries

```typescript
const DASHBOARD_QUERY_TEMPLATES = {
  // Summary billboards
  totalClusters: {
    aws: "SELECT uniqueCount(entity.guid) FROM AwsMskClusterSample",
    confluent: "SELECT uniqueCount(dimension.ConfluentResourceId) FROM Metric WHERE metricName like '%confluent%'"
  },
  
  // Health monitoring
  clusterHealth: {
    template: `
      SELECT 
        latest(activeControllers) as controllers,
        latest(offlinePartitions) as offline,
        latest(underReplicated) as underRep
      FROM (
        SELECT 
          {activeControllersMetric} as activeControllers,
          {offlinePartitionsMetric} as offlinePartitions,
          {underReplicatedMetric} as underReplicated
        FROM {eventType}
        FACET {clusterDimension}
      )
      WHERE {filters}
    `
  },
  
  // Performance metrics
  throughputByCluster: {
    template: `
      SELECT 
        sum(bytesIn) as 'Incoming (bytes/sec)',
        sum(bytesOut) as 'Outgoing (bytes/sec)'
      FROM (
        SELECT 
          {bytesInMetric} as bytesIn,
          {bytesOutMetric} as bytesOut
        FROM {eventType}
        FACET {clusterDimension}, {brokerDimension}
        LIMIT MAX
      )
      FACET {clusterDimension}
      TIMESERIES AUTO
    `
  }
};
```

### Entity Detail Queries

```typescript
const ENTITY_DETAIL_QUERIES = {
  // Cluster details
  clusterMetrics: `
    SELECT 
      latest(\`provider.activeControllerCount.Sum\`) as 'Active Controllers',
      latest(\`provider.offlinePartitionsCount.Sum\`) as 'Offline Partitions',
      latest(\`provider.globalPartitionCount.Maximum\`) as 'Total Partitions',
      latest(\`provider.globalTopicCount.Maximum\`) as 'Total Topics'
    FROM AwsMskClusterSample
    WHERE entity.guid = '{entityGuid}'
    SINCE 5 minutes ago
  `,
  
  // Broker details
  brokerMetrics: `
    SELECT 
      average(provider.cpuUser.Average) as 'CPU Usage %',
      average(provider.kafkaDataLogsDiskUsed.Average) as 'Disk Usage %',
      average(provider.bytesInPerSec.Average) as 'Bytes In/sec',
      average(provider.bytesOutPerSec.Average) as 'Bytes Out/sec'
    FROM AwsMskBrokerSample
    WHERE entity.guid = '{entityGuid}'
    TIMESERIES AUTO
    SINCE 1 hour ago
  `,
  
  // Topic details
  topicMetrics: `
    SELECT 
      average(provider.bytesInPerSec.Average) as 'Bytes In/sec',
      average(provider.bytesOutPerSec.Average) as 'Bytes Out/sec',
      average(provider.messagesInPerSec.Average) as 'Messages/sec'
    FROM AwsMskTopicSample
    WHERE entity.guid = '{entityGuid}'
    TIMESERIES AUTO
    SINCE 1 hour ago
  `
};
```

## Advanced Query Patterns

### Cross-Account Queries

```typescript
const crossAccountQuery = {
  query: `
    SELECT count(*) 
    FROM AwsMskClusterSample
    WHERE provider.accountId IN (${accountIds.join(',')})
    FACET provider.accountId, provider.clusterName
    LIMIT MAX
  `,
  
  // Using NerdGraph for cross-account
  graphql: ngql`
    query CrossAccountClusters($accountIds: [Int!]!) {
      actor {
        accounts(ids: $accountIds) {
          id
          name
          nrql(query: "FROM AwsMskClusterSample SELECT count(*)") {
            results
          }
        }
      }
    }
  `
};
```

### Relationship Queries

```sql
-- Find topics with their producing applications
SELECT 
  latest(provider.topic) as topic,
  latest(appName) as producer
FROM AwsMskTopicSample, Transaction
WHERE 
  provider.topic = kafkaProducerTopic AND
  appName IS NOT NULL
FACET provider.topic, appName
SINCE 1 hour ago
```

### Anomaly Detection Queries

```sql
-- Detect throughput anomalies
SELECT 
  average(bytesInPerSec) as current,
  stddev(bytesInPerSec) as deviation,
  average(bytesInPerSec) + (3 * stddev(bytesInPerSec)) as upperBound,
  average(bytesInPerSec) - (3 * stddev(bytesInPerSec)) as lowerBound
FROM (
  SELECT average(provider.bytesInPerSec.Average) as bytesInPerSec
  FROM AwsMskBrokerSample
  FACET provider.clusterName
  LIMIT MAX
)
COMPARE WITH 1 hour ago
```

## Query Performance Best Practices

### 1. Use Appropriate Limits
```sql
-- For display
LIMIT 20

-- For aggregations
LIMIT MAX

-- For exploration
LIMIT 100
```

### 2. Optimize Time Windows
```sql
-- Real-time monitoring
SINCE 5 minutes ago

-- Operational view
SINCE 1 hour ago

-- Historical analysis
SINCE 7 days ago
```

### 3. Strategic Faceting
```sql
-- Cluster level for overview
FACET provider.clusterName

-- Detailed for drill-down
FACET provider.clusterName, provider.brokerId, provider.topic
```

### 4. Efficient Filtering
```sql
-- Use indexed fields first
WHERE entity.type = 'AWSMSKCLUSTER' 
  AND provider.accountId = '123456'
  AND provider.clusterName LIKE 'prod%'
```

## Conclusion

The NRQL query system provides powerful capabilities for Kafka monitoring. By understanding query patterns, optimization strategies, and provider-specific implementations, you can build efficient and effective monitoring solutions within the Message Queues application.