# Visualization Systems

## Overview

The Message Queues monitoring system employs sophisticated visualization techniques to present complex Kafka infrastructure data in an intuitive, actionable format. This document details the visualization systems, including the signature HoneyComb view, charts, and interactive elements.

## HoneyComb Visualization System

### Conceptual Design

The HoneyComb visualization represents Kafka entities as hexagonal cells in a honeycomb pattern, providing:
- High-density information display
- Intuitive health status visualization
- Scalable representation (100s to 1000s of entities)
- Interactive exploration capabilities

### Technical Implementation

#### Core Architecture

```typescript
// HoneyComb visualization stack
interface HoneyCombArchitecture {
  // Rendering engine
  renderer: {
    type: 'SVG' | 'Canvas';
    library: '@datanerd/fsi-high-density-view';
    performance: 'GPU-accelerated';
  };
  
  // Layout algorithm
  layout: {
    algorithm: 'hexagonal-packing';
    optimization: 'force-directed';
    constraints: 'viewport-bounded';
  };
  
  // Interaction layer
  interactions: {
    selection: 'multi-select';
    hover: 'tooltip-on-delay';
    zoom: 'mouse-wheel + pinch';
    pan: 'click-drag';
  };
}
```

#### Hexagon Calculation Engine

```typescript
class HexagonLayoutEngine {
  private readonly HEX_RATIO = Math.sqrt(3) / 2;
  
  calculateLayout(entities: Entity[], viewport: Viewport): HexagonLayout {
    // Calculate optimal hexagon size
    const hexSize = this.calculateOptimalSize(entities.length, viewport);
    
    // Generate hexagonal grid positions
    const positions = this.generateHexGrid(entities.length, hexSize, viewport);
    
    // Apply force-directed optimization
    const optimized = this.applyForceSimulation(entities, positions);
    
    // Group by cluster if needed
    const grouped = this.applyGrouping(optimized, entities);
    
    return {
      hexagons: grouped,
      hexSize,
      bounds: this.calculateBounds(grouped)
    };
  }
  
  private calculateOptimalSize(count: number, viewport: Viewport): number {
    // Base size calculation
    const area = viewport.width * viewport.height;
    const hexArea = area / (count * 1.2); // 20% spacing
    
    // Derive radius from area
    const radius = Math.sqrt(hexArea / (3 * this.HEX_RATIO));
    
    // Apply constraints
    return Math.max(20, Math.min(60, radius));
  }
  
  private generateHexGrid(count: number, size: number, viewport: Viewport): Position[] {
    const positions: Position[] = [];
    const cols = Math.floor(viewport.width / (size * 3));
    const rows = Math.ceil(count / cols);
    
    for (let row = 0; row < rows; row++) {
      for (let col = 0; col < cols && positions.length < count; col++) {
        const x = col * size * 3 + (row % 2) * size * 1.5;
        const y = row * size * this.HEX_RATIO * 2;
        
        positions.push({ x, y });
      }
    }
    
    return positions;
  }
}
```

#### Visual Encoding System

```typescript
interface VisualEncoding {
  // Color encoding for health status
  color: {
    healthy: '#11A968';      // Green
    warning: '#F5A623';      // Orange
    critical: '#D0021B';     // Red
    unknown: '#9B9B9B';      // Gray
    noData: '#E7E7E8';      // Light gray
  };
  
  // Size encoding for importance
  size: {
    base: 40;
    scaling: {
      byMetric: (value: number, max: number) => 
        base + (value / max) * 20;
      byImportance: (score: number) => 
        base * (1 + score * 0.5);
    };
  };
  
  // Border encoding for alerts
  border: {
    noAlert: { width: 1, color: '#CCCCCC' };
    warning: { width: 2, color: '#F5A623' };
    critical: { width: 3, color: '#D0021B' };
    selected: { width: 4, color: '#007EDB' };
  };
  
  // Pattern encoding for states
  patterns: {
    notReporting: 'diagonal-stripes';
    maintenance: 'dots';
    degraded: 'horizontal-stripes';
  };
}
```

### Hexagon Rendering

#### SVG Implementation

```typescript
const HexagonSVG: React.FC<HexagonProps> = ({ 
  entity, 
  position, 
  size, 
  selected,
  onSelect,
  onHover 
}) => {
  // Calculate hexagon points
  const points = useMemo(() => {
    const pts: string[] = [];
    for (let i = 0; i < 6; i++) {
      const angle = (Math.PI / 3) * i;
      const x = position.x + size * Math.cos(angle);
      const y = position.y + size * Math.sin(angle);
      pts.push(`${x},${y}`);
    }
    return pts.join(' ');
  }, [position, size]);
  
  const encoding = getVisualEncoding(entity);
  
  return (
    <g className="hexagon-entity">
      {/* Background hexagon */}
      <polygon
        points={points}
        fill={encoding.fill}
        stroke={encoding.stroke}
        strokeWidth={encoding.strokeWidth}
        opacity={encoding.opacity}
        onClick={() => onSelect(entity)}
        onMouseEnter={() => onHover(entity)}
        onMouseLeave={() => onHover(null)}
      />
      
      {/* Pattern overlay if needed */}
      {encoding.pattern && (
        <polygon
          points={points}
          fill={`url(#${encoding.pattern})`}
          opacity={0.3}
          pointerEvents="none"
        />
      )}
      
      {/* Entity label */}
      <text
        x={position.x}
        y={position.y}
        textAnchor="middle"
        dominantBaseline="central"
        fontSize={Math.min(size * 0.3, 12)}
        fill="white"
        pointerEvents="none"
      >
        {truncateLabel(entity.name, size)}
      </text>
      
      {/* Alert indicator */}
      {entity.alertSeverity && (
        <circle
          cx={position.x + size * 0.7}
          cy={position.y - size * 0.7}
          r={6}
          fill={getAlertColor(entity.alertSeverity)}
          stroke="white"
          strokeWidth={1}
        />
      )}
    </g>
  );
};
```

#### Canvas Implementation (Performance Mode)

```typescript
class HexagonCanvasRenderer {
  private ctx: CanvasRenderingContext2D;
  private hexagons: Map<string, HexagonData>;
  
  render(entities: Entity[], layout: HexagonLayout): void {
    // Clear canvas
    this.ctx.clearRect(0, 0, this.canvas.width, this.canvas.height);
    
    // Enable anti-aliasing
    this.ctx.imageSmoothingEnabled = true;
    
    // Render each hexagon
    entities.forEach((entity, index) => {
      const hex = layout.hexagons[index];
      this.renderHexagon(entity, hex);
    });
    
    // Render selection overlay
    this.renderSelections();
  }
  
  private renderHexagon(entity: Entity, hex: HexagonData): void {
    const { x, y, size } = hex;
    const encoding = getVisualEncoding(entity);
    
    // Draw hexagon path
    this.ctx.beginPath();
    for (let i = 0; i < 6; i++) {
      const angle = (Math.PI / 3) * i;
      const px = x + size * Math.cos(angle);
      const py = y + size * Math.sin(angle);
      
      if (i === 0) {
        this.ctx.moveTo(px, py);
      } else {
        this.ctx.lineTo(px, py);
      }
    }
    this.ctx.closePath();
    
    // Fill
    this.ctx.fillStyle = encoding.fill;
    this.ctx.fill();
    
    // Stroke
    this.ctx.strokeStyle = encoding.stroke;
    this.ctx.lineWidth = encoding.strokeWidth;
    this.ctx.stroke();
    
    // Draw text
    this.ctx.fillStyle = 'white';
    this.ctx.font = `${Math.min(size * 0.3, 12)}px sans-serif`;
    this.ctx.textAlign = 'center';
    this.ctx.textBaseline = 'middle';
    this.ctx.fillText(truncateLabel(entity.name, size), x, y);
  }
}
```

### Interactive Features

#### Tooltip System

```typescript
interface TooltipSystem {
  // Tooltip data structure
  data: {
    entity: Entity;
    metrics: RealtimeMetrics;
    position: Position;
    visible: boolean;
  };
  
  // Tooltip renderer
  renderer: (data: TooltipData) => ReactNode;
  
  // Interaction config
  config: {
    delay: 500; // ms before showing
    offset: { x: 10, y: -10 };
    followMouse: boolean;
    smartPositioning: boolean; // Avoid viewport edges
  };
}

const EntityTooltip: React.FC<TooltipProps> = ({ entity, metrics, position }) => {
  const tooltipRef = useRef<HTMLDivElement>(null);
  const adjustedPosition = useSmartPosition(position, tooltipRef);
  
  return (
    <Portal>
      <div
        ref={tooltipRef}
        className="entity-tooltip"
        style={{
          position: 'fixed',
          left: adjustedPosition.x,
          top: adjustedPosition.y,
          zIndex: 9999
        }}
      >
        <div className="tooltip-header">
          <EntityIcon type={entity.type} />
          <h4>{entity.name}</h4>
          <HealthBadge status={entity.healthStatus} />
        </div>
        
        <div className="tooltip-metrics">
          <MetricRow label="Throughput" value={metrics.throughput} />
          <MetricRow label="Messages/sec" value={metrics.messageRate} />
          <MetricRow label="Lag" value={metrics.lag} />
        </div>
        
        <div className="tooltip-footer">
          <small>Click for details</small>
        </div>
      </div>
    </Portal>
  );
};
```

#### Selection System

```typescript
class SelectionManager {
  private selected: Set<string> = new Set();
  private selectionMode: 'single' | 'multiple' = 'single';
  
  handleSelection(entity: Entity, event: MouseEvent): void {
    const guid = entity.guid;
    
    if (this.selectionMode === 'single') {
      this.selected.clear();
      this.selected.add(guid);
    } else {
      // Multi-select with Ctrl/Cmd
      if (event.ctrlKey || event.metaKey) {
        if (this.selected.has(guid)) {
          this.selected.delete(guid);
        } else {
          this.selected.add(guid);
        }
      } else {
        // Single select without modifier
        this.selected.clear();
        this.selected.add(guid);
      }
    }
    
    this.notifySelectionChange();
  }
  
  handleRangeSelection(start: Position, end: Position): void {
    const bounds = {
      left: Math.min(start.x, end.x),
      right: Math.max(start.x, end.x),
      top: Math.min(start.y, end.y),
      bottom: Math.max(start.y, end.y)
    };
    
    // Select all hexagons within bounds
    this.hexagons.forEach((hex, guid) => {
      if (isWithinBounds(hex.position, bounds)) {
        this.selected.add(guid);
      }
    });
    
    this.notifySelectionChange();
  }
}
```

#### Zoom and Pan Controls

```typescript
class ZoomPanController {
  private scale: number = 1;
  private translation: Position = { x: 0, y: 0 };
  private bounds: Bounds;
  
  handleWheel(event: WheelEvent): void {
    event.preventDefault();
    
    const delta = event.deltaY > 0 ? 0.9 : 1.1;
    const newScale = this.scale * delta;
    
    // Apply scale limits
    if (newScale >= 0.5 && newScale <= 5) {
      // Scale around mouse position
      const mousePos = this.getMousePosition(event);
      this.scaleAroundPoint(mousePos, delta);
    }
  }
  
  handlePan(deltaX: number, deltaY: number): void {
    // Apply translation with bounds checking
    const newX = this.translation.x + deltaX;
    const newY = this.translation.y + deltaY;
    
    // Ensure content stays visible
    const minX = this.viewport.width - this.bounds.width * this.scale;
    const maxX = 0;
    const minY = this.viewport.height - this.bounds.height * this.scale;
    const maxY = 0;
    
    this.translation = {
      x: Math.max(minX, Math.min(maxX, newX)),
      y: Math.max(minY, Math.min(maxY, newY))
    };
    
    this.updateTransform();
  }
  
  private updateTransform(): void {
    const transform = `translate(${this.translation.x}px, ${this.translation.y}px) scale(${this.scale})`;
    this.container.style.transform = transform;
  }
}
```

## Chart Visualization System

### Chart Types and Usage

#### 1. Line Charts (Time Series)

```typescript
interface LineChartConfig {
  id: string;
  title: string;
  data: TimeSeriesData[];
  config: {
    xAxis: {
      type: 'time';
      format: 'HH:mm';
    };
    yAxis: {
      label: string;
      formatter: (value: number) => string;
    };
    series: {
      smoothing: boolean;
      area: boolean;
      markers: boolean;
    };
  };
}

const ThroughputChart: React.FC<ChartProps> = ({ data, timeRange }) => {
  const chartConfig: LineChartConfig = {
    id: 'throughput-chart',
    title: 'Cluster Throughput Over Time',
    data: transformToTimeSeries(data),
    config: {
      xAxis: { type: 'time', format: getTimeFormat(timeRange) },
      yAxis: { 
        label: 'Throughput', 
        formatter: (v) => humanizeBytes(v) + '/s' 
      },
      series: {
        smoothing: true,
        area: true,
        markers: false
      }
    }
  };
  
  return (
    <LineChart
      {...chartConfig}
      height={300}
      onDataPointClick={handleDataPointClick}
      onBrush={handleTimeRangeSelection}
    />
  );
};
```

#### 2. Bar Charts (Comparisons)

```typescript
interface BarChartConfig {
  orientation: 'horizontal' | 'vertical';
  data: CategoryData[];
  config: {
    bars: {
      width: number | 'auto';
      spacing: number;
      colors: string[] | ((value: number) => string);
    };
    labels: {
      rotation: number;
      truncate: number;
    };
  };
}

const TopTopicsChart: React.FC = ({ topics }) => {
  const config: BarChartConfig = {
    orientation: 'horizontal',
    data: topics.slice(0, 20).map(t => ({
      category: t.name,
      value: t.throughput,
      metadata: t
    })),
    config: {
      bars: {
        width: 'auto',
        spacing: 2,
        colors: (value) => getColorByThreshold(value)
      },
      labels: {
        rotation: 0,
        truncate: 30
      }
    }
  };
  
  return (
    <BarChart
      {...config}
      height={400}
      onBarClick={(data) => navigateToTopic(data.metadata)}
    />
  );
};
```

#### 3. Area Charts (Stacked Metrics)

```typescript
const MessageRateChart: React.FC = ({ clusters }) => {
  const series = clusters.map(cluster => ({
    name: cluster.name,
    data: cluster.messageRateHistory,
    color: getClusterColor(cluster.id)
  }));
  
  return (
    <AreaChart
      series={series}
      stacked={true}
      height={300}
      config={{
        legend: { position: 'bottom', columns: 3 },
        tooltip: { 
          shared: true,
          formatter: (point) => `${point.series}: ${formatNumber(point.y)} msg/s`
        },
        area: { opacity: 0.7 }
      }}
    />
  );
};
```

#### 4. Billboard Charts (Key Metrics)

```typescript
interface BillboardConfig {
  value: number | string;
  title: string;
  subtitle?: string;
  comparison?: {
    value: number;
    period: string;
  };
  trend?: {
    direction: 'up' | 'down' | 'stable';
    percentage: number;
  };
  thresholds?: {
    critical: number;
    warning: number;
  };
}

const MetricBillboard: React.FC<BillboardConfig> = ({
  value,
  title,
  subtitle,
  comparison,
  trend,
  thresholds
}) => {
  const status = getStatusByThresholds(value, thresholds);
  
  return (
    <div className={`billboard billboard--${status}`}>
      <div className="billboard__header">
        <h3>{title}</h3>
        {subtitle && <p>{subtitle}</p>}
      </div>
      
      <div className="billboard__value">
        <span className="value">{formatValue(value)}</span>
        {trend && <TrendIndicator {...trend} />}
      </div>
      
      {comparison && (
        <div className="billboard__comparison">
          <ComparisonChart
            current={value}
            previous={comparison.value}
            label={comparison.period}
          />
        </div>
      )}
    </div>
  );
};
```

### Advanced Visualization Features

#### 1. Real-time Updates

```typescript
class RealtimeVisualizationManager {
  private updateInterval: number = 30000; // 30 seconds
  private subscriptions: Map<string, Subscription> = new Map();
  
  subscribeToUpdates(chartId: string, callback: UpdateCallback): void {
    const subscription = {
      id: chartId,
      callback,
      interval: setInterval(() => {
        this.fetchLatestData(chartId).then(data => {
          callback(this.transformData(data));
        });
      }, this.updateInterval)
    };
    
    this.subscriptions.set(chartId, subscription);
  }
  
  private transformData(rawData: any): ChartUpdate {
    return {
      timestamp: Date.now(),
      data: rawData,
      transition: {
        duration: 750,
        easing: 'easeInOutQuad'
      }
    };
  }
}
```

#### 2. Interactive Legends

```typescript
const InteractiveLegend: React.FC<LegendProps> = ({ series, onToggle }) => {
  const [visibility, setVisibility] = useState<Record<string, boolean>>(
    series.reduce((acc, s) => ({ ...acc, [s.id]: true }), {})
  );
  
  const handleToggle = (seriesId: string) => {
    const newVisibility = {
      ...visibility,
      [seriesId]: !visibility[seriesId]
    };
    setVisibility(newVisibility);
    onToggle(newVisibility);
  };
  
  return (
    <div className="interactive-legend">
      {series.map(s => (
        <LegendItem
          key={s.id}
          series={s}
          visible={visibility[s.id]}
          onClick={() => handleToggle(s.id)}
        />
      ))}
      <LegendActions>
        <button onClick={() => toggleAll(true)}>Show All</button>
        <button onClick={() => toggleAll(false)}>Hide All</button>
      </LegendActions>
    </div>
  );
};
```

#### 3. Drill-down Navigation

```typescript
interface DrillDownConfig {
  levels: Array<{
    type: 'account' | 'cluster' | 'broker' | 'topic';
    aggregation: string;
    navigation: (item: any) => void;
  }>;
  currentLevel: number;
}

const DrillDownChart: React.FC<DrillDownProps> = ({ 
  data, 
  config, 
  currentLevel 
}) => {
  const handleItemClick = (item: ChartItem) => {
    if (currentLevel < config.levels.length - 1) {
      const nextLevel = config.levels[currentLevel + 1];
      nextLevel.navigation(item);
    }
  };
  
  return (
    <>
      <Breadcrumb>
        {config.levels.slice(0, currentLevel + 1).map((level, i) => (
          <BreadcrumbItem
            key={i}
            active={i === currentLevel}
            onClick={() => navigateToLevel(i)}
          >
            {level.type}
          </BreadcrumbItem>
        ))}
      </Breadcrumb>
      
      <Chart
        data={data}
        onItemClick={handleItemClick}
        showDrillDownIndicator={currentLevel < config.levels.length - 1}
      />
    </>
  );
};
```

## Performance Optimization

### Rendering Optimization

```typescript
class VisualizationOptimizer {
  // Virtual scrolling for large datasets
  virtualizeData(data: any[], viewport: Viewport): any[] {
    const visibleRange = this.calculateVisibleRange(data, viewport);
    return data.slice(visibleRange.start, visibleRange.end);
  }
  
  // Level of detail (LOD) rendering
  applyLOD(entities: Entity[], zoomLevel: number): RenderConfig {
    if (zoomLevel < 0.5) {
      return {
        showLabels: false,
        showPatterns: false,
        simplifiedShapes: true
      };
    } else if (zoomLevel < 1) {
      return {
        showLabels: true,
        showPatterns: false,
        simplifiedShapes: false
      };
    } else {
      return {
        showLabels: true,
        showPatterns: true,
        simplifiedShapes: false
      };
    }
  }
  
  // Debounced updates
  debouncedUpdate = debounce((callback: () => void) => {
    requestAnimationFrame(callback);
  }, 16); // 60 FPS
}
```

### Data Aggregation

```typescript
class DataAggregator {
  // Progressive aggregation based on zoom level
  aggregateByZoom(data: DataPoint[], zoomLevel: number): DataPoint[] {
    const bucketSize = this.calculateBucketSize(zoomLevel);
    return this.bucketize(data, bucketSize);
  }
  
  // Smart sampling for large datasets
  smartSample(data: DataPoint[], maxPoints: number): DataPoint[] {
    if (data.length <= maxPoints) return data;
    
    // Use LTTB (Largest Triangle Three Buckets) algorithm
    return this.lttbDownsample(data, maxPoints);
  }
}
```

## Accessibility Features

### Keyboard Navigation

```typescript
const KeyboardNavigableHoneyComb: React.FC = ({ entities }) => {
  const [focusedIndex, setFocusedIndex] = useState(0);
  
  const handleKeyDown = (e: KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowRight':
        setFocusedIndex(i => Math.min(i + 1, entities.length - 1));
        break;
      case 'ArrowLeft':
        setFocusedIndex(i => Math.max(i - 1, 0));
        break;
      case 'Enter':
      case ' ':
        selectEntity(entities[focusedIndex]);
        break;
    }
  };
  
  return (
    <div
      role="grid"
      aria-label="Kafka infrastructure visualization"
      onKeyDown={handleKeyDown}
      tabIndex={0}
    >
      {entities.map((entity, i) => (
        <Hexagon
          key={entity.guid}
          entity={entity}
          focused={i === focusedIndex}
          aria-label={`${entity.name}, ${entity.healthStatus}`}
          role="gridcell"
        />
      ))}
    </div>
  );
};
```

### Screen Reader Support

```typescript
const AccessibleChart: React.FC = ({ data, type }) => {
  const description = generateChartDescription(data, type);
  const dataTable = convertToDataTable(data);
  
  return (
    <div role="img" aria-label={description}>
      <Chart data={data} type={type} />
      
      {/* Hidden table for screen readers */}
      <table className="sr-only">
        <caption>{description}</caption>
        <thead>
          <tr>
            {dataTable.headers.map(h => <th key={h}>{h}</th>)}
          </tr>
        </thead>
        <tbody>
          {dataTable.rows.map((row, i) => (
            <tr key={i}>
              {row.map((cell, j) => <td key={j}>{cell}</td>)}
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
};
```

## Best Practices

### 1. Visual Hierarchy
- Use size to indicate importance
- Apply color consistently for status
- Maintain clear grouping and spacing

### 2. Performance
- Implement progressive rendering
- Use appropriate visualization for data volume
- Apply smart aggregation strategies

### 3. Interactivity
- Provide immediate visual feedback
- Support multiple interaction methods
- Enable progressive disclosure

### 4. Accessibility
- Include keyboard navigation
- Provide text alternatives
- Support high contrast modes

### 5. Responsiveness
- Adapt to different screen sizes
- Adjust detail level based on viewport
- Maintain usability on touch devices

## Conclusion

The visualization systems in Message Queues monitoring provide powerful, intuitive ways to understand complex Kafka infrastructure. By combining innovative techniques like the HoneyComb view with traditional charts and modern interaction patterns, users can quickly identify issues and make informed decisions about their message queue systems.