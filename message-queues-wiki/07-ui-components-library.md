# UI Components Library

## Overview

The Message Queues application utilizes a comprehensive library of reusable UI components built on top of the New Relic One SDK. This document details each component, its purpose, props, and usage patterns.

## Component Architecture

### Component Organization

```
common/components/
├── Core Components/
│   ├── EntityNavigator/      # Main visualization component
│   ├── filter-bar/           # Advanced filtering system
│   └── honey-comb-view/      # Hexagonal grid visualization
├── Table Components/
│   ├── home-table/           # Main data table
│   ├── topics-table/         # Topic-specific table
│   └── entity-list-table/    # Generic entity table
├── Chart Components/
│   ├── summary-chart/        # Billboard metrics
│   └── home-billboards/      # Dashboard billboards
├── Filter Components/
│   ├── home-search-bar/      # Global search
│   ├── home-add-filter/      # Custom filters
│   └── home-edit-filter/     # Filter editing
└── Utility Components/
    ├── custom-tooltip/       # Rich tooltips
    ├── name-column/          # Entity name display
    └── lazy/                 # Lazy loading wrapper
```

## Core Components

### EntityNavigator Component (Actual Implementation)

The central visualization component for exploring Kafka infrastructure.

```typescript
// From EntityNavigator.tsx - Actual implementation
import {
  HeadingText,
  Card,
  CardHeader,
  CardBody,
  useNerdletState,
  useNerdGraphQuery,
  EntitySearchQuery,
  InlineMessage,
  Stack,
  StackItem,
} from 'nr1';
import React, { useState } from 'react';
import { mergeFiltersWithAND } from '@datanerd/fsi-high-density-view';
import SORT_BY from '@datanerd/fsi-high-density-view/dist/commonjs/common/shared/enums/SORT_BY';
import { OTHERS_GROUP_ID } from '@datanerd/fsi-high-density-view/dist/commonjs/common/shared/queries/EntitiesQuery';

import { ENTITY_GROUP_QUERY, getGroupFilter } from '../entity-group-query';
import { HoneyCombView } from '../honey-comb-view';
import * as FilterQuery from '../../utils/query-utils';
import { CONFLUENT_CLOUD_PROVIDER } from '../../config/constants';
import { useFetchEntityMetrics } from '../../hooks/use-fetch-entity-metrics';
import {
  ALERT_SEVERITY_ORDER,
  prepareEntityGroups,
} from '../../utils/data-utils';

export function EntityNavigator({ filterSet }: { filterSet: any[] }) {
  // State management
  const [filters = { show: 'cluster', metric: 'health' }, setFilters]: [
    any,
    any,
  ] = useState();
  const [{ item = { 'Account Id': '' } }]: [any, any] = useNerdletState();
  
  // Fetch entity metrics
  const {
    loading: entityMetricsLoading,
    e: entityMetricsError,
    EntityMetrics,
  } = useFetchEntityMetrics({
    item,
    show: filters.show,
    groupBy: filters.groupBy,
    filterSet,
  });

  // Prepare entity groups
  let entityGroups = prepareEntityGroups(
    EntityMetrics,
    item.Provider,
    filters.show,
    filters.groupBy,
  );

  // Build provider-specific query filters
  const queryFilters: string = [
    item?.Provider === CONFLUENT_CLOUD_PROVIDER
      ? (FilterQuery as { [key: string]: any })[
          `CONFLUENT_CLOUD_QUERY_FILTER_${filters.show.toUpperCase()}`
        ]
      : (FilterQuery as { [key: string]: any })[
          `AWS_${filters.show.toUpperCase()}_QUERY_FILTER`
        ],
    `tags.accountId = '${item['Account Id']}'`,
  ].join(' AND ');

  // Entity search query
  const { loading, error, data } = useNerdGraphQuery({
    query: ENTITY_GROUP_QUERY,
    variables: {
      filters: mergeFiltersWithAND(queryFilters, getGroupFilter(OTHERS_GROUP_ID, 'Kafka Entity Type')),
      sortBy: EntitySearchQuery.SORT_TYPE.NAME || SORT_BY.NAME,
      caseSensitiveTagMatching: false,
    },
    attributionHeaders: {
      component: 'Message - Queues - Entity - Group - Query',
      componentId: 'message-queues-entity-group-query',
    },
  });

  // Map alert severity to entities
  entityGroups = entityGroups.map((group) => {
    const counts = group.counts.map((count: any) => {
      const entityGroup = data?.actor?.entitySearch?.results?.entities.find(
        (entity: any) => entity.name === count[filters.show],
      );

      return {
        ...count,
        type:
          filters.metric === 'alert-status'
            ? entityGroup?.alertSeverity || 'NOT_CONFIGURED'
            : count.type,
      };
    });
    
    return {
      ...group,
      counts:
        filters.metric === 'alert-status'
          ? counts.sort((a: any, b: any) => {
              const aIndex = ALERT_SEVERITY_ORDER.indexOf(a.type);
              const bIndex = ALERT_SEVERITY_ORDER.indexOf(b.type);
              return aIndex - bIndex;
            })
          : counts,
    };
  });

  return (
    <div className="entity-navigator">
      <NavigatorControlBar
        filters={filters}
        setFilters={setFilters}
      />
      <HoneyCombView
        entityGroups={entityGroups}
        filters={filters}
        loading={loading || entityMetricsLoading}
        error={error || entityMetricsError}
      />
    </div>
  );
}
```

**Key Features**:
- Interactive hexagonal visualization
- Multiple view modes (cluster/broker/topic)
- Real-time metric display
- Entity selection and navigation

### HoneyCombView Component (Actual Implementation)

Renders the hexagonal grid visualization using high-density view library.

```typescript
// From honey-comb-view/index.tsx - Actual implementation
import React, { useMemo } from 'react';
import { Stack, StackItem, HeadingText, InlineMessage } from 'nr1';
import GridViewVirtualized from '@datanerd/fsi-high-density-view/dist/commonjs/components/GridViewVirtualized';
import { getEntityGroupHexType } from '@datanerd/fsi-high-density-view/dist/commonjs/data-sources/EntityDataSource';

export default function HoneyCombView({
  filters,
  EntityMetrics,
  loading,
  error,
  filterSet,
  entityGroups
}: HoneyCombViewProps) {
  const hexagonContent = useMemo(() => {
    if (loading) {
      return (
        <InlineMessage
          type={InlineMessage.TYPE.INFO}
          title="Loading entities..."
        />
      );
    }

    if (error) {
      return (
        <InlineMessage
          type={InlineMessage.TYPE.CRITICAL}
          title="Error loading entities"
          description={error.message}
        />
      );
    }

    if (!entityGroups || entityGroups.length === 0) {
      return (
        <InlineMessage
          type={InlineMessage.TYPE.INFO}
          title="No entities found"
          description="Try adjusting your filters"
        />
      );
    }

    return (
      <GridViewVirtualized
        entityGroups={entityGroups}
        entityMetrics={EntityMetrics}
        getHexType={filters.metric === 'alert-status' 
          ? getEntityGroupHexType 
          : getHealthScoreHexType
        }
        hexagonContent={(entity) => ({
          title: entity.name,
          value: entity.metrics?.[filters.metric] || 0,
          subvalue: `${entity.count} entities`
        })}
        onEntityClick={handleEntityNavigation}
        useVirtualization={entityGroups.length > 100}
      />
    );
  }, [entityGroups, loading, error, filters, EntityMetrics]);

  return (
    <div className="honey-comb-view">
      <NavigatorControlBar
        filters={filters}
        onFilterChange={handleFilterChange}
      />
      {hexagonContent}
    </div>
  );
}
```

**Visual Features**:
- Hexagonal entity representation
- Dynamic color coding based on metric type
- Entity count display
- Virtualized rendering for performance
- Loading and error states
- Empty state handling

## Filter System Components

### CustomFilterBar Component (Actual Implementation)

The main filter bar providing predefined and custom filtering capabilities.

```typescript
// From filter-bar/index.tsx - Actual implementation with EditableValueFilter
import React, { useEffect, useState } from 'react';
import { HeadingText, Stack, StackItem } from 'nr1';
import { AVAILABLE_FILTER_OPTIONS } from '../../config/constants';
import EditableValueFilter from './EditableValueFilter';

export default function CustomFilterBar({
  predefinedFilters,
  accountIds,
  attributeListType,
  onFilterChange,
  filterBarFilters,
  AdditionalSearchComponent,
  customAddFilter,
  additionalFilters,
  homeFilterData,
}: CustomFilterBarProps) {
  const [filterChangeEvent, setFilterChangeEvent] = useState(null);
  const [prevFilterBarFilters, setPrevFilterBarFilters] = useState([]);

  useEffect(() => {
    if (filterBarFilters && filterBarFilters !== prevFilterBarFilters) {
      setPrevFilterBarFilters(filterBarFilters);
      // Process filter changes
      processFilterUpdate(filterBarFilters);
    }
  }, [filterBarFilters]);

  const processFilterUpdate = (filters) => {
    const processedFilters = filters.map(filter => ({
      ...filter,
      // Convert filter format for query building
      value: Array.isArray(filter.value) 
        ? filter.value.map(v => `'${v}'`).join(', ')
        : `'${filter.value}'`
    }));
    
    onFilterChange(processedFilters);
  };

  return (
    <div className="custom-filter-bar">
      <Stack 
        directionType={Stack.DIRECTION_TYPE.HORIZONTAL} 
        gapType={Stack.GAP_TYPE.NORMAL}
        className="filter-bar-container"
      >
        {/* Predefined filters using EditableValueFilter */}
        {predefinedFilters.map((filter, index) => (
          <StackItem key={filter.key}>
            <EditableValueFilter
              title={filter.title}
              options={filter.options || AVAILABLE_FILTER_OPTIONS[filter.key]}
              value={filterBarFilters?.find(f => f.key === filter.key)?.value}
              onChange={(value) => handleFilterChange(filter.key, value)}
              multiSelect={filter.multiSelect}
              placeholder={filter.placeholder}
            />
          </StackItem>
        ))}
        
        {/* Search component */}
        {AdditionalSearchComponent && (
          <StackItem grow>
            {AdditionalSearchComponent}
          </StackItem>
        )}
        
        {/* Custom filters */}
        {customAddFilter && (
          <StackItem>
            {customAddFilter}
          </StackItem>
        )}
      </Stack>
      
      {/* Additional filter display */}
      {additionalFilters && (
        <div className="additional-filters-container">
          {additionalFilters}
        </div>
      )}
    </div>
  );
}
```

### EditableValueFilter Component

```typescript
// From filter-bar/EditableValueFilter.tsx
import React, { useState } from 'react';
import { Button, Dropdown, DropdownItem, TextField } from 'nr1';

export default function EditableValueFilter({
  title,
  options,
  value,
  onChange,
  multiSelect = false,
  placeholder = 'Select value',
  allowCustomValue = true
}: EditableValueFilterProps) {
  const [isEditing, setIsEditing] = useState(false);
  const [customValue, setCustomValue] = useState('');

  const handleSelect = (selectedValue) => {
    if (multiSelect) {
      const currentValues = value || [];
      const newValues = currentValues.includes(selectedValue)
        ? currentValues.filter(v => v !== selectedValue)
        : [...currentValues, selectedValue];
      onChange(newValues);
    } else {
      onChange(selectedValue);
      setIsEditing(false);
    }
  };

  const handleCustomValue = () => {
    if (customValue.trim()) {
      onChange(multiSelect ? [...(value || []), customValue] : customValue);
      setCustomValue('');
      setIsEditing(false);
    }
  };

  return (
    <div className="editable-value-filter">
      <Dropdown 
        title={title}
        label={value ? (Array.isArray(value) ? value.join(', ') : value) : placeholder}
      >
        {options.map(option => (
          <DropdownItem key={option.value} onClick={() => handleSelect(option.value)}>
            {multiSelect && value?.includes(option.value) && '✓ '}
            {option.label}
          </DropdownItem>
        ))}
        
        {allowCustomValue && (
          <>  
            <DropdownItem onClick={() => setIsEditing(true)}>
              + Add custom value
            </DropdownItem>
            
            {isEditing && (
              <div className="custom-value-input">
                <TextField
                  value={customValue}
                  onChange={e => setCustomValue(e.target.value)}
                  placeholder="Enter custom value"
                />
                <Button onClick={handleCustomValue}>Add</Button>
              </div>
            )}
          </>
        )}
      </Dropdown>
      
      {value && (
        <Button 
          type={Button.TYPE.PLAIN}
          iconType={Button.ICON_TYPE.INTERFACE__OPERATIONS__CLOSE}
          onClick={() => onChange(null)}
        />
      )}
    </div>
  );
}
```

### HomeSearchBar Component

Global search functionality with debouncing.

```typescript
// common/components/home-search-bar/index.tsx
interface HomeSearchBarProps {
  handleSearchChange: (event: React.ChangeEvent<HTMLInputElement>) => void;
  placeholder?: string;
  value?: string;
}

export default function HomeSearchBar({
  handleSearchChange,
  placeholder = "Search clusters, accounts, or providers...",
  value = ""
}: HomeSearchBarProps) {
  const [searchTerm, setSearchTerm] = useState(value);
  const debouncedSearch = useDebounce(searchTerm, 300);
  
  useEffect(() => {
    if (debouncedSearch !== value) {
      handleSearchChange({
        target: { value: debouncedSearch }
      } as React.ChangeEvent<HTMLInputElement>);
    }
  }, [debouncedSearch]);
  
  return (
    <TextField
      type={TextField.TYPE.SEARCH}
      placeholder={placeholder}
      value={searchTerm}
      onChange={(e) => setSearchTerm(e.target.value)}
      className="home-search-bar"
      autoFocus
    />
  );
}
```

### HomeAddFilter Component

Custom filter configuration interface.

```typescript
// common/components/home-add-filter/index.tsx
interface HomeAddFilterProps {
  onSubmit: (type: string, values: string) => void;
  counts: {
    totalClusters: number;
    totalTopics: number;
  };
  accountIds: string;
  addedFilters: {
    [HOME_CLUSTER_FILTER_TEXT]: string;
    [HOME_TOPIC_FILTER_TEXT]: string;
  };
  filterClusterData: {
    topicClusters: string;
    clusterFilterClusters: string;
  };
}

export default function HomeAddFilter({
  onSubmit,
  counts,
  accountIds,
  addedFilters,
  filterClusterData
}: HomeAddFilterProps) {
  const [isOpen, setIsOpen] = useState(false);
  const [filterType, setFilterType] = useState<'cluster' | 'topic'>('cluster');
  const [selectedValues, setSelectedValues] = useState<string[]>([]);
  
  return (
    <>
      <Button
        type={Button.TYPE.NORMAL}
        iconType={Button.ICON_TYPE.INTERFACE__SIGN__PLUS}
        onClick={() => setIsOpen(true)}
      >
        Add Filter
      </Button>
      
      <Modal hidden={!isOpen} onClose={() => setIsOpen(false)}>
        <HeadingText>Add Custom Filter</HeadingText>
        
        <RadioGroup
          value={filterType}
          onChange={setFilterType}
        >
          <Radio value="cluster">
            Filter by Clusters ({counts.totalClusters})
          </Radio>
          <Radio value="topic">
            Filter by Topics ({counts.totalTopics})
          </Radio>
        </RadioGroup>
        
        <MultiSelect
          placeholder={`Select ${filterType}s...`}
          value={selectedValues}
          onChange={setSelectedValues}
          items={getFilterOptions(filterType)}
        />
        
        <Button
          type={Button.TYPE.PRIMARY}
          onClick={() => {
            onSubmit(filterType, selectedValues.join(','));
            setIsOpen(false);
          }}
        >
          Apply Filter
        </Button>
      </Modal>
    </>
  );
}
```

## Table Components

### HomeTable Component (Actual Implementation)

The main data table displaying Kafka accounts and clusters.

```typescript
// From home-table/index.tsx - Actual implementation
import React from 'react';
import { navigation, Table, TableHeader, TableHeaderCell, TableRow, TableRowCell } from 'nr1';
import ClusterCount from '../cluster-count';
import { humanizeNumber } from '../../utils/humanize';
import NameColumn from '../name-column';
import SummaryMetricCell from '../summary-metric-cell';

export default function HomeTable({
  tableData,
  loading,
  topicsLoading,
  sortedItems,
  sorting,
  totalClusterItems,
  setSortedItems,
  setSorting,
  setTotalClusterItems,
  finalClusterFilters,
  nr1,
}) {
  const _onClickTable = (item) => {
    navigation.openStackedNerdlet({
      id: 'message-queues.mq-detail',
      urlState: {
        providerId: item.Provider,
        accountId: item['Account Id'],
        accountName: item.Name,
        isMetricStream: item['Is Metric Stream'],
      },
    });
  };

  const _onSort = (sortingData) => {
    const { items } = sortingData;
    setSortedItems([...items]);
    setSorting(sortingData);
  };

  return (
    <Table 
      items={sortedItems}
      onSort={_onSort}
      sortingState={sorting}
      className="home-table"
    >
      <TableHeader>
        <TableHeaderCell 
          sortable 
          sortingType={TableHeaderCell.SORTING_TYPE.PRIMARY}
          value={({ item }) => item.Name}
        >
          Name
        </TableHeaderCell>
        
        <TableHeaderCell 
          sortable
          value={({ item }) => item.Clusters}
          alignmentType={TableHeaderCell.ALIGNMENT_TYPE.CENTER}
        >
          Clusters
        </TableHeaderCell>
        
        <TableHeaderCell>
          Health
        </TableHeaderCell>
        
        <TableHeaderCell 
          sortable
          value={({ item }) => item['Incoming Throughput']}
          alignmentType={TableHeaderCell.ALIGNMENT_TYPE.RIGHT}
        >
          Incoming Throughput
        </TableHeaderCell>
        
        <TableHeaderCell 
          sortable
          value={({ item }) => item['Outgoing Throughput']}
          alignmentType={TableHeaderCell.ALIGNMENT_TYPE.RIGHT}
        >
          Outgoing Throughput
        </TableHeaderCell>
      </TableHeader>
      
      {({ item }) => (
        <TableRow onClick={() => _onClickTable(item)}>
          <TableRowCell>
            <NameColumn
              accountName={item.Name}
              accountId={item['Account Id']}
              provider={item.Provider}
              isMetricStream={item['Is Metric Stream']}
            />
          </TableRowCell>
          
          <TableRowCell alignmentType={TableRowCell.ALIGNMENT_TYPE.CENTER}>
            <ClusterCount
              accountName={item.Name}
              accountId={item['Account Id']}
              accountsTableRef={tableData}
              value={item.Clusters}
              provider={item.Provider}
              totalAccountClusters={totalClusterItems}
              isMetricStream={item['Is Metric Stream']}
              setTotalAccountClusters={setTotalClusterItems}
              topicsLoading={topicsLoading}
              finalClusterFilters={finalClusterFilters}
            />
          </TableRowCell>
          
          <TableRowCell>
            <SummaryMetricCell
              healthValue={item.Health}
              loading={loading}
            />
          </TableRowCell>
          
          <TableRowCell alignmentType={TableRowCell.ALIGNMENT_TYPE.RIGHT}>
            <SummaryMetricCell
              loading={loading}
              metricValue={humanizeNumber(
                item['Incoming Throughput'],
                'Per Second',
                'BYTES'
              )}
            />
          </TableRowCell>
          
          <TableRowCell alignmentType={TableRowCell.ALIGNMENT_TYPE.RIGHT}>
            <SummaryMetricCell
              loading={loading}
              metricValue={humanizeNumber(
                item['Outgoing Throughput'],
                'Per Second',
                'BYTES'
              )}
            />
          </TableRowCell>
        </TableRow>
      )}
    </Table>
  );
}
```

### TopicsTable Component

Displays topic-level metrics and information.

```typescript
// common/components/topics-table/index.tsx
interface TopicsTableProps {
  loading: boolean;
  topicsTableData: TopicRowItem[];
  orderBy: string;
  showRelatedEntities: boolean;
  isFromNavigator?: boolean;
}

export default function TopicsTable({
  loading,
  topicsTableData,
  orderBy,
  showRelatedEntities,
  isFromNavigator
}: TopicsTableProps) {
  const columns = [
    {
      key: 'Topic Name',
      title: 'Topic Name',
      sortable: true
    },
    {
      key: 'Incoming Throughput',
      title: 'Bytes In/sec',
      sortable: true,
      render: (value) => humanizeBytes(value)
    },
    {
      key: 'Outgoing Throughput',
      title: 'Bytes Out/sec',
      sortable: true,
      render: (value) => humanizeBytes(value)
    },
    {
      key: 'Message rate',
      title: 'Messages/sec',
      sortable: true,
      render: (value) => humanizeNumber(value)
    }
  ];
  
  if (showRelatedEntities) {
    columns.push({
      key: 'APM entity',
      title: 'Related Applications',
      render: (row) => <APMEntityLinks guids={row.apmGuids} />
    });
  }
  
  return (
    <EntityListTable
      columns={columns}
      data={topicsTableData}
      loading={loading}
      emptyMessage="No topics found"
      onRowClick={isFromNavigator ? handleNavigatorClick : undefined}
    />
  );
}
```

## Chart Components

### SummaryChart Component

Billboard-style metric display with optional time series.

```typescript
// common/components/summary-chart/SummaryChart.tsx
interface SummaryChartProps {
  title: string;
  value: number | string;
  subtitle?: string;
  trend?: TrendData;
  comparison?: ComparisonData;
  status?: 'healthy' | 'warning' | 'critical';
  onClick?: () => void;
  loading?: boolean;
  error?: Error;
  tooltip?: string;
}

export default function SummaryChart({
  title,
  value,
  subtitle,
  trend,
  comparison,
  status,
  onClick,
  loading,
  error,
  tooltip
}: SummaryChartProps) {
  if (loading) {
    return <BillboardSkeleton />;
  }
  
  if (error) {
    return <BillboardError error={error} />;
  }
  
  const formattedValue = formatValue(value);
  const statusClass = status ? `summary-chart--${status}` : '';
  
  return (
    <Card 
      className={`summary-chart ${statusClass}`}
      onClick={onClick}
    >
      <CardHeader 
        title={title}
        subtitle={subtitle}
        additionalInfo={
          tooltip && <Tooltip text={tooltip} />
        }
      />
      
      <CardBody>
        <div className="summary-chart__value">
          {formattedValue}
        </div>
        
        {trend && (
          <TrendIndicator
            direction={trend.direction}
            percentage={trend.percentage}
          />
        )}
        
        {comparison && (
          <ComparisonBar
            current={comparison.current}
            previous={comparison.previous}
          />
        )}
      </CardBody>
    </Card>
  );
}
```

### HomeBillboards Component

Collection of billboard metrics for the home dashboard.

```typescript
// common/components/home-billboards/index.tsx
interface HomeBillboardsProps {
  data: BillboardData[];
  loading: boolean;
  timeRange: TimeRange;
}

export default function HomeBillboards({
  data,
  loading,
  timeRange
}: HomeBillboardsProps) {
  const billboardConfigs = [
    {
      id: 'total-clusters',
      title: 'Total Clusters',
      query: TOTAL_CLUSTERS_QUERY,
      status: (value) => value > 0 ? 'healthy' : 'warning'
    },
    {
      id: 'unhealthy-clusters',
      title: 'Unhealthy Clusters',
      query: UNHEALTHY_CLUSTERS_QUERY,
      status: (value) => value === 0 ? 'healthy' : 'critical'
    },
    {
      id: 'total-topics',
      title: 'Total Topics',
      query: TOTAL_TOPICS_QUERY
    },
    {
      id: 'total-partitions',
      title: 'Total Partitions',
      query: TOTAL_PARTITIONS_QUERY
    },
    {
      id: 'total-brokers',
      title: 'Total Brokers',
      query: TOTAL_BROKERS_QUERY
    }
  ];
  
  return (
    <Grid className="home-billboards">
      {billboardConfigs.map(config => (
        <GridItem key={config.id} columnSpan={3}>
          <VisualizationBillboard
            {...config}
            data={data.find(d => d.id === config.id)}
            loading={loading}
            timeRange={timeRange}
          />
        </GridItem>
      ))}
    </Grid>
  );
}
```

## Utility Components

### CustomTooltip Component (Actual Implementation)

Rich tooltip for data visualizations.

```typescript
// From custom-tooltip/index.tsx - Actual implementation
import React from 'react';
import { humanizeNumber } from '../../utils/humanize';
import { addCommasToNumber } from '../../utils/data-utils';

const CustomTooltip = ({ active, payload, label }) => {
  if (active && payload && payload.length) {
    // Extract metric information
    const metricData = payload[0];
    const value = metricData.value;
    const metricName = metricData.dataKey || metricData.name;
    
    // Determine formatting based on metric type
    const formatValue = () => {
      if (metricName.toLowerCase().includes('bytes')) {
        return humanizeNumber(value, 'Per Second', 'BYTES');
      }
      if (metricName.toLowerCase().includes('messages')) {
        return `${addCommasToNumber(value)} msg/s`;
      }
      if (metricName.toLowerCase().includes('percent')) {
        return `${value.toFixed(2)}%`;
      }
      return addCommasToNumber(value);
    };

    return (
      <div className="custom-tooltip">
        <div className="custom-tooltip__header">
          {label}
        </div>
        <div className="custom-tooltip__content">
          <div className="custom-tooltip__metric">
            <span className="custom-tooltip__metric-name">
              {metricName}:
            </span>
            <span className="custom-tooltip__metric-value">
              {formatValue()}
            </span>
          </div>
          {payload.length > 1 && payload.slice(1).map((entry, index) => (
            <div key={index} className="custom-tooltip__metric">
              <span className="custom-tooltip__metric-name">
                {entry.name}:
              </span>
              <span className="custom-tooltip__metric-value">
                {addCommasToNumber(entry.value)}
              </span>
            </div>
          ))}
        </div>
      </div>
    );
  }
  return null;
};

export default CustomTooltip;
```

### Tooltip Styles

```scss
// From custom-tooltip/styles.scss
.custom-tooltip {
  background-color: #ffffff;
  border: 1px solid #e3e4e4;
  border-radius: 3px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  padding: 12px;
  min-width: 200px;
  
  &__header {
    font-size: 12px;
    color: #8e9494;
    margin-bottom: 8px;
    font-weight: 600;
  }
  
  &__content {
    font-size: 14px;
  }
  
  &__metric {
    display: flex;
    justify-content: space-between;
    margin-bottom: 4px;
    
    &:last-child {
      margin-bottom: 0;
    }
  }
  
  &__metric-name {
    color: #464e4e;
    margin-right: 8px;
  }
  
  &__metric-value {
    color: #000d0d;
    font-weight: 600;
  }
}
```

### NameColumn Component (Actual Implementation)

Displays entity name with provider logo and badges.

```typescript
// From name-column/index.tsx - Actual implementation
import React from 'react';
import { Badge } from 'nr1';
import ItemLogo from './ItemLogo';
import { MSK_PROVIDER } from '../../config/constants';

export default function NameColumn({
  accountName,
  accountId,
  provider,
  isMetricStream
}) {
  return (
    <div className="name-column">
      <ItemLogo provider={provider} />
      <div className="name-column__content">
        <div className="name-column__primary">
          {accountName}
        </div>
        <div className="name-column__secondary">
          {accountId}
        </div>
      </div>
      {provider === MSK_PROVIDER && (
        <div className="name-column__badges">
          <Badge 
            type={isMetricStream ? Badge.TYPE.SUCCESS : Badge.TYPE.NORMAL}
          >
            {isMetricStream ? 'Metric Stream' : 'Polling'}
          </Badge>
        </div>
      )}
    </div>
  );
}
```

### ItemLogo Component

```typescript
// From name-column/ItemLogo.tsx
import React from 'react';
import { MSK_PROVIDER, CONFLUENT_CLOUD_PROVIDER } from '../../config/constants';
import awsMskLogo from '../../assets/kafka-dark-mode-logo.svg';
import confluentLogo from '../../assets/confluent-logo.svg';
import defaultIcon from '../../assets/default-icon.png';

export default function ItemLogo({ provider }) {
  const getLogoSrc = () => {
    switch (provider) {
      case MSK_PROVIDER:
        return awsMskLogo;
      case CONFLUENT_CLOUD_PROVIDER:
        return confluentLogo;
      default:
        return defaultIcon;
    }
  };

  const getAltText = () => {
    switch (provider) {
      case MSK_PROVIDER:
        return 'AWS MSK';
      case CONFLUENT_CLOUD_PROVIDER:
        return 'Confluent Cloud';
      default:
        return 'Kafka Provider';
    }
  };

  return (
    <img 
      className="name-column__logo"
      src={getLogoSrc()}
      alt={getAltText()}
      width={32}
      height={32}
    />
  );
}
```

### NameColumn Styles

```scss
// From name-column/styles.scss
.name-column {
  display: flex;
  align-items: center;
  gap: 12px;
  
  &__logo {
    flex-shrink: 0;
  }
  
  &__content {
    flex: 1;
    min-width: 0;
  }
  
  &__primary {
    font-size: 14px;
    font-weight: 600;
    color: #000d0d;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  
  &__secondary {
    font-size: 12px;
    color: #8e9494;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  
  &__badges {
    flex-shrink: 0;
  }
}
```

### Lazy Component

Lazy loading wrapper for code splitting.

```typescript
// common/components/lazy/index.tsx
interface LazyComponentProps {
  loader: () => Promise<{ default: React.ComponentType<any> }>;
  fallback?: React.ReactNode;
  errorBoundary?: boolean;
}

export default function LazyComponent({
  loader,
  fallback = <Spinner />,
  errorBoundary = true
}: LazyComponentProps) {
  const Component = React.lazy(loader);
  
  const content = (
    <React.Suspense fallback={fallback}>
      <Component />
    </React.Suspense>
  );
  
  if (errorBoundary) {
    return (
      <ErrorBoundary fallback={<ErrorState />}>
        {content}
      </ErrorBoundary>
    );
  }
  
  return content;
}
```

## Component Patterns

### Composition Pattern

```typescript
// Composable component structure
const ComplexComponent = () => {
  return (
    <ComponentContainer>
      <ComponentHeader>
        <Title />
        <Actions />
      </ComponentHeader>
      
      <ComponentBody>
        <Sidebar>
          <Filters />
        </Sidebar>
        
        <MainContent>
          <DataDisplay />
        </MainContent>
      </ComponentBody>
      
      <ComponentFooter>
        <Pagination />
      </ComponentFooter>
    </ComponentContainer>
  );
};
```

### Render Props Pattern

```typescript
// Flexible rendering with render props
const DataProvider = ({ render, query }) => {
  const { data, loading, error } = useQuery(query);
  
  if (loading) return <Spinner />;
  if (error) return <ErrorState error={error} />;
  
  return render(data);
};

// Usage
<DataProvider
  query={CLUSTER_QUERY}
  render={(data) => <ClusterView clusters={data} />}
/>
```

### Hook Pattern

```typescript
// Custom hooks for component logic
const useTableData = (initialData) => {
  const [data, setData] = useState(initialData);
  const [sorting, setSorting] = useState(null);
  const [filters, setFilters] = useState([]);
  
  const sortedData = useMemo(() => {
    if (!sorting) return data;
    return sortData(data, sorting);
  }, [data, sorting]);
  
  const filteredData = useMemo(() => {
    if (!filters.length) return sortedData;
    return filterData(sortedData, filters);
  }, [sortedData, filters]);
  
  return {
    data: filteredData,
    sorting,
    filters,
    setSorting,
    setFilters,
    updateData: setData
  };
};
```

## Styling System

### CSS Module Structure

```scss
// Component-specific styles
.home-table {
  &__header {
    background-color: var(--nr1-color-background-secondary);
    border-bottom: 1px solid var(--nr1-color-border);
  }
  
  &__row {
    &:hover {
      background-color: var(--nr1-color-background-hover);
    }
    
    &--selected {
      background-color: var(--nr1-color-selection);
    }
  }
  
  &__cell {
    padding: var(--nr1-spacing-medium);
    
    &--numeric {
      text-align: right;
      font-variant-numeric: tabular-nums;
    }
  }
}
```

### Theme Variables

```scss
// Common theme variables
:root {
  // Colors
  --kafka-healthy: #11a968;
  --kafka-warning: #f5a623;
  --kafka-critical: #d0021b;
  --kafka-unknown: #9b9b9b;
  
  // Spacing
  --kafka-spacing-xs: 4px;
  --kafka-spacing-sm: 8px;
  --kafka-spacing-md: 16px;
  --kafka-spacing-lg: 24px;
  
  // Typography
  --kafka-font-size-sm: 12px;
  --kafka-font-size-md: 14px;
  --kafka-font-size-lg: 16px;
}
```

## Best Practices

### 1. Component Design
- Keep components focused and single-purpose
- Use composition over inheritance
- Implement proper prop validation

### 2. Performance
- Memoize expensive computations
- Use virtual scrolling for large lists
- Implement lazy loading for heavy components

### 3. Accessibility
- Include proper ARIA labels
- Ensure keyboard navigation
- Provide focus indicators

### 4. Testing
- Write unit tests for logic
- Implement component tests
- Use snapshot testing for UI

### 5. Documentation
- Document all props with TypeScript
- Include usage examples
- Maintain storybook stories

## Conclusion

The UI component library provides a robust foundation for building the Message Queues monitoring interface. By following established patterns and best practices, developers can create consistent, performant, and maintainable UI components.