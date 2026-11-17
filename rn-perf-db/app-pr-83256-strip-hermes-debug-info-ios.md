# [No QA] Strip Hermes debug info from production iOS bytecode
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/83256>
- **Merged Date:** 2026-02-27
- **Category:** App Size (+ Bundle Size)

## Problem
`react-native-xcode.sh` only passed `-output-source-map` to `hermesc` when `SOURCEMAP_FILE` was set. If the build environment did not propagate it, debug metadata stayed embedded in the shipped `main.jsbundle`. The PR states this is around 10 MB of debug data (the patch notes say ~13.4 MB), and shows Sentry bundle-size analysis before/after as screenshots.

## Solution
A React Native `patch-package` patch makes the script pass `-output-source-map` for every non-dev build, so `hermesc` writes debug info to a separate `.map` file instead of the bytecode. If source maps are not being composed, the generated `.map` file is deleted so it does not ship in the IPA. The normal compose-and-cleanup flow is unchanged. A matching fix was applied to Mobile-Expensify for HybridApp builds.

## Code Pattern (Before vs. After)
```bash
# Before
if [[ $EMIT_SOURCEMAP == true ]]; then
  EXTRA_COMPILER_ARGS="$EXTRA_COMPILER_ARGS -output-source-map"
fi

# After
if [[ $EMIT_SOURCEMAP == true || $DEV != true ]]; then
  EXTRA_COMPILER_ARGS="$EXTRA_COMPILER_ARGS -output-source-map"
fi
# ...
elif [[ $DEV != true ]]; then
  rm -f "$DEST/$BUNDLE_NAME.jsbundle.map"
fi
```
