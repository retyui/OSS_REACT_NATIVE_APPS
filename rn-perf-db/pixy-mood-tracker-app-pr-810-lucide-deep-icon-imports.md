# perf(icons): use lucide deep imports and drop react-native-feather
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/810>
- **Merged Date:** 2026-10-18
- **Category:** Bundle Size (+ Startup)

## Problem
Importing from the `lucide-react-native` barrel pulled in all ~1,850 icon modules, because Metro does not tree-shake. `react-native-feather` added another 287 modules. Measured with `expo export --platform ios --no-bytecode`: minified JS 13,268,410 B → 11,295,157 B (-14.9%), gzip 2,728,969 B → 2,521,151 B (-7.6%), Metro modules 6,524 → 4,448. The PR also lists a cold iOS bundle time of 39.9 s → 7.4 s but says it was not controlled (cache state differed).

## Solution
- Import each icon from `lucide-react-native/icons/<kebab-name>`.
- Replace every `react-native-feather` icon with a Lucide equivalent and remove the dependency; the GitHub brand icon, which Lucide lacks, becomes a local `GithubIcon` component.
- Add a Jest `moduleNameMapper` so deep icon paths resolve to the CJS build.
- Add an oxlint `no-restricted-imports` rule that blocks the barrel (type imports allowed) and `react-native-feather`, so the regression cannot return.

## Code Pattern (Before vs. After)
```tsx
// Before
import { Moon, Sun, SunMedium, Sunrise, Sunset } from "lucide-react-native";

// After
import Moon from "lucide-react-native/icons/moon";
import Sun from "lucide-react-native/icons/sun";
import SunMedium from "lucide-react-native/icons/sun-medium";
import Sunrise from "lucide-react-native/icons/sunrise";
import Sunset from "lucide-react-native/icons/sunset";
```

## Takeaway
Metro does not tree-shake barrel files from icon libraries by default; use deep imports and enforce them with a lint rule.
