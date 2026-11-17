# chore: improve Sentry rewriteBreadcrumb performance
- **Repository:** MetaMask/metamask-mobile
- **PR Link:** <https://github.com/MetaMask/metamask-mobile/pull/19945>
- **Merged Date:** 2025-09-18
- **Category:** Startup (+ Rendering)

## Problem
Sentry's `rewriteBreadcrumb` hook took about 600 ms during login and froze the app after login. It parsed every breadcrumb URL with `new URL(url)` just to read the protocol, and the `URL` API is slow in React Native (the app additionally uses `react-native-url-polyfill`, which is slower than the built-in one). The PR reports about 600 ms -> about 20 ms (30x faster).

## Solution
- Replace `new URL(url).protocol` with plain string manipulation: find the first `:` and take the substring up to and including it.
- Throw on a missing colon to keep the "invalid URL" behavior the callers relied on.
- Some smaller Sentry perf issues remain, per the PR.

## Code Pattern (Before vs. After)
```js
// Before
function getProtocolFromURL(url) {
  return new URL(url).protocol;
}

// After
function getProtocolFromURL(url) {
  // Don't use URL api because it's slow in React Native
  const colonIndex = url.indexOf(':');
  if (colonIndex === -1) {
    throw new Error('Invalid URL');
  }
  return url.substring(0, colonIndex + 1);
}
```

## Takeaway
Avoid the `URL` API (especially with a polyfill) in hot paths such as logging/analytics hooks; use string operations when you only need a simple part.
