# Memory Management

This file helps Claude identify memory leak patterns in code and guide users through leak detection.

## Leak Patterns to Identify in Code

When reviewing code, look for these common leak patterns:

### 1. Uncleared Subscriptions

```tsx
// 🚩 LEAK: listener never removed
useEffect(() => {
  const subscription = EventEmitter.addListener('event', handler);
  // Missing cleanup!
}, []);

// ✅ FIXED: cleanup on unmount
useEffect(() => {
  const subscription = EventEmitter.addListener('event', handler);
  return () => subscription.remove();
}, []);
```

**What to look for:** Any `addListener`, `subscribe`, `on()` without corresponding removal in useEffect return.

### 2. Uncleared Timers

```tsx
// 🚩 LEAK: interval runs forever
useEffect(() => {
  setInterval(() => doSomething(), 1000);
  // Missing cleanup!
}, []);

// ✅ FIXED: clear on unmount
useEffect(() => {
  const id = setInterval(() => doSomething(), 1000);
  return () => clearInterval(id);
}, []);
```

**What to look for:** `setInterval`, `setTimeout` without `clearInterval`, `clearTimeout` in cleanup.

### 3. Animated Values Without Stop

```tsx
// 🚩 LEAK: animation continues after unmount
useEffect(() => {
  Animated.loop(
    Animated.timing(opacity, { toValue: 1, duration: 1000 })
  ).start();
  // Animation never stopped!
}, []);

// ✅ FIXED: stop animation on unmount
useEffect(() => {
  const animation = Animated.loop(
    Animated.timing(opacity, { toValue: 1, duration: 1000 })
  );
  animation.start();
  return () => animation.stop();
}, []);
```

**What to look for:** `Animated.timing`, `Animated.loop`, `Animated.spring` without `.stop()` in cleanup.

### 4. Closures Capturing Large Data

```tsx
// 🚩 LEAK: closure retains entire array
const createHandler = (largeData: string[]) => {
  return () => largeData.length; // Captures entire array
};

// ✅ FIXED: capture only needed value
const createHandler = (largeData: string[]) => {
  const length = largeData.length;
  return () => length; // Only captures primitive
};
```

**What to look for:** Callbacks or handlers that reference large objects/arrays when they only need a small piece.

### 5. Unbounded Global Caches

```tsx
// 🚩 LEAK: cache grows indefinitely
const cache: Record<string, object> = {};

const fetchData = async (id: string) => {
  const data = await api.get(id);
  cache[id] = data; // Never evicted
  return data;
};

// ✅ FIXED: bounded cache with eviction
const cache = new Map<string, object>();
const MAX_CACHE = 100;

const fetchData = async (id: string) => {
  if (cache.size >= MAX_CACHE) {
    const firstKey = cache.keys().next().value;
    cache.delete(firstKey);
  }
  cache.set(id, data);
  return data;
};
```

**What to look for:** Module-level objects/Maps that only add entries, never remove.

### 6. Refs Holding Large Objects

```tsx
// 🚩 Potential issue: ref holds reference until unmount
const ref = useRef<HeavyObject | null>(null);

useEffect(() => {
  ref.current = createHeavyObject();
  // No cleanup
}, []);

// ✅ Better: explicit cleanup
useEffect(() => {
  ref.current = createHeavyObject();
  return () => { ref.current = null; };
}, []);
```

**What to look for:** `useRef` holding objects that should be released before component unmounts.

### 7. Fetch/WebSocket Without Abort

```tsx
// 🚩 LEAK: request completes after unmount, updates state
useEffect(() => {
  fetch('/api/data').then(res => res.json()).then(setData);
  // Can't cancel!
}, []);

// ✅ FIXED: abort on unmount
useEffect(() => {
  const controller = new AbortController();
  fetch('/api/data', { signal: controller.signal })
    .then(res => res.json())
    .then(setData)
    .catch(e => { if (e.name !== 'AbortError') throw e; });
  return () => controller.abort();
}, []);
```

**What to look for:** `fetch`, `axios`, WebSocket connections without cancellation on unmount.

## Guiding Users to Detect Leaks

If user reports "app gets slower over time" or "memory keeps growing", provide these instructions:

### Quick Leak Test

```
1. Open Perf Monitor (Dev Menu → Perf Monitor)
2. Note current memory usage
3. Navigate to suspected screen
4. Go back
5. Repeat steps 3-4 ten times
6. Check memory: if it keeps growing, there's a leak
```

### Heap Snapshot Comparison

```
1. Open React Native DevTools (press 'j' in Metro)
2. Go to Memory tab
3. Take heap snapshot (baseline)
4. Perform the leaky action several times
5. Take another snapshot
6. Select snapshot 2 → "Comparison" view
7. Share objects with growing count or large retained size
```

## Interpreting User-Reported Results

If user shares memory profiling results:

| User Reports | Interpretation | Where to Look |
|--------------|----------------|---------------|
| "Memory grows after navigating back" | Component doesn't cleanup on unmount | useEffect returns, event listeners |
| "Heap shows growing array of closures" | Callbacks capturing data | Event handlers, callbacks passed to services |
| "Same component appearing multiple times" | Component not unmounting properly | Navigation config, conditional rendering logic |
| "Large retained size vs small shallow size" | Closure holding reference to large object | Check what variables the closure captures |

## Code Review Checklist

When reviewing code for potential memory leaks:

```
□ Every useEffect with external subscription has cleanup return
□ All setInterval/setTimeout have corresponding clear
□ Animated values call .stop() on unmount
□ Fetch requests use AbortController
□ WebSocket connections close on unmount
□ Global/module caches have size limits or eviction
□ Refs holding large objects are nullified in cleanup
□ Event listeners added with .on() or .addListener() are removed
```

## Quick Fixes to Suggest

| Leak Type | Fix |
|-----------|-----|
| Missing cleanup | Add `return () => { ... }` to useEffect |
| Timer leak | Store ID, add `clearInterval(id)` / `clearTimeout(id)` |
| Animation leak | Store animation ref, add `.stop()` in cleanup |
| Fetch leak | Add AbortController with `signal` option |
| Cache leak | Implement LRU eviction or size limit |
| Listener leak | Store subscription, call `.remove()` in cleanup |
