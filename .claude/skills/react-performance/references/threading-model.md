# React Native Threading Model

Understanding how threads interact helps diagnose performance issues and write efficient code.

## Thread Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    React Native App                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  UI Thread   │  │  JS Thread   │  │ Background Thread│   │
│  │  (Main)      │  │              │  │                  │   │
│  ├──────────────┤  ├──────────────┤  ├──────────────────┤   │
│  │ • Layout     │  │ • React      │  │ • Turbo Modules  │   │
│  │ • Drawing    │  │ • Business   │  │ • Native async   │   │
│  │ • Touch      │  │   logic      │  │   operations     │   │
│  │ • Animations │  │ • Hermes     │  │                  │   │
│  │   (native)   │  │   execution  │  │                  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
│         │                 │                   │              │
│         └────────────────JSI─────────────────┘              │
│                (Synchronous binding)                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Threads Explained

### UI Thread (Main Thread)

- Handles all native rendering
- Processes touch events
- Runs native animations
- Must stay under 16ms/frame for 60 FPS

**If blocked**: UI freezes, animations jank, touches delayed

### JS Thread

- Runs Hermes JavaScript engine
- Executes React component logic
- Handles state updates and re-renders
- Communicates with native via JSI

**If blocked**: React becomes unresponsive, updates delayed

### Background Threads

- Turbo Modules can spawn own threads
- Network requests
- Database operations
- Heavy computations

## New Architecture (Fabric + JSI)

### JSI (JavaScript Interface)

Direct synchronous binding between JS and native, replacing the old async bridge.

```
Old Architecture:                 New Architecture:
┌────┐  async   ┌────────┐       ┌────┐  sync   ┌────────┐
│ JS │ ──────── │ Bridge │       │ JS │ ─────── │  JSI   │
└────┘ (batch)  └────────┘       └────┘ (direct)└────────┘
         │                                │
     ┌───┴───┐                       ┌────┴────┐
     │Native │                       │ Native  │
     └───────┘                       └─────────┘
```

**Benefits of JSI**:
- No serialization overhead
- Synchronous calls when needed
- Direct memory access
- Better performance for frequent calls

### Fabric (New Renderer)

C++ rendering layer with direct JSI integration.

```tsx
// Same React code, but Fabric handles rendering
const Component = () => (
  <View style={styles.container}>
    <Text>Hello Fabric</Text>
  </View>
);
```

**Fabric benefits**:
- Synchronous layout
- Better scroll performance
- Concurrent rendering support
- Reduced bridge overhead

### Turbo Modules

Lazy-loaded native modules with JSI bindings.

```tsx
// Turbo Module: loaded only when first accessed
import { TurboModuleRegistry } from 'react-native';

const MyModule = TurboModuleRegistry.get<MyModuleSpec>('MyModule');
```

**Benefits**:
- Lazy initialization (faster startup)
- Synchronous calls via JSI
- Type-safe with codegen

## Thread Communication

### Old Bridge (Deprecated)

```
JS Thread                           UI Thread
    │                                   │
    │ 1. setState({ count: 1 })         │
    │                                   │
    │──── serialize to JSON ────────────│
    │                                   │
    │ 2. Batch updates                  │
    │                                   │
    │──── async message ────────────────│
    │                                   │
    │                    3. Deserialize │
    │                    4. Apply       │
    │                                   │
```

**Problems**: Async overhead, serialization cost, batching delays

### New JSI Communication

```
JS Thread                           Native
    │                                   │
    │ 1. Direct call via JSI            │
    │───────────────────────────────────│
    │                    2. Execute     │
    │                    3. Return      │
    │───────────────────────────────────│
    │ 4. Continue                       │
    │                                   │
```

**Benefits**: No serialization, synchronous when needed

## Performance Implications

### Keep UI Thread Free

```tsx
// ❌ Heavy work on UI thread (via useLayoutEffect)
useLayoutEffect(() => {
  const result = expensiveCalculation(); // Blocks UI
  setData(result);
}, []);

// ✅ Heavy work on JS thread
useEffect(() => {
  const result = expensiveCalculation(); // On JS thread
  setData(result);
}, []);
```

### Keep JS Thread Responsive

```tsx
// ❌ Blocks JS thread
const processData = (items: Item[]) => {
  return items.map(heavyTransform); // Long-running
};

// ✅ Chunk work to stay responsive
const processData = async (items: Item[]) => {
  const results: Item[] = [];
  const chunkSize = 100;
  
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    results.push(...chunk.map(heavyTransform));
    
    // Yield to allow other updates
    await new Promise(resolve => setTimeout(resolve, 0));
  }
  
  return results;
};
```

### Move Work Off Main Threads

```tsx
// Modern: defer with transition
startTransition(() => {
  setProcessedData(heavyComputation(data));
});

// For truly background work: Turbo Module
const result = await TurboModule.computeInBackground(data);
```

> ⚠️ **Legacy approach**: `InteractionManager.runAfterInteractions()` waits for native animations to complete. Still useful when you need to sync specifically with native animation timing, but `startTransition` covers most cases. See [concurrent-react.md](concurrent-react.md).

## Diagnosing Thread Issues

### Perf Monitor

Shows FPS for both threads:
- **UI FPS drops** → Native/UI thread issue
- **JS FPS drops** → JavaScript thread issue
- **Both drop** → Complex issue, often JS triggering UI work

### Common Patterns

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| UI jank, JS FPS fine | Heavy native work | Optimize layouts, use native driver |
| JS FPS drops, UI fine | Heavy JS computation | Defer work, optimize algorithms |
| Both drop during scroll | Re-renders during scroll | Memoize, optimize FlatList |
| Lag on navigation | Heavy screen initialization | Lazy load, defer work |

## Thread-Aware Patterns

### Animations

```tsx
// Run on UI thread (via native driver or Reanimated)
Animated.timing(opacity, {
  toValue: 1,
  useNativeDriver: true, // UI thread
});
```

### Heavy Computations

```tsx
// Keep off both main threads if possible
const result = await computeInBackground(data);
// Turbo Modules can run on dedicated threads
```

### React Updates

```tsx
// Non-blocking updates with transitions
startTransition(() => {
  setExpensiveState(newValue); // Can be interrupted
});
```

## Checklist

```
□ Animations use native driver or Reanimated
□ Heavy computations chunked or deferred with startTransition
□ Understanding which thread is bottleneck (Perf Monitor)
□ Turbo Modules for native performance-critical code
□ startTransition for non-urgent React updates
```
