# APM Integration

## Overview

The Message Queues monitoring system provides deep integration with New Relic APM (Application Performance Monitoring) to correlate Kafka infrastructure metrics with application performance. This enables users to understand the impact of message queue performance on their applications and vice versa.

## Integration Architecture

### APM Correlation Framework

```
┌─────────────────────────────────────────────────────────┐
│                Application Layer (APM)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  Producer   │  │  Consumer   │  │   Stream    │   │
│  │   Apps      │  │    Apps     │  │ Processing  │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
├─────────────────────────────────────────────────────────┤
│              Correlation Layer                          │
│  ┌──────────────────────────────────────────────────┐  │
│  │  • Transaction Tracing                           │  │
│  │  • Distributed Tracing                           │  │
│  │  • Entity Relationships                          │  │
│  │  • Metric Correlation                            │  │
│  └──────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│            Infrastructure Layer (Kafka)                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   Topics    │  │   Brokers   │  │  Clusters   │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Entity Relationship Mapping

### Relationship Discovery

```typescript
interface EntityRelationship {
  sourceGuid: string;
  targetGuid: string;
  relationshipType: 'PRODUCES_TO' | 'CONSUMES_FROM' | 'CONNECTS_TO';
  metadata?: {
    topicName?: string;
    consumerGroup?: string;
    throughput?: number;
  };
}

class APMRelationshipDiscovery {
  async discoverRelationships(accountId: string): Promise<EntityRelationship[]> {
    // Query for APM entities with Kafka instrumentation
    const apmQuery = ngql`
      query DiscoverKafkaRelationships($accountId: Int!) {
        actor {
          entitySearch(query: "accountId = $accountId AND type IN ('APPLICATION')") {
            results {
              entities {
                guid
                name
                type
                tags {
                  key
                  values
                }
                relationships {
                  source {
                    entity {
                      guid
                      type
                    }
                  }
                  target {
                    entity {
                      guid
                      type
                    }
                  }
                  type
                }
                ... on ApmApplicationEntity {
                  settings {
                    kafkaProducerTopics
                    kafkaConsumerTopics
                    kafkaConsumerGroups
                  }
                }
              }
            }
          }
        }
      }
    `;
    
    const result = await NerdGraphQuery.query({
      query: apmQuery,
      variables: { accountId }
    });
    
    return this.extractRelationships(result.data);
  }
  
  private extractRelationships(data: any): EntityRelationship[] {
    const relationships: EntityRelationship[] = [];
    
    data.actor.entitySearch.results.entities.forEach(app => {
      // Producer relationships
      if (app.settings?.kafkaProducerTopics) {
        app.settings.kafkaProducerTopics.forEach(topic => {
          relationships.push({
            sourceGuid: app.guid,
            targetGuid: this.getTopicGuid(topic),
            relationshipType: 'PRODUCES_TO',
            metadata: { topicName: topic }
          });
        });
      }
      
      // Consumer relationships
      if (app.settings?.kafkaConsumerTopics) {
        app.settings.kafkaConsumerTopics.forEach((topic, index) => {
          relationships.push({
            sourceGuid: this.getTopicGuid(topic),
            targetGuid: app.guid,
            relationshipType: 'CONSUMES_FROM',
            metadata: {
              topicName: topic,
              consumerGroup: app.settings.kafkaConsumerGroups?.[index]
            }
          });
        });
      }
    });
    
    return relationships;
  }
}
```

### Transaction Tracing

```typescript
interface KafkaTransaction {
  transactionId: string;
  applicationGuid: string;
  operation: 'produce' | 'consume';
  topic: string;
  partition?: number;
  offset?: number;
  timestamp: number;
  duration: number;
  success: boolean;
  errorDetails?: string;
}

class KafkaTransactionTracer {
  // Trace producer transactions
  async traceProducerTransactions(appGuid: string): Promise<KafkaTransaction[]> {
    const query = `
      FROM Transaction
      SELECT 
        guid,
        appId,
        name,
        duration,
        timestamp,
        kafka.topic,
        kafka.partition,
        kafka.offset,
        error
      WHERE appGuid = '${appGuid}'
        AND name LIKE '%kafka%produce%'
      SINCE 1 hour ago
      LIMIT 1000
    `;
    
    const results = await this.executeNRQL(query);
    
    return results.map(this.mapToKafkaTransaction);
  }
  
  // Trace consumer transactions
  async traceConsumerTransactions(appGuid: string): Promise<KafkaTransaction[]> {
    const query = `
      FROM Transaction
      SELECT 
        guid,
        appId,
        name,
        duration,
        timestamp,
        kafka.topic,
        kafka.partition,
        kafka.offset,
        kafka.consumer.group,
        kafka.consumer.lag,
        error
      WHERE appGuid = '${appGuid}'
        AND name LIKE '%kafka%consume%'
      SINCE 1 hour ago
      LIMIT 1000
    `;
    
    const results = await this.executeNRQL(query);
    
    return results.map(this.mapToKafkaTransaction);
  }
  
  // Correlate with infrastructure metrics
  async correlateWithInfrastructure(
    transaction: KafkaTransaction
  ): Promise<CorrelatedMetrics> {
    const timeWindow = {
      start: transaction.timestamp - 60000, // 1 minute before
      end: transaction.timestamp + 60000    // 1 minute after
    };
    
    // Get topic metrics during transaction time
    const topicMetrics = await this.getTopicMetrics(
      transaction.topic,
      timeWindow
    );
    
    // Get broker metrics
    const brokerMetrics = await this.getBrokerMetrics(
      transaction.topic,
      transaction.partition,
      timeWindow
    );
    
    return {
      transaction,
      infrastructure: {
        topic: topicMetrics,
        broker: brokerMetrics,
        correlation: this.calculateCorrelation(transaction, topicMetrics)
      }
    };
  }
}
```

## Distributed Tracing Integration

### Trace Context Propagation

```typescript
interface KafkaTraceContext {
  traceId: string;
  spanId: string;
  parentSpanId?: string;
  baggage?: Record<string, string>;
}

class DistributedTracingIntegration {
  // Extract trace context from Kafka headers
  extractTraceContext(headers: KafkaHeaders): KafkaTraceContext | null {
    const traceParent = headers['traceparent'];
    const traceState = headers['tracestate'];
    
    if (!traceParent) return null;
    
    // Parse W3C Trace Context format
    const parts = traceParent.split('-');
    if (parts.length !== 4) return null;
    
    return {
      traceId: parts[1],
      spanId: parts[2],
      parentSpanId: parts[3],
      baggage: this.parseBaggage(headers['baggage'])
    };
  }
  
  // Create distributed trace view
  async buildDistributedTrace(traceId: string): Promise<DistributedTrace> {
    // Get all spans for this trace
    const spans = await this.fetchTraceSpans(traceId);
    
    // Build trace tree
    const trace = this.buildTraceTree(spans);
    
    // Enhance with Kafka-specific information
    return this.enhanceWithKafkaContext(trace);
  }
  
  private async fetchTraceSpans(traceId: string): Promise<TraceSpan[]> {
    const query = ngql`
      query GetTraceSpans($traceId: String!) {
        actor {
          distributedTrace(traceId: $traceId) {
            spans {
              id
              parentId
              name
              timestamp
              duration
              serviceName
              attributes
              events {
                name
                timestamp
                attributes
              }
            }
          }
        }
      }
    `;
    
    const result = await NerdGraphQuery.query({
      query,
      variables: { traceId }
    });
    
    return result.data.actor.distributedTrace.spans;
  }
  
  private enhanceWithKafkaContext(trace: TraceTree): DistributedTrace {
    trace.spans.forEach(span => {
      // Identify Kafka operations
      if (this.isKafkaSpan(span)) {
        span.kafkaContext = {
          operation: this.getKafkaOperation(span),
          topic: span.attributes['messaging.destination'],
          partition: span.attributes['messaging.kafka.partition'],
          offset: span.attributes['messaging.kafka.offset'],
          consumerGroup: span.attributes['messaging.kafka.consumer_group']
        };
      }
    });
    
    return trace as DistributedTrace;
  }
}
```

### Performance Impact Analysis

```typescript
class PerformanceImpactAnalyzer {
  // Analyze impact of Kafka latency on application performance
  async analyzeKafkaImpact(
    applicationGuid: string,
    timeRange: TimeRange
  ): Promise<ImpactAnalysis> {
    // Get application performance metrics
    const appMetrics = await this.getApplicationMetrics(applicationGuid, timeRange);
    
    // Get Kafka operation metrics
    const kafkaMetrics = await this.getKafkaOperationMetrics(applicationGuid, timeRange);
    
    // Correlate metrics
    const correlation = this.calculateCorrelations(appMetrics, kafkaMetrics);
    
    // Identify bottlenecks
    const bottlenecks = this.identifyBottlenecks(kafkaMetrics);
    
    return {
      summary: {
        kafkaOperationsPercentage: this.calculateKafkaPercentage(appMetrics),
        averageKafkaLatency: this.calculateAverageLatency(kafkaMetrics),
        impactScore: this.calculateImpactScore(correlation)
      },
      correlations: correlation,
      bottlenecks: bottlenecks,
      recommendations: this.generateRecommendations(bottlenecks, correlation)
    };
  }
  
  private calculateCorrelations(
    appMetrics: ApplicationMetrics,
    kafkaMetrics: KafkaOperationMetrics
  ): CorrelationResult {
    return {
      responseTimeCorrelation: this.pearsonCorrelation(
        appMetrics.responseTimes,
        kafkaMetrics.latencies
      ),
      throughputCorrelation: this.pearsonCorrelation(
        appMetrics.throughput,
        kafkaMetrics.messageRate
      ),
      errorRateCorrelation: this.pearsonCorrelation(
        appMetrics.errorRates,
        kafkaMetrics.failureRates
      )
    };
  }
  
  private identifyBottlenecks(
    kafkaMetrics: KafkaOperationMetrics
  ): Bottleneck[] {
    const bottlenecks: Bottleneck[] = [];
    
    // Check producer bottlenecks
    if (kafkaMetrics.producerLatency.p95 > 100) {
      bottlenecks.push({
        type: 'PRODUCER_LATENCY',
        severity: 'HIGH',
        description: 'High producer latency detected',
        metrics: {
          p95Latency: kafkaMetrics.producerLatency.p95,
          affectedTransactions: kafkaMetrics.slowProducerTransactions
        }
      });
    }
    
    // Check consumer lag
    if (kafkaMetrics.consumerLag.max > 10000) {
      bottlenecks.push({
        type: 'CONSUMER_LAG',
        severity: 'CRITICAL',
        description: 'Significant consumer lag detected',
        metrics: {
          maxLag: kafkaMetrics.consumerLag.max,
          affectedConsumerGroups: kafkaMetrics.laggedConsumerGroups
        }
      });
    }
    
    return bottlenecks;
  }
}
```

## Application Topology Visualization

### Dynamic Topology Discovery

```typescript
interface ApplicationTopology {
  nodes: TopologyNode[];
  edges: TopologyEdge[];
  metrics: TopologyMetrics;
}

class TopologyBuilder {
  async buildApplicationTopology(
    accountId: string
  ): Promise<ApplicationTopology> {
    // Discover all entities
    const [applications, kafkaEntities] = await Promise.all([
      this.discoverApplications(accountId),
      this.discoverKafkaEntities(accountId)
    ]);
    
    // Build nodes
    const nodes = [
      ...applications.map(app => ({
        id: app.guid,
        type: 'APPLICATION',
        name: app.name,
        metadata: {
          language: app.language,
          framework: app.framework,
          health: app.health
        }
      })),
      ...kafkaEntities.map(entity => ({
        id: entity.guid,
        type: entity.type,
        name: entity.name,
        metadata: {
          provider: entity.provider,
          health: entity.health
        }
      }))
    ];
    
    // Discover edges (relationships)
    const edges = await this.discoverRelationships(nodes);
    
    // Calculate metrics
    const metrics = await this.calculateTopologyMetrics(nodes, edges);
    
    return { nodes, edges, metrics };
  }
  
  private async discoverRelationships(
    nodes: TopologyNode[]
  ): Promise<TopologyEdge[]> {
    const edges: TopologyEdge[] = [];
    
    // Query for producer/consumer relationships
    const relationshipQuery = `
      FROM KafkaProducerSample, KafkaConsumerSample
      SELECT 
        appName,
        kafka.topic,
        kafka.producer.client.id as producerId,
        kafka.consumer.group.id as consumerId
      WHERE appName IS NOT NULL
      SINCE 1 hour ago
    `;
    
    const results = await this.executeNRQL(relationshipQuery);
    
    results.forEach(result => {
      // Producer edge
      if (result.producerId) {
        const appNode = nodes.find(n => n.name === result.appName);
        const topicNode = nodes.find(n => 
          n.type === 'TOPIC' && n.name === result['kafka.topic']
        );
        
        if (appNode && topicNode) {
          edges.push({
            source: appNode.id,
            target: topicNode.id,
            type: 'PRODUCES_TO',
            metrics: {
              throughput: result.throughput,
              latency: result.latency
            }
          });
        }
      }
      
      // Consumer edge
      if (result.consumerId) {
        const topicNode = nodes.find(n => 
          n.type === 'TOPIC' && n.name === result['kafka.topic']
        );
        const appNode = nodes.find(n => n.name === result.appName);
        
        if (topicNode && appNode) {
          edges.push({
            source: topicNode.id,
            target: appNode.id,
            type: 'CONSUMED_BY',
            metrics: {
              throughput: result.throughput,
              lag: result.lag
            }
          });
        }
      }
    });
    
    return edges;
  }
}
```

### Topology Visualization Component

```typescript
const ApplicationTopologyView: React.FC<TopologyViewProps> = ({ 
  accountId,
  filters 
}) => {
  const [topology, setTopology] = useState<ApplicationTopology | null>(null);
  const [selectedNode, setSelectedNode] = useState<TopologyNode | null>(null);
  
  useEffect(() => {
    const builder = new TopologyBuilder();
    builder.buildApplicationTopology(accountId).then(setTopology);
  }, [accountId]);
  
  const renderTopology = () => {
    if (!topology) return <Spinner />;
    
    return (
      <ForceDirectedGraph
        nodes={topology.nodes}
        edges={topology.edges}
        onNodeClick={setSelectedNode}
        nodeRenderer={(node) => (
          <TopologyNode
            node={node}
            isSelected={selectedNode?.id === node.id}
            showMetrics={filters.showMetrics}
          />
        )}
        edgeRenderer={(edge) => (
          <TopologyEdge
            edge={edge}
            animated={edge.metrics.throughput > 0}
            color={getEdgeColor(edge.type)}
          />
        )}
      />
    );
  };
  
  return (
    <div className="topology-view">
      <TopologyControls
        onFilterChange={handleFilterChange}
        onLayoutChange={handleLayoutChange}
      />
      
      <div className="topology-canvas">
        {renderTopology()}
      </div>
      
      {selectedNode && (
        <NodeDetailsPanel
          node={selectedNode}
          onClose={() => setSelectedNode(null)}
        />
      )}
    </div>
  );
};
```

## APM Dashboards Integration

### Embedded Kafka Metrics

```typescript
class APMDashboardIntegration {
  // Add Kafka widgets to APM dashboard
  async enhanceAPMDashboard(
    dashboardGuid: string,
    applicationGuid: string
  ): Promise<void> {
    const kafkaWidgets = [
      {
        title: 'Kafka Producer Metrics',
        row: 1,
        column: 1,
        width: 6,
        height: 3,
        configuration: {
          nrql: `
            FROM Transaction
            SELECT 
              average(duration) as 'Avg Duration',
              percentile(duration, 95) as 'P95 Duration',
              rate(count(*), 1 minute) as 'Throughput'
            WHERE appGuid = '${applicationGuid}'
              AND name LIKE '%kafka%produce%'
            TIMESERIES
          `
        }
      },
      {
        title: 'Kafka Consumer Metrics',
        row: 1,
        column: 7,
        width: 6,
        height: 3,
        configuration: {
          nrql: `
            FROM Transaction
            SELECT 
              average(duration) as 'Avg Duration',
              max(kafka.consumer.lag) as 'Max Lag',
              uniqueCount(kafka.consumer.group) as 'Consumer Groups'
            WHERE appGuid = '${applicationGuid}'
              AND name LIKE '%kafka%consume%'
            TIMESERIES
          `
        }
      },
      {
        title: 'Kafka Topics Performance',
        row: 4,
        column: 1,
        width: 12,
        height: 4,
        configuration: {
          nrql: `
            FROM Transaction
            SELECT 
              count(*) as 'Messages',
              average(duration) as 'Avg Latency',
              percentage(count(*), WHERE error = true) as 'Error Rate'
            WHERE appGuid = '${applicationGuid}'
              AND kafka.topic IS NOT NULL
            FACET kafka.topic
            LIMIT 20
          `
        }
      }
    ];
    
    await this.addWidgetsToDashboard(dashboardGuid, kafkaWidgets);
  }
  
  // Create APM-Kafka correlation dashboard
  async createCorrelationDashboard(
    applicationGuid: string,
    kafkaClusterGuid: string
  ): Promise<string> {
    const dashboard = {
      name: 'APM-Kafka Correlation Dashboard',
      pages: [{
        name: 'Overview',
        widgets: [
          this.createApplicationOverviewWidget(applicationGuid),
          this.createKafkaOverviewWidget(kafkaClusterGuid),
          this.createCorrelationWidget(applicationGuid, kafkaClusterGuid),
          this.createTransactionFlowWidget(applicationGuid)
        ]
      }]
    };
    
    const result = await this.createDashboard(dashboard);
    return result.dashboardGuid;
  }
}
```

## Alerting Integration

### Correlated Alerts

```typescript
interface CorrelatedAlert {
  condition: AlertCondition;
  correlatedEntities: string[];
  impactAnalysis: ImpactAnalysis;
}

class APMKafkaAlerting {
  // Create correlated alert conditions
  async createCorrelatedAlerts(
    applicationGuid: string,
    kafkaEntities: string[]
  ): Promise<CorrelatedAlert[]> {
    const alerts: CorrelatedAlert[] = [];
    
    // High producer latency impacting application
    alerts.push({
      condition: {
        name: 'High Kafka Producer Latency',
        type: 'NRQL',
        nrql: `
          SELECT percentile(duration, 95)
          FROM Transaction
          WHERE appGuid = '${applicationGuid}'
            AND name LIKE '%kafka%produce%'
        `,
        threshold: {
          critical: 1000, // 1 second
          warning: 500    // 500ms
        }
      },
      correlatedEntities: kafkaEntities,
      impactAnalysis: {
        metric: 'application.responseTime',
        expectedImpact: 'INCREASE',
        severity: 'HIGH'
      }
    });
    
    // Consumer lag affecting processing
    alerts.push({
      condition: {
        name: 'High Consumer Lag',
        type: 'NRQL',
        nrql: `
          SELECT max(kafka.consumer.lag)
          FROM Transaction
          WHERE appGuid = '${applicationGuid}'
            AND kafka.consumer.lag IS NOT NULL
        `,
        threshold: {
          critical: 10000,
          warning: 5000
        }
      },
      correlatedEntities: kafkaEntities,
      impactAnalysis: {
        metric: 'application.processingDelay',
        expectedImpact: 'INCREASE',
        severity: 'CRITICAL'
      }
    });
    
    return alerts;
  }
  
  // Analyze alert impact across APM and Kafka
  async analyzeAlertImpact(
    alertIncident: AlertIncident
  ): Promise<AlertImpactAnalysis> {
    const affectedEntities = await this.findAffectedEntities(alertIncident);
    const downstreamImpact = await this.analyzeDownstreamImpact(affectedEntities);
    const rootCause = await this.performRootCauseAnalysis(alertIncident);
    
    return {
      incident: alertIncident,
      affectedApplications: affectedEntities.applications,
      affectedKafkaEntities: affectedEntities.kafkaEntities,
      downstreamImpact: downstreamImpact,
      rootCauseAnalysis: rootCause,
      recommendedActions: this.generateRecommendations(rootCause)
    };
  }
}
```

## Performance Optimization

### Query Optimization for APM-Kafka Correlation

```typescript
class APMKafkaQueryOptimizer {
  // Optimized query for transaction-infrastructure correlation
  buildOptimizedCorrelationQuery(
    applicationGuid: string,
    timeRange: TimeRange
  ): string {
    return `
      FROM Transaction, KafkaProducerSample, KafkaConsumerSample
      SELECT 
        -- Transaction metrics
        filter(average(duration), WHERE appGuid = '${applicationGuid}') as appDuration,
        filter(rate(count(*), 1 minute), WHERE appGuid = '${applicationGuid}') as appThroughput,
        
        -- Kafka producer metrics
        filter(average(producer.request.latency.avg), WHERE appName IS NOT NULL) as producerLatency,
        filter(sum(producer.byte.rate), WHERE appName IS NOT NULL) as producerByteRate,
        
        -- Kafka consumer metrics  
        filter(max(consumer.lag), WHERE appName IS NOT NULL) as consumerLag,
        filter(sum(consumer.byte.rate), WHERE appName IS NOT NULL) as consumerByteRate
        
      WHERE timestamp >= ${timeRange.start}
        AND timestamp <= ${timeRange.end}
      TIMESERIES 1 minute
    `;
  }
  
  // Cached correlation queries
  private correlationCache = new Map<string, CachedCorrelation>();
  
  async getCachedCorrelation(
    applicationGuid: string,
    kafkaEntityGuid: string
  ): Promise<CorrelationData> {
    const cacheKey = `${applicationGuid}:${kafkaEntityGuid}`;
    const cached = this.correlationCache.get(cacheKey);
    
    if (cached && !this.isExpired(cached)) {
      return cached.data;
    }
    
    const data = await this.calculateCorrelation(applicationGuid, kafkaEntityGuid);
    
    this.correlationCache.set(cacheKey, {
      data,
      timestamp: Date.now(),
      ttl: 300000 // 5 minutes
    });
    
    return data;
  }
}
```

## Best Practices

### 1. Instrumentation
- Ensure proper Kafka client instrumentation in applications
- Use distributed tracing headers for end-to-end visibility
- Instrument custom Kafka operations
- Monitor instrumentation overhead

### 2. Correlation
- Map application services to Kafka topics
- Track consumer group membership
- Monitor transaction flows through Kafka
- Correlate errors across boundaries

### 3. Performance
- Cache correlation data appropriately
- Use optimized queries for large datasets
- Implement sampling for high-volume applications
- Monitor query performance

### 4. Visualization
- Show clear relationships between apps and Kafka
- Highlight performance bottlenecks
- Provide drill-down capabilities
- Use consistent visual language

### 5. Alerting
- Create correlated alert conditions
- Consider cascade effects
- Set appropriate thresholds
- Include remediation guidance

## Conclusion

APM integration provides crucial visibility into how Kafka infrastructure impacts application performance. By implementing comprehensive correlation, visualization, and alerting capabilities, teams can quickly identify and resolve issues that span both application and infrastructure boundaries.