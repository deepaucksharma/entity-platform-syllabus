# Alert Integration

## Overview

The Message Queues monitoring system provides comprehensive alerting capabilities that integrate with New Relic's alerting platform. This enables proactive monitoring of Kafka infrastructure with intelligent alert conditions, correlation, and automated response capabilities.

## Alert Architecture

### Alert System Components

```
┌─────────────────────────────────────────────────────────┐
│                  Alert Definition Layer                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │  Condition   │  │  Threshold  │  │   Policy    │   │
│  │  Templates   │  │   Manager   │  │   Engine    │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
├─────────────────────────────────────────────────────────┤
│                 Alert Evaluation Layer                  │
│  ┌──────────────────────────────────────────────────┐  │
│  │  • Real-time Evaluation                          │  │
│  │  • Anomaly Detection                             │  │
│  │  • Correlation Analysis                          │  │
│  │  • Suppression Logic                             │  │
│  └──────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                 Alert Response Layer                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │
│  │   Incident   │  │ Notification│  │  Automation │   │
│  │   Manager    │  │   Router    │  │   Engine    │   │
│  └─────────────┘  └─────────────┘  └─────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Alert Condition Templates

### Kafka-Specific Alert Conditions

```typescript
interface KafkaAlertTemplate {
  id: string;
  name: string;
  description: string;
  category: 'cluster' | 'broker' | 'topic' | 'consumer';
  severity: 'critical' | 'high' | 'medium' | 'low';
  condition: AlertCondition;
  recommendations: string[];
}

class KafkaAlertTemplates {
  private templates: Map<string, KafkaAlertTemplate> = new Map([
    // Cluster Health Alerts
    ['cluster_offline_partitions', {
      id: 'cluster_offline_partitions',
      name: 'Offline Partitions Detected',
      description: 'One or more partitions have no active leader',
      category: 'cluster',
      severity: 'critical',
      condition: {
        type: 'NRQL',
        query: `
          SELECT latest(provider.offlinePartitions)
          FROM AwsMskClusterSample, ConfluentCloudClusterSample
          WHERE provider.offlinePartitions > 0
        `,
        threshold: {
          critical: { value: 1, duration: 60 },
          warning: null
        },
        signalLostDuration: 300
      },
      recommendations: [
        'Check broker health and availability',
        'Review recent configuration changes',
        'Verify network connectivity between brokers',
        'Check disk space on affected brokers'
      ]
    }],
    
    // Broker Performance Alerts
    ['broker_high_cpu', {
      id: 'broker_high_cpu',
      name: 'High Broker CPU Usage',
      description: 'Broker CPU usage exceeds threshold',
      category: 'broker',
      severity: 'high',
      condition: {
        type: 'NRQL',
        query: `
          SELECT average(provider.cpuUser + provider.cpuSystem)
          FROM AwsMskBrokerSample
          FACET provider.brokerId, provider.clusterName
        `,
        threshold: {
          critical: { value: 90, duration: 300 },
          warning: { value: 75, duration: 300 }
        }
      },
      recommendations: [
        'Scale up broker instance type',
        'Redistribute partitions across brokers',
        'Optimize client configurations',
        'Review producer/consumer patterns'
      ]
    }],
    
    // Topic Performance Alerts
    ['topic_high_throughput', {
      id: 'topic_high_throughput',
      name: 'Topic Throughput Threshold Exceeded',
      description: 'Topic receiving data above configured threshold',
      category: 'topic',
      severity: 'medium',
      condition: {
        type: 'NRQL',
        query: `
          SELECT rate(sum(provider.bytesInPerSec.Average), 1 minute)
          FROM AwsMskTopicSample
          FACET provider.topic
        `,
        threshold: {
          critical: { value: 100000000, duration: 180 }, // 100MB/s
          warning: { value: 50000000, duration: 180 }    // 50MB/s
        }
      },
      recommendations: [
        'Increase partition count for better distribution',
        'Review producer batching configuration',
        'Consider implementing rate limiting',
        'Evaluate retention policies'
      ]
    }],
    
    // Consumer Lag Alerts
    ['consumer_lag_critical', {
      id: 'consumer_lag_critical',
      name: 'Critical Consumer Lag',
      description: 'Consumer group falling behind significantly',
      category: 'consumer',
      severity: 'critical',
      condition: {
        type: 'NRQL',
        query: `
          SELECT max(consumerLag)
          FROM KafkaConsumerSample
          FACET consumerGroup, topic
        `,
        threshold: {
          critical: { value: 100000, duration: 300 },
          warning: { value: 10000, duration: 300 }
        }
      },
      recommendations: [
        'Scale out consumer instances',
        'Optimize consumer processing logic',
        'Check for consumer errors or restarts',
        'Review partition assignment strategy'
      ]
    }]
  ]);
  
  getTemplate(id: string): KafkaAlertTemplate | undefined {
    return this.templates.get(id);
  }
  
  getTemplatesByCategory(category: string): KafkaAlertTemplate[] {
    return Array.from(this.templates.values())
      .filter(template => template.category === category);
  }
  
  getAllTemplates(): KafkaAlertTemplate[] {
    return Array.from(this.templates.values());
  }
}
```

### Dynamic Alert Conditions

```typescript
class DynamicAlertBuilder {
  // Build alert condition based on entity characteristics
  buildDynamicCondition(entity: KafkaEntity): AlertCondition {
    switch (entity.type) {
      case 'CLUSTER':
        return this.buildClusterCondition(entity);
      case 'BROKER':
        return this.buildBrokerCondition(entity);
      case 'TOPIC':
        return this.buildTopicCondition(entity);
      default:
        throw new Error(`Unsupported entity type: ${entity.type}`);
    }
  }
  
  private buildClusterCondition(cluster: ClusterEntity): AlertCondition {
    // Dynamic thresholds based on cluster size
    const brokerCount = cluster.metadata.brokerCount || 3;
    const partitionCount = cluster.metadata.totalPartitions || 100;
    
    return {
      name: `${cluster.name} - Cluster Health`,
      type: 'NRQL',
      query: `
        SELECT 
          latest(provider.offlinePartitions) as offlinePartitions,
          latest(provider.underReplicatedPartitions) as underReplicated,
          latest(provider.activeControllers) as activeControllers
        FROM AwsMskClusterSample
        WHERE provider.clusterName = '${cluster.name}'
      `,
      threshold: {
        critical: {
          offlinePartitions: { value: 1, duration: 60 },
          underReplicated: { value: partitionCount * 0.1, duration: 300 },
          activeControllers: { value: 0, duration: 60 }
        },
        warning: {
          underReplicated: { value: partitionCount * 0.05, duration: 300 }
        }
      },
      evaluationDelay: 60,
      signalLostDuration: 300
    };
  }
  
  private buildTopicCondition(topic: TopicEntity): AlertCondition {
    // Baseline from historical data
    const baseline = this.calculateBaseline(topic);
    
    return {
      name: `${topic.name} - Anomaly Detection`,
      type: 'NRQL',
      query: `
        SELECT 
          average(provider.bytesInPerSec.Average) as bytesIn,
          average(provider.bytesOutPerSec.Average) as bytesOut,
          average(provider.messagesInPerSec.Average) as messagesIn
        FROM AwsMskTopicSample
        WHERE provider.topic = '${topic.name}'
      `,
      threshold: {
        critical: {
          bytesIn: { 
            value: baseline.bytesIn * 3, // 3x baseline
            duration: 300 
          }
        },
        warning: {
          bytesIn: { 
            value: baseline.bytesIn * 2, // 2x baseline
            duration: 300 
          }
        }
      },
      anomalyDetection: {
        enabled: true,
        sensitivity: 3,
        direction: 'BOTH'
      }
    };
  }
}
```

## Alert Correlation

### Multi-Entity Correlation

```typescript
interface CorrelatedAlert {
  primaryEntity: string;
  relatedEntities: string[];
  correlationType: 'causal' | 'temporal' | 'spatial';
  confidence: number;
  evidence: CorrelationEvidence[];
}

class AlertCorrelationEngine {
  // Correlate alerts across Kafka infrastructure
  async correlateAlerts(
    incident: AlertIncident
  ): Promise<CorrelatedAlert[]> {
    const correlations: CorrelatedAlert[] = [];
    
    // Temporal correlation
    const temporalCorrelations = await this.findTemporalCorrelations(incident);
    correlations.push(...temporalCorrelations);
    
    // Causal correlation
    const causalCorrelations = await this.findCausalCorrelations(incident);
    correlations.push(...causalCorrelations);
    
    // Spatial correlation (same cluster/topic)
    const spatialCorrelations = await this.findSpatialCorrelations(incident);
    correlations.push(...spatialCorrelations);
    
    return this.rankCorrelations(correlations);
  }
  
  private async findCausalCorrelations(
    incident: AlertIncident
  ): Promise<CorrelatedAlert[]> {
    const correlations: CorrelatedAlert[] = [];
    
    // Get entity relationships
    const relationships = await this.getEntityRelationships(incident.entityGuid);
    
    // Check for alerts on related entities
    for (const relationship of relationships) {
      const relatedAlerts = await this.getActiveAlerts(relationship.targetGuid);
      
      if (relatedAlerts.length > 0) {
        // Calculate correlation confidence based on relationship type
        const confidence = this.calculateCausalConfidence(
          incident,
          relatedAlerts,
          relationship
        );
        
        if (confidence > 0.7) {
          correlations.push({
            primaryEntity: incident.entityGuid,
            relatedEntities: relatedAlerts.map(a => a.entityGuid),
            correlationType: 'causal',
            confidence,
            evidence: [
              {
                type: 'relationship',
                description: `${relationship.type} relationship exists`,
                weight: 0.8
              },
              {
                type: 'timing',
                description: 'Related alerts occurred within expected timeframe',
                weight: 0.2
              }
            ]
          });
        }
      }
    }
    
    return correlations;
  }
  
  // Alert pattern recognition
  async detectAlertPatterns(
    timeRange: TimeRange
  ): Promise<AlertPattern[]> {
    const alerts = await this.getHistoricalAlerts(timeRange);
    const patterns: AlertPattern[] = [];
    
    // Recurring patterns
    const recurringPatterns = this.findRecurringPatterns(alerts);
    patterns.push(...recurringPatterns);
    
    // Cascade patterns
    const cascadePatterns = this.findCascadePatterns(alerts);
    patterns.push(...cascadePatterns);
    
    // Cyclical patterns
    const cyclicalPatterns = this.findCyclicalPatterns(alerts);
    patterns.push(...cyclicalPatterns);
    
    return patterns;
  }
}
```

### Root Cause Analysis

```typescript
class RootCauseAnalyzer {
  // Analyze root cause of Kafka incidents
  async analyzeRootCause(
    incident: AlertIncident
  ): Promise<RootCauseAnalysis> {
    // Collect evidence
    const evidence = await this.collectEvidence(incident);
    
    // Build causal graph
    const causalGraph = this.buildCausalGraph(evidence);
    
    // Identify root causes
    const rootCauses = this.identifyRootCauses(causalGraph);
    
    // Generate remediation steps
    const remediation = this.generateRemediation(rootCauses);
    
    return {
      incident,
      rootCauses,
      confidenceScore: this.calculateConfidence(rootCauses),
      evidence,
      causalGraph,
      remediation,
      preventionRecommendations: this.generatePreventionSteps(rootCauses)
    };
  }
  
  private async collectEvidence(
    incident: AlertIncident
  ): Promise<Evidence[]> {
    const evidence: Evidence[] = [];
    
    // Metric anomalies
    const anomalies = await this.detectMetricAnomalies(
      incident.entityGuid,
      incident.timestamp
    );
    evidence.push(...anomalies.map(a => ({
      type: 'metric_anomaly',
      description: `${a.metric} showed ${a.deviation}σ deviation`,
      timestamp: a.timestamp,
      relevance: a.significance
    })));
    
    // Configuration changes
    const changes = await this.getConfigurationChanges(
      incident.entityGuid,
      incident.timestamp - 3600000 // 1 hour before
    );
    evidence.push(...changes.map(c => ({
      type: 'config_change',
      description: `Configuration ${c.key} changed from ${c.oldValue} to ${c.newValue}`,
      timestamp: c.timestamp,
      relevance: 0.8
    })));
    
    // Error logs
    const errors = await this.getErrorLogs(
      incident.entityGuid,
      incident.timestamp
    );
    evidence.push(...errors.map(e => ({
      type: 'error_log',
      description: e.message,
      timestamp: e.timestamp,
      relevance: this.calculateErrorRelevance(e, incident)
    })));
    
    return evidence;
  }
  
  private buildCausalGraph(evidence: Evidence[]): CausalGraph {
    const graph = new CausalGraph();
    
    // Add nodes for each evidence
    evidence.forEach(e => {
      graph.addNode({
        id: e.id,
        type: e.type,
        description: e.description,
        timestamp: e.timestamp
      });
    });
    
    // Establish causal relationships
    evidence.forEach(e1 => {
      evidence.forEach(e2 => {
        if (e1.id !== e2.id) {
          const causality = this.calculateCausality(e1, e2);
          if (causality > 0.5) {
            graph.addEdge(e1.id, e2.id, causality);
          }
        }
      });
    });
    
    return graph;
  }
}
```

## Alert Automation

### Automated Response Actions

```typescript
interface AutomatedAction {
  id: string;
  name: string;
  description: string;
  trigger: ActionTrigger;
  actions: Action[];
  validation: ValidationStep[];
  rollback?: RollbackPlan;
}

class AlertAutomation {
  private actions = new Map<string, AutomatedAction>([
    // Auto-scaling for high load
    ['auto_scale_consumers', {
      id: 'auto_scale_consumers',
      name: 'Auto-scale Consumer Group',
      description: 'Automatically scale consumer instances when lag exceeds threshold',
      trigger: {
        condition: 'consumer_lag_critical',
        threshold: { lag: 50000, duration: 300 }
      },
      actions: [
        {
          type: 'scale_out',
          target: 'consumer_group',
          parameters: {
            scaleUpBy: 2,
            maxInstances: 10
          }
        },
        {
          type: 'notification',
          target: 'ops_team',
          parameters: {
            message: 'Auto-scaled consumer group {{consumerGroup}} by {{scaleUpBy}} instances'
          }
        }
      ],
      validation: [
        {
          type: 'metric_check',
          metric: 'consumerLag',
          expected: 'decreasing',
          timeout: 600
        }
      ],
      rollback: {
        trigger: { lag: 1000, duration: 1800 },
        actions: [{ type: 'scale_in', parameters: { scaleDownBy: 1 } }]
      }
    }],
    
    // Partition rebalancing
    ['rebalance_partitions', {
      id: 'rebalance_partitions',
      name: 'Rebalance Topic Partitions',
      description: 'Redistribute partitions when broker load is uneven',
      trigger: {
        condition: 'broker_load_imbalance',
        threshold: { imbalanceRatio: 0.3, duration: 600 }
      },
      actions: [
        {
          type: 'kafka_admin',
          operation: 'reassign_partitions',
          parameters: {
            strategy: 'even_distribution',
            throttle: 10000000 // 10MB/s
          }
        }
      ],
      validation: [
        {
          type: 'cluster_health',
          checks: ['no_offline_partitions', 'all_replicas_in_sync'],
          timeout: 1200
        }
      ]
    }]
  ]);
  
  async executeAutomation(
    incident: AlertIncident
  ): Promise<AutomationResult> {
    // Find matching automation
    const automation = this.findMatchingAutomation(incident);
    if (!automation) {
      return { executed: false, reason: 'No matching automation found' };
    }
    
    // Validate execution conditions
    const validation = await this.validateExecution(automation, incident);
    if (!validation.canExecute) {
      return { executed: false, reason: validation.reason };
    }
    
    // Execute actions
    const results = [];
    for (const action of automation.actions) {
      try {
        const result = await this.executeAction(action, incident);
        results.push(result);
      } catch (error) {
        // Rollback on failure
        await this.rollbackActions(results, automation);
        throw error;
      }
    }
    
    // Validate results
    const validationResult = await this.validateResults(
      automation.validation,
      results
    );
    
    return {
      executed: true,
      automation: automation.id,
      actions: results,
      validation: validationResult,
      timestamp: Date.now()
    };
  }
}
```

### Runbook Integration

```typescript
interface Runbook {
  id: string;
  title: string;
  description: string;
  alertConditions: string[];
  steps: RunbookStep[];
  automation: AutomationConfig;
}

class RunbookManager {
  // Execute runbook for incident
  async executeRunbook(
    incident: AlertIncident,
    runbookId: string
  ): Promise<RunbookExecution> {
    const runbook = await this.getRunbook(runbookId);
    const execution = {
      id: generateId(),
      runbookId,
      incidentId: incident.id,
      startTime: Date.now(),
      steps: [],
      status: 'running'
    };
    
    for (const step of runbook.steps) {
      const stepResult = await this.executeStep(step, incident, execution);
      execution.steps.push(stepResult);
      
      if (stepResult.status === 'failed' && step.onFailure === 'abort') {
        execution.status = 'failed';
        break;
      }
    }
    
    execution.endTime = Date.now();
    execution.status = execution.status === 'running' ? 'completed' : execution.status;
    
    return execution;
  }
  
  // Kafka-specific runbooks
  private kafkaRunbooks: Runbook[] = [
    {
      id: 'kafka_broker_recovery',
      title: 'Kafka Broker Recovery',
      description: 'Steps to recover an unhealthy Kafka broker',
      alertConditions: ['broker_down', 'broker_high_load'],
      steps: [
        {
          name: 'Diagnose broker health',
          type: 'diagnostic',
          commands: [
            'kafka-broker-api-shell.sh --describe --broker-id {{brokerId}}',
            'kafka-log-dirs.sh --describe --broker-list {{brokerId}}'
          ]
        },
        {
          name: 'Check disk usage',
          type: 'metric_check',
          query: `
            SELECT latest(diskUsedPercent)
            FROM SystemSample
            WHERE hostname = '{{brokerHost}}'
          `,
          threshold: { critical: 90, warning: 80 }
        },
        {
          name: 'Restart broker if needed',
          type: 'remediation',
          condition: 'broker_not_responding',
          action: {
            type: 'shell',
            command: 'systemctl restart kafka',
            requireApproval: true
          }
        }
      ],
      automation: {
        enabled: false,
        approvalRequired: true
      }
    }
  ];
}
```

## Alert Visualization

### Alert Dashboard

```typescript
const AlertDashboard: React.FC<AlertDashboardProps> = ({ accountId }) => {
  const [activeIncidents, setActiveIncidents] = useState<AlertIncident[]>([]);
  const [alertHistory, setAlertHistory] = useState<AlertHistoryData>([]);
  const [correlations, setCorrelations] = useState<CorrelatedAlert[]>([]);
  
  return (
    <div className="alert-dashboard">
      {/* Active Incidents */}
      <section className="active-incidents">
        <h2>Active Incidents ({activeIncidents.length})</h2>
        <IncidentList
          incidents={activeIncidents}
          onIncidentClick={handleIncidentClick}
          showCorrelations={true}
        />
      </section>
      
      {/* Alert Heatmap */}
      <section className="alert-heatmap">
        <h2>Alert Patterns - Last 7 Days</h2>
        <AlertHeatmap
          data={alertHistory}
          groupBy="entity"
          timeGranularity="hour"
          colorScale={['green', 'yellow', 'orange', 'red']}
        />
      </section>
      
      {/* Correlation Graph */}
      <section className="correlation-graph">
        <h2>Alert Correlations</h2>
        <CorrelationGraph
          correlations={correlations}
          layout="force-directed"
          onNodeClick={handleEntityClick}
        />
      </section>
      
      {/* Alert Timeline */}
      <section className="alert-timeline">
        <h2>Incident Timeline</h2>
        <Timeline
          events={buildTimelineEvents(activeIncidents, correlations)}
          onEventClick={handleTimelineEventClick}
        />
      </section>
    </div>
  );
};
```

### Alert Analytics

```typescript
class AlertAnalytics {
  // Alert frequency analysis
  async analyzeAlertFrequency(
    entityGuid: string,
    timeRange: TimeRange
  ): Promise<FrequencyAnalysis> {
    const query = `
      FROM AlertIncident
      SELECT 
        count(*) as incidentCount,
        uniqueCount(conditionName) as uniqueConditions,
        average(duration) as avgDuration,
        percentage(count(*), WHERE priority = 'CRITICAL') as criticalPercentage
      WHERE entityGuid = '${entityGuid}'
        AND openTime >= ${timeRange.start}
        AND openTime <= ${timeRange.end}
      FACET conditionName
      TIMESERIES 1 day
    `;
    
    const results = await this.executeQuery(query);
    
    return {
      totalIncidents: results.totalCount,
      averagePerDay: results.totalCount / this.getDays(timeRange),
      topConditions: results.facets.slice(0, 10),
      trend: this.calculateTrend(results.timeseries),
      recommendations: this.generateFrequencyRecommendations(results)
    };
  }
  
  // Alert noise reduction
  async identifyNoisyAlerts(
    accountId: string
  ): Promise<NoisyAlert[]> {
    const query = `
      FROM AlertIncident
      SELECT 
        count(*) as count,
        average(duration) as avgDuration,
        uniqueCount(entityGuid) as affectedEntities
      WHERE accountId = ${accountId}
        AND closeTime IS NOT NULL
        AND duration < 300000  -- Less than 5 minutes
      FACET conditionName
      SINCE 7 days ago
      LIMIT 20
    `;
    
    const results = await this.executeQuery(query);
    
    return results.facets
      .filter(f => f.count > 10 && f.avgDuration < 180000)
      .map(f => ({
        conditionName: f.conditionName,
        incidentCount: f.count,
        avgDuration: f.avgDuration,
        noiseScore: this.calculateNoiseScore(f),
        recommendation: 'Consider increasing threshold or duration'
      }));
  }
}
```

## Alert Configuration Management

### Alert Policy Templates

```yaml
# Kafka Alert Policy Template
name: "Kafka Infrastructure Alerts - {{environment}}"
incident_preference: PER_CONDITION_AND_TARGET
conditions:
  - name: "Cluster Health"
    enabled: true
    templates:
      - cluster_offline_partitions
      - cluster_controller_failure
      - cluster_under_replicated_partitions
    
  - name: "Broker Performance"
    enabled: true
    templates:
      - broker_high_cpu
      - broker_disk_usage
      - broker_network_errors
    
  - name: "Topic Monitoring"
    enabled: true
    templates:
      - topic_high_throughput
      - topic_replication_lag
      - topic_partition_skew
    
  - name: "Consumer Monitoring"
    enabled: true
    templates:
      - consumer_lag_critical
      - consumer_rebalancing
      - consumer_errors

notification_channels:
  - type: email
    configuration:
      recipients: ["kafka-team@company.com"]
      include_graphs: true
  
  - type: slack
    configuration:
      channel: "#kafka-alerts"
      mention_on_critical: "@kafka-oncall"
  
  - type: pagerduty
    configuration:
      service_key: "{{pagerduty_service_key}}"
      severity_mapping:
        critical: "critical"
        high: "error"
        medium: "warning"
        low: "info"
```

### Dynamic Threshold Management

```typescript
class DynamicThresholdManager {
  // Calculate dynamic thresholds based on historical data
  async calculateDynamicThresholds(
    entityGuid: string,
    metric: string
  ): Promise<DynamicThreshold> {
    // Get historical data
    const historicalData = await this.getHistoricalMetricData(
      entityGuid,
      metric,
      { days: 30 }
    );
    
    // Calculate statistical baselines
    const stats = {
      mean: this.calculateMean(historicalData),
      stdDev: this.calculateStdDev(historicalData),
      percentiles: this.calculatePercentiles(historicalData, [50, 75, 90, 95, 99])
    };
    
    // Apply time-based patterns
    const patterns = this.detectPatterns(historicalData);
    
    // Generate thresholds
    return {
      baseline: stats.mean,
      thresholds: {
        critical: this.calculateThreshold(stats, patterns, 'critical'),
        warning: this.calculateThreshold(stats, patterns, 'warning')
      },
      schedule: patterns.timeBasedSchedule,
      confidence: this.calculateConfidence(historicalData.length, stats.stdDev)
    };
  }
  
  // Update thresholds based on feedback
  async updateThresholdsFromFeedback(
    conditionId: string,
    feedback: ThresholdFeedback[]
  ): Promise<void> {
    const currentThreshold = await this.getThreshold(conditionId);
    const adjustment = this.calculateAdjustment(feedback);
    
    const newThreshold = {
      ...currentThreshold,
      critical: currentThreshold.critical * (1 + adjustment.critical),
      warning: currentThreshold.warning * (1 + adjustment.warning)
    };
    
    await this.updateThreshold(conditionId, newThreshold);
    
    // Log adjustment for ML training
    await this.logThresholdAdjustment({
      conditionId,
      oldThreshold: currentThreshold,
      newThreshold,
      feedback,
      timestamp: Date.now()
    });
  }
}
```

## Best Practices

### 1. Alert Design
- Create meaningful alert conditions with clear titles
- Set appropriate thresholds based on baselines
- Include context in alert descriptions
- Group related alerts into policies

### 2. Threshold Management
- Use dynamic thresholds for variable workloads
- Implement time-based thresholds for predictable patterns
- Regular review and adjustment of thresholds
- Consider seasonal variations

### 3. Alert Correlation
- Configure entity relationships properly
- Use tags for grouping related alerts
- Implement suppression rules for cascading failures
- Leverage correlation for root cause analysis

### 4. Automation
- Start with semi-automated responses
- Implement proper validation and rollback
- Log all automated actions
- Regular review of automation effectiveness

### 5. Noise Reduction
- Identify and tune noisy alerts
- Use aggregation windows appropriately
- Implement alert suppression during maintenance
- Regular alert effectiveness reviews

## Conclusion

The alert integration system provides comprehensive monitoring and incident management capabilities for Kafka infrastructure. By leveraging intelligent alerting, correlation, and automation, teams can proactively manage their Kafka deployments and quickly resolve issues when they arise.