# perf(startup): keep feature screens out of startup imports
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/811>
- **Merged Date:** 2026-10-18
- **Category:** Startup

## Problem
`src/shell/Providers.tsx`, `StorageLoadGate.tsx` and `usageSummary.ts` imported feature barrels (`@/features/<name>`), and those barrels re-exported every screen. Cold start therefore evaluated all screen modules before first render. By a grep-based import walk from the root layouts, reachable `src` modules dropped from 519 to 251 and screen modules from 117 to 0. Bundle size did not change (Hermes bundle 16,716,512 B → 16,717,293 B); the PR states the gain is evaluation at launch, and reports no startup timing.

## Solution
- Feature `index.ts` files now export only providers, hooks, logic and small components.
- A new `src/features/*/routes.ts` per feature exports the screens; `src/app/**` route files import screens from there, so they load per route.
- `EmotionItem` moved out of the screens folder so `calendar/index.ts` can export it without touching screens.
- oxlint rules enforce it: startup code (`src/shell/**`, `_layout.tsx`, `*Provider.tsx`) may import only `@/features/<name>`, and `index.ts` may not re-export `./routes` or `./screens/**`.

## Code Pattern (Before vs. After)
```tsx
// Before: src/app/(app)/calendar.tsx
import { CalendarScreen } from "@/features/calendar";

// After
import { CalendarScreen } from "@/features/calendar/routes";
```
```ts
// src/features/calendar/routes.ts (new)
// Route screens. Import only from src/app route files and other screens, never from startup code.
export { EntryPhotosScreen } from "./screens/EntryPhotos";
export { LogList } from "./screens/LogList";
export { default as CalendarScreen } from "./screens/Calendar";
```

## Takeaway
Keep screens out of barrels that the startup path imports, and add a lint rule so a new import cannot pull the whole screen graph back in.
