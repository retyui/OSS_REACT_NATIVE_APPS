# default freeze on immer for perf
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/25438>
- **Merged Date:** 2023-01-18
- **Category:** Rendering (+ Memory)

## Problem
The app disabled immer's auto-freeze in production builds (`setAutoFreeze(false)`) on the assumption that freezing was only a dev-time safety check and not worth paying for. The PR links an immer maintainer comment (immerjs/immer#687) arguing the opposite: auto-freeze is better left on for performance. The PR reports no measurements.

## Solution
- Remove the `if (!__DEV__) setAutoFreeze(false)` block so immer keeps its default behaviour (deep-freeze produced state) in release builds.
- Frozen results let immer skip re-traversing untouched subtrees on later `produce` calls, so subsequent drafts are cheaper (per the linked immer discussion, not measured in this PR).
- Side effect: dev and prod now behave identically, so accidental mutation throws everywhere instead of only in dev.

## Code Pattern (Before vs. After)
```tsx
// Before
import {type Draft as _Draft, setAutoFreeze} from 'immer'
// ...
// don't pay for this in prod builds
if (!__DEV__) {
  setAutoFreeze(false)
}

// After
import {type Draft as _Draft} from 'immer'
// (no setAutoFreeze call: immer's default auto-freeze stays on in production)
```

## Takeaway
Do not turn off immer auto-freeze in production to "save" work without profiling; frozen state can make later produce calls faster.
