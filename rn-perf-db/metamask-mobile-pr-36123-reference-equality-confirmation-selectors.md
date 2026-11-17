# perf(confirmations): rely on reference equality in confirmation selectors
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/36123>
- **Merged Date:** 2026-09-11
- **Category:** Rendering

## Problem
Several hooks on the confirmation path compared Redux state by deep equality (`isEqual`, `createDeepEqualSelector`, `cloneDeep`) before deciding to re-render. Controller state arrives through Immer, which already preserves identity of unchanged subtrees, so these deep comparisons re-derived a guarantee the state layer provides, on every store update and proportional to state size. The PR reports no measurements.

## Solution
- `useApprovalRequest` reads a memoized `selectFirstPendingApproval` selector instead of deep-comparing the whole pending-approvals map, rebuilding an array, and deep-cloning the result.
- Transaction and gas-fee selectors use `createSelector` instead of `createDeepEqualSelector`; `useGasFeeEstimates` drops its `isEqual` comparator.
- `selectTransactions` is exported directly so consumers share one memoized instance.
- Empty fallbacks are hoisted to module-level constants for stable references.

## Code Pattern (Before vs. After)
```tsx
// Before
const pendingApprovals = useSelector(selectPendingApprovals, isEqual);
const pendingApprovalList = Object.values(pendingApprovals ?? {});
const firstPendingApproval = pendingApprovalList[0] as
  | ApprovalRequestType
  | undefined;
const approvalRequest = useMemo(
  () => cloneDeep(firstPendingApproval),
  [firstPendingApproval],
);

// After
const approvalRequest = useSelector(selectFirstPendingApproval) as
  | ApprovalRequestType
  | undefined;
```

```ts
// After (selectors/approvalController.ts)
const DEFAULT_APPROVALS = {};

export const selectFirstPendingApproval = createSelector(
  selectPendingApprovals,
  (pendingApprovals) => Object.values(pendingApprovals ?? DEFAULT_APPROVALS)[0],
);
```

```tsx
// Before
const gasFeeEstimates = useSelector(
  (state: RootState) => selectGasFeeEstimatesByChainId(state, chainId),
  isEqual,
);

// After
const gasFeeEstimates = useSelector((state: RootState) =>
  selectGasFeeEstimatesByChainId(state, chainId),
);
```

## Takeaway
If state is produced immutably (Immer/Redux Toolkit), prefer reference equality; deep-equal comparators on large state cost more than the re-renders they avoid.
