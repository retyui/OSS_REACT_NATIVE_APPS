# perf(settings): load license notices from asset, not JS bundle
- **Repository:** mrzmyr/pixy-mood-tracker-app
- **PR Link:** <https://github.com/mrzmyr/pixy-mood-tracker-app/pull/813>
- **Merged Date:** 2026-10-18
- **Category:** Bundle Size (+ Startup)

## Problem
Third-party license notices lived in `disclaimer.js`, a 1,093 KB string module that was part of the JS bundle even though it is only needed on Settings > Licenses. It also listed dev-only packages.

## Solution
- Delete `disclaimer.js`; generate `assets/licenses.txt` with `bun run licenses`, walking only `dependencies` and `optionalDependencies` and merging packages with identical license text.
- The Licenses screen loads the asset on open via `expo-asset` + `expo-file-system` (`fetch` on web), with a loading spinner and an error/retry state that logs and reports to Sentry.
- Add `txt` to Metro `assetExts`.
- Reported with `expo export --platform ios`: minified JS 13,269,272 B → 12,156,859 B (-8.4%), Hermes bytecode 16,716,508 B → 14,554,276 B (-12.9%); the notices asset is 533,065 B, loaded only on that screen.

## Code Pattern (Before vs. After)
```tsx
// Before
import disclaimer from "../../../../disclaimer";

export const LicensesScreen = () => {
  const colors = useColors();
  const slices = disclaimer.split("-----");

// After (licensesText.ts)
// Text asset, not a JS module: keeps the notices out of the startup bundle.
const LICENSES_ASSET = require("../../../assets/licenses.txt");

await asset.downloadAsync();
return new File(asset.localUri).text();

// After (Licenses.tsx)
useEffect(() => {
  const load = async () => {
    const text = await loadLicensesText();
    setState({ status: "loaded", slices: text.split("-----") });
  };
  void load();
}, []);
```

## Takeaway
Move large, rarely used static text (licenses, legal copy, big JSON) out of JS modules into assets that are read on demand.
