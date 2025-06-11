# State Management

## Overview

The Message Queues application implements a sophisticated state management system that handles application state, URL state synchronization, data caching, and cross-component communication. This document details the complete state management architecture.

## State Architecture

### State Layers

```
┌─────────────────────────────────────────────────────────┐
│                    Platform State                       │
│         (Time Range, Account Context, User)             │
├─────────────────────────────────────────────────────────┤
│                     URL State                           │
│        (Deep Linking, Navigation Context)               │
├─────────────────────────────────────────────────────────┤
│                   Nerdlet State                         │
│         (Shared State Across Components)                │
├─────────────────────────────────────────────────────────┤
│                   Local State                           │
│         (Component-Specific UI State)                   │
├─────────────────────────────────────────────────────────┤
│                   Cache Layer                           │
│         (Performance Optimization)                      │
└─────────────────────────────────────────────────────────┘
```

## Platform State Integration

### Accessing Platform State

```typescript
import { usePlatformState } from 'nr1';

interface PlatformStateData {
  timeRange: {
    begin_time: number;
    end_time: number;
    duration: number;
  };
  accountId: string;
  accountIds: string[];
  user: {
    email: string;
    id: string;
    name: string;
  };
}

const MyComponent: React.FC = () => {
  const platformState = usePlatformState();
  
  // Extract commonly used values
  const {
    timeRange,
    accountId,
    accountIds
  } = platformState;
  
  // React to platform state changes
  useEffect(() => {
    // Refetch data when time range changes
    if (timeRange) {
      fetchDataForTimeRange(timeRange);
    }
  }, [timeRange]);
  
  return (
    <div>
      <p>Current account: {accountId}</p>
      <p>Time window: {timeRange.duration}ms</p>
    </div>
  );
};
```

### Platform State Context Provider

```typescript
const PlatformStateProvider: React.FC = ({ children }) => {
  const platformState = usePlatformState();
  const [enhancedState, setEnhancedState] = useState({
    ...platformState,
    // Add computed values
    isMultiAccount: platformState.accountIds?.length > 1,
    timeRangeLabel: getTimeRangeLabel(platformState.timeRange)
  });
  
  // Update enhanced state when platform state changes
  useEffect(() => {
    setEnhancedState({
      ...platformState,
      isMultiAccount: platformState.accountIds?.length > 1,
      timeRangeLabel: getTimeRangeLabel(platformState.timeRange)
    });
  }, [platformState]);
  
  return (
    <PlatformContext.Provider value={enhancedState}>
      {children}
    </PlatformContext.Provider>
  );
};
```

## URL State Management

### URL State Schema

```typescript
interface URLStateSchema {
  // Navigation context
  provider?: 'AWS_MSK' | 'CONFLUENT_CLOUD';
  account?: string;
  accountName?: string;
  
  // Entity selection
  entityGuid?: string;
  entityType?: string;
  selectedEntities?: string[]; // Multiple selection
  
  // View configuration
  view?: 'home' | 'summary' | 'detail';
  navigatorView?: 'cluster' | 'broker' | 'topic';
  navigatorMetric?: 'health' | 'alerts' | 'throughput';
  
  // Filters
  filters?: SerializedFilters;
  search?: string;
  
  // UI state
  expandedSections?: string[];
  entityDetailOpen?: boolean;
  panelWidth?: 'normal' | 'expanded';
  
  // Time range override
  timeRange?: SerializedTimeRange;
}
```

### URL State Hook

```typescript
import { useNerdletState } from 'nr1';

const useURLState = <T extends Partial<URLStateSchema>>() => {
  const [urlState, setURLState] = useNerdletState<T>();
  
  // Serialize complex objects for URL
  const setURLStateWithSerialization = useCallback((updates: Partial<T>) => {
    const serialized = Object.entries(updates).reduce((acc, [key, value]) => {
      if (typeof value === 'object' && value !== null) {
        acc[key] = JSON.stringify(value);
      } else {
        acc[key] = value;
      }
      return acc;
    }, {} as any);
    
    setURLState(prev => ({ ...prev, ...serialized }));
  }, [setURLState]);
  
  // Deserialize complex objects from URL
  const deserializedState = useMemo(() => {
    return Object.entries(urlState).reduce((acc, [key, value]) => {
      if (typeof value === 'string' && value.startsWith('{') || value.startsWith('[')) {
        try {
          acc[key] = JSON.parse(value);
        } catch {
          acc[key] = value;
        }
      } else {
        acc[key] = value;
      }
      return acc;
    }, {} as T);
  }, [urlState]);
  
  return [deserializedState, setURLStateWithSerialization] as const;
};
```

### Deep Linking Implementation

```typescript
class DeepLinkManager {
  // Generate shareable link
  static generateDeepLink(state: URLStateSchema): string {
    const baseUrl = window.location.origin + window.location.pathname;
    const params = new URLSearchParams();
    
    Object.entries(state).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        if (typeof value === 'object') {
          params.set(key, JSON.stringify(value));
        } else {
          params.set(key, String(value));
        }
      }
    });
    
    return `${baseUrl}?${params.toString()}`;
  }
  
  // Parse deep link
  static parseDeepLink(url: string): URLStateSchema {
    const params = new URLSearchParams(url.split('?')[1] || '');
    const state: URLStateSchema = {};
    
    params.forEach((value, key) => {
      try {
        // Try to parse as JSON first
        if (value.startsWith('{') || value.startsWith('[')) {
          state[key] = JSON.parse(value);
        } else {
          state[key] = value;
        }
      } catch {
        state[key] = value;
      }
    });
    
    return state;
  }
}
```

## Nerdlet State Management

### Shared State Structure

```typescript
interface NerdletSharedState {
  // Home screen state
  homeFilters: {
    provider: string[];
    account: string[];
    status: string;
    clusters: string[];
    topics: string[];
    search: string;
  };
  
  // Summary screen state
  summaryFilters: {
    clusterName?: string;
    topicFilter?: string;
    brokerFilter?: string;
    healthFilter?: string;
  };
  
  // Navigator configuration
  navigatorConfig: {
    show: 'cluster' | 'broker' | 'topic';
    metric: 'health' | 'alerts' | 'throughput';
    groupBy: string;
  };
  
  // Selected entities
  selectedEntities: {
    guid: string;
    type: string;
    name: string;
  }[];
  
  // UI preferences
  preferences: {
    tablePageSize: number;
    chartTimeRange: string;
    refreshInterval: number;
  };
}
```

### Nerdlet State Hook

```typescript
const useNerdletSharedState = () => {
  const [nerdletState, setNerdletState] = useNerdletState<NerdletSharedState>();
  
  // State updaters with validation
  const updateHomeFilters = useCallback((filters: Partial<HomeFilters>) => {
    setNerdletState(prev => ({
      ...prev,
      homeFilters: {
        ...prev.homeFilters,
        ...filters
      }
    }));
  }, [setNerdletState]);
  
  const updateNavigatorConfig = useCallback((config: Partial<NavigatorConfig>) => {
    setNerdletState(prev => ({
      ...prev,
      navigatorConfig: {
        ...prev.navigatorConfig,
        ...config
      }
    }));
  }, [setNerdletState]);
  
  const selectEntity = useCallback((entity: SelectedEntity) => {
    setNerdletState(prev => ({
      ...prev,
      selectedEntities: [...(prev.selectedEntities || []), entity]
    }));
  }, [setNerdletState]);
  
  return {
    state: nerdletState,
    updateHomeFilters,
    updateNavigatorConfig,
    selectEntity,
    clearSelection: () => setNerdletState(prev => ({ ...prev, selectedEntities: [] }))
  };
};
```

### Cross-Component Communication

```typescript
// Event emitter for complex state coordination
class StateEventEmitter extends EventEmitter {
  private static instance: StateEventEmitter;
  
  static getInstance(): StateEventEmitter {
    if (!this.instance) {
      this.instance = new StateEventEmitter();
    }
    return this.instance;
  }
  
  emitFilterChange(filters: any): void {
    this.emit('filters:changed', filters);
  }
  
  emitEntitySelection(entity: any): void {
    this.emit('entity:selected', entity);
  }
  
  emitNavigationRequest(target: string, params: any): void {
    this.emit('navigation:requested', { target, params });
  }
}

// Usage in components
const MyComponent: React.FC = () => {
  const emitter = StateEventEmitter.getInstance();
  
  useEffect(() => {
    const handleFilterChange = (filters) => {
      // React to filter changes from other components
      updateLocalState(filters);
    };
    
    emitter.on('filters:changed', handleFilterChange);
    return () => emitter.off('filters:changed', handleFilterChange);
  }, []);
  
  const applyFilter = (newFilter) => {
    // Emit event for other components
    emitter.emitFilterChange(newFilter);
  };
};
```

## Local State Patterns

### Component State Management

```typescript
// Custom hook for complex component state
const useComponentState = <T extends object>(initialState: T) => {
  const [state, setState] = useState<T>(initialState);
  const [history, setHistory] = useState<T[]>([initialState]);
  const [historyIndex, setHistoryIndex] = useState(0);
  
  // State updater with history
  const updateState = useCallback((updates: Partial<T> | ((prev: T) => T)) => {
    setState(prev => {
      const newState = typeof updates === 'function' 
        ? updates(prev) 
        : { ...prev, ...updates };
      
      // Add to history
      setHistory(h => [...h.slice(0, historyIndex + 1), newState]);
      setHistoryIndex(i => i + 1);
      
      return newState;
    });
  }, [historyIndex]);
  
  // Undo/redo functionality
  const undo = useCallback(() => {
    if (historyIndex > 0) {
      setHistoryIndex(i => i - 1);
      setState(history[historyIndex - 1]);
    }
  }, [history, historyIndex]);
  
  const redo = useCallback(() => {
    if (historyIndex < history.length - 1) {
      setHistoryIndex(i => i + 1);
      setState(history[historyIndex + 1]);
    }
  }, [history, historyIndex]);
  
  return {
    state,
    updateState,
    undo,
    redo,
    canUndo: historyIndex > 0,
    canRedo: historyIndex < history.length - 1
  };
};
```

### Form State Management

```typescript
interface FormState<T> {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  isSubmitting: boolean;
  isValid: boolean;
}

const useFormState = <T extends object>(
  initialValues: T,
  validationSchema: ValidationSchema<T>
) => {
  const [formState, setFormState] = useState<FormState<T>>({
    values: initialValues,
    errors: {},
    touched: {},
    isSubmitting: false,
    isValid: true
  });
  
  // Field change handler
  const handleChange = useCallback((field: keyof T, value: any) => {
    setFormState(prev => {
      const newValues = { ...prev.values, [field]: value };
      const errors = validationSchema.validate(newValues);
      
      return {
        ...prev,
        values: newValues,
        errors,
        touched: { ...prev.touched, [field]: true },
        isValid: Object.keys(errors).length === 0
      };
    });
  }, [validationSchema]);
  
  // Form submission
  const handleSubmit = useCallback(async (onSubmit: (values: T) => Promise<void>) => {
    setFormState(prev => ({ ...prev, isSubmitting: true }));
    
    try {
      await onSubmit(formState.values);
      // Reset form on success
      setFormState({
        values: initialValues,
        errors: {},
        touched: {},
        isSubmitting: false,
        isValid: true
      });
    } catch (error) {
      setFormState(prev => ({ 
        ...prev, 
        isSubmitting: false,
        errors: { ...prev.errors, submit: error.message }
      }));
    }
  }, [formState.values, initialValues]);
  
  return {
    values: formState.values,
    errors: formState.errors,
    touched: formState.touched,
    isSubmitting: formState.isSubmitting,
    isValid: formState.isValid,
    handleChange,
    handleSubmit,
    resetForm: () => setFormState({
      values: initialValues,
      errors: {},
      touched: {},
      isSubmitting: false,
      isValid: true
    })
  };
};
```

## Cache Management

### Cache Architecture

```typescript
interface CacheEntry<T> {
  data: T;
  timestamp: number;
  ttl: number;
  tags: string[];
  dependsOn?: string[];
}

class CacheManager {
  private cache = new Map<string, CacheEntry<any>>();
  private subscribers = new Map<string, Set<(data: any) => void>>();
  
  // Set cache with dependencies
  set<T>(key: string, data: T, options: CacheOptions = {}): void {
    const entry: CacheEntry<T> = {
      data,
      timestamp: Date.now(),
      ttl: options.ttl || 300000, // 5 minutes default
      tags: options.tags || [],
      dependsOn: options.dependsOn || []
    };
    
    this.cache.set(key, entry);
    this.notifySubscribers(key, data);
    
    // Schedule cleanup
    setTimeout(() => this.evict(key), entry.ttl);
  }
  
  // Get with automatic refresh
  async get<T>(
    key: string,
    fetcher?: () => Promise<T>
  ): Promise<T | null> {
    const entry = this.cache.get(key);
    
    if (!entry) {
      if (fetcher) {
        const data = await fetcher();
        this.set(key, data);
        return data;
      }
      return null;
    }
    
    // Check if expired
    if (Date.now() - entry.timestamp > entry.ttl) {
      this.evict(key);
      if (fetcher) {
        const data = await fetcher();
        this.set(key, data);
        return data;
      }
      return null;
    }
    
    // Check dependencies
    if (entry.dependsOn?.some(dep => !this.cache.has(dep))) {
      this.evict(key);
      if (fetcher) {
        const data = await fetcher();
        this.set(key, data);
        return data;
      }
      return null;
    }
    
    return entry.data;
  }
  
  // Invalidate by tags
  invalidateByTags(tags: string[]): void {
    const keysToInvalidate = new Set<string>();
    
    this.cache.forEach((entry, key) => {
      if (tags.some(tag => entry.tags.includes(tag))) {
        keysToInvalidate.add(key);
      }
    });
    
    keysToInvalidate.forEach(key => this.evict(key));
  }
  
  // Subscribe to cache updates
  subscribe(key: string, callback: (data: any) => void): () => void {
    if (!this.subscribers.has(key)) {
      this.subscribers.set(key, new Set());
    }
    
    this.subscribers.get(key)!.add(callback);
    
    // Return unsubscribe function
    return () => {
      this.subscribers.get(key)?.delete(callback);
    };
  }
  
  private evict(key: string): void {
    // Evict dependent entries
    this.cache.forEach((entry, k) => {
      if (entry.dependsOn?.includes(key)) {
        this.evict(k);
      }
    });
    
    this.cache.delete(key);
    this.notifySubscribers(key, null);
  }
  
  private notifySubscribers(key: string, data: any): void {
    this.subscribers.get(key)?.forEach(callback => callback(data));
  }
}
```

### React Cache Hook

```typescript
const useCache = <T>(
  key: string,
  fetcher: () => Promise<T>,
  options?: CacheOptions
) => {
  const cacheManager = useMemo(() => CacheManager.getInstance(), []);
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<Error | null>(null);
  
  useEffect(() => {
    let cancelled = false;
    
    const loadData = async () => {
      setLoading(true);
      setError(null);
      
      try {
        const cachedData = await cacheManager.get(key, fetcher);
        if (!cancelled) {
          setData(cachedData);
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
    
    loadData();
    
    // Subscribe to cache updates
    const unsubscribe = cacheManager.subscribe(key, (newData) => {
      if (!cancelled) {
        setData(newData);
      }
    });
    
    return () => {
      cancelled = true;
      unsubscribe();
    };
  }, [key, cacheManager]);
  
  const refresh = useCallback(async () => {
    cacheManager.evict(key);
    const newData = await fetcher();
    cacheManager.set(key, newData, options);
    setData(newData);
  }, [key, fetcher, options, cacheManager]);
  
  return { data, loading, error, refresh };
};
```

## State Persistence

### Local Storage Integration

```typescript
class StateStorage {
  private readonly prefix = 'messageQueues:';
  
  // Save state to local storage
  save(key: string, state: any): void {
    try {
      const serialized = JSON.stringify(state);
      localStorage.setItem(this.prefix + key, serialized);
    } catch (error) {
      console.error('Failed to save state:', error);
    }
  }
  
  // Load state from local storage
  load<T>(key: string, defaultValue: T): T {
    try {
      const item = localStorage.getItem(this.prefix + key);
      if (!item) return defaultValue;
      
      return JSON.parse(item) as T;
    } catch (error) {
      console.error('Failed to load state:', error);
      return defaultValue;
    }
  }
  
  // Clear specific state
  clear(key: string): void {
    localStorage.removeItem(this.prefix + key);
  }
  
  // Clear all application state
  clearAll(): void {
    Object.keys(localStorage)
      .filter(key => key.startsWith(this.prefix))
      .forEach(key => localStorage.removeItem(key));
  }
}

// Hook for persistent state
const usePersistentState = <T>(
  key: string,
  defaultValue: T
): [T, (value: T) => void] => {
  const storage = useMemo(() => new StateStorage(), []);
  
  const [state, setState] = useState<T>(() => 
    storage.load(key, defaultValue)
  );
  
  const setPersistentState = useCallback((value: T) => {
    setState(value);
    storage.save(key, value);
  }, [key, storage]);
  
  return [state, setPersistentState];
};
```

### Session Storage for Temporary State

```typescript
const useSessionState = <T>(
  key: string,
  defaultValue: T
): [T, (value: T) => void] => {
  const [state, setState] = useState<T>(() => {
    try {
      const item = sessionStorage.getItem(key);
      return item ? JSON.parse(item) : defaultValue;
    } catch {
      return defaultValue;
    }
  });
  
  const setSessionState = useCallback((value: T) => {
    setState(value);
    try {
      sessionStorage.setItem(key, JSON.stringify(value));
    } catch (error) {
      console.error('Failed to save to session storage:', error);
    }
  }, [key]);
  
  return [state, setSessionState];
};
```

## State Synchronization

### Multi-Tab Synchronization

```typescript
class StateSynchronizer {
  private channel: BroadcastChannel;
  private listeners = new Map<string, Set<(data: any) => void>>();
  
  constructor(channelName: string = 'messageQueues:state') {
    this.channel = new BroadcastChannel(channelName);
    
    this.channel.onmessage = (event) => {
      const { type, key, data } = event.data;
      
      if (type === 'state:update') {
        this.notifyListeners(key, data);
      }
    };
  }
  
  broadcast(key: string, data: any): void {
    this.channel.postMessage({
      type: 'state:update',
      key,
      data,
      timestamp: Date.now()
    });
  }
  
  subscribe(key: string, callback: (data: any) => void): () => void {
    if (!this.listeners.has(key)) {
      this.listeners.set(key, new Set());
    }
    
    this.listeners.get(key)!.add(callback);
    
    return () => {
      this.listeners.get(key)?.delete(callback);
    };
  }
  
  private notifyListeners(key: string, data: any): void {
    this.listeners.get(key)?.forEach(callback => callback(data));
  }
}

// Hook for synchronized state
const useSynchronizedState = <T>(
  key: string,
  defaultValue: T
): [T, (value: T) => void] => {
  const synchronizer = useMemo(() => new StateSynchronizer(), []);
  const [state, setState] = useState<T>(defaultValue);
  
  useEffect(() => {
    // Subscribe to updates from other tabs
    return synchronizer.subscribe(key, (data) => {
      setState(data);
    });
  }, [key, synchronizer]);
  
  const setSynchronizedState = useCallback((value: T) => {
    setState(value);
    synchronizer.broadcast(key, value);
  }, [key, synchronizer]);
  
  return [state, setSynchronizedState];
};
```

## State DevTools

### State Inspector

```typescript
const StateInspector: React.FC = () => {
  const [isOpen, setIsOpen] = useState(false);
  const [nerdletState] = useNerdletState();
  const platformState = usePlatformState();
  
  if (!isOpen) {
    return (
      <button 
        className="state-inspector-toggle"
        onClick={() => setIsOpen(true)}
      >
        🔍
      </button>
    );
  }
  
  return (
    <div className="state-inspector">
      <div className="state-inspector-header">
        <h3>State Inspector</h3>
        <button onClick={() => setIsOpen(false)}>×</button>
      </div>
      
      <div className="state-inspector-content">
        <section>
          <h4>Platform State</h4>
          <pre>{JSON.stringify(platformState, null, 2)}</pre>
        </section>
        
        <section>
          <h4>Nerdlet State</h4>
          <pre>{JSON.stringify(nerdletState, null, 2)}</pre>
        </section>
        
        <section>
          <h4>URL State</h4>
          <pre>{JSON.stringify(parseURLState(), null, 2)}</pre>
        </section>
        
        <section>
          <h4>Cache Status</h4>
          <CacheStatus />
        </section>
      </div>
    </div>
  );
};
```

## Best Practices

### 1. State Organization
- Keep state close to where it's used
- Lift state up only when necessary
- Use appropriate state layer for the use case

### 2. Performance
- Memoize expensive state computations
- Use selective subscriptions
- Implement proper cache strategies

### 3. Persistence
- Persist user preferences
- Clear sensitive data appropriately
- Handle storage quota limits

### 4. Synchronization
- Keep URL state minimal
- Use proper serialization
- Handle conflicts gracefully

### 5. Debugging
- Use meaningful state keys
- Implement state logging
- Provide development tools

## Conclusion

The state management system provides a robust foundation for managing complex application state across multiple layers. By properly utilizing each state layer and following best practices, developers can build responsive, maintainable applications with excellent user experience.