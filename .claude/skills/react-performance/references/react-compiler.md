# React Compiler

React Compiler automatically memoizes components and hooks at build time. When enabled, manual `useMemo`, `useCallback`, and `memo()` are often unnecessary.

## What the Compiler Does

```tsx
// What you write
const Component = ({ items }: Props) => {
  const sorted = items.slice().sort((a, b) => a.name.localeCompare(b.name));
  const handlePress = (id: string) => console.log(id);
  
  return <List data={sorted} onItemPress={handlePress} />;
};

// What the compiler produces (conceptually)
const Component = ({ items }: Props) => {
  const sorted = useMemo(
    () => items.slice().sort((a, b) => a.name.localeCompare(b.name)),
    [items]
  );
  const handlePress = useCallback((id: string) => console.log(id), []);
  
  return <List data={sorted} onItemPress={handlePress} />;
};
```

**Key insight**: If your codebase has React Compiler enabled, you don't need to manually add `useMemo`/`useCallback`/`memo()` unless the compiler can't optimize a specific case.

## Rules of React (Must Follow)

The compiler can only optimize code that follows these rules. Violations break optimization.

### 1. Components Must Be Pure

```tsx
// ❌ Side effect during render → compiler can't optimize
const Bad = () => {
  document.title = 'Hello'; // Side effect!
  return <Text>Hello</Text>;
};

// ✅ Side effect in useEffect → compiler optimizes
const Good = () => {
  useEffect(() => {
    document.title = 'Hello';
  }, []);
  return <Text>Hello</Text>;
};
```

### 2. Props and State Are Immutable

```tsx
// ❌ Mutating props → compiler can't optimize
const Bad = ({ user }: Props) => {
  user.name = 'Modified'; // Mutation!
  return <Text>{user.name}</Text>;
};

// ✅ Create new object → compiler optimizes
const Good = ({ user }: Props) => {
  const modified = { ...user, name: 'Modified' };
  return <Text>{modified.name}</Text>;
};
```

### 3. Hook Return Values Are Immutable

```tsx
// ❌ Mutating hook return → compiler can't optimize
const Component = () => {
  const items = useItems();
  items.push(newItem); // Mutation!
  return <List data={items} />;
};

// ✅ Copy before mutating → compiler optimizes
const Component = () => {
  const items = useItems();
  const newItems = [...items, newItem];
  return <List data={newItems} />;
};
```

### 4. No Conditional Hook Calls

```tsx
// ❌ Conditional hook → compiler can't optimize
const Bad = ({ condition }: Props) => {
  if (condition) {
    const [value] = useState(0); // Hook called conditionally!
  }
};

// ✅ Always call hooks at top level → compiler optimizes
const Good = ({ condition }: Props) => {
  const [value] = useState(0);
  // Use value conditionally instead
};
```

## When Compiler Can't Optimize

The compiler skips optimization for:

| Scenario | Why | Solution |
|----------|-----|----------|
| Class components | Not supported | Convert to function component |
| Rules violation | Can't safely memoize | Fix the violation |
| `'use no memo'` directive | Explicitly opted out | Remove directive if unintended |
| Dynamic patterns | Can't analyze statically | Manual memoization |
| Refs read during render | Breaks purity | Move ref read to useEffect |

### Detecting Unoptimized Components

If a component re-renders unexpectedly, check:

1. **ESLint warnings** - `eslint-plugin-react-compiler` reports violations
2. **React DevTools Profiler** - "Why did this render?" still works
3. **Component name in console** - Compiler logs skipped components in dev

## Working with the Compiler

### You DON'T Need Manual Memoization For:

```tsx
// Compiler handles these automatically ✅

// Inline callbacks
<Button onPress={() => handleAction(id)} />

// Computed values
const filtered = items.filter(x => x.active);

// Object literals
<Child style={{ padding: 10 }} />

// Array literals
<List data={[item1, item2]} />
```

### You STILL Need Manual Memoization For:

```tsx
// 1. Expensive computations the compiler can't detect
const result = useMemo(() => {
  return veryExpensiveOperation(data); // Compiler doesn't know this is expensive
}, [data]);

// 2. Values passed to non-React APIs
const stableRef = useMemo(() => createExternalLibraryInstance(), []);

// 3. Intentional referential equality for effects
const config = useMemo(() => ({ threshold: 10 }), []);
useEffect(() => {
  subscribe(config); // Effect should only run once
}, [config]);
```

## Opt-Out Directive

If a component breaks with compiler optimization:

```tsx
function ProblematicComponent() {
  'use no memo'; // Compiler skips this component
  
  // Legacy code that violates Rules of React
  // Fix later, but unblock for now
}
```

Use sparingly - prefer fixing the underlying issue.

## Coexistence with Manual Memoization

Existing `useMemo`/`useCallback`/`memo()` are preserved. The compiler doesn't remove them - it adds optimization where you didn't.

```tsx
// Both work together
const Component = memo(({ items }: Props) => {
  // Your manual memo() is kept
  // Compiler adds memoization for things you missed
  const processed = items.map(transform); // Compiler memoizes this
  return <List data={processed} />;
});
```

## Quick Reference

| Question | Answer |
|----------|--------|
| Should I remove existing useMemo/useCallback? | Optional. They still work, but code is cleaner without them |
| Component re-renders unexpectedly? | Check ESLint for Rules violations |
| Need to force re-render? | Use key prop or explicit state |
| Compiler not optimizing? | Check for mutations, side effects, conditional hooks |
