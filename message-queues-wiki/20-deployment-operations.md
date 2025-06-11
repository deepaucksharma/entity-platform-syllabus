# Deployment and Operations

## Overview

This document provides comprehensive guidelines for deploying and operating the Message Queues monitoring system in production environments. It covers deployment strategies, operational procedures, monitoring, and maintenance best practices.

## Deployment Architecture

### Deployment Models

```
Deployment Options
├── New Relic Hosted
│   ├── Standard Deployment
│   ├── EU Region Deployment
│   └── FedRAMP Deployment
├── Hybrid Deployment
│   ├── Data Collection On-Premise
│   ├── Processing in New Relic
│   └── Secure Gateway
└── Multi-Region Deployment
    ├── Region-Specific Instances
    ├── Global Aggregation
    └── Cross-Region Replication
```

## Pre-Deployment Requirements

### Infrastructure Requirements

```yaml
# Minimum infrastructure requirements
infrastructure:
  new_relic:
    accounts:
      - type: primary
        features:
          - full_stack_observability
          - dashboards
          - alerts
          - nerdpacks
    
  kafka_infrastructure:
    aws_msk:
      minimum_version: 2.8.0
      monitoring:
        - cloudwatch_enabled: true
        - enhanced_monitoring: PER_BROKER
        - open_monitoring:
            prometheus:
              jmx_exporter: true
              node_exporter: true
    
    confluent_cloud:
      api_access:
        - metrics_api: v2
        - admin_api: v3
      
  network:
    connectivity:
      - new_relic_endpoints:
          - api.newrelic.com
          - nrql.newrelic.com
          - infrastructure-api.newrelic.com
    
    bandwidth:
      minimum: 10_mbps
      recommended: 100_mbps
    
    firewall_rules:
      outbound:
        - protocol: https
          port: 443
          destinations:
            - "*.newrelic.com"
            - "*.amazonaws.com"
            - "*.confluent.cloud"
```

### Access Requirements

```typescript
interface AccessRequirements {
  aws: {
    permissions: [
      "kafka:ListClusters",
      "kafka:DescribeCluster",
      "kafka:ListNodes",
      "kafka:DescribeConfiguration",
      "cloudwatch:GetMetricData",
      "cloudwatch:ListMetrics",
      "tag:GetResources",
      "iam:PassRole"
    ];
    roles: {
      monitoring: {
        trust_policy: {
          principal: "arn:aws:iam::754728514883:root", // New Relic account
          conditions: {
            external_id: "YOUR_EXTERNAL_ID"
          }
        };
      };
    };
  };
  
  confluent: {
    api_keys: {
      cloud_api: {
        permissions: ["MetricsViewer", "CloudClusterViewer"];
      };
      metrics_api: {
        permissions: ["MetricsReader"];
      };
    };
  };
  
  new_relic: {
    user_permissions: [
      "nerdpack:write",
      "dashboard:write",
      "alert:write",
      "entity:read"
    ];
    api_keys: {
      user_key: {
        type: "USER",
        permissions: ["graphql:query", "nrql:query"]
      };
      ingest_key: {
        type: "INGEST",
        permissions: ["metric:write"]
      };
    };
  };
}
```

## Deployment Process

### 1. Nerdpack Deployment

```bash
# Clone repository
git clone https://github.com/newrelic/message-queues.git
cd message-queues

# Install dependencies
npm install

# Configure New Relic CLI
nr1 profiles:add --name production --api-key YOUR_USER_KEY --region US

# Validate nerdpack
nr1 nerdpack:validate

# Deploy to New Relic
nr1 nerdpack:publish
nr1 nerdpack:deploy --channel STABLE
nr1 nerdpack:subscribe --channel STABLE

# Tag release
nr1 nerdpack:tag --tag v1.0.0
```

### 2. Integration Configuration

```typescript
class IntegrationDeployment {
  async deployAWSIntegration(config: AWSIntegrationConfig): Promise<void> {
    // 1. Create IAM role via CloudFormation
    const cfnTemplate = this.generateCloudFormationTemplate(config);
    await this.deployCloudFormation(cfnTemplate);
    
    // 2. Configure New Relic integration
    const integration = {
      name: `Kafka-Monitoring-${config.environment}`,
      type: 'AWS_MSK',
      linkedAccount: {
        id: config.awsAccountId,
        externalId: config.externalId,
        roleArn: `arn:aws:iam::${config.awsAccountId}:role/NewRelicKafkaRole`
      },
      settings: {
        metricsPollingInterval: 300,
        enabledRegions: config.regions,
        tagFilters: config.tagFilters
      }
    };
    
    await this.newRelicAPI.createIntegration(integration);
    
    // 3. Validate integration
    await this.validateIntegration(integration);
  }
  
  async deployConfluentIntegration(config: ConfluentConfig): Promise<void> {
    // Store credentials securely
    await this.secureStore.save({
      service: 'CONFLUENT_CLOUD',
      credentials: {
        apiKey: config.apiKey,
        apiSecret: config.apiSecret
      }
    });
    
    // Configure metric collection
    const collector = {
      name: `Confluent-Collector-${config.environment}`,
      type: 'CONFLUENT_CLOUD',
      schedule: '*/5 * * * *', // Every 5 minutes
      config: {
        environments: config.environments,
        clusters: config.clusters,
        metrics: [
          'io.confluent.kafka.server.*',
          'io.confluent.kafka.topic.*',
          'io.confluent.kafka.consumer.*'
        ]
      }
    };
    
    await this.deployMetricCollector(collector);
  }
}
```

### 3. Dashboard Deployment

```typescript
class DashboardDeployment {
  async deployDashboards(environment: string): Promise<void> {
    const dashboards = [
      'kafka-executive-overview',
      'kafka-cluster-health',
      'kafka-performance-analytics',
      'kafka-consumer-lag-analysis',
      'kafka-troubleshooting'
    ];
    
    for (const dashboardName of dashboards) {
      const template = await this.loadDashboardTemplate(dashboardName);
      const customized = this.customizeDashboard(template, environment);
      
      const result = await this.newRelicAPI.createDashboard({
        name: `${dashboardName}-${environment}`,
        permissions: 'PUBLIC_READ_WRITE',
        pages: customized.pages,
        variables: customized.variables
      });
      
      console.log(`Deployed dashboard: ${result.guid}`);
    }
  }
  
  private customizeDashboard(template: any, environment: string): any {
    // Replace variables with environment-specific values
    return {
      ...template,
      variables: template.variables.map(v => {
        if (v.name === 'environment') {
          return { ...v, defaultValue: environment };
        }
        return v;
      })
    };
  }
}
```

### 4. Alert Configuration

```typescript
class AlertDeployment {
  async deployAlertPolicies(config: AlertConfig): Promise<void> {
    // Create alert policies
    const policies = [
      {
        name: `Kafka Infrastructure - ${config.environment}`,
        incident_preference: 'PER_CONDITION_AND_TARGET',
        channels: config.notificationChannels
      }
    ];
    
    for (const policy of policies) {
      const policyResult = await this.createAlertPolicy(policy);
      
      // Add conditions
      const conditions = this.getAlertConditions(config.environment);
      for (const condition of conditions) {
        await this.createAlertCondition(policyResult.id, condition);
      }
    }
    
    // Set up notification channels
    await this.configureNotificationChannels(config.notifications);
  }
  
  private getAlertConditions(environment: string): AlertCondition[] {
    const baseConditions = [
      {
        name: 'Offline Partitions',
        type: 'NRQL',
        nrql: {
          query: `SELECT latest(offlinePartitions) FROM KafkaClusterSample WHERE environment = '${environment}' AND offlinePartitions > 0`
        },
        critical: { threshold: 1, duration: 60 }
      },
      {
        name: 'High Consumer Lag',
        type: 'NRQL',
        nrql: {
          query: `SELECT max(consumerLag) FROM KafkaConsumerSample WHERE environment = '${environment}' FACET consumerGroup`
        },
        critical: { threshold: 100000, duration: 300 },
        warning: { threshold: 10000, duration: 300 }
      }
    ];
    
    return baseConditions;
  }
}
```

## Operational Procedures

### Health Checks

```typescript
class HealthCheckService {
  async performHealthCheck(): Promise<HealthCheckResult> {
    const checks = [
      this.checkNewRelicConnectivity(),
      this.checkKafkaIntegrations(),
      this.checkDataIngestion(),
      this.checkDashboardHealth(),
      this.checkAlertingPipeline()
    ];
    
    const results = await Promise.all(checks);
    
    return {
      timestamp: Date.now(),
      overall: results.every(r => r.status === 'healthy') ? 'healthy' : 'unhealthy',
      components: results,
      recommendations: this.generateRecommendations(results)
    };
  }
  
  private async checkDataIngestion(): Promise<ComponentHealth> {
    const query = `
      SELECT count(*)
      FROM KafkaClusterSample
      SINCE 10 minutes ago
    `;
    
    const result = await this.nrqlQuery(query);
    const recordCount = result.results[0].count;
    
    return {
      component: 'data_ingestion',
      status: recordCount > 0 ? 'healthy' : 'unhealthy',
      metrics: {
        recordsIngested: recordCount,
        dataFreshness: this.calculateDataFreshness()
      },
      message: recordCount > 0 
        ? `Ingesting ${recordCount} records in last 10 minutes`
        : 'No data received in last 10 minutes'
    };
  }
}
```

### Monitoring Operations

```bash
#!/bin/bash
# Operational monitoring script

# Check integration status
check_integration_status() {
  echo "Checking integration status..."
  
  # AWS MSK integration
  aws_status=$(nr1 api graphql -q '
    {
      actor {
        account(id: ACCOUNT_ID) {
          cloud {
            linkedAccounts {
              name
              integrations {
                name
                service
                lastReportingChangeAt
              }
            }
          }
        }
      }
    }
  ')
  
  echo "AWS Integration Status:"
  echo "$aws_status" | jq '.data.actor.account.cloud.linkedAccounts'
}

# Monitor data ingestion rate
monitor_ingestion_rate() {
  echo "Monitoring data ingestion rate..."
  
  ingestion_query='
    SELECT rate(count(*), 1 minute) as ingestRate
    FROM KafkaClusterSample, KafkaBrokerSample, KafkaTopicSample
    SINCE 1 hour ago
    TIMESERIES 5 minutes
  '
  
  nr1 api nrql -a ACCOUNT_ID -q "$ingestion_query"
}

# Check alert status
check_alert_status() {
  echo "Checking active alerts..."
  
  alert_query='
    FROM AlertViolation
    SELECT count(*), latest(label)
    WHERE targetName LIKE "%kafka%"
    FACET conditionName
    SINCE 1 hour ago
  '
  
  nr1 api nrql -a ACCOUNT_ID -q "$alert_query"
}

# Main monitoring loop
while true; do
  check_integration_status
  monitor_ingestion_rate
  check_alert_status
  
  sleep 300 # 5 minutes
done
```

### Backup and Recovery

```typescript
class BackupService {
  async backupConfiguration(): Promise<BackupResult> {
    const backup = {
      timestamp: Date.now(),
      version: this.getVersion(),
      components: {}
    };
    
    // Backup dashboards
    backup.components.dashboards = await this.backupDashboards();
    
    // Backup alert configurations
    backup.components.alerts = await this.backupAlerts();
    
    // Backup integration settings
    backup.components.integrations = await this.backupIntegrations();
    
    // Backup custom configurations
    backup.components.custom = await this.backupCustomConfigs();
    
    // Store backup
    const backupId = await this.storeBackup(backup);
    
    return {
      backupId,
      timestamp: backup.timestamp,
      components: Object.keys(backup.components),
      size: JSON.stringify(backup).length
    };
  }
  
  async restoreConfiguration(backupId: string): Promise<RestoreResult> {
    const backup = await this.loadBackup(backupId);
    const results = [];
    
    // Restore in correct order
    const restoreOrder = [
      'integrations',
      'dashboards',
      'alerts',
      'custom'
    ];
    
    for (const component of restoreOrder) {
      try {
        const result = await this.restoreComponent(
          component,
          backup.components[component]
        );
        results.push({ component, status: 'success', ...result });
      } catch (error) {
        results.push({ 
          component, 
          status: 'failed', 
          error: error.message 
        });
      }
    }
    
    return {
      backupId,
      timestamp: Date.now(),
      results
    };
  }
}
```

## Performance Optimization

### Query Optimization

```typescript
class QueryOptimizationService {
  async optimizeQueries(): Promise<void> {
    // Identify slow queries
    const slowQueries = await this.identifySlowQueries();
    
    for (const query of slowQueries) {
      const optimized = await this.optimizeQuery(query);
      
      if (optimized.improvement > 0.2) { // 20% improvement
        await this.replaceQuery(query.id, optimized.query);
        
        console.log(`Optimized query ${query.id}: ${optimized.improvement * 100}% improvement`);
      }
    }
  }
  
  private async identifySlowQueries(): Promise<SlowQuery[]> {
    const performanceData = await this.getQueryPerformance();
    
    return performanceData
      .filter(q => q.avgDuration > 5000) // > 5 seconds
      .sort((a, b) => b.avgDuration - a.avgDuration)
      .slice(0, 10); // Top 10 slowest
  }
  
  private optimizeQuery(query: SlowQuery): OptimizedQuery {
    let optimized = query.nrql;
    
    // Add LIMIT if missing
    if (!optimized.includes('LIMIT')) {
      optimized += ' LIMIT 100';
    }
    
    // Use SINCE instead of time functions
    optimized = optimized.replace(
      /WHERE timestamp > \d+/,
      'SINCE 1 hour ago'
    );
    
    // Pre-aggregate in subqueries
    if (optimized.includes('SELECT sum') && optimized.includes('FACET')) {
      optimized = this.addSubqueryAggregation(optimized);
    }
    
    return {
      original: query.nrql,
      query: optimized,
      improvement: this.estimateImprovement(query.nrql, optimized)
    };
  }
}
```

### Resource Management

```yaml
# Resource allocation guidelines
resources:
  metric_collection:
    cpu_allocation: 2_cores
    memory_allocation: 4_gb
    scaling:
      min_instances: 2
      max_instances: 10
      target_cpu: 70%
  
  data_processing:
    batch_size: 1000
    parallel_workers: 4
    timeout: 30_seconds
  
  caching:
    query_cache:
      size: 1_gb
      ttl: 300_seconds
    entity_cache:
      size: 500_mb
      ttl: 600_seconds
```

## Maintenance Procedures

### Regular Maintenance Tasks

```typescript
class MaintenanceScheduler {
  private schedule = [
    {
      task: 'cleanupOldData',
      frequency: 'daily',
      time: '02:00'
    },
    {
      task: 'optimizeQueries',
      frequency: 'weekly',
      day: 'sunday',
      time: '03:00'
    },
    {
      task: 'updateIntegrations',
      frequency: 'monthly',
      day: 1,
      time: '04:00'
    },
    {
      task: 'securityAudit',
      frequency: 'quarterly',
      months: [0, 3, 6, 9],
      day: 15
    }
  ];
  
  async executeMaintenance(): Promise<void> {
    const tasks = this.getScheduledTasks();
    
    for (const task of tasks) {
      try {
        console.log(`Starting maintenance task: ${task.name}`);
        
        const result = await this.executeTask(task);
        
        await this.logMaintenanceResult({
          task: task.name,
          status: 'success',
          duration: result.duration,
          details: result.details
        });
      } catch (error) {
        await this.handleMaintenanceError(task, error);
      }
    }
  }
  
  private async cleanupOldData(): Promise<MaintenanceResult> {
    const deleted = await this.deleteOldRecords({
      olderThan: 90, // days
      types: ['debug_logs', 'temporary_data']
    });
    
    return {
      duration: Date.now() - startTime,
      details: {
        recordsDeleted: deleted.count,
        spaceReclaimed: deleted.size
      }
    };
  }
}
```

### Upgrade Procedures

```bash
#!/bin/bash
# Upgrade procedure script

# Pre-upgrade checks
pre_upgrade_checks() {
  echo "Running pre-upgrade checks..."
  
  # Check current version
  current_version=$(nr1 nerdpack:info | grep version)
  echo "Current version: $current_version"
  
  # Backup current configuration
  ./backup_configuration.sh
  
  # Check compatibility
  if ! check_compatibility "$new_version"; then
    echo "ERROR: Version incompatible"
    exit 1
  fi
}

# Perform upgrade
perform_upgrade() {
  echo "Starting upgrade to version $new_version..."
  
  # Update code
  git fetch origin
  git checkout "v$new_version"
  
  # Install dependencies
  npm install
  
  # Run migrations if needed
  if [ -f "migrations/to_${new_version}.js" ]; then
    node "migrations/to_${new_version}.js"
  fi
  
  # Deploy new version
  nr1 nerdpack:publish
  nr1 nerdpack:deploy --channel STABLE
  
  # Tag deployment
  nr1 nerdpack:tag --tag "v$new_version"
}

# Post-upgrade validation
post_upgrade_validation() {
  echo "Validating upgrade..."
  
  # Check deployment status
  deployment_status=$(nr1 nerdpack:info)
  
  # Run health checks
  ./health_check.sh
  
  # Validate data flow
  if ! validate_data_flow; then
    echo "ERROR: Data flow validation failed"
    rollback_upgrade
    exit 1
  fi
  
  echo "Upgrade completed successfully!"
}
```

## Troubleshooting Guide

### Common Issues

```typescript
interface TroubleshootingGuide {
  issues: {
    no_data_visible: {
      symptoms: ['Empty dashboards', 'No entities shown'];
      causes: [
        'Integration not configured',
        'Permissions issues',
        'Network connectivity',
        'Time range too narrow'
      ];
      solutions: [
        'Verify integration status in New Relic UI',
        'Check IAM role permissions',
        'Test network connectivity to APIs',
        'Expand time range to last 24 hours'
      ];
    };
    
    high_latency: {
      symptoms: ['Slow dashboard loading', 'Query timeouts'];
      causes: [
        'Large data volumes',
        'Inefficient queries',
        'Too many concurrent users'
      ];
      solutions: [
        'Implement query optimization',
        'Add data sampling',
        'Scale infrastructure',
        'Enable caching'
      ];
    };
    
    missing_metrics: {
      symptoms: ['Partial data', 'Specific metrics not appearing'];
      causes: [
        'Metric not enabled in Kafka',
        'CloudWatch metrics not configured',
        'API rate limiting'
      ];
      solutions: [
        'Enable JMX metrics in Kafka',
        'Configure enhanced monitoring',
        'Implement request throttling'
      ];
    };
  };
}
```

### Diagnostic Tools

```typescript
class DiagnosticService {
  async runDiagnostics(): Promise<DiagnosticReport> {
    const tests = [
      this.testNewRelicConnectivity(),
      this.testKafkaConnectivity(),
      this.testDataIngestion(),
      this.testQueryPerformance(),
      this.testAlertDelivery()
    ];
    
    const results = await Promise.all(tests);
    
    return {
      timestamp: Date.now(),
      environment: this.getEnvironment(),
      results,
      recommendations: this.generateRecommendations(results)
    };
  }
  
  private async testKafkaConnectivity(): Promise<TestResult> {
    const endpoints = [
      { type: 'AWS_MSK', endpoint: 'kafka.amazonaws.com' },
      { type: 'Confluent', endpoint: 'api.confluent.cloud' }
    ];
    
    const results = [];
    
    for (const endpoint of endpoints) {
      const start = Date.now();
      try {
        await this.ping(endpoint.endpoint);
        results.push({
          endpoint: endpoint.type,
          status: 'connected',
          latency: Date.now() - start
        });
      } catch (error) {
        results.push({
          endpoint: endpoint.type,
          status: 'failed',
          error: error.message
        });
      }
    }
    
    return {
      test: 'kafka_connectivity',
      passed: results.every(r => r.status === 'connected'),
      details: results
    };
  }
}
```

## Monitoring Best Practices

### 1. Deployment
- Use infrastructure as code
- Implement automated testing
- Maintain staging environments
- Document all configurations

### 2. Operations
- Regular health checks
- Proactive monitoring
- Automated alerting
- Runbook documentation

### 3. Performance
- Query optimization
- Resource monitoring
- Capacity planning
- Load testing

### 4. Maintenance
- Regular updates
- Configuration backups
- Security patching
- Performance tuning

## Conclusion

Successful deployment and operation of the Message Queues monitoring system requires careful planning, automated procedures, and ongoing maintenance. By following these guidelines and best practices, teams can ensure reliable and efficient Kafka monitoring at scale.