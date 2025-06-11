# Development Guide

## Overview

This guide provides comprehensive information for developers working on the Message Queues monitoring system. It covers the development environment setup, coding standards, architecture patterns, and contribution guidelines.

## Development Environment

### Prerequisites

```yaml
# Required software versions
requirements:
  node:
    version: ">=16.0.0"
    npm: ">=8.0.0"
  
  new_relic_cli:
    version: ">=2.0.0"
    installation: "npm install -g @newrelic/nr1"
  
  development_tools:
    - git: ">=2.30.0"
    - vscode: "recommended"
    - docker: "optional, for testing"
  
  accounts:
    new_relic:
      - type: "developer"
      - permissions: ["nerdpack:write", "api:read"]
    
    kafka_test_environment:
      - aws_sandbox: "optional"
      - confluent_trial: "optional"
```

### Environment Setup

```bash
#!/bin/bash
# Development environment setup script

# Clone repository
git clone https://github.com/newrelic/message-queues.git
cd message-queues

# Install dependencies
npm install

# Configure New Relic profile
nr1 profiles:add --name dev --api-key $NR_API_KEY --region US

# Set default profile
nr1 profiles:default --name dev

# Create local configuration
cp .env.example .env
echo "Configure your .env file with appropriate values"

# Install development tools
npm install -g \
  @typescript-eslint/eslint-plugin \
  @typescript-eslint/parser \
  prettier \
  husky \
  lint-staged

# Setup git hooks
npx husky install
```

### Local Development Configuration

```typescript
// config/development.ts
export const developmentConfig = {
  nr1: {
    accountId: process.env.NR_ACCOUNT_ID || 'YOUR_ACCOUNT_ID',
    nerdletId: 'message-queues-dev',
    catalogId: 'message-queues-dev',
    displayName: 'Message Queues (Dev)'
  },
  
  kafka: {
    mockData: process.env.USE_MOCK_DATA === 'true',
    testClusters: [
      {
        name: 'dev-kafka-cluster',
        type: 'AWS_MSK',
        endpoint: process.env.KAFKA_DEV_ENDPOINT
      }
    ]
  },
  
  features: {
    debugMode: true,
    performanceMetrics: true,
    experimentalFeatures: true
  }
};
```

## Project Structure

### Directory Organization

```
message-queues/
├── src/
│   ├── nerdlets/              # Nerdlet applications
│   │   ├── home/              # Home dashboard nerdlet
│   │   ├── summary/           # Summary view nerdlet
│   │   └── mq-detail/         # Detail view nerdlet
│   ├── common/
│   │   ├── components/        # Shared React components
│   │   ├── hooks/            # Custom React hooks
│   │   ├── utils/            # Utility functions
│   │   ├── queries/          # NRQL query builders
│   │   └── types/            # TypeScript definitions
│   ├── visualizations/        # Custom visualizations
│   └── launchers/            # Application launchers
├── config/                    # Configuration files
├── scripts/                   # Build and utility scripts
├── tests/                     # Test files
│   ├── unit/                 # Unit tests
│   ├── integration/          # Integration tests
│   └── e2e/                  # End-to-end tests
├── docs/                      # Documentation
└── package.json              # Project configuration
```

### Module Architecture

```typescript
// Example module structure
// src/common/modules/kafka-health/index.ts

export interface KafkaHealthModule {
  // Public API
  calculateClusterHealth(cluster: Cluster): HealthScore;
  calculateBrokerHealth(broker: Broker): HealthScore;
  calculateTopicHealth(topic: Topic): HealthScore;
  
  // Health factors
  factors: {
    availability: AvailabilityCalculator;
    performance: PerformanceCalculator;
    reliability: ReliabilityCalculator;
  };
  
  // Configuration
  config: HealthCalculationConfig;
}

// Implementation
class KafkaHealthModuleImpl implements KafkaHealthModule {
  constructor(private config: HealthCalculationConfig) {
    this.factors = {
      availability: new AvailabilityCalculator(config),
      performance: new PerformanceCalculator(config),
      reliability: new ReliabilityCalculator(config)
    };
  }
  
  calculateClusterHealth(cluster: Cluster): HealthScore {
    // Implementation following single responsibility principle
    const availability = this.factors.availability.calculate(cluster);
    const performance = this.factors.performance.calculate(cluster);
    const reliability = this.factors.reliability.calculate(cluster);
    
    return this.combineScores({ availability, performance, reliability });
  }
}

// Factory function
export const createKafkaHealthModule = (
  config?: Partial<HealthCalculationConfig>
): KafkaHealthModule => {
  return new KafkaHealthModuleImpl({
    ...defaultConfig,
    ...config
  });
};
```

## Coding Standards

### TypeScript Guidelines

```typescript
// 1. Use explicit types, avoid 'any'
// ❌ Bad
function processData(data: any) {
  return data.map(item => item.value);
}

// ✅ Good
interface DataItem {
  id: string;
  value: number;
  metadata?: Record<string, unknown>;
}

function processData(data: DataItem[]): number[] {
  return data.map(item => item.value);
}

// 2. Use enums for constants
enum KafkaProvider {
  AWS_MSK = 'AWS_MSK',
  CONFLUENT_CLOUD = 'CONFLUENT_CLOUD'
}

// 3. Prefer interfaces over type aliases for objects
interface ClusterConfig {
  name: string;
  provider: KafkaProvider;
  region: string;
}

// 4. Use generic types for reusable components
interface ApiResponse<T> {
  data: T;
  status: number;
  timestamp: number;
}

// 5. Document complex types
/**
 * Represents a Kafka cluster health calculation result
 * @property score - Health score from 0-100
 * @property factors - Individual factor scores
 * @property issues - List of identified health issues
 */
interface HealthCalculation {
  score: number;
  factors: Record<string, number>;
  issues: HealthIssue[];
}
```

### React Component Patterns

```typescript
// 1. Functional components with TypeScript
interface KafkaClusterCardProps {
  cluster: Cluster;
  onSelect?: (cluster: Cluster) => void;
  className?: string;
}

export const KafkaClusterCard: React.FC<KafkaClusterCardProps> = ({
  cluster,
  onSelect,
  className
}) => {
  // Use hooks for state and effects
  const [expanded, setExpanded] = useState(false);
  const { healthScore, loading } = useClusterHealth(cluster.id);
  
  // Memoize expensive calculations
  const healthStatus = useMemo(() => 
    getHealthStatus(healthScore),
    [healthScore]
  );
  
  // Event handlers
  const handleClick = useCallback(() => {
    onSelect?.(cluster);
  }, [cluster, onSelect]);
  
  return (
    <Card 
      className={classNames('cluster-card', className, {
        'cluster-card--expanded': expanded,
        'cluster-card--healthy': healthStatus === 'healthy'
      })}
      onClick={handleClick}
    >
      <CardHeader>
        <ClusterIcon provider={cluster.provider} />
        <h3>{cluster.name}</h3>
        <HealthBadge score={healthScore} />
      </CardHeader>
      
      {expanded && (
        <CardBody>
          <ClusterMetrics cluster={cluster} />
        </CardBody>
      )}
    </Card>
  );
};

// 2. Custom hooks pattern
export const useClusterHealth = (clusterId: string) => {
  const [healthScore, setHealthScore] = useState<number | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    let cancelled = false;
    
    const fetchHealth = async () => {
      try {
        setLoading(true);
        const score = await calculateClusterHealth(clusterId);
        
        if (!cancelled) {
          setHealthScore(score);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err as Error);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    };
    
    fetchHealth();
    
    return () => {
      cancelled = true;
    };
  }, [clusterId]);
  
  return { healthScore, loading, error };
};
```

### NRQL Query Patterns

```typescript
// src/common/queries/kafka-queries.ts

export class KafkaQueryBuilder {
  // Use builder pattern for complex queries
  private query: string[] = [];
  private facets: string[] = [];
  private filters: string[] = [];
  
  select(...fields: string[]): this {
    this.query.push(`SELECT ${fields.join(', ')}`);
    return this;
  }
  
  from(...sources: string[]): this {
    this.query.push(`FROM ${sources.join(', ')}`);
    return this;
  }
  
  where(condition: string): this {
    this.filters.push(condition);
    return this;
  }
  
  facet(...fields: string[]): this {
    this.facets = fields;
    return this;
  }
  
  build(): string {
    const parts = [...this.query];
    
    if (this.filters.length > 0) {
      parts.push(`WHERE ${this.filters.join(' AND ')}`);
    }
    
    if (this.facets.length > 0) {
      parts.push(`FACET ${this.facets.join(', ')}`);
    }
    
    return parts.join(' ');
  }
}

// Query templates with type safety
export const queries = {
  clusterHealth: (clusterId: string, timeRange: string = '1 hour ago') => 
    new KafkaQueryBuilder()
      .select(
        'latest(offlinePartitions) as offlinePartitions',
        'latest(underReplicatedPartitions) as underReplicated',
        'latest(activeControllers) as controllers'
      )
      .from('KafkaClusterSample')
      .where(`clusterId = '${clusterId}'`)
      .where(`SINCE ${timeRange}`)
      .build(),
  
  topicThroughput: (topicName: string) =>
    new KafkaQueryBuilder()
      .select(
        'average(bytesInPerSec) as bytesIn',
        'average(bytesOutPerSec) as bytesOut'
      )
      .from('KafkaTopicSample')
      .where(`topicName = '${topicName}'`)
      .where('SINCE 1 hour ago')
      .facet('TIMESERIES 5 minutes')
      .build()
};
```

## Testing

### Unit Testing

```typescript
// src/common/utils/__tests__/health-calculator.test.ts
import { calculateClusterHealth } from '../health-calculator';
import { mockCluster } from '../../__mocks__/kafka-entities';

describe('Health Calculator', () => {
  describe('calculateClusterHealth', () => {
    it('should return 100 for perfectly healthy cluster', () => {
      const cluster = mockCluster({
        offlinePartitions: 0,
        underReplicatedPartitions: 0,
        activeControllers: 1
      });
      
      const health = calculateClusterHealth(cluster);
      
      expect(health.score).toBe(100);
      expect(health.status).toBe('excellent');
      expect(health.issues).toHaveLength(0);
    });
    
    it('should deduct points for offline partitions', () => {
      const cluster = mockCluster({
        offlinePartitions: 5,
        totalPartitions: 100
      });
      
      const health = calculateClusterHealth(cluster);
      
      expect(health.score).toBeLessThan(100);
      expect(health.issues).toContainEqual(
        expect.objectContaining({
          type: 'offline_partitions',
          severity: 'critical'
        })
      );
    });
    
    it('should handle edge cases gracefully', () => {
      const cluster = mockCluster({
        offlinePartitions: -1, // Invalid data
        activeControllers: 0
      });
      
      const health = calculateClusterHealth(cluster);
      
      expect(health.score).toBeGreaterThanOrEqual(0);
      expect(health.score).toBeLessThanOrEqual(100);
    });
  });
});
```

### Integration Testing

```typescript
// tests/integration/kafka-integration.test.ts
import { KafkaIntegrationService } from '../../src/services/kafka-integration';
import { TestKafkaCluster } from '../fixtures/test-kafka-cluster';

describe('Kafka Integration', () => {
  let testCluster: TestKafkaCluster;
  let integration: KafkaIntegrationService;
  
  beforeAll(async () => {
    // Setup test Kafka cluster
    testCluster = await TestKafkaCluster.create({
      brokers: 3,
      topics: ['test-topic-1', 'test-topic-2']
    });
    
    integration = new KafkaIntegrationService({
      endpoint: testCluster.endpoint,
      credentials: testCluster.credentials
    });
  });
  
  afterAll(async () => {
    await testCluster.destroy();
  });
  
  it('should discover cluster entities', async () => {
    const entities = await integration.discoverEntities();
    
    expect(entities.clusters).toHaveLength(1);
    expect(entities.brokers).toHaveLength(3);
    expect(entities.topics).toHaveLength(2);
  });
  
  it('should collect real-time metrics', async () => {
    // Produce test messages
    await testCluster.produceMessages('test-topic-1', 100);
    
    // Wait for metrics to be available
    await new Promise(resolve => setTimeout(resolve, 5000));
    
    // Collect metrics
    const metrics = await integration.collectMetrics();
    
    expect(metrics.throughput).toBeGreaterThan(0);
    expect(metrics.messageCount).toBeGreaterThanOrEqual(100);
  });
});
```

### Component Testing

```typescript
// src/common/components/__tests__/KafkaClusterCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { KafkaClusterCard } from '../KafkaClusterCard';
import { mockCluster } from '../../__mocks__/kafka-entities';

describe('KafkaClusterCard', () => {
  const defaultProps = {
    cluster: mockCluster(),
    onSelect: jest.fn()
  };
  
  beforeEach(() => {
    jest.clearAllMocks();
  });
  
  it('should render cluster information', () => {
    render(<KafkaClusterCard {...defaultProps} />);
    
    expect(screen.getByText(defaultProps.cluster.name)).toBeInTheDocument();
    expect(screen.getByRole('img', { name: /kafka logo/i })).toBeInTheDocument();
  });
  
  it('should display health status', async () => {
    render(<KafkaClusterCard {...defaultProps} />);
    
    // Wait for health calculation
    const healthBadge = await screen.findByTestId('health-badge');
    
    expect(healthBadge).toHaveTextContent(/health:/i);
    expect(healthBadge).toHaveClass('health-badge--healthy');
  });
  
  it('should call onSelect when clicked', () => {
    render(<KafkaClusterCard {...defaultProps} />);
    
    fireEvent.click(screen.getByRole('article'));
    
    expect(defaultProps.onSelect).toHaveBeenCalledWith(defaultProps.cluster);
  });
  
  it('should expand to show metrics on click', async () => {
    render(<KafkaClusterCard {...defaultProps} />);
    
    // Initially metrics are not visible
    expect(screen.queryByTestId('cluster-metrics')).not.toBeInTheDocument();
    
    // Click to expand
    fireEvent.click(screen.getByRole('button', { name: /expand/i }));
    
    // Metrics should now be visible
    expect(screen.getByTestId('cluster-metrics')).toBeInTheDocument();
  });
});
```

## API Development

### GraphQL Schema

```graphql
# src/graphql/schema.graphql

type Query {
  # Kafka entity queries
  kafkaClusters(filter: ClusterFilter): [KafkaCluster!]!
  kafkaCluster(id: ID!): KafkaCluster
  kafkaTopics(clusterId: ID!, filter: TopicFilter): [KafkaTopic!]!
  
  # Metrics queries
  clusterMetrics(
    clusterId: ID!
    timeRange: TimeRange!
  ): ClusterMetrics!
  
  # Health queries
  clusterHealth(clusterId: ID!): HealthScore!
}

type KafkaCluster {
  id: ID!
  name: String!
  provider: KafkaProvider!
  region: String!
  status: ClusterStatus!
  brokers: [KafkaBroker!]!
  topics: [KafkaTopic!]!
  metrics: ClusterMetrics
  health: HealthScore
}

type ClusterMetrics {
  throughput: ThroughputMetrics!
  storage: StorageMetrics!
  latency: LatencyMetrics!
  availability: AvailabilityMetrics!
}

input ClusterFilter {
  provider: KafkaProvider
  region: String
  status: ClusterStatus
  healthScore: IntRange
}

enum KafkaProvider {
  AWS_MSK
  CONFLUENT_CLOUD
}
```

### REST API Endpoints

```typescript
// src/api/routes/kafka-routes.ts
import { Router } from 'express';
import { KafkaController } from '../controllers/kafka-controller';

export const kafkaRoutes = (controller: KafkaController): Router => {
  const router = Router();
  
  // Cluster endpoints
  router.get('/clusters', controller.listClusters);
  router.get('/clusters/:id', controller.getCluster);
  router.get('/clusters/:id/health', controller.getClusterHealth);
  router.get('/clusters/:id/metrics', controller.getClusterMetrics);
  
  // Topic endpoints
  router.get('/clusters/:clusterId/topics', controller.listTopics);
  router.get('/topics/:id', controller.getTopic);
  router.get('/topics/:id/metrics', controller.getTopicMetrics);
  
  // Consumer group endpoints
  router.get('/consumer-groups', controller.listConsumerGroups);
  router.get('/consumer-groups/:id/lag', controller.getConsumerLag);
  
  return router;
};

// Controller implementation
export class KafkaController {
  constructor(private kafkaService: KafkaService) {}
  
  listClusters = async (req: Request, res: Response) => {
    try {
      const { provider, region, status } = req.query;
      
      const clusters = await this.kafkaService.listClusters({
        provider: provider as KafkaProvider,
        region: region as string,
        status: status as ClusterStatus
      });
      
      res.json({
        data: clusters,
        count: clusters.length,
        timestamp: Date.now()
      });
    } catch (error) {
      res.status(500).json({
        error: error.message,
        code: 'CLUSTER_LIST_ERROR'
      });
    }
  };
}
```

## Performance Optimization

### React Performance

```typescript
// 1. Use React.memo for expensive components
export const ExpensiveChart = React.memo<ChartProps>(({ data, config }) => {
  return <ComplexVisualization data={data} config={config} />;
}, (prevProps, nextProps) => {
  // Custom comparison function
  return (
    prevProps.data.length === nextProps.data.length &&
    prevProps.config.type === nextProps.config.type
  );
});

// 2. Virtualize large lists
import { VariableSizeList } from 'react-window';

export const VirtualizedTopicList: React.FC<{ topics: Topic[] }> = ({ topics }) => {
  const getItemSize = (index: number) => {
    // Variable heights based on content
    return topics[index].expanded ? 200 : 80;
  };
  
  return (
    <VariableSizeList
      height={600}
      itemCount={topics.length}
      itemSize={getItemSize}
      width="100%"
    >
      {({ index, style }) => (
        <TopicRow
          topic={topics[index]}
          style={style}
        />
      )}
    </VariableSizeList>
  );
};

// 3. Optimize context usage
const KafkaDataContext = React.createContext<KafkaData | null>(null);

export const KafkaDataProvider: React.FC = ({ children }) => {
  const data = useKafkaData();
  
  // Split context to avoid unnecessary re-renders
  return (
    <KafkaDataContext.Provider value={data}>
      <KafkaUIContext.Provider value={data.ui}>
        {children}
      </KafkaUIContext.Provider>
    </KafkaDataContext.Provider>
  );
};
```

### Query Optimization

```typescript
// Implement query caching
class QueryCache {
  private cache = new Map<string, CacheEntry>();
  
  async execute<T>(
    query: string,
    options: QueryOptions = {}
  ): Promise<T> {
    const cacheKey = this.getCacheKey(query, options);
    
    // Check cache
    const cached = this.cache.get(cacheKey);
    if (cached && !this.isExpired(cached)) {
      return cached.data as T;
    }
    
    // Execute query
    const result = await this.runQuery<T>(query, options);
    
    // Cache result
    this.cache.set(cacheKey, {
      data: result,
      timestamp: Date.now(),
      ttl: options.cacheTTL || 300000 // 5 minutes default
    });
    
    // Cleanup old entries
    this.cleanup();
    
    return result;
  }
  
  private cleanup(): void {
    const now = Date.now();
    
    for (const [key, entry] of this.cache.entries()) {
      if (now - entry.timestamp > entry.ttl) {
        this.cache.delete(key);
      }
    }
  }
}
```

## Debugging

### Debug Configuration

```typescript
// src/common/debug/debug-config.ts
export const debugConfig = {
  // Enable debug mode
  enabled: process.env.NODE_ENV === 'development',
  
  // Debug levels
  levels: {
    error: true,
    warn: true,
    info: true,
    debug: true,
    trace: false
  },
  
  // Component-specific debugging
  components: {
    queries: true,
    health: true,
    integration: true,
    ui: false
  },
  
  // Performance monitoring
  performance: {
    logSlowQueries: true,
    slowQueryThreshold: 1000, // ms
    logRenderTime: true,
    slowRenderThreshold: 100 // ms
  }
};

// Debug logger
export class DebugLogger {
  constructor(private component: string) {}
  
  log(level: string, message: string, data?: any): void {
    if (!debugConfig.enabled) return;
    if (!debugConfig.levels[level]) return;
    if (!debugConfig.components[this.component]) return;
    
    const timestamp = new Date().toISOString();
    const prefix = `[${timestamp}] [${this.component}] [${level.toUpperCase()}]`;
    
    console.log(`${prefix} ${message}`, data || '');
    
    // Send to debug panel
    if (window.__KAFKA_DEBUG_PANEL__) {
      window.__KAFKA_DEBUG_PANEL__.addLog({
        timestamp,
        component: this.component,
        level,
        message,
        data
      });
    }
  }
  
  time(label: string): void {
    if (debugConfig.performance.logRenderTime) {
      console.time(`${this.component}:${label}`);
    }
  }
  
  timeEnd(label: string): void {
    if (debugConfig.performance.logRenderTime) {
      console.timeEnd(`${this.component}:${label}`);
    }
  }
}
```

### Debug Panel

```typescript
// src/common/components/DebugPanel/DebugPanel.tsx
export const DebugPanel: React.FC = () => {
  const [logs, setLogs] = useState<DebugLog[]>([]);
  const [filter, setFilter] = useState<DebugFilter>({});
  
  useEffect(() => {
    // Register global debug panel
    window.__KAFKA_DEBUG_PANEL__ = {
      addLog: (log) => setLogs(prev => [...prev, log].slice(-1000))
    };
    
    return () => {
      delete window.__KAFKA_DEBUG_PANEL__;
    };
  }, []);
  
  const filteredLogs = useMemo(() => 
    logs.filter(log => {
      if (filter.component && log.component !== filter.component) return false;
      if (filter.level && log.level !== filter.level) return false;
      if (filter.search && !log.message.includes(filter.search)) return false;
      return true;
    }),
    [logs, filter]
  );
  
  return (
    <div className="debug-panel">
      <div className="debug-panel__header">
        <h3>Debug Console</h3>
        <DebugFilters value={filter} onChange={setFilter} />
      </div>
      
      <div className="debug-panel__logs">
        <VirtualizedList
          items={filteredLogs}
          renderItem={(log) => <DebugLogEntry log={log} />}
        />
      </div>
      
      <div className="debug-panel__actions">
        <button onClick={() => setLogs([])}>Clear</button>
        <button onClick={() => downloadLogs(logs)}>Export</button>
      </div>
    </div>
  );
};
```

## Deployment

### Build Process

```json
// package.json scripts
{
  "scripts": {
    "start": "nr1 nerdpack:serve",
    "build": "nr1 nerdpack:build",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx",
    "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx --fix",
    "format": "prettier --write \"src/**/*.{js,jsx,ts,tsx,json,css,scss}\"",
    "validate": "nr1 nerdpack:validate",
    "publish": "nr1 nerdpack:publish",
    "deploy": "nr1 nerdpack:deploy",
    "clean": "rm -rf dist tmp",
    "prepare": "husky install"
  }
}
```

### CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linter
        run: npm run lint
      
      - name: Run tests
        run: npm run test:coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
      
      - name: Build nerdpack
        run: npm run build
      
      - name: Validate nerdpack
        run: npm run validate

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Configure NR1 CLI
        run: |
          nr1 profiles:add --name prod --api-key ${{ secrets.NR_API_KEY }}
          nr1 profiles:default --name prod
      
      - name: Publish nerdpack
        run: nr1 nerdpack:publish
      
      - name: Deploy nerdpack
        run: nr1 nerdpack:deploy --channel STABLE
      
      - name: Tag release
        run: |
          VERSION=$(node -p "require('./package.json').version")
          nr1 nerdpack:tag --tag v$VERSION
```

## Contributing

### Contribution Guidelines

1. **Fork and Clone**: Fork the repository and clone locally
2. **Branch**: Create a feature branch from `develop`
3. **Commit**: Follow conventional commit format
4. **Test**: Add tests for new functionality
5. **Document**: Update documentation as needed
6. **Pull Request**: Submit PR with clear description

### Code Review Checklist

- [ ] Code follows TypeScript and React best practices
- [ ] All tests pass
- [ ] Coverage maintained or improved
- [ ] Documentation updated
- [ ] No console.logs or debugging code
- [ ] Performance implications considered
- [ ] Security best practices followed
- [ ] Accessibility requirements met

## Resources

### Documentation
- [New Relic One SDK Documentation](https://developer.newrelic.com)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [React Documentation](https://reactjs.org/docs)
- [Kafka Documentation](https://kafka.apache.org/documentation/)

### Tools
- [NR1 CLI Reference](https://developer.newrelic.com/explore-docs/nr1-cli)
- [NRQL Reference](https://docs.newrelic.com/docs/query-data/nrql-new-relic-query-language)
- [GraphQL Explorer](https://api.newrelic.com/graphiql)

## Conclusion

This development guide provides the foundation for contributing to the Message Queues monitoring system. By following these guidelines and best practices, developers can create high-quality, maintainable code that enhances Kafka monitoring capabilities.