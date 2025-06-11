# Screen Specifications

## Overview

This document provides detailed specifications for each screen in the Message Queues monitoring application, including layout, components, interactions, and data requirements.

## Screen Architecture

```
Message Queues Application
├── Home Dashboard (Landing Page)
├── Summary Dashboard (Account View)
└── MQ Detail Screen (Entity Details)
```

## Home Dashboard Screen

### Screen Overview
- **Route**: `/message-queues/home`
- **Nerdlet ID**: `message-queues.home`
- **Purpose**: Primary landing page displaying all Kafka clusters across accounts

### Screen Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Action Bar                                                 │
│  [+ Add new account]                                        │
├─────────────────────────────────────────────────────────────┤
│  Filter Bar                                                 │
│  [Provider ▼] [Message Queue Type ▼] [Account ▼] [Status ▼]│
│                                      [🔍 Search...]         │
├─────────────────────────────────────────────────────────────┤
│  Additional Filters                                         │
│  [× Clusters: prod-kafka-01] [× Topics: user-events]       │
├─────────────────────────────────────────────────────────────┤
│  Data Table                                                 │
│  ┌─────────┬──────────┬────────┬──────────┬──────────────┐│
│  │ Name    │ Clusters │ Health │ In (B/s) │ Out (B/s)    ││
│  ├─────────┼──────────┼────────┼──────────┼──────────────┤│
│  │ 🟧 Prod │    12    │ 11/12  │ 125 MB/s │ 118 MB/s     ││
│  │ Account │          │   ✓    │    ↗     │    ↗         ││
│  └─────────┴──────────┴────────┴──────────┴──────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### Component Specifications

#### 1. Action Bar
```typescript
interface ActionBarProps {
  onAddAccount: () => void;
}

// Implementation
<Stack className="Add-Account-Stack">
  <StackItem>
    <Button
      type={Button.TYPE.PRIMARY}
      iconType={Button.ICON_TYPE.INTERFACE__SIGN__PLUS}
      onClick={() => navigation.openLauncher('nr-integrations')}
    >
      Add new account
    </Button>
  </StackItem>
</Stack>
```

**Behavior**:
- Click opens New Relic Instant Observability
- Pre-filters to Kafka integrations
- Tracks `add_account_clicked` event

#### 2. Filter Bar Section

##### Provider Filter
```typescript
const providerOptions = [
  { label: 'All Providers', value: 'all' },
  { label: 'AWS MSK', value: 'AWS_MSK' },
  { label: 'Confluent Cloud', value: 'CONFLUENT_CLOUD' }
];
```

##### Account Filter
```typescript
interface AccountFilterProps {
  accounts: Account[];
  selectedAccounts: string[];
  onChange: (accounts: string[]) => void;
}

// Dynamic population
const accountOptions = accounts.map(acc => ({
  label: `${acc.name} (${acc.id})`,
  value: acc.id,
  searchableText: `${acc.name} ${acc.id}`
}));
```

##### Status Filter
```typescript
const statusOptions = [
  { label: 'All', value: 'all' },
  { label: 'Healthy', value: 'healthy' },
  { label: 'Unhealthy', value: 'unhealthy' }
];

// Health calculation
const isHealthy = (cluster) => 
  cluster.activeControllers === 1 && 
  cluster.offlinePartitions === 0;
```

##### Search Bar
```typescript
interface SearchBarProps {
  placeholder: string;
  value: string;
  onChange: (value: string) => void;
  debounceMs?: number;
}

// Features
- Real-time search across all columns
- 300ms debounce
- Case-insensitive matching
- Searches: name, account, provider, cluster names
```

#### 3. Custom Filters Section

##### Add Filter Modal
```typescript
interface AddFilterModalState {
  isOpen: boolean;
  filterType: 'cluster' | 'topic';
  selectedValues: string[];
  availableOptions: FilterOption[];
}

// Filter options fetched via GraphQL
const clusterOptions = await fetchClusterNames(accountIds);
const topicOptions = await fetchTopicNames(selectedClusters);
```

##### Active Filter Pills
```typescript
interface FilterPill {
  id: string;
  type: 'cluster' | 'topic';
  label: string;
  values: string[];
  onRemove: () => void;
}

// Display format
<FilterPill
  label="Clusters"
  values={['prod-kafka-01', 'staging-kafka']}
  onRemove={() => removeFilter('clusters')}
/>
```

#### 4. Data Table

##### Table Configuration
```typescript
const tableColumns: TableColumn[] = [
  {
    key: 'Name',
    header: 'Name',
    width: '25%',
    sortable: true,
    sortType: 'array(string)',
    render: (row) => <NameColumn {...row} />
  },
  {
    key: 'Clusters',
    header: 'Clusters',
    width: '15%',
    align: 'center',
    sortable: true,
    sortType: 'number'
  },
  {
    key: 'Health',
    header: 'Health',
    width: '20%',
    sortable: true,
    sortType: 'array(number)', // [healthy, total]
    render: (row) => <HealthStatus {...row} />
  },
  {
    key: 'Incoming Throughput',
    header: 'Incoming Throughput',
    width: '20%',
    sortable: true,
    sortType: 'number',
    render: (value) => <ThroughputCell value={value} />
  },
  {
    key: 'Outgoing Throughput',
    header: 'Outgoing Throughput',
    width: '20%',
    sortable: true,
    sortType: 'number',
    render: (value) => <ThroughputCell value={value} />
  }
];
```

##### Row Data Structure
```typescript
interface TableRow {
  'Name': string[];              // [accountName, accountId]
  'Clusters': number;            // Total cluster count
  'Health': [number, number];    // [healthy, total]
  'Incoming Throughput': number; // Bytes per second
  'Outgoing Throughput': number; // Bytes per second
  'Provider': string;            // AWS_MSK | CONFLUENT_CLOUD
  'Account Name': string;
  'Account Id': string;
  'Is Metric Stream': boolean;
  'hasError': boolean;
}
```

##### Cell Renderers

**NameColumn Component**:
```typescript
<div className="name-column">
  <ProviderLogo provider={provider} />
  <div className="name-column__text">
    <div className="primary">{accountName}</div>
    <div className="secondary">{accountId}</div>
  </div>
  <IntegrationBadge type={isMetricStream ? 'stream' : 'polling'} />
</div>
```

**HealthStatus Component**:
```typescript
<div className="health-status">
  <HealthBar healthy={healthy} total={total} />
  <span className="health-text">
    {healthy === total ? '✓ All healthy' : `${healthy}/${total} healthy`}
  </span>
</div>
```

**ThroughputCell Component**:
```typescript
<div className="throughput-cell">
  <span className="value">{humanizeBytes(value)}/s</span>
  <TrendIndicator trend={calculateTrend(value, previousValue)} />
</div>
```

### State Management

```typescript
interface HomeScreenState {
  // Filter state
  filters: {
    provider: string[];
    account: string[];
    status: 'all' | 'healthy' | 'unhealthy';
    clusters: string[];
    topics: string[];
    search: string;
  };
  
  // Table state
  sorting: {
    column: string;
    direction: 'asc' | 'desc';
  };
  
  // UI state
  loading: boolean;
  error: Error | null;
  
  // Data
  tableData: TableRow[];
  totalClusters: number;
  totalTopics: number;
}
```

### Data Queries

```typescript
// Main data query
const { data, loading, error } = useNerdGraphQuery({
  query: ALL_KAFKA_TABLE_QUERY,
  variables: {
    awsQuery: buildAWSQuery(filters),
    confluentCloudQuery: buildConfluentQuery(filters),
    facet: 'ACCOUNT_ID',
    orderBy: 'ASC'
  }
});

// Throughput aggregation
const throughputQuery = `
  SELECT 
    sum(bytesInPerSec) as incomingThroughput,
    sum(bytesOutPerSec) as outgoingThroughput
  FROM (
    SELECT 
      average(provider.bytesInPerSec.Average) as bytesInPerSec,
      average(provider.bytesOutPerSec.Average) as bytesOutPerSec
    FROM AwsMskBrokerSample
    FACET provider.clusterName, provider.brokerId
  )
  WHERE provider.accountId = '{accountId}'
`;
```

### User Interactions

#### Row Click Navigation
```typescript
const handleRowClick = (row: TableRow) => {
  trackEvent('table_row_clicked', {
    provider: row.Provider,
    accountId: row['Account Id']
  });
  
  navigation.openStackedNerdlet({
    id: 'message-queues.summary',
    urlState: {
      provider: row.Provider,
      account: row['Account Id'],
      accountName: row['Account Name'],
      filters: currentFilters,
      timeRange: currentTimeRange
    }
  });
};
```

#### Filter Application
```typescript
const applyFilters = (newFilters: Partial<Filters>) => {
  // Update state
  setFilters({ ...filters, ...newFilters });
  
  // Update URL for deep linking
  nerdlet.setUrlState({ filters: { ...filters, ...newFilters } });
  
  // Track filter usage
  trackEvent('filters_applied', newFilters);
  
  // Refresh data
  refetchData();
};
```

### Loading & Error States

#### Loading State
```typescript
<TableSkeleton>
  {[1, 2, 3].map(i => (
    <SkeletonRow key={i}>
      <SkeletonCell width="25%" />
      <SkeletonCell width="15%" />
      <SkeletonCell width="20%" />
      <SkeletonCell width="20%" />
      <SkeletonCell width="20%" />
    </SkeletonRow>
  ))}
</TableSkeleton>
```

#### Empty State
```typescript
<EmptyState
  type={hasFilters ? EmptyState.TYPE.NO_DATA : EmptyState.TYPE.WELCOME}
  title={hasFilters 
    ? "No clusters match your filters" 
    : "Get started with Message Queues"
  }
  description={hasFilters
    ? "Try adjusting your filters or search terms"
    : "Set up Kafka monitoring in minutes"
  }
  action={hasFilters ? (
    <Button onClick={clearFilters}>Clear filters</Button>
  ) : (
    <Button onClick={openIntegrations}>Set up integration</Button>
  )}
/>
```

## Summary Dashboard Screen

### Screen Overview
- **Route**: `/message-queues/summary`
- **Nerdlet ID**: `message-queues.summary`
- **Purpose**: Account-level Kafka infrastructure analytics

### Screen Layout

```
┌─────────────────────────────────────────────────────────────┐
│  Header                                     [Time Range ▼]  │
│  Message Queues > Production Account                        │
│  AWS MSK • us-east-1 • Metric Stream                      │
├─────────────────────────────────────────────────────────────┤
│  Summary Billboards                                         │
│  ┌──────────┬──────────┬──────────┬──────────┬──────────┐ │
│  │ Clusters │Unhealthy │  Topics  │Partitions│ Brokers  │ │
│  │    12    │    1     │   856    │  4,284   │    48    │ │
│  └──────────┴──────────┴──────────┴──────────┴──────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Time Series Charts                                         │
│  ┌─────────────────────────┬─────────────────────────────┐ │
│  │ Throughput by Cluster   │ Message Rate by Cluster     │ │
│  │ [Line Chart]           │ [Area Chart]               │ │
│  └─────────────────────────┴─────────────────────────────┘ │
│  ┌─────────────────────────┬─────────────────────────────┐ │
│  │ Top Topics by Activity  │ Broker Distribution        │ │
│  │ [Bar Chart]            │ [Column Chart]             │ │
│  └─────────────────────────┴─────────────────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│  Kafka Navigator                           [View Controls]  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  HoneyComb Visualization                             │   │
│  │  ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡                       │   │
│  │  ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡ ⬡                       │   │
│  └─────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────┤
│  Topics Table                                               │
│  [Expandable table with topic details]                      │
└─────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. Page Header
```typescript
interface PageHeaderProps {
  accountName: string;
  accountId: string;
  provider: Provider;
  region?: string;
  integrationType: 'polling' | 'metricStream';
  onTimeRangeChange: (timeRange: TimeRange) => void;
}

<PageHeader>
  <Breadcrumb>
    <Link to="/message-queues">Message Queues</Link>
    <span>{accountName}</span>
  </Breadcrumb>
  
  <EntityInfo>
    <h1>{accountName}</h1>
    <Metadata>
      <ProviderBadge provider={provider} />
      <RegionBadge region={region} />
      <IntegrationBadge type={integrationType} />
    </Metadata>
  </EntityInfo>
  
  <TimePicker
    defaultValue="1 hour"
    onChange={onTimeRangeChange}
  />
</PageHeader>
```

#### 2. Summary Billboards

```typescript
const billboardConfigs = [
  {
    id: 'total-clusters',
    title: 'Total Clusters',
    query: TOTAL_CLUSTERS_QUERY(provider, account),
    onClick: () => setNavigatorFilter({ show: 'cluster' })
  },
  {
    id: 'unhealthy-clusters',
    title: 'Unhealthy Clusters',
    query: UNHEALTHY_CLUSTERS_QUERY(provider, account),
    status: value => value === 0 ? 'healthy' : 'critical',
    onClick: () => setNavigatorFilter({ 
      show: 'cluster', 
      health: 'unhealthy' 
    })
  },
  {
    id: 'total-topics',
    title: 'Total Topics',
    query: TOTAL_TOPICS_QUERY(provider, account),
    onClick: () => setNavigatorFilter({ show: 'topic' })
  },
  {
    id: 'total-partitions',
    title: 'Total Partitions',
    query: TOTAL_PARTITIONS_QUERY(provider, account)
  },
  {
    id: 'total-brokers',
    title: 'Total Brokers',
    query: TOTAL_BROKERS_QUERY(provider, account),
    onClick: () => setNavigatorFilter({ show: 'broker' })
  }
];
```

#### 3. Time Series Charts

##### Chart Grid Configuration
```typescript
const chartGrid = [
  {
    row: 1,
    charts: [
      {
        id: 'throughput-by-cluster',
        title: 'Incoming Throughput by Cluster',
        type: 'line',
        width: 6,
        query: THROUGHPUT_BY_CLUSTER_QUERY,
        yAxis: { label: 'Bytes/sec', formatter: humanizeBytes }
      },
      {
        id: 'message-rate-by-cluster',
        title: 'Message Rate by Cluster',
        type: 'area',
        width: 6,
        query: MESSAGE_RATE_QUERY,
        stacked: true
      }
    ]
  },
  {
    row: 2,
    charts: [
      {
        id: 'top-topics',
        title: 'Top 20 Topics by Throughput',
        type: 'bar',
        width: 6,
        query: TOP_TOPICS_QUERY,
        horizontal: true
      },
      {
        id: 'broker-distribution',
        title: 'Brokers by Cluster',
        type: 'column',
        width: 6,
        query: BROKER_DISTRIBUTION_QUERY
      }
    ]
  }
];
```

#### 4. Kafka Navigator Section

```typescript
interface NavigatorState {
  view: 'cluster' | 'broker' | 'topic';
  metric: 'health' | 'alerts' | 'throughput';
  groupBy: 'type' | 'cluster' | 'provider';
  selectedEntities: string[];
  filters: NavigatorFilter[];
}

<NavigatorSection>
  <NavigatorControlBar
    view={navigatorState.view}
    metric={navigatorState.metric}
    groupBy={navigatorState.groupBy}
    onViewChange={handleViewChange}
    onMetricChange={handleMetricChange}
    onGroupByChange={handleGroupByChange}
  />
  
  <NavigatorContent>
    <HoneyCombView
      entities={filteredEntities}
      viewConfig={navigatorState}
      onEntitySelect={handleEntitySelect}
      onEntityHover={handleEntityHover}
    />
    
    <LegendsPanel
      entityCounts={entityCounts}
      healthDistribution={healthDistribution}
      selectedCount={selectedEntities.length}
    />
  </NavigatorContent>
</NavigatorSection>
```

#### 5. Topics Table

```typescript
interface TopicsTableConfig {
  columns: TableColumn[];
  pageSize: number;
  sortable: boolean;
  expandable: boolean;
}

const topicsTableColumns = [
  { key: 'name', title: 'Topic Name', sortable: true },
  { key: 'partitions', title: 'Partitions', sortable: true },
  { key: 'bytesIn', title: 'Bytes In/sec', sortable: true },
  { key: 'bytesOut', title: 'Bytes Out/sec', sortable: true },
  { key: 'messages', title: 'Messages/sec', sortable: true },
  { key: 'lag', title: 'Consumer Lag', sortable: true },
  { key: 'health', title: 'Health', sortable: true }
];

<CollapsibleSection
  title="Topics"
  defaultExpanded={false}
  badge={`${topicCount} topics`}
>
  <TopicsTable
    data={topicsData}
    columns={topicsTableColumns}
    onRowClick={handleTopicClick}
    loading={topicsLoading}
  />
</CollapsibleSection>
```

### State Management

```typescript
interface SummaryScreenState {
  // Context
  provider: Provider;
  account: string;
  accountName: string;
  timeRange: TimeRange;
  
  // Navigator state
  navigatorView: 'cluster' | 'broker' | 'topic';
  navigatorMetric: 'health' | 'alerts' | 'throughput';
  selectedEntities: string[];
  
  // UI state
  expandedSections: {
    charts: boolean;
    navigator: boolean;
    topics: boolean;
  };
  
  // Data
  summaryMetrics: SummaryMetrics;
  chartData: ChartData[];
  entities: Entity[];
  topicsData: Topic[];
}
```

## MQ Detail Screen

### Screen Overview
- **Route**: `/message-queues/mq-detail`
- **Nerdlet ID**: `message-queues.mq-detail`
- **Purpose**: Detailed entity view with navigation coordination

### Screen Functions

```typescript
interface MQDetailScreenProps {
  // URL state
  provider: Provider;
  account: string;
  entityGuid?: string;
  entityType?: EntityType;
  
  // Navigation state
  fromScreen?: 'home' | 'summary';
  returnUrl?: string;
}

// Main coordination logic
const MQDetailScreen = (props: MQDetailScreenProps) => {
  // Validate access
  const hasAccess = useAccountAccess(props.account);
  if (!hasAccess) return <AccessDenied />;
  
  // Route to appropriate view
  if (props.entityGuid) {
    return <EntityDetailView entityGuid={props.entityGuid} />;
  }
  
  // Default to summary view
  return <Summary {...props} />;
};
```

### Entity Detail Panel

```typescript
interface EntityDetailPanel {
  entity: Entity;
  metrics: EntityMetrics;
  relationships: Relationship[];
  onClose: () => void;
}

<EntityDetailPanel>
  <PanelHeader>
    <EntityTitle>{entity.name}</EntityTitle>
    <EntityType>{entity.type}</EntityType>
    <CloseButton onClick={onClose} />
  </PanelHeader>
  
  <PanelBody>
    <MetricsSection metrics={metrics} />
    <RelationshipsSection relationships={relationships} />
    <HistoricalTrends entity={entity} />
  </PanelBody>
  
  <PanelFooter>
    <Button onClick={navigateToDashboard}>View Dashboard</Button>
    <Button onClick={configureAlerts}>Configure Alerts</Button>
  </PanelFooter>
</EntityDetailPanel>
```

## Navigation Flows

### Home to Summary
```typescript
Home Dashboard 
  → Click row
  → Open Summary with context
  → Show account-level view
```

### Summary to Entity
```typescript
Summary Dashboard
  → Click entity in Navigator
  → Open Entity Detail Panel
  → Show entity metrics
```

### Deep Linking
```typescript
Direct URL
  → Parse parameters
  → Validate access
  → Load appropriate view
```

## Responsive Design

### Breakpoints
```scss
$breakpoints: (
  mobile: 480px,
  tablet: 768px,
  desktop: 1024px,
  wide: 1440px
);
```

### Mobile Adaptations
- Stack filters vertically
- Simplify table columns
- Full-screen navigator
- Touch-optimized controls

## Performance Considerations

### Data Loading
- Progressive loading for large datasets
- Virtual scrolling in tables
- Lazy loading for charts
- Optimistic UI updates

### Caching Strategy
- Cache entity data for 5 minutes
- Cache metrics for 1 minute
- Cache filter options for session
- Invalidate on user actions

## Accessibility

### Keyboard Navigation
- Tab through all controls
- Enter to select/activate
- Escape to close panels
- Arrow keys in tables

### Screen Reader Support
- Proper ARIA labels
- Role attributes
- Live regions for updates
- Descriptive text

## Best Practices

1. **Consistent Navigation**: Maintain context across screens
2. **Progressive Disclosure**: Show details on demand
3. **Performance First**: Optimize for large datasets
4. **Error Recovery**: Graceful degradation
5. **User Feedback**: Clear loading and error states

## Conclusion

These screen specifications provide a comprehensive blueprint for implementing the Message Queues monitoring interface, ensuring consistency, usability, and performance across all user interactions.