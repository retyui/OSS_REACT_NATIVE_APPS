# Boot time improvements, lazy tab loading
- **Repository:** rainbow-me/rainbow
- **PR Link:** <https://github.com/rainbow-me/rainbow/pull/6583>
- **Merged Date:** 2025-05-07
- **Category:** Startup

## Problem
Cold start waited on several async, variable-duration steps (remote config with a 3 s timeout, analytics init, migrations, WalletConnect and FCM token) before the wallet screen rendered, and all tabs were rendered up front. The PR reports iOS TTI averaged over 8 cold starts: 1,984 ms → 585 ms (-70.5%), with similar gains on Android.

## Solution
- Remote config moved from React Query to `createQueryStore`; after it has been fetched once for an app version, a fast path skips waiting, and unused keys are stripped. `useRemoteConfig('key', ...)` selects specific keys to limit re-renders.
- `analytics` initializes its client internally and queues events until ready, so init no longer blocks the app.
- WalletConnect and FCM token are no longer awaited.
- `migrate()` first checks the last migration and skips the loop if already done.
- `Swipe.Navigator` starts with `lazy: true` (with a placeholder), then lazy is turned off via `requestIdleCallback` once the wallet screen mounts.
- `useNavigation` replaced with `Navigation.x` calls in heavy top-level components, since it re-renders on every tab change; less critical boot work wrapped in `requestIdleCallback`.

## Code Pattern (Before vs. After)
```tsx
// Before
const { dapp_browser, points_enabled } = useRemoteConfig();
// ...
await Promise.all([initializeRemoteConfig(), migrate(), analytics.initializeRudderstack()]);

export async function migrate() {
  await runMigrations(migrations);
}

// After
const { dapp_browser, points_enabled } = useRemoteConfig('dapp_browser', 'points_enabled');
const lazy = useNavigationStore(state => enableLazyTabs || !state.isWalletScreenMounted);

const getScreenOptions = useCallback(
  (props: { route: RouteProp<ParamListBase, string> }) => ({
    animationEnabled: false,
    lazy,
    lazyPlaceholder: () => <LazyPlaceholder route={props.route.name} />,
    // ...
  }),
  [isCoinListEdited, lazy]
);

export async function migrate(): Promise<void> {
  if (isFullyMigrated()) return;
  await runMigrations(migrations);
}
```

## Takeaway
Take non-essential async work off the pre-render path, render only the focused tab first and enable the rest when idle, and short-circuit migrations that have already run.
