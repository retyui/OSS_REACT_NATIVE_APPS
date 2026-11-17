# perf(avatar): cache mobile avatars across httpSrv token churn
- **Repository:** keybase/client
- **PR Link:** <https://github.com/keybase/client/pull/29407>
- **Merged Date:** 2026-07-10
- **Category:** Images (+ Networking)

## Problem
Avatars are served by a local HTTP server that restarts with a new token on every iOS foreground. The token is part of the avatar URI, so every avatar URI changed on each foreground, which defeated expo-image's URI-keyed cache and refetched every visible avatar. The PR gives no numbers.

## Solution
Pass an explicit `cacheKey` to expo-image built from avatar identity (type, name, size, mode, counter) and leave out address and token. The `counter` only bumps on the server's avatar-changed notification, so unchanged avatars are served from cache and changed ones are evicted by the new key.

## Code Pattern (Before vs. After)
```tsx
// Before
let source: {uri: string} | null = null
// ...
source = {uri: `http://${address}/av?typ=${typ}&name=${name}&format=square_${imgSize}&mode=${mode}&token=${token}&count=${counter}`}

// After
let source: {uri: string; cacheKey?: string} | null = null
// ...
source = {
  cacheKey: `av-${typ}-${name}-${imgSize}-${mode}-${counter}`,
  uri: `http://${address}/av?typ=${typ}&name=${name}&format=square_${imgSize}&mode=${mode}&token=${token}&count=${counter}`,
}
```

## Takeaway
When image URLs contain rotating tokens or hosts, give the image library an explicit cache key based on stable identity.
