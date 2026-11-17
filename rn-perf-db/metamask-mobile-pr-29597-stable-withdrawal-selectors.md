# fix: stabilize useWithdrawalRequests selectors to prevent unnecessary re-renders
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/29597>
- **Merged Date:** 2026-05-01
- **Category:** Rendering

## Problem
Inline `[]` fallbacks inside `useWithdrawalRequests` selectors created a new array reference on every evaluation. This triggered React-Redux's "Selector unknown returned a different result" warning and caused unnecessary re-renders of `PerpsProgressBar` when the Perps tab mounted. The PR reports FPS on Perps tab mount going from a 60→22 drop to a stable 60→57, and the Perps screen no longer takes about a second to show.

## Solution
- Replaced inline `[]` literals with module-level constants (`EMPTY_WITHDRAWAL_REQUESTS`, `EMPTY_TX_HASHES`).
- Short-circuited `.filter()` when the source array is empty, so the selector returns the same constant reference instead of a freshly filtered array.
- With stable references, `useSelector` equality checks pass and the consumers skip re-rendering.

## Code Pattern (Before vs. After)
```tsx
// Before
const allWithdrawals = useStableArray(
  usePerpsSelector((state) => {
    const withdrawals = state?.withdrawalRequests || [];
    if (!selectedAddress) return [];
    return withdrawals.filter(
      (req) =>
        req.accountAddress?.toLowerCase() === selectedAddress.toLowerCase(),
    );
  }),
);
const lastCompletedTxHashes = usePerpsSelector(
  (state) => state?.lastCompletedWithdrawalTxHashes ?? [],
);

// After
const EMPTY_WITHDRAWAL_REQUESTS: WithdrawalRequest[] = [];
const EMPTY_TX_HASHES: string[] = [];

const allWithdrawals = useStableArray(
  usePerpsSelector((state) => {
    const withdrawals =
      state?.withdrawalRequests ?? EMPTY_WITHDRAWAL_REQUESTS;
    if (!selectedAddress || withdrawals.length === 0)
      return EMPTY_WITHDRAWAL_REQUESTS;
    return withdrawals.filter(
      (req) =>
        req.accountAddress?.toLowerCase() === selectedAddress.toLowerCase(),
    );
  }),
);
const lastCompletedTxHashes = usePerpsSelector(
  (state) => state?.lastCompletedWithdrawalTxHashes ?? EMPTY_TX_HASHES,
);
```

## Takeaway
Never return a literal `[]`/`{}` from a `useSelector` callback; hoist a module-level empty constant so the "no data" case keeps a stable reference.
