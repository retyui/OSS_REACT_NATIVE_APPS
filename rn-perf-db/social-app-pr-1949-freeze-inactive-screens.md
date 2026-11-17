# Freeze inactive screens
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/1949>
- **Merged Date:** 2023-11-17
- **Category:** Rendering

## Problem
Screens deeper in the navigation stack (and other tabs) kept re-rendering on global updates such as the timestamp tick, even though they were not visible. The PR reports no timings; the test plan shows that before the change all timestamps re-rendered on each tick (even in other tabs or up the nav stack), and after it far fewer did.

## Solution
- Call `enableFreeze(true)` from `react-native-screens`, which integrates `react-freeze` into React Navigation.
- Frozen screens have their updates deprioritized; React processes their pending state updates after they are unfrozen, i.e. when the stack is popped.
- By default only screens more than one nav pop away are frozen, so it helps mostly when pushing deeper into the stack.
- Caveat noted: the mechanism is hacky but relies on a real React feature (Suspense-based); a first-class API is expected later. Integrating freeze manually into the pager was left as future work.

## Code Pattern (Before vs. After)
```tsx
// Before (src/App.native.tsx)
i18n.load('en', messages)
i18n.activate('en')

SplashScreen.preventAutoHideAsync()

// After
import {enableFreeze} from 'react-native-screens'
// ...
enableFreeze(true)
SplashScreen.preventAutoHideAsync()
```

## Takeaway
Enabling `enableFreeze(true)` is a one-line way to stop off-screen navigation screens from re-rendering; verify screens that must keep updating in the background.
