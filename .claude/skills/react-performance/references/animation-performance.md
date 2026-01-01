# Animation Performance

Smooth animations require staying under 16ms per frame (60 FPS) or 8ms (120 FPS). Running animations on the UI thread avoids JS thread bottlenecks.

## Animation Architecture

```
┌─────────────────┐     ┌──────────────────┐
│   JS Thread     │     │   UI Thread      │
├─────────────────┤     ├──────────────────┤
│ React render    │     │ Native rendering │
│ Business logic  │     │ Layout           │
│ State updates   │     │ Animations ✓     │
└─────────────────┘     └──────────────────┘
        │                        │
        └── Async bridge ────────┘
            (bottleneck)
```

**Key insight**: Animations should run on the UI thread to avoid the async bridge.

## Animated API (Built-in)

React Native's `Animated` API with `useNativeDriver: true` runs on UI thread.

```tsx
import { Animated } from 'react-native';

const FadeIn = () => {
  const opacity = useRef(new Animated.Value(0)).current;
  
  useEffect(() => {
    Animated.timing(opacity, {
      toValue: 1,
      duration: 300,
      useNativeDriver: true, // ✅ Required for UI thread
    }).start();
  }, []);
  
  return (
    <Animated.View style={{ opacity }}>
      <Content />
    </Animated.View>
  );
};
```

### Native Driver Limitations

`useNativeDriver: true` only supports:
- `opacity`
- `transform` (translateX, translateY, scale, rotate, etc.)

**NOT supported** (must run on JS thread):
- `width`, `height`
- `padding`, `margin`
- `backgroundColor`
- Layout properties

```tsx
// ❌ Cannot use native driver
Animated.timing(width, {
  toValue: 200,
  useNativeDriver: true, // Error!
});

// ✅ Use transform instead
Animated.timing(scaleX, {
  toValue: 2, // Effectively doubles width
  useNativeDriver: true,
});
```

### Cleanup

Always stop animations on unmount:

```tsx
useEffect(() => {
  const animation = Animated.timing(value, config);
  animation.start();
  return () => animation.stop();
}, []);
```

## React Native Reanimated

Industry standard for complex animations. Runs on UI thread via worklets.

### Basic Usage

```tsx
import Animated, { 
  useSharedValue, 
  useAnimatedStyle, 
  withTiming 
} from 'react-native-reanimated';

const Component = () => {
  const opacity = useSharedValue(0);
  
  const animatedStyle = useAnimatedStyle(() => ({
    opacity: opacity.value, // Runs on UI thread
  }));
  
  useEffect(() => {
    opacity.value = withTiming(1, { duration: 300 });
  }, []);
  
  return <Animated.View style={animatedStyle} />;
};
```

### Worklets

JavaScript functions that run on the UI thread.

```tsx
const handleGesture = () => {
  'worklet'; // Marks as worklet
  // This runs on UI thread
  opacity.value = withSpring(1);
};

// useAnimatedStyle callback is implicitly a worklet
const style = useAnimatedStyle(() => {
  console.log('Running on UI thread');
  return { opacity: opacity.value };
});
```

### Communicating Between Threads

```tsx
// UI thread → JS thread
import { runOnJS } from 'react-native-reanimated';

const onComplete = () => {
  // Regular JS function
  navigation.goBack();
};

const animatedStyle = useAnimatedStyle(() => {
  if (progress.value === 1) {
    runOnJS(onComplete)(); // Call JS from UI thread
  }
  return { transform: [{ translateX: progress.value * 100 }] };
});

// JS thread → UI thread
import { runOnUI } from 'react-native-reanimated';

const triggerAnimation = () => {
  runOnUI(() => {
    'worklet';
    opacity.value = withSpring(1);
  })();
};
```

### Layout Animations

```tsx
import Animated, { 
  FadeIn, 
  FadeOut, 
  Layout 
} from 'react-native-reanimated';

const Item = ({ visible }: Props) => {
  if (!visible) return null;
  
  return (
    <Animated.View 
      entering={FadeIn.duration(200)}
      exiting={FadeOut.duration(200)}
      layout={Layout.springify()}
    >
      <Content />
    </Animated.View>
  );
};
```

## Deferring Work During Animations

### Modern Approach: startTransition

Use `startTransition` to defer heavy work without blocking animations.

```tsx
import { startTransition } from 'react';
import { useFocusEffect } from '@react-navigation/native';

const Screen = () => {
  const [data, setData] = useState<Data | null>(null);
  
  useFocusEffect(
    useCallback(() => {
      startTransition(() => {
        // Heavy work runs without blocking navigation animation
        setData(processLargeDataset());
      });
    }, [])
  );
  
  return <Content data={data} />;
};
```

See [concurrent-react.md](concurrent-react.md) for more on transitions.

### Legacy Approach: InteractionManager

For older codebases or specific native animation timing:

```tsx
import { InteractionManager } from 'react-native';

useFocusEffect(
  useCallback(() => {
    const task = InteractionManager.runAfterInteractions(() => {
      // Runs after ALL native animations complete
      loadData();
    });
    return () => task.cancel();
  }, [])
);
```

#### Manual Interaction Handles

Block `runAfterInteractions` during custom animations:

```tsx
const handle = InteractionManager.createInteractionHandle();

Animated.timing(value, config).start(() => {
  InteractionManager.clearInteractionHandle(handle);
  // Now runAfterInteractions callbacks execute
});
```

### When to Use Which

| Approach | Use When |
|----------|----------|
| `startTransition` | Modern React (18+), state updates, most cases |
| `InteractionManager` | Legacy code, need to wait for specific native animations |

## Gesture + Animation

Gestures handled on UI thread for responsive feel.

```tsx
import { Gesture, GestureDetector } from 'react-native-gesture-handler';
import Animated, { 
  useSharedValue, 
  useAnimatedStyle, 
  withSpring 
} from 'react-native-reanimated';

const DraggableCard = () => {
  const translateX = useSharedValue(0);
  const translateY = useSharedValue(0);
  
  const gesture = Gesture.Pan()
    .onUpdate((e) => {
      // Runs on UI thread
      translateX.value = e.translationX;
      translateY.value = e.translationY;
    })
    .onEnd(() => {
      // Snap back with spring
      translateX.value = withSpring(0);
      translateY.value = withSpring(0);
    });
  
  const animatedStyle = useAnimatedStyle(() => ({
    transform: [
      { translateX: translateX.value },
      { translateY: translateY.value },
    ],
  }));
  
  return (
    <GestureDetector gesture={gesture}>
      <Animated.View style={[styles.card, animatedStyle]} />
    </GestureDetector>
  );
};
```

## Common Performance Issues

### JS Thread Blocking During Animation

```tsx
// ❌ Heavy computation blocks animation
const onPress = () => {
  startAnimation();
  processLargeDataset(); // Blocks JS thread
};

// ✅ Defer with startTransition (modern)
const onPress = () => {
  startAnimation();
  startTransition(() => {
    setProcessedData(processLargeDataset());
  });
};

// ✅ Or with InteractionManager (legacy)
const onPress = () => {
  startAnimation();
  InteractionManager.runAfterInteractions(() => {
    processLargeDataset();
  });
};
```

### Too Many Animated Values

```tsx
// ❌ Many independent values = overhead
const x = useSharedValue(0);
const y = useSharedValue(0);
const scale = useSharedValue(1);
const rotate = useSharedValue(0);
// 4 subscriptions, 4 style updates

// ✅ Combine when updated together
const transform = useSharedValue({ x: 0, y: 0, scale: 1, rotate: 0 });

const style = useAnimatedStyle(() => ({
  transform: [
    { translateX: transform.value.x },
    { translateY: transform.value.y },
    { scale: transform.value.scale },
    { rotate: `${transform.value.rotate}rad` },
  ],
}));
```

### Expensive Style Calculations

```tsx
// ❌ Complex calculation every frame
const style = useAnimatedStyle(() => {
  const computed = heavyCalculation(progress.value);
  return { transform: [{ translateX: computed }] };
});

// ✅ Precompute or interpolate
const style = useAnimatedStyle(() => {
  const x = interpolate(progress.value, [0, 1], [0, 200]);
  return { transform: [{ translateX: x }] };
});
```

## Performance Checklist

```
□ Using useNativeDriver: true with Animated API
□ Or using Reanimated for complex animations
□ Heavy work deferred with startTransition (or InteractionManager)
□ Gestures handled with react-native-gesture-handler
□ Not blocking JS thread during animations
□ Animations stopped on component unmount
□ Using interpolate instead of heavy calculations
□ Minimal animated values (combine when possible)
```

## Debug Animation Performance

1. Enable Perf Monitor (Dev Menu)
2. Watch UI FPS during animation
3. If UI FPS drops but JS FPS is fine → Native issue
4. If JS FPS drops → Animation running on JS thread or blocking
