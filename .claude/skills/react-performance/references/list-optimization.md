# List Optimization

Lists are often the biggest performance bottleneck in React Native apps. Proper virtualization and optimization are critical.

## Virtualization Basics

**Problem**: Rendering thousands of items at once blocks the JS thread and consumes memory.

**Solution**: Only render items visible on screen + small buffer.

```tsx
// ❌ Never do this: renders ALL items
const BadList = ({ items }: Props) => (
  <ScrollView>
    {items.map(item => <Item key={item.id} {...item} />)}
  </ScrollView>
);

// ✅ Virtualized: only renders visible items
const GoodList = ({ items }: Props) => (
  <FlatList
    data={items}
    renderItem={({ item }) => <Item {...item} />}
    keyExtractor={item => item.id}
  />
);
```

## FlatList Essential Props

### keyExtractor (Required)

Stable unique key for each item. Critical for proper recycling.

```tsx
// ❌ Index as key: breaks on reorder/filter
keyExtractor={(item, index) => index.toString()}

// ✅ Stable ID
keyExtractor={(item) => item.id}
```

### getItemLayout (Major Optimization)

If all items have the same height, provide layout upfront to skip measurement.

```tsx
const ITEM_HEIGHT = 72;

<FlatList
  data={items}
  renderItem={renderItem}
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
/>
```

**Benefits:**
- Enables instant `scrollToIndex`
- Faster initial render
- Smoother scrolling

### windowSize

Controls render window as multiplier of viewport. Default is 21 (10 screens above + 10 below).

```tsx
// Reduce for memory savings (slight flash risk)
<FlatList windowSize={5} />

// Increase for smoother scroll (more memory)
<FlatList windowSize={31} />
```

### initialNumToRender

Items to render before scroll. Default is 10.

```tsx
// Match roughly one screen of items
<FlatList initialNumToRender={8} />
```

### maxToRenderPerBatch

Items to render per batch during scroll. Default is 10.

```tsx
// Lower for complex items, higher for simple
<FlatList maxToRenderPerBatch={5} />
```

### updateCellsBatchingPeriod

Milliseconds between batch renders. Default is 50.

```tsx
// Increase for smoother scroll on slow devices
<FlatList updateCellsBatchingPeriod={100} />
```

### removeClippedSubviews

Unmounts off-screen views. Use cautiously.

```tsx
// Can help on long lists, but may cause issues
<FlatList removeClippedSubviews={true} />
```

> ⚠️ May cause blank spaces on fast scroll. Test thoroughly.

## Optimized Item Components

### Memoize Item Components

```tsx
// Item receives id and stable callback
const Item = memo(({ id, title, onPress }: ItemProps) => {
  console.log('Item render:', title);
  
  const handlePress = useCallback(() => {
    onPress(id); // Call parent handler with id
  }, [id, onPress]);
  
  return (
    <Pressable onPress={handlePress} style={styles.item}>
      <Text>{title}</Text>
    </Pressable>
  );
});

// Parent provides stable callback, no inline functions
const List = ({ items }: Props) => {
  const handlePress = useCallback((id: string) => {
    navigation.navigate('Detail', { id });
  }, [navigation]);
  
  const renderItem = useCallback(
    ({ item }: { item: Item }) => (
      <Item 
        id={item.id}
        title={item.title} 
        onPress={handlePress} // Same reference for all items
      />
    ),
    [handlePress]
  );
  
  return <FlatList data={items} renderItem={renderItem} />;
};
```

**Key insight**: Pass `id` as prop, let the memoized Item create the final callback. This way `onPress` reference is stable across all items.

### Flatten Item Structure

```tsx
// ❌ Deep nesting = more work
const Item = () => (
  <View>
    <View>
      <View>
        <View>
          <Text>Content</Text>
        </View>
      </View>
    </View>
  </View>
);

// ✅ Flat structure
const Item = () => (
  <View style={styles.item}>
    <Text>Content</Text>
  </View>
);
```

### Avoid Inline Styles

```tsx
// ❌ New object every render
<View style={{ padding: 10 }}>

// ✅ StyleSheet reference
const styles = StyleSheet.create({
  item: { padding: 10 }
});
<View style={styles.item}>
```

## Handling Expensive Items

### Defer Heavy Content

```tsx
const Item = memo(({ item }: Props) => {
  const [ready, setReady] = useState(false);
  
  useEffect(() => {
    // Defer heavy content after mount
    const id = requestAnimationFrame(() => setReady(true));
    return () => cancelAnimationFrame(id);
  }, []);
  
  return (
    <View style={styles.item}>
      <Text>{item.title}</Text>
      {ready ? <HeavyChart data={item.data} /> : <Placeholder />}
    </View>
  );
});
```

### Use Placeholders

```tsx
const Item = memo(({ item }: Props) => {
  const [imageLoaded, setImageLoaded] = useState(false);
  
  return (
    <View style={styles.item}>
      {!imageLoaded && <View style={styles.imagePlaceholder} />}
      <Image 
        source={{ uri: item.imageUrl }}
        style={[styles.image, !imageLoaded && styles.hidden]}
        onLoad={() => setImageLoaded(true)}
      />
      <Text>{item.title}</Text>
    </View>
  );
});
```

## SectionList Optimization

Same principles apply, plus section-specific props.

```tsx
<SectionList
  sections={sections}
  keyExtractor={(item) => item.id}
  renderItem={renderItem}
  renderSectionHeader={renderSectionHeader}
  stickySectionHeadersEnabled={false} // Disable if not needed
  getItemLayout={getItemLayout} // If fixed height
/>
```

## Common Issues & Fixes

### Blank Spaces on Fast Scroll

```tsx
// Increase render window
<FlatList windowSize={21} />

// Or disable removeClippedSubviews
<FlatList removeClippedSubviews={false} />
```

### Slow Initial Render

```tsx
// Reduce initial items
<FlatList 
  initialNumToRender={5}
  maxToRenderPerBatch={3}
/>

// Add getItemLayout if possible
```

### Jank During Scroll

```tsx
// Check: is renderItem expensive?
// Use React Profiler to identify

// Solutions:
// 1. Memoize item component
// 2. Simplify item structure
// 3. Defer heavy content
// 4. Use InteractionManager
```

### Memory Issues with Long Lists

```tsx
// Reduce window
<FlatList windowSize={5} />

// Enable cleanup
<FlatList removeClippedSubviews={true} />

// Paginate data
<FlatList 
  data={visibleItems}
  onEndReached={loadMore}
  onEndReachedThreshold={0.5}
/>
```

## Performance Checklist

```
□ Using FlatList/SectionList, not ScrollView with map
□ keyExtractor returns stable unique ID
□ getItemLayout provided (if fixed height)
□ Item component wrapped in memo()
□ renderItem is memoized with useCallback
□ No inline functions/styles in items
□ Item structure is flat (minimal nesting)
□ Heavy content deferred or lazy-loaded
□ windowSize tuned for use case
□ Tested with production data volume
```

## Quick Tuning Guide

| Symptom | Adjustment |
|---------|------------|
| Slow initial render | Lower `initialNumToRender` |
| Blank spaces on scroll | Increase `windowSize` |
| Memory issues | Lower `windowSize`, enable `removeClippedSubviews` |
| Scroll jank | Lower `maxToRenderPerBatch`, simplify items |
| Jumpy scrollToIndex | Add `getItemLayout` |
