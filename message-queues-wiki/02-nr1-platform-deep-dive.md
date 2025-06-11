# NR1 Platform Deep Dive

## Introduction to New Relic One Platform

New Relic One (NR1) is a programmable observability platform that enables developers to build custom applications and visualizations. The Message Queues monitoring system leverages the full capabilities of this platform.

## Nerdpack Architecture

### What is a Nerdpack?

A Nerdpack is a deployable unit containing one or more artifacts that extend the New Relic platform. The Message Queues application is structured as a complete Nerdpack.

```
message-queues (Nerdpack)
├── nerdlets/          # Visual components (screens)
├── visualizations/    # Custom chart types
├── launchers/         # Entry points
├── common/           # Shared resources
└── nr1.json          # Nerdpack metadata
```

### Nerdpack Configuration

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

Key attributes:
- **id**: Unique identifier (GUID) for the Nerdpack
- **sdkVersion**: Platform SDK version (v3 = latest features)
- **teamStoreId**: Internal team identifier for ownership

## Nerdlets - The Core Components

### What are Nerdlets?

Nerdlets are React components that render within the New Relic platform at defined extension points. Each represents a distinct view or screen in the application.

### Message Queues Nerdlets

#### 1. Home Nerdlet
```json
// nerdlets/home/nr1.json
{
  "schemaType": "NERDLET",
  "id": "home",
  "displayName": "home",
  "description": "Primary landing page for Message Queues"
}
```

```typescript
// nerdlets/home/index.tsx
import React from 'react';
import { NerdletStateContext, PlatformStateContext } from 'nr1';

export default function HomeNerdlet() {
  return (
    <PlatformStateContext.Consumer>
      {(platformState) => (
        <NerdletStateContext.Consumer>
          {(nerdletState) => (
            <QueueAndStreamsHomeNerdlet 
              platformState={platformState}
              nerdletState={nerdletState}
            />
          )}
        </NerdletStateContext.Consumer>
      )}
    </PlatformStateContext.Consumer>
  );
}
```

#### 2. Summary Nerdlet
- Displays account-level analytics
- Hosts the Kafka Navigator visualization
- Manages complex state for filtering and navigation

#### 3. MQ Detail Nerdlet
- Entity-specific detailed views
- Coordinates navigation between different entity types
- Handles deep linking through URL state

### Nerdlet Lifecycle

```typescript
// Nerdlet lifecycle hooks
interface NerdletLifecycle {
  // Component mounting
  componentDidMount(): void {
    // Initialize data fetching
    // Set up subscriptions
    // Configure platform integration
  }
  
  // State updates
  componentDidUpdate(prevProps, prevState): void {
    // Handle navigation changes
    // Update queries based on filters
    // Manage cache invalidation
  }
  
  // Cleanup
  componentWillUnmount(): void {
    // Cancel pending requests
    // Clear subscriptions
    // Save user preferences
  }
}
```

## Platform SDK Components

### Core UI Components

The NR1 SDK provides a rich set of UI components:

```typescript
import {
  Button,
  TextField,
  Select,
  Stack,
  StackItem,
  Grid,
  GridItem,
  Card,
  CardHeader,
  CardBody,
  HeadingText,
  BlockText,
  Link,
  Icon,
  Spinner,
  EmptyState,
  NrqlQuery,
  Table,
  TableHeader,
  TableHeaderCell,
  TableRow,
  TableRowCell,
  UserStorageMutation,
  EntityStorageMutation,
  AccountStorageMutation,
  Toast,
  Tooltip,
  Modal,
  Checkbox,
  RadioGroup,
  Tabs,
  TabsItem
} from 'nr1';
```

### Layout Components

#### Stack Layout
```tsx
<Stack directionType={Stack.DIRECTION_TYPE.VERTICAL} gapType={Stack.GAP_TYPE.MEDIUM}>
  <StackItem>
    <HeadingText type={HeadingText.TYPE.HEADING_1}>
      Message Queues Dashboard
    </HeadingText>
  </StackItem>
  <StackItem grow>
    <MainContent />
  </StackItem>
</Stack>
```

#### Grid System
```tsx
<Grid spacingType={[Grid.SPACING_TYPE.MEDIUM]}>
  <GridItem columnSpan={3}>
    <FilterPanel />
  </GridItem>
  <GridItem columnSpan={9}>
    <DataDisplay />
  </GridItem>
</Grid>
```

### Data Fetching Components

#### NrqlQuery Component
```tsx
<NrqlQuery
  accountIds={[accountId]}
  query={nrqlQuery}
  pollInterval={60000}
  formatType={NrqlQuery.FORMAT_TYPE.CHART}
>
  {({ data, loading, error }) => {
    if (loading) return <Spinner />;
    if (error) return <EmptyState type={EmptyState.TYPE.ERROR} />;
    return <ChartDisplay data={data} />;
  }}
</NrqlQuery>
```

#### NerdGraphQuery Hook
```typescript
import { useNerdGraphQuery } from 'nr1';

const { loading, error, data } = useNerdGraphQuery({
  query: ALL_KAFKA_TABLE_QUERY,
  variables: {
    awsQuery: AWS_CLUSTER_QUERY_FILTER_FUNC(filters),
    confluentCloudQuery: CONFLUENT_CLOUD_QUERY_FILTER_CLUSTER_FUNC(filters),
    facet: 'ACCOUNT_ID',
    orderBy: 'ASC'
  }
});
```

## Platform State Management

### NerdletStateContext

Provides shared state across components within a nerdlet:

```typescript
import { NerdletStateContext, nerdlet } from 'nr1';

// Setting state
const setNerdletState = useCallback((updates) => {
  nerdlet.setNerdletState(updates);
}, []);

// Reading state
const [nerdletState, setNerdletState] = useNerdletState();
const { filters, selectedEntity, timeRange } = nerdletState;
```

### PlatformStateContext

Access to platform-wide state:

```typescript
const { timeRange, accountId, accountIds } = usePlatformState();

// Time range from platform picker
console.log(timeRange); // { begin_time, end_time, duration }
```

### URL State Management

Deep linking support through URL state:

```typescript
// Setting URL state
navigation.openStackedNerdlet({
  id: 'message-queues.summary',
  urlState: {
    provider: 'AWS_MSK',
    account: accountId,
    filters: JSON.stringify(filters),
    selectedEntity: entityGuid
  }
});

// Reading URL state
const { provider, account, filters, selectedEntity } = nerdlet.getUrlState();
```

## Navigation System

### Navigation API

```typescript
import { navigation } from 'nr1';

// Navigate to entity
navigation.openEntity(entityGuid);

// Open stacked nerdlet
navigation.openStackedNerdlet({
  id: 'message-queues.mq-detail',
  urlState: { /* state */ }
});

// Replace current nerdlet
navigation.replaceNerdlet({
  id: 'message-queues.summary'
});

// Open launcher
navigation.openLauncher('message-queues');
```

### Navigation Patterns

```typescript
// Navigation flow handler
const handleNavigation = {
  toSummary: (account, provider) => {
    navigation.openStackedNerdlet({
      id: 'message-queues.summary',
      urlState: { account, provider }
    });
  },
  
  toEntity: (entityGuid) => {
    navigation.openEntity(entityGuid);
  },
  
  toDetail: (entity) => {
    navigation.openStackedNerdlet({
      id: 'message-queues.mq-detail',
      urlState: {
        entityGuid: entity.guid,
        entityType: entity.type
      }
    });
  }
};
```

## Platform Hooks

### Custom Hooks Usage

```typescript
// usePlatformState - Access platform context
const { accountId, timeRange } = usePlatformState();

// useNerdletState - Shared component state
const [state, setState] = useNerdletState();

// useEntityQuery - Fetch entity data
const { data: entity } = useEntityQuery(entityGuid);

// useNrqlQuery - Execute NRQL queries
const { data, loading } = useNrqlQuery({
  accountIds: [accountId],
  query: nrqlQuery
});

// useInterval - Polling mechanism
useInterval(() => {
  refetchData();
}, 60000); // Poll every minute
```

### Storage Hooks

```typescript
// User-specific storage
const [preferences, setPreferences] = useUserStorageValue('messageQueues.preferences');

// Account-level storage
const [config, setConfig] = useAccountStorageValue('messageQueues.config');

// Entity-specific storage
const [entityData, setEntityData] = useEntityStorageValue(entityGuid, 'customData');
```

## Visualization Framework

### Custom Visualizations

```typescript
// visualizations/kafka-infra-map/index.tsx
import { VisualizationWidget } from 'nr1';

export default class KafkaInfraMap extends VisualizationWidget {
  static propTypes = {
    entities: PropTypes.array.isRequired,
    viewMode: PropTypes.string,
    metricMode: PropTypes.string
  };

  render() {
    const { entities, viewMode, metricMode } = this.props;
    return (
      <HoneyCombVisualization
        entities={entities}
        viewMode={viewMode}
        metricMode={metricMode}
      />
    );
  }
}
```

### Visualization Configuration

```json
// visualizations/kafka-infra-map/nr1.json
{
  "schemaType": "VISUALIZATION",
  "id": "kafka-infra-map",
  "displayName": "Kafka Infrastructure Map",
  "description": "Interactive hexagonal visualization of Kafka topology",
  "configuration": [
    {
      "name": "viewMode",
      "title": "View Mode",
      "type": "enum",
      "items": [
        { "title": "Clusters", "value": "cluster" },
        { "title": "Brokers", "value": "broker" },
        { "title": "Topics", "value": "topic" }
      ]
    }
  ]
}
```

## Platform APIs

### NerdGraph Integration

```typescript
import { NerdGraphQuery, ngql } from 'nr1';

// GraphQL query with template literal
const ENTITY_SEARCH_QUERY = ngql`
  query EntitySearch($query: String!) {
    actor {
      entitySearch(query: $query) {
        count
        results {
          entities {
            guid
            name
            type
            alertSeverity
            tags {
              key
              values
            }
          }
        }
      }
    }
  }
`;

// Execute query
const result = await NerdGraphQuery.query({
  query: ENTITY_SEARCH_QUERY,
  variables: { query: entitySearchQuery }
});
```

### Entity APIs

```typescript
// Entity synthesis and relationships
const EntityAPI = {
  // Fetch entity details
  getEntity: async (guid) => {
    const query = ngql`
      query GetEntity($guid: EntityGuid!) {
        actor {
          entity(guid: $guid) {
            name
            type
            domain
            alertSeverity
            relationships {
              source {
                entity {
                  guid
                  name
                }
              }
              target {
                entity {
                  guid
                  name
                }
              }
              type
            }
          }
        }
      }
    `;
    return NerdGraphQuery.query({ query, variables: { guid } });
  }
};
```

## Performance Considerations

### Platform Optimization

```typescript
// Memoization for expensive operations
const memoizedData = useMemo(() => {
  return processComplexData(rawData);
}, [rawData]);

// Callback optimization
const stableCallback = useCallback((item) => {
  handleItemClick(item);
}, []);

// Component optimization
const OptimizedComponent = React.memo(({ data }) => {
  return <ExpensiveVisualization data={data} />;
}, (prevProps, nextProps) => {
  return isEqual(prevProps.data, nextProps.data);
});
```

### Query Optimization

```typescript
// Batch queries for efficiency
const batchQuery = ngql`
  query BatchEntityData($guids: [EntityGuid]!) {
    actor {
      entities(guids: $guids) {
        guid
        name
        ... on InfrastructureAwsKafkaClusterEntity {
          clusterMetrics {
            activeControllers
            offlinePartitions
          }
        }
      }
    }
  }
`;
```

## Error Handling

### Platform Error Boundaries

```typescript
import { ErrorBoundary } from 'nr1';

class NerdletErrorBoundary extends React.Component {
  componentDidCatch(error, errorInfo) {
    // Log to New Relic
    instrumentation.noticeError(error, {
      component: 'MessageQueues',
      errorInfo
    });
  }

  render() {
    if (this.state.hasError) {
      return (
        <EmptyState
          type={EmptyState.TYPE.ERROR}
          iconType={EmptyState.ICON_TYPE.HARDWARE_AND_SOFTWARE__SOFTWARE__SERVICE__S_ERROR}
          title="An error occurred"
          description="Please refresh the page or contact support"
        />
      );
    }
    return this.props.children;
  }
}
```

## Security Integration

### Platform Security Features

```typescript
// Account access validation
const validateAccess = async (accountId) => {
  try {
    const { data } = await NerdGraphQuery.query({
      query: ngql`
        query ValidateAccess($accountId: Int!) {
          actor {
            account(id: $accountId) {
              id
              name
            }
          }
        }
      `,
      variables: { accountId }
    });
    return !!data.actor.account;
  } catch (error) {
    return false;
  }
};
```

## Development Tools

### Platform CLI

```bash
# Create new nerdpack
nr1 create --type nerdpack --name message-queues

# Local development
nr1 nerdpack:serve

# UUID management
nr1 nerdpack:uuid --generate

# Validation
nr1 nerdpack:validate

# Publishing workflow
nr1 nerdpack:publish
nr1 nerdpack:deploy -c STABLE
nr1 nerdpack:subscribe -c STABLE
```

### Development Helpers

```typescript
// Platform logging
import { logger } from 'nr1';

logger.debug('Debug message', { context: data });
logger.info('Info message');
logger.warn('Warning message');
logger.error('Error message', error);

// Feature flags
if (platform.getFeatureFlag('message-queues.new-feature')) {
  // New feature code
}
```

## Best Practices

### 1. Component Structure
- Keep nerdlets focused on single responsibilities
- Use shared components for reusability
- Implement proper error boundaries

### 2. State Management
- Use URL state for deep linking
- Leverage nerdlet state for cross-component communication
- Keep local state minimal

### 3. Performance
- Implement virtual scrolling for large lists
- Use memoization for expensive calculations
- Batch API calls when possible

### 4. User Experience
- Provide loading states for all async operations
- Show meaningful error messages
- Implement progressive disclosure

### 5. Platform Integration
- Follow NR1 design guidelines
- Use platform components consistently
- Respect platform conventions

## Conclusion

The NR1 platform provides a robust foundation for building sophisticated monitoring applications. By leveraging its component library, state management, and API capabilities, the Message Queues application delivers a seamless, integrated experience within the New Relic ecosystem.