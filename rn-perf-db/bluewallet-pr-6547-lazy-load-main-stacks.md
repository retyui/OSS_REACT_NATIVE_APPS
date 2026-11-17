# REF: Lazy load main stacks
- **Repository:** BlueWallet/BlueWallet
- **PR Link:** <https://github.com/BlueWallet/BlueWallet/pull/6547>
- **Merged Date:** 2024-05-12
- **Category:** Startup

## Problem
`Navigation.tsx` statically imported every screen, nested stack and the drawer content at the top of the file. With Metro, a top-level `import` evaluates the module and its whole dependency tree at launch, so all wallet, send, lightning and settings screens were loaded before the first screen (the unlock screen) could render. The PR reports no before/after numbers.

## Solution
- The two main navigators (`DetailViewScreensStack` and `DrawerRoot`) were moved into their own files and are loaded with `React.lazy(() => import(...))`.
- `MainRoot` renders only a small `UnlockRoot` (just `UnlockWith`) until `walletsInitialized` is true.
- After unlock, the heavy navigator is mounted inside `<Suspense fallback={<LazyLoadingIndicator />}>`, so its module graph is first evaluated at that point.
- Handsets get the stack directly, larger screens get the drawer root.
- Caveat: a short loading indicator can show right after unlock while the chunk is evaluated.

## Code Pattern (Before vs. After)
```tsx
// Before
import AddWalletStack from './navigation/AddWalletStack';
import WalletsList from './screen/wallets/WalletsList';
import WalletTransactions from './screen/wallets/transactions';
import SendDetailsStack from './navigation/SendDetailsStack';
// ... ~60 more static screen/stack imports

const DetailViewStackScreensStack = () => {
  const { walletsInitialized } = useContext(BlueStorageContext);
  // ...
  return (
    <DetailViewRoot.Navigator initialRouteName="UnlockWithScreen">
      {!walletsInitialized ? (
        <DetailViewRoot.Screen name="UnlockWithScreen" component={UnlockWith} />
      ) : (
        <DetailViewRoot.Group>{/* every screen */}</DetailViewRoot.Group>
      )}
    </DetailViewRoot.Navigator>
  );
};

// After
import React, { Suspense, lazy } from 'react';
import UnlockWith from './screen/UnlockWith';
import { LazyLoadingIndicator } from './navigation/LazyLoadingIndicator';

const DetailViewScreensStack = lazy(() => import('./navigation/DetailViewScreensStack'));
const DrawerRoot = lazy(() => import('./navigation/DrawerRoot'));

const MainRoot = () => {
  const { walletsInitialized } = useStorage();
  if (!walletsInitialized) {
    return <UnlockRoot />;
  }
  const Component = isHandset ? DetailViewScreensStack : DrawerRoot;
  return (
    <Suspense fallback={<LazyLoadingIndicator />}>
      <Component />
    </Suspense>
  );
};
```

## Takeaway
Keep the first rendered route's import graph minimal and put the rest of the navigation tree behind `React.lazy` or `getComponent`, so it is evaluated only after the user gets past the gate.
