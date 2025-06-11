# GraphQL & NerdGraph

## Overview

NerdGraph is New Relic's GraphQL API that provides a unified interface for querying and managing New Relic data. The Message Queues application leverages NerdGraph extensively for entity discovery, relationship mapping, and cross-account data access.

## NerdGraph Fundamentals

### What is NerdGraph?

NerdGraph is:
- A GraphQL-based API for New Relic platform
- The primary interface for entity operations
- A powerful tool for complex data relationships
- The gateway to cross-account queries

### GraphQL vs REST

```graphql
# GraphQL - Request exactly what you need
{
  actor {
    entitySearch(query: "type = 'AWSMSKCLUSTER'") {
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

# vs REST would require multiple calls:
# GET /entities?type=AWSMSKCLUSTER
# GET /entities/{id}/alertStatus (for each entity)
```

## Core GraphQL Queries

### Entity Search Queries

#### Basic Entity Search
```typescript
export const ALL_KAFKA_TABLE_QUERY = ngql`
  query ALL_KAFKA_TABLE_QUERY(
    $awsQuery: String!, 
    $confluentCloudQuery: String!, 
    $facet: EntitySearchCountsFacet!,
    $orderBy: EntitySearchOrderBy!
  ) {
    actor {
      awsEntitySearch: entitySearch(query: $awsQuery) {
        count
        facetedCounts(facets: {facetCriterion: {facet: $facet}, orderBy: $orderBy}) {
          counts {
            count
            facet
          }
        }
        results {
          accounts {
            id
            name
            reportingEventTypes(filter: "AwsMskBrokerSample")
          }
        }
      }
      confluentCloudEntitySearch: entitySearch(query: $confluentCloudQuery) {
        count
        facetedCounts(facets: {facetCriterion: {facet: $facet}, orderBy: $orderBy}) {
          counts {
            count
            facet
          }
        }
        results {
          accounts {
            id
            name
          }
        }
      }
    }
  }
`;
```

#### Entity Search with Relationships
```graphql
query EntityWithRelationships($guid: EntityGuid!) {
  actor {
    entity(guid: $guid) {
      guid
      name
      type
      domain
      entityType
      alertSeverity
      
      # Tags for filtering and grouping
      tags {
        key
        values
      }
      
      # Relationships to other entities
      relationships {
        source {
          entity {
            guid
            name
            type
          }
        }
        target {
          entity {
            guid
            name
            type
          }
        }
        type
      }
      
      # Type-specific fields
      ... on InfrastructureAwsKafkaClusterEntity {
        clusterName
        awsRegion
        activeControllers
        offlinePartitions
      }
    }
  }
}
```

### Cluster Discovery Queries

#### Get Clusters from Topic Filter
```typescript
export const GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY = ngql`
  query GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY(
    $awsTopicQuery: String!, 
    $confluentTopicQuery: String!
  ) {
    actor {
      awsTopicEntitySearch: entitySearch(query: $awsTopicQuery) {
        __typename
        polling: groupedResults(by: {tag: "aws.clusterName"}) {
          group
        }
        metrics: groupedResults(by: {tag: "aws.kafka.ClusterName"}) {
          group
        }
        providerClusterName: groupedResults(by: {tag: "provider.clusterName"}) {
          group
        }
      }
      confluentTopicEntitySearch: entitySearch(query: $confluentTopicQuery) {
        __typename
        results(cursor: null, limit: 2000) {
          entities {
            ... on InfrastructureAwsKafkaTopicEntity {
              tags {
                key
                values
              }
              entityType
            }
          }
        }
      }
    }
  }
`;
```

### APM Relationship Queries

#### Get Related APM Entities
```typescript
export const GET_RELATED_APM_ENTITIES_FOR_TOPIC = ngql`
  query GET_RELATED_APM_ENTITIES_FOR_TOPIC($guid: String!) {
    actor {
      entity(guid: $guid) {
        relatedEntities(filter: {
          relationshipTypes: {include: [PRODUCES, CONSUMES]},
          entityDomainTypes: {include: {domain: APM}}
        }) {
          results {
            target {
              entity {
                name
                guid
              }
            }
            type
          }
        }
      }
    }
  }
`;
```

## Entity Search Patterns

### Search Query Building

```typescript
// Entity search query builders
export const EntitySearchQueries = {
  // AWS MSK Clusters
  awsClusters: (filters?: string) => 
    `domain IN ('INFRA') AND type='AWSMSKCLUSTER' ${filters || ''}`,
  
  // Confluent Clusters  
  confluentClusters: (filters?: string) =>
    `domain IN ('INFRA') AND type='CONFLUENTCLOUDCLUSTER' ${filters || ''}`,
  
  // All topics
  allTopics: () => 
    `domain IN ('INFRA') AND type IN ('AWSMSKTOPIC', 'CONFLUENTCLOUDKAFKATOPIC')`,
  
  // Filtered search
  withFilters: (base: string, filters: Filter[]) => {
    const filterClauses = filters.map(f => {
      switch (f.type) {
        case 'tag':
          return `tags.${f.key} = '${f.value}'`;
        case 'attribute':
          return `${f.key} = '${f.value}'`;
        case 'name':
          return `name LIKE '${f.value}'`;
        default:
          return '';
      }
    }).filter(Boolean);
    
    return `${base} AND ${filterClauses.join(' AND ')}`;
  }
};
```

### Faceted Search

```typescript
// Faceted entity counts
const FacetedSearchQuery = ngql`
  query FacetedEntitySearch($query: String!, $facets: [EntitySearchCountsFacetInput!]!) {
    actor {
      entitySearch(query: $query) {
        count
        facetedCounts(facets: $facets) {
          counts {
            count
            facet
          }
          facet
        }
      }
    }
  }
`;

// Usage
const facetConfig = [
  { facetCriterion: { facet: 'ACCOUNT_ID' }, orderBy: 'COUNT' },
  { facetCriterion: { facet: 'TYPE' }, orderBy: 'COUNT' },
  { facetCriterion: { tag: 'provider' }, orderBy: 'COUNT' }
];
```

## Relationship Queries

### Relationship Types

```typescript
enum RelationshipType {
  CONTAINS = 'CONTAINS',      // Cluster contains brokers/topics
  HOSTED_BY = 'HOSTED_BY',    // Inverse of contains
  PRODUCES = 'PRODUCES',      // APM app produces to topic
  CONSUMES = 'CONSUMES',      // APM app consumes from topic
  CALLS = 'CALLS',           // Service dependencies
  SERVES = 'SERVES'          // Infrastructure serves application
}
```

### Complex Relationship Queries

```graphql
query KafkaTopologyMap($clusterGuid: EntityGuid!) {
  actor {
    entity(guid: $clusterGuid) {
      name
      
      # Get all brokers in cluster
      brokers: relatedEntities(filter: {
        relationshipTypes: {include: [CONTAINS]},
        entityDomainTypes: {include: {type: "AWSMSKBROKER"}}
      }) {
        results {
          target {
            entity {
              guid
              name
              ... on InfrastructureAwsKafkaBrokerEntity {
                brokerId
                cpuUsage
                diskUsage
              }
            }
          }
        }
      }
      
      # Get all topics in cluster
      topics: relatedEntities(filter: {
        relationshipTypes: {include: [CONTAINS]},
        entityDomainTypes: {include: {type: "AWSMSKTOPIC"}}
      }) {
        results {
          target {
            entity {
              guid
              name
              ... on InfrastructureAwsKafkaTopicEntity {
                partitionCount
                bytesInPerSec
                bytesOutPerSec
              }
            }
          }
        }
      }
      
      # Get producing applications
      producers: relatedEntities(filter: {
        relationshipTypes: {include: [PRODUCES]},
        entityDomainTypes: {include: {domain: "APM"}}
      }) {
        results {
          source {
            entity {
              guid
              name
              alertSeverity
            }
          }
          target {
            entity {
              name
            }
          }
        }
      }
    }
  }
}
```

## Batch Operations

### Batch Entity Queries

```typescript
const BATCH_ENTITY_QUERY = ngql`
  query BatchEntityData($guids: [EntityGuid!]!) {
    actor {
      entities(guids: $guids) {
        guid
        name
        type
        domain
        alertSeverity
        tags {
          key
          values
        }
        
        # Type-specific fields
        ... on InfrastructureAwsKafkaClusterEntity {
          activeControllers
          offlinePartitions
          totalBrokers
          totalTopics
        }
        
        ... on InfrastructureAwsKafkaTopicEntity {
          partitionCount
          replicationFactor
          messagesPerSec
        }
      }
    }
  }
`;
```

### Parallel Query Execution

```typescript
// Execute multiple queries in parallel
const executeParallelQueries = async () => {
  const [clustersResult, topicsResult, metricsResult] = await Promise.all([
    // Entity search
    NerdGraphQuery.query({
      query: ALL_KAFKA_TABLE_QUERY,
      variables: { 
        awsQuery: AWS_CLUSTER_QUERY, 
        confluentCloudQuery: CONFLUENT_CLUSTER_QUERY 
      }
    }),
    
    // Topic discovery
    NerdGraphQuery.query({
      query: GET_CLUSTERS_FROM_TOPIC_FILTER_QUERY,
      variables: { 
        awsTopicQuery: topicQuery,
        confluentTopicQuery: confluentTopicQuery
      }
    }),
    
    // NRQL metrics
    NerdGraphQuery.query({
      query: NRQL_QUERY,
      variables: { 
        accountId,
        nrql: metricsQuery 
      }
    })
  ]);
  
  return {
    clusters: clustersResult.data,
    topics: topicsResult.data,
    metrics: metricsResult.data
  };
};
```

## NRQL via GraphQL

### Execute NRQL Queries

```graphql
query ExecuteNRQL($accountId: Int!, $nrql: Nrql!) {
  actor {
    account(id: $accountId) {
      nrql(query: $nrql) {
        results
        totalResult
        metadata {
          eventTypes
          facets
          messages
          timeWindow {
            begin
            end
          }
        }
        embeddedChartUrl
      }
    }
  }
}
```

### Cross-Account NRQL

```typescript
const CROSS_ACCOUNT_METRICS_QUERY = ngql`
  query CrossAccountMetrics($accountIds: [Int!]!, $nrql: Nrql!) {
    actor {
      accounts(ids: $accountIds) {
        id
        name
        nrql(query: $nrql) {
          results
        }
      }
    }
  }
`;

// Usage
const results = await NerdGraphQuery.query({
  query: CROSS_ACCOUNT_METRICS_QUERY,
  variables: {
    accountIds: [123456, 789012],
    nrql: "FROM AwsMskClusterSample SELECT count(*)"
  }
});
```

## Entity Management

### Entity Tags Management

```graphql
mutation AddEntityTags($guid: EntityGuid!, $tags: [EntityTagInput!]!) {
  taggingAddTagsToEntity(guid: $guid, tags: $tags) {
    errors {
      message
      type
    }
  }
}

mutation RemoveEntityTags($guid: EntityGuid!, $tagKeys: [String!]!) {
  taggingDeleteTagFromEntity(guid: $guid, tagKeys: $tagKeys) {
    errors {
      message
      type
    }
  }
}
```

### Entity Metadata Updates

```typescript
const updateEntityMetadata = async (guid: string, metadata: any) => {
  const mutation = ngql`
    mutation UpdateEntity($guid: EntityGuid!, $metadata: EntityMetadataInput!) {
      entityUpdate(guid: $guid, metadata: $metadata) {
        entity {
          guid
          name
          tags {
            key
            values
          }
        }
        errors {
          message
        }
      }
    }
  `;
  
  return NerdGraphQuery.query({
    query: mutation,
    variables: { guid, metadata }
  });
};
```

## Performance Optimization

### Query Optimization Strategies

```typescript
const QueryOptimization = {
  // 1. Request only needed fields
  minimal: ngql`
    query MinimalEntity($guid: EntityGuid!) {
      actor {
        entity(guid: $guid) {
          guid
          name
          alertSeverity
        }
      }
    }
  `,
  
  // 2. Use fragments for reusable selections
  withFragments: ngql`
    fragment ClusterFields on InfrastructureAwsKafkaClusterEntity {
      clusterName
      activeControllers
      offlinePartitions
    }
    
    query ClusterData($guid: EntityGuid!) {
      actor {
        entity(guid: $guid) {
          ...ClusterFields
        }
      }
    }
  `,
  
  // 3. Batch similar queries
  batched: ngql`
    query BatchedData($clusterGuids: [EntityGuid!]!, $topicGuids: [EntityGuid!]!) {
      actor {
        clusters: entities(guids: $clusterGuids) {
          guid
          name
        }
        topics: entities(guids: $topicGuids) {
          guid
          name
        }
      }
    }
  `,
  
  // 4. Use appropriate limits
  withLimits: ngql`
    query LimitedSearch($query: String!) {
      actor {
        entitySearch(query: $query, options: {limit: 200}) {
          results {
            entities {
              guid
              name
            }
            nextCursor
          }
        }
      }
    }
  `
};
```

### Caching Strategies

```typescript
class GraphQLCache {
  private cache: Map<string, CacheEntry> = new Map();
  
  async query(options: QueryOptions): Promise<any> {
    const cacheKey = this.getCacheKey(options);
    const cached = this.cache.get(cacheKey);
    
    if (cached && !this.isExpired(cached)) {
      return cached.data;
    }
    
    const result = await NerdGraphQuery.query(options);
    
    this.cache.set(cacheKey, {
      data: result,
      timestamp: Date.now(),
      ttl: this.getTTL(options)
    });
    
    return result;
  }
  
  private getTTL(options: QueryOptions): number {
    // Entity data: 5 minutes
    if (options.query.includes('entitySearch')) return 300000;
    
    // Metrics: 1 minute
    if (options.query.includes('nrql')) return 60000;
    
    // Relationships: 10 minutes
    if (options.query.includes('relationships')) return 600000;
    
    // Default: 2 minutes
    return 120000;
  }
}
```

## Error Handling

### GraphQL Error Types

```typescript
interface GraphQLError {
  message: string;
  extensions?: {
    code: string;
    exception?: {
      stacktrace: string[];
    };
  };
  path?: (string | number)[];
}

const handleGraphQLErrors = (result: any) => {
  if (result.errors) {
    result.errors.forEach((error: GraphQLError) => {
      switch (error.extensions?.code) {
        case 'FORBIDDEN':
          console.error('Access denied:', error.message);
          break;
        case 'NOT_FOUND':
          console.error('Entity not found:', error.message);
          break;
        case 'RATE_LIMITED':
          console.error('Rate limit exceeded:', error.message);
          break;
        default:
          console.error('GraphQL error:', error);
      }
    });
  }
  
  // Partial results are still valid
  return result.data;
};
```

### Retry Logic

```typescript
const retryableQuery = async (
  query: any,
  variables: any,
  maxRetries = 3
): Promise<any> => {
  let lastError;
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const result = await NerdGraphQuery.query({ query, variables });
      
      if (!result.errors) {
        return result;
      }
      
      // Check if errors are retryable
      const hasRetryableError = result.errors.some(e => 
        ['TIMEOUT', 'INTERNAL_ERROR'].includes(e.extensions?.code)
      );
      
      if (!hasRetryableError) {
        return result; // Non-retryable errors
      }
      
      lastError = result.errors;
    } catch (error) {
      lastError = error;
    }
    
    // Exponential backoff
    await new Promise(resolve => 
      setTimeout(resolve, Math.pow(2, attempt) * 1000)
    );
  }
  
  throw lastError;
};
```

## Security Considerations

### Query Validation

```typescript
const validateEntityQuery = (query: string): boolean => {
  // Prevent injection attacks
  const forbidden = [
    'mutation',
    'subscription',
    '__schema',
    '__type'
  ];
  
  return !forbidden.some(term => 
    query.toLowerCase().includes(term)
  );
};
```

### Access Control

```typescript
const checkEntityAccess = async (guid: string): Promise<boolean> => {
  try {
    const result = await NerdGraphQuery.query({
      query: ngql`
        query CheckAccess($guid: EntityGuid!) {
          actor {
            entity(guid: $guid) {
              guid
            }
          }
        }
      `,
      variables: { guid }
    });
    
    return !!result.data?.actor?.entity;
  } catch (error) {
    return false;
  }
};
```

## Best Practices

### 1. Query Structure
- Request only the fields you need
- Use fragments for reusable selections
- Batch related queries when possible

### 2. Error Handling
- Always check for partial results
- Implement proper retry logic
- Log errors for debugging

### 3. Performance
- Implement caching for expensive queries
- Use appropriate limits and pagination
- Avoid N+1 query problems

### 4. Security
- Validate all user inputs
- Use parameterized queries
- Check access permissions

### 5. Monitoring
- Track query performance
- Monitor error rates
- Set up alerts for failures

## Conclusion

NerdGraph provides powerful capabilities for entity discovery, relationship mapping, and data querying in the Message Queues application. By understanding GraphQL patterns and optimization strategies, you can build efficient and scalable monitoring solutions.