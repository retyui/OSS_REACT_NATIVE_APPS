# Reduce some startup load
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/28709>
- **Merged Date:** 2025-12-16
- **Category:** Startup

## Problem
Modules on the app's startup path (router, deeplinks, team-building, whats-new, tab bar badge) imported the whole crypto store and the whole `fs/common` barrel just to read a few constants or one icon component. Importing them eagerly pulled the crypto zustand store and the file-system component tree into the initial JS bundle evaluation. The PR gives no timing numbers.

## Solution
- Split the pure constants (tabs, route keys, `Operations`, info messages) out of `constants/crypto/index.tsx` into a side-effect-free `constants/crypto/util.tsx`; startup-path files import from `util`.
- Register the crypto store in `storeRegistry` with a lazy `require('./crypto')`, and call it via `storeRegistry.getState('crypto')` from deeplinks and team-building instead of a static import.
- Replace `import * as Kbfs from '@/fs/common'` with a direct import of `upload-icon`, and `constants/fs` with `constants/fs/util`.
- Reorder cheap checks before store reads in the router, and combine three selectors into one `useShallow` selector in the reset modal.
- Renamed the exported hook `useState` to `useCryptoState` (avoids confusion with React's `useState`).

## Code Pattern (Before vs. After)
```tsx
// Before
import * as Crypto from '../crypto'
// ...
const {onSaltpackOpenFile} = Crypto.useState.getState().dispatch
onSaltpackOpenFile(operation, path)

// After
import * as Crypto from '../crypto/util'
// ...
storeRegistry.getState('crypto').dispatch.onSaltpackOpenFile(operation, path)
```

```tsx
// store-registry.tsx (After)
case 'crypto': {
  const {useCryptoState} = require('./crypto')
  return useCryptoState
}
```

## Takeaway
Keep constants in a leaf module and reach rarely-used stores through a lazy `require` registry so they are not evaluated at app start.
