# perf: optimise tokenlistitem selectors to use weakmap parameter memoization
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/16095>
- **Merged Date:** 2025-06-05
- **Category:** Rendering (+ Lists)

## Problem
Each `TokenListItem` subscribed to whole controller state (`selectTokensBalances`, `selectTokenMarketData`, `selectCurrencyRates`, network configurations) and also ran `useEarnTokens` per row, then picked out its own entry. Any change to those large objects re-rendered every row, and reselect's default LRU memoization (cache size 1) thrashes when many rows call one selector with different arguments. The PR shares before/after recordings but no numbers.

## Solution
- Add per-token selectors (`selectSingleTokenBalance`, `selectSingleTokenPriceMarketData`, `selectPricePercentChange1d`, `selectCurrencyRateForChainId`) so rows select only the small value they need.
- Build them with reselect's `weakMapMemoize` for both `memoize` and `argsMemoize`, which keeps a cache per distinct argument set instead of a single-entry cache.
- Replace the per-row `useEarnTokens` call with a narrow `useHasSupportedStablecoin` hook.
- Add `selectHasAnyBalance` so `ProtectWalletMandatoryModal` no longer subscribes to the full balances object, and wrap `TokenList` in `React.memo`.

## Code Pattern (Before vs. After)
```tsx
// Before
const multiChainTokenBalance = useSelector(selectTokensBalances);
const multiChainMarketData = useSelector(selectTokenMarketData);
const multiChainCurrencyRates = useSelector(selectCurrencyRates);
const earnStablecoins = useEarnTokens({ includeLendingTokens: true });
const exchangeRates = multiChainMarketData?.[chainId as Hex];
const tokenBalances =
  multiChainTokenBalance?.[selectedInternalAccountAddress as Hex]?.[chainId as Hex];

// After
const exchangeRates = useSelector((state: RootState) =>
  selectSingleTokenPriceMarketData(state, chainId, asset?.address as Hex),
);
const tokenBalances = useSelector((state: RootState) =>
  selectSingleTokenBalance(state, selectedInternalAccountAddress as Hex, chainId, asset?.address as Hex),
);
const conversionRate = useSelector((state: RootState) =>
  selectCurrencyRateForChainId(state, chainId as Hex),
);
```

```ts
// selectors/tokenBalancesController.ts
export const selectSingleTokenBalance = createSelector(
  [
    (state, accountAddress, chainId, tokenAddress) =>
      selectTokenBalancesControllerState(state).tokenBalances?.[accountAddress]?.[chainId]?.[tokenAddress],
    (_state, _accountAddress, _chainId, tokenAddress) => tokenAddress,
  ],
  (balance, tokenAddress) => (balance ? { [tokenAddress]: balance } : {}),
  { memoize: weakMapMemoize, argsMemoize: weakMapMemoize },
);
```

## Takeaway
For selectors called with different arguments by many list rows, select the smallest slice possible and use `weakMapMemoize` instead of the default single-entry cache.
