# [LWM] chore(mobile): drop eager prepareCurrency(ethereum) at startup [LIVE-28532]
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/16198>
- **Merged Date:** 2026-04-09
- **Category:** Startup

## Problem
`live-common-setup.ts` unconditionally called `prepareCurrency(getCryptoCurrencyById("ethereum"))` at module load time, adding a blocking network/bridge call to the startup path regardless of whether the user owns any Ethereum account. No timing numbers are reported.

## Solution
- Remove the eager call and its imports.
- It was redundant: `prepareCurrency` is already invoked lazily by `BridgeSync` (per currency during account sync), by the scan-device-accounts view model before scanning, and by the select-device view model when the user reaches that screen.
- No functional change; the Ethereum bridge is prepared on demand instead.

## Code Pattern (Before vs. After)
```ts
// Before
import { prepareCurrency } from "./bridge/cache";
// ...
setSecp256k1Instance(require("./logic/secp256k1"));

prepareCurrency(getCryptoCurrencyById("ethereum"));

BigNumber.set({ DECIMAL_PLACES: getEnv("BIG_NUMBER_DECIMAL_PLACES") });

// After
setSecp256k1Instance(require("./logic/secp256k1"));

BigNumber.set({ DECIMAL_PLACES: getEnv("BIG_NUMBER_DECIMAL_PLACES") });
```

## Takeaway
Look for module-level side effects (prefetch, warm-up calls) that run at import time and replace them with calls at the point of first use.
