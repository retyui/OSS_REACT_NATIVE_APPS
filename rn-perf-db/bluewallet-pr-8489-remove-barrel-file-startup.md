# FIX: greatly improve startup time
- **Repository:** BlueWallet/BlueWallet
- **PR Link:** <https://github.com/BlueWallet/BlueWallet/pull/8489>
- **Merged Date:** 2026-04-21
- **Category:** Startup

## Problem
`class/index.ts` was a barrel file that re-exported `BlueApp` and about 20 wallet classes. Importing anything from `'../class'` therefore evaluated every wallet class and its crypto dependencies at boot, and some modules (for example `BlueElectrum` and `LegacyWallet`) formed import cycles through the barrel. The PR author reports that this "shed 4 sec of boot time on android" just by rearranging imports. No measurement method is given in the PR.

## Solution
- Delete `class/index.ts` and change about 78 files to import each class directly from its own module.
- In `BlueElectrum.ts` and `legacy-wallet.ts`, wallet classes used only inside a function are imported with `import type` at module scope and loaded with an inline `require()` when the function runs. This breaks the module-scope cycles and defers evaluation.
- Modules on the startup path (`analytics`, `start-and-decrypt`, `SettingsProvider`, `StorageProvider`) now import only `class/blue-app` and the specific wallets they need.
- Caveat: inline `require()` is relied on for cycles, so it must only run after modules have finished loading.

## Code Pattern (Before vs. After)
```ts
// Before: class/index.ts
export * from './blue-app';
export * from './wallets/hd-segwit-bech32-wallet';
export * from './wallets/legacy-wallet';
export * from './wallets/multisig-hd-wallet';
// ... ~20 re-exports

// blue_modules/analytics.ts
import { BlueApp as BlueAppClass } from '../class';

// blue_modules/BlueElectrum.ts
import { LegacyWallet, SegwitBech32Wallet, SegwitP2SHWallet, TaprootWallet } from '../class';

// After: barrel deleted, direct imports
import { BlueApp as BlueAppClass } from '../class/blue-app';

// blue_modules/BlueElectrum.ts
import type { LegacyWallet as LegacyWalletT } from '../class/wallets/legacy-wallet';
import type { SegwitBech32Wallet as SegwitBech32WalletT } from '../class/wallets/segwit-bech32-wallet';

export function txhexToElectrumTransaction(txhex: string) {
  // Lazy require to avoid the module-scope cycle described above.
  const { SegwitBech32Wallet } = require('../class/wallets/segwit-bech32-wallet') as {
    SegwitBech32Wallet: typeof SegwitBech32WalletT;
  };
  // ...
}
```

## Takeaway
Replace `export *` barrel files on the startup path with direct imports, and use `import type` plus a function-level `require()` for dependencies that are only needed later or that cause cycles.
