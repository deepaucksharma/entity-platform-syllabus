# Extension Guide

## Overview

This guide provides comprehensive instructions for extending the Message Queues monitoring system with new features, providers, visualizations, and integrations. It covers the extension points, development patterns, and best practices for creating custom extensions.

## Extension Architecture

### Extension Points

```
Extension System
├── Provider Extensions
│   ├── Custom Kafka Providers
│   ├── Metric Collectors
│   └── Data Transformers
├── Visualization Extensions
│   ├── Custom Chart Types
│   ├── Dashboard Widgets
│   └── Interactive Components
├── Integration Extensions
│   ├── External Systems
│   ├── Notification Channels
│   └── Data Exporters
└── Feature Extensions
    ├── Custom Health Calculations
    ├── New Filtering Options
    └── Additional Analytics
```

## Provider Extensions

### Adding a New Kafka Provider

```typescript
// src/providers/custom-kafka-provider.ts
import { KafkaProvider, ProviderConfig, KafkaEntity } from '../types';

export interface CustomKafkaProviderConfig extends ProviderConfig {
  endpoint: string;
  authMethod: 'oauth' | 'mtls' | 'custom';
  customOptions?: Record<string, any>;
}

export class CustomKafkaProvider implements KafkaProvider {
  readonly type = 'CUSTOM_KAFKA';
  readonly displayName = 'Custom Kafka Provider';
  readonly icon = 'custom-kafka-icon.svg';
  
  constructor(private config: CustomKafkaProviderConfig) {
    this.validateConfig();
  }
  
  async discoverEntities(): Promise<KafkaEntity[]> {
    const entities: KafkaEntity[] = [];
    
    // Discover clusters
    const clusters = await this.discoverClusters();
    entities.push(...clusters);
    
    // Discover brokers for each cluster
    for (const cluster of clusters) {
      const brokers = await this.discoverBrokers(cluster.id);
      entities.push(...brokers);
      
      // Discover topics
      const topics = await this.discoverTopics(cluster.id);
      entities.push(...topics);
    }
    
    return entities;
  }
  
  async collectMetrics(entityId: string): Promise<Metrics> {
    const entity = await this.getEntity(entityId);
    
    switch (entity.type) {
      case 'CLUSTER':
        return this.collectClusterMetrics(entityId);
      case 'BROKER':
        return this.collectBrokerMetrics(entityId);
      case 'TOPIC':
        return this.collectTopicMetrics(entityId);
      default:
        throw new Error(`Unknown entity type: ${entity.type}`);
    }
  }
  
  private async collectClusterMetrics(clusterId: string): Promise<ClusterMetrics> {
    // Custom API calls to collect metrics
    const response = await this.apiClient.get(`/clusters/${clusterId}/metrics`);
    
    // Transform to standard format
    return {
      timestamp: Date.now(),
      clusterId,
      metrics: {
        brokerCount: response.brokers.length,
        topicCount: response.topics.length,
        partitionCount: response.partitions.total,
        offlinePartitions: response.partitions.offline,
        underReplicatedPartitions: response.partitions.underReplicated,
        throughput: {
          bytesInPerSec: response.throughput.in,
          bytesOutPerSec: response.throughput.out
        }
      }
    };
  }
  
  // Entity synthesis mapping
  getEntitySynthesis(): EntitySynthesisRule[] {
    return [
      {
        entityType: 'CUSTOM_KAFKA_CLUSTER',
        synthesis: {
          name: 'customKafkaCluster',
          identifier: (data) => `${data.provider}:${data.clusterId}`,
          tags: {
            provider: this.type,
            region: (data) => data.region,
            environment: (data) => data.tags?.environment
          },
          goldenMetrics: [
            'provider.brokerCount',
            'provider.topicCount',
            'provider.offlinePartitions',
            'provider.throughput'
          ]
        }
      }
    ];
  }
}

// Register the provider
export function registerCustomProvider(): void {
  ProviderRegistry.register({
    type: 'CUSTOM_KAFKA',
    factory: (config) => new CustomKafkaProvider(config),
    configSchema: customProviderConfigSchema
  });
}
```

### Provider Integration Configuration

```typescript
// src/config/provider-registry.ts
interface ProviderRegistration {
  type: string;
  factory: (config: any) => KafkaProvider;
  configSchema: JSONSchema;
  capabilities?: ProviderCapabilities;
}

class ProviderRegistry {
  private static providers = new Map<string, ProviderRegistration>();
  
  static register(registration: ProviderRegistration): void {
    if (this.providers.has(registration.type)) {
      throw new Error(`Provider ${registration.type} already registered`);
    }
    
    this.providers.set(registration.type, registration);
    
    // Register with New Relic entity synthesis
    this.registerEntitySynthesis(registration);
    
    // Register metric mappings
    this.registerMetricMappings(registration);
  }
  
  static getProvider(type: string, config: any): KafkaProvider {
    const registration = this.providers.get(type);
    if (!registration) {
      throw new Error(`Unknown provider type: ${type}`);
    }
    
    // Validate config against schema
    this.validateConfig(config, registration.configSchema);
    
    return registration.factory(config);
  }
}
```

## Visualization Extensions

### Creating Custom Visualizations

```typescript
// src/visualizations/custom-topology-map/index.tsx
import React from 'react';
import { Visualization, VisualizationProps } from 'nr1';

interface TopologyMapConfig {
  layout: 'force' | 'hierarchical' | 'circular';
  nodeSize: 'uniform' | 'throughput' | 'connections';
  edgeWeight: 'uniform' | 'traffic' | 'latency';
  animation: boolean;
}

export class TopologyMapVisualization extends React.Component<
  VisualizationProps<TopologyMapConfig>
> {
  static propTypes = {
    layout: PropTypes.oneOf(['force', 'hierarchical', 'circular']),
    nodeSize: PropTypes.oneOf(['uniform', 'throughput', 'connections']),
    edgeWeight: PropTypes.oneOf(['uniform', 'traffic', 'latency']),
    animation: PropTypes.bool
  };
  
  static defaultProps = {
    layout: 'force',
    nodeSize: 'throughput',
    edgeWeight: 'traffic',
    animation: true
  };
  
  render() {
    const { data, config } = this.props;
    
    return (
      <div className="topology-map-container">
        <TopologyCanvas
          nodes={this.transformNodes(data)}
          edges={this.transformEdges(data)}
          layout={config.layout}
          onNodeClick={this.handleNodeClick}
          onEdgeClick={this.handleEdgeClick}
        />
        <TopologyLegend />
        <TopologyControls
          config={config}
          onChange={this.handleConfigChange}
        />
      </div>
    );
  }
  
  private transformNodes(data: any): TopologyNode[] {
    return data.facets.map(facet => ({
      id: facet.name,
      type: this.getNodeType(facet),
      size: this.calculateNodeSize(facet, this.props.config.nodeSize),
      metrics: {
        throughput: facet.results[0]?.throughput || 0,
        health: facet.results[0]?.healthScore || 100
      },
      position: this.getInitialPosition(facet)
    }));
  }
  
  private transformEdges(data: any): TopologyEdge[] {
    // Build edges from relationships
    const edges: TopologyEdge[] = [];
    
    data.relationships?.forEach(rel => {
      edges.push({
        source: rel.sourceId,
        target: rel.targetId,
        weight: this.calculateEdgeWeight(rel, this.props.config.edgeWeight),
        metrics: {
          traffic: rel.traffic,
          latency: rel.latency
        }
      });
    });
    
    return edges;
  }
}

// Visualization configuration
export const visualizationConfig = {
  id: 'topology-map',
  displayName: 'Kafka Topology Map',
  description: 'Interactive topology visualization for Kafka infrastructure',
  category: 'kafka',
  examples: [
    {
      name: 'Basic Topology',
      config: {
        layout: 'force',
        nodeSize: 'uniform'
      },
      nrql: `
        FROM KafkaClusterSample, KafkaBrokerSample
        SELECT uniqueCount(entityGuid), average(throughput)
        FACET clusterName, brokerName
        SINCE 1 hour ago
      `
    }
  ]
};
```

### Custom Chart Component

```typescript
// src/components/charts/KafkaFlowChart.tsx
import React, { useEffect, useRef } from 'react';
import * as d3 from 'd3';
import { useNerdletStateContext } from 'nr1';

interface KafkaFlowChartProps {
  data: FlowData[];
  height?: number;
  onNodeSelect?: (nodeId: string) => void;
}

export const KafkaFlowChart: React.FC<KafkaFlowChartProps> = ({
  data,
  height = 400,
  onNodeSelect
}) => {
  const svgRef = useRef<SVGSVGElement>(null);
  const { timeRange } = useNerdletStateContext();
  
  useEffect(() => {
    if (!svgRef.current || !data.length) return;
    
    const svg = d3.select(svgRef.current);
    const width = svgRef.current.clientWidth;
    
    // Clear previous chart
    svg.selectAll('*').remove();
    
    // Create flow layout
    const sankey = d3.sankey()
      .nodeWidth(15)
      .nodePadding(10)
      .extent([[1, 1], [width - 1, height - 6]]);
    
    // Prepare data
    const { nodes, links } = sankey({
      nodes: data.map(d => ({ name: d.name, ...d })),
      links: data.flatMap(d => 
        d.targets.map(t => ({
          source: d.name,
          target: t.name,
          value: t.value
        }))
      )
    });
    
    // Draw links
    const link = svg.append('g')
      .selectAll('.link')
      .data(links)
      .enter().append('path')
      .attr('class', 'link')
      .attr('d', d3.sankeyLinkHorizontal())
      .style('stroke-width', d => Math.max(1, d.width))
      .style('stroke', d => this.getLinkColor(d))
      .style('fill', 'none')
      .style('opacity', 0.5);
    
    // Draw nodes
    const node = svg.append('g')
      .selectAll('.node')
      .data(nodes)
      .enter().append('g')
      .attr('class', 'node')
      .attr('transform', d => `translate(${d.x0},${d.y0})`)
      .on('click', (event, d) => onNodeSelect?.(d.name));
    
    // Node rectangles
    node.append('rect')
      .attr('height', d => d.y1 - d.y0)
      .attr('width', sankey.nodeWidth())
      .style('fill', d => this.getNodeColor(d))
      .style('stroke', '#000');
    
    // Node labels
    node.append('text')
      .attr('x', -6)
      .attr('y', d => (d.y1 - d.y0) / 2)
      .attr('dy', '.35em')
      .attr('text-anchor', 'end')
      .text(d => d.name)
      .filter(d => d.x0 < width / 2)
      .attr('x', 6 + sankey.nodeWidth())
      .attr('text-anchor', 'start');
    
    // Add tooltips
    this.addTooltips(node, link);
    
  }, [data, height, timeRange]);
  
  return (
    <div className="kafka-flow-chart">
      <svg ref={svgRef} width="100%" height={height} />
    </div>
  );
};
```

## Integration Extensions

### External System Integration

```typescript
// src/integrations/external-monitoring/index.ts
interface ExternalIntegration {
  name: string;
  type: 'push' | 'pull';
  config: Record<string, any>;
}

export class PrometheusIntegration implements ExternalIntegration {
  name = 'Prometheus';
  type = 'push' as const;
  
  constructor(private config: PrometheusConfig) {
    this.validateConfig();
  }
  
  async exportMetrics(metrics: KafkaMetrics[]): Promise<void> {
    const promMetrics = this.transformToPrometheus(metrics);
    
    await fetch(`${this.config.pushGateway}/metrics/job/kafka-monitoring`, {
      method: 'POST',
      body: promMetrics,
      headers: {
        'Content-Type': 'text/plain'
      }
    });
  }
  
  private transformToPrometheus(metrics: KafkaMetrics[]): string {
    const lines: string[] = [];
    
    metrics.forEach(metric => {
      // Add metric metadata
      lines.push(`# HELP kafka_${metric.name} ${metric.description}`);
      lines.push(`# TYPE kafka_${metric.name} ${metric.type}`);
      
      // Add metric values
      metric.values.forEach(value => {
        const labels = this.formatLabels(value.labels);
        lines.push(`kafka_${metric.name}${labels} ${value.value} ${value.timestamp}`);
      });
    });
    
    return lines.join('\n');
  }
  
  private formatLabels(labels: Record<string, string>): string {
    if (!Object.keys(labels).length) return '';
    
    const pairs = Object.entries(labels)
      .map(([key, value]) => `${key}="${value}"`)
      .join(',');
    
    return `{${pairs}}`;
  }
}
```

### Custom Notification Channel

```typescript
// src/notifications/custom-channel.ts
import { NotificationChannel, Alert } from '../types';

export class SlackNotificationChannel implements NotificationChannel {
  constructor(private config: SlackConfig) {}
  
  async sendAlert(alert: Alert): Promise<void> {
    const message = this.formatAlert(alert);
    
    await fetch(this.config.webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message)
    });
  }
  
  private formatAlert(alert: Alert): SlackMessage {
    const color = this.getAlertColor(alert.severity);
    
    return {
      attachments: [{
        color,
        title: `${alert.severity.toUpperCase()}: ${alert.title}`,
        text: alert.description,
        fields: [
          {
            title: 'Cluster',
            value: alert.cluster,
            short: true
          },
          {
            title: 'Time',
            value: new Date(alert.timestamp).toISOString(),
            short: true
          },
          {
            title: 'Current Value',
            value: alert.currentValue.toString(),
            short: true
          },
          {
            title: 'Threshold',
            value: alert.threshold.toString(),
            short: true
          }
        ],
        actions: [
          {
            type: 'button',
            text: 'View in New Relic',
            url: alert.link
          },
          {
            type: 'button',
            text: 'Acknowledge',
            url: `${this.config.ackUrl}?alertId=${alert.id}`
          }
        ]
      }]
    };
  }
}

// Register notification channel
NotificationRegistry.register({
  type: 'slack',
  displayName: 'Slack',
  factory: (config) => new SlackNotificationChannel(config),
  configSchema: {
    type: 'object',
    properties: {
      webhookUrl: { type: 'string', format: 'uri' },
      channel: { type: 'string' },
      username: { type: 'string' }
    },
    required: ['webhookUrl']
  }
});
```

## Feature Extensions

### Custom Health Calculation

```typescript
// src/health/custom-health-calculator.ts
import { HealthCalculator, HealthScore } from '../types';

export class MLBasedHealthCalculator implements HealthCalculator {
  private model: TensorFlowModel;
  
  constructor() {
    this.model = this.loadModel();
  }
  
  async calculateHealth(entity: KafkaEntity, metrics: Metrics): Promise<HealthScore> {
    // Prepare features for ML model
    const features = this.extractFeatures(entity, metrics);
    
    // Run inference
    const prediction = await this.model.predict(features);
    
    // Calculate health score
    const score = prediction.healthScore * 100;
    const anomalies = this.detectAnomalies(features, prediction);
    
    return {
      score: Math.round(score),
      status: this.getStatus(score),
      factors: {
        predicted: prediction.factors,
        anomalies: anomalies
      },
      confidence: prediction.confidence,
      recommendations: this.generateRecommendations(prediction)
    };
  }
  
  private extractFeatures(entity: KafkaEntity, metrics: Metrics): Features {
    return {
      // Time-based features
      hourOfDay: new Date().getHours(),
      dayOfWeek: new Date().getDay(),
      
      // Metric features
      cpuUsage: metrics.cpu.usage,
      memoryUsage: metrics.memory.usage,
      diskUsage: metrics.disk.usage,
      networkThroughput: metrics.network.throughput,
      
      // Kafka-specific features
      partitionCount: entity.partitions?.length || 0,
      replicationFactor: entity.replicationFactor || 1,
      consumerLag: metrics.consumerLag || 0,
      
      // Historical features
      avgCpuLast24h: metrics.historical.cpu.avg,
      maxCpuLast24h: metrics.historical.cpu.max,
      throughputTrend: metrics.historical.throughput.trend
    };
  }
  
  private detectAnomalies(features: Features, prediction: Prediction): Anomaly[] {
    const anomalies: Anomaly[] = [];
    
    // Check each feature against predicted normal range
    Object.entries(features).forEach(([feature, value]) => {
      const expected = prediction.expectedRanges[feature];
      if (expected && (value < expected.min || value > expected.max)) {
        anomalies.push({
          feature,
          value,
          expected,
          severity: this.calculateAnomalySeverity(value, expected)
        });
      }
    });
    
    return anomalies;
  }
}

// Register custom health calculator
HealthCalculatorRegistry.register({
  name: 'ml-based',
  displayName: 'Machine Learning Based Health',
  calculator: new MLBasedHealthCalculator(),
  description: 'Uses machine learning to predict health scores and detect anomalies'
});
```

### Advanced Filtering Extension

```typescript
// src/filters/advanced-filters.ts
interface AdvancedFilter {
  id: string;
  name: string;
  type: 'custom';
  evaluate: (entity: any, context: FilterContext) => boolean;
  config?: React.ComponentType<FilterConfigProps>;
}

export const geoProximityFilter: AdvancedFilter = {
  id: 'geo-proximity',
  name: 'Geographic Proximity',
  type: 'custom',
  evaluate: (entity, context) => {
    if (!context.config.centerPoint || !entity.location) {
      return true;
    }
    
    const distance = calculateDistance(
      context.config.centerPoint,
      entity.location
    );
    
    return distance <= context.config.radius;
  },
  config: GeoProximityFilterConfig
};

export const anomalyDetectionFilter: AdvancedFilter = {
  id: 'anomaly-detection',
  name: 'Anomaly Detection',
  type: 'custom',
  evaluate: async (entity, context) => {
    const anomalyScore = await detectAnomalies(entity, context.timeRange);
    
    return anomalyScore >= context.config.threshold;
  },
  config: AnomalyDetectionFilterConfig
};

// Register filters
FilterRegistry.registerBatch([
  geoProximityFilter,
  anomalyDetectionFilter,
  performanceBaselineFilter,
  customTagFilter
]);
```

## Extension Development Workflow

### 1. Development Setup

```bash
# Create extension project
mkdir message-queues-extension
cd message-queues-extension

# Initialize extension
nr1 create --type extension --name my-kafka-extension

# Install dependencies
npm install

# Link to main project
npm link ../message-queues
```

### 2. Extension Structure

```
my-kafka-extension/
├── src/
│   ├── index.ts           # Extension entry point
│   ├── providers/         # Custom providers
│   ├── visualizations/    # Custom visualizations
│   ├── integrations/      # External integrations
│   └── components/        # Shared components
├── config/
│   ├── manifest.json      # Extension manifest
│   └── schema.json        # Configuration schema
├── tests/
│   └── extension.test.ts
└── package.json
```

### 3. Extension Manifest

```json
{
  "name": "my-kafka-extension",
  "version": "1.0.0",
  "description": "Custom Kafka monitoring extensions",
  "type": "extension",
  "compatibility": {
    "messageQueues": ">=2.0.0",
    "nr1": ">=1.50.0"
  },
  "provides": {
    "providers": ["custom-kafka"],
    "visualizations": ["topology-map", "flow-chart"],
    "integrations": ["prometheus", "slack"],
    "filters": ["geo-proximity", "anomaly-detection"]
  },
  "configuration": {
    "schema": "./config/schema.json"
  },
  "permissions": [
    "external-api:write",
    "entity:create"
  ]
}
```

### 4. Testing Extensions

```typescript
// tests/extension.test.ts
import { ExtensionTester } from '@newrelic/extension-test-framework';
import { MyKafkaExtension } from '../src';

describe('My Kafka Extension', () => {
  let tester: ExtensionTester;
  
  beforeEach(() => {
    tester = new ExtensionTester({
      extension: MyKafkaExtension,
      mockData: {
        entities: mockKafkaEntities,
        metrics: mockKafkaMetrics
      }
    });
  });
  
  test('provider discovers entities correctly', async () => {
    const provider = await tester.getProvider('custom-kafka');
    const entities = await provider.discoverEntities();
    
    expect(entities).toHaveLength(10);
    expect(entities[0]).toMatchObject({
      type: 'CUSTOM_KAFKA_CLUSTER',
      name: expect.any(String)
    });
  });
  
  test('visualization renders without errors', async () => {
    const viz = await tester.renderVisualization('topology-map', {
      data: mockTopologyData,
      config: { layout: 'force' }
    });
    
    expect(viz).toMatchSnapshot();
    expect(viz.find('.topology-node')).toHaveLength(5);
  });
});
```

## Best Practices

### 1. Extension Design
- Follow single responsibility principle
- Use TypeScript for type safety
- Implement proper error handling
- Provide comprehensive documentation

### 2. Performance
- Lazy load heavy dependencies
- Implement efficient data transformations
- Use React.memo for expensive components
- Cache API responses appropriately

### 3. Compatibility
- Test with multiple versions
- Handle missing features gracefully
- Provide fallback implementations
- Version your APIs

### 4. Security
- Validate all inputs
- Sanitize external data
- Use secure communication
- Follow principle of least privilege

## Publishing Extensions

### 1. Prepare for Publishing

```bash
# Run tests
npm test

# Build extension
npm run build

# Validate extension
nr1 extension:validate

# Create package
nr1 extension:pack
```

### 2. Publish to Catalog

```bash
# Publish to New Relic catalog
nr1 catalog:submit

# Or publish to npm
npm publish
```

### 3. Documentation

Create comprehensive documentation including:
- Installation instructions
- Configuration guide
- API reference
- Usage examples
- Troubleshooting guide

## Conclusion

The extension system provides powerful capabilities for customizing and extending the Message Queues monitoring system. By following the patterns and best practices outlined in this guide, developers can create robust extensions that enhance the monitoring capabilities for their specific needs.