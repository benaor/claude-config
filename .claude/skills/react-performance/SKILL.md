---
name: react-performance
description: React Native performance optimization guide. Use when working on performance issues, optimizing renders, improving startup time, debugging memory leaks, optimizing lists/animations, or analyzing bundle size. Covers profiling, memoization, React Compiler, Concurrent React, state patterns, FlatList optimization, Reanimated, native threading, and bundle optimization.
---

# React Native Performance

## How to Use This Skill

1. **Read ONLY this file first** - Don't load references yet
2. **Use Quick Diagnosis table** to identify the ONE relevant reference
3. **Load ONLY that reference** - Never load multiple references at once
4. **If user's problem is unclear** - Ask clarifying questions before loading any reference

> ⚠️ Each reference file is 5-10KB. Loading multiple files wastes context. Be surgical.

Comprehensive guide for optimizing React Native applications. Always **measure before optimizing** - use profiling tools to identify actual bottlenecks.

## Quick Diagnosis

| Symptom                    | Start With                                                      |
| -------------------------- | --------------------------------------------------------------- |
| Too many re-renders        | [rerender-optimization.md](references/rerender-optimization.md) |
| Slow lists/scrolling       | [list-optimization.md](references/list-optimization.md)         |
| App takes long to start    | [startup-optimization.md](references/startup-optimization.md)   |
| Memory keeps growing       | [memory-management.md](references/memory-management.md)         |
| Animations dropping frames | [animation-performance.md](references/animation-performance.md) |
| Bundle too large           | [bundle-optimization.md](references/bundle-optimization.md)     |

### Don't know where the problem comes from?

**Ask the user first:**

- What feels slow? (startup, scroll, animation, interaction)
- When does it happen?
- Which screen/component?

Then load [profiling.md](references/profiling.md) ONLY if you need help guiding them through profiling.

## Reference Files

> **Loading strategy**: Pick ONE file based on the diagnosed symptom. Never preload "just in case".

### 🔬 Investigation

- **[profiling.md](references/profiling.md)** - React DevTools, JS Profiler, Flashlight, Perf Monitor. Use to identify unknown bottlenecks.

### 🧠 Memory

- **[memory-management.md](references/memory-management.md)** - Heap snapshots, allocation tracking, common leak patterns.

### ⚛️ React Optimization

- **[rerender-optimization.md](references/rerender-optimization.md)** - Why components re-render, anti-patterns, prevention strategies.
- **[memoization.md](references/memoization.md)** - `useMemo`, `useCallback`, `React.memo` usage and pitfalls.
- **[react-compiler.md](references/react-compiler.md)** - Automatic memoization, Rules of React, when compiler can't optimize.
- **[concurrent-react.md](references/concurrent-react.md)** - `startTransition`, `useDeferredValue`, `<Activity>`, `Suspense` for responsive UI.
- **[state-patterns.md](references/state-patterns.md)** - State colocation, Zustand selectors, uncontrolled components.

### 📱 Runtime Performance

- **[list-optimization.md](references/list-optimization.md)** - `FlatList`, `SectionList`, virtualization, `getItemLayout`.
- **[animation-performance.md](references/animation-performance.md)** - Reanimated worklets, UI thread, native driver, gesture handling.

### 🚀 Startup & Bundle

- **[startup-optimization.md](references/startup-optimization.md)** - TTI optimization, lazy loading, Hermes, deferred initialization.
- **[bundle-optimization.md](references/bundle-optimization.md)** - Bundle analysis, tree shaking, barrel exports, lightweight alternatives.

### 🔧 Native Layer

- **[threading-model.md](references/threading-model.md)** - JS thread, UI thread, Fabric, JSI architecture.
- **[native-optimization.md](references/native-optimization.md)** - View flattening, native SDKs, shadow optimization, layout batching.

## Performance Targets

| Metric   | Target                              | Tool                     |
| -------- | ----------------------------------- | ------------------------ |
| FPS      | 60 (ideally 120 on capable devices) | Perf Monitor, Flashlight |
| TTI      | < 2s cold start                     | Native profilers         |
| JS Frame | < 16ms (8ms for 120fps)             | JS Profiler              |
| Memory   | Stable over time (no growth)        | Heap snapshots           |

## Optimization Workflow

1. **Measure** - Profile the app, identify the actual bottleneck
2. **Isolate** - Reproduce the issue in a minimal scenario
3. **Fix** - Apply targeted optimization from relevant reference
4. **Verify** - Re-measure to confirm improvement
5. **Monitor** - Set up continuous performance tracking

## Key Principles

- **Measure first**: Never optimize based on assumptions
- **80/20 rule**: 80% of issues come from 20% of code
- **JS thread < 16ms**: Keep frame budget for 60 FPS
- **Minimize bridge crossing**: Batch operations, use native drivers
- **Virtualize large lists**: Never render off-screen content
- **Defer non-critical work**: Use `startTransition` and lazy loading
