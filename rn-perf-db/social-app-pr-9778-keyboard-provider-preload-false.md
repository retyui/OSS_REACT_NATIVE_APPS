# Disable KeyboardProvider preload for faster initial load
- **Repository:** bluesky-social/social-app
- **PR Link:** <https://github.com/bluesky-social/social-app/pull/9778>
- **Merged Date:** 2026-01-27
- **Category:** Startup

## Problem
`react-native-keyboard-controller`'s `KeyboardProvider` preloads (initialises) the keyboard machinery at app start, even though the app wraps it with `enabled={false}` and only enables it on screens that need it. The PR describes this as unnecessary initialization during startup; no measurements are reported.

## Solution
Add `preload={false}` to the provider so initialization happens on demand when `useEnableKeyboardController` turns it on. One-line change; the test plan is to verify keyboard input still works on screens that enable the controller.

## Code Pattern (Before vs. After)
```tsx
// Before
<KeyboardProvider enabled={false}>
  <KeyboardControllerProviderInner>
    {children}
  </KeyboardControllerProviderInner>
</KeyboardProvider>

// After
<KeyboardProvider enabled={false} preload={false}>
  <KeyboardControllerProviderInner>
    {children}
  </KeyboardControllerProviderInner>
</KeyboardProvider>
```

## Takeaway
Check third-party providers for eager preload options and disable them where the feature is only needed on a few screens.
