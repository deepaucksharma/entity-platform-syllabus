# Data Flow & Processing

## Overview

The Message Queues monitoring system implements a sophisticated data pipeline that transforms raw Kafka telemetry into actionable insights. This document details the complete data flow from ingestion through visualization.

## Data Flow Architecture

### High-Level Data Flow

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Data Sources  │────▶│ New Relic Ingest │────▶│ Entity Platform │
└─────────────────┘     └──────────────────┘     └────────┬────────┘
                                                           │
┌─────────────────┐     ┌──────────────────┐     ┌────────▼────────┐
│  UI Components  │◀────│ Query Processing │◀────│   Data Store    │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

### Detailed Component Flow

```typescript
interface DataFlowPipeline {
  // 1. Data Ingestion
  ingestion: {
    awsMsk: {
      polling: CloudWatchPollingIntegration;
      metricStreams: KinesisMetricStreams;
    };
    confluent: {
      api: ConfluentMetricsAPI;
    };
  };
  
  // 2. Entity Synthesis
  synthesis: {
    telemetryToEntity: TelemetryProcessor;
    relationshipDiscovery: RelationshipEngine;
    metadataEnrichment: MetadataEnricher;
  };
  
  // 3. Query Processing
  queryProcessing: {
    graphql: NerdGraphProcessor;
    nrql: NRQLEngine;
    aggregation: AggregationEngine;
  };
  
  // 4. Data Transformation
  transformation: {
    normalization: DataNormalizer;
    calculation: MetricCalculator;
    filtering: FilterEngine;
  };
  
  // 5. Presentation
  presentation: {
    visualization: VisualizationEngine;
    caching: CacheLayer;
    realtime: RealtimeUpdater;
  };
}
```

## Data Ingestion Layer

### AWS MSK Data Ingestion

#### CloudWatch Polling Integration
```typescript
// Polling data flow
const MSKPollingFlow = {
  // 1. CloudWatch collects metrics
  source: 'AWS CloudWatch',
  interval: '5 minutes',
  
  // 2. New Relic polls CloudWatch
  collection: {
    api: 'CloudWatch GetMetricStatistics',
    frequency: '5 minutes',
    metrics: [
      'ActiveControllerCount',
      'OfflinePartitionsCount',
      'BytesInPerSec',
      'BytesOutPerSec'
    ]
  },
  
  // 3. Transform to New Relic events
  transformation: {
    eventTypes: [
      'AwsMskClusterSample',
      'AwsMskBrokerSample',
      'AwsMskTopicSample'
    ],
    attributes: {
      'provider.clusterName': 'ClusterName',
      'provider.brokerId': 'BrokerID',
      'provider.topic': 'Topic',
      'provider.accountId': 'AWS Account ID'
    }
  }
};
```

#### Metric Streams Integration
```typescript
// Metric Streams real-time flow
const MSKMetricStreamsFlow = {
  // 1. CloudWatch Metric Streams
  source: 'AWS Kinesis Data Firehose',
  latency: '< 1 minute',
  
  // 2. Direct to New Relic
  delivery: {
    endpoint: 'New Relic Metric API',
    format: 'OpenTelemetry 0.7',
    compression: 'GZIP'
  },
  
  // 3. Metric namespacing
  transformation: {
    namespace: 'aws.kafka.*',
    dimensions: {
      'aws.kafka.ClusterName': 'Cluster identifier',
      'aws.kafka.BrokerID': 'Broker identifier',
      'aws.kafka.Topic': 'Topic name'
    }
  }
};
```

### Confluent Cloud Data Ingestion

```typescript
const ConfluentDataFlow = {
  // 1. Confluent Metrics API
  source: 'Confluent Cloud Metrics API',
  endpoint: 'https://api.telemetry.confluent.cloud/v2/metrics',
  
  // 2. Collection process
  collection: {
    method: 'HTTP POST',
    interval: '1 minute',
    authentication: 'API Key + Secret'
  },
  
  // 3. Metric transformation
  transformation: {
    eventType: 'Metric',
    namespace: 'confluent.*',
    metricMapping: {
      'io.confluent.kafka.server/received_bytes': 'bytesInPerSec',
      'io.confluent.kafka.server/sent_bytes': 'bytesOutPerSec',
      'io.confluent.kafka.server/received_records': 'messagesInPerSec'
    }
  }
};
```

## Entity Synthesis Pipeline

### Telemetry to Entity Transformation

```typescript
class EntitySynthesisPipeline {
  // Process incoming telemetry
  processTelemetry(event: TelemetryEvent): Entity | null {
    // 1. Identify entity type
    const entityType = this.identifyEntityType(event);
    if (!entityType) return null;
    
    // 2. Extract entity attributes
    const attributes = this.extractAttributes(event, entityType);
    
    // 3. Generate entity GUID
    const guid = this.generateGuid(attributes, entityType);
    
    // 4. Create/update entity
    return this.synthesizeEntity({
      guid,
      type: entityType,
      name: attributes.name,
      attributes,
      timestamp: event.timestamp
    });
  }
  
  private identifyEntityType(event: TelemetryEvent): EntityType | null {
    const typeMapping = {
      'AwsMskClusterSample': EntityType.AWSMSKCLUSTER,
      'AwsMskBrokerSample': EntityType.AWSMSKBROKER,
      'AwsMskTopicSample': EntityType.AWSMSKTOPIC,
      'Metric': this.identifyMetricEntityType(event)
    };
    
    return typeMapping[event.eventType] || null;
  }
  
  private extractAttributes(event: TelemetryEvent, type: EntityType): EntityAttributes {
    const extractors = {
      [EntityType.AWSMSKCLUSTER]: this.extractClusterAttributes,
      [EntityType.AWSMSKBROKER]: this.extractBrokerAttributes,
      [EntityType.AWSMSKTOPIC]: this.extractTopicAttributes
    };
    
    return extractors[type](event);
  }
}
```

### Relationship Discovery

```typescript
class RelationshipDiscoveryEngine {
  discoverRelationships(entity: Entity): Relationship[] {
    const relationships: Relationship[] = [];
    
    // 1. Parent-child relationships
    if (entity.type === EntityType.AWSMSKBROKER) {
      relationships.push({
        type: RelationshipType.CONTAINS,
        source: this.findClusterEntity(entity.attributes.clusterName),
        target: entity,
        bidirectional: true
      });
    }
    
    // 2. Producer/consumer relationships
    if (entity.type === EntityType.AWSMSKTOPIC) {
      const producers = this.findProducerApplications(entity.attributes.topicName);
      const consumers = this.findConsumerApplications(entity.attributes.topicName);
      
      producers.forEach(producer => {
        relationships.push({
          type: RelationshipType.PRODUCES,
          source: producer,
          target: entity
        });
      });
      
      consumers.forEach(consumer => {
        relationships.push({
          type: RelationshipType.CONSUMES,
          source: consumer,
          target: entity
        });
      });
    }
    
    return relationships;
  }
}
```

## Query Processing Flow

### Query Execution Pipeline

```typescript
class QueryProcessor {
  async executeQuery(request: QueryRequest): Promise<QueryResult> {
    // 1. Query parsing and validation
    const parsedQuery = this.parseQuery(request);
    this.validateQuery(parsedQuery);
    
    // 2. Query optimization
    const optimizedQuery = this.optimizeQuery(parsedQuery);
    
    // 3. Cache check
    const cachedResult = await this.checkCache(optimizedQuery);
    if (cachedResult) return cachedResult;
    
    // 4. Execute query
    const result = await this.executeOptimizedQuery(optimizedQuery);
    
    // 5. Post-processing
    const processedResult = this.postProcess(result, request);
    
    // 6. Cache result
    await this.cacheResult(optimizedQuery, processedResult);
    
    return processedResult;
  }
  
  private optimizeQuery(query: ParsedQuery): OptimizedQuery {
    // Apply optimization strategies
    return {
      ...query,
      // Use appropriate limits
      limit: this.optimizeLimit(query),
      // Add efficient filters
      filters: this.optimizeFilters(query.filters),
      // Choose best aggregation level
      aggregation: this.optimizeAggregation(query)
    };
  }
}
```

### Filter Processing Flow

```typescript
class FilterProcessor {
  processFilters(filters: Filter[], context: QueryContext): ProcessedFilters {
    // 1. Normalize filters
    const normalized = this.normalizeFilters(filters);
    
    // 2. Validate filter compatibility
    this.validateFilterCompatibility(normalized, context);
    
    // 3. Apply provider-specific transformations
    const transformed = this.applyProviderTransformations(normalized, context.provider);
    
    // 4. Optimize filter order
    const optimized = this.optimizeFilterOrder(transformed);
    
    // 5. Generate WHERE clauses
    return {
      whereClause: this.generateWhereClause(optimized),
      appliedFilters: optimized,
      metadata: this.generateFilterMetadata(optimized)
    };
  }
  
  private applyProviderTransformations(filters: Filter[], provider: Provider): Filter[] {
    const transformers = {
      [Provider.AWS_MSK]: this.transformAWSFilters,
      [Provider.CONFLUENT]: this.transformConfluentFilters
    };
    
    return transformers[provider](filters);
  }
}
```

## Data Transformation Layer

### Metric Calculation Pipeline

```typescript
class MetricCalculator {
  calculateDerivedMetrics(rawData: RawMetricData): DerivedMetrics {
    return {
      // Health calculations
      healthStatus: this.calculateHealthStatus(rawData),
      healthScore: this.calculateHealthScore(rawData),
      
      // Throughput calculations
      totalThroughput: this.calculateTotalThroughput(rawData),
      throughputTrend: this.calculateThroughputTrend(rawData),
      
      // Efficiency metrics
      utilizationRate: this.calculateUtilization(rawData),
      efficiencyScore: this.calculateEfficiency(rawData),
      
      // Anomaly detection
      anomalyScore: this.calculateAnomalyScore(rawData),
      outliers: this.detectOutliers(rawData)
    };
  }
  
  private calculateHealthStatus(data: RawMetricData): HealthStatus {
    // AWS MSK health logic
    if (data.provider === Provider.AWS_MSK) {
      if (data.activeControllers !== 1) return HealthStatus.CRITICAL;
      if (data.offlinePartitions > 0) return HealthStatus.CRITICAL;
      if (data.underReplicatedPartitions > 0) return HealthStatus.WARNING;
      return HealthStatus.HEALTHY;
    }
    
    // Confluent health logic
    if (data.provider === Provider.CONFLUENT) {
      if (data.clusterLoadPercent > 90) return HealthStatus.CRITICAL;
      if (data.clusterLoadPercent > 70) return HealthStatus.WARNING;
      if (data.hotPartitions > 0) return HealthStatus.WARNING;
      return HealthStatus.HEALTHY;
    }
  }
}
```

### Data Aggregation Engine

```typescript
class AggregationEngine {
  aggregate(data: TimeSeriesData[], config: AggregationConfig): AggregatedData {
    // 1. Group by dimensions
    const grouped = this.groupByDimensions(data, config.dimensions);
    
    // 2. Apply aggregation functions
    const aggregated = Object.entries(grouped).map(([key, values]) => ({
      dimension: key,
      metrics: this.applyAggregationFunctions(values, config.functions)
    }));
    
    // 3. Apply time bucketing
    const bucketed = this.applyTimeBucketing(aggregated, config.timeBucket);
    
    // 4. Calculate rollups
    const rollups = this.calculateRollups(bucketed, config.rollupLevels);
    
    return {
      data: bucketed,
      rollups,
      metadata: this.generateAggregationMetadata(config)
    };
  }
  
  private applyAggregationFunctions(
    values: MetricValue[],
    functions: AggregationFunction[]
  ): AggregatedMetrics {
    const results = {};
    
    functions.forEach(func => {
      switch (func.type) {
        case 'average':
          results[func.name] = this.average(values);
          break;
        case 'sum':
          results[func.name] = this.sum(values);
          break;
        case 'percentile':
          results[func.name] = this.percentile(values, func.value);
          break;
        case 'rate':
          results[func.name] = this.rate(values);
          break;
      }
    });
    
    return results;
  }
}
```

## Data Transformation Implementation (Actual)

### Entity Data Transformation (from data-utils.ts)

```typescript
// From data-utils.ts - Actual transformation functions
import { ALERT_SEVERITY_ORDER } from '../config/constants';

// Transform entity groups for HoneyComb visualization
export const prepareEntityGroups = (
  EntityMetrics: any,
  provider: string,
  show: string,
  groupBy?: string
) => {
  if (!EntityMetrics || Object.keys(EntityMetrics).length === 0) {
    return [];
  }

  // Group entities by their type or custom grouping
  const groups = {};
  
  Object.entries(EntityMetrics).forEach(([key, entity]: [string, any]) => {
    const groupKey = groupBy ? entity[groupBy] : entity.type || 'default';
    
    if (!groups[groupKey]) {
      groups[groupKey] = {
        name: groupKey,
        provider,
        counts: []
      };
    }
    
    groups[groupKey].counts.push({
      [show]: entity.name || key,
      type: entity.alertSeverity || entity.healthStatus || 'HEALTHY',
      count: entity.count || 1,
      ...entity
    });
  });

  // Convert to array and sort
  return Object.values(groups).map((group: any) => ({
    ...group,
    counts: group.counts.sort((a: any, b: any) => {
      // Sort by severity if available
      if (a.type && b.type && ALERT_SEVERITY_ORDER.includes(a.type)) {
        return ALERT_SEVERITY_ORDER.indexOf(a.type) - 
               ALERT_SEVERITY_ORDER.indexOf(b.type);
      }
      return 0;
    })
  }));
};

// Transform topics data for table display
export const prepareTopicsTableData = (
  topicsData: any[],
  provider: string
) => {
  if (!topicsData || !Array.isArray(topicsData)) {
    return [];
  }

  return topicsData.map(topic => ({
    guid: topic.guid,
    'Topic Name': topic.Name || topic.name || topic.Topic,
    'Incoming Throughput': topic.bytesInPerSec || 0,
    'Outgoing Throughput': topic.bytesOutPerSec || 0,
    'Message rate': topic.messagesInPerSec || 0,
    provider,
    // Additional metadata
    clusterName: topic.clusterName,
    partitionCount: topic.partitionCount,
    replicationFactor: topic.replicationFactor
  }));
};

// Add commas to numbers for display
export const addCommasToNumber = (num: number | string) => {
  if (num === null || num === undefined) return '0';
  
  const number = typeof num === 'string' ? parseFloat(num) : num;
  
  if (isNaN(number)) return '0';
  
  return number.toString().replace(/\B(?=(\d{3})+(?!\d))/g, ',');
};
```

### Health Score Calculation (Actual Implementation)

```typescript
// Health calculation patterns from actual codebase
export const calculateHealthScore = (metrics: any, entityType: string) => {
  switch (entityType) {
    case 'cluster':
      return calculateClusterHealth(metrics);
    case 'broker':
      return calculateBrokerHealth(metrics);
    case 'topic':
      return calculateTopicHealth(metrics);
    default:
      return 100; // Default healthy
  }
};

const calculateClusterHealth = (metrics: any) => {
  // AWS MSK cluster health logic
  if (metrics['Active Controllers'] !== 1) {
    return 0; // Critical - no active controller
  }
  
  if (metrics['Offline Partitions'] > 0) {
    return 25; // Critical - offline partitions
  }
  
  if (metrics['Under Replicated Partitions'] > 0) {
    return 75; // Warning - under replicated
  }
  
  return 100; // Healthy
};

const calculateBrokerHealth = (metrics: any) => {
  const underMinISR = metrics['Under Min ISR Partitions'] || 0;
  const underReplicated = metrics['Under Replicated Partitions'] || 0;
  
  if (underMinISR > 0 || underReplicated > 0) {
    // Calculate degradation based on partition issues
    const totalIssues = underMinISR + underReplicated;
    const healthScore = Math.max(0, 100 - (totalIssues * 10));
    return healthScore;
  }
  
  return 100;
};

const calculateTopicHealth = (metrics: any) => {
  // Simple throughput-based health for topics
  const bytesIn = metrics['Bytes In'] || metrics.bytesInPerSec || 0;
  const bytesOut = metrics['Bytes Out'] || metrics.bytesOutPerSec || 0;
  
  // Topic is healthy if it has any activity
  if (bytesIn > 0 || bytesOut > 0) {
    return 100;
  }
  
  // No activity might indicate issues
  return 50;
};

// Alert severity ordering constant
export const ALERT_SEVERITY_ORDER = [
  'CRITICAL',
  'HIGH', 
  'WARNING',
  'LOW',
  'NOT_CONFIGURED',
  'NOT_ALERTING'
];
```

## Real-Time Data Processing

### Streaming Data Pipeline

```typescript
class StreamingDataProcessor {
  private subscriptions: Map<string, Subscription> = new Map();
  
  // Subscribe to real-time updates
  subscribe(entityGuid: string, callback: DataCallback): Subscription {
    const subscription = {
      id: generateId(),
      entityGuid,
      callback,
      interval: this.getOptimalInterval(entityGuid)
    };
    
    // Set up polling/streaming
    const intervalId = setInterval(() => {
      this.fetchLatestData(entityGuid).then(data => {
        const processed = this.processRealtimeData(data);
        callback(processed);
      });
    }, subscription.interval);
    
    subscription.cleanup = () => clearInterval(intervalId);
    this.subscriptions.set(subscription.id, subscription);
    
    return subscription;
  }
  
  private processRealtimeData(data: RawData): ProcessedData {
    return {
      // Current values
      current: data,
      
      // Calculate deltas
      delta: this.calculateDelta(data),
      
      // Moving averages
      movingAverage: this.calculateMovingAverage(data),
      
      // Trend analysis
      trend: this.analyzeTrend(data),
      
      // Anomaly detection
      anomalies: this.detectAnomalies(data),
      
      // Timestamp
      timestamp: Date.now()
    };
  }
}
```

### Cache Management

```typescript
class CacheManager {
  private cache: LRUCache<string, CachedData>;
  
  constructor() {
    this.cache = new LRUCache({
      max: 1000,
      ttl: 1000 * 60 * 5, // 5 minutes default
      updateAgeOnGet: true
    });
  }
  
  async get(key: string): Promise<CachedData | null> {
    const cached = this.cache.get(key);
    
    if (!cached) return null;
    
    // Check if stale
    if (this.isStale(cached)) {
      // Async refresh
      this.refreshInBackground(key);
    }
    
    return cached;
  }
  
  set(key: string, data: any, options?: CacheOptions): void {
    const ttl = this.calculateTTL(key, options);
    
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl,
      metadata: options?.metadata
    });
  }
  
  private calculateTTL(key: string, options?: CacheOptions): number {
    // Entity data: longer TTL
    if (key.includes('entity')) return 1000 * 60 * 10; // 10 minutes
    
    // Metrics: shorter TTL
    if (key.includes('metrics')) return 1000 * 60; // 1 minute
    
    // Custom TTL
    if (options?.ttl) return options.ttl;
    
    // Default
    return 1000 * 60 * 5; // 5 minutes
  }
}
```

## Data Validation & Quality

### Data Validation Pipeline

```typescript
class DataValidator {
  validate(data: IncomingData): ValidationResult {
    const errors: ValidationError[] = [];
    const warnings: ValidationWarning[] = [];
    
    // 1. Schema validation
    const schemaErrors = this.validateSchema(data);
    errors.push(...schemaErrors);
    
    // 2. Data quality checks
    const qualityIssues = this.checkDataQuality(data);
    warnings.push(...qualityIssues);
    
    // 3. Business rule validation
    const ruleViolations = this.validateBusinessRules(data);
    errors.push(...ruleViolations);
    
    // 4. Consistency checks
    const inconsistencies = this.checkConsistency(data);
    warnings.push(...inconsistencies);
    
    return {
      valid: errors.length === 0,
      errors,
      warnings,
      sanitizedData: this.sanitizeData(data)
    };
  }
  
  private checkDataQuality(data: IncomingData): ValidationWarning[] {
    const warnings: ValidationWarning[] = [];
    
    // Check for missing optional fields
    if (!data.metadata?.region) {
      warnings.push({
        field: 'metadata.region',
        message: 'Region information missing',
        severity: 'low'
      });
    }
    
    // Check for suspicious values
    if (data.metrics?.throughput > 1e9) { // 1GB/s
      warnings.push({
        field: 'metrics.throughput',
        message: 'Unusually high throughput value',
        severity: 'medium'
      });
    }
    
    return warnings;
  }
}
```

## Error Handling & Recovery

### Error Recovery Pipeline

```typescript
class ErrorRecoveryManager {
  async handleDataError(error: DataError, context: ProcessingContext): Promise<RecoveryResult> {
    // 1. Classify error
    const errorType = this.classifyError(error);
    
    // 2. Apply recovery strategy
    const strategy = this.getRecoveryStrategy(errorType);
    
    try {
      const result = await this.executeRecovery(strategy, error, context);
      
      // 3. Log recovery
      this.logRecovery(error, strategy, result);
      
      return result;
    } catch (recoveryError) {
      // 4. Fallback to degraded mode
      return this.degradedModeResponse(context);
    }
  }
  
  private getRecoveryStrategy(errorType: ErrorType): RecoveryStrategy {
    const strategies = {
      [ErrorType.NETWORK]: {
        retry: { count: 3, backoff: 'exponential' },
        fallback: 'cached_data'
      },
      [ErrorType.PARSING]: {
        sanitize: true,
        fallback: 'partial_data'
      },
      [ErrorType.VALIDATION]: {
        sanitize: true,
        notify: true,
        fallback: 'default_values'
      }
    };
    
    return strategies[errorType] || strategies[ErrorType.UNKNOWN];
  }
}
```

## Performance Monitoring

### Data Pipeline Metrics

```typescript
class PipelineMonitor {
  private metrics: PipelineMetrics = {
    ingestion: new MetricCollector('ingestion'),
    processing: new MetricCollector('processing'),
    query: new MetricCollector('query'),
    cache: new MetricCollector('cache')
  };
  
  recordMetric(stage: PipelineStage, metric: Metric): void {
    this.metrics[stage].record(metric);
    
    // Alert on anomalies
    if (this.isAnomalous(stage, metric)) {
      this.alertOnAnomaly(stage, metric);
    }
  }
  
  generateReport(): PipelineReport {
    return {
      timestamp: Date.now(),
      stages: Object.entries(this.metrics).map(([stage, collector]) => ({
        stage,
        metrics: collector.getMetrics(),
        health: this.calculateStageHealth(collector)
      })),
      overall: this.calculateOverallHealth()
    };
  }
}
```

## Best Practices

### 1. Data Ingestion
- Validate data at ingestion point
- Handle partial data gracefully
- Implement retry mechanisms

### 2. Processing
- Use streaming for real-time data
- Batch processing for efficiency
- Implement circuit breakers

### 3. Caching
- Cache at multiple levels
- Use appropriate TTLs
- Implement cache warming

### 4. Error Handling
- Fail gracefully
- Provide meaningful errors
- Log for debugging

### 5. Performance
- Monitor pipeline metrics
- Optimize bottlenecks
- Scale horizontally

## Conclusion

The data flow and processing pipeline ensures reliable, efficient transformation of Kafka telemetry into actionable insights. By implementing proper validation, caching, and error handling, the system maintains high performance and reliability at scale.