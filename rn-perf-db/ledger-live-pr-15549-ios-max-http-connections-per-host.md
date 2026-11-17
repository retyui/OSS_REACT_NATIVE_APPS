# [LWM] feat(mobile): increase iOS max HTTP connections per host for faster boot
- **Repository:** LedgerHQ/ledger-live
- **PR Link:** <https://github.com/LedgerHQ/ledger-live/pull/15549>
- **Merged Date:** 2026-03-23
- **Category:** Networking (+ Startup)

## Problem
At launch the app fires many requests, often to the same hosts. The iOS default for `httpMaximumConnectionsPerHost` is 6, so extra requests queue until a connection frees up. The PR does not report measured numbers.

## Solution
- Register a custom `NSURLSessionConfiguration` for React Native's networking via `RCTSetCustomNSURLSessionConfigurationProvider` in `AppDelegate`.
- Set `httpMaximumConnectionsPerHost = 16` to reduce request queuing during boot.
- Re-apply the default RN behaviours the provider replaces (cellular-access flag from Info.plist, cookie handling) so nothing else changes.
- The `RCTHTTPRequestHandler.h` header is added to the bridging header so Swift can call the setter.
- Caveat: more parallel connections per host can increase server load and contention on slow networks.

## Code Pattern (Before vs. After)
```swift
// Before
// (default RN URLSession configuration, max 6 connections per host)

// After
RCTSetCustomNSURLSessionConfigurationProvider {
  let config = URLSessionConfiguration.default
  config.httpMaximumConnectionsPerHost = 16
  if let useWifiOnly = Bundle.main.infoDictionary?["ReactNetworkForceWifiOnly"] as? Bool, useWifiOnly {
    config.allowsCellularAccess = false
  }
  config.httpShouldSetCookies = true
  config.httpCookieAcceptPolicy = .always
  config.httpCookieStorage = HTTPCookieStorage.shared
  return config
}
```

```objc
// LedgerWallet-Bridging-Header.h
#import <React/RCTHTTPRequestHandler.h>
```

## Takeaway
If your app issues bursts of requests to a few hosts at startup, raise the iOS per-host connection limit through RN's custom session configuration provider.
