# Performance Optimization

## Overview

The Message Queues monitoring system is designed to handle large-scale Kafka infrastructures with thousands of entities. This document details the performance optimization strategies implemented across the application.

## Performance Architecture

### Optimization Layers

```
┌─────────────────────────────────────────────────────────┐
│                 Query Optimization                      │
│         (NRQL Query Planning & Execution)               │
├─────────────────────────────────────────────────────────┤
│               Data Management                           │
│      (Caching, Pagination, Virtual Scrolling)           │
├─────────────────────────────────────────────────────────┤
│              Rendering Optimization                     │
│       (React Memoization, Canvas Rendering)             │
├─────────────────────────────────────────────────────────┤
│              Network Optimization                       │
│        (Request Batching, Data Compression)             │
├─────────────────────────────────────────────────────────┤
│               Memory Management                         │
│        (Garbage Collection, Resource Cleanup)           │
└─────────────────────────────────────────────────────────┘
```

## Query Optimization

### NRQL Query Strategies

#### 1. Pre-aggregation Patterns

```typescript
class QueryOptimizer {
  // Pre-aggregate at the database level
  optimizeClusterQuery(filters: QueryFilters): string {
    const baseQuery = `
      FROM (
        SELECT 
          latest(provider.clusterName) as clusterName,
          latest(provider.activeControllers) as activeControllers,
          latest(provider.offlinePartitions) as offlinePartitions,
          average(provider.bytesInPerSec.Average) as bytesIn,
          average(provider.bytesOutPerSec.Average) as bytesOut
        FROM AwsMskBrokerSample
        WHERE provider.clusterName IS NOT NULL
        FACET provider.clusterName, provider.brokerId
        LIMIT MAX
      )
      SELECT 
        latest(clusterName) as clusterName,
        sum(activeControllers) as activeControllers,
        sum(offlinePartitions) as offlinePartitions,
        sum(bytesIn) as totalBytesIn,
        sum(bytesOut) as totalBytesOut
      FACET clusterName
    `;
    
    return this.applyFilters(baseQuery, filters);
  }
  
  // Use sub-queries for complex aggregations
  optimizeThroughputQuery(accountIds: string[]): string {
    return `
      FROM (
        SELECT 
          sum(bytesInPerSec) as incomingThroughput,
          sum(bytesOutPerSec) as outgoingThroughput
        FROM (
          SELECT 
            average(provider.bytesInPerSec.Average) as bytesInPerSec,
            average(provider.bytesOutPerSec.Average) as bytesOutPerSec
          FROM AwsMskBrokerSample
          WHERE provider.accountId IN (${accountIds.join(',')})
          FACET provider.clusterName, provider.brokerId
        )
      )
      SELECT 
        sum(incomingThroughput) as totalIncoming,
        sum(outgoingThroughput) as totalOutgoing
    `;
  }
}
```

#### 2. Efficient Time Window Queries

```typescript
interface TimeWindowOptimization {
  // Adaptive time windows based on data volume
  getOptimalTimeWindow(entityCount: number): string {
    if (entityCount < 100) return 'SINCE 1 hour ago';
    if (entityCount < 500) return 'SINCE 30 minutes ago';
    if (entityCount < 1000) return 'SINCE 15 minutes ago';
    return 'SINCE 5 minutes ago';
  }
  
  // Time-based sampling for historical data
  getHistoricalSampling(timeRange: TimeRange): string {
    const duration = timeRange.end - timeRange.begin;
    const hours = duration / (1000 * 60 * 60);
    
    if (hours <= 1) return 'TIMESERIES 1 minute';
    if (hours <= 24) return 'TIMESERIES 5 minutes';
    if (hours <= 168) return 'TIMESERIES 30 minutes';
    return 'TIMESERIES 1 hour';
  }
}
```

#### 3. Query Result Caching

```typescript
class QueryCache {
  private cache = new Map<string, CachedQuery>();
  private readonly MAX_CACHE_SIZE = 100;
  private readonly DEFAULT_TTL = 60000; // 1 minute
  
  async executeQuery(query: string, options: QueryOptions = {}): Promise<any> {
    const cacheKey = this.generateCacheKey(query, options);
    const cached = this.cache.get(cacheKey);
    
    // Return cached result if valid
    if (cached && !this.isExpired(cached)) {
      return cached.result;
    }
    
    // Execute query
    const result = await NrqlQuery.query({
      query,
      accountId: options.accountId,
      timeout: options.timeout || 30000
    });
    
    // Cache result
    this.setCached(cacheKey, result, options.ttl);
    
    return result;
  }
  
  private setCached(key: string, result: any, ttl?: number): void {
    // Implement LRU eviction
    if (this.cache.size >= this.MAX_CACHE_SIZE) {
      const oldestKey = this.findOldestEntry();
      this.cache.delete(oldestKey);
    }
    
    this.cache.set(key, {
      result,
      timestamp: Date.now(),
      ttl: ttl || this.DEFAULT_TTL
    });
  }
}
```

### GraphQL Query Optimization

#### 1. Batch Entity Fetching

```typescript
class EntityBatchFetcher {
  private pendingRequests = new Map<string, Promise<any>>();
  private batchQueue: BatchRequest[] = [];
  private batchTimer: NodeJS.Timeout | null = null;
  
  async fetchEntity(guid: string): Promise<Entity> {
    // Check if request is already pending
    if (this.pendingRequests.has(guid)) {
      return this.pendingRequests.get(guid)!;
    }
    
    // Create promise for this request
    const promise = new Promise<Entity>((resolve, reject) => {
      this.batchQueue.push({ guid, resolve, reject });
      this.scheduleBatch();
    });
    
    this.pendingRequests.set(guid, promise);
    return promise;
  }
  
  private scheduleBatch(): void {
    if (this.batchTimer) return;
    
    this.batchTimer = setTimeout(() => {
      this.executeBatch();
    }, 10); // 10ms debounce
  }
  
  private async executeBatch(): Promise<void> {
    const batch = [...this.batchQueue];
    this.batchQueue = [];
    this.batchTimer = null;
    
    if (batch.length === 0) return;
    
    const query = ngql`
      query BatchEntityFetch($guids: [EntityGuid!]!) {
        actor {
          entities(guids: $guids) {
            guid
            name
            type
            tags {
              key
              values
            }
            ... on InfrastructureAwsKafkaClusterEntity {
              clusterName
              activeControllers
              offlinePartitions
            }
          }
        }
      }
    `;
    
    try {
      const result = await NerdGraphQuery.query({
        query,
        variables: { guids: batch.map(b => b.guid) }
      });
      
      const entityMap = new Map(
        result.data.actor.entities.map(e => [e.guid, e])
      );
      
      batch.forEach(({ guid, resolve, reject }) => {
        const entity = entityMap.get(guid);
        if (entity) {
          resolve(entity);
        } else {
          reject(new Error(`Entity ${guid} not found`));
        }
        this.pendingRequests.delete(guid);
      });
    } catch (error) {
      batch.forEach(({ guid, reject }) => {
        reject(error);
        this.pendingRequests.delete(guid);
      });
    }
  }
}
```

## Data Management

### Virtual Scrolling Implementation

```typescript
interface VirtualScrollConfig {
  itemHeight: number;
  bufferSize: number;
  containerHeight: number;
}

class VirtualScrollManager {
  private scrollTop = 0;
  private visibleRange = { start: 0, end: 0 };
  
  constructor(private config: VirtualScrollConfig) {}
  
  calculateVisibleItems<T>(items: T[]): {
    visibleItems: T[];
    offsetY: number;
    totalHeight: number;
  } {
    const { itemHeight, bufferSize, containerHeight } = this.config;
    
    // Calculate visible range with buffer
    const startIndex = Math.max(
      0,
      Math.floor(this.scrollTop / itemHeight) - bufferSize
    );
    const endIndex = Math.min(
      items.length,
      Math.ceil((this.scrollTop + containerHeight) / itemHeight) + bufferSize
    );
    
    this.visibleRange = { start: startIndex, end: endIndex };
    
    return {
      visibleItems: items.slice(startIndex, endIndex),
      offsetY: startIndex * itemHeight,
      totalHeight: items.length * itemHeight
    };
  }
  
  handleScroll(scrollTop: number): void {
    this.scrollTop = scrollTop;
  }
}

// React component using virtual scrolling
const VirtualTable: React.FC<VirtualTableProps> = ({ data, columns }) => {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scrollManager] = useState(() => new VirtualScrollManager({
    itemHeight: 48,
    bufferSize: 5,
    containerHeight: 600
  }));
  
  const [visibleData, setVisibleData] = useState({
    items: [],
    offsetY: 0,
    totalHeight: 0
  });
  
  useEffect(() => {
    const result = scrollManager.calculateVisibleItems(data);
    setVisibleData(result);
  }, [data, scrollManager]);
  
  const handleScroll = useCallback((e: React.UIEvent) => {
    const scrollTop = e.currentTarget.scrollTop;
    scrollManager.handleScroll(scrollTop);
    
    const result = scrollManager.calculateVisibleItems(data);
    setVisibleData(result);
  }, [data, scrollManager]);
  
  return (
    <div
      ref={containerRef}
      className="virtual-table"
      onScroll={handleScroll}
      style={{ height: 600, overflow: 'auto' }}
    >
      <div style={{ height: visibleData.totalHeight, position: 'relative' }}>
        <div
          style={{
            transform: `translateY(${visibleData.offsetY}px)`,
            position: 'absolute',
            top: 0,
            left: 0,
            right: 0
          }}
        >
          {visibleData.items.map((item, index) => (
            <TableRow key={item.id} data={item} columns={columns} />
          ))}
        </div>
      </div>
    </div>
  );
};
```

### Pagination Strategy

```typescript
class PaginationManager {
  private pageSize = 50;
  private currentPage = 1;
  private totalItems = 0;
  private cache = new Map<number, any[]>();
  
  async fetchPage(page: number): Promise<PageResult> {
    // Check cache first
    if (this.cache.has(page)) {
      return {
        data: this.cache.get(page)!,
        page,
        totalPages: Math.ceil(this.totalItems / this.pageSize),
        fromCache: true
      };
    }
    
    // Fetch data
    const offset = (page - 1) * this.pageSize;
    const query = `
      SELECT *
      FROM KafkaCluster
      LIMIT ${this.pageSize}
      OFFSET ${offset}
    `;
    
    const result = await this.executeQuery(query);
    
    // Cache result
    this.cache.set(page, result.data);
    
    // Implement cache size limit
    if (this.cache.size > 10) {
      const oldestPage = Math.min(...this.cache.keys());
      this.cache.delete(oldestPage);
    }
    
    return {
      data: result.data,
      page,
      totalPages: Math.ceil(result.totalCount / this.pageSize),
      fromCache: false
    };
  }
  
  // Prefetch adjacent pages
  async prefetchAdjacentPages(currentPage: number): Promise<void> {
    const pagesToPrefetch = [
      currentPage - 1,
      currentPage + 1
    ].filter(p => p > 0 && p <= this.totalPages);
    
    await Promise.all(
      pagesToPrefetch.map(page => this.fetchPage(page))
    );
  }
}
```

## Rendering Optimization

### React Performance Patterns

#### 1. Component Memoization

```typescript
// Memoized expensive components
const HoneyCombHexagon = React.memo<HexagonProps>(({ 
  entity, 
  position, 
  size, 
  selected,
  onSelect 
}) => {
  // Only re-render if props actually changed
  return (
    <Hexagon
      entity={entity}
      position={position}
      size={size}
      selected={selected}
      onClick={() => onSelect(entity)}
    />
  );
}, (prevProps, nextProps) => {
  // Custom comparison function
  return (
    prevProps.entity.guid === nextProps.entity.guid &&
    prevProps.position.x === nextProps.position.x &&
    prevProps.position.y === nextProps.position.y &&
    prevProps.size === nextProps.size &&
    prevProps.selected === nextProps.selected
  );
});

// Memoized callbacks
const useOptimizedCallbacks = () => {
  const handleEntitySelect = useCallback((entity: Entity) => {
    // Selection logic
  }, []); // Empty deps = stable reference
  
  const handleFilterChange = useCallback((filters: Filter[]) => {
    // Filter logic
  }, []); // Stable reference
  
  return { handleEntitySelect, handleFilterChange };
};
```

#### 2. Render Optimization with useMemo

```typescript
const OptimizedTableComponent: React.FC<TableProps> = ({ data, filters, sorting }) => {
  // Expensive filtering operation
  const filteredData = useMemo(() => {
    if (!filters.length) return data;
    
    return data.filter(item => {
      return filters.every(filter => 
        matchesFilter(item, filter)
      );
    });
  }, [data, filters]); // Only recalculate when dependencies change
  
  // Expensive sorting operation
  const sortedData = useMemo(() => {
    if (!sorting) return filteredData;
    
    return [...filteredData].sort((a, b) => {
      const aValue = a[sorting.column];
      const bValue = b[sorting.column];
      
      return sorting.direction === 'asc' 
        ? compare(aValue, bValue)
        : compare(bValue, aValue);
    });
  }, [filteredData, sorting]);
  
  return <Table data={sortedData} />;
};
```

### Canvas Rendering for Large Datasets

```typescript
class CanvasRenderer {
  private ctx: CanvasRenderingContext2D;
  private offscreenCanvas: OffscreenCanvas;
  private renderQueue: RenderTask[] = [];
  private isRendering = false;
  
  constructor(canvas: HTMLCanvasElement) {
    this.ctx = canvas.getContext('2d', {
      alpha: false,
      desynchronized: true
    })!;
    
    // Create offscreen canvas for double buffering
    this.offscreenCanvas = new OffscreenCanvas(
      canvas.width,
      canvas.height
    );
  }
  
  // Batch render operations
  queueRender(task: RenderTask): void {
    this.renderQueue.push(task);
    
    if (!this.isRendering) {
      this.processRenderQueue();
    }
  }
  
  private async processRenderQueue(): Promise<void> {
    this.isRendering = true;
    
    while (this.renderQueue.length > 0) {
      // Process in batches
      const batch = this.renderQueue.splice(0, 100);
      
      await new Promise(resolve => {
        requestAnimationFrame(() => {
          this.renderBatch(batch);
          resolve(void 0);
        });
      });
    }
    
    this.isRendering = false;
  }
  
  private renderBatch(tasks: RenderTask[]): void {
    const offCtx = this.offscreenCanvas.getContext('2d')!;
    
    // Clear offscreen canvas
    offCtx.clearRect(0, 0, this.offscreenCanvas.width, this.offscreenCanvas.height);
    
    // Render all tasks to offscreen canvas
    tasks.forEach(task => {
      this.renderTask(offCtx, task);
    });
    
    // Copy to main canvas in one operation
    this.ctx.drawImage(this.offscreenCanvas, 0, 0);
  }
  
  // Efficient hexagon rendering
  private renderHexagon(ctx: CanvasRenderingContext2D, hex: HexagonData): void {
    const { x, y, size, color } = hex;
    
    ctx.save();
    ctx.translate(x, y);
    
    // Use pre-calculated path
    if (!this.hexPath) {
      this.hexPath = new Path2D();
      for (let i = 0; i < 6; i++) {
        const angle = (Math.PI / 3) * i;
        const px = size * Math.cos(angle);
        const py = size * Math.sin(angle);
        
        if (i === 0) {
          this.hexPath.moveTo(px, py);
        } else {
          this.hexPath.lineTo(px, py);
        }
      }
      this.hexPath.closePath();
    }
    
    ctx.fillStyle = color;
    ctx.fill(this.hexPath);
    
    ctx.restore();
  }
}
```

## Network Optimization

### Request Batching and Deduplication

```typescript
class NetworkOptimizer {
  private requestQueue = new Map<string, PendingRequest>();
  private batchTimer: NodeJS.Timeout | null = null;
  private readonly BATCH_DELAY = 50; // ms
  private readonly MAX_BATCH_SIZE = 20;
  
  async request<T>(config: RequestConfig): Promise<T> {
    const key = this.getRequestKey(config);
    
    // Check for duplicate request
    if (this.requestQueue.has(key)) {
      return this.requestQueue.get(key)!.promise as Promise<T>;
    }
    
    // Create deferred promise
    const deferred = this.createDeferred<T>();
    
    this.requestQueue.set(key, {
      config,
      deferred,
      promise: deferred.promise
    });
    
    // Schedule batch execution
    this.scheduleBatch();
    
    return deferred.promise;
  }
  
  private scheduleBatch(): void {
    if (this.batchTimer) return;
    
    this.batchTimer = setTimeout(() => {
      this.executeBatch();
    }, this.BATCH_DELAY);
  }
  
  private async executeBatch(): Promise<void> {
    this.batchTimer = null;
    
    // Group requests by endpoint
    const groups = this.groupRequestsByEndpoint();
    
    // Execute each group
    await Promise.all(
      Array.from(groups.entries()).map(([endpoint, requests]) => 
        this.executeBatchRequest(endpoint, requests)
      )
    );
    
    this.requestQueue.clear();
  }
  
  private async executeBatchRequest(
    endpoint: string,
    requests: PendingRequest[]
  ): Promise<void> {
    try {
      // Combine requests into batch
      const batchPayload = {
        requests: requests.map(r => r.config.payload)
      };
      
      const response = await fetch(endpoint, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Batch-Request': 'true'
        },
        body: JSON.stringify(batchPayload)
      });
      
      const results = await response.json();
      
      // Resolve individual promises
      requests.forEach((request, index) => {
        request.deferred.resolve(results[index]);
      });
    } catch (error) {
      // Reject all promises in batch
      requests.forEach(request => {
        request.deferred.reject(error);
      });
    }
  }
}
```

### Data Compression

```typescript
class CompressionManager {
  // Compress large payloads before sending
  async compressRequest(data: any): Promise<ArrayBuffer> {
    const json = JSON.stringify(data);
    const encoder = new TextEncoder();
    const input = encoder.encode(json);
    
    // Use CompressionStream API
    const stream = new CompressionStream('gzip');
    const writer = stream.writable.getWriter();
    
    writer.write(input);
    writer.close();
    
    const compressed = [];
    const reader = stream.readable.getReader();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      compressed.push(value);
    }
    
    return new Blob(compressed).arrayBuffer();
  }
  
  // Decompress responses
  async decompressResponse(data: ArrayBuffer): Promise<any> {
    const stream = new DecompressionStream('gzip');
    const writer = stream.writable.getWriter();
    
    writer.write(data);
    writer.close();
    
    const decompressed = [];
    const reader = stream.readable.getReader();
    
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;
      decompressed.push(value);
    }
    
    const decoder = new TextDecoder();
    const json = decoder.decode(new Blob(decompressed));
    
    return JSON.parse(json);
  }
}
```

## Memory Management

### Resource Cleanup

```typescript
class ResourceManager {
  private resources = new Set<Disposable>();
  private timers = new Set<NodeJS.Timeout>();
  private subscriptions = new Set<Subscription>();
  
  // Register resources for cleanup
  register(resource: Disposable): void {
    this.resources.add(resource);
  }
  
  // Register timer for cleanup
  registerTimer(timer: NodeJS.Timeout): void {
    this.timers.add(timer);
  }
  
  // Register subscription for cleanup
  registerSubscription(subscription: Subscription): void {
    this.subscriptions.add(subscription);
  }
  
  // Clean up all resources
  cleanup(): void {
    // Clear timers
    this.timers.forEach(timer => clearTimeout(timer));
    this.timers.clear();
    
    // Unsubscribe from all subscriptions
    this.subscriptions.forEach(sub => sub.unsubscribe());
    this.subscriptions.clear();
    
    // Dispose of resources
    this.resources.forEach(resource => resource.dispose());
    this.resources.clear();
  }
}

// React hook for automatic cleanup
const useResourceManager = () => {
  const [manager] = useState(() => new ResourceManager());
  
  useEffect(() => {
    return () => {
      manager.cleanup();
    };
  }, [manager]);
  
  return manager;
};
```

### Memory Leak Prevention

```typescript
class MemoryLeakPrevention {
  // Weak references for event handlers
  private eventHandlers = new WeakMap<object, Map<string, Function>>();
  
  addEventListener(
    target: object,
    event: string,
    handler: Function
  ): void {
    if (!this.eventHandlers.has(target)) {
      this.eventHandlers.set(target, new Map());
    }
    
    const handlers = this.eventHandlers.get(target)!;
    handlers.set(event, handler);
  }
  
  removeEventListener(
    target: object,
    event: string
  ): void {
    const handlers = this.eventHandlers.get(target);
    if (handlers) {
      handlers.delete(event);
      
      if (handlers.size === 0) {
        this.eventHandlers.delete(target);
      }
    }
  }
  
  // Automatic cleanup for React components
  useAutoCleanup<T extends object>(
    ref: React.RefObject<T>,
    setup: (element: T) => (() => void)
  ): void {
    useEffect(() => {
      if (!ref.current) return;
      
      const cleanup = setup(ref.current);
      
      return () => {
        cleanup();
      };
    }, [ref, setup]);
  }
}
```

## Performance Monitoring

### Performance Metrics Collection

```typescript
class PerformanceMonitor {
  private metrics = new Map<string, PerformanceMetric>();
  
  // Measure operation performance
  async measure<T>(
    name: string,
    operation: () => Promise<T>
  ): Promise<T> {
    const start = performance.now();
    
    try {
      const result = await operation();
      const duration = performance.now() - start;
      
      this.recordMetric(name, {
        duration,
        success: true,
        timestamp: Date.now()
      });
      
      return result;
    } catch (error) {
      const duration = performance.now() - start;
      
      this.recordMetric(name, {
        duration,
        success: false,
        error: error.message,
        timestamp: Date.now()
      });
      
      throw error;
    }
  }
  
  // Get performance report
  getReport(): PerformanceReport {
    const report: PerformanceReport = {};
    
    this.metrics.forEach((metrics, name) => {
      const durations = metrics.map(m => m.duration);
      
      report[name] = {
        count: metrics.length,
        averageDuration: average(durations),
        minDuration: Math.min(...durations),
        maxDuration: Math.max(...durations),
        p95Duration: percentile(durations, 95),
        successRate: metrics.filter(m => m.success).length / metrics.length
      };
    });
    
    return report;
  }
  
  // Send metrics to New Relic
  async reportToNewRelic(): Promise<void> {
    const report = this.getReport();
    
    await UserStorage.setDocument({
      document: {
        id: 'performance-metrics',
        timestamp: Date.now(),
        metrics: report
      }
    });
  }
}
```

## Best Practices

### 1. Query Optimization
- Pre-aggregate data at the database level
- Use appropriate time windows
- Implement query result caching
- Batch similar queries together

### 2. Data Management
- Implement virtual scrolling for large lists
- Use pagination for data tables
- Cache frequently accessed data
- Clean up unused data regularly

### 3. Rendering Performance
- Memoize expensive components
- Use React.memo and useMemo appropriately
- Implement canvas rendering for large visualizations
- Batch DOM updates

### 4. Network Optimization
- Batch API requests
- Implement request deduplication
- Use data compression for large payloads
- Cache API responses

### 5. Memory Management
- Clean up event listeners and timers
- Use weak references where appropriate
- Monitor memory usage
- Implement proper resource disposal

## Performance Targets

### Response Time Goals
- Initial page load: < 3 seconds
- Data refresh: < 1 second
- User interactions: < 100ms
- Search operations: < 500ms

### Scalability Targets
- Support 1000+ Kafka clusters
- Handle 10,000+ topics
- Render 5000+ entities in HoneyComb view
- Process 100+ concurrent users

## Conclusion

Performance optimization is critical for the Message Queues monitoring system to handle large-scale Kafka infrastructures effectively. By implementing these optimization strategies across all layers of the application, we ensure a responsive and scalable monitoring solution.