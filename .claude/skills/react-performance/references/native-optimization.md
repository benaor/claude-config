# Native Layer Optimization

Optimizations at the native layer can significantly improve performance, especially for UI rendering and native module interactions.

## View Flattening

React Native creates native views for each component. Fabric automatically flattens views that only affect layout.

**Views eligible for flattening:**
- Only have layout styles (flex, padding, margin)
- No background color, border, shadow
- No event handlers

For manual view structure optimization, see [list-optimization.md](list-optimization.md#flatten-item-structure).

### collapsable Prop

Force view preservation or allow removal:

```tsx
// Allow flattening (default behavior in Fabric)
<View collapsable={true} />

// Prevent flattening (when view identity matters)
<View collapsable={false} />
```

Use `collapsable={false}` for:
- Views with refs that need measurement
- Animation targets
- Native components requiring view identity

## Native SDKs Over Web

Use dedicated React Native SDKs instead of web libraries when available.

| Category | Web Approach ❌ | Native SDK ✅ |
|----------|-----------------|---------------|
| Maps | WebView + Google Maps JS | react-native-maps |
| Video | HTML5 video | react-native-video |
| Charts | D3.js + SVG | react-native-charts-wrapper |
| Payments | Stripe.js | @stripe/stripe-react-native |
| Auth | Firebase JS | @react-native-firebase/auth |
| Analytics | Web SDK | Native SDK bindings |

**Why native SDKs:**
- Run on UI thread (smoother)
- Native gesture handling
- Platform-specific optimizations
- Better memory management

### Example: Maps

```tsx
// ❌ WebView approach
<WebView source={{ uri: 'https://maps.google.com/...' }} />

// ✅ Native maps
import MapView from 'react-native-maps';
<MapView
  region={region}
  onRegionChange={setRegion}
/>
```

## Turbo Modules Performance

For Turbo Modules architecture and JSI explanation, see [threading-model.md](threading-model.md#turbo-modules).

### Sync vs Async

```tsx
// Sync (blocks JS until complete - use sparingly)
const result = TurboModule.syncMethod();

// Async (non-blocking - preferred for heavy work)
const result = await TurboModule.asyncMethod();
```

**Use sync for:** Quick operations (< 1ms), required before render, no I/O

**Use async for:** File operations, network requests, heavy computations

## Native Module Performance Tips

### Batch Calls

```tsx
// ❌ Many individual calls
items.forEach(item => {
  NativeModule.processItem(item);
});

// ✅ Single batch call
NativeModule.processItems(items);
```

### Avoid Frequent Bridge Crossings

```tsx
// ❌ Crossing bridge in scroll handler
const onScroll = (event) => {
  NativeModule.trackScroll(event.nativeEvent.contentOffset.y);
};

// ✅ Debounce or use native scroll tracking
const debouncedTrack = useMemo(
  () => debounce((y: number) => NativeModule.trackScroll(y), 100),
  []
);

const onScroll = (event) => {
  debouncedTrack(event.nativeEvent.contentOffset.y);
};
```

### Move Logic to Native

For performance-critical operations:

```tsx
// ❌ Process in JS, pass results to native
const processed = items.map(heavyTransform);
NativeModule.saveItems(processed);

// ✅ Pass raw data, process in native
NativeModule.processAndSaveItems(items);
```

## Platform-Specific Optimizations

For ProGuard/R8 configuration, see [bundle-optimization.md](bundle-optimization.md#android).

### Android: Fast Image Loading

```tsx
import FastImage from 'react-native-fast-image';

<FastImage
  source={{ uri: imageUrl, priority: FastImage.priority.high }}
  resizeMode={FastImage.resizeMode.cover}
/>
```

### iOS

**Metal Rendering:** Modern iOS uses Metal by default. Ensure you're not forcing software rendering.

**Optimize Images:**
```tsx
// Use appropriate scale factors
<Image 
  source={require('./image@3x.png')} 
  style={{ width: 100, height: 100 }}
/>
```

## Shadow and Border Optimization

Shadows are expensive, especially on Android.

```tsx
// ❌ Expensive: runtime shadow calculation
<View style={{
  shadowColor: '#000',
  shadowOffset: { width: 0, height: 2 },
  shadowOpacity: 0.25,
  shadowRadius: 4,
  elevation: 5,
}}>

// ✅ Better: pre-rendered shadow image
<ImageBackground 
  source={require('./shadow-bg.png')}
  style={styles.container}
>
```

### Shadow Alternatives

1. **Use elevation sparingly** on Android
2. **Pre-bake shadows** into images
3. **Use libraries** like react-native-shadow-2 for optimized shadows

## Measurement and Layout

### Avoid Layout Thrashing

```tsx
// ❌ Multiple measurements cause reflows
const measureAll = () => {
  const size1 = view1Ref.current?.measure();
  const size2 = view2Ref.current?.measure();
  const size3 = view3Ref.current?.measure();
};

// ✅ Batch measurements
const measureAll = () => {
  requestAnimationFrame(() => {
    // All measurements in single frame
    const sizes = [view1Ref, view2Ref, view3Ref].map(
      ref => ref.current?.measure()
    );
  });
};
```

### Use onLayout Sparingly

```tsx
// ❌ Heavy work in onLayout
<View onLayout={(e) => {
  const { width, height } = e.nativeEvent.layout;
  expensiveCalculation(width, height);
}}>

// ✅ Debounce or defer
const handleLayout = useMemo(
  () => debounce((width: number, height: number) => {
    expensiveCalculation(width, height);
  }, 100),
  []
);

<View onLayout={(e) => {
  const { width, height } = e.nativeEvent.layout;
  handleLayout(width, height);
}}>
```

## Checklist

```
□ View hierarchy is flat (see list-optimization.md)
□ Using native SDKs instead of WebView solutions
□ Native calls batched where possible
□ ProGuard/R8 enabled (see bundle-optimization.md)
□ Images optimized for platform
□ Shadows used sparingly or pre-rendered
□ Layout measurements batched
```
