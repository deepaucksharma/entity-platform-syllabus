# System Architecture

## Overview

The New Relic Message Queues monitoring system is a sophisticated observability platform built as a New Relic One (NR1) Nerdpack. It provides comprehensive monitoring capabilities for Kafka message queue infrastructures across multiple cloud providers.

## Core Architecture Components

### 1. Platform Foundation

#### New Relic One Integration
```json
{
  "schemaType": "NERDPACK",
  "id": "c0086118-99a7-455b-a5ee-8fd35e749fbd",
  "name": "message-queues",
  "displayName": "Message Queues",
  "description": "Monitor your Message queues here.",
  "teamStoreId": "617",
  "sdkVersion": 3,
  "repositoryUrl": "git@source.datanerd.us:APM/message-queues.git"
}
```

The application leverages:
- **NR1 SDK v3**: Latest platform capabilities
- **Entity Platform**: First-class entity synthesis and relationships
- **NerdGraph API**: GraphQL-based data access
- **NRQL Engine**: Powerful query capabilities

### 2. Entity-Centric Architecture

#### Entity Hierarchy Model
```
┌─────────────────────────────────────────────────────┐
│                    Account                          │
│                  (1:N relationship)                 │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
┌───────▼────────┐       ┌─────────▼──────────┐
│   AWS MSK      │       │ Confluent Cloud    │
│   Provider     │       │    Provider        │
└───────┬────────┘       └─────────┬──────────┘
        │                           │
        │ (1:N)                     │ (1:N)
        │                           │
┌───────▼────────┐       ┌─────────▼──────────┐
│    Clusters    │       │     Clusters       │
└───────┬────────┘       └─────────┬──────────┘
        │                           │
    ┌───┴───┐                   ┌───┴───┐
    │       │                   │       │
┌───▼───┐ ┌▼────┐         ┌────▼───┐ ┌▼────┐
│Brokers│ │Topics│         │Topics  │ │Parts│
└───────┘ └──┬───┘         └────┬───┘ └─────┘
             │                  │
         ┌───▼───┐          ┌───▼───┐
         │ Parts │          │ APM   │
         └───────┘          │Entities│
                            └───────┘
```

#### Entity Type Definitions

**AWS MSK Entities**:
- `AWSMSKCLUSTER`: Represents MSK Kafka clusters
- `AWSMSKBROKER`: Individual broker nodes within clusters
- `AWSMSKTOPIC`: Kafka topics with metrics and relationships

**Confluent Cloud Entities**:
- `CONFLUENTCLOUDCLUSTER`: Confluent-managed Kafka clusters
- `CONFLUENTCLOUDKAFKATOPIC`: Topics in Confluent Cloud
- `CONFLUENTCLOUDPARTITION`: Individual partitions (optional)

### 3. Application Structure

#### Directory Layout
```
message-queues/
├── nerdlets/                    # Main application screens
│   ├── home/                    # Landing dashboard
│   ├── summary/                 # Account summary view
│   └── mq-detail/              # Detailed entity view
├── common/
│   ├── components/             # Reusable UI components
│   ├── hooks/                  # Custom React hooks
│   ├── utils/                  # Utility functions
│   ├── config/                 # Configuration files
│   └── types/                  # TypeScript definitions
├── visualizations/             # Custom visualizations
└── launchers/                  # Application entry points
```

### 4. Data Architecture

#### Data Sources
1. **AWS MSK Integration**
   - CloudWatch Polling (5-minute intervals)
   - Metric Streams (real-time)
   - Event Types: `AwsMskClusterSample`, `AwsMskBrokerSample`, `AwsMskTopicSample`

2. **Confluent Cloud Integration**
   - Metrics API polling
   - Event Type: `Metric` with `confluent.*` namespace

#### Query Layer
```typescript
// Central query system architecture
interface QuerySystem {
  // GraphQL for entity discovery and relationships
  graphQL: {
    entitySearch: EntitySearchQuery;
    relationships: RelationshipQuery;
    metadata: MetadataQuery;
  };
  
  // NRQL for metrics and analytics
  nrql: {
    metrics: MetricQuery;
    aggregations: AggregationQuery;
    timeseries: TimeSeriesQuery;
  };
  
  // Query builders and utilities
  builders: {
    provider: ProviderQueryBuilder;
    filter: FilterQueryBuilder;
    optimization: QueryOptimizer;
  };
}
```

### 5. Component Architecture

#### Core Component Hierarchy
```
App
├── Nerdlets (Screen Components)
│   ├── Home
│   │   ├── FilterBar
│   │   ├── SearchBar
│   │   └── DataTable
│   ├── Summary
│   │   ├── Billboards
│   │   ├── Charts
│   │   └── Navigator
│   └── Detail
│       ├── EntityPanel
│       └── MetricsView
├── Shared Components
│   ├── EntityNavigator
│   │   ├── HoneyCombView
│   │   ├── ControlBar
│   │   └── Legend
│   └── FilterSystem
│       ├── PredefinedFilters
│       └── CustomFilters
└── Utility Layer
    ├── QueryUtils
    ├── DataUtils
    └── Helpers
```

### 6. State Management

#### State Layers
1. **URL State**: Deep linking and navigation
2. **Nerdlet State**: Cross-component communication
3. **Local State**: Component-specific UI state
4. **Cache Layer**: Performance optimization

```typescript
// State management architecture
interface StateArchitecture {
  // URL-based state for deep linking
  urlState: {
    provider: string;
    account: string;
    filters: FilterSet[];
    timeRange: TimeRange;
    selectedEntity?: string;
  };
  
  // Shared nerdlet state
  nerdletState: {
    homeFilters: HomeFilterState;
    summaryFilters: SummaryFilterState;
    navigatorConfig: NavigatorConfig;
  };
  
  // Component local state
  localState: {
    loading: boolean;
    error: Error | null;
    data: any;
  };
  
  // Cache management
  cache: {
    entities: Map<string, CachedEntity>;
    metrics: Map<string, CachedMetric>;
    ttl: number;
  };
}
```

### 7. Integration Points

#### Platform Integrations
- **New Relic APM**: Producer/consumer relationship mapping
- **New Relic Infrastructure**: Host and container context
- **New Relic Alerts**: Alert policy integration
- **New Relic Dashboards**: Export capabilities

#### External Integrations
- **AWS CloudWatch**: MSK metrics source
- **Confluent Cloud API**: Confluent metrics source
- **Entity Synthesis Pipeline**: Custom entity creation

### 8. Security Architecture

#### Access Control Layers
```
User → New Relic RBAC → Account Access → Entity Permissions → Data Access
```

#### Security Features
- Role-based access control (RBAC)
- Account-scoped data isolation
- Encrypted data transmission
- No credential storage
- Audit logging

### 9. Performance Architecture

#### Optimization Strategies
1. **Query Optimization**
   - Nested aggregations for efficiency
   - Strategic LIMIT usage
   - Facet optimization

2. **Component Optimization**
   - React.memo for expensive components
   - useMemo for complex calculations
   - Virtual scrolling for large lists

3. **Data Optimization**
   - 15-minute cache TTL
   - Progressive loading
   - Debounced searches

### 10. Scalability Design

#### Scaling Capabilities
- **Entity Count**: Supports 1000+ clusters
- **Topic Count**: Handles 100,000+ topics
- **Concurrent Users**: Designed for enterprise scale
- **Data Volume**: Optimized for high-throughput metrics

#### Scaling Strategies
```typescript
// Scalability patterns
const ScalabilityPatterns = {
  // Virtual scrolling for large datasets
  virtualization: {
    enabled: entityCount > 500,
    pageSize: 50,
    bufferSize: 10
  },
  
  // Progressive data loading
  progressiveLoading: {
    initial: 'essential',
    secondary: 'detailed',
    tertiary: 'historical'
  },
  
  // Query optimization
  queryOptimization: {
    maxLimit: entityCount > 1000 ? 'MAX' : 100,
    aggregationLevel: 'cluster' | 'broker' | 'topic',
    timeWindow: 'adaptive'
  }
};
```

### 11. Deployment Architecture

#### Build Pipeline
```bash
# Development
nr1 nerdpack:serve

# Production Build
nr1 nerdpack:build

# Publishing
nr1 nerdpack:publish
nr1 nerdpack:deploy -c STABLE
```

#### Deployment Targets
- **DEV**: Development testing
- **BETA**: Pre-production validation
- **STABLE**: Production deployment

### 12. Monitoring Architecture

#### Self-Monitoring
The application monitors its own performance:
- Query execution times
- Component render performance
- Error rates and types
- User interaction metrics

```typescript
// Self-monitoring implementation
instrumentation.noticeError(error, {
  component: 'MessageQueues',
  version: '1.0.0',
  context: {
    provider,
    entityCount,
    queryDuration
  }
});
```

## Architecture Principles

### 1. Entity-First Design
Every piece of Kafka infrastructure is represented as a first-class entity with:
- Unique identity (GUID)
- Relationships to other entities
- Alertability
- Metadata and tags

### 2. Provider Abstraction
Common interfaces abstract provider differences:
```typescript
interface ProviderAdapter {
  getEntities(): Entity[];
  getMetrics(): Metric[];
  mapToCommonModel(): CommonModel;
}
```

### 3. Query Composition
Modular query building enables:
- Dynamic filter application
- Provider-specific optimization
- Reusable query patterns

### 4. Progressive Enhancement
Features gracefully degrade:
- Basic functionality without JavaScript
- Enhanced features with modern browsers
- Mobile-responsive design

### 5. Extensibility
Designed for future expansion:
- New provider support
- Additional message queue types
- Custom visualizations
- Plugin architecture

## Technology Stack

### Frontend
- **React 17+**: Component framework
- **TypeScript**: Type safety
- **SCSS**: Styling with CSS modules
- **D3.js**: Advanced visualizations

### Platform
- **New Relic One SDK**: Platform integration
- **NerdGraph**: GraphQL API
- **NRQL**: Query language
- **Entity Platform**: Entity management

### Development
- **Jest**: Unit testing
- **Playwright**: E2E testing
- **ESLint**: Code quality
- **Prettier**: Code formatting

### Build Tools
- **Webpack**: Module bundling
- **Babel**: JavaScript transpilation
- **nr1 CLI**: Platform tooling

## Conclusion

This architecture provides a robust, scalable foundation for Kafka infrastructure monitoring within the New Relic ecosystem. The entity-centric design, combined with powerful query capabilities and a modular component structure, enables comprehensive observability while maintaining performance and user experience.