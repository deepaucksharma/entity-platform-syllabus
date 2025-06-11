# Hooks Implementation Guide

## Overview

This document describes the actual React hooks implementation in the Message Queues monitoring system, showing the patterns and utilities used for state management, data fetching, and component logic.

## Data Fetching Hooks

### useFetchEntityMetrics Hook (Actual Implementation)

The primary hook for fetching entity metrics based on provider, entity type, and filters.

```typescript
// From use-fetch-entity-metrics/index.ts
import { useEffect, useState } from 'react';
import { NrqlQuery } from 'nr1';
import { getQueryString } from '../../utils/query-utils';
import { DIM_QUERIES, MTS_QUERIES, CONFLUENT_CLOUD_DIM_QUERIES } from '../../utils/query-utils';
import { CONFLUENT_CLOUD_PROVIDER, MSK_PROVIDER } from '../../config/constants';

export const useFetchEntityMetrics = ({ 
  item, 
  show, 
  groupBy, 
  filterSet 
}: UseFetchEntityMetricsProps) => {
  const [state, setState] = useState({
    loading: true,
    error: null,
    EntityMetrics: {}
  });

  useEffect(() => {
    fetchMetrics();
  }, [item, show, groupBy, filterSet]);

  const fetchMetrics = async () => {
    if (!item?.['Account Id']) {
      setState({ loading: false, error: null, EntityMetrics: {} });
      return;
    }

    setState(prev => ({ ...prev, loading: true }));

    try {
      // Determine query type based on provider
      const isMetricStream = item['Is Metric Stream'];
      const provider = item.Provider;
      
      // Select appropriate query set
      let queries;
      if (provider === CONFLUENT_CLOUD_PROVIDER) {
        queries = CONFLUENT_CLOUD_DIM_QUERIES;
      } else if (isMetricStream) {
        queries = MTS_QUERIES;
      } else {
        queries = DIM_QUERIES;
      }

      // Build query key
      const queryKey = buildQueryKey(show, groupBy);
      const queryDef = queries[queryKey];
      
      if (!queryDef) {
        throw new Error(`No query found for ${queryKey}`);
      }

      // Execute query
      const query = getQueryString(
        queryDef, 
        provider,
        false,
        filterSet
      );

      const { data, error } = await NrqlQuery.query({
        accountId: parseInt(item['Account Id']),
        query,
        formatType: NrqlQuery.FORMAT_TYPE.RAW
      });

      if (error) {
        throw error;
      }

      // Process results
      const entityMetrics = processEntityMetrics(
        data,
        show,
        provider
      );

      setState({
        loading: false,
        error: null,
        EntityMetrics: entityMetrics
      });

    } catch (error) {
      console.error('Error fetching entity metrics:', error);
      setState({
        loading: false,
        error,
        EntityMetrics: {}
      });
    }
  };

  return state;
};

// Helper functions
const buildQueryKey = (show: string, groupBy?: string) => {
  const showMap = {
    cluster: 'CLUSTER',
    topic: 'TOPIC',
    broker: 'BROKER'
  };
  
  const base = showMap[show] || 'CLUSTER';
  
  if (groupBy) {
    return `${base}_GROUPBY_${groupBy.toUpperCase()}`;
  }
  
  return `${base}_HEALTH_QUERY`;
};

const processEntityMetrics = (data, entityType, provider) => {
  if (!data?.facets) return {};
  
  const metrics = {};
  
  data.facets.forEach(facet => {
    const entityName = facet.name;
    const results = facet.results?.[0] || {};
    
    metrics[entityName] = {
      name: entityName,
      type: entityType,
      provider,
      ...results,
      healthScore: calculateHealthScore(results, entityType)
    };
  });
  
  return metrics;
};
```

### useTotalClusters Hook

Hook for fetching total cluster count with caching.

```typescript
// From use-total-clusters/index.ts
import { useState, useEffect } from 'react';
import { NrqlQuery } from 'nr1';
import { TOTAL_CLUSTERS_QUERY } from '../../utils/query-utils';

export const useTotalClusters = ({ 
  accountId, 
  provider, 
  isMetricStream,
  where = []
}: UseTotalClustersProps) => {
  const [totalClusters, setTotalClusters] = useState(0);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!accountId) {
      setLoading(false);
      return;
    }

    fetchTotalClusters();
  }, [accountId, provider, isMetricStream, where]);

  const fetchTotalClusters = async () => {
    try {
      const queryDef = TOTAL_CLUSTERS_QUERY(provider, isMetricStream);
      
      if (!queryDef.select) {
        setTotalClusters(0);
        setLoading(false);
        return;
      }

      // Build query with additional where conditions
      const query = new NRQLModel(queryDef.from)
        .select(queryDef.select)
        .where([...where, ...(queryDef.where || [])])
        .toString();

      const { data, error } = await NrqlQuery.query({
        accountId: parseInt(accountId),
        query,
        formatType: NrqlQuery.FORMAT_TYPE.RAW
      });

      if (error) throw error;

      const count = data?.results?.[0]?.['Total clusters'] || 0;
      setTotalClusters(count);
      setError(null);
    } catch (err) {
      console.error('Error fetching total clusters:', err);
      setError(err);
      setTotalClusters(0);
    } finally {
      setLoading(false);
    }
  };

  return { totalClusters, loading, error };
};
```

### useUnhealthyClustersCount Hook

Hook for calculating unhealthy clusters count.

```typescript
// From use-unhealthy-clusters-count/index.ts
import { useState, useEffect } from 'react';
import { NrqlQuery } from 'nr1';

export const useUnhealthyClustersCount = ({
  accountId,
  provider,
  isMetricStream
}: UseUnhealthyClustersCountProps) => {
  const [unhealthyCount, setUnhealthyCount] = useState(0);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!accountId) {
      setLoading(false);
      return;
    }

    calculateUnhealthyClusters();
  }, [accountId, provider, isMetricStream]);

  const calculateUnhealthyClusters = async () => {
    try {
      // Query for cluster health metrics
      const healthQuery = buildHealthQuery(provider, isMetricStream);
      
      const { data, error } = await NrqlQuery.query({
        accountId: parseInt(accountId),
        query: healthQuery,
        formatType: NrqlQuery.FORMAT_TYPE.RAW
      });

      if (error) throw error;

      // Count unhealthy clusters
      const unhealthy = data?.facets?.filter(facet => {
        const results = facet.results?.[0] || {};
        return isUnhealthy(results, provider);
      }).length || 0;

      setUnhealthyCount(unhealthy);
    } catch (error) {
      console.error('Error calculating unhealthy clusters:', error);
      setUnhealthyCount(0);
    } finally {
      setLoading(false);
    }
  };

  const isUnhealthy = (metrics, provider) => {
    if (provider === CONFLUENT_CLOUD_PROVIDER) {
      // Confluent specific health checks
      return metrics['Hot partition Ingress'] > 0 || 
             metrics['Hot partition Egress'] > 0 ||
             metrics['Cluster load percent'] > 90;
    } else {
      // AWS MSK health checks
      return metrics['Active Controllers'] !== 1 || 
             metrics['Offline Partitions'] > 0;
    }
  };

  return { unhealthyCount, loading };
};
```

## Chart Hooks

### useSummaryChart Hook

Hook for managing summary chart data and configurations.

```typescript
// From use-summary-chart/index.ts
import { useEffect, useState } from 'react';
import { NrqlQuery } from 'nr1';
import { METRIC_IDS } from '../../config/constants';
import { getQueryByProviderAndPreference } from '../../utils/query-utils';

export const useSummaryChart = ({
  accountIds,
  providers,
  metricId,
  timeRange,
  filters = []
}: UseSummaryChartProps) => {
  const [chartData, setChartData] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!accountIds?.length || !metricId) {
      setLoading(false);
      return;
    }

    fetchChartData();
  }, [accountIds, providers, metricId, timeRange, filters]);

  const fetchChartData = async () => {
    setLoading(true);
    setError(null);

    try {
      // Execute queries for each account/provider combination
      const queries = accountIds.map((accountId, index) => {
        const provider = providers[index];
        const isMetricStream = detectMetricStream(provider);
        
        const query = getQueryByProviderAndPreference(
          isMetricStream,
          provider,
          metricId,
          filters,
          [],
          timeRange,
          { unhealthyClusters: 0 }
        );

        return NrqlQuery.query({
          accountId: parseInt(accountId),
          query,
          formatType: NrqlQuery.FORMAT_TYPE.CHART
        });
      });

      const results = await Promise.all(queries);
      
      // Process and combine results
      const processedData = processChartData(results, metricId);
      setChartData(processedData);
      
    } catch (err) {
      console.error('Error fetching chart data:', err);
      setError(err);
    } finally {
      setLoading(false);
    }
  };

  const processChartData = (results, metricId) => {
    // Handle different metric types
    switch (metricId) {
      case METRIC_IDS.TOTAL_PRODUCED_THROUGHPUT:
      case METRIC_IDS.MESSAGES_PRODUCED_RATE:
        return combineTimeseriesData(results);
        
      case METRIC_IDS.BROKER_COUNT_BY_CLUSTER:
      case METRIC_IDS.TOPIC_COUNT_BY_CLUSTER:
        return combineFacetedData(results);
        
      default:
        return combineSingleValueData(results);
    }
  };

  return {
    data: chartData,
    loading,
    error,
    refetch: fetchChartData
  };
};
```

## Filter Hooks

### useFilterItems Hook

Hook for managing filter state and propagation.

```typescript
// From use-filter-items/index.tsx
import { useState, useEffect, useCallback } from 'react';
import { navigation } from 'nr1';

export const useFilterItems = (initialFilters = []) => {
  const [filters, setFilters] = useState(initialFilters);
  const [urlState, setUrlState] = useState({});

  // Sync filters with URL state
  useEffect(() => {
    const encodedFilters = encodeFilters(filters);
    navigation.setUrlState({ filters: encodedFilters });
    setUrlState({ filters: encodedFilters });
  }, [filters]);

  // Add filter
  const addFilter = useCallback((filter) => {
    setFilters(prev => {
      // Check if filter already exists
      const existing = prev.find(f => 
        f.key === filter.key && f.value === filter.value
      );
      
      if (existing) return prev;
      
      return [...prev, {
        ...filter,
        id: generateFilterId()
      }];
    });
  }, []);

  // Remove filter
  const removeFilter = useCallback((filterId) => {
    setFilters(prev => prev.filter(f => f.id !== filterId));
  }, []);

  // Update filter
  const updateFilter = useCallback((filterId, updates) => {
    setFilters(prev => prev.map(f => 
      f.id === filterId ? { ...f, ...updates } : f
    ));
  }, []);

  // Clear all filters
  const clearFilters = useCallback(() => {
    setFilters([]);
  }, []);

  // Build query conditions from filters
  const getQueryConditions = useCallback(() => {
    return filters.map(filter => {
      const operator = filter.operator || 'IN';
      const value = Array.isArray(filter.value) 
        ? `(${filter.value.map(v => `'${v}'`).join(', ')})`
        : `'${filter.value}'`;
      
      return `\`${filter.key}\` ${operator} ${value}`;
    });
  }, [filters]);

  return {
    filters,
    addFilter,
    removeFilter,
    updateFilter,
    clearFilters,
    getQueryConditions,
    urlState
  };
};

// Helper functions
const encodeFilters = (filters) => {
  return btoa(JSON.stringify(filters));
};

const decodeFilters = (encoded) => {
  try {
    return JSON.parse(atob(encoded));
  } catch {
    return [];
  }
};

const generateFilterId = () => {
  return `filter-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
};
```

### useTopicFilterQuery Hook

Hook for managing topic-specific filter queries.

```typescript
// From use-topic-filter-query/index.ts
import { useState, useEffect } from 'react';
import { NerdGraphQuery } from 'nr1';
import { GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY } from '../../utils/query-utils';

export const useTopicFilterQuery = (topicFilters, provider) => {
  const [clusters, setClusters] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    if (!topicFilters?.length) {
      setClusters([]);
      return;
    }

    fetchClustersFromTopics();
  }, [topicFilters, provider]);

  const fetchClustersFromTopics = async () => {
    setLoading(true);

    try {
      // Build topic query conditions
      const topicNames = topicFilters.map(t => `'${t}'`).join(', ');
      const awsQuery = `domain = 'INFRA' AND type = 'AWSMSKTOPIC' AND name IN (${topicNames})`;
      const confluentQuery = `domain = 'INFRA' AND type = 'CONFLUENTCLOUDKAFKATOPIC' AND name IN (${topicNames})`;

      const { data, error } = await NerdGraphQuery.query({
        query: GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY,
        variables: {
          awsTopicQuery: awsQuery,
          confluentTopicQuery: confluentQuery
        }
      });

      if (error) throw error;

      // Extract cluster names
      const clusterNames = extractClusterNames(data, provider);
      setClusters(clusterNames);

    } catch (error) {
      console.error('Error fetching clusters from topics:', error);
      setClusters([]);
    } finally {
      setLoading(false);
    }
  };

  const extractClusterNames = (data, provider) => {
    const clusters = new Set();

    if (provider === 'AWS MSK') {
      // Extract from AWS results
      const pollingClusters = data?.actor?.awsTopicEntitySearch?.polling || [];
      const metricsClusters = data?.actor?.awsTopicEntitySearch?.metrics || [];
      
      [...pollingClusters, ...metricsClusters].forEach(item => {
        if (item.group) clusters.add(item.group);
      });
    } else {
      // Extract from Confluent results
      const confluentClusters = data?.actor?.confluentTopicEntitySearch?.groupedResults || [];
      
      confluentClusters.forEach(item => {
        if (item.group) clusters.add(item.group);
      });
    }

    return Array.from(clusters);
  };

  return { clusters, loading };
};
```

## Utility Hooks

### useOfferExists Hook

Hook for checking if a specific New Relic offer/feature exists.

```typescript
// From use-offer-exists/index.ts
import { useState, useEffect } from 'react';
import { NerdGraphQuery } from 'nr1';

const OFFER_CHECK_QUERY = `
  query CheckOffer($offerId: String!) {
    actor {
      account {
        offers {
          id
          name
          active
        }
      }
    }
  }
`;

export const useOfferExists = (offerId) => {
  const [exists, setExists] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!offerId) {
      setLoading(false);
      return;
    }

    checkOffer();
  }, [offerId]);

  const checkOffer = async () => {
    try {
      const { data, error } = await NerdGraphQuery.query({
        query: OFFER_CHECK_QUERY,
        variables: { offerId }
      });

      if (error) throw error;

      const offers = data?.actor?.account?.offers || [];
      const offerExists = offers.some(offer => 
        offer.id === offerId && offer.active
      );

      setExists(offerExists);
    } catch (error) {
      console.error('Error checking offer:', error);
      setExists(false);
    } finally {
      setLoading(false);
    }
  };

  return { exists, loading };
};
```

## Custom Hook Patterns

### Query Result Caching

```typescript
// Custom hook with caching
const useQueryWithCache = (query, variables, options = {}) => {
  const cacheKey = JSON.stringify({ query, variables });
  const cached = queryCache.get(cacheKey);
  
  const [state, setState] = useState({
    data: cached?.data || null,
    loading: !cached,
    error: null,
    fromCache: !!cached
  });

  useEffect(() => {
    if (cached && Date.now() - cached.timestamp < options.ttl) {
      return;
    }

    executeQuery();
  }, [cacheKey, options.ttl]);

  const executeQuery = async () => {
    setState(prev => ({ ...prev, loading: true }));

    try {
      const result = await NerdGraphQuery.query({ query, variables });
      
      if (result.error) throw result.error;

      // Cache the result
      queryCache.set(cacheKey, {
        data: result.data,
        timestamp: Date.now()
      });

      setState({
        data: result.data,
        loading: false,
        error: null,
        fromCache: false
      });
    } catch (error) {
      setState({
        data: null,
        loading: false,
        error,
        fromCache: false
      });
    }
  };

  return state;
};
```

### Debounced State Updates

```typescript
// Hook for debounced state updates
const useDebouncedState = (initialValue, delay = 300) => {
  const [value, setValue] = useState(initialValue);
  const [debouncedValue, setDebouncedValue] = useState(initialValue);
  const timeoutRef = useRef(null);

  useEffect(() => {
    timeoutRef.current = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    return () => {
      clearTimeout(timeoutRef.current);
    };
  }, [value, delay]);

  return [value, setValue, debouncedValue];
};
```

### Entity Context Hook

```typescript
// Hook for accessing entity context
const useEntityContext = () => {
  const [nerdletState] = useNerdletState();
  const { entityGuid } = useNerdletUrlState();
  
  const [entity, setEntity] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    if (!entityGuid) {
      setLoading(false);
      return;
    }

    loadEntity();
  }, [entityGuid]);

  const loadEntity = async () => {
    try {
      const { data } = await EntityByGuidQuery.query({
        entityGuid
      });

      setEntity(data.actor.entity);
    } catch (error) {
      console.error('Error loading entity:', error);
      setEntity(null);
    } finally {
      setLoading(false);
    }
  };

  return {
    entity,
    loading,
    accountId: entity?.accountId || nerdletState?.accountId,
    provider: entity?.tags?.provider?.[0],
    entityType: entity?.type
  };
};
```

## Hook Composition Patterns

### Combining Multiple Hooks

```typescript
// Example of composing multiple hooks
const useKafkaMetrics = ({ accountId, provider, entityType }) => {
  // Use multiple hooks together
  const { totalClusters } = useTotalClusters({ 
    accountId, 
    provider,
    isMetricStream: provider === 'AWS MSK'
  });
  
  const { unhealthyCount } = useUnhealthyClustersCount({
    accountId,
    provider
  });
  
  const { EntityMetrics } = useFetchEntityMetrics({
    item: { 'Account Id': accountId, Provider: provider },
    show: entityType,
    groupBy: null
  });
  
  // Combine results
  return {
    summary: {
      total: totalClusters,
      unhealthy: unhealthyCount,
      healthy: totalClusters - unhealthyCount
    },
    entities: EntityMetrics,
    loading: !totalClusters || !EntityMetrics
  };
};
```

## Best Practices

### 1. Error Handling
- Always handle loading and error states
- Provide meaningful error messages
- Log errors for debugging
- Gracefully degrade functionality

### 2. Performance
- Implement caching for expensive queries
- Use debouncing for user input
- Memoize computed values
- Clean up subscriptions and timers

### 3. Type Safety
- Use TypeScript interfaces for props and returns
- Define query result types
- Validate API responses
- Handle null/undefined cases

### 4. Reusability
- Keep hooks focused and single-purpose
- Use composition over complex hooks
- Provide sensible defaults
- Make hooks configurable

### 5. Testing
- Test hooks in isolation
- Mock external dependencies
- Test error scenarios
- Verify cleanup functions

## Conclusion

The hooks implementation in the Message Queues monitoring system follows React best practices while providing powerful abstractions for data fetching, state management, and business logic. These hooks form the foundation for building responsive and maintainable UI components.