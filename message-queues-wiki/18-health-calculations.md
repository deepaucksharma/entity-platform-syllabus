# Health Calculations

## Overview

This document details the comprehensive health calculation methodologies used in the Message Queues monitoring system. It covers health scoring algorithms, threshold configurations, and interpretation guidelines for Kafka infrastructure components.

## Health Score Framework

### Health Score Architecture

```
Health Score System
├── Component Health Scores
│   ├── Cluster Health (0-100)
│   ├── Broker Health (0-100)
│   ├── Topic Health (0-100)
│   └── Consumer Health (0-100)
├── Aggregated Health Scores
│   ├── Service Health
│   ├── Regional Health
│   └── Global Health
└── Health Factors
    ├── Availability Factors
    ├── Performance Factors
    ├── Capacity Factors
    └── Reliability Factors
```

## Cluster Health Calculation

### Cluster Health Score Algorithm

```typescript
interface ClusterHealthFactors {
  availability: {
    activeController: boolean;
    offlinePartitions: number;
    totalPartitions: number;
  };
  reliability: {
    underReplicatedPartitions: number;
    inSyncReplicas: number;
    totalReplicas: number;
  };
  performance: {
    avgBrokerCpu: number;
    avgBrokerLoad: number;
    networkUtilization: number;
  };
  capacity: {
    diskUsagePercent: number;
    partitionUtilization: number;
    connectionUtilization: number;
  };
}

class ClusterHealthCalculator {
  calculateHealth(factors: ClusterHealthFactors): HealthScore {
    const weights = {
      availability: 0.40,  // 40% weight
      reliability: 0.30,   // 30% weight
      performance: 0.20,   // 20% weight
      capacity: 0.10       // 10% weight
    };
    
    // Calculate component scores
    const availabilityScore = this.calculateAvailabilityScore(factors.availability);
    const reliabilityScore = this.calculateReliabilityScore(factors.reliability);
    const performanceScore = this.calculatePerformanceScore(factors.performance);
    const capacityScore = this.calculateCapacityScore(factors.capacity);
    
    // Apply weights
    const weightedScore = 
      (availabilityScore * weights.availability) +
      (reliabilityScore * weights.reliability) +
      (performanceScore * weights.performance) +
      (capacityScore * weights.capacity);
    
    return {
      overall: Math.round(weightedScore),
      components: {
        availability: availabilityScore,
        reliability: reliabilityScore,
        performance: performanceScore,
        capacity: capacityScore
      },
      status: this.getHealthStatus(weightedScore),
      factors: this.identifyCriticalFactors(factors)
    };
  }
  
  private calculateAvailabilityScore(availability: any): number {
    let score = 100;
    
    // Active controller check (critical)
    if (!availability.activeController) {
      score = 0; // No controller = cluster unavailable
      return score;
    }
    
    // Offline partitions (critical impact)
    if (availability.offlinePartitions > 0) {
      const offlineRatio = availability.offlinePartitions / availability.totalPartitions;
      score -= Math.min(50, offlineRatio * 100);
    }
    
    return Math.max(0, score);
  }
  
  private calculateReliabilityScore(reliability: any): number {
    let score = 100;
    
    // Under-replicated partitions
    if (reliability.underReplicatedPartitions > 0) {
      const underReplicatedRatio = 
        reliability.underReplicatedPartitions / reliability.totalReplicas;
      
      // Progressive penalty
      if (underReplicatedRatio < 0.05) {
        score -= 10; // < 5% = minor issue
      } else if (underReplicatedRatio < 0.10) {
        score -= 25; // 5-10% = moderate issue
      } else if (underReplicatedRatio < 0.25) {
        score -= 50; // 10-25% = significant issue
      } else {
        score -= 75; // > 25% = critical issue
      }
    }
    
    // In-sync replica ratio
    const isrRatio = reliability.inSyncReplicas / reliability.totalReplicas;
    if (isrRatio < 0.95) {
      score -= (1 - isrRatio) * 20;
    }
    
    return Math.max(0, score);
  }
  
  private calculatePerformanceScore(performance: any): number {
    let score = 100;
    
    // CPU utilization
    if (performance.avgBrokerCpu > 90) {
      score -= 40; // Critical CPU
    } else if (performance.avgBrokerCpu > 75) {
      score -= 20; // High CPU
    } else if (performance.avgBrokerCpu > 60) {
      score -= 10; // Moderate CPU
    }
    
    // Load average
    if (performance.avgBrokerLoad > 2.0) {
      score -= 30; // Very high load
    } else if (performance.avgBrokerLoad > 1.5) {
      score -= 15; // High load
    } else if (performance.avgBrokerLoad > 1.0) {
      score -= 5;  // Moderate load
    }
    
    // Network utilization
    if (performance.networkUtilization > 0.9) {
      score -= 30; // Network saturated
    } else if (performance.networkUtilization > 0.75) {
      score -= 15; // High network usage
    }
    
    return Math.max(0, score);
  }
  
  private calculateCapacityScore(capacity: any): number {
    let score = 100;
    
    // Disk usage
    if (capacity.diskUsagePercent > 90) {
      score -= 50; // Critical disk usage
    } else if (capacity.diskUsagePercent > 80) {
      score -= 30; // High disk usage
    } else if (capacity.diskUsagePercent > 70) {
      score -= 10; // Moderate disk usage
    }
    
    // Partition utilization
    if (capacity.partitionUtilization > 0.9) {
      score -= 20; // Near partition limit
    } else if (capacity.partitionUtilization > 0.8) {
      score -= 10; // High partition usage
    }
    
    // Connection utilization
    if (capacity.connectionUtilization > 0.9) {
      score -= 20; // Near connection limit
    }
    
    return Math.max(0, score);
  }
  
  private getHealthStatus(score: number): HealthStatus {
    if (score >= 90) return 'excellent';
    if (score >= 80) return 'good';
    if (score >= 70) return 'fair';
    if (score >= 50) return 'poor';
    return 'critical';
  }
}
```

### NRQL Health Query

```sql
-- Comprehensive cluster health calculation
FROM AwsMskClusterSample, ConfluentCloudClusterSample
SELECT 
  -- Availability factors
  latest(provider.activeControllers) as activeControllers,
  latest(provider.offlinePartitions) as offlinePartitions,
  latest(provider.totalPartitions) as totalPartitions,
  
  -- Reliability factors
  latest(provider.underReplicatedPartitions) as underReplicated,
  latest(provider.inSyncReplicas) as inSyncReplicas,
  latest(provider.totalReplicas) as totalReplicas,
  
  -- Performance factors (from broker aggregation)
  filter(average(cpuUsage), WHERE entityType = 'BROKER') as avgBrokerCpu,
  filter(average(loadAverage), WHERE entityType = 'BROKER') as avgLoad,
  
  -- Capacity factors
  filter(average(diskUsagePercent), WHERE entityType = 'BROKER') as avgDiskUsage,
  
  -- Calculate health score
  100 
  - IF(activeControllers != 1, 100, 0)  -- No controller = 0 health
  - IF(offlinePartitions > 0, 50, 0)    -- Offline partitions critical
  - IF(underReplicated > totalPartitions * 0.1, 30, 
      IF(underReplicated > totalPartitions * 0.05, 15, 0))
  - IF(avgBrokerCpu > 90, 20, IF(avgBrokerCpu > 75, 10, 0))
  - IF(avgDiskUsage > 90, 10, IF(avgDiskUsage > 80, 5, 0))
  as healthScore
  
WHERE provider.clusterName IS NOT NULL
FACET provider.clusterName
```

## Broker Health Calculation

### Broker Health Components

```typescript
interface BrokerHealthMetrics {
  status: 'online' | 'offline' | 'degraded';
  cpu: {
    user: number;
    system: number;
    iowait: number;
  };
  memory: {
    heapUsed: number;
    heapMax: number;
    nonHeapUsed: number;
  };
  disk: {
    usedPercent: number;
    freeSpaceGB: number;
    throughputMBps: number;
  };
  network: {
    bytesInPerSec: number;
    bytesOutPerSec: number;
    errorRate: number;
  };
  replication: {
    isrShrinkRate: number;
    replicationLag: number;
    failedFetches: number;
  };
}

class BrokerHealthCalculator {
  calculateBrokerHealth(metrics: BrokerHealthMetrics): BrokerHealth {
    // Base score
    let healthScore = 100;
    const issues: HealthIssue[] = [];
    
    // Status check (binary)
    if (metrics.status === 'offline') {
      return {
        score: 0,
        status: 'critical',
        issues: [{ severity: 'critical', message: 'Broker is offline' }]
      };
    }
    
    if (metrics.status === 'degraded') {
      healthScore -= 30;
      issues.push({ severity: 'warning', message: 'Broker is in degraded state' });
    }
    
    // CPU health (20 points max deduction)
    const cpuTotal = metrics.cpu.user + metrics.cpu.system;
    if (cpuTotal > 95) {
      healthScore -= 20;
      issues.push({ severity: 'critical', message: `CPU at ${cpuTotal}%` });
    } else if (cpuTotal > 85) {
      healthScore -= 15;
      issues.push({ severity: 'warning', message: `High CPU usage: ${cpuTotal}%` });
    } else if (cpuTotal > 75) {
      healthScore -= 5;
    }
    
    // Memory health (15 points max deduction)
    const heapUsagePercent = (metrics.memory.heapUsed / metrics.memory.heapMax) * 100;
    if (heapUsagePercent > 95) {
      healthScore -= 15;
      issues.push({ 
        severity: 'critical', 
        message: `Heap memory critical: ${heapUsagePercent.toFixed(1)}%` 
      });
    } else if (heapUsagePercent > 85) {
      healthScore -= 10;
      issues.push({ 
        severity: 'warning', 
        message: `High heap usage: ${heapUsagePercent.toFixed(1)}%` 
      });
    }
    
    // Disk health (25 points max deduction)
    if (metrics.disk.usedPercent > 95) {
      healthScore -= 25;
      issues.push({ 
        severity: 'critical', 
        message: `Disk space critical: ${metrics.disk.usedPercent}%` 
      });
    } else if (metrics.disk.usedPercent > 85) {
      healthScore -= 15;
      issues.push({ 
        severity: 'warning', 
        message: `High disk usage: ${metrics.disk.usedPercent}%` 
      });
    } else if (metrics.disk.usedPercent > 75) {
      healthScore -= 5;
    }
    
    // Network health (10 points max deduction)
    if (metrics.network.errorRate > 0.01) { // > 1% error rate
      healthScore -= 10;
      issues.push({ 
        severity: 'warning', 
        message: `Network error rate: ${(metrics.network.errorRate * 100).toFixed(2)}%` 
      });
    }
    
    // Replication health (30 points max deduction)
    if (metrics.replication.isrShrinkRate > 0.1) {
      healthScore -= 20;
      issues.push({ 
        severity: 'critical', 
        message: 'High ISR shrink rate detected' 
      });
    }
    
    if (metrics.replication.replicationLag > 1000) {
      healthScore -= 10;
      issues.push({ 
        severity: 'warning', 
        message: `Replication lag: ${metrics.replication.replicationLag} messages` 
      });
    }
    
    return {
      score: Math.max(0, healthScore),
      status: this.getHealthStatus(healthScore),
      issues: issues,
      metrics: this.summarizeMetrics(metrics)
    };
  }
}
```

## Topic Health Calculation

### Topic Health Algorithm

```typescript
interface TopicHealthFactors {
  availability: {
    onlinePartitions: number;
    totalPartitions: number;
    minInSyncReplicas: number;
  };
  performance: {
    throughputBalance: number; // Standard deviation of partition throughput
    producerLatencyMs: number;
    consumerLagRatio: number;
  };
  configuration: {
    replicationFactor: number;
    minInSyncReplicas: number;
    retentionUtilization: number;
  };
}

class TopicHealthCalculator {
  calculateTopicHealth(factors: TopicHealthFactors): TopicHealth {
    let score = 100;
    const diagnostics: HealthDiagnostic[] = [];
    
    // Partition availability (40 points)
    const availabilityRatio = factors.availability.onlinePartitions / 
                             factors.availability.totalPartitions;
    if (availabilityRatio < 1.0) {
      const deduction = Math.min(40, (1 - availabilityRatio) * 100);
      score -= deduction;
      diagnostics.push({
        category: 'availability',
        issue: `${factors.availability.totalPartitions - factors.availability.onlinePartitions} offline partitions`,
        impact: deduction,
        recommendation: 'Check broker health and network connectivity'
      });
    }
    
    // Throughput balance (20 points)
    if (factors.performance.throughputBalance > 0.5) {
      const deduction = Math.min(20, factors.performance.throughputBalance * 20);
      score -= deduction;
      diagnostics.push({
        category: 'performance',
        issue: 'Unbalanced partition throughput',
        impact: deduction,
        recommendation: 'Consider partition reassignment'
      });
    }
    
    // Producer latency (15 points)
    if (factors.performance.producerLatencyMs > 1000) {
      score -= 15;
      diagnostics.push({
        category: 'performance',
        issue: `High producer latency: ${factors.performance.producerLatencyMs}ms`,
        impact: 15,
        recommendation: 'Check broker load and network latency'
      });
    } else if (factors.performance.producerLatencyMs > 500) {
      score -= 10;
    } else if (factors.performance.producerLatencyMs > 100) {
      score -= 5;
    }
    
    // Consumer lag (15 points)
    if (factors.performance.consumerLagRatio > 0.1) {
      const deduction = Math.min(15, factors.performance.consumerLagRatio * 50);
      score -= deduction;
      diagnostics.push({
        category: 'performance',
        issue: `Consumer lag at ${(factors.performance.consumerLagRatio * 100).toFixed(1)}% of topic size`,
        impact: deduction,
        recommendation: 'Scale consumer groups or optimize processing'
      });
    }
    
    // Configuration health (10 points)
    if (factors.configuration.replicationFactor < 3) {
      score -= 5;
      diagnostics.push({
        category: 'configuration',
        issue: `Low replication factor: ${factors.configuration.replicationFactor}`,
        impact: 5,
        recommendation: 'Increase replication factor for better durability'
      });
    }
    
    if (factors.configuration.retentionUtilization > 0.9) {
      score -= 5;
      diagnostics.push({
        category: 'configuration',
        issue: 'Approaching retention limits',
        impact: 5,
        recommendation: 'Review retention policy or add storage'
      });
    }
    
    return {
      score: Math.max(0, score),
      status: this.getHealthStatus(score),
      diagnostics: diagnostics,
      trends: this.calculateHealthTrends(factors)
    };
  }
}
```

## Consumer Group Health

### Consumer Health Metrics

```typescript
interface ConsumerGroupHealth {
  lag: {
    current: number;
    trend: 'increasing' | 'stable' | 'decreasing';
    velocity: number; // messages/minute
  };
  throughput: {
    currentRate: number;
    expectedRate: number;
    efficiency: number;
  };
  stability: {
    rebalanceFrequency: number;
    lastRebalance: Date;
    memberStability: number;
  };
  errors: {
    errorRate: number;
    lastError: Date;
    errorTypes: Map<string, number>;
  };
}

class ConsumerHealthCalculator {
  calculateConsumerHealth(metrics: ConsumerGroupHealth): HealthScore {
    const weights = {
      lag: 0.35,
      throughput: 0.30,
      stability: 0.20,
      errors: 0.15
    };
    
    const lagScore = this.calculateLagScore(metrics.lag);
    const throughputScore = this.calculateThroughputScore(metrics.throughput);
    const stabilityScore = this.calculateStabilityScore(metrics.stability);
    const errorScore = this.calculateErrorScore(metrics.errors);
    
    const overallScore = 
      (lagScore * weights.lag) +
      (throughputScore * weights.throughput) +
      (stabilityScore * weights.stability) +
      (errorScore * weights.errors);
    
    return {
      score: Math.round(overallScore),
      components: {
        lag: lagScore,
        throughput: throughputScore,
        stability: stabilityScore,
        errors: errorScore
      },
      status: this.getHealthStatus(overallScore),
      alerts: this.generateAlerts(metrics)
    };
  }
  
  private calculateLagScore(lag: ConsumerGroupHealth['lag']): number {
    let score = 100;
    
    // Current lag impact
    if (lag.current > 1000000) {
      score = 0; // Critical lag
    } else if (lag.current > 100000) {
      score -= 50;
    } else if (lag.current > 10000) {
      score -= 30;
    } else if (lag.current > 1000) {
      score -= 10;
    }
    
    // Trend impact
    if (lag.trend === 'increasing' && lag.velocity > 100) {
      score = Math.max(0, score - 20);
    }
    
    return score;
  }
  
  private calculateThroughputScore(throughput: ConsumerGroupHealth['throughput']): number {
    // Efficiency-based scoring
    const efficiency = throughput.efficiency;
    
    if (efficiency >= 0.95) return 100;
    if (efficiency >= 0.90) return 90;
    if (efficiency >= 0.80) return 75;
    if (efficiency >= 0.70) return 60;
    if (efficiency >= 0.50) return 40;
    return 20;
  }
}
```

## Composite Health Calculations

### Service-Level Health

```typescript
class ServiceHealthCalculator {
  calculateServiceHealth(
    clusters: ClusterHealth[],
    dependencies: ServiceDependency[]
  ): ServiceHealth {
    // Weighted average based on cluster importance
    const clusterWeights = this.calculateClusterWeights(clusters, dependencies);
    
    let weightedScore = 0;
    let totalWeight = 0;
    
    clusters.forEach((cluster, index) => {
      const weight = clusterWeights[index];
      weightedScore += cluster.score * weight;
      totalWeight += weight;
    });
    
    const baseScore = weightedScore / totalWeight;
    
    // Apply dependency penalties
    const dependencyPenalty = this.calculateDependencyPenalty(dependencies);
    const finalScore = Math.max(0, baseScore - dependencyPenalty);
    
    return {
      score: Math.round(finalScore),
      clusterScores: clusters.map(c => ({ 
        name: c.name, 
        score: c.score,
        weight: clusterWeights[clusters.indexOf(c)]
      })),
      dependencies: dependencies,
      recommendations: this.generateServiceRecommendations(clusters, dependencies)
    };
  }
  
  private calculateClusterWeights(
    clusters: ClusterHealth[],
    dependencies: ServiceDependency[]
  ): number[] {
    // Base weights on traffic and criticality
    return clusters.map(cluster => {
      let weight = 1.0;
      
      // Traffic volume factor
      const trafficRatio = cluster.throughput / this.getTotalThroughput(clusters);
      weight *= (0.5 + trafficRatio * 0.5);
      
      // Criticality factor
      const criticalDeps = dependencies.filter(d => 
        d.source === cluster.id && d.criticality === 'critical'
      ).length;
      weight *= (1 + criticalDeps * 0.2);
      
      return weight;
    });
  }
}
```

## Health Trend Analysis

### Trend Calculation

```typescript
interface HealthTrend {
  current: number;
  previous: number;
  trend: 'improving' | 'stable' | 'degrading';
  velocity: number; // Points per hour
  projection: {
    oneHour: number;
    oneDay: number;
    oneWeek: number;
  };
}

class HealthTrendAnalyzer {
  analyzeHealthTrend(
    historicalScores: TimeSeries<number>,
    currentScore: number
  ): HealthTrend {
    const recentScores = this.getRecentScores(historicalScores, '1 hour');
    const previousScore = recentScores[recentScores.length - 2] || currentScore;
    
    // Calculate velocity (rate of change)
    const velocity = this.calculateVelocity(recentScores);
    
    // Determine trend
    let trend: 'improving' | 'stable' | 'degrading';
    if (Math.abs(velocity) < 0.5) {
      trend = 'stable';
    } else if (velocity > 0) {
      trend = 'improving';
    } else {
      trend = 'degrading';
    }
    
    // Project future scores
    const projection = {
      oneHour: Math.min(100, Math.max(0, currentScore + velocity)),
      oneDay: Math.min(100, Math.max(0, currentScore + velocity * 24)),
      oneWeek: Math.min(100, Math.max(0, currentScore + velocity * 168))
    };
    
    return {
      current: currentScore,
      previous: previousScore,
      trend,
      velocity,
      projection
    };
  }
  
  // NRQL for trend analysis
  getTrendQuery(entityType: string, entityId: string): string {
    return `
      SELECT 
        average(healthScore) as score,
        stddev(healthScore) as volatility,
        derivative(average(healthScore), 1 hour) as velocity
      FROM ${entityType}Sample
      WHERE entityId = '${entityId}'
      SINCE 24 hours ago
      TIMESERIES 1 hour
    `;
  }
}
```

## Health Alerting Thresholds

### Dynamic Threshold Configuration

```typescript
interface HealthThresholds {
  static: {
    critical: number;
    warning: number;
    info: number;
  };
  dynamic: {
    enabled: boolean;
    baseline: number;
    standardDeviations: {
      warning: number;
      critical: number;
    };
  };
  timeBasedOverrides: Array<{
    schedule: CronExpression;
    thresholds: Partial<HealthThresholds['static']>;
  }>;
}

const defaultHealthThresholds: HealthThresholds = {
  static: {
    critical: 50,
    warning: 70,
    info: 80
  },
  dynamic: {
    enabled: true,
    baseline: 85,
    standardDeviations: {
      warning: 2,
      critical: 3
    }
  },
  timeBasedOverrides: [
    {
      schedule: '0 2-4 * * *', // 2-4 AM maintenance window
      thresholds: {
        critical: 30,
        warning: 50
      }
    }
  ]
};
```

## Health Visualization

### Health Status Mapping

```typescript
const healthStatusConfig = {
  excellent: {
    range: [90, 100],
    color: '#11a968',
    icon: 'check-circle',
    label: 'Excellent'
  },
  good: {
    range: [80, 90],
    color: '#5aa15a',
    icon: 'check',
    label: 'Good'
  },
  fair: {
    range: [70, 80],
    color: '#f5a623',
    icon: 'alert-triangle',
    label: 'Fair'
  },
  poor: {
    range: [50, 70],
    color: '#ff7f0e',
    icon: 'alert-circle',
    label: 'Poor'
  },
  critical: {
    range: [0, 50],
    color: '#d0021b',
    icon: 'x-circle',
    label: 'Critical'
  }
};
```

## Best Practices

### 1. Health Score Design
- Use weighted scoring for different factors
- Apply progressive penalties for critical issues
- Consider dependencies and relationships
- Include both current state and trends

### 2. Threshold Management
- Set realistic thresholds based on baselines
- Use dynamic thresholds for variable workloads
- Implement time-based overrides
- Regular threshold review and adjustment

### 3. Health Monitoring
- Track health score trends over time
- Alert on rapid health degradation
- Correlate health with business metrics
- Regular health report generation

### 4. Troubleshooting
- Provide detailed health diagnostics
- Include actionable recommendations
- Track health improvement actions
- Measure remediation effectiveness

## Conclusion

The health calculation system provides a comprehensive framework for assessing and monitoring Kafka infrastructure health. By combining multiple factors and using intelligent scoring algorithms, teams can proactively identify and address issues before they impact service availability.