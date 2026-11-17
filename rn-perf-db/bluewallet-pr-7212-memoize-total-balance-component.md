# FIX: Consntant rerender of total balance would slow down app
- **Repository:** BlueWallet/BlueWallet
- **PR Link:** <https://github.com/BlueWallet/BlueWallet/pull/7212>
- **Merged Date:** 2024-10-26
- **Category:** Rendering

## Problem
`TotalWalletsBalance` recomputed the sum over all wallets, rebuilt its `StyleSheet`, rebuilt the tooltip action list and recreated its handlers on every render. Since it sits on the wallets list and re-renders with store updates, this repeated work made the app feel slow, as the PR title says. The PR gives no measurements.

## Solution
- `totalBalance` is wrapped in `useMemo` keyed on `wallets`.
- The per-render `StyleSheet.create` is replaced with a `useMemo` object keyed on the single colour it depends on.
- The tooltip menu is built once per unit/currency change. All three "view in" actions are always present and toggled with a `hidden` flag, instead of branching to pick one.
- `onPressMenuItem` becomes a `useCallback` without `totalBalancePreferredUnit` in its dependencies. The tap handler for cycling units is a separate `useCallback`.
- The early `return null` for fewer than two wallets is hoisted after the hooks, which keeps the JSX flat.

## Code Pattern (Before vs. After)
```tsx
// Before
const styleHooks = StyleSheet.create({
  balance: { color: colors.foregroundColor },
  currency: { color: colors.foregroundColor },
});

const totalBalance = wallets.reduce((prev, curr) => {
  if (!curr.hideBalance) {
    return prev + curr.getBalance();
  }
  return prev;
}, 0);

const onPressMenuItem = useMemo(
  () => async (id: string) => {
    // switch on id and on totalBalancePreferredUnit ...
  },
  [totalBalancePreferredUnit, setIsTotalBalanceEnabledStorage, formattedBalance, setTotalBalancePreferredUnitStorage],
);

// After
const styleHooks = useMemo(
  () => ({
    balance: { color: colors.foregroundColor },
    currency: { color: colors.foregroundColor },
  }),
  [colors.foregroundColor],
);

const totalBalance = useMemo(() => wallets.reduce((prev, curr) => (!curr.hideBalance ? prev + curr.getBalance() : prev), 0), [wallets]);

const onPressMenuItem = useCallback(
  async (id: string) => {
    if (id === CommonToolTipActions.ViewInFiat.id) {
      await setTotalBalancePreferredUnitStorage(BitcoinUnit.LOCAL_CURRENCY);
    } else if (id === CommonToolTipActions.ViewInSats.id) {
      await setTotalBalancePreferredUnitStorage(BitcoinUnit.SATS);
    } // ...
  },
  [setIsTotalBalanceEnabledStorage, formattedBalance, setTotalBalancePreferredUnitStorage],
);

if (wallets.length <= 1) return null;
```

## Takeaway
Memoize aggregates over store data, style objects and menu definitions in components that sit under frequently updating contexts, and give callbacks minimal dependency lists so child menus do not rebuild.
