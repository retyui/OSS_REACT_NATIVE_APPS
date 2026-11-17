# perf(confirmations): defer metamask pay init until after first layout
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/36948>
- **Merged Date:** 2026-10-19
- **Category:** Startup (+ Rendering)

## Problem
Money deposit confirmations blocked their first frame on work not needed to draw it or accept input: automatic payment token selection, pay metrics, and alert evaluation. The PR reports no before/after timings.

## Solution
- A new `ConfirmationFirstFrameProvider` waits two animation frames after mount. The amount, keyboard and account selector render and are interactive on the first frame.
- Non-essential hooks (auto pay-token selection, pay metrics, post-quote config, Transak key pre-warm) moved into a child component (`CustomAmountInfoDeferredInit`) mounted only after the first frame; alert evaluation is also deferred.
- Alerts stay blocking while pending, so the confirmation cannot be submitted before they are evaluated.
- Only `moneyAccountDeposit` opts in. The PR also fixes layout shift (skeleton height, first-frame safe-area inset).

## Code Pattern (Before vs. After)
```tsx
// Before (custom-amount-info.tsx)
useEnsureTransakApiKey();
useAutomaticTransactionPayToken({
  autoSelectFiatPayment,
  disable: disablePay,
  preferredToken,
});
useTransactionPayMetrics();
useTransactionPayPostQuote();

// After
const firstFrame = useConfirmationFirstFrame();
const isDeferredInitReady = !firstFrame || firstFrame.isFirstFrameComplete;
// ...
{isDeferredInitReady && (
  <CustomAmountInfoDeferredInit
    autoSelectFiatPayment={autoSelectFiatPayment}
    disablePay={disablePay}
    preferredToken={preferredToken}
  />
)}
```

```tsx
// Provider: yield two rAF callbacks so the shell frame is presented first
useEffect(() => {
  let frame = requestAnimationFrame(() => {
    frame = requestAnimationFrame(() => {
      setIsFirstFrameComplete(true);
    });
  });
  return () => cancelAnimationFrame(frame);
}, []);
```

## Takeaway
Move hook-driven side effects into a child component that mounts after a double `requestAnimationFrame` so the first frame only pays for what is visible.
