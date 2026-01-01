# Concurrent React

Concurrent React (React 18+) lets you mark updates as non-urgent, keeping the UI responsive during expensive operations.

## Core Concept

**Problem**: Heavy computation blocks the UI thread, causing jank.

```tsx
// ❌ Typing lags because filtering blocks the thread
const Search = () => {
  const [query, setQuery] = useState('');
  const results = expensiveFilter(items, query); // Blocks every keystroke
  
  return (
    <>
      <TextInput value={query} onChangeText={setQuery} />
      <List data={results} />
    </>
  );
};
```

**Solution**: Defer the expensive work so input stays responsive.

## useDeferredValue

Defers updating a value, showing stale content while computing.

```tsx
const Search = () => {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query); // Lags behind query
  
  // Filter uses deferred value → doesn't block input
  const results = expensiveFilter(items, deferredQuery);
  
  // Show visual feedback during calculation
  const isStale = query !== deferredQuery;
  
  return (
    <>
      <TextInput value={query} onChangeText={setQuery} />
      <View style={{ opacity: isStale ? 0.7 : 1 }}>
        <List data={results} />
      </View>
    </>
  );
};
```

**How it works:**
1. User types → `query` updates immediately
2. React renders input with new `query` (responsive)
3. In background, React prepares render with new `deferredQuery`
4. When ready, `deferredQuery` catches up and list updates

### When to Use

✅ Use `useDeferredValue` for:
- Filtering/searching large lists
- Expensive derived calculations
- Any computed value that can show stale data briefly

## useTransition

Marks a state update as non-urgent, providing `isPending` state.

```tsx
const TabContainer = () => {
  const [tab, setTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  const handleTabChange = (newTab: string) => {
    startTransition(() => {
      setTab(newTab); // Non-urgent: can be interrupted
    });
  };
  
  return (
    <>
      <TabBar 
        selected={tab} 
        onSelect={handleTabChange}
        disabled={isPending} // Prevent spam clicks
      />
      {isPending && <ActivityIndicator />}
      <TabContent tab={tab} />
    </>
  );
};
```

**Key difference from `useDeferredValue`:**
- `useDeferredValue` defers a **value** (read from props/state)
- `useTransition` defers a **state update** (write operation)

### When to Use

✅ Use `useTransition` for:
- Navigation between heavy screens
- Tab switches with expensive content
- Any state update that can be interrupted

## startTransition (Without Hook)

Standalone function for marking updates as non-urgent.

```tsx
import { startTransition } from 'react';

const handleSearch = (text: string) => {
  // Urgent: update input immediately
  setInputValue(text);
  
  // Non-urgent: filter can wait
  startTransition(() => {
    setFilteredResults(expensiveFilter(items, text));
  });
};
```

Use when you don't need `isPending` state.

## Activity Component (React 19+)

`<Activity>` hides content without unmounting, preserving state and DOM.

```tsx
import { Activity } from 'react';

const TabContainer = () => {
  const [activeTab, setActiveTab] = useState('home');
  
  return (
    <>
      <TabBar selected={activeTab} onSelect={setActiveTab} />
      
      {/* All tabs stay mounted, only visibility changes */}
      <Activity mode={activeTab === 'home' ? 'visible' : 'hidden'}>
        <HomeTab />
      </Activity>
      
      <Activity mode={activeTab === 'search' ? 'visible' : 'hidden'}>
        <SearchTab /> {/* Keeps search state when hidden */}
      </Activity>
      
      <Activity mode={activeTab === 'profile' ? 'visible' : 'hidden'}>
        <ProfileTab />
      </Activity>
    </>
  );
};
```

### Benefits Over Conditional Rendering

```tsx
// ❌ Without Activity: state lost on tab switch
{activeTab === 'search' && <SearchTab />}

// ✅ With Activity: state preserved
<Activity mode={activeTab === 'search' ? 'visible' : 'hidden'}>
  <SearchTab />
</Activity>
```

### Use Cases

| Scenario | Benefit |
|----------|---------|
| Tab navigation | Preserve scroll position, form inputs |
| Modal/drawer | Keep background content mounted |
| Screen stack | Pre-render next screen for instant transition |
| Virtualized lists | Keep offscreen items ready |

### With Transitions

Combine with `useTransition` for smooth tab switches:

```tsx
const TabContainer = () => {
  const [activeTab, setActiveTab] = useState('home');
  const [isPending, startTransition] = useTransition();
  
  const switchTab = (tab: string) => {
    startTransition(() => setActiveTab(tab));
  };
  
  return (
    <View style={{ opacity: isPending ? 0.8 : 1 }}>
      <TabBar selected={activeTab} onSelect={switchTab} />
      
      <Activity mode={activeTab === 'home' ? 'visible' : 'hidden'}>
        <HomeTab />
      </Activity>
      
      <Activity mode={activeTab === 'search' ? 'visible' : 'hidden'}>
        <SearchTab />
      </Activity>
    </View>
  );
};
```

> **Note**: `<Activity>` is available in React 19+. For earlier versions, use conditional rendering with state lifting or external state management to preserve state.

## Suspense Integration

Concurrent features work with Suspense for data fetching.

```tsx
const Screen = () => {
  const [resource, setResource] = useState(initialResource);
  const [isPending, startTransition] = useTransition();
  
  const handleRefresh = () => {
    startTransition(() => {
      setResource(fetchNewResource()); // Triggers Suspense
    });
  };
  
  return (
    <>
      <Button onPress={handleRefresh} disabled={isPending}>
        Refresh
      </Button>
      {isPending && <ActivityIndicator />}
      <Suspense fallback={<Skeleton />}>
        <Content resource={resource} />
      </Suspense>
    </>
  );
};
```

With transition:
- Old content stays visible during fetch
- `isPending` shows loading indicator alongside old content
- New content appears when ready

Without transition:
- Suspense fallback replaces content immediately
- Jarring UX

## Practical Patterns

### Search with Debounce Alternative

```tsx
// Instead of debounce, use deferred value
const Search = () => {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  
  // Memoize to skip re-filter when query matches deferred
  const results = useMemo(
    () => expensiveFilter(items, deferredQuery),
    [deferredQuery]
  );
  
  return (
    <>
      <TextInput 
        value={query} 
        onChangeText={setQuery}
        placeholder="Search..."
      />
      <List data={results} />
    </>
  );
};
```

### Heavy Screen Navigation

```tsx
const Navigator = () => {
  const [screen, setScreen] = useState('list');
  const [isPending, startTransition] = useTransition();
  
  const navigate = (to: string) => {
    startTransition(() => setScreen(to));
  };
  
  return (
    <View style={{ opacity: isPending ? 0.8 : 1 }}>
      {screen === 'list' && <HeavyListScreen onItemPress={() => navigate('detail')} />}
      {screen === 'detail' && <HeavyDetailScreen onBack={() => navigate('list')} />}
    </View>
  );
};
```

### Form with Expensive Validation

```tsx
const Form = () => {
  const [formData, setFormData] = useState(initial);
  const deferredFormData = useDeferredValue(formData);
  
  // Expensive validation doesn't block input
  const errors = useMemo(
    () => validateForm(deferredFormData),
    [deferredFormData]
  );
  
  return (
    <>
      <Input 
        value={formData.email} 
        onChangeText={(email) => setFormData(d => ({ ...d, email }))}
      />
      {errors.email && <ErrorText>{errors.email}</ErrorText>}
    </>
  );
};
```

## React Native Considerations

### InteractionManager Alternative

Before Concurrent React, you'd use `InteractionManager`:

```tsx
// Old approach (legacy)
useFocusEffect(
  useCallback(() => {
    const task = InteractionManager.runAfterInteractions(() => {
      loadExpensiveData();
    });
    return () => task.cancel();
  }, [])
);

// New approach with transitions
const [data, setData] = useState(null);
const [isPending, startTransition] = useTransition();

useFocusEffect(
  useCallback(() => {
    startTransition(() => {
      setData(loadExpensiveData());
    });
  }, [])
);
```

### With React Navigation

```tsx
useFocusEffect(
  useCallback(() => {
    startTransition(() => {
      // Heavy computation after animation completes
      setProcessedData(processData(rawData));
    });
  }, [rawData])
);
```

## Decision Guide

```
What's the problem?
│
├── Expensive calculation blocking input?
│   ├── Value from props/state → useDeferredValue
│   └── Need to trigger state update → startTransition / useTransition
│
├── Tab/screen switch feels janky?
│   ├── Need loading indicator → useTransition
│   ├── Need to preserve hidden tab state → <Activity>
│   └── Both → useTransition + <Activity>
│
├── Need to preserve state when hiding content?
│   └── <Activity mode="hidden"> (React 19+)
│
├── Replacing debounce for search?
│   └── useDeferredValue (simpler, automatic)
│
└── Data fetching with Suspense?
    └── Wrap state update in startTransition to keep old content visible
```

## Limitations

- Transitions can be interrupted (intentional for responsiveness)
- Not suitable for urgent updates (form submissions, mutations)
- Requires React 18+ (React Native 0.69+)
- `<Activity>` requires React 19+
