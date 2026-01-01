# Memoization

Memoization caches computation results to avoid redundant work. In React, it prevents unnecessary re-renders and recalculations.

## Three Memoization Tools

| Tool | Purpose | Caches |
|------|---------|--------|
| `React.memo()` | Skip re-render if props unchanged | Component render result |
| `useMemo()` | Skip recalculation if deps unchanged | Any computed value |
| `useCallback()` | Stable function reference | Function reference |

## React.memo()

Wraps a component to skip re-render when props are shallowly equal.

```tsx
// Without memo: re-renders whenever parent renders
const ListItem = ({ title, onPress }: Props) => {
  console.log('ListItem render');
  return (
    <Pressable onPress={onPress}>
      <Text>{title}</Text>
    </Pressable>
  );
};

// With memo: only re-renders if title or onPress changes
const ListItem = memo(({ title, onPress }: Props) => {
  console.log('ListItem render');
  return (
    <Pressable onPress={onPress}>
      <Text>{title}</Text>
    </Pressable>
  );
});
```

### Custom Comparison

```tsx
// Default: shallow equality on all props
const Item = memo(Component);

// Custom: only compare specific props
const Item = memo(Component, (prevProps, nextProps) => {
  return prevProps.id === nextProps.id; // Skip render if id same
});
```

### When to Use

✅ Use `memo()` for:
- List items rendered many times
- Components with expensive render logic
- Pure components that don't depend on context
- Children of frequently updating parents

❌ Don't use for:
- Components that always receive new props
- Very simple components (memo overhead > render cost)
- Components that need to re-render often anyway

## useMemo()

Caches a computed value between renders.

```tsx
// ❌ Recalculates on every render
const Component = ({ items }: { items: Item[] }) => {
  const sorted = items.slice().sort((a, b) => a.name.localeCompare(b.name));
  return <List data={sorted} />;
};

// ✅ Only recalculates when items change
const Component = ({ items }: { items: Item[] }) => {
  const sorted = useMemo(
    () => items.slice().sort((a, b) => a.name.localeCompare(b.name)),
    [items]
  );
  return <List data={sorted} />;
};
```

### Stable Object References

```tsx
// ❌ New object every render → triggers child re-render
const Parent = ({ width, height }: Props) => {
  return <Child dimensions={{ width, height }} />;
};

// ✅ Stable reference when deps unchanged
const Parent = ({ width, height }: Props) => {
  const dimensions = useMemo(() => ({ width, height }), [width, height]);
  return <Child dimensions={dimensions} />;
};
```

### When to Use

✅ Use `useMemo()` for:
- Expensive calculations (sorting, filtering large arrays)
- Creating objects/arrays passed to memoized children
- Derived state that shouldn't trigger effects

❌ Don't use for:
- Simple calculations (addition, string concat)
- Values only used in render (no child dependency)
- Premature optimization without profiling

## useCallback()

Returns a memoized callback function.

```tsx
// ❌ New function every render → ListItem re-renders
const List = ({ items }: Props) => {
  const handlePress = (id: string) => {
    console.log('Pressed:', id);
  };
  
  return items.map(item => (
    <ListItem key={item.id} onPress={() => handlePress(item.id)} />
  ));
};

// ✅ Stable callback reference
const List = ({ items }: Props) => {
  const handlePress = useCallback((id: string) => {
    console.log('Pressed:', id);
  }, []);
  
  return items.map(item => (
    <ListItem key={item.id} id={item.id} onPress={handlePress} />
  ));
};

// ListItem receives id as prop and calls onPress(id)
const ListItem = memo(({ id, onPress }: ItemProps) => (
  <Pressable onPress={() => onPress(id)}>...</Pressable>
));
```

### With Dependencies

```tsx
const Component = ({ userId }: Props) => {
  const [items, setItems] = useState<Item[]>([]);
  
  // Recreated only when userId changes
  const fetchItems = useCallback(async () => {
    const data = await api.getItems(userId);
    setItems(data);
  }, [userId]);
  
  useEffect(() => {
    fetchItems();
  }, [fetchItems]);
  
  return <List data={items} />;
};
```

## Common Mistakes

### 1. Missing Dependencies

```tsx
// ❌ Bug: stale count value
const Counter = () => {
  const [count, setCount] = useState(0);
  
  const increment = useCallback(() => {
    setCount(count + 1); // Always uses initial count!
  }, []); // Missing count dependency
  
  return <Button onPress={increment} />;
};

// ✅ Fixed: functional update
const Counter = () => {
  const [count, setCount] = useState(0);
  
  const increment = useCallback(() => {
    setCount(c => c + 1); // Uses latest count
  }, []);
  
  return <Button onPress={increment} />;
};
```

### 2. Memoizing Without memo()

```tsx
// ❌ useCallback is useless if child isn't memoized
const Parent = () => {
  const handlePress = useCallback(() => {}, []);
  return <Child onPress={handlePress} />; // Child still re-renders!
};

const Child = ({ onPress }) => <Button onPress={onPress} />;

// ✅ Both are needed
const Child = memo(({ onPress }) => <Button onPress={onPress} />);
```

### 3. Over-Memoization

```tsx
// ❌ Unnecessary: simple calculation
const total = useMemo(() => price * quantity, [price, quantity]);

// ✅ Just calculate directly
const total = price * quantity;
```

## Memoization Decision Tree

```
Is the component re-rendering unnecessarily?
├── No → Don't memoize
└── Yes → Profile first, then:
    │
    ├── Component is pure, props unchanged?
    │   └── Wrap with memo()
    │
    ├── Passing callback to memoized child?
    │   └── useCallback() for the function
    │
    ├── Passing object/array to memoized child?
    │   └── useMemo() for the value
    │
    └── Expensive calculation in render?
        └── useMemo() for the result
```

## Performance Comparison

```tsx
// Baseline: No memoization
const List = ({ items, onItemPress }) => (
  items.map(item => <Item key={item.id} {...item} onPress={onItemPress} />)
);

// Optimized: Full memoization chain
const List = memo(({ items, onItemPress }) => (
  items.map(item => <Item key={item.id} {...item} onPress={onItemPress} />)
));

const Item = memo(({ id, title, onPress }) => (
  <Pressable onPress={() => onPress(id)}>
    <Text>{title}</Text>
  </Pressable>
));

// Parent must also memoize the callback
const Parent = () => {
  const handleItemPress = useCallback((id: string) => {
    // handle press
  }, []);
  
  return <List items={items} onItemPress={handleItemPress} />;
};
```

See [react-compiler.md](react-compiler.md) for automatic memoization.
