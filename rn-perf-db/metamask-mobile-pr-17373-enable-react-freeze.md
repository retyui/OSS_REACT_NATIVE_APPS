# perf: Enable react-freeze
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/17373>
- **Merged Date:** 2025-07-24
- **Category:** Rendering

## Problem
Screens that were not visible (for example the wallet view after navigating away) kept re-rendering in the background. The PR shows recordings of the wallet view still re-rendering while inactive; it reports no numbers.

## Solution
- Call `enableFreeze(true)` from `react-native-screens` at app entry. React Navigation already detaches inactive screens, and with freeze enabled their React trees are suspended via React Freeze, so they stop rendering until focused again.
- Three-line change; the PR recordings show the wallet view no longer re-rendering when inactive.
- Caveat: frozen screens will not update while hidden, so code that relied on background renders must tolerate that.

## Code Pattern (Before vs. After)
```js
// Before
import {
  setReactNativeDefaultHandler,
} from './app/core/ErrorHandler';

// Setup Sentry
setupSentry();

// After
import {
  setReactNativeDefaultHandler,
} from './app/core/ErrorHandler';

import { enableFreeze } from 'react-native-screens';
enableFreeze(true);

// Setup Sentry
setupSentry();
```

## Takeaway
With React Navigation and react-native-screens, enable freeze once at startup to stop background screens from re-rendering.
