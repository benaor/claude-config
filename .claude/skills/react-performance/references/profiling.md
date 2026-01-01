# Profiling Guide

This file helps Claude assist users with performance profiling. Claude cannot run profiling tools directly but can analyze code, interpret results, and guide users through the profiling process.

## When User Reports Performance Issues

### Step 1: Clarify the Symptom

Ask the user to describe:
- **What feels slow?** (startup, scrolling, navigation, animations, specific interaction)
- **When does it happen?** (always, after X seconds, with large data)
- **Which device/platform?** (iOS/Android, simulator/real device)

### Step 2: Map Symptom to Likely Cause

| User Says | Likely Bottleneck | Suggest Profiling | Reference |
|-----------|-------------------|-------------------|-----------|
| "App takes forever to load" | Startup/TTI | Native profilers, React Profiler on mount | startup-optimization.md |
| "List scrolling is janky" | Too many re-renders or heavy items | React Profiler during scroll | list-optimization.md |
| "Animation stutters" | JS thread blocking UI | Perf Monitor (JS vs UI FPS) | animation-performance.md |
| "Gets slower over time" | Memory leak | Heap snapshots | memory-management.md |
| "Typing in input lags" | Re-renders on every keystroke | React Profiler | state-patterns.md |
| "Tab switch is slow" | Heavy screen render | React Profiler | concurrent-react.md |

### Step 3: Guide User to Profile

Provide these instructions based on the symptom:

**For re-render issues:**
```
1. Open React Native DevTools (press 'j' in Metro terminal)
2. Go to Profiler tab
3. Enable "Record why each component rendered" in settings
4. Click "Start profiling"
5. Perform the slow action
6. Stop and share the flame graph screenshot or "Why did this render?" info
```

**For FPS drops:**
```
1. Open Dev Menu (shake device or Cmd+M / Cmd+D)
2. Enable "Perf Monitor"
3. Watch JS and UI FPS while performing slow action
4. Tell me: which drops? JS, UI, or both?
```

**For memory issues:**
```
1. Open Chrome DevTools (chrome://inspect)
2. Go to Memory tab
3. Take heap snapshot before action
4. Perform action multiple times
5. Take another snapshot
6. Compare sizes and share growth
```

## Interpreting Results Users Share

### React Profiler Results

If user shares "Why did this render?":
- **"Props changed"** → Look for inline functions/objects in parent
- **"State changed"** → Check if state is colocated properly
- **"Context changed"** → Context may be too broad
- **"Parent rendered"** → Missing memo() on child

If user shares render times:
- **> 16ms** → Component too expensive for 60 FPS
- **Many yellow components** → Cascade of unnecessary re-renders

### Perf Monitor Results

| User Reports | Interpretation | Action |
|--------------|----------------|--------|
| JS drops, UI stable | JS thread overloaded | Look for heavy computation in render, missing memoization |
| UI drops, JS stable | Native layer issue | Check animations using native driver, complex shadows |
| Both drop | Combined issue | Often heavy JS triggering expensive native updates |

### Memory Snapshots

If user reports heap growth:
- Ask for retained objects list
- Look for: event listeners, timers, closures capturing large data
- Check cleanup in useEffect returns

## Code Analysis (What Claude CAN Do)

When user shares code, look for these patterns:

### Re-render Red Flags

```tsx
// 🚩 Inline function in JSX
<Button onPress={() => handlePress(id)} />

// 🚩 Inline object in JSX
<View style={{ padding: 10 }} />

// 🚩 New array/object every render
const config = { threshold: 10 };
return <Child config={config} />;

// 🚩 Missing memo on list items
const Item = ({ data }) => <View>...</View>;
// Used in: items.map(item => <Item key={item.id} data={item} />)
```

### Memory Leak Red Flags

```tsx
// 🚩 Missing cleanup
useEffect(() => {
  const subscription = eventEmitter.addListener('event', handler);
  // No return statement!
}, []);

// 🚩 Timer without cleanup
useEffect(() => {
  setInterval(doSomething, 1000);
}, []);

// 🚩 Closure capturing component reference
useEffect(() => {
  someService.onUpdate = (data) => {
    setData(data); // This closure may outlive component
  };
}, []);
```

### JS Thread Blocking Red Flags

```tsx
// 🚩 Heavy computation in render
const Component = ({ items }) => {
  const sorted = items.sort((a, b) => ...); // Every render!
  return <List data={sorted} />;
};

// 🚩 Large data transformation without memoization
const processed = items.filter(...).map(...).reduce(...);

// 🚩 Synchronous work that should be deferred
useEffect(() => {
  const result = heavyComputation(); // Blocks first render
  setData(result);
}, []);
```

## Suggesting Fixes

After identifying the issue, reference the appropriate file:

| Issue Found | Direct To |
|-------------|-----------|
| Unnecessary re-renders | rerender-optimization.md, memoization.md |
| Heavy computation in render | concurrent-react.md (useDeferredValue) |
| Memory leak pattern | memory-management.md |
| List performance | list-optimization.md |
| Animation jank | animation-performance.md |
| Slow startup | startup-optimization.md |

## Key Reminders

- **Always ask for symptoms first** before analyzing code
- **Request profiler output** when code looks fine but user reports issues
- **Dev mode is slower** - remind users to test in release mode for accurate metrics
- **Real device > Simulator** for performance testing
- **Measure before and after** - ask user to verify fix worked
