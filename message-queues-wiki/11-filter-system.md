# Filter System

## Overview

The Message Queues application implements a sophisticated filtering system that enables users to efficiently navigate and analyze their Kafka infrastructure. This document provides comprehensive details about the filter architecture, implementation, and usage patterns.

## Filter Architecture

### Filter System Components

```
┌─────────────────────────────────────────────────────────┐
│                   Filter System                         │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────────────┐   │
│  │ Predefined      │    │ Custom Filters          │   │
│  │ Filters         │    │ • Cluster Selection     │   │
│  │ • Provider      │    │ • Topic Selection       │   │
│  │ • Account       │    │ • Tag-based Filters     │   │
│  │ • Status        │    └─────────────────────────┘   │
│  │ • Type          │                                   │
│  └─────────────────┘    ┌─────────────────────────┐   │
│                         │ Search Filters          │   │
│  ┌─────────────────┐    │ • Global Search        │   │
│  │ Filter Engine   │◄───┤ • Entity Name Search   │   │
│  │ • Query Builder │    │ • Metadata Search      │   │
│  │ • Optimizer     │    └─────────────────────────┘   │
│  │ • Validator     │                                   │
│  └─────────────────┘    ┌─────────────────────────┐   │
│                         │ Applied Filters Display │   │
│                         │ • Active Filter Pills   │   │
│                         │ • Clear Actions        │   │
│                         └─────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## Filter Types

### 1. Predefined Filters

#### Provider Filter

```typescript
interface ProviderFilter {
  type: 'provider';
  operators: ['in', 'not_in'];
  values: ['AWS_MSK', 'CONFLUENT_CLOUD'];
  multiple: true;
}

const ProviderFilterComponent: React.FC<FilterProps> = ({ 
  value, 
  onChange 
}) => {
  const options = [
    { label: 'All Providers', value: 'all' },
    { label: 'AWS MSK', value: 'AWS_MSK', icon: 'aws-logo' },
    { label: 'Confluent Cloud', value: 'CONFLUENT_CLOUD', icon: 'confluent-logo' }
  ];
  
  return (
    <MultiSelectDropdown
      label="Provider"
      options={options}
      value={value}
      onChange={onChange}
      placeholder="Select providers..."
    />
  );
};
```

#### Account Filter

```typescript
interface AccountFilter {
  type: 'account';
  operators: ['in', 'not_in'];
  dynamic: true; // Values fetched from API
  searchable: true;
  displayFormat: '{name} ({id})';
}

const AccountFilterComponent: React.FC<FilterProps> = ({ 
  accountIds,
  value, 
  onChange 
}) => {
  const { data: accounts, loading } = useAccountData(accountIds);
  
  const options = accounts?.map(account => ({
    label: `${account.name} (${account.id})`,
    value: account.id,
    searchableText: `${account.name} ${account.id}`,
    metadata: account
  }));
  
  return (
    <SearchableMultiSelect
      label="Account"
      options={options}
      value={value}
      onChange={onChange}
      loading={loading}
      placeholder="Search accounts..."
      renderOption={(option) => (
        <AccountOption
          name={option.metadata.name}
          id={option.metadata.id}
          region={option.metadata.region}
        />
      )}
    />
  );
};
```

#### Status Filter

```typescript
interface StatusFilter {
  type: 'status';
  operators: ['equals'];
  values: ['all', 'healthy', 'unhealthy'];
  exclusive: true; // Only one value at a time
}

const StatusFilterComponent: React.FC<FilterProps> = ({ 
  value, 
  onChange,
  entityCounts 
}) => {
  const options = [
    { 
      label: 'All', 
      value: 'all',
      count: entityCounts.total 
    },
    { 
      label: 'Healthy', 
      value: 'healthy',
      count: entityCounts.healthy,
      icon: 'check-circle',
      color: 'green'
    },
    { 
      label: 'Unhealthy', 
      value: 'unhealthy',
      count: entityCounts.unhealthy,
      icon: 'warning-circle',
      color: 'red'
    }
  ];
  
  return (
    <RadioButtonGroup
      label="Status"
      options={options}
      value={value}
      onChange={onChange}
      renderOption={(option) => (
        <StatusOption
          label={option.label}
          count={option.count}
          icon={option.icon}
          color={option.color}
        />
      )}
    />
  );
};
```

### 2. Custom Filters

#### Cluster Filter

```typescript
interface ClusterFilter {
  type: 'cluster';
  operators: ['in', 'not_in', 'contains'];
  dynamic: true;
  hierarchical: true; // Can filter by provider -> cluster
  cascading: true; // Affects topic filter options
}

class ClusterFilterManager {
  // Fetch available clusters
  async fetchClusters(
    accountIds: string[],
    provider?: string
  ): Promise<ClusterOption[]> {
    const query = ngql`
      query GetClusters($query: String!) {
        actor {
          entitySearch(query: $query) {
            results {
              entities {
                guid
                name
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
        }
      }
    `;
    
    const searchQuery = this.buildClusterSearchQuery(accountIds, provider);
    const result = await NerdGraphQuery.query({ 
      query, 
      variables: { query: searchQuery } 
    });
    
    return this.transformToOptions(result.data);
  }
  
  // Build search query with filters
  private buildClusterSearchQuery(
    accountIds: string[],
    provider?: string
  ): string {
    const conditions = [
      `domain IN ('INFRA')`,
      `type IN ('AWSMSKCLUSTER', 'CONFLUENTCLOUDCLUSTER')`
    ];
    
    if (accountIds.length > 0) {
      conditions.push(`tags.accountId IN (${accountIds.join(',')})`);
    }
    
    if (provider) {
      const typeMap = {
        'AWS_MSK': 'AWSMSKCLUSTER',
        'CONFLUENT_CLOUD': 'CONFLUENTCLOUDCLUSTER'
      };
      conditions.push(`type = '${typeMap[provider]}'`);
    }
    
    return conditions.join(' AND ');
  }
}
```

#### Topic Filter

```typescript
interface TopicFilter {
  type: 'topic';
  operators: ['in', 'not_in', 'like'];
  dynamic: true;
  dependsOn: ['cluster']; // Options depend on selected clusters
  searchable: true;
  maxSelections: 100;
}

const TopicFilterComponent: React.FC<TopicFilterProps> = ({ 
  selectedClusters,
  value,
  onChange 
}) => {
  const [searchTerm, setSearchTerm] = useState('');
  const [options, setOptions] = useState<TopicOption[]>([]);
  const [loading, setLoading] = useState(false);
  
  // Fetch topics based on selected clusters
  useEffect(() => {
    if (selectedClusters.length === 0) {
      setOptions([]);
      return;
    }
    
    const fetchTopics = async () => {
      setLoading(true);
      
      const query = ngql`
        query GetTopics($clusterNames: [String!]!) {
          actor {
            entitySearch(
              query: "domain='INFRA' AND type IN ('AWSMSKTOPIC', 'CONFLUENTCLOUDKAFKATOPIC')"
            ) {
              results {
                entities {
                  name
                  guid
                  tags {
                    key
                    values
                  }
                  ... on InfrastructureAwsKafkaTopicEntity {
                    bytesInPerSec
                    bytesOutPerSec
                    messagesPerSec
                  }
                }
              }
            }
          }
        }
      `;
      
      const result = await NerdGraphQuery.query({ query });
      const topics = this.filterTopicsByCluster(
        result.data.actor.entitySearch.results.entities,
        selectedClusters
      );
      
      setOptions(topics);
      setLoading(false);
    };
    
    fetchTopics();
  }, [selectedClusters]);
  
  // Filter options based on search
  const filteredOptions = useMemo(() => {
    if (!searchTerm) return options;
    
    const term = searchTerm.toLowerCase();
    return options.filter(option => 
      option.name.toLowerCase().includes(term) ||
      option.metadata?.tags?.some(tag => 
        tag.toLowerCase().includes(term)
      )
    );
  }, [options, searchTerm]);
  
  return (
    <div className="topic-filter">
      <SearchInput
        value={searchTerm}
        onChange={setSearchTerm}
        placeholder="Search topics..."
        loading={loading}
      />
      
      <VirtualizedMultiSelect
        options={filteredOptions}
        value={value}
        onChange={onChange}
        maxSelections={100}
        height={300}
        renderOption={(option) => (
          <TopicOption
            name={option.name}
            throughput={option.metadata?.throughput}
            messageRate={option.metadata?.messageRate}
          />
        )}
      />
      
      <FilterStats
        total={options.length}
        filtered={filteredOptions.length}
        selected={value.length}
      />
    </div>
  );
};
```

### 3. Search Filters

#### Global Search Implementation

```typescript
interface SearchFilter {
  type: 'search';
  operators: ['contains', 'starts_with', 'ends_with', 'matches'];
  fields: ['name', 'id', 'tags', 'metadata'];
  debounceMs: 300;
  minLength: 2;
}

class SearchFilterEngine {
  private searchIndex: SearchIndex;
  
  constructor() {
    this.searchIndex = new SearchIndex({
      fields: ['name', 'id', 'tags', 'metadata'],
      storeDocuments: true,
      searchOptions: {
        fuzzy: 0.2,
        prefix: true
      }
    });
  }
  
  // Index entities for fast search
  indexEntities(entities: Entity[]): void {
    entities.forEach(entity => {
      this.searchIndex.add({
        id: entity.guid,
        name: entity.name,
        tags: entity.tags.map(t => `${t.key}:${t.values.join(',')}`).join(' '),
        metadata: JSON.stringify(entity.metadata)
      });
    });
  }
  
  // Perform search
  search(query: string, options: SearchOptions = {}): SearchResult[] {
    const results = this.searchIndex.search(query, {
      limit: options.limit || 100,
      threshold: options.threshold || 0.5,
      fields: options.fields || ['name', 'tags']
    });
    
    return results.map(result => ({
      entity: result.document,
      score: result.score,
      matches: result.matches
    }));
  }
  
  // Advanced search with operators
  advancedSearch(filters: AdvancedSearchFilter[]): SearchResult[] {
    let results = this.getAllDocuments();
    
    filters.forEach(filter => {
      results = this.applyFilter(results, filter);
    });
    
    return results;
  }
  
  private applyFilter(
    documents: Document[],
    filter: AdvancedSearchFilter
  ): Document[] {
    switch (filter.operator) {
      case 'contains':
        return documents.filter(doc => 
          doc[filter.field].toLowerCase().includes(filter.value.toLowerCase())
        );
        
      case 'starts_with':
        return documents.filter(doc => 
          doc[filter.field].toLowerCase().startsWith(filter.value.toLowerCase())
        );
        
      case 'matches':
        const regex = new RegExp(filter.value, 'i');
        return documents.filter(doc => regex.test(doc[filter.field]));
        
      default:
        return documents;
    }
  }
}
```

## Filter Processing Pipeline

### Query Building Process

```typescript
class FilterQueryBuilder {
  // Build complete filter query
  buildQuery(filters: AppliedFilter[]): FilterQuery {
    // 1. Validate filters
    const validFilters = this.validateFilters(filters);
    
    // 2. Group by provider
    const groupedFilters = this.groupByProvider(validFilters);
    
    // 3. Build provider-specific queries
    const queries = Object.entries(groupedFilters).map(([provider, filters]) => {
      return this.buildProviderQuery(provider as Provider, filters);
    });
    
    // 4. Combine queries
    return this.combineQueries(queries);
  }
  
  // Validate filter compatibility
  private validateFilters(filters: AppliedFilter[]): AppliedFilter[] {
    const validated: AppliedFilter[] = [];
    
    filters.forEach(filter => {
      // Check for conflicts
      if (this.hasConflict(filter, validated)) {
        console.warn(`Filter conflict detected: ${filter.type}`);
        return;
      }
      
      // Validate values
      if (!this.isValidFilter(filter)) {
        console.warn(`Invalid filter: ${filter.type}`);
        return;
      }
      
      validated.push(filter);
    });
    
    return validated;
  }
  
  // Build provider-specific query
  private buildProviderQuery(
    provider: Provider,
    filters: AppliedFilter[]
  ): ProviderQuery {
    const queryBuilder = this.getQueryBuilder(provider);
    
    // Base query
    let query = queryBuilder.getBaseQuery();
    
    // Apply filters
    filters.forEach(filter => {
      query = queryBuilder.applyFilter(query, filter);
    });
    
    // Optimize query
    query = queryBuilder.optimize(query);
    
    return query;
  }
}
```

### Filter Application Logic

```typescript
class FilterApplicationEngine {
  // Apply filters to data
  applyFilters<T extends FilterableEntity>(
    data: T[],
    filters: AppliedFilter[]
  ): T[] {
    // Start with all data
    let filtered = [...data];
    
    // Apply each filter in sequence
    filters.forEach(filter => {
      filtered = this.applySingleFilter(filtered, filter);
    });
    
    // Apply post-processing
    filtered = this.postProcess(filtered, filters);
    
    return filtered;
  }
  
  // Apply single filter
  private applySingleFilter<T extends FilterableEntity>(
    data: T[],
    filter: AppliedFilter
  ): T[] {
    const filterFunction = this.getFilterFunction(filter);
    return data.filter(item => filterFunction(item, filter));
  }
  
  // Get appropriate filter function
  private getFilterFunction(filter: AppliedFilter): FilterFunction {
    const filterFunctions: Record<string, FilterFunction> = {
      provider: (item, filter) => 
        filter.values.includes(item.provider),
        
      account: (item, filter) => 
        filter.values.includes(item.accountId),
        
      status: (item, filter) => {
        if (filter.value === 'all') return true;
        const isHealthy = this.calculateHealth(item);
        return filter.value === 'healthy' ? isHealthy : !isHealthy;
      },
      
      cluster: (item, filter) => 
        filter.values.includes(item.clusterName),
        
      topic: (item, filter) => 
        filter.values.some(topic => item.topics?.includes(topic)),
        
      search: (item, filter) => {
        const searchTerm = filter.value.toLowerCase();
        return item.name.toLowerCase().includes(searchTerm) ||
               item.id.toLowerCase().includes(searchTerm) ||
               item.tags?.some(tag => 
                 tag.toLowerCase().includes(searchTerm)
               );
      }
    };
    
    return filterFunctions[filter.type] || (() => true);
  }
}
```

## Filter UI Components

### Filter Bar Implementation

```typescript
const FilterBar: React.FC<FilterBarProps> = ({
  predefinedFilters,
  customFilters,
  onFiltersChange,
  entityCounts
}) => {
  const [expandedSections, setExpandedSections] = useState({
    predefined: true,
    custom: false,
    search: true
  });
  
  const [activeFilters, setActiveFilters] = useState<AppliedFilter[]>([]);
  
  // Handle filter change
  const handleFilterChange = useCallback((
    filterType: string,
    values: any
  ) => {
    const newFilters = updateFilters(activeFilters, filterType, values);
    setActiveFilters(newFilters);
    onFiltersChange(newFilters);
  }, [activeFilters, onFiltersChange]);
  
  // Clear all filters
  const clearAllFilters = useCallback(() => {
    setActiveFilters([]);
    onFiltersChange([]);
  }, [onFiltersChange]);
  
  return (
    <div className="filter-bar">
      {/* Predefined Filters Section */}
      <FilterSection
        title="Quick Filters"
        expanded={expandedSections.predefined}
        onToggle={() => toggleSection('predefined')}
      >
        <div className="filter-row">
          {predefinedFilters.map(filter => (
            <FilterControl
              key={filter.type}
              filter={filter}
              value={getFilterValue(activeFilters, filter.type)}
              onChange={(value) => handleFilterChange(filter.type, value)}
              entityCounts={entityCounts}
            />
          ))}
        </div>
      </FilterSection>
      
      {/* Custom Filters Section */}
      <FilterSection
        title="Advanced Filters"
        expanded={expandedSections.custom}
        onToggle={() => toggleSection('custom')}
        badge={getActiveCustomFiltersCount(activeFilters)}
      >
        <CustomFiltersPanel
          filters={customFilters}
          activeFilters={activeFilters}
          onChange={handleFilterChange}
        />
      </FilterSection>
      
      {/* Search Section */}
      <FilterSection
        title="Search"
        expanded={expandedSections.search}
        onToggle={() => toggleSection('search')}
      >
        <GlobalSearchBar
          value={getSearchValue(activeFilters)}
          onChange={(value) => handleFilterChange('search', value)}
          placeholder="Search by name, ID, or tags..."
        />
      </FilterSection>
      
      {/* Active Filters Display */}
      {activeFilters.length > 0 && (
        <ActiveFiltersDisplay
          filters={activeFilters}
          onRemove={(filterType) => handleFilterChange(filterType, null)}
          onClearAll={clearAllFilters}
        />
      )}
    </div>
  );
};
```

### Active Filters Display

```typescript
const ActiveFiltersDisplay: React.FC<ActiveFiltersProps> = ({
  filters,
  onRemove,
  onClearAll
}) => {
  return (
    <div className="active-filters">
      <div className="active-filters__header">
        <span className="label">Active Filters ({filters.length})</span>
        <button 
          className="clear-all"
          onClick={onClearAll}
        >
          Clear All
        </button>
      </div>
      
      <div className="active-filters__list">
        {filters.map(filter => (
          <FilterPill
            key={filter.id}
            filter={filter}
            onRemove={() => onRemove(filter.type)}
          />
        ))}
      </div>
    </div>
  );
};

const FilterPill: React.FC<FilterPillProps> = ({ filter, onRemove }) => {
  const getFilterLabel = () => {
    switch (filter.type) {
      case 'provider':
        return `Provider: ${filter.values.join(', ')}`;
      case 'cluster':
        return `Clusters: ${filter.values.length} selected`;
      case 'search':
        return `Search: "${filter.value}"`;
      default:
        return `${filter.type}: ${filter.displayValue}`;
    }
  };
  
  return (
    <div className="filter-pill">
      <span className="filter-pill__label">
        {getFilterLabel()}
      </span>
      <button 
        className="filter-pill__remove"
        onClick={onRemove}
        aria-label={`Remove ${filter.type} filter`}
      >
        ×
      </button>
    </div>
  );
};
```

## Filter Persistence

### Saving Filter Preferences

```typescript
class FilterPreferenceManager {
  private readonly STORAGE_KEY = 'messageQueues:filterPreferences';
  
  // Save filter preferences
  savePreferences(preferences: FilterPreferences): void {
    const data = {
      savedFilters: preferences.savedFilters,
      recentFilters: preferences.recentFilters.slice(0, 10),
      defaultFilters: preferences.defaultFilters,
      lastUpdated: Date.now()
    };
    
    localStorage.setItem(this.STORAGE_KEY, JSON.stringify(data));
  }
  
  // Load filter preferences
  loadPreferences(): FilterPreferences {
    try {
      const stored = localStorage.getItem(this.STORAGE_KEY);
      if (!stored) return this.getDefaultPreferences();
      
      const data = JSON.parse(stored);
      return {
        ...data,
        savedFilters: new Map(data.savedFilters),
        recentFilters: data.recentFilters || []
      };
    } catch (error) {
      console.error('Failed to load filter preferences:', error);
      return this.getDefaultPreferences();
    }
  }
  
  // Save filter set
  saveFilterSet(name: string, filters: AppliedFilter[]): void {
    const preferences = this.loadPreferences();
    
    preferences.savedFilters.set(name, {
      id: generateId(),
      name,
      filters,
      createdAt: Date.now(),
      usageCount: 0
    });
    
    this.savePreferences(preferences);
  }
  
  // Add to recent filters
  addToRecent(filters: AppliedFilter[]): void {
    const preferences = this.loadPreferences();
    
    // Remove duplicates
    preferences.recentFilters = preferences.recentFilters.filter(
      recent => !this.areFiltersEqual(recent.filters, filters)
    );
    
    // Add to beginning
    preferences.recentFilters.unshift({
      filters,
      timestamp: Date.now()
    });
    
    // Keep only last 10
    preferences.recentFilters = preferences.recentFilters.slice(0, 10);
    
    this.savePreferences(preferences);
  }
}
```

### Filter URL Synchronization

```typescript
class FilterURLSync {
  // Serialize filters for URL
  static serialize(filters: AppliedFilter[]): string {
    const simplified = filters.map(filter => ({
      t: filter.type,
      v: filter.values || filter.value,
      o: filter.operator
    }));
    
    return encodeURIComponent(JSON.stringify(simplified));
  }
  
  // Deserialize filters from URL
  static deserialize(serialized: string): AppliedFilter[] {
    try {
      const simplified = JSON.parse(decodeURIComponent(serialized));
      
      return simplified.map(item => ({
        type: item.t,
        values: Array.isArray(item.v) ? item.v : undefined,
        value: !Array.isArray(item.v) ? item.v : undefined,
        operator: item.o || 'in',
        id: generateId()
      }));
    } catch (error) {
      console.error('Failed to deserialize filters:', error);
      return [];
    }
  }
  
  // Update URL with current filters
  static updateURL(filters: AppliedFilter[]): void {
    const params = new URLSearchParams(window.location.search);
    
    if (filters.length > 0) {
      params.set('filters', this.serialize(filters));
    } else {
      params.delete('filters');
    }
    
    const newURL = `${window.location.pathname}?${params.toString()}`;
    window.history.replaceState({}, '', newURL);
  }
  
  // Load filters from URL
  static loadFromURL(): AppliedFilter[] {
    const params = new URLSearchParams(window.location.search);
    const serialized = params.get('filters');
    
    return serialized ? this.deserialize(serialized) : [];
  }
}
```

## Performance Optimization

### Filter Caching

```typescript
class FilterCache {
  private cache = new Map<string, CachedFilterResult>();
  private readonly MAX_CACHE_SIZE = 100;
  private readonly TTL = 5 * 60 * 1000; // 5 minutes
  
  // Get cached result
  get(filters: AppliedFilter[]): FilterResult | null {
    const key = this.generateKey(filters);
    const cached = this.cache.get(key);
    
    if (!cached) return null;
    
    // Check TTL
    if (Date.now() - cached.timestamp > this.TTL) {
      this.cache.delete(key);
      return null;
    }
    
    return cached.result;
  }
  
  // Set cached result
  set(filters: AppliedFilter[], result: FilterResult): void {
    const key = this.generateKey(filters);
    
    // Implement LRU eviction
    if (this.cache.size >= this.MAX_CACHE_SIZE) {
      const oldest = this.findOldestEntry();
      this.cache.delete(oldest);
    }
    
    this.cache.set(key, {
      result,
      timestamp: Date.now(),
      accessCount: 0
    });
  }
  
  // Generate cache key
  private generateKey(filters: AppliedFilter[]): string {
    const sorted = [...filters].sort((a, b) => a.type.localeCompare(b.type));
    return JSON.stringify(sorted);
  }
}
```

### Debounced Filter Application

```typescript
const useDebouncedFilters = (
  filters: AppliedFilter[],
  delay: number = 300
) => {
  const [debouncedFilters, setDebouncedFilters] = useState(filters);
  const [isDebouncing, setIsDebouncing] = useState(false);
  
  useEffect(() => {
    setIsDebouncing(true);
    
    const timer = setTimeout(() => {
      setDebouncedFilters(filters);
      setIsDebouncing(false);
    }, delay);
    
    return () => {
      clearTimeout(timer);
    };
  }, [filters, delay]);
  
  return { debouncedFilters, isDebouncing };
};
```

## Best Practices

### 1. Filter Design
- Keep filter options intuitive
- Provide clear visual feedback
- Support keyboard navigation
- Include filter counters

### 2. Performance
- Cache filter results
- Debounce rapid changes
- Use indexed fields
- Implement virtual scrolling

### 3. User Experience
- Save user preferences
- Provide filter presets
- Show active filters clearly
- Enable bulk operations

### 4. Data Handling
- Validate filter inputs
- Handle edge cases gracefully
- Provide meaningful empty states
- Support filter combinations

### 5. Accessibility
- Include ARIA labels
- Support keyboard shortcuts
- Provide screen reader context
- Maintain focus management

## Conclusion

The filter system provides powerful capabilities for navigating and analyzing Kafka infrastructure data. By combining predefined filters, custom selections, and search functionality with a performant implementation, users can efficiently find and focus on the data that matters most to them.