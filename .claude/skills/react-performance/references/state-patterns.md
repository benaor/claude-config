# State Patterns for Performance

How you structure and manage state directly impacts re-render frequency and app responsiveness.

## State Colocation

**Principle**: Keep state as close as possible to where it's used.

```tsx
// ❌ Global state causes unnecessary re-renders
const App = () => {
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [isModalOpen, setIsModalOpen] = useState(false);
  
  return (
    <>
      <SearchBar query={searchQuery} onQueryChange={setSearchQuery} />
      <List selectedId={selectedId} onSelect={setSelectedId} />
      <Modal open={isModalOpen} onClose={() => setIsModalOpen(false)} />
    </>
  );
  // Every state change re-renders ALL children
};

// ✅ Colocated state minimizes re-render scope
const App = () => (
  <>
    <SearchSection />
    <ListSection />
    <ModalSection />
  </>
);

const SearchSection = () => {
  const [query, setQuery] = useState('');
  return <SearchBar query={query} onQueryChange={setQuery} />;
  // Only SearchSection re-renders on query change
};
```

## Uncontrolled Components

For inputs that don't need real-time sync with React state, let the native component manage its own state.

### Controlled vs Uncontrolled

```tsx
// Controlled: React manages value (syncs on every keystroke)
const ControlledInput = () => {
  const [value, setValue] = useState('');
  return <TextInput value={value} onChangeText={setValue} />;
  // Re-renders on every keystroke
};

// Uncontrolled: Native manages value (no sync overhead)
// Access value only when needed
```

### Web Example

```tsx
const UncontrolledForm = () => {
  const inputRef = useRef<HTMLInputElement>(null);
  
  const handleSubmit = () => {
    const value = inputRef.current?.value; // Access only on submit
    submitForm(value);
  };
  
  return (
    <>
      <input ref={inputRef} defaultValue="" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
};
```

### React Native Example

React Native TextInput doesn't expose `.value` on ref. Use a ref to store the value without triggering re-renders:

```tsx
const UncontrolledForm = () => {
  const valueRef = useRef('');
  
  const handleSubmit = () => {
    submitForm(valueRef.current); // Access only on submit
  };
  
  return (
    <>
      <TextInput
        defaultValue=""
        onChangeText={(text) => {
          valueRef.current = text; // Store without re-render
        }}
      />
      <Button onPress={handleSubmit} title="Submit" />
    </>
  );
};
```

### When to Use Uncontrolled

✅ Use uncontrolled for:
- Forms that only need values on submit
- Search inputs with debounced API calls
- Performance-critical input-heavy screens

❌ Keep controlled for:
- Real-time validation/formatting
- Dependent field updates
- Character limits/masks

### Hybrid Approach

```tsx
const SearchInput = ({ onSearch }: Props) => {
  const [inputValue, setInputValue] = useState('');
  const deferredValue = useDeferredValue(inputValue);
  
  // Trigger search on deferred value
  useEffect(() => {
    onSearch(deferredValue);
  }, [deferredValue, onSearch]);
  
  return (
    <TextInput 
      value={inputValue} 
      onChangeText={setInputValue}
    />
  );
};
```

## Atomic State Updates

Split large state objects into independent atoms to minimize re-render scope.

```tsx
// ❌ Monolithic state: any change re-renders everything
const Form = () => {
  const [form, setForm] = useState({
    name: '',
    email: '',
    phone: '',
    address: '',
    preferences: { theme: 'light', notifications: true }
  });
  
  const updateName = (name: string) => {
    setForm(f => ({ ...f, name })); // Entire form re-renders
  };
};

// ✅ Atomic state: independent updates
const Form = () => {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [phone, setPhone] = useState('');
  const [address, setAddress] = useState('');
  const [preferences, setPreferences] = useState({ theme: 'light', notifications: true });
  
  // Or use useReducer for complex interdependent state
};
```

### With Zustand

Zustand with selectors enables atomic updates:

```tsx
import { create } from 'zustand';

// Store with multiple fields
const useFormStore = create<FormStore>((set) => ({
  name: '',
  email: '',
  phone: '',
  setName: (name) => set({ name }),
  setEmail: (email) => set({ email }),
  setPhone: (phone) => set({ phone }),
}));

// ❌ Subscribes to entire store → re-renders on any field change
const BadInput = () => {
  const store = useFormStore(); // Full store subscription
  return <TextInput value={store.name} onChangeText={store.setName} />;
};

// ✅ Subscribes only to name → re-renders only when name changes
const NameInput = () => {
  const name = useFormStore((s) => s.name);
  const setName = useFormStore((s) => s.setName);
  return <TextInput value={name} onChangeText={setName} />;
};

const EmailInput = () => {
  const email = useFormStore((s) => s.email);
  const setEmail = useFormStore((s) => s.setEmail);
  return <TextInput value={email} onChangeText={setEmail} />;
};
```

**Key pattern**: Always use selectors to subscribe to specific slices of state.

## Context Splitting

Split contexts by update frequency to prevent unrelated re-renders.

```tsx
// ❌ Single context: all consumers re-render on any change
const AppContext = createContext({
  user: null,
  theme: 'light',
  notifications: [],
});

// ✅ Split contexts by update frequency
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<'light' | 'dark'>('light');
const NotificationsContext = createContext<Notification[]>([]);

// Components subscribe only to what they need
const Avatar = () => {
  const user = useContext(UserContext);
  return <Image source={{ uri: user?.avatarUrl }} />;
  // Doesn't re-render on theme/notification changes
};
```

### Context Value Memoization

```tsx
// ❌ New object every render → all consumers re-render
const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {children}
    </AuthContext.Provider>
  );
};

// ✅ Memoized value
const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  
  const value = useMemo(() => ({ user, setUser }), [user]);
  
  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};
```

## Derived State

Compute derived values during render instead of syncing with state.

```tsx
// ❌ Synced state: risk of inconsistency, extra re-renders
const List = ({ items }: Props) => {
  const [sortedItems, setSortedItems] = useState<Item[]>([]);
  
  useEffect(() => {
    setSortedItems(items.slice().sort(compareFn));
  }, [items]); // Extra render after items change
  
  return <FlatList data={sortedItems} />;
};

// ✅ Derived during render: always consistent
const List = ({ items }: Props) => {
  const sortedItems = useMemo(
    () => items.slice().sort(compareFn),
    [items]
  );
  
  return <FlatList data={sortedItems} />;
};
```

## Lifting State Wisely

Sometimes lifting state up is necessary, but lift minimally.

```tsx
// ❌ Lifted too high: App re-renders on selection change
const App = () => {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  
  return (
    <View>
      <Header />
      <List selectedId={selectedId} onSelect={setSelectedId} />
      <Detail itemId={selectedId} />
    </View>
  );
};

// ✅ Lifted to nearest common ancestor only
const App = () => (
  <View>
    <Header />
    <ListWithDetail />
  </View>
);

const ListWithDetail = () => {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  
  return (
    <>
      <List selectedId={selectedId} onSelect={setSelectedId} />
      <Detail itemId={selectedId} />
    </>
  );
};
```

## Performance Checklist

```
□ State lives as close to usage as possible
□ Form inputs are uncontrolled where possible
□ Large state objects are split into atoms
□ Context is split by update frequency
□ Context values are memoized
□ Derived data computed with useMemo, not synced state
□ State lifted only to nearest common ancestor
```
