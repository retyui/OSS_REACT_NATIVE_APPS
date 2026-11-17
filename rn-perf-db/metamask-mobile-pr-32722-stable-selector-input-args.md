# perf: fix `Set` input arg defeating memo in `selectAccountGroupsByAddress`
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/32722>
- **Merged Date:** 2026-07-03
- **Category:** Rendering

## Problem
`selectAccountGroupsByAddress` built a `new Set(...)` inside its second input selector. Because the Set is a fresh reference each call, `createDeepEqualSelector` always saw changed input and re-ran the O(groups x accounts) result function on every call, even with unchanged addresses. No timing numbers are reported; a unit test asserts reference stability.

## Solution
- The input selector now returns the raw `addresses` array, which deep-equal can compare by content; the `Set` is built inside the result function.
- Callers (`SnapUIAvatar`, `useDisplayName`) memoize the `[address]` array with `useMemo` so its reference is stable across renders.

## Code Pattern (Before vs. After)
```ts
// Before
export const selectAccountGroupsByAddress = createDeepEqualSelector(
  [
    selectAccountGroupWithInternalAccounts,
    (_state: RootState, addresses: string[]) =>
      new Set(addresses.map((address) => address.toLowerCase())),
  ],
  (accountGroupWithInternalAccounts, addressesSet: Set<string>) => {
    // ...
  },
);

// After
export const selectAccountGroupsByAddress = createDeepEqualSelector(
  [
    selectAccountGroupWithInternalAccounts,
    (_state: RootState, addresses: string[]) => addresses,
  ],
  (accountGroupWithInternalAccounts, addresses: string[]) => {
    const addressesSet = new Set(addresses.map((a) => a.toLowerCase()));
    // ...
  },
);
```

```tsx
// Before
const accountGroups = useSelector((state: RootState) =>
  selectAccountGroupsByAddress(state, [address]),
);

// After
const addressArray = useMemo(() => [address], [address]);
const accountGroups = useSelector((state: RootState) =>
  selectAccountGroupsByAddress(state, addressArray),
);
```

## Takeaway
Selector input functions must return stable, comparable values; do transformations (Set/Map construction) inside the result function.
