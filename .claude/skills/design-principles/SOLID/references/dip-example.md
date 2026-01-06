# DIP Example

## Before — DIP Violation

```typescript
// SyncUseCase.ts — high-level depends on low-level
import AsyncStorage from '@react-native-async-storage/async-storage';
import analytics from '@segment/analytics-react-native';

class SyncUseCase {
  async execute(data: SyncData): Promise<void> {
    // Direct dependency on concrete implementation
    const lastSync = await AsyncStorage.getItem('lastSync');

    if (this.needsSync(lastSync)) {
      const response = await fetch('/api/sync', {
        method: 'POST',
        body: JSON.stringify(data)
      });

      await AsyncStorage.setItem('lastSync', Date.now().toString());

      // Direct dependency on analytics SDK
      analytics.track('sync_completed', { itemCount: data.items.length });
    }
  }

  private needsSync(lastSync: string | null): boolean { /* ... */ }
}
```

## After — DIP Applied

```typescript
// Ports defined in domain layer
interface StoragePort {
  get(key: string): Promise<string | null>;
  set(key: string, value: string): Promise<void>;
}

interface SyncApiPort {
  sync(data: SyncData): Promise<SyncResult>;
}

interface AnalyticsPort {
  track(event: string, properties?: Record<string, unknown>): void;
}

// SyncUseCase.ts — depends only on abstractions
class SyncUseCase {
  constructor(
    private readonly storage: StoragePort,
    private readonly syncApi: SyncApiPort,
    private readonly analytics: AnalyticsPort
  ) {}

  async execute(data: SyncData): Promise<void> {
    const lastSync = await this.storage.get('lastSync');

    if (this.needsSync(lastSync)) {
      await this.syncApi.sync(data);
      await this.storage.set('lastSync', Date.now().toString());
      this.analytics.track('sync_completed', { itemCount: data.items.length });
    }
  }

  private needsSync(lastSync: string | null): boolean { /* ... */ }
}

// AsyncStorageAdapter.ts — in infrastructure layer
class AsyncStorageAdapter implements StoragePort {
  async get(key: string): Promise<string | null> {
    return AsyncStorage.getItem(key);
  }

  async set(key: string, value: string): Promise<void> {
    await AsyncStorage.setItem(key, value);
  }
}

// Composition root wires everything together
const syncUseCase = new SyncUseCase(
  new AsyncStorageAdapter(),
  new SyncApiAdapter(httpClient),
  new SegmentAnalyticsAdapter()
);
```
