# fix: cp-7.46.0 Improve `TokenList` performance with `FlashList` and improved data flow
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/14988>
- **Merged Date:** 2025-04-30
- **Category:** Lists (+ Rendering)

## Problem
The wallet token list became slow and could freeze the app on accounts with many assets (EVM and Solana), especially when toggling the Popular Networks / Current Network filter. The full sorted and filtered asset objects were passed down as props to a `FlatList`, so every update re-rendered large data. The PR gives no numeric before/after, only recordings.

## Solution
- Use `FlashList` with `estimatedItemSize`, `estimatedListSize`, `removeClippedSubviews` and a `viewabilityConfig`, and call `recomputeViewableItems()` when the network filter changes.
- Memoize `renderItem` with `useCallback`.
- Sorting and filtering stay in the parent `Tokens` component in one `useMemo`, which now passes only a minimal list of `{ address, chainId }` keys to the list.
- Each `TokenListItem` selects its own asset from Redux via new memoized single-token selectors (EVM and multichain), so unrelated changes do not re-render every row.
- `ScamWarningModal` moved from each row to the parent, and the footer reads its own data instead of receiving the whole token array.
- Caveat from the PR: very large lists may briefly show blank space when scrolling aggressively.

## Code Pattern (Before vs. After)
```tsx
// Before
<FlatList
  data={tokens}
  renderItem={({ item }) => (
    <TokenListItem asset={item} showScamWarningModal={showScamWarningModal} /* ... */ />
  )}
  keyExtractor={(_, index) => index.toString()}
/>

// After
const renderTokenListItem = useCallback(
  ({ item }: { item: { address: string; chainId: string | undefined } }) => (
    <TokenListItem assetKey={item} showRemoveMenu={showRemoveMenu} /* ... */ />
  ),
  [showRemoveMenu, setShowScamWarningModal, privacyMode, showPercentageChange],
);

<FlashList
  ref={listRef}
  data={tokenKeys}
  estimatedItemSize={itemHeight}
  estimatedListSize={{ height: estimatedListHeight, width: deviceWidth }}
  removeClippedSubviews
  renderItem={renderTokenListItem}
  keyExtractor={(item) => `${item.address}-${item.chainId}`}
  extraData={{ isTokenNetworkFilterEqualCurrentNetwork, listLength }}
/>
```

```tsx
// TokenListItem: select the row's own data from Redux
const selectEvmAsset = useMemo(makeSelectAssetByAddressAndChainId, []);
const evmAsset = useSelector((state: RootState) =>
  selectEvmAsset(state, { address: assetKey.address, chainId: assetKey.chainId ?? '' }),
);
```

## Takeaway
Pass list rows only an id/key and let each row select its data from the store, and give FlashList accurate size estimates.
