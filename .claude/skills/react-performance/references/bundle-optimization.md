# Bundle Optimization

A smaller bundle means faster downloads, faster parsing, and faster startup. Understanding your bundle composition is the first step.

## Analyzing Bundle Size

### JS Bundle Analysis

Generate bundle with source map:

```bash
# React Native CLI
npx react-native bundle \
  --entry-file index.js \
  --bundle-output /tmp/bundle.js \
  --sourcemap-output /tmp/bundle.js.map \
  --dev false \
  --platform ios

# Analyze with source-map-explorer
npx source-map-explorer /tmp/bundle.js /tmp/bundle.js.map
```

### Using react-native-bundle-visualizer

```bash
npx react-native-bundle-visualizer
# Opens interactive treemap in browser
```

### What to Look For

- **Large dependencies**: lodash, moment.js, date-fns (full package)
- **Duplicate packages**: Same library at different versions
- **Unused code**: Dead code that didn't tree-shake
- **Heavy utilities**: Entire library for one function

## Tree Shaking

Remove unused exports from bundles.

### Enable in Metro

```js
// metro.config.js
module.exports = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: true, // Enables tree shaking
        inlineRequires: true,
      },
    }),
  },
};
```

### Write Tree-Shakeable Code

```tsx
// ❌ Default exports prevent tree shaking
export default {
  formatDate,
  parseDate,
  validateDate,
};

// ✅ Named exports enable tree shaking
export { formatDate, parseDate, validateDate };
```

### Direct Imports

```tsx
// ❌ Imports entire library
import { format } from 'date-fns';
// Bundle includes ALL of date-fns

// ✅ Direct import
import format from 'date-fns/format';
// Bundle includes only format
```

## Avoid Barrel Exports

Barrel files (index.ts re-exporting everything) break tree shaking.

```tsx
// components/index.ts (barrel file)
export { Button } from './Button';
export { Card } from './Card';
export { Modal } from './Modal';
export { Form } from './Form';
// ... 50 more exports

// ❌ Imports entire barrel
import { Button } from '@/components';
// All 50+ components included in bundle!

// ✅ Direct import
import { Button } from '@/components/Button';
// Only Button included
```

### Detecting Barrel Impact

```bash
# Check if barrel is causing bloat
npx react-native-bundle-visualizer

# Look for large "components" or "utils" chunks
# that shouldn't be fully included
```

## Lightweight Alternatives

| Heavy Library | Lightweight Alternative |
|---------------|------------------------|
| `moment` (67KB) | `date-fns` (tree-shakeable) or `dayjs` (2KB) |
| `lodash` (72KB) | `lodash-es` + direct imports |
| `axios` (13KB) | `ky` (3KB) or native `fetch` |
| `uuid` (9KB) | `nanoid` (1KB) |
| `validator` (48KB) | Write own or `yup`/`zod` |

### Example: Lodash

```tsx
// ❌ Full lodash (72KB)
import _ from 'lodash';
_.debounce(fn, 300);

// ✅ Direct import (~1KB per function)
import debounce from 'lodash/debounce';
debounce(fn, 300);

// ✅ Or use lodash-es with tree shaking
import { debounce } from 'lodash-es';
```

## Platform-Specific Bundles

Exclude platform-specific code from wrong platform.

```tsx
// Button.ios.tsx
export const Button = () => <IOSButton />;

// Button.android.tsx  
export const Button = () => <AndroidButton />;

// Usage (Metro automatically selects correct file)
import { Button } from './Button';
```

Metro excludes `.ios.tsx` from Android bundle and vice versa.

## App Bundle Size (Native)

### Android

Analyze APK:
```bash
# Build release APK
./gradlew assembleRelease

# Analyze with bundletool
bundletool build-apks --bundle=app.aab --output=app.apks
bundletool get-size total --apks=app.apks
```

Or use Android Studio: Build → Analyze APK

#### Shrink with R8

```groovy
// android/app/build.gradle
android {
    buildTypes {
        release {
            minifyEnabled true
            shrinkResources true
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt')
        }
    }
}
```

### iOS

Check in Xcode: Window → Organizer → Select Archive → App Thinning Size Report

## Native Assets

Large assets increase bundle size. Optimize them:

### Images

```bash
# Compress PNGs
pngquant --quality=65-80 image.png

# Use WebP for better compression
cwebp image.png -o image.webp
```

### Fonts

- Only include needed font weights
- Subset fonts to used characters
- Use system fonts when possible

## Code Splitting (Advanced)

> ⚠️ **You probably don't need this.** Metro with `React.lazy` + inline requires covers 90% of use cases. Re.Pack adds significant complexity (replaces Metro with Webpack). Consider only if:
> - Bundle > 10MB
> - Features used by <5% of users (admin panels, debug tools)
> - Building a super app with multiple mini-apps

Split bundle into chunks loaded on demand.

### With Re.Pack

```bash
npm install @callstack/repack
```

```js
// webpack.config.js
module.exports = {
  // ... Re.Pack configuration
  optimization: {
    splitChunks: {
      chunks: 'all',
    },
  },
};
```

```tsx
// Lazy load feature modules
const AdminModule = lazy(() => import('./modules/Admin'));
```

## Monitoring Bundle Size

### CI Integration

```bash
# In CI pipeline
npx react-native bundle --dev false --bundle-output bundle.js

# Check size
ls -la bundle.js | awk '{print $5}'

# Fail if over threshold
MAX_SIZE=5000000  # 5MB
ACTUAL_SIZE=$(stat -f%z bundle.js)
if [ $ACTUAL_SIZE -gt $MAX_SIZE ]; then
  echo "Bundle too large: $ACTUAL_SIZE bytes"
  exit 1
fi
```

### Track Over Time

Keep bundle size in your metrics dashboard. Alert on significant increases.

## Optimization Checklist

```
□ Analyzed bundle with source-map-explorer
□ Identified largest dependencies
□ Replaced heavy libs with lightweight alternatives
□ Enabled tree shaking in Metro
□ Using direct imports (not barrels)
□ Platform-specific code in .ios/.android files
□ R8/ProGuard enabled for Android
□ Images compressed (PNG/WebP)
□ Fonts subsetted
□ Bundle size tracked in CI
```

## Quick Wins

1. **Replace moment.js** → Save ~60KB
2. **Direct lodash imports** → Save ~50KB
3. **Remove unused dependencies** → Varies
4. **Enable R8 (Android)** → 10-20% reduction
5. **Compress images** → Varies by assets
