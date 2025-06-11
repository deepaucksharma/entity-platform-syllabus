# Testing Strategy

## Overview

This document outlines the comprehensive testing strategy for the Message Queues monitoring system. It covers testing approaches, methodologies, tools, and best practices to ensure high-quality, reliable software delivery.

## Testing Architecture

### Testing Pyramid

```
Testing Pyramid
├── Unit Tests (60%)
│   ├── Component Logic
│   ├── Utility Functions
│   ├── Query Builders
│   └── Data Transformations
├── Integration Tests (25%)
│   ├── API Integration
│   ├── Database Queries
│   ├── Service Communication
│   └── External Dependencies
├── E2E Tests (10%)
│   ├── User Workflows
│   ├── Critical Paths
│   └── Cross-browser Testing
└── Other Tests (5%)
    ├── Performance Tests
    ├── Security Tests
    └── Accessibility Tests
```

## Unit Testing

### Test Configuration

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/tests/setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '\\.(css|scss)$': 'identity-obj-proxy',
    '^nr1': '<rootDir>/tests/__mocks__/nr1.ts'
  },
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
    '!src/**/__tests__/**'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  testMatch: [
    '**/__tests__/**/*.test.{ts,tsx}',
    '**/*.spec.{ts,tsx}'
  ]
};
```

### Unit Test Patterns

#### Component Testing

```typescript
// src/components/__tests__/HealthIndicator.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { HealthIndicator } from '../HealthIndicator';

describe('HealthIndicator', () => {
  const defaultProps = {
    score: 85,
    entity: 'test-cluster',
    onHealthClick: jest.fn()
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('Rendering', () => {
    it('should render health score correctly', () => {
      render(<HealthIndicator {...defaultProps} />);
      
      expect(screen.getByText('85')).toBeInTheDocument();
      expect(screen.getByLabelText('Health score: 85')).toBeInTheDocument();
    });

    it('should apply correct status class based on score', () => {
      const { rerender } = render(<HealthIndicator {...defaultProps} />);
      
      expect(screen.getByTestId('health-indicator')).toHaveClass('health-indicator--good');
      
      rerender(<HealthIndicator {...defaultProps} score={45} />);
      expect(screen.getByTestId('health-indicator')).toHaveClass('health-indicator--poor');
      
      rerender(<HealthIndicator {...defaultProps} score={95} />);
      expect(screen.getByTestId('health-indicator')).toHaveClass('health-indicator--excellent');
    });

    it('should render loading state', () => {
      render(<HealthIndicator {...defaultProps} score={null} loading />);
      
      expect(screen.getByTestId('health-skeleton')).toBeInTheDocument();
      expect(screen.queryByText('85')).not.toBeInTheDocument();
    });
  });

  describe('Interactions', () => {
    it('should call onHealthClick when clicked', async () => {
      const user = userEvent.setup();
      render(<HealthIndicator {...defaultProps} />);
      
      await user.click(screen.getByTestId('health-indicator'));
      
      expect(defaultProps.onHealthClick).toHaveBeenCalledWith(85, 'test-cluster');
    });

    it('should show tooltip on hover', async () => {
      const user = userEvent.setup();
      render(<HealthIndicator {...defaultProps} />);
      
      await user.hover(screen.getByTestId('health-indicator'));
      
      expect(await screen.findByRole('tooltip')).toHaveTextContent(
        'Click to view health details for test-cluster'
      );
    });

    it('should be keyboard accessible', async () => {
      const user = userEvent.setup();
      render(<HealthIndicator {...defaultProps} />);
      
      await user.tab();
      expect(screen.getByTestId('health-indicator')).toHaveFocus();
      
      await user.keyboard('{Enter}');
      expect(defaultProps.onHealthClick).toHaveBeenCalled();
    });
  });
});
```

#### Hook Testing

```typescript
// src/hooks/__tests__/useKafkaHealth.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useKafkaHealth } from '../useKafkaHealth';
import { mockNrqlQuery } from '../../__mocks__/nr1';

describe('useKafkaHealth', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should fetch health data successfully', async () => {
    const mockData = {
      results: [{
        healthScore: 92,
        offlinePartitions: 0,
        underReplicated: 2
      }]
    };
    
    mockNrqlQuery.mockResolvedValueOnce({ data: mockData });
    
    const { result } = renderHook(() => 
      useKafkaHealth('cluster-123', { pollInterval: 0 })
    );
    
    // Initial state
    expect(result.current.loading).toBe(true);
    expect(result.current.health).toBeNull();
    expect(result.current.error).toBeNull();
    
    // After data loads
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });
    
    expect(result.current.health).toEqual({
      score: 92,
      status: 'excellent',
      issues: []
    });
    expect(mockNrqlQuery).toHaveBeenCalledWith(
      expect.stringContaining('cluster-123')
    );
  });

  it('should handle errors gracefully', async () => {
    const mockError = new Error('Query failed');
    mockNrqlQuery.mockRejectedValueOnce(mockError);
    
    const { result } = renderHook(() => useKafkaHealth('cluster-123'));
    
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });
    
    expect(result.current.error).toEqual(mockError);
    expect(result.current.health).toBeNull();
  });

  it('should refetch data on interval', async () => {
    const mockData = { results: [{ healthScore: 85 }] };
    mockNrqlQuery.mockResolvedValue({ data: mockData });
    
    const { result } = renderHook(() => 
      useKafkaHealth('cluster-123', { pollInterval: 100 })
    );
    
    await waitFor(() => {
      expect(result.current.loading).toBe(false);
    });
    
    expect(mockNrqlQuery).toHaveBeenCalledTimes(1);
    
    // Wait for next poll
    await waitFor(() => {
      expect(mockNrqlQuery).toHaveBeenCalledTimes(2);
    }, { timeout: 200 });
  });
});
```

#### Utility Function Testing

```typescript
// src/utils/__tests__/health-calculator.test.ts
import { 
  calculateClusterHealth, 
  calculateBrokerHealth,
  combineHealthScores 
} from '../health-calculator';

describe('Health Calculator', () => {
  describe('calculateClusterHealth', () => {
    it('should calculate perfect health score', () => {
      const metrics = {
        offlinePartitions: 0,
        underReplicatedPartitions: 0,
        activeControllers: 1,
        brokerCount: 3
      };
      
      const result = calculateClusterHealth(metrics);
      
      expect(result).toEqual({
        score: 100,
        status: 'excellent',
        factors: {
          availability: 100,
          reliability: 100,
          performance: 100
        },
        issues: []
      });
    });

    it('should deduct for offline partitions', () => {
      const metrics = {
        offlinePartitions: 5,
        totalPartitions: 100,
        underReplicatedPartitions: 0,
        activeControllers: 1
      };
      
      const result = calculateClusterHealth(metrics);
      
      expect(result.score).toBeLessThan(100);
      expect(result.issues).toContainEqual({
        type: 'offline_partitions',
        severity: 'critical',
        impact: expect.any(Number),
        description: '5 partitions are offline'
      });
    });

    it('should handle edge cases', () => {
      const invalidMetrics = {
        offlinePartitions: -1,
        underReplicatedPartitions: null,
        activeControllers: undefined
      };
      
      const result = calculateClusterHealth(invalidMetrics as any);
      
      expect(result.score).toBeGreaterThanOrEqual(0);
      expect(result.score).toBeLessThanOrEqual(100);
      expect(result.status).toBeDefined();
    });
  });

  describe('combineHealthScores', () => {
    it('should calculate weighted average correctly', () => {
      const scores = [
        { score: 100, weight: 0.5 },
        { score: 80, weight: 0.3 },
        { score: 60, weight: 0.2 }
      ];
      
      const result = combineHealthScores(scores);
      
      expect(result).toBe(86); // (100*0.5 + 80*0.3 + 60*0.2)
    });

    it('should normalize weights if they dont sum to 1', () => {
      const scores = [
        { score: 100, weight: 2 },
        { score: 50, weight: 1 }
      ];
      
      const result = combineHealthScores(scores);
      
      expect(result).toBeCloseTo(83.33, 2);
    });
  });
});
```

## Integration Testing

### API Integration Tests

```typescript
// tests/integration/kafka-api.test.ts
import { KafkaAPIClient } from '../../src/services/kafka-api-client';
import { MockKafkaServer } from '../mocks/mock-kafka-server';

describe('Kafka API Integration', () => {
  let mockServer: MockKafkaServer;
  let apiClient: KafkaAPIClient;

  beforeAll(async () => {
    mockServer = new MockKafkaServer();
    await mockServer.start();
    
    apiClient = new KafkaAPIClient({
      baseUrl: mockServer.url,
      apiKey: 'test-key'
    });
  });

  afterAll(async () => {
    await mockServer.stop();
  });

  describe('Cluster Operations', () => {
    it('should list clusters successfully', async () => {
      mockServer.addHandler({
        method: 'GET',
        path: '/api/clusters',
        response: {
          clusters: [
            { id: '1', name: 'prod-cluster', status: 'healthy' },
            { id: '2', name: 'dev-cluster', status: 'healthy' }
          ]
        }
      });

      const clusters = await apiClient.listClusters();

      expect(clusters).toHaveLength(2);
      expect(clusters[0]).toMatchObject({
        id: '1',
        name: 'prod-cluster',
        status: 'healthy'
      });
    });

    it('should handle pagination correctly', async () => {
      mockServer.addHandler({
        method: 'GET',
        path: '/api/clusters',
        query: { page: '1', limit: '10' },
        response: {
          clusters: Array(10).fill(null).map((_, i) => ({
            id: `${i}`,
            name: `cluster-${i}`
          })),
          pagination: {
            page: 1,
            limit: 10,
            total: 25
          }
        }
      });

      const result = await apiClient.listClusters({ page: 1, limit: 10 });

      expect(result.clusters).toHaveLength(10);
      expect(result.pagination).toEqual({
        page: 1,
        limit: 10,
        total: 25
      });
    });

    it('should retry on transient failures', async () => {
      let attempts = 0;
      mockServer.addHandler({
        method: 'GET',
        path: '/api/clusters/:id',
        handler: (req, res) => {
          attempts++;
          if (attempts < 3) {
            res.status(503).json({ error: 'Service unavailable' });
          } else {
            res.json({ id: req.params.id, name: 'test-cluster' });
          }
        }
      });

      const cluster = await apiClient.getCluster('123');

      expect(attempts).toBe(3);
      expect(cluster).toMatchObject({
        id: '123',
        name: 'test-cluster'
      });
    });
  });

  describe('Metrics Collection', () => {
    it('should aggregate metrics correctly', async () => {
      mockServer.addHandler({
        method: 'POST',
        path: '/api/metrics/query',
        response: {
          results: [
            { timestamp: 1000, value: 100 },
            { timestamp: 2000, value: 200 },
            { timestamp: 3000, value: 150 }
          ]
        }
      });

      const metrics = await apiClient.queryMetrics({
        metric: 'bytesInPerSec',
        aggregation: 'average',
        timeRange: { start: 1000, end: 3000 }
      });

      expect(metrics.results).toHaveLength(3);
      expect(metrics.results[1].value).toBe(200);
    });
  });
});
```

### Database Integration Tests

```typescript
// tests/integration/nrdb-queries.test.ts
import { NRDBClient } from '../../src/services/nrdb-client';
import { testAccountId } from '../config/test-config';

describe('NRDB Query Integration', () => {
  let nrdbClient: NRDBClient;

  beforeAll(() => {
    nrdbClient = new NRDBClient({
      accountId: testAccountId,
      apiKey: process.env.TEST_NR_API_KEY
    });
  });

  describe('Entity Queries', () => {
    it('should fetch kafka entities', async () => {
      const query = `
        FROM KafkaClusterSample
        SELECT uniqueCount(entityGuid)
        WHERE provider IS NOT NULL
        SINCE 1 hour ago
      `;

      const result = await nrdbClient.query(query);

      expect(result).toBeDefined();
      expect(result.results).toBeInstanceOf(Array);
      expect(result.metadata).toHaveProperty('executionTime');
    });

    it('should handle complex aggregations', async () => {
      const query = `
        FROM KafkaBrokerSample
        SELECT 
          average(cpuUsage) as avgCpu,
          max(cpuUsage) as maxCpu,
          percentile(cpuUsage, 95) as p95Cpu
        WHERE clusterName IS NOT NULL
        FACET clusterName
        SINCE 1 day ago
        LIMIT 10
      `;

      const result = await nrdbClient.query(query);

      expect(result.facets).toBeDefined();
      result.facets.forEach(facet => {
        expect(facet).toHaveProperty('name');
        expect(facet.results[0]).toHaveProperty('avgCpu');
        expect(facet.results[0]).toHaveProperty('maxCpu');
        expect(facet.results[0]).toHaveProperty('p95Cpu');
      });
    });
  });

  describe('Time Series Queries', () => {
    it('should return time series data', async () => {
      const query = `
        FROM KafkaTopicSample
        SELECT rate(sum(bytesInPerSec), 1 minute)
        WHERE topicName = 'test-topic'
        SINCE 1 hour ago
        TIMESERIES 5 minutes
      `;

      const result = await nrdbClient.query(query);

      expect(result.timeSeries).toBeDefined();
      expect(result.timeSeries.length).toBeGreaterThan(0);
      
      result.timeSeries.forEach(point => {
        expect(point).toHaveProperty('beginTimeSeconds');
        expect(point).toHaveProperty('endTimeSeconds');
        expect(point.results[0]).toHaveProperty('result');
      });
    });
  });
});
```

## End-to-End Testing

### E2E Test Setup

```typescript
// e2e/setup/test-environment.ts
import { chromium, Browser, Page } from 'playwright';
import { TestDataGenerator } from './test-data-generator';

export class E2ETestEnvironment {
  private browser: Browser;
  private page: Page;
  private testData: TestDataGenerator;

  async setup(): Promise<void> {
    // Launch browser
    this.browser = await chromium.launch({
      headless: process.env.HEADLESS !== 'false'
    });

    // Create page with New Relic session
    const context = await this.browser.newContext({
      storageState: './e2e/auth/nr-session.json'
    });
    this.page = await context.newPage();

    // Setup test data
    this.testData = new TestDataGenerator();
    await this.testData.seedTestData();
  }

  async teardown(): Promise<void> {
    await this.testData.cleanup();
    await this.browser.close();
  }

  getPage(): Page {
    return this.page;
  }

  getTestData(): TestDataGenerator {
    return this.testData;
  }
}
```

### E2E Test Scenarios

```typescript
// e2e/tests/kafka-monitoring-flow.e2e.ts
import { test, expect } from '@playwright/test';
import { E2ETestEnvironment } from '../setup/test-environment';

let env: E2ETestEnvironment;

test.beforeAll(async () => {
  env = new E2ETestEnvironment();
  await env.setup();
});

test.afterAll(async () => {
  await env.teardown();
});

test.describe('Kafka Monitoring User Flow', () => {
  test('should display cluster health dashboard', async () => {
    const page = env.getPage();
    
    // Navigate to home dashboard
    await page.goto('/launcher/message-queues.home');
    
    // Wait for data to load
    await page.waitForSelector('[data-testid="cluster-table"]');
    
    // Verify clusters are displayed
    const clusterRows = await page.$$('[data-testid="cluster-row"]');
    expect(clusterRows.length).toBeGreaterThan(0);
    
    // Check health indicators
    const healthIndicator = await page.$('[data-testid="health-indicator"]');
    const healthScore = await healthIndicator?.textContent();
    expect(Number(healthScore)).toBeGreaterThanOrEqual(0);
    expect(Number(healthScore)).toBeLessThanOrEqual(100);
  });

  test('should navigate to cluster details', async () => {
    const page = env.getPage();
    
    // Click on first cluster
    await page.click('[data-testid="cluster-row"]:first-child');
    
    // Wait for navigation
    await page.waitForURL(/.*summary/);
    
    // Verify summary page loaded
    await expect(page.locator('h1')).toContainText('Cluster Summary');
    
    // Check for key metrics
    await expect(page.locator('[data-testid="total-topics"]')).toBeVisible();
    await expect(page.locator('[data-testid="total-brokers"]')).toBeVisible();
    await expect(page.locator('[data-testid="throughput-chart"]')).toBeVisible();
  });

  test('should filter clusters by provider', async () => {
    const page = env.getPage();
    
    // Go back to home
    await page.goto('/launcher/message-queues.home');
    
    // Open provider filter
    await page.click('[data-testid="provider-filter"]');
    
    // Select AWS MSK only
    await page.click('text=AWS MSK');
    
    // Wait for table to update
    await page.waitForTimeout(1000);
    
    // Verify only AWS clusters shown
    const providerLogos = await page.$$('[data-testid="provider-logo"]');
    for (const logo of providerLogos) {
      const alt = await logo.getAttribute('alt');
      expect(alt).toContain('AWS');
    }
  });

  test('should handle consumer lag alerts', async () => {
    const page = env.getPage();
    const testData = env.getTestData();
    
    // Create high lag scenario
    await testData.createHighLagScenario('test-consumer-group');
    
    // Navigate to consumer monitoring
    await page.goto('/launcher/message-queues.home');
    await page.click('text=Consumer Groups');
    
    // Find high lag consumer
    const lagIndicator = await page.$('[data-testid="lag-critical"]');
    expect(lagIndicator).toBeTruthy();
    
    // Click for details
    await lagIndicator?.click();
    
    // Verify lag details modal
    await expect(page.locator('[data-testid="lag-details-modal"]')).toBeVisible();
    await expect(page.locator('text=Consumer Lag Critical')).toBeVisible();
  });
});
```

### Visual Regression Testing

```typescript
// e2e/tests/visual-regression.e2e.ts
import { test, expect } from '@playwright/test';

test.describe('Visual Regression Tests', () => {
  test('home dashboard screenshot', async ({ page }) => {
    await page.goto('/launcher/message-queues.home');
    await page.waitForLoadState('networkidle');
    
    // Take screenshot
    await expect(page).toHaveScreenshot('home-dashboard.png', {
      fullPage: true,
      animations: 'disabled'
    });
  });

  test('honeycomb visualization', async ({ page }) => {
    await page.goto('/launcher/message-queues.summary');
    await page.waitForSelector('[data-testid="honeycomb-view"]');
    
    // Hover over hexagon to show tooltip
    await page.hover('[data-testid="hexagon"]:first-child');
    await page.waitForSelector('[role="tooltip"]');
    
    await expect(page.locator('[data-testid="honeycomb-view"]'))
      .toHaveScreenshot('honeycomb-with-tooltip.png');
  });

  test('dark mode compatibility', async ({ page }) => {
    // Enable dark mode
    await page.emulateMedia({ colorScheme: 'dark' });
    
    await page.goto('/launcher/message-queues.home');
    await page.waitForLoadState('networkidle');
    
    await expect(page).toHaveScreenshot('home-dashboard-dark.png', {
      fullPage: true
    });
  });
});
```

## Performance Testing

### Load Testing

```typescript
// tests/performance/load-test.ts
import { check, sleep } from 'k6';
import http from 'k6/http';
import { Rate } from 'k6/metrics';

export const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // Ramp up to 10 users
    { duration: '5m', target: 50 },   // Stay at 50 users
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 0 },    // Ramp down to 0 users
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'], // 95% of requests under 2s
    errors: ['rate<0.1'],              // Error rate under 10%
  },
};

export default function () {
  const BASE_URL = __ENV.BASE_URL || 'https://api.newrelic.com';
  
  // Test home dashboard query
  const homeQuery = `
    query KafkaClusters {
      actor {
        entitySearch(query: "type IN ('AWSMSKCLUSTER', 'CONFLUENTCLOUDCLUSTER')") {
          results {
            entities {
              guid
              name
              alertSeverity
            }
          }
        }
      }
    }
  `;
  
  const response = http.post(
    `${BASE_URL}/graphql`,
    JSON.stringify({ query: homeQuery }),
    {
      headers: {
        'Content-Type': 'application/json',
        'API-Key': __ENV.API_KEY,
      },
    }
  );
  
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response has data': (r) => JSON.parse(r.body).data !== null,
    'no errors': (r) => !JSON.parse(r.body).errors,
  });
  
  errorRate.add(response.status !== 200);
  
  sleep(1);
}
```

### Query Performance Testing

```typescript
// tests/performance/query-performance.test.ts
import { measureQueryPerformance } from '../../src/utils/performance';

describe('Query Performance Tests', () => {
  const queries = [
    {
      name: 'Cluster Health Query',
      query: `
        SELECT latest(healthScore)
        FROM KafkaClusterSample
        WHERE provider IS NOT NULL
        FACET clusterName
        LIMIT 100
      `
    },
    {
      name: 'Topic Throughput Aggregation',
      query: `
        FROM KafkaTopicSample
        SELECT 
          sum(bytesInPerSec) as totalBytesIn,
          sum(bytesOutPerSec) as totalBytesOut
        WHERE clusterName IS NOT NULL
        FACET clusterName, topicName
        SINCE 1 hour ago
        LIMIT 1000
      `
    },
    {
      name: 'Consumer Lag Time Series',
      query: `
        SELECT max(consumerLag)
        FROM KafkaConsumerSample
        WHERE consumerGroup IS NOT NULL
        FACET consumerGroup
        SINCE 24 hours ago
        TIMESERIES 5 minutes
      `
    }
  ];

  queries.forEach(({ name, query }) => {
    it(`${name} should complete within performance threshold`, async () => {
      const results = await measureQueryPerformance(query, {
        iterations: 5,
        warmup: 2
      });

      expect(results.avgDuration).toBeLessThan(2000); // 2 seconds
      expect(results.p95Duration).toBeLessThan(3000); // 3 seconds
      expect(results.errorRate).toBe(0);
      
      console.log(`${name} Performance:`, {
        avg: `${results.avgDuration}ms`,
        p95: `${results.p95Duration}ms`,
        min: `${results.minDuration}ms`,
        max: `${results.maxDuration}ms`
      });
    });
  });
});
```

## Security Testing

### Security Test Suite

```typescript
// tests/security/security.test.ts
describe('Security Tests', () => {
  describe('Authentication', () => {
    it('should reject requests without authentication', async () => {
      const response = await fetch('/api/kafka/clusters', {
        headers: {
          // No auth headers
        }
      });

      expect(response.status).toBe(401);
      expect(await response.json()).toEqual({
        error: 'Authentication required'
      });
    });

    it('should reject invalid API keys', async () => {
      const response = await fetch('/api/kafka/clusters', {
        headers: {
          'X-API-Key': 'invalid-key-12345'
        }
      });

      expect(response.status).toBe(403);
    });

    it('should enforce rate limiting', async () => {
      const requests = Array(150).fill(null).map(() => 
        fetch('/api/kafka/metrics', {
          headers: { 'X-API-Key': process.env.TEST_API_KEY }
        })
      );

      const responses = await Promise.all(requests);
      const rateLimited = responses.filter(r => r.status === 429);

      expect(rateLimited.length).toBeGreaterThan(0);
    });
  });

  describe('Authorization', () => {
    it('should enforce entity-level permissions', async () => {
      const restrictedClusterId = 'prod-critical-cluster';
      
      const response = await authenticatedRequest(
        `/api/kafka/clusters/${restrictedClusterId}`,
        { role: 'viewer' }
      );

      expect(response.status).toBe(403);
      expect(await response.json()).toEqual({
        error: 'Insufficient permissions for this resource'
      });
    });
  });

  describe('Input Validation', () => {
    it('should sanitize NRQL queries', async () => {
      const maliciousQuery = "'; DROP TABLE KafkaClusterSample; --";
      
      const response = await fetch('/api/kafka/query', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': process.env.TEST_API_KEY
        },
        body: JSON.stringify({ query: maliciousQuery })
      });

      expect(response.status).toBe(400);
      expect(await response.json()).toEqual({
        error: 'Invalid query syntax'
      });
    });

    it('should validate request payloads', async () => {
      const invalidPayload = {
        clusterId: '../../../etc/passwd',
        timeRange: 'invalid'
      };

      const response = await fetch('/api/kafka/metrics', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-API-Key': process.env.TEST_API_KEY
        },
        body: JSON.stringify(invalidPayload)
      });

      expect(response.status).toBe(400);
      const error = await response.json();
      expect(error.validationErrors).toBeDefined();
    });
  });
});
```

## Accessibility Testing

### Automated A11y Tests

```typescript
// tests/accessibility/a11y.test.ts
import { injectAxe, checkA11y } from 'axe-playwright';

describe('Accessibility Tests', () => {
  beforeEach(async ({ page }) => {
    await injectAxe(page);
  });

  test('home dashboard should be accessible', async ({ page }) => {
    await page.goto('/launcher/message-queues.home');
    await checkA11y(page, null, {
      detailedReport: true,
      detailedReportOptions: {
        html: true
      }
    });
  });

  test('keyboard navigation should work', async ({ page }) => {
    await page.goto('/launcher/message-queues.home');
    
    // Tab through interactive elements
    await page.keyboard.press('Tab');
    const firstFocused = await page.evaluate(() => 
      document.activeElement?.getAttribute('data-testid')
    );
    expect(firstFocused).toBe('add-account-button');

    // Continue tabbing
    await page.keyboard.press('Tab');
    await page.keyboard.press('Tab');
    
    // Activate filter with keyboard
    await page.keyboard.press('Enter');
    await expect(page.locator('[role="listbox"]')).toBeVisible();
    
    // Navigate options with arrow keys
    await page.keyboard.press('ArrowDown');
    await page.keyboard.press('Enter');
    
    // Verify selection
    await expect(page.locator('[data-testid="selected-provider"]'))
      .toContainText('AWS MSK');
  });

  test('screen reader announcements', async ({ page }) => {
    await page.goto('/launcher/message-queues.summary');
    
    // Check for live regions
    const liveRegions = await page.$$('[aria-live]');
    expect(liveRegions.length).toBeGreaterThan(0);
    
    // Trigger an update
    await page.click('[data-testid="refresh-button"]');
    
    // Check announcement
    const announcement = await page.textContent('[role="status"]');
    expect(announcement).toContain('Data refreshed');
  });
});
```

## Test Data Management

### Test Data Factory

```typescript
// tests/factories/kafka-factory.ts
import { Factory } from 'fishery';
import { faker } from '@faker-js/faker';
import { KafkaCluster, KafkaTopic, KafkaBroker } from '../../src/types';

export const clusterFactory = Factory.define<KafkaCluster>(({ sequence }) => ({
  id: `cluster-${sequence}`,
  name: faker.helpers.arrayElement(['prod', 'staging', 'dev']) + `-kafka-${sequence}`,
  provider: faker.helpers.arrayElement(['AWS_MSK', 'CONFLUENT_CLOUD']),
  region: faker.helpers.arrayElement(['us-east-1', 'us-west-2', 'eu-west-1']),
  status: faker.helpers.weightedArrayElement([
    { weight: 8, value: 'healthy' },
    { weight: 1, value: 'degraded' },
    { weight: 1, value: 'unhealthy' }
  ]),
  brokerCount: faker.number.int({ min: 3, max: 9 }),
  topicCount: faker.number.int({ min: 10, max: 500 }),
  health: {
    score: faker.number.int({ min: 70, max: 100 }),
    offlinePartitions: faker.helpers.maybe(() => faker.number.int({ min: 1, max: 5 }), { probability: 0.1 }) || 0,
    underReplicatedPartitions: faker.number.int({ min: 0, max: 10 })
  }
}));

export const topicFactory = Factory.define<KafkaTopic>(({ sequence, params }) => ({
  id: `topic-${sequence}`,
  name: faker.helpers.arrayElement(['events', 'logs', 'metrics', 'commands']) + `-${faker.word.noun()}-${sequence}`,
  clusterId: params.clusterId || `cluster-${faker.number.int({ min: 1, max: 10 })}`,
  partitionCount: faker.number.int({ min: 1, max: 100 }),
  replicationFactor: faker.helpers.arrayElement([1, 3, 5]),
  retentionMs: faker.helpers.arrayElement([86400000, 604800000, 2592000000]), // 1d, 7d, 30d
  throughput: {
    bytesInPerSec: faker.number.int({ min: 1000, max: 10000000 }),
    bytesOutPerSec: faker.number.int({ min: 1000, max: 10000000 }),
    messagesInPerSec: faker.number.int({ min: 10, max: 10000 })
  }
}));

// Usage in tests
const testCluster = clusterFactory.build({
  name: 'test-cluster',
  status: 'healthy'
});

const testTopics = topicFactory.buildList(5, {
  clusterId: testCluster.id
});
```

## Test Utilities

### Custom Test Helpers

```typescript
// tests/helpers/test-utils.ts
import { render, RenderOptions } from '@testing-library/react';
import { NerdletStateContext, PlatformStateContext } from 'nr1';

interface TestProviderProps {
  children: React.ReactNode;
  nerdletState?: any;
  platformState?: any;
}

const TestProviders: React.FC<TestProviderProps> = ({ 
  children, 
  nerdletState = {}, 
  platformState = {} 
}) => {
  return (
    <PlatformStateContext.Provider value={platformState}>
      <NerdletStateContext.Provider value={nerdletState}>
        {children}
      </NerdletStateContext.Provider>
    </PlatformStateContext.Provider>
  );
};

export const renderWithProviders = (
  ui: React.ReactElement,
  options?: Omit<RenderOptions, 'wrapper'> & {
    nerdletState?: any;
    platformState?: any;
  }
) => {
  const { nerdletState, platformState, ...renderOptions } = options || {};

  return render(ui, {
    wrapper: ({ children }) => (
      <TestProviders nerdletState={nerdletState} platformState={platformState}>
        {children}
      </TestProviders>
    ),
    ...renderOptions
  });
};

// Query test helpers
export const waitForQuery = async (queryMock: jest.Mock, expectedCall?: any) => {
  await waitFor(() => {
    if (expectedCall) {
      expect(queryMock).toHaveBeenCalledWith(expectedCall);
    } else {
      expect(queryMock).toHaveBeenCalled();
    }
  });
};

// Assertion helpers
export const expectHealthScore = (element: HTMLElement, score: number) => {
  const status = getHealthStatus(score);
  expect(element).toHaveTextContent(score.toString());
  expect(element).toHaveClass(`health-indicator--${status}`);
};
```

## Continuous Integration

### CI Test Pipeline

```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node
        uses: actions/setup-node@v3
        with:
          node-version: '16'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
          flags: unit

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup test environment
        run: |
          docker-compose -f docker-compose.test.yml up -d
          ./scripts/wait-for-services.sh
      
      - name: Run integration tests
        env:
          TEST_NR_API_KEY: ${{ secrets.TEST_NR_API_KEY }}
        run: npm run test:integration
      
      - name: Cleanup
        if: always()
        run: docker-compose -f docker-compose.test.yml down

  e2e-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    strategy:
      matrix:
        browser: [chromium, firefox, webkit]
    steps:
      - uses: actions/checkout@v3
      
      - name: Install Playwright
        run: npx playwright install --with-deps ${{ matrix.browser }}
      
      - name: Run E2E tests
        env:
          NR_API_KEY: ${{ secrets.E2E_NR_API_KEY }}
        run: npm run test:e2e -- --project=${{ matrix.browser }}
      
      - name: Upload test artifacts
        if: failure()
        uses: actions/upload-artifact@v3
        with:
          name: e2e-artifacts-${{ matrix.browser }}
          path: |
            test-results/
            playwright-report/

  performance-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - uses: actions/checkout@v3
      
      - name: Run performance tests
        run: |
          npm run test:performance
          
      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const results = require('./performance-results.json');
            const comment = `### Performance Test Results
            
            | Query | Avg Duration | P95 Duration | Status |
            |-------|--------------|--------------|--------|
            ${results.map(r => `| ${r.name} | ${r.avg}ms | ${r.p95}ms | ${r.status} |`).join('\n')}
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
```

## Test Reporting

### Test Coverage Configuration

```javascript
// jest.config.js coverage settings
module.exports = {
  collectCoverageFrom: [
    'src/**/*.{ts,tsx}',
    '!src/**/*.d.ts',
    '!src/**/index.ts',
    '!src/**/__tests__/**',
    '!src/**/__mocks__/**'
  ],
  coverageReporters: ['text', 'lcov', 'html', 'json-summary'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/utils/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    },
    './src/components/': {
      branches: 75,
      functions: 75,
      lines: 75,
      statements: 75
    }
  }
};
```

## Best Practices

### 1. Test Organization
- Follow AAA pattern (Arrange, Act, Assert)
- Use descriptive test names
- Group related tests with describe blocks
- Keep tests focused and atomic

### 2. Test Data
- Use factories for consistent test data
- Avoid hardcoded values
- Clean up test data after tests
- Use realistic data scenarios

### 3. Mocking
- Mock external dependencies
- Use partial mocks when possible
- Verify mock calls appropriately
- Reset mocks between tests

### 4. Performance
- Run tests in parallel when possible
- Use test databases/environments
- Optimize slow tests
- Monitor test execution time

### 5. Maintenance
- Keep tests up to date with code
- Refactor tests along with code
- Remove obsolete tests
- Document complex test scenarios

## Conclusion

A comprehensive testing strategy ensures the reliability and quality of the Message Queues monitoring system. By implementing multiple layers of testing and following best practices, teams can confidently deploy changes while maintaining system stability.