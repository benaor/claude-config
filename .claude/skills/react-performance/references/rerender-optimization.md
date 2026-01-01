# Re-render Optimization

React re-renders components to keep UI in sync with state. Unnecessary re-renders waste CPU cycles and can cause jank.

## When Components Re-render

A component re-renders when:

1. **Parent re-renders** (default behavior)
2. **State changes** (`useState`, `useReducer`)
3. **Props change** (shallow comparison)
4. **Context changes** (any subscribed context)
5. **Force update** (escape hatch, avoid)

```tsx
const Parent = () => {
  const [count, setCount] = useState(0);
  
  return (
    <View>
      <Text>{count}</Text>
      <Child /> {/* Re-renders on every count change! */}
      <Button onPress={() => setCount(c => c + 1)} />
    </View>
  );
};

const Child = () => {
  console.log('Child rendered'); // Logs on every parent update
  return <Text>I never change</Text>;
};
```

## Identifying Unnecessary Re-renders

Use React DevTools Profiler with "Record why each component rendered" enabled.

Common "why did this render" reasons:
- "The parent component rendered" → Consider `memo()`
- "Props changed: onPress" → Memoize callback with `useCallback`
- "Props changed: style" → Memoize object with `useMemo`
- "Context changed" → Split context or use selectors

## Prevention Patterns

### 1. Memoize Static Children

```tsx
// ❌ Child re-renders on every parent update
const Parent = () => {
  const [count, setCount] = useState(0);
  return (
    <View>
      <ExpensiveChild />
      <Text>{count}</Text>
    </View>
  );
};

// ✅ Child skips re-render if props unchanged
const ExpensiveChild = memo(() => {
  return <ComplexView />;
});
```

### 2. Stable Callback References

```tsx
// ❌ New function on every render → child re-renders
const Parent = () => {
  const [items, setItems] = useState<string[]>([]);
  
  return (
    <List 
      items={items}
      onItemPress={(id) => console.log(id)} // New ref every render!
    />
  );
};

// ✅ Stable reference with useCallback
const Parent = () => {
  const [items, setItems] = useState<string[]>([]);
  
  const handleItemPress = useCallback((id: string) => {
    console.log(id);
  }, []); // Empty deps = stable forever
  
  return <List items={items} onItemPress={handleItemPress} />;
};
```

### 3. Stable Object/Array References

```tsx
// ❌ New object on every render
const Parent = () => {
  return <Child style={{ padding: 10 }} />; // New object!
};

// ✅ Option 1: StyleSheet (best for static styles)
const styles = StyleSheet.create({
  container: { padding: 10 }
});
const Parent = () => <Child style={styles.container} />;

// ✅ Option 2: useMemo (for dynamic styles)
const Parent = ({ size }: { size: number }) => {
  const style = useMemo(() => ({ padding: size }), [size]);
  return <Child style={style} />;
};
```

### 4. State Colocation

Move state closer to where it's used to reduce re-render scope.

```tsx
// ❌ Entire form re-renders on any input change
const Form = () => {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  
  return (
    <View>
      <Input value={name} onChangeText={setName} />
      <Input value={email} onChangeText={setEmail} />
      <Input value={phone} onChangeText={setPhone} />
      <ExpensiveValidation /> {/* Re-renders on every keystroke! */}
    </View>
  );
};

// ✅ Each input manages its own state
const NameInput = () => {
  const [name, setName] = useState('');
  return <Input value={name} onChangeText={setName} />;
};

const Form = () => (
  <View>
    <NameInput />
    <EmailInput />
    <PhoneInput />
    <ExpensiveValidation /> {/* Only re-renders when needed */}
  </View>
);
```

### 5. Context Optimization

```tsx
// ❌ All consumers re-render on any context change
const AppContext = createContext({ user: null, theme: 'light', settings: {} });

// ✅ Split into focused contexts
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<'light' | 'dark'>('light');
const SettingsContext = createContext<Settings>({});

// Components only subscribe to what they need
const Avatar = () => {
  const user = useContext(UserContext); // Only re-renders on user change
  return <Image source={user?.avatar} />;
};
```

### 6. Children as Props Pattern

```tsx
// ❌ Children re-render with parent
const Layout = () => {
  const [sidebarOpen, setSidebarOpen] = useState(false);
  
  return (
    <View>
      <Sidebar open={sidebarOpen} />
      <ExpensiveContent /> {/* Re-renders on sidebar toggle! */}
    </View>
  );
};

// ✅ Children passed as props don't re-render
const Layout = ({ children }: { children: ReactNode }) => {
  const [sidebarOpen, setSidebarOpen] = useState(false);
  
  return (
    <View>
      <Sidebar open={sidebarOpen} />
      {children} {/* Stable reference from parent */}
    </View>
  );
};

// Usage
<Layout>
  <ExpensiveContent /> {/* Doesn't re-render on sidebar toggle */}
</Layout>
```

## Anti-Patterns

### Data Transformation in Render

```tsx
// ❌ Recalculates on every render
const UserList = ({ users }: { users: User[] }) => {
  const sorted = users
    .filter(u => u.active)
    .sort((a, b) => a.name.localeCompare(b.name));
  
  return <FlatList data={sorted} renderItem={renderUser} />;
};

// ✅ Memoize the transformation
const UserList = ({ users }: { users: User[] }) => {
  const sorted = useMemo(
    () => users.filter(u => u.active).sort((a, b) => a.name.localeCompare(b.name)),
    [users]
  );
  return <FlatList data={sorted} renderItem={renderUser} />;
};
```

### Business Logic in Render

```tsx
// ❌ Complex calculation blocks render
const PriceDisplay = ({ items }: { items: CartItem[] }) => {
  const total = items.reduce((sum, item) => {
    const discount = item.quantity > 10 ? 0.1 : 0;
    return sum + (item.price * item.quantity * (1 - discount));
  }, 0);
  
  return <Text>${total.toFixed(2)}</Text>;
};

// ✅ Extract to hook or useMemo
const useCartTotal = (items: CartItem[]) => {
  return useMemo(() => {
    return items.reduce((sum, item) => {
      const discount = item.quantity > 10 ? 0.1 : 0;
      return sum + (item.price * item.quantity * (1 - discount));
    }, 0);
  }, [items]);
};

const PriceDisplay = ({ items }: { items: CartItem[] }) => {
  const total = useCartTotal(items);
  return <Text>${total.toFixed(2)}</Text>;
};
```

### Spreading Props

```tsx
// ❌ Any extra prop change triggers re-render
const Parent = (props: ParentProps) => {
  return <Child {...props} />; // Opaque dependency
};

// ✅ Explicit props = clear re-render triggers
const Parent = ({ name, onPress }: ParentProps) => {
  return <Child name={name} onPress={onPress} />;
};
```

### Object/Array Literals in JSX

```tsx
// ❌ Always triggers re-render even with memo()
<List 
  data={items}
  extraData={{ userId }} // New object every render
  contentContainerStyle={{ padding: 10 }} // New object!
/>

// ✅ Stable references
const extraData = useMemo(() => ({ userId }), [userId]);
const contentStyle = useMemo(() => ({ padding: 10 }), []);
```

## Quick Checklist

```
□ Use React DevTools Profiler to find unnecessary re-renders
□ Wrap pure components with memo()
□ Memoize callbacks with useCallback
□ Memoize objects/arrays with useMemo
□ Move state close to where it's used
□ Split large contexts into focused ones
□ Avoid inline objects/functions in JSX
□ Consider React Compiler for automatic memoization
```

See [memoization.md](memoization.md) for detailed memoization patterns, [react-compiler.md](react-compiler.md) for automatic optimization.
