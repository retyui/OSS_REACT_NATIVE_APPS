# Perf: Unmount Lottie animation when invisible to avoid a leak
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/67237>
- **Merged Date:** 2025-07-28
- **Category:** Memory (+ Animations)

## Problem
The Lottie wrapper only treated `hasNavigatedAway` as a reason to skip rendering when `shouldLoadAfterInteractions` was set, so other Lottie animations stayed mounted after the user left the screen and kept holding memory (the PR describes it as a leak). No numbers are reported.

## Solution
Make `hasNavigatedAway` an unconditional reason to render the placeholder `View` instead of the animation. The `shouldLoadAfterInteractions` check now only gates on interaction completion. The animation is unmounted whenever its screen is not visible.

## Code Pattern (Before vs. After)
```tsx
// Before
if (
    isError ||
    appState.isBackground ||
    !animationFile ||
    splashScreenState !== CONST.BOOT_SPLASH_STATE.HIDDEN ||
    ((!isInteractionComplete || hasNavigatedAway) && shouldLoadAfterInteractions)
) {

// After
if (
    isError ||
    appState.isBackground ||
    !animationFile ||
    hasNavigatedAway ||
    splashScreenState !== CONST.BOOT_SPLASH_STATE.HIDDEN ||
    (!isInteractionComplete && shouldLoadAfterInteractions)
) {
```
