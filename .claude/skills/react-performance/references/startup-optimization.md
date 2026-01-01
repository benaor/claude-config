# Startup Optimization

Time to Interactive (TTI) measures how quickly users can interact with your app. Optimizing startup impacts first impressions and retention.

## Startup Phases

```
App Launch
    │
    ▼
┌─────────────────────────────┐
│ 1. Native Initialization    │  ← OS + native code
│    - Process creation       │
│    - Native modules load    │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 2. JS Engine Init           │  ← Hermes startup
│    - Hermes initialization  │
│    - Bytecode loading       │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 3. JS Bundle Execution      │  ← Your code runs
│    - Module evaluation      │
│    - Initial render         │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 4. First Meaningful Paint   │  ← User sees content
│    - Layout calculation     │
│    - Native view creation   │
└─────────────────────────────┘
    │
    ▼
┌─────────────────────────────┐
│ 5. Interactive              │  ← User can interact
│    - Event handlers ready   │
│    - Data loaded            │
└─────────────────────────────┘
```

## Measuring TTI

Guide users to measure Time to Interactive with these tools:

### Native Profilers

**Android (Android Studio Profiler)**:
1. Run app in profile mode
2. Open Android Studio → View → Tool Windows → Profiler
3. Select your app process
4. Check CPU timeline for startup phases

**iOS (Instruments)**:
1. Xcode → Product → Profile
2. Select "App Launch" template
3. Record launch sequence

### React Native DevTools

1. Open DevTools → Profiler
2. Click "Reload and start profiling"
3. Analyze initial render flame graph

## Optimization Strategies

### 1. Hermes (Required)

Hermes pre-compiles JS to bytecode, dramatically reducing startup time.

```js
// android/app/build.gradle
project.ext.react = [
    enableHermes: true  // ✅ Should be default in new projects
]
```

Verify Hermes is enabled:
```tsx
const isHermes = () => !!global.HermesInternal;
console.log('Hermes enabled:', isHermes());
```

### 2. Lazy Loading Screens

Don't load all screens upfront.

```tsx
// ❌ Eager: all screens loaded at startup
import HomeScreen from './screens/Home';
import ProfileScreen from './screens/Profile';
import SettingsScreen from './screens/Settings';

// ✅ Lazy: screens loaded on demand
const HomeScreen = lazy(() => import('./screens/Home'));
const ProfileScreen = lazy(() => import('./screens/Profile'));
const SettingsScreen = lazy(() => import('./screens/Settings'));

// With Suspense fallback
const App = () => (
  <Suspense fallback={<LoadingScreen />}>
    <Navigator>
      <Screen name="Home" component={HomeScreen} />
      <Screen name="Profile" component={ProfileScreen} />
      <Screen name="Settings" component={SettingsScreen} />
    </Navigator>
  </Suspense>
);
```

### 3. Defer Non-Critical Initialization

```tsx
// ❌ Everything at startup
const App = () => {
  useEffect(() => {
    initializeAnalytics();
    initializeCrashReporting();
    preloadImages();
    syncOfflineData();
    checkForUpdates();
  }, []);
};

// ✅ Prioritize critical path
const App = () => {
  useEffect(() => {
    // Critical: needed immediately
    initializeCrashReporting();
  }, []);
  
  useEffect(() => {
    // Defer with transition (modern)
    startTransition(() => {
      initializeAnalytics();
      preloadImages();
    });
    
    // Low priority: run later
    setTimeout(() => {
      syncOfflineData();
      checkForUpdates();
    }, 3000);
  }, []);
};
```

### 4. Inline Requires

Load modules when needed, not at startup.

```tsx
// ❌ Top-level import: loaded at startup
import { processData } from './heavyModule';

const Component = () => {
  const handlePress = () => {
    const result = processData(data);
  };
};

// ✅ Inline require: loaded on demand
const Component = () => {
  const handlePress = () => {
    const { processData } = require('./heavyModule');
    const result = processData(data);
  };
};
```

Enable in Metro config:
```js
// metro.config.js
module.exports = {
  transformer: {
    inlineRequires: true,
  },
};
```

### 5. Reduce Initial Bundle

Only include essential code in main bundle.

```tsx
// Avoid barrel exports at app root
// ❌ Imports entire module tree
import { Button, Card, Modal } from '@/components';

// ✅ Direct imports
import Button from '@/components/Button';
```

See [bundle-optimization.md](bundle-optimization.md) for more.

### 6. Optimize Initial Data Loading

```tsx
// ❌ Blocks render until data loads
const HomeScreen = () => {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetchData().then(setData).finally(() => setLoading(false));
  }, []);
  
  if (loading) return <FullScreenLoader />; // Blocks UI
  
  return <Content data={data} />;
};

// ✅ Render immediately, load data progressively
const HomeScreen = () => {
  const [data, setData] = useState<Data | null>(null);
  
  useEffect(() => {
    fetchData().then(setData);
  }, []);
  
  return (
    <View>
      <Header /> {/* Shows immediately */}
      {data ? <Content data={data} /> : <ContentSkeleton />}
    </View>
  );
};
```

### 7. Splash Screen Strategy

Keep splash screen while loading critical resources.

```tsx
import * as SplashScreen from 'expo-splash-screen';
// Or: import RNBootSplash from 'react-native-bootsplash';

// Prevent auto-hide
SplashScreen.preventAutoHideAsync();

const App = () => {
  const [ready, setReady] = useState(false);
  
  useEffect(() => {
    async function prepare() {
      // Load critical resources
      await loadFonts();
      await loadCriticalData();
      setReady(true);
    }
    prepare();
  }, []);
  
  useEffect(() => {
    if (ready) {
      SplashScreen.hideAsync();
    }
  }, [ready]);
  
  if (!ready) return null;
  
  return <AppContent />;
};
```

## Platform-Specific Optimizations

### Android

**Enable ProGuard/R8** for smaller native code:
```groovy
// android/app/build.gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android.txt')
        }
    }
}
```

### iOS

**Enable dead code stripping**:
- Xcode → Build Settings → Dead Code Stripping → Yes

## Startup Checklist

```
□ Hermes enabled
□ Screens lazy-loaded with React.lazy
□ Non-critical init deferred (startTransition/setTimeout)
□ Inline requires enabled in Metro
□ No barrel exports at app entry
□ Initial render shows UI immediately (skeletons)
□ Splash screen managed programmatically
□ ProGuard/R8 enabled (Android)
□ Measured TTI before/after optimizations
```

## Measuring Impact

Before and after each optimization:

1. Fresh install (clear cache)
2. Force stop app
3. Cold launch
4. Measure time to first meaningful content
5. Repeat 5+ times for average

```bash
# Android: measure cold start time
adb shell am start-activity -W com.yourapp/.MainActivity

# Output shows TotalTime in ms
```
