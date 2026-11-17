# perf(predict): fix JS-thread FPS collapse on market details with live prices cp-7.82.0
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/31801>
- **Merged Date:** 2026-06-16
- **Category:** Rendering (+ Networking)

## Problem
Opening a Predict market with a live Polymarket WebSocket price feed (most noticeably a live soccer game) collapsed JS-thread FPS to near 0. Every incoming price message did an `O(subscriptions x tokens)` `new Set(key.split(','))` allocation and re-rendered the whole details screen, because nothing was throttled or memoized. Profiling showed `performWorkOnRoot` at about 70% of the trace. The PR reports JS FPS going from about 0 to the mid-50s on live game events, and regular markets from about 25-35 to comparable smoothness.

## Solution
- `WebSocketManager`: precompute the token-id `Set` once per subscription at subscribe time, and coalesce price emission with a leading plus trailing 250 ms throttle, so subscribers are notified at most about 4 times per second. State is cleaned up on disconnect.
- `useLiveMarketPrices`: skip `setPrices` when a tick carries no change in `price`/`bestBid`/`bestAsk`, using a ref mirror of the latest map.
- `usePredictPrices`: memoize `JSON.stringify(queries)`.
- `useOpenOutcomes`: disable the live subscription for game markets (result never rendered) and preserve object identity for unchanged outcomes.
- Wrap `PredictMarketOutcome` and `PredictGameDetailsContent` in `React.memo`, and memoize the screen's press handlers.

## Code Pattern (Before vs. After)
```ts
// Before (useLiveMarketPrices)
setPrices((prevPrices) => {
  const newPrices = new Map(prevPrices);
  updates.forEach((update) => {
    newPrices.set(update.tokenId, update);
  });
  return newPrices;
});

// After
const current = pricesRef.current;
const hasChange = updates.some((update) => {
  const existing = current.get(update.tokenId);
  return (
    !existing ||
    existing.price !== update.price ||
    existing.bestBid !== update.bestBid ||
    existing.bestAsk !== update.bestAsk
  );
});

if (!hasChange) {
  return;
}

setPrices((prevPrices) => {
  const newPrices = new Map(prevPrices);
  updates.forEach((update) => {
    newPrices.set(update.tokenId, update);
  });
  pricesRef.current = newPrices;
  return newPrices;
});
```

```ts
// Before (WebSocketManager): per-message parse, no throttle
// (new Set(key.split(',')) rebuilt for every subscription on every message)

// After
private priceSubscriptionTokenSets: Map<string, Set<string>> = new Map();
// at subscribe time:
this.priceSubscriptionTokenSets.set(
  subscriptionKey,
  new Set(subscriptionKey.split(',').filter(Boolean)),
);

private scheduleMarketPriceEmit(): void {
  if (this.marketPriceEmitTimer === null) {
    this.flushMarketPriceUpdates();
    this.marketPriceEmitTimer = setTimeout(() => {
      this.marketPriceEmitTimer = null;
      if (this.marketPricePendingTokenIds.size > 0) {
        this.flushMarketPriceUpdates();
      }
    }, MARKET_PRICE_EMIT_THROTTLE_MS);
  }
}
```

## Takeaway
For high-frequency streams, precompute lookup structures at subscribe time, throttle fan-out with leading plus trailing edges, and drop no-op ticks before they reach `setState`.
