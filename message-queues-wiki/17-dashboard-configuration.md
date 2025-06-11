# Dashboard Configuration

## Overview

This document provides detailed configuration guidelines for creating and customizing Message Queues dashboards in New Relic. It covers dashboard structure, widget configurations, variable strategies, and best practices for effective Kafka monitoring.

## Dashboard Architecture

### Dashboard Hierarchy

```
Message Queues Dashboards
├── Executive Dashboard (High-level KPIs)
├── Operational Dashboards
│   ├── Cluster Health Dashboard
│   ├── Performance Monitoring Dashboard
│   └── Capacity Planning Dashboard
├── Diagnostic Dashboards
│   ├── Troubleshooting Dashboard
│   ├── Consumer Lag Analysis Dashboard
│   └── Broker Deep Dive Dashboard
├── Specialized Dashboards
│   ├── Topic Analytics Dashboard
│   ├── APM Correlation Dashboard
│   └── Multi-Region Overview Dashboard
└── Custom Team Dashboards
```

## Core Dashboard Configurations

### 1. Executive Dashboard

```json
{
  "name": "Kafka Executive Overview",
  "description": "High-level KPIs for Kafka infrastructure",
  "permissions": "PUBLIC_READ_WRITE",
  "pages": [
    {
      "name": "KPI Overview",
      "description": "Key performance indicators",
      "widgets": [
        {
          "title": "Kafka Infrastructure Health",
          "row": 1,
          "column": 1,
          "width": 12,
          "height": 3,
          "configuration": {
            "type": "viz.billboard",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT percentage(count(*), WHERE healthScore >= 80) as 'Health Score' FROM (SELECT latest(healthScore) as healthScore FROM KafkaClusterSample FACET clusterName LIMIT MAX)"
              }
            ],
            "thresholds": [
              { "value": 90, "type": "success" },
              { "value": 80, "type": "warning" },
              { "value": 0, "type": "critical" }
            ]
          }
        },
        {
          "title": "Total Clusters",
          "row": 1,
          "column": 1,
          "width": 3,
          "height": 3,
          "configuration": {
            "type": "viz.billboard",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT uniqueCount(clusterName) as 'Total Clusters' FROM KafkaClusterSample WHERE provider IN (${provider})"
              }
            ]
          }
        },
        {
          "title": "Critical Alerts",
          "row": 1,
          "column": 4,
          "width": 3,
          "height": 3,
          "configuration": {
            "type": "viz.billboard",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT count(*) as 'Critical Alerts' FROM AlertIncident WHERE priority = 'CRITICAL' AND targetName LIKE '%kafka%' SINCE 1 hour ago"
              }
            ],
            "critical": { "value": 1 }
          }
        },
        {
          "title": "Total Throughput",
          "row": 1,
          "column": 7,
          "width": 3,
          "height": 3,
          "configuration": {
            "type": "viz.billboard",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT rate(sum(bytesInPerSec + bytesOutPerSec), 1 second) as 'Total Throughput' FROM KafkaBrokerSample"
              }
            ],
            "unit": "BYTES_PER_SECOND"
          }
        },
        {
          "title": "Consumer Lag Trend",
          "row": 1,
          "column": 10,
          "width": 3,
          "height": 3,
          "configuration": {
            "type": "viz.sparkline",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT sum(maxLag) FROM KafkaConsumerSample TIMESERIES AUTO"
              }
            ]
          }
        }
      ]
    }
  ],
  "variables": [
    {
      "name": "accountId",
      "title": "Account",
      "type": "nrql",
      "nrqlQuery": {
        "query": "SELECT uniques(accountId) FROM KafkaClusterSample SINCE 1 day ago"
      },
      "defaultValue": "all"
    },
    {
      "name": "provider",
      "title": "Provider",
      "type": "enum",
      "items": [
        { "title": "All Providers", "value": "'AWS_MSK','CONFLUENT_CLOUD'" },
        { "title": "AWS MSK", "value": "'AWS_MSK'" },
        { "title": "Confluent Cloud", "value": "'CONFLUENT_CLOUD'" }
      ],
      "defaultValue": "'AWS_MSK','CONFLUENT_CLOUD'"
    }
  ]
}
```

### 2. Cluster Health Dashboard

```json
{
  "name": "Kafka Cluster Health Monitor",
  "pages": [
    {
      "name": "Health Overview",
      "widgets": [
        {
          "title": "Cluster Health Matrix",
          "row": 1,
          "column": 1,
          "width": 12,
          "height": 4,
          "configuration": {
            "type": "viz.heatmap",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT latest(healthScore) FROM KafkaClusterSample FACET clusterName, dateOf(timestamp) SINCE 7 days ago"
              }
            ],
            "colors": {
              "type": "diverging",
              "low": "#d62728",
              "mid": "#ffdd00", 
              "high": "#2ca02c"
            }
          }
        },
        {
          "title": "Offline Partitions by Cluster",
          "row": 5,
          "column": 1,
          "width": 6,
          "height": 3,
          "configuration": {
            "type": "viz.bar",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT latest(offlinePartitions) FROM KafkaClusterSample WHERE offlinePartitions > 0 FACET clusterName"
              }
            ],
            "colors": {
              "seriesOverrides": [
                { "seriesName": "offlinePartitions", "color": "#d62728" }
              ]
            }
          }
        },
        {
          "title": "Under-Replicated Partitions Trend",
          "row": 5,
          "column": 7,
          "width": 6,
          "height": 3,
          "configuration": {
            "type": "viz.area",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT sum(underReplicatedPartitions) FROM KafkaClusterSample FACET clusterName TIMESERIES AUTO SINCE 24 hours ago"
              }
            ],
            "stacked": true,
            "fillOpacity": 0.3
          }
        }
      ]
    },
    {
      "name": "Broker Health",
      "widgets": [
        {
          "title": "Broker Status Grid",
          "row": 1,
          "column": 1,
          "width": 12,
          "height": 6,
          "configuration": {
            "type": "custom:kafka-broker-grid",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT latest(status), latest(cpuUsage), latest(diskUsage), latest(networkIn), latest(networkOut) FROM KafkaBrokerSample FACET brokerId, clusterName"
              }
            ],
            "visualization": {
              "type": "grid",
              "cellSize": 80,
              "showMetrics": true,
              "colorBy": "status"
            }
          }
        }
      ]
    }
  ]
}
```

### 3. Performance Monitoring Dashboard

```json
{
  "name": "Kafka Performance Analytics",
  "pages": [
    {
      "name": "Throughput Analysis",
      "widgets": [
        {
          "title": "Cluster Throughput Overview",
          "row": 1,
          "column": 1,
          "width": 12,
          "height": 4,
          "configuration": {
            "type": "viz.line",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT sum(bytesInPerSec) as 'Bytes In', sum(bytesOutPerSec) as 'Bytes Out' FROM KafkaBrokerSample FACET clusterName TIMESERIES AUTO SINCE ${timeRange}"
              }
            ],
            "yAxisLeft": {
              "zero": true,
              "label": "Throughput (bytes/sec)"
            },
            "legend": {
              "enabled": true
            },
            "units": {
              "unit": "BYTES_PER_SECOND"
            }
          }
        },
        {
          "title": "Top Topics by Throughput",
          "row": 5,
          "column": 1,
          "width": 6,
          "height": 4,
          "configuration": {
            "type": "viz.pie",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT sum(bytesInPerSec + bytesOutPerSec) FROM KafkaTopicSample FACET topicName LIMIT 10"
              }
            ],
            "pieChartOptions": {
              "donut": true,
              "donutWidth": 40
            }
          }
        },
        {
          "title": "Message Rate by Cluster",
          "row": 5,
          "column": 7,
          "width": 6,
          "height": 4,
          "configuration": {
            "type": "viz.stacked-bar",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT rate(sum(messagesInPerSec), 1 minute) FROM KafkaTopicSample FACET clusterName TIMESERIES AUTO"
              }
            ]
          }
        }
      ]
    },
    {
      "name": "Latency Analysis",
      "widgets": [
        {
          "title": "Producer Latency Distribution",
          "row": 1,
          "column": 1,
          "width": 6,
          "height": 4,
          "configuration": {
            "type": "viz.histogram",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT histogram(requestLatency, 20, 10) FROM KafkaProducerSample WHERE appName IS NOT NULL"
              }
            ]
          }
        },
        {
          "title": "Request Latency Percentiles",
          "row": 1,
          "column": 7,
          "width": 6,
          "height": 4,
          "configuration": {
            "type": "viz.line",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT percentile(requestLatency, 50, 75, 95, 99) FROM KafkaProducerSample TIMESERIES AUTO"
              }
            ]
          }
        }
      ]
    }
  ],
  "variables": [
    {
      "name": "timeRange",
      "title": "Time Range",
      "type": "time",
      "defaultValue": {
        "duration": 3600000,
        "endTime": null
      }
    },
    {
      "name": "cluster",
      "title": "Cluster",
      "type": "nrql",
      "nrqlQuery": {
        "accountId": "${accountId}",
        "query": "SELECT uniques(clusterName) FROM KafkaClusterSample"
      }
    }
  ]
}
```

### 4. Consumer Lag Analysis Dashboard

```json
{
  "name": "Kafka Consumer Lag Analysis",
  "pages": [
    {
      "name": "Lag Overview",
      "widgets": [
        {
          "title": "Consumer Group Lag Heatmap",
          "row": 1,
          "column": 1,
          "width": 12,
          "height": 5,
          "configuration": {
            "type": "viz.heatmap",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT latest(maxLag) FROM KafkaConsumerSample FACET consumerGroup, topic SINCE 6 hours ago"
              }
            ],
            "colors": {
              "type": "sequential",
              "stops": [
                { "value": 0, "color": "#2ca02c" },
                { "value": 1000, "color": "#ffdd00" },
                { "value": 10000, "color": "#ff7f0e" },
                { "value": 100000, "color": "#d62728" }
              ]
            }
          }
        },
        {
          "title": "Lag Trend by Consumer Group",
          "row": 6,
          "column": 1,
          "width": 12,
          "height": 4,
          "configuration": {
            "type": "viz.line",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT max(lag) FROM KafkaConsumerSample FACET consumerGroup TIMESERIES AUTO SINCE 24 hours ago LIMIT 20"
              }
            ],
            "legend": {
              "enabled": true
            },
            "yAxisLeft": {
              "zero": true,
              "label": "Lag (messages)"
            }
          }
        }
      ]
    },
    {
      "name": "Consumer Performance",
      "widgets": [
        {
          "title": "Consumption Rate vs Production Rate",
          "row": 1,
          "column": 1,
          "width": 12,
          "height": 4,
          "configuration": {
            "type": "viz.line",
            "nrqlQueries": [
              {
                "accountId": "${accountId}",
                "query": "SELECT rate(sum(messagesProduced), 1 minute) as 'Production Rate', rate(sum(messagesConsumed), 1 minute) as 'Consumption Rate' FROM KafkaTopicSample, KafkaConsumerSample WHERE topic = ${topic} TIMESERIES AUTO"
              }
            ],
            "nullValues": {
              "nullValue": "zero"
            }
          }
        }
      ]
    }
  ]
}
```

## Widget Configuration Patterns

### Billboard Widgets

```json
{
  "type": "viz.billboard",
  "configuration": {
    "nrqlQueries": [{
      "accountId": "${accountId}",
      "query": "SELECT count(*) FROM KafkaClusterSample"
    }],
    "thresholds": [
      { "value": 10, "type": "success" },
      { "value": 5, "type": "warning" },
      { "value": 0, "type": "critical" }
    ],
    "warnings": {
      "warningCriticalThreshold": {
        "aboveThreshold": true
      }
    }
  }
}
```

### Time Series Widgets

```json
{
  "type": "viz.line",
  "configuration": {
    "nrqlQueries": [{
      "accountId": "${accountId}",
      "query": "SELECT average(cpuUsage) FROM KafkaBrokerSample TIMESERIES AUTO"
    }],
    "yAxisLeft": {
      "zero": false,
      "min": 0,
      "max": 100
    },
    "units": {
      "unit": "PERCENTAGE"
    },
    "colors": {
      "seriesOverrides": [
        { "seriesName": "CPU Usage", "color": "#ff7f0e" }
      ]
    },
    "fillOpacity": 0.1,
    "legend": {
      "enabled": true
    }
  }
}
```

### Table Widgets

```json
{
  "type": "viz.table",
  "configuration": {
    "nrqlQueries": [{
      "accountId": "${accountId}",
      "query": "SELECT latest(status), average(cpuUsage), average(diskUsage) FROM KafkaBrokerSample FACET brokerId, clusterName"
    }],
    "formatters": [
      {
        "columnName": "cpuUsage",
        "type": "percentage",
        "precision": 1
      },
      {
        "columnName": "status",
        "type": "custom",
        "customFormat": {
          "healthy": { "displayValue": "✓", "color": "green" },
          "unhealthy": { "displayValue": "✗", "color": "red" }
        }
      }
    ]
  }
}
```

## Variable Configuration

### Dynamic Variables

```json
{
  "variables": [
    {
      "name": "cluster",
      "title": "Select Cluster",
      "type": "nrql",
      "multiSelect": true,
      "nrqlQuery": {
        "accountId": "${accountId}",
        "query": "SELECT uniques(clusterName) FROM KafkaClusterSample WHERE provider = ${provider}"
      },
      "defaultValue": "all"
    },
    {
      "name": "topic",
      "title": "Select Topic",
      "type": "nrql",
      "searchable": true,
      "nrqlQuery": {
        "accountId": "${accountId}",
        "query": "SELECT uniques(topicName) FROM KafkaTopicSample WHERE clusterName IN (${cluster})"
      }
    },
    {
      "name": "timeRange",
      "title": "Time Range",
      "type": "time",
      "defaultValue": {
        "duration": 3600000
      }
    }
  ]
}
```

### Variable Dependencies

```typescript
// Variable dependency chain
const variableDependencies = {
  provider: {
    dependents: ['cluster']
  },
  cluster: {
    dependencies: ['provider'],
    dependents: ['broker', 'topic']
  },
  topic: {
    dependencies: ['cluster'],
    dependents: ['partition', 'consumerGroup']
  },
  consumerGroup: {
    dependencies: ['topic']
  }
};
```

## Dashboard Linking

### Cross-Dashboard Navigation

```json
{
  "title": "Cluster Details",
  "configuration": {
    "type": "viz.markdown",
    "text": "[View Cluster Dashboard](/launcher/dashboards.browse?dashboard=${clusterDashboardId}&var-cluster=${cluster})"
  }
}
```

### Facet Drilldown

```json
{
  "configuration": {
    "facetDrilldown": {
      "enabled": true,
      "dashboardId": "${detailDashboardId}",
      "variableMappings": {
        "clusterName": "cluster",
        "topicName": "topic"
      }
    }
  }
}
```

## Advanced Configurations

### Custom Visualizations

```javascript
// Custom Kafka topology visualization
const KafkaTopologyWidget = {
  type: "custom:kafka-topology",
  configuration: {
    nrqlQueries: [{
      accountId: "${accountId}",
      query: `
        FROM KafkaClusterSample, KafkaBrokerSample, KafkaTopicSample
        SELECT 
          latest(clusterName) as cluster,
          latest(brokerId) as broker,
          latest(topicName) as topic,
          latest(partitionCount) as partitions
        WHERE clusterName IS NOT NULL
      `
    }],
    visualization: {
      layout: "hierarchical",
      nodeSize: "dynamic",
      colorBy: "health",
      showMetrics: true
    }
  }
};
```

### Composite Widgets

```json
{
  "title": "Kafka Health Summary",
  "type": "composite",
  "widgets": [
    {
      "type": "viz.billboard",
      "row": 1,
      "column": 1,
      "width": 3,
      "query": "SELECT latest(healthScore) FROM KafkaClusterSample"
    },
    {
      "type": "viz.sparkline",
      "row": 1,
      "column": 4,
      "width": 9,
      "query": "SELECT average(healthScore) FROM KafkaClusterSample TIMESERIES AUTO"
    }
  ]
}
```

## Performance Optimization

### Query Optimization

```sql
-- Use pre-aggregated data
FROM (
  SELECT 
    average(cpuUsage) as avgCpu,
    max(cpuUsage) as maxCpu
  FROM KafkaBrokerSample 
  FACET brokerId 
  LIMIT MAX
)
SELECT 
  average(avgCpu) as clusterAvgCpu,
  max(maxCpu) as clusterMaxCpu
```

### Widget Loading Strategies

```json
{
  "loadingStrategy": {
    "type": "lazy",
    "priority": "viewport",
    "refreshInterval": {
      "visible": 60000,
      "hidden": 300000
    }
  }
}
```

## Best Practices

### 1. Dashboard Organization
- Group related metrics in pages
- Use consistent widget sizing
- Implement logical flow from overview to detail
- Limit widgets per page (12-15 max)

### 2. Variable Usage
- Use variables for all entity selections
- Implement cascading variables
- Set sensible defaults
- Enable multi-select where appropriate

### 3. Visual Design
- Use consistent color schemes
- Apply thresholds meaningfully
- Include units and labels
- Optimize for different screen sizes

### 4. Performance
- Limit time ranges for heavy queries
- Use sampling for large datasets
- Implement query result caching
- Monitor dashboard load times

### 5. Maintenance
- Version control dashboard JSON
- Document custom configurations
- Regular review and optimization
- Monitor widget error rates

## Conclusion

Effective dashboard configuration is crucial for successful Kafka monitoring. By following these patterns and best practices, teams can create informative, performant, and maintainable dashboards that provide real value for operations and troubleshooting.